# AGENTS.md

This file provides guidance to agents when working with code in this repository.

# Plan Mode — Architecture Constraints

## Non-obvious architectural decisions

- **SQLite is the intentional production DB** — `DATABASE_URL` is deliberately unset on ECS; [`db.py`](booking_system_backend/db.py) defaults to `sqlite:///./booking.db`. Data is ephemeral per container task by design.
- **MCP tools are architecturally separate from FastAPI DI** — MCP tools call `SessionLocal()` directly and must never use `Depends(get_db)`. This means session lifecycle for MCP calls is manually managed; there is no middleware layer.
- **Holds flow is cross-service, not in-process** — a "hold" lives entirely in the Java service; Python only knows about it when Java calls `/internal/bookings/from-hold` at confirmation time. The Python backend has no hold state.
- **Python proxy swallows Java 404s as HTTP 200** — proxy endpoints catch `httpx.HTTPError` broadly. Any new proxy endpoint must document this in its callers; do not assume HTTP status reflects success.
- **Service layer is stateless and I/O-free** — `services/{booking,flight,user}.py` perform only DB queries; no HTTP calls, no file I/O, no side effects beyond DB writes. Keep this invariant when adding service functions.
- **FastMCP + FastAPI lifespan coupling** — `FastMCP` wraps `FastAPI` and composes lifespans. Order of instantiation in [`server.py`](booking_system_backend/server.py) is load-bearing; do not restructure the module-level setup.
- **Java scheduler runs every 60 s; hold TTL is 15 min** — these are independent knobs (`application.properties`). Changing either requires a corresponding update to the e2e slow test timeout in `test_holds.py`.
- **`book_flight()` is the single booking entry point for both REST and the Java callback** — `/book` (REST) and `/internal/bookings/from-hold` (Java callback) both call `booking.book_flight()`. Side effects (seat decrement, validation) happen exactly once, in one place.
- **Error response shape is a first-class contract** — `ErrorResponse` (`success`, `error`, `error_code`, `details`) is used identically in Python schemas, frontend TypeScript types, and the axios interceptor. Changing the shape breaks all three.
