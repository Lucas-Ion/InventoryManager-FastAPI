# A Logistics API

A RESTful API built with FastAPI for managing a logistics company that ships cargo via vessels.

---

## Table of Contents

- [Overview](#overview)
- [Setup Instructions](#setup-instructions)
- [Running the Application](#running-the-application)
- [Running the Tests](#running-the-tests)
- [Design Decisions](#design-decisions)
- [Data Model](#data-model)
- [API Documentation](#api-documentation)
- [Deployment](#deployment)
- [Production Considerations](#production-considerations)

---

## Overview

This API imagines what would be needed to allows a logistics company to manage the full lifecycle of a shipment. This goes from creating a client and signing a contract, to assigning cargo to a vessel and tracking its movement through to delivery.

The core pieces are:

- **Clients** - companies or individuals that enter into shipping agreements
- **Vessels** - ships responsible for transporting cargo
- **Contracts** - shipping agreements between a client and the company
- **Cargoes** - goods being transported under a specific contract
- **Tracking Events** - an immutable audit log of cargo movements and locations


### To interact with the deployed API online, click [HERE!](https://deus-logistics-api.wonderfulmeadow-a48258d9.westeurope.azurecontainerapps.io/docs)


## Setup Instructions


### For running locally

Clone the repository:

```
git clone github.com/Lucas-Ion/InventoryManager-FastAPI
cd InventoryManager-FastAPI
```
Then run:

```
python3 -m venv antenv
source antenv/bin/activate
pip install -r requirements/dev-requirements.txt
```

Copy the environment file and configure it:

```
cp .env.example .env
```

The default values in `.env.example` work for local development without any changes.


You can now start the application:
```
uvicorn app.app:app --reload
```

The API will be available at:

- `http://localhost:8000/health` — health check
- `http://localhost:8000/docs` — Swagger UI
- `http://localhost:8000/redoc` — ReDoc
---

### If you want to run with Docker

```
docker compose up --build
```

---

## Running the Tests

```
source antenv/bin/activate
pytest tests/ -v
```

## Design Decisions

### Architecture — Service Layer Pattern

The codebase follows a three-layer architecture:

```
HTTP Layer (endpoints/)

      | (communicates with)

Business Logic Layer (services/)

      | (communicates with)

Data Layer (models/ + db/)
```

Endpoints are kept intentionally thin following the skinny controller fat model design.  The endpoints  handle HTTP concerns only (status codes, request parsing, and response serialisation) and delegate all logic to the service layer. This makes both the endpoints and the services independently testable and easy to understand for anyone who is viewing it for the first time.

I used this pattern as it is directly comparable to the View/Serializer/Model separation in Django REST Framework which I currently use in Production. This gave me a clear reference point for the project structure as I was building with FastAPI for the first time

### Schema Separation with Base / Create / Read

Each entity has three Pydantic schema variants rather than a single schema:

- `Base` contains shared fields common to all operations
- `Create` extends Base with write-only fields (e.g. `client_id`) that a caller supplies on creation
- `Read` extends Base with server-generated fields (e.g. `id`, `created_at`) returned in responses

I made it this way so that the application is prevented from accidentally exposing internal fields and makes the API contract explicit. A client can never supply an `id` on a POST request because `id` simply does not exist on the `Create` schema.

### Audit History using an append-only TrackingEvent

The tracking history requirement was implemented as an append-only `TrackingEvent` table rather than a single mutable `location` field on `Cargo`. Every status change or location update writes a new immutable row and records are never updated or deleted.

This gives a complete and ordered history of everywhere a cargo has been throughout its lifecycle. The `GET /api/v1/cargoes/{id}/history` endpoint returns the cargo alongside its full ordered event list making it clear to infer the start and finish of a cargo lifecycle.

Kafka was deliberately not used for this even though in production this would be a perfect use case for it. For a single-service application persisting to a relational database, an append-only table is the simpler solution although I mention this as in a real customer app I would have made a different design decision.

### Database with SQLite with SQLAlchemy Async

SQLite was used as specified by the challenge. I chose to use the async variant (`aiosqlite`) so that I can match FastAPI's async architecture and avoid blocking the event loop during database operations.

The SQLAlchemy 2.0 ORM style was so that I can provide full mypy type checking support and cleaner relationship definitions.

Table creation is handled by `Base.metadata.create_all` on startup via the `lifespan` context manager.

### API Versioning

All entity endpoints are prefixed with `/api/v1/` configured via the `api_prefix` setting in `config.py`. The health check endpoint is located at `/health` outside the versioned prefix. Although not explcitly defined in the challenge I also deployed to Azure for convenience of testing, and the Azure Container Apps health probes expect the health endpoint at a predictable root-level path so that is why it's there.

### Dependency Injection - get_db

With regards to managing database session I used FastAPI's `Depends(get_db)` pattern. Each request will get its own session that commits on success and rolls back on any exception. 

### Logging - Loguru with App Insights Sink

Loguru was chosen over the standard `logging` module for its significantly cleaner API and structured output in my opinion and the experience I had using it in various production applications. A custom `InterceptHandler` routes all standard library loggers through Loguru so all logs flow through a single consistent pipeline.

In production, when `APPLICATIONINSIGHTS_CONNECTION_STRING` is set, logs, traces, and exceptions are automatically forwarded to Azure Application Insights via the `azure-monitor-opentelemetry` package.

<img width="1407" height="827" alt="Screenshot 2026-03-30 at 01 41 37" src="https://github.com/user-attachments/assets/6188c770-d6f6-449a-b923-33c4e35c0452" />

### Error Handling

Business rule violations (e.g. referencing a non-existent client, duplicate email) are raised as `ValueError` in the service layer and caught at the endpoint layer where they are converted to HTTP 422 responses. Database integrity errors (`IntegrityError`) are caught at the service layer before they propagate as unhandled 500 errors.

This separation means the service layer has no knowledge of HTTP and it only raises Python exceptions which keeps the layers quite separated.

### Some Code Quality Descisions I made

- I chose `Ruff` to handle both linting and formatting, I chose ruff because it is much faster than flake8 or black
- `mypy` runs in strict mode for max type safety
- Throughout the codebase I used [Google-style docstrings](https://github.com/google/styleguide/blob/gh-pages/pyguide.md#38-comments-and-docstrings) as they are one of the most common styles in the industry
- All configuration for Ruff, mypy, and pytest is set in `pyproject.toml` as a single source of truth

### Infrastructure for ease of tsting

I provisioned a Container Registry, Container Apps Environment and Container App via the Azure Portal. In do recognize that in a production environment this would be managed as code using Azure Bicep or Terraform for reproducibility and version control.

### A quick note on my Test Approach


Alongside my standard unit tests I wrote explicit service mocking in `test_mocked.py` which uses `unittest.mock.AsyncMock` and `patch` to mock the service layer directly. This tests endpoint behaviour in complete isolation from the database, so that the tests can verif that endpoints correctly handle service-layer responses such as `None` returns and `ValueError` exceptions.

---

## Data Model

```
clients
  id, name, email (unique), phone, created_at

vessels
  id, name (unique), capacity_tons, current_location, created_at

contracts
  id, client_id (FK -> clients), cargo_type, destination, price, created_at

cargoes
  id, contract_id (FK -> contracts, unique), vessel_id (FK -> vessels),
  description, weight_tons, status (pending/in_transit/delivered),
  created_at, updated_at

tracking_events
  id, cargo_id (FK -> cargoes), location, status, recorded_at
```

Relationship I assumed:

- A client has many contracts
- A contract has exactly one cargo
- A vessel has many cargoes
- A cargo has many tracking events (append-only audit log)

Cascade rules behaviour I defined:

- Deleting a client cascades to their contracts and cargoes
- Deleting a vessel sets `vessel_id` to NULL on related cargoes rather than deleting them (as if they are stranded or inbetween ships)

---

## API Documentation

I generated full API documentation which is available at `/docs` (Swagger UI) and `/redoc` when the application is running.

Here is an overview off all the endpoints:

### Health

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Returns API status, version, and environment |

### Clients

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/v1/clients/` | Create a new client |
| GET | `/api/v1/clients/` | List all clients (paginated) |
| GET | `/api/v1/clients/{client_id}` | Get a single client |
| GET | `/api/v1/clients/{client_id}/contracts` | List all contracts for a client |

### Vessels

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/v1/vessels/` | Register a new vessel |
| GET | `/api/v1/vessels/` | List all vessels (paginated) |
| GET | `/api/v1/vessels/{vessel_id}` | Get a single vessel |
| PATCH | `/api/v1/vessels/{vessel_id}` | Update vessel fields (e.g. location) |

### Contracts

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/v1/contracts/` | Create a new contract |
| GET | `/api/v1/contracts/` | List all contracts (paginated) |
| GET | `/api/v1/contracts/{contract_id}` | Get a single contract |

### Cargoes and Tracking

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/v1/cargoes/` | Create a cargo entry for a contract |
| GET | `/api/v1/cargoes/` | List all cargoes (paginated) |
| GET | `/api/v1/cargoes/{cargo_id}` | Get a single cargo |
| GET | `/api/v1/cargoes/{cargo_id}/history` | Get cargo with full tracking history |
| PATCH | `/api/v1/cargoes/{cargo_id}/status` | Update cargo status and append tracking event |
| POST | `/api/v1/cargoes/{cargo_id}/tracking` | Manually append a tracking event |
| GET | `/api/v1/cargoes/{cargo_id}/tracking` | List all tracking events for a cargo |

### Example Payloads

**Create Client**
```json
POST /api/v1/clients/
{
    "name": "DEUS Shipping Corp",
    "email": "contact@deusshipping.com",
    "phone": "+351-555-987-177"
}
```

**Create Vessel**
```json
POST /api/v1/vessels/
{
    "name": "Atlantic Star",
    "capacity_tons": 25000.0,
    "current_location": "Port of Leixões"
}
```

**Create Contract**
```json
POST /api/v1/contracts/
{
    "client_id": 1,
    "cargo_type": "Electronics",
    "destination": "Singapore",
    "price": 150000.00
}
```

**Create Cargo**
```json
POST /api/v1/cargoes/
{
    "contract_id": 1,
    "vessel_id": 1,
    "description": "500 units of LCD panels",
    "weight_tons": 12.5
}
```

**Update Cargo Status**
```json
PATCH /api/v1/cargoes/1/status
{
    "status": "in_transit",
    "location": "Suez Canal"
}
```

**Response — Cargo with History**
```json
GET /api/v1/cargoes/1/history
{
    "id": 1,
    "contract_id": 1,
    "vessel_id": 1,
    "description": "500 units of LCD panels",
    "weight_tons": 12.5,
    "status": "delivered",
    "created_at": "2024-01-01T10:00:00Z",
    "updated_at": "2024-01-10T15:30:00Z",
    "tracking_events": [
        {
            "id": 1,
            "cargo_id": 1,
            "location": "Port of Rotterdam",
            "status": "in_transit",
            "recorded_at": "2024-01-01T10:00:00Z"
        },
        {
            "id": 2,
            "cargo_id": 1,
            "location": "Suez Canal",
            "status": "in_transit",
            "recorded_at": "2024-01-05T08:00:00Z"
        },
        {
            "id": 3,
            "cargo_id": 1,
            "location": "Port of Singapore",
            "status": "delivered",
            "recorded_at": "2024-01-10T15:30:00Z"
        }
    ]
}
```


---

## Deployment

The application is deployed to an Azure Container App via GitHub Actions.

### CI/CD Pipeline

Every push to any branch triggers the CI workflow which runs Ruff linting and the full pytest suite.

Every merge to `main` triggers the deploy workflow which:

1. Runs the full test suite
2. Builds a Docker image tagged with both `latest` and the Git commit SHA
3. Pushes the image to Azure Container Registry
4. Updates the Container App to use the new image revision

For the pipeline I used my Azure cli to create a credential for RBAC

## Some final Production Considerations

In conclusion I want to also lay out some last considerations in this code which would differ in a production scope and which would be addressed before taking this system to production:

Database - I would switch to PostgreSQL via Azure Database for PostgreSQL.

Authentication - Right now the API currently has no authentication layer. In production I would require JWT-based authentication or OAuth2 with role-based access control so clients can only access their own contracts and cargo.

Infrastructure as Code - Azure infrastructure was provisioned via the Azure Portal for this submission. Production infrastructure would be defined using Azure Bicep (as that is what I am familiar with) or of course Terraform as an alternative

Integration Tests - Right now the current test suite covers all endpoints with unit-level tests. In a prod codebase I would add integration tests covering multi-step flows such as the full lifecycle from client creation through contract signing, cargo assignment, and delivery tracking etc.

---

### Thanks for reading, I hope you enjoyed taking a look through my submission! :)
