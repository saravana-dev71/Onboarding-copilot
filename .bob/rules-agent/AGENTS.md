# AGENTS.md

This file provides guidance to agents when working with code in this repository.

# Agent Mode — Coding Rules

## Non-obvious patterns

- **Service layer never raises** — all error cases return `ErrorResponse`; REST endpoints map `error_code` → HTTP status with a manual `isinstance` check. Mirroring this pattern is required for new endpoints.
- **MCP tools bypass FastAPI DI** — MCP tool functions in [`server.py`](booking_system_backend/server.py) call `SessionLocal()` + `db.close()` directly. Do NOT use `Depends(get_db)` inside an MCP tool.
- **FastMCP must be instantiated before FastAPI** — in [`server.py`](booking_system_backend/server.py), the `FastMCP` object is created at module level before `FastAPI()`. Reversing this breaks lifespan composition silently.
- **Pydantic schemas require `model_config = ConfigDict(from_attributes=True)`** — omitting this breaks `.model_validate(orm_obj)` calls from service functions.
- **`book_flight()` requires both `user_id` AND matching `name`** — the name check in [`booking.py`](booking_system_backend/services/booking.py) is intentional security; a name mismatch returns `NAME_MISMATCH`, not `USER_NOT_FOUND`.
- **`SEED_DEMO_DATA=true` re-seeds on every startup** — set to `false` when data persistence across restarts matters locally.
- **Tests patch `SessionLocal` in TWO modules** — [`conftest.py` lines 49–50](booking_system_backend/tests/conftest.py:49) patches both `db.SessionLocal` and `server.SessionLocal`. Patching only one leaves MCP tools hitting the real DB.

## Test commands

```bash
# All backend tests (must run from booking_system_backend/)
cd booking_system_backend && .venv/bin/python -m pytest -v --tb=short

# Single test
cd booking_system_backend && .venv/bin/python -m pytest tests/test_rest.py::TestBookEndpoint::test_book_flight_success -v

# Single test file
cd booking_system_backend && .venv/bin/python -m pytest tests/test_services.py -v

# Java unit tests (requires Java 17 or 21 — Lombok incompatible with 22+)
cd booking_system_inventory_hold_service && mvn test -q

# Frontend lint (no unit tests exist)
cd booking_system_frontend && npm run lint

# e2e native (manages its own service lifecycle)
./e2e/run-native.sh
E2E_RUN_SLOW=1 ./e2e/run-native.sh   # includes ~90 s auto-expiry test
```

## Hooks that affect coding

- **mypy runs after every Python file write** — new type errors are injected at the top of the next prompt (pre-existing baseline in `.bob/hooks/mypy-baseline.txt` is suppressed). Fix them proactively.
- **`git commit` is gated** — blocked if backend pytest suite is red. Read `.bob/hooks/state/.last-block` for details.
- **Ruff ignores B008** — `Depends()` in FastAPI function signatures is intentional; do not refactor it out.

## Tailwind custom tokens (do not use standard names outside this list)

`space-dark` `space-blue` `cosmic-purple` `nebula-pink` `alien-green` `solar-orange` `star-white`
Gradients: `bg-space-gradient` `bg-cosmic-gradient`
Animations: `animate-float` `animate-twinkle`

## Do not touch

- `booking_system_inventory_hold_service/target/` — Maven build output
- `booking_system_frontend/dist/` — Vite build output
- `scripts/terraform/.terraform/` — provider binaries
- `holds.db`, `booking.db` — committed seed artefacts; hook blocks deletion
