# Galaxium Travels — Architecture Map

> **Evidence policy:** Every claim in this document is traced to a specific file and line.
> Items marked **[UNVERIFIED]** could not be confirmed from repository sources.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Service Inventory](#2-service-inventory)
3. [Component Diagram](#3-component-diagram)
4. [Data Models](#4-data-models)
5. [Entry Points & Ports](#5-entry-points--ports)
6. [Key Flows](#6-key-flows)
   - [Direct Booking Flow](#61-direct-booking-flow)
   - [Hold-Based Booking Flow](#62-hold-based-booking-flow)
7. [Environment Variables & Configuration](#7-environment-variables--configuration)
8. [Dependency Graph (Python)](#8-dependency-graph-python)
9. [Test Architecture](#9-test-architecture)
10. [Infrastructure & Deployment](#10-infrastructure--deployment)
11. [Architectural Invariants & Footguns](#11-architectural-invariants--footguns)

---

## 1. System Overview

Galaxium Travels is a demo interplanetary flight-booking application built to showcase challenges agents face in a multi-service codebase. It consists of three runtime services and a test harness:

| Service | Language/Framework | Database | Primary Role |
|---|---|---|---|
| **Python Backend** | Python 3 / FastAPI + FastMCP | SQLite (default) or PostgreSQL | REST API, MCP server, seat inventory |
| **Java Hold Service** | Java 17–21 / Spring Boot 3 | SQLite (`holds.db`) | Quote & hold lifecycle management |
| **React Frontend** | React 19 / TypeScript / Vite | — | Browser UI |
| **e2e Suite** | pytest / Docker Compose | Ephemeral SQLite | Cross-service integration tests |

Sources: [`AGENTS.md`](../../AGENTS.md), [`docker-compose.yml`](../../docker-compose.yml).

---

## 2. Service Inventory

### 2.1 Python Backend (`booking_system_backend/`)

- **Entry point:** [`server.py`](../../booking_system_backend/server.py) — `FastAPI` app is created at line 44; `FastApiMCP(app)` is mounted at line 336 after all routes are registered. No `FastMCP` exists at line 22; that line is an import.
- **Port:** `8001` ([`server.py:344`](../../booking_system_backend/server.py:344))
- **URL prefix:** `/api` via `root_path="/api"` ([`server.py:49`](../../booking_system_backend/server.py:49))
- **Lifespan:** `init_db()` → optional `seed()` controlled by `SEED_DEMO_DATA` env var ([`server.py:29–39`](../../booking_system_backend/server.py:29))
- **MCP server:** `FastApiMCP(app)` auto-generates MCP tools from all FastAPI routes ([`server.py:336–337`](../../booking_system_backend/server.py:336))

#### HTTP Routes

| Method | Path | Handler | Line | Notes |
|---|---|---|---|---|
| GET | `/` | `health_check` | [64](../../booking_system_backend/server.py:64) | Returns `{"status":"OK"}` |
| GET | `/flights` | `get_flights` | [70](../../booking_system_backend/server.py:70) | 14 optional query params |
| POST | `/book` | `book_flight_endpoint` | [152](../../booking_system_backend/server.py:152) | Direct seat booking |
| GET | `/bookings/{user_id}` | `get_user_bookings` | [169](../../booking_system_backend/server.py:169) | |
| POST | `/cancel/{booking_id}` | `cancel_booking_endpoint` | [175](../../booking_system_backend/server.py:175) | |
| POST | `/register` | `register_user_endpoint` | [189](../../booking_system_backend/server.py:189) | |
| GET | `/user` | `get_user_endpoint` | [202](../../booking_system_backend/server.py:202) | Lookup by name+email |
| POST | `/internal/bookings/from-hold` | `create_booking_from_hold` | [221](../../booking_system_backend/server.py:221) | Called by Java only |
| POST | `/quotes` | `create_quote` (proxy) | [242](../../booking_system_backend/server.py:242) | Forwards to Java |
| GET | `/quotes/{quote_id}` | `get_quote` (proxy) | [258](../../booking_system_backend/server.py:258) | Forwards to Java |
| POST | `/quotes/{quote_id}/holds` | `create_hold` (proxy) | [273](../../booking_system_backend/server.py:273) | Forwards to Java |
| GET | `/holds/{hold_id}` | `get_hold` (proxy) | [288](../../booking_system_backend/server.py:288) | Forwards to Java |
| POST | `/holds/{hold_id}/confirm` | `confirm_hold` (proxy) | [303](../../booking_system_backend/server.py:303) | Forwards to Java |
| POST | `/holds/{hold_id}/release` | `release_hold` (proxy) | [318](../../booking_system_backend/server.py:318) | Forwards to Java |

Proxy routes use `httpx.AsyncClient` and catch `httpx.HTTPError`, returning `{"error": "..."}` with HTTP 200 — callers must inspect the body ([`AGENTS.md`](../../AGENTS.md)).

### 2.2 Java Hold Service (`booking_system_inventory_hold_service/`)

- **Entry point:** [`HoldServiceApplication.java`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/HoldServiceApplication.java)
- **Port:** `8080` ([`application.properties:3`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:3))
- **Hold duration:** 15 minutes ([`application.properties:19`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:19))
- **Expiry scheduler interval:** 60 seconds ([`application.properties:20`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:20))
- **Python backend URL:** `${PYTHON_BACKEND_URL:http://localhost:8001}` ([`application.properties:16`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:16))

#### HTTP Routes (Java)

| Method | Path | Controller | Notes |
|---|---|---|---|
| GET | `/api/v1/health` | [`HealthController`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/HealthController.java:16) | Returns `{"status":"UP"}` |
| POST | `/api/v1/quotes` | [`QuoteController:22`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/QuoteController.java:22) | Creates quote, 201 |
| GET | `/api/v1/quotes/{quoteId}` | [`QuoteController:28`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/QuoteController.java:28) | 200 or 404 |
| POST | `/api/v1/quotes/{quoteId}/holds` | [`HoldController:19`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/HoldController.java:19) | 201 or 400/404 |
| GET | `/api/v1/holds/{holdId}` | [`HoldController:34`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/HoldController.java:34) | 200 or 404 |
| POST | `/api/v1/holds/{holdId}/confirm` | [`HoldController:42`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/HoldController.java:42) | Triggers Python callback |
| POST | `/api/v1/holds/{holdId}/release` | [`HoldController:57`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/HoldController.java:57) | 200 or 400/404 |

### 2.3 React Frontend (`booking_system_frontend/`)

- **Dev server port:** `5173` ([`AGENTS.md`](../../AGENTS.md))
- **API base URL:** `VITE_API_URL || '/api'` ([`api.ts:14`](../../booking_system_frontend/src/services/api.ts:14))
- **Pages:** `Home`, `Flights`, `MyBookings`, `DestinationDetail`
- **API service:** [`api.ts`](../../booking_system_frontend/src/services/api.ts) — all backend calls centralised here; response interceptor normalises errors ([`api.ts:21–37`](../../booking_system_frontend/src/services/api.ts:21))

---

## 3. Component Diagram

```mermaid
graph TD
    subgraph Browser
        FE["React Frontend\n:5173"]
    end

    subgraph Python Service [:8001]
        FAST["FastAPI app\nserver.py"]
        MCP["FastApiMCP\n(MCP tools)"]
        SVC_B["booking.py"]
        SVC_F["flight.py"]
        SVC_U["user.py"]
        DB_PY[("SQLite / PostgreSQL\nbooking.db")]
    end

    subgraph Java Service [:8080]
        SPRING["Spring Boot\nHoldServiceApplication"]
        Q_CTRL["QuoteController"]
        H_CTRL["HoldController"]
        SCHED["HoldExpirationScheduler\n(every 60 s)"]
        PY_CLI["PythonBackendClient\n(RestTemplate)"]
        DB_JAVA[("SQLite\nholds.db")]
    end

    FE -->|"REST /api/*"| FAST
    FAST -->|"proxy via httpx"| SPRING
    FAST --> SVC_B
    FAST --> SVC_F
    FAST --> SVC_U
    SVC_B --> DB_PY
    SVC_F --> DB_PY
    SVC_U --> DB_PY
    SPRING --> Q_CTRL
    SPRING --> H_CTRL
    SPRING --> SCHED
    H_CTRL -->|"POST /internal/bookings/from-hold"| FAST
    PY_CLI -->|"POST /internal/bookings/from-hold"| FAST
    H_CTRL --> PY_CLI
    SPRING --> DB_JAVA
    SCHED --> DB_JAVA

    style DB_PY fill:#2a3a5c,color:#fff
    style DB_JAVA fill:#2a3a5c,color:#fff
```

---

## 4. Data Models

### 4.1 Python — SQLAlchemy Models ([`models.py`](../../booking_system_backend/models.py))

```mermaid
erDiagram
    users {
        int user_id PK
        string name
        string email
    }
    flights {
        int flight_id PK
        string origin
        string destination
        string departure_time
        string arrival_time
        int base_price
        int economy_seats_available
        int business_seats_available
        int galaxium_seats_available
    }
    bookings {
        int booking_id PK
        int user_id FK
        int flight_id FK
        string status
        string booking_time
        string seat_class
        int price_paid
    }
    users ||--o{ bookings : "places"
    flights ||--o{ bookings : "has"
```

**Seat class pricing multipliers** ([`booking.py:9–13`](../../booking_system_backend/services/booking.py:9)):

| Class | Multiplier | Formula |
|---|---|---|
| economy | 1.0× | `base_price * 1.0` |
| business | 2.5× | `base_price * 2.5` |
| galaxium | 5.0× | `base_price * 5.0` |

**Booking status values:** `booked`, `cancelled`, `completed` ([`models.py:8–12`](../../booking_system_backend/models.py:8))

### 4.2 Java — JPA Entities

Stored in `holds.db` (SQLite). Schema managed by `spring.jpa.hibernate.ddl-auto=update`.

| Entity | Key Fields | Source |
|---|---|---|
| `Quote` | `quoteId` (Q-YYYY-XXXXXX), `flightId`, `seatClass`, `travelerId`, `travelerName`, `pricePerSeat`, `totalPrice`, `expiresAt` (now + 24 h), `status` | [`Quote.java`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/domain/Quote.java) |
| `Hold` | `holdId` (H-YYYY-XXXXXX), `quoteId`, `status` (HELD/EXPIRED/CONFIRMED/RELEASED/CONFIRMATION_FAILED), `reservedUntil` (now + 15 min), `externalBookingReference` | [`Hold.java`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/domain/Hold.java) |
| `AuditEvent` | `eventId` (UUID), `entityType`, `entityId`, `eventType`, `details`, `createdAt` | [`AuditEvent.java`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/domain/AuditEvent.java) |

---

## 5. Entry Points & Ports

| Service | Local Port | Docker Port | Start Command |
|---|---|---|---|
| Python Backend | `8001` | `8001→8080` | `.venv/bin/python server.py` |
| Java Hold Service | `8080` | `8082→8080` (compose), `8080` (native) | `mvn spring-boot:run` |
| React Frontend | `5173` | `5173→8080` | `npm run dev` |
| PostgreSQL (compose only) | `5433` | `5433→5432` | managed by Docker |
| e2e Backend | `18001` | `18001→8080` | Docker Compose e2e |
| e2e Java Service | `18082` | `18082→8080` | Docker Compose e2e |

Sources: [`docker-compose.yml`](../../docker-compose.yml), [`e2e/docker-compose.e2e.yml`](../../e2e/docker-compose.e2e.yml), [`server.py:344`](../../booking_system_backend/server.py:344), [`application.properties:3`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:3).

**All-in-one local startup:** `./start.sh` → [`scripts/local/start_locally.sh`](../../scripts/local/start_locally.sh) (starts backend → Java → frontend, waits for healthchecks between each).

---

## 6. Key Flows

### 6.1 Direct Booking Flow

The simplest path: frontend calls Python directly; no Java involvement.

```mermaid
sequenceDiagram
    participant FE as React Frontend
    participant PY as Python Backend
    participant DB as booking.db

    FE->>PY: GET /flights (with filters)
    PY->>DB: SELECT flights WHERE ...
    DB-->>PY: []Flight
    PY-->>FE: []FlightOut

    FE->>PY: POST /register {name, email}
    PY->>DB: INSERT users
    DB-->>PY: User
    PY-->>FE: UserOut

    FE->>PY: POST /book {user_id, name, flight_id, seat_class}
    PY->>DB: SELECT flight (check seats)
    PY->>DB: SELECT user (validate name matches)
    PY->>DB: UPDATE flight (decrement seat counter)
    PY->>DB: INSERT bookings
    DB-->>PY: Booking
    PY-->>FE: BookingOut {booking_id, price_paid, ...}
```

Key validation in [`booking.py:51–65`](../../booking_system_backend/services/booking.py:51): `user_id` AND `name` must both match — name mismatch returns `NAME_MISMATCH` error (non-standard security pattern, intentional).

### 6.2 Hold-Based Booking Flow

Seat inventory is only decremented at confirmation. The Java service owns the quote/hold lifecycle; the Python backend owns actual seat inventory.

```mermaid
sequenceDiagram
    participant FE as React Frontend
    participant PY as Python Backend (proxy)
    participant JAVA as Java Hold Service
    participant DB_J as holds.db
    participant DB_P as booking.db

    FE->>PY: POST /quotes {flightId, seatClass, travelerId, travelerName}
    PY->>JAVA: POST /api/v1/quotes (httpx proxy)
    JAVA->>DB_J: INSERT quotes (expiresAt = now + 24h)
    DB_J-->>JAVA: Quote
    JAVA-->>PY: Quote {quoteId: "Q-YYYY-XXXXXX"}
    PY-->>FE: Quote

    FE->>PY: POST /quotes/{quoteId}/holds
    PY->>JAVA: POST /api/v1/quotes/{quoteId}/holds (httpx proxy)
    JAVA->>DB_J: INSERT holds (reservedUntil = now + 15 min, status = HELD)
    DB_J-->>JAVA: Hold
    JAVA-->>PY: Hold {holdId: "H-YYYY-XXXXXX", status: "HELD"}
    PY-->>FE: Hold
    Note over DB_P: No seat change yet

    FE->>PY: POST /holds/{holdId}/confirm
    PY->>JAVA: POST /api/v1/holds/{holdId}/confirm (httpx proxy)
    JAVA->>DB_J: Check hold (status=HELD, reservedUntil > now)
    JAVA->>PY: POST /internal/bookings/from-hold {travelerId, travelerName, flightId, seatClass}
    PY->>DB_P: INSERT bookings + UPDATE flight (decrement seat)
    DB_P-->>PY: Booking
    PY-->>JAVA: BookingOut {booking_id}
    JAVA->>DB_J: UPDATE holds SET status=CONFIRMED, externalBookingReference=booking_id
    DB_J-->>JAVA: Hold
    JAVA-->>PY: Hold {status: "CONFIRMED"}
    PY-->>FE: Hold

    Note over JAVA: Background: HoldExpirationScheduler
    loop Every 60 s
        JAVA->>DB_J: SELECT holds WHERE reservedUntil < now AND status = HELD
        JAVA->>DB_J: UPDATE holds SET status = EXPIRED
    end
```

Sources: [`server.py:221–330`](../../booking_system_backend/server.py:221), [`HoldController.java:42–55`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/api/HoldController.java:42), [`PythonBackendClient.java:30–58`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/client/PythonBackendClient.java:30), [`HoldExpirationScheduler.java:24`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/scheduler/HoldExpirationScheduler.java:24).

#### Hold Status State Machine

```mermaid
stateDiagram-v2
    [*] --> HELD : createHold()
    HELD --> CONFIRMED : confirmHold() + Python callback success
    HELD --> CONFIRMATION_FAILED : confirmHold() + Python callback error
    HELD --> RELEASED : releaseHold()
    HELD --> EXPIRED : scheduler (reservedUntil elapsed)
    CONFIRMED --> [*]
    CONFIRMATION_FAILED --> [*]
    RELEASED --> [*]
    EXPIRED --> [*]
```

Source: [`Hold.java` (HoldStatus enum)](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/domain/Hold.java), [`HoldService.java:75–155`](../../booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/service/HoldService.java:75).

---

## 7. Environment Variables & Configuration

### Python Backend

| Variable | Default | Effect | Source |
|---|---|---|---|
| `DATABASE_URL` | `sqlite:///./booking.db` | Database connection string | [`db.py:9`](../../booking_system_backend/db.py:9) |
| `SEED_DEMO_DATA` | `true` | Seed demo users/flights/bookings on startup | [`server.py:33`](../../booking_system_backend/server.py:33) |
| `CORS_ORIGINS` | `*` | CORS allowed origins | [`server.py:53`](../../booking_system_backend/server.py:53) |
| `JAVA_SERVICE_URL` | `http://localhost:8080` | Java hold service base URL for proxy | [`server.py:218`](../../booking_system_backend/server.py:218) |

### Java Hold Service

| Variable | Default | Effect | Source |
|---|---|---|---|
| `PYTHON_BACKEND_URL` | `http://localhost:8001` | Python backend URL for callbacks | [`application.properties:16`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:16) |
| `HOLD_DURATION_MINUTES` | `15` | How long a hold is valid | [`application.properties:19`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:19) |
| `HOLD_EXPIRATION_CHECK_INTERVAL_SECONDS` | `60` | Scheduler interval | [`application.properties:20`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:20) |
| `SPRING_DATASOURCE_URL` | `jdbc:sqlite:./holds.db` | Java DB path | [`application.properties:6`](../../booking_system_inventory_hold_service/src/main/resources/application.properties:6) |

### Frontend

| Variable | Default | Effect | Source |
|---|---|---|---|
| `VITE_API_URL` | `/api` | Backend base URL for all API calls | [`api.ts:14`](../../booking_system_frontend/src/services/api.ts:14) |

### Docker Compose overrides ([`docker-compose.yml`](../../docker-compose.yml))

- `backend` container: `DATABASE_URL=postgresql://galaxium:local_dev_password@postgres:5432/galaxium_booking`
- `java-service` container: `PYTHON_BACKEND_URL=http://backend:8080`, `SPRING_DATASOURCE_URL=jdbc:sqlite:/app/data/holds.db`
- `frontend` container: `VITE_API_URL=http://localhost:8001` (build arg)

### e2e overrides ([`e2e/docker-compose.e2e.yml`](../../e2e/docker-compose.e2e.yml))

- No Postgres — both services use SQLite
- Java: `HOLD_DURATION_MINUTES=1`, `HOLD_EXPIRATION_CHECK_INTERVAL_SECONDS=5` (fast expiry for tests)
- Ports offset: backend `18001`, java `18082`

---

## 8. Dependency Graph (Python)

```mermaid
graph LR
    server["server.py\n(entry point)"] --> db["db.py\nengine, SessionLocal, get_db"]
    server --> models["models.py\nUser, Flight, Booking"]
    server --> schemas["schemas.py\nPydantic models"]
    server --> svc_b["services/booking.py"]
    server --> svc_f["services/flight.py"]
    server --> svc_u["services/user.py"]
    server --> seed["seed.py\n(demo data)"]
    svc_b --> models
    svc_b --> schemas
    svc_f --> models
    svc_f --> schemas
    svc_u --> models
    svc_u --> schemas
    db --> models
    seed --> db
    seed --> models
```

Key third-party packages ([`requirements.txt`](../../booking_system_backend/requirements.txt)):
`fastapi`, `fastmcp`, `fastapi-mcp`, `uvicorn`, `sqlalchemy`, `pydantic`, `python-dotenv`, `httpx`, `psycopg2-binary`, `pytest`, `pytest-asyncio`, `pytest-cov`.

---

## 9. Test Architecture

### Unit / Integration Tests (Python)

- Location: [`booking_system_backend/tests/`](../../booking_system_backend/tests/)
- Runner: `pytest` from inside `booking_system_backend/` ([`AGENTS.md`](../../AGENTS.md))
- Database: in-memory SQLite (`StaticPool`)
- **Critical:** `conftest.py` uses FastAPI's `dependency_overrides` mechanism — `server.app.dependency_overrides[db_module.get_db] = override_get_db` ([`conftest.py:55`](../../booking_system_backend/tests/conftest.py:55)) — to redirect `get_db` to the in-memory test session. `SessionLocal` is **not** monkeypatched.

### e2e Tests

| File | Scope | Hold Duration |
|---|---|---|
| [`e2e/test_smoke.py`](../../e2e/test_smoke.py) | Python backend only (health, flights, booking, name-mismatch, unknown-flight) | N/A |
| [`e2e/test_holds.py`](../../e2e/test_holds.py) | Cross-service (quote→hold→confirm/release, idempotency, auto-expiry) | 1 min (e2e config) |

Run: `./e2e/run-native.sh`; add `E2E_RUN_SLOW=1` for the auto-expiry test (~90 s).

### Java Unit Tests

- Runner: `mvn test -q` from `booking_system_inventory_hold_service/`

---

## 10. Infrastructure & Deployment

### Local (Native)

```
./start.sh
  → scripts/local/start_locally.sh
     1. Python backend on :8001
     2. Java hold service on :8080 (auto-detects Java 17/21 via sdkman)
     3. React frontend on :5173
```

### Docker Compose (partial / full stack)

```bash
docker compose up                           # backend + frontend + postgres
docker compose --profile hold-service up    # + Java hold service
```

Java service is **behind a profile** in [`docker-compose.yml`](../../docker-compose.yml) and must be opted in explicitly.

### AWS (ECS/ALB)

Scripts: [`scripts/aws/deploy-to-aws.sh`](../../scripts/aws/deploy-to-aws.sh)
Terraform: [`scripts/terraform/`](../../scripts/terraform/) — VPC, ECS, ALB, ECR, IAM, CloudWatch.
**Note:** `DATABASE_URL` is intentionally unset on ECS → falls back to `./booking.db` (ephemeral per task). Data does not survive task restarts. Source: [`AGENTS.md`](../../AGENTS.md).

### IBM Cloud (Code Engine)

Scripts: [`scripts/ibm/deploy-to-ibm.sh`](../../scripts/ibm/deploy-to-ibm.sh).

---

## 11. Architectural Invariants & Footguns

All items below are documented in [`AGENTS.md`](../../AGENTS.md) and verified in source.

| # | Invariant | Evidence |
|---|---|---|
| 1 | **`FastApiMCP` is mounted after `FastAPI` and all routes** — `mcp = FastApiMCP(app)` at line 336 wraps the already-constructed app; adding routes after `FastApiMCP` is mounted means they are not exposed as MCP tools | [`server.py:336`](../../booking_system_backend/server.py:336) |
| 2 | **MCP tools bypass FastAPI DI** — use `SessionLocal()` + `db.close()` directly, not `Depends(get_db)` | [`server.py`](../../booking_system_backend/server.py) |
| 3 | **Service functions return `T \| ErrorResponse`, never raise** — callers use `isinstance(result, ErrorResponse)` | [`booking.py`](../../booking_system_backend/services/booking.py) |
| 4 | **`book_flight()` validates both `user_id` AND `name`** — name mismatch rejects booking (intentional) | [`booking.py:51–65`](../../booking_system_backend/services/booking.py:51) |
| 5 | **SQLite is the ECS production database** — data is ephemeral per container task | [`db.py:9`](../../booking_system_backend/db.py:9), [`AGENTS.md`](../../AGENTS.md) |
| 6 | **`SEED_DEMO_DATA=true` calls `seed()`, but `seed()` is guarded** — it exits immediately if `User.count() > 0` ([`seed.py:17`](../../booking_system_backend/seed.py:17)), so existing data is never overwritten. The flag only matters on a fresh (empty) database. | [`server.py:34–36`](../../booking_system_backend/server.py:34), [`seed.py:17`](../../booking_system_backend/seed.py:17) |
| 7 | **`conftest.py` uses `dependency_overrides` — not `SessionLocal` patching** — override `get_db` via `server.app.dependency_overrides[db_module.get_db]`; clearing the overrides after each test is mandatory | [`conftest.py:55–60`](../../booking_system_backend/tests/conftest.py:55) |
| 8 | **Java requires Java 17 or 21** — Lombok does not support Java 22+ | [`AGENTS.md`](../../AGENTS.md) |
| 9 | **Java service is behind a Docker Compose profile** — use `--profile hold-service` | [`docker-compose.yml`](../../docker-compose.yml) |
| 10 | **Python proxy swallows Java 404s as HTTP 200** — check `{"error":"..."}` in response body | [`server.py:242–330`](../../booking_system_backend/server.py:242), [`test_holds.py:82`](../../e2e/test_holds.py:82) |
| 11 | **`holds.db` and `booking.db` are committed artefacts** — do not delete; they seed local dev | [`AGENTS.md`](../../AGENTS.md) |
| 12 | **Lifecycle hooks block destructive actions** — see [`.bob/settings.json`](../../.bob/settings.json); check `.bob/hooks/state/.last-block` when blocked | [`AGENTS.md`](../../AGENTS.md) |

---

## 12. Demo Seed Data

Loaded at startup when `SEED_DEMO_DATA=true` ([`seed.py`](../../booking_system_backend/seed.py)):

- **10 users:** Alice, Bob, Charlie, Diana, Eve, Frank, Grace, Heidi, Ivan, Judy
- **10 flights:** Routes between Earth, Mars, Moon, Venus, Jupiter, Europa, Pluto; base prices 500K–5M credits; departures 2099-01-01 through 2099-01-10; 10 seats each (6 economy / 3 business / 1 galaxium)
- **20 bookings:** Random distribution across users, flights, statuses, and seat classes

---

*Generated by ramp-architecture-mapper. All claims traced to repository files.*
*Last updated: based on repository state at time of generation.*
