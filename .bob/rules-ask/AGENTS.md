# AGENTS.md

This file provides guidance to agents when working with code in this repository.

# Ask Mode — Documentation Context

## Non-obvious structure

- **No frontend unit tests** — ESLint (`npm run lint`) is the only automated frontend check. Do not suggest or look for Jest/Vitest/React Testing Library setups.
- **`/api` is the FastAPI `root_path`** — the app is mounted with `root_path="/api"` for ALB routing. All REST paths shown in Swagger are prefixed `/api` in production but plain (e.g. `/flights`) locally.
- **Frontend API errors are HTTP 200** — the Python proxy endpoints (`/quotes`, `/quotes/{id}/holds`, etc.) catch `httpx.HTTPError` and return `{"error": "..."}` with status 200. Consumers must check the response body, not the status code.
- **Two overlapping filter param sets on `/flights`** — the endpoint accepts both a "main branch" filter set (`sort`, `order`) and a "feature branch" filter set (`sort_by`, `sort_order`, `seat_class`, `departure_time_period`, etc.) for backward compatibility. Both are live and functional.
- **mypy is not clean** — the repo has a pre-existing error baseline at `.bob/hooks/mypy-baseline.txt`. Only *new* errors introduced by an edit are surfaced. Do not interpret baseline errors as bugs to fix.
- **Java hold duration is 15 min** — configured in `application.properties` (`hold.duration.minutes=15`); the scheduler checks every 60 s. If modifying either value, update `e2e/test_holds.py` timeouts accordingly.
- **`holds.db` and `booking.db` are committed** — they are seed artefacts for local dev, not accidental commits. A hook blocks their deletion.
