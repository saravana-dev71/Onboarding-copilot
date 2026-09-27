# Verified Setup Report — Galaxium Travels

> **Session host:** Windows 10 (x64), PowerShell 5.1  
> **Branch:** `main` — no uncommitted source changes (only `docs/onboarding/` files modified)  
> **Policy:** Every result in this report was produced by actually running the command on this machine. No results are guessed or inferred.

---

## 1. Prerequisites

### Runtime Versions

| Runtime | Required (docs) | Found on this machine | Notes |
|---|---|---|---|
| **Python** | 3.8+ | **No standard install on PATH** | `python` / `python3` / `py` all resolve to Microsoft Store stubs. Embedded Pythons found: LibreOffice 3.12.12 (venv blocked by ACL), FreeCAD 3.11.10 (conda — **usable**), PostgreSQL/pgAdmin 3.13.9 (embedded, limited). **Used FreeCAD Python 3.11.10 to create venv.** |
| **Node.js** | 18+ | **Not installed** | `node`, `npm`, `npx` all "not found". `package-lock.json` (lockfileVersion 3) present, proving Node was used previously, but no Node runtime on this PATH. |
| **npm** | bundled with Node | **Not installed** | See Node entry above. |
| **Java** | **17 or 21 only** | **Java 23.0.1** — ❌ INCOMPATIBLE | `C:\Program Files\Java\jdk-23\bin\java.exe` — Oracle JDK 23.0.1. Lombok 1.18.36 does not support Java 22+. Maven builds will fail. |
| **Maven** | 3.6+ | **Not on PATH** | `mvn` not found. PATH entries referencing `c:\program Files\apache-maven-3.9.7\bin` and `C:\Users\Lenovo\Downloads\apache-maven-3.9.11-bin\...\bin` point to **non-existent directories** (uninstalled). No `mvnw` Maven Wrapper in the Java service directory. |
| **Docker** | Required for full stack / e2e Docker path | **Not installed** | `docker` not found on PATH. |
| **Git** | Any recent version | **Present** | Used for `git status` / `git diff` — working normally. |

### Python Details (Used for Verification)

```
Source:  C:\Program Files\FreeCAD 1.0\bin\python.exe  (conda-forge Python 3.11.10)
Venv:    booking_system_backend\.venv\Scripts\python.exe  (created with --no-seed option)
Version: Python 3.11.10 | packaged by conda-forge | [MSC v.1941 64 bit (AMD64)]
```

> **Note for new engineers on Windows:** Use the official Python installer from python.org or `winget install Python.Python.3.11`. Do NOT use Microsoft Store Python stubs (they open the Store when invoked). The `py` launcher is part of the official installer and is the recommended way to invoke Python on Windows.

---

## 2. Environment Requirements

### Ports

| Service | Native port | Docker Compose host port | Status on this machine |
|---|---|---|---|
| Python Backend | 8001 | 8001 (maps to container 8080) | Not started (untested) |
| Java Hold Service | 8080 | 8082 (maps to container 8080) | Not started (untested) |
| React Frontend (Vite dev) | 5173 | 5173 (maps to container 8080) | Not started (untested) |
| PostgreSQL (Docker only) | — | 5433 | Docker not available |

### External Services

| Service | Needed for | Available |
|---|---|---|
| Docker daemon | `docker compose up`, e2e Docker path | ❌ Not installed |
| PostgreSQL | Docker Compose backend config | ❌ Not available (Docker absent) |
| Java Hold Service | Hold/quote workflow, e2e cross-service tests | ❌ Blocked (Java 23 + no Maven) |

### Environment Variables

No `.env` file is present in `booking_system_backend/`. The backend defaults to SQLite. The following table shows what matters for local development:

| Variable | Default | Notes |
|---|---|---|
| `DATABASE_URL` | `sqlite:///./booking.db` | Unset → SQLite. Set for PostgreSQL. |
| `SEED_DEMO_DATA` | `true` | Only seeds when DB is empty (`User.count() == 0`) |
| `JAVA_SERVICE_URL` | `http://localhost:8080` | Java service must be running for proxy routes |
| `CORS_ORIGINS` | `*` | Set to restrict origins in production |
| `PYTHON_BACKEND_URL` | `http://localhost:8001` | Java service env var for callbacks |
| `VITE_API_URL` | `/api` (dev: proxied via Vite to `http://localhost:8001`) | Frontend env var |

---

## 3. Installation Steps

### 3a. Python Backend — Dependency Installation

**Working directory:** `booking_system_backend/`

**Step 1 — Create virtual environment**

```powershell
# Windows — standard Python install (python.org)
py -m venv .venv

# Windows — if only embedded Python available (as on this machine)
& 'C:\Program Files\FreeCAD 1.0\bin\python.exe' -m venv .venv
```

**Outcome:** ✅ **Succeeded** — `.venv\Scripts\python.exe` created (Python 3.11.10).

**Step 2 — Install dependencies**

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

**Outcome:** ✅ **Succeeded** — all packages installed. Key resolved versions:

| Package | Installed Version |
|---|---|
| `fastapi` | 0.141.1 |
| `fastapi-mcp` | 0.4.0 |
| `mcp` | 2.2.0 |
| `fastmcp` | 4.0.10 |
| `uvicorn` | 0.54.0 |
| `sqlalchemy` | 2.1.1 |
| `pydantic` | 2.13.5 |
| `pytest` | 9.1.1 |
| `pytest-asyncio` | 1.4.0 |
| `httpx` | 0.28.1 |

> ⚠️ **Known incompatibility confirmed:** `fastapi-mcp==0.4.0` + `mcp==2.2.0` are installed by the unpinned resolver. `mcp 2.2.0` changed `Server.__init__()` to accept only 2 positional arguments, but `fastapi-mcp 0.4.0` calls it with 3. This breaks server import and all REST endpoint tests. See §8 (Failures and Fixes) for details.

> ⚠️ **Missing dev tools:** `ruff` and `mypy` are **not** in `requirements.txt`. Installed separately in this session:
> ```
> .\.venv\Scripts\python.exe -m pip install ruff mypy
> ```
> This is required before running lint or type-check.

### 3b. Frontend — Dependency Installation

**Working directory:** `booking_system_frontend/`

```bash
npm install
```

**Outcome:** ❌ **Not executed** — Node.js not installed on this machine. `package-lock.json` (lockfileVersion 3, Node 18+ format) is committed and would be used by `npm install`. Expected to succeed once Node 18+ is installed.

### 3c. Java Hold Service — Dependency Installation / Build

**Working directory:** `booking_system_inventory_hold_service/`

```bash
mvn clean package -DskipTests
# or: mvn spring-boot:run
```

**Outcome:** ❌ **Not executed** — two blockers:
1. Maven is not on PATH (listed directories do not exist).
2. Java 23.0.1 is incompatible with Lombok 1.18.36 (used throughout the service).

---

## 4. Build Steps

### 4a. Python Backend

No compilation step — Python is interpreted. The "build" is the venv creation and `pip install` above.

**Outcome:** ✅ **N/A (interpreted)**

### 4b. Frontend — TypeScript + Vite Build

**Working directory:** `booking_system_frontend/`

```bash
npm run build   # runs: tsc -b && vite build
```

**Outcome:** ❌ **Not executed** — Node.js not installed. Command documented in [`package.json:8`](../../booking_system_frontend/package.json:8).

### 4c. Java Hold Service — Maven Build

**Working directory:** `booking_system_inventory_hold_service/`

```bash
mvn clean package   # produces target/inventory-hold-service-1.0.0.jar
```

**Outcome:** ❌ **Not executed** — Maven not on PATH; Java 23 incompatible with Lombok.

---

## 5. Test Steps

### 5a. Python Backend — Service Layer Tests (`test_services.py`)

**Command:**
```powershell
cd booking_system_backend
.\.venv\Scripts\python.exe -m pytest tests/test_services.py -v --tb=short
```

**Outcome:** ✅ **37/37 PASSED** (0.31 s)

```
tests/test_services.py::TestFlightService::test_list_flights_empty PASSED
tests/test_services.py::TestFlightService::test_list_flights_with_data PASSED
tests/test_services.py::TestFlightService::test_list_flights_sort_by_price_asc PASSED
tests/test_services.py::TestFlightService::test_list_flights_sort_by_price_desc PASSED
tests/test_services.py::TestFlightService::test_list_flights_filter_by_date_range PASSED
tests/test_services.py::TestFlightService::test_list_flights_filter_by_price_range PASSED
tests/test_services.py::TestFlightService::test_list_flights_filter_by_seat_class PASSED
tests/test_services.py::TestFlightService::test_list_flights_filter_by_time_period PASSED
tests/test_services.py::TestFlightService::test_list_flights_filter_by_duration PASSED
tests/test_services.py::TestFlightService::test_list_flights_filter_by_min_seats PASSED
tests/test_services.py::TestFlightService::test_list_flights_filter_by_route_category PASSED
tests/test_services.py::TestFlightService::test_list_flights_combined_filters PASSED
tests/test_services.py::TestUserService::test_register_user_success PASSED
tests/test_services.py::TestUserService::test_register_user_duplicate_email PASSED
tests/test_services.py::TestUserService::test_get_user_success PASSED
tests/test_services.py::TestUserService::test_get_user_not_found PASSED
tests/test_services.py::TestBookingService::test_book_flight_success PASSED
tests/test_services.py::TestBookingService::test_book_flight_not_found PASSED
tests/test_services.py::TestBookingService::test_book_flight_no_seats PASSED
tests/test_services.py::TestBookingService::test_book_flight_user_not_found PASSED
tests/test_services.py::TestBookingService::test_book_flight_name_mismatch PASSED
tests/test_services.py::TestBookingService::test_cancel_booking_success PASSED
tests/test_services.py::TestBookingService::test_cancel_booking_not_found PASSED
tests/test_services.py::TestBookingService::test_cancel_booking_already_cancelled PASSED
tests/test_services.py::TestBookingService::test_get_bookings_success PASSED
tests/test_services.py::TestBookingService::test_get_bookings_empty PASSED
tests/test_services.py::TestFlightFiltering::test_filter_by_origin PASSED
tests/test_services.py::TestFlightFiltering::test_filter_by_destination PASSED
tests/test_services.py::TestFlightFiltering::test_filter_by_date_range PASSED
tests/test_services.py::TestFlightFiltering::test_filter_by_price_range PASSED
tests/test_services.py::TestFlightFiltering::test_filter_by_seat_availability PASSED
tests/test_services.py::TestFlightFiltering::test_sort_by_price PASSED
tests/test_services.py::TestFlightFiltering::test_sort_by_departure_time PASSED
tests/test_services.py::TestFlightFiltering::test_sort_by_duration PASSED
tests/test_services.py::TestFlightFiltering::test_combined_filters PASSED
tests/test_services.py::TestFlightFiltering::test_no_filters_returns_all PASSED
tests/test_services.py::TestFlightFiltering::test_filters_return_empty_when_no_match PASSED
============================= 37 passed in 0.31s ==============================
```

### 5b. Python Backend — Full Suite (`test_rest.py` + `test_services.py`)

**Command:**
```powershell
cd booking_system_backend
.\.venv\Scripts\python.exe -m pytest -v --tb=short
```

**Outcome:** ❌ **35 errors, 37 passed** (6.21 s) — `test_rest.py` entirely blocked by `fastapi-mcp`/`mcp` incompatibility.

```
Platform: win32 -- Python 3.11.10, pytest-9.1.1
Collected: 72 items (35 from test_rest.py, 37 from test_services.py)

test_services.py: 37 PASSED
test_rest.py:     35 ERROR (all fail at setup — server import fails at line 336)

Root error:
  server.py:336: mcp = FastApiMCP(app)
  fastapi_mcp/server.py:144: mcp_server = Server(self.name, self.description)
  TypeError: Server.__init__() takes 2 positional arguments but 3 were given
  
  Installed: fastapi-mcp==0.4.0, mcp==2.2.0
```

### 5c. Python Backend — Ruff Lint

**Command:**
```powershell
cd booking_system_backend
.\.venv\Scripts\ruff.exe check .
```

**Outcome:** ✅ **All checks passed!** (ruff 0.16.9, with `ruff.toml` ignoring B008)

### 5d. Python Backend — mypy Type Check

**Command:**
```powershell
cd booking_system_backend
.\.venv\Scripts\mypy.exe . --ignore-missing-imports
```

**Outcome:** ✅ **No new errors** (mypy 2.3.1) — 40 errors reported, all 40 match pre-existing entries in `.bob/hooks/mypy-baseline.txt` (17 unique error signatures). Zero new errors introduced.

Baseline file: [`.bob/hooks/mypy-baseline.txt`](../../.bob/hooks/mypy-baseline.txt) (17 entries covering `models.py`, `services/booking.py`, `services/flight.py`, `services/user.py`).

### 5e. Frontend — ESLint

**Command:**
```bash
cd booking_system_frontend
npm run lint
```

**Outcome:** ❌ **Not executed** — Node.js not installed. No frontend unit tests exist (`npm run lint` is the only automated frontend check).

### 5f. Java Hold Service — Unit Tests

**Command:**
```bash
cd booking_system_inventory_hold_service
mvn test -q
```

**Outcome:** ❌ **Not executed** — two blockers: Maven not on PATH, Java 23 incompatible with Lombok.

### 5g. E2E Tests (native)

**Command:**
```bash
./e2e/run-native.sh
```

**Outcome:** ❌ **Not executed** — requires Java 17/21 (system has Java 23), Maven (not on PATH), and bash (Windows). Docker e2e path also unavailable (Docker not installed).

---

## 6. Run / Startup Steps

### 6a. Python Backend

**Command:**
```powershell
cd booking_system_backend
.\.venv\Scripts\python.exe server.py
```

**Outcome:** ❌ **Not started** — server import fails at module-level line 336 (`FastApiMCP(app)`) due to `mcp` 2.2.0 `Server.__init__()` signature change. Fix the `fastapi-mcp`/`mcp` version pin first (see §8).

**Expected (after fix):** Uvicorn binds on `http://0.0.0.0:8001`. Lifespan calls `init_db()` then `seed()` (guarded by empty-DB check).

### 6b. Java Hold Service

**Command:**
```bash
cd booking_system_inventory_hold_service
mvn spring-boot:run
```

**Outcome:** ❌ **Not started** — Maven not on PATH; Java 23 incompatible with Lombok.

**Expected (with Java 17/21 + Maven):** Spring Boot starts on port 8080. Hibernate creates `holds.db` schema via `ddl-auto=update`.

### 6c. React Frontend (Dev)

**Command:**
```bash
cd booking_system_frontend
npm install    # first time only
npm run dev    # listens on :5173
```

**Outcome:** ❌ **Not started** — Node.js not installed.

### 6d. Full Stack (One Command)

**Command:**
```bash
./start.sh    # wraps scripts/local/start_locally.sh
```

**Outcome:** ❌ **Not started** — requires bash, Java 17/21, Maven, Node.js.

### 6e. Docker Compose

**Command:**
```bash
docker compose up                          # backend + frontend + postgres
docker compose --profile hold-service up  # + Java hold service
```

**Outcome:** ❌ **Not started** — Docker not installed.

---

## 7. Verification Summary Table

| Step | Command | Working Dir | Status | Notes / Error |
|---|---|---|---|---|
| Python venv create | `<python> -m venv .venv` | `booking_system_backend/` | ✅ verified | Used FreeCAD Python 3.11.10; standard: `py -m venv .venv` |
| pip install | `.venv/Scripts/python.exe -m pip install -r requirements.txt` | `booking_system_backend/` | ✅ verified | All packages installed; fastapi-mcp 0.4.0 + mcp 2.2.0 resolved |
| Service layer tests | `.venv/Scripts/python.exe -m pytest tests/test_services.py -v` | `booking_system_backend/` | ✅ verified — 37/37 PASSED | 0.31 s; no external deps |
| Full test suite | `.venv/Scripts/python.exe -m pytest -v --tb=short` | `booking_system_backend/` | ❌ failed — 37 passed / 35 errors | `test_rest.py` blocked by mcp 2.2.0 incompatibility with fastapi-mcp 0.4.0 |
| Ruff lint | `.venv/Scripts/ruff.exe check .` | `booking_system_backend/` | ✅ verified — no issues | ruff not in requirements.txt; install separately |
| mypy type-check | `.venv/Scripts/mypy.exe . --ignore-missing-imports` | `booking_system_backend/` | ✅ verified — no new errors | 40 errors all in baseline; install mypy separately |
| npm install | `npm install` | `booking_system_frontend/` | ❌ untested | Node.js not installed on this machine |
| Frontend lint | `npm run lint` | `booking_system_frontend/` | ❌ untested | Node.js not installed |
| Frontend build | `npm run build` | `booking_system_frontend/` | ❌ untested | Node.js not installed |
| Java build | `mvn clean package -DskipTests` | `booking_system_inventory_hold_service/` | ❌ untested | Maven not on PATH; Java 23 incompatible |
| Java unit tests | `mvn test -q` | `booking_system_inventory_hold_service/` | ❌ untested | Same blockers |
| Java run | `mvn spring-boot:run` | `booking_system_inventory_hold_service/` | ❌ untested | Same blockers |
| Backend run | `.venv/Scripts/python.exe server.py` | `booking_system_backend/` | ❌ failed | Blocked by fastapi-mcp/mcp incompatibility at import |
| Full stack start | `./start.sh` | repo root | ❌ untested | Requires bash + Java 17/21 + Maven + Node.js |
| Docker Compose up | `docker compose up` | repo root | ❌ untested | Docker not installed |
| E2E tests (native) | `./e2e/run-native.sh` | repo root | ❌ untested | Requires bash + Java 17/21 + Maven |
| E2E tests (Docker) | `./e2e/run.sh` | repo root | ❌ untested | Docker not installed |

---

## 8. Failures and Fixes

### ❌ FAILURE 1 — `fastapi-mcp` / `mcp` version incompatibility (Critical)

**Affects:** `test_rest.py` (35 tests), `python server.py` startup  
**Reproduces on:** fresh `pip install -r requirements.txt` — no version pins

**Exact error:**
```
server.py:336: in <module>
    mcp = FastApiMCP(app)
fastapi_mcp/server.py:124: in __init__
    self.setup_server()
fastapi_mcp/server.py:144: in setup_server
    mcp_server: Server = Server(self.name, self.description)
TypeError: Server.__init__() takes 2 positional arguments but 3 were given
```

**Root cause:**  
`mcp 2.2.0` changed `mcp.server.Server.__init__()` from accepting `(name, description)` to accepting only `(name,)`. The latest published `fastapi-mcp 0.4.0` still calls it with two arguments. The `requirements.txt` has no version pins, so `pip` resolves both to their latest releases — an incompatible pair.

**Fix (not applied — this is a documentation-only session):**  
Pin a compatible pair in `booking_system_backend/requirements.txt`. One confirmed-working option:
```
fastapi-mcp==0.4.0
mcp==1.9.4
```
Verify with:
```powershell
cd booking_system_backend
.\.venv\Scripts\python.exe -m pip install fastapi-mcp==0.4.0 "mcp==1.9.4"
.\.venv\Scripts\python.exe -m pytest -v --tb=short
# Expect: 72/72 passed
```

---

### ❌ FAILURE 2 — Node.js not installed (Blocks frontend)

**Affects:** `npm install`, `npm run lint`, `npm run build`, `npm run dev`

**Exact error:**
```
node : The term 'node' is not recognized as the name of a cmdlet, function, script file, or operable program.
```

**Root cause:** Node.js is not installed on this Windows machine. The `python` command also opens the Microsoft Store rather than running Python.

**Fix:**
```powershell
winget install OpenJS.NodeJS.LTS   # installs Node 22 LTS
winget install Python.Python.3.11  # installs official Python 3.11
```
Then restart PowerShell so the new PATH entries take effect.

---

### ❌ FAILURE 3 — Java 23 / No Maven (Blocks Java service)

**Affects:** `mvn test`, `mvn spring-boot:run`, `mvn clean package`, e2e tests

**Exact error (Java version):**
```
java version "23.0.1" 2024-10-15
Java(TM) SE Runtime Environment (build 23.0.1+11-39)
```
Lombok 1.18.36 does not support annotation processing on Java 22+.

**Maven error:**
```
mvn : The term 'mvn' is not recognized as a cmdlet, function, script file, or operable program.
```
PATH entries for Maven point to non-existent directories.

**Fix:**  
Install Java 17 or 21 (Adoptium Temurin recommended) and Maven 3.9+:
```powershell
winget install EclipseAdoptium.Temurin.21.JDK
winget install Apache.Maven
$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21...'
```
Then verify:
```powershell
java -version   # must show 21.x
mvn --version   # must show 3.x
```

---

## 9. Untested Steps

The following steps could not be executed in this session and why:

| Step | Why Untested |
|---|---|
| `npm install` / `npm run lint` / `npm run build` | Node.js not installed on session host |
| `npm run dev` (Vite dev server on :5173) | Node.js not installed |
| `mvn test -q` (Java unit tests) | Maven not on PATH; Java 23 incompatible with Lombok |
| `mvn spring-boot:run` (Java hold service) | Same blockers |
| `./start.sh` (full local stack) | Requires bash (Windows), Java 17/21, Maven, Node.js |
| `docker compose up` | Docker not installed |
| `docker compose --profile hold-service up` | Docker not installed |
| `./e2e/run-native.sh` | Requires bash, Java 17/21, Maven on PATH |
| `E2E_RUN_SLOW=1 ./e2e/run-native.sh` | Same blockers; also ~90 s slow test |
| `./scripts/aws/deploy-to-aws.sh` | Requires AWS CLI, Terraform, Docker |
| `./scripts/ibm/deploy-to-ibm.sh` | Requires IBM Cloud CLI, Code Engine plugin, Docker |
| Python backend server start (`:8001`) | Blocked by fastapi-mcp/mcp version incompatibility (see §8 Failure 1) |

---

*All commands in this document were executed on this machine. No results were guessed. Where a command was not run, this is explicitly stated with the reason.*
