# Payment System — Auth & Transfer API

![Python](https://img.shields.io/badge/Python-3.11-blue) ![FastAPI](https://img.shields.io/badge/API-FastAPI-teal) ![PostgreSQL](https://img.shields.io/badge/DB-PostgreSQL-336791) ![Redis](https://img.shields.io/badge/Cache-Redis-red) ![Docker](https://img.shields.io/badge/Docker-Compose-blue)

The behind-the-scenes engine for a money-transfer app, focused on keeping people's money **safe** and **accurate**.

> **Status:** work in progress / learning project. Logging in and sending money work and are tested. Fingerprint/Face ID confirmation is started but not finished. See [Roadmap](#roadmap--known-gaps).

---

## What is this?

When you send money in a banking app, a lot happens behind the scenes that you never see. This project builds that hidden part: the **"back end"** that the app talks to. It doesn't have screens or buttons. Its job is to make sure that:

- **Only you can get into your account.**
- **Sending money needs an extra check.** Being logged in isn't enough. You have to confirm it's really you again, right before the payment, the way many banking apps ask for your fingerprint.
- **Money is never lost or created by mistake**, even if thousands of people send money at the same moment.
- **Nobody gets charged twice.** If your internet drops and the app retries, or you tap "Send" twice, the money only moves once.
- **Every payment is permanently recorded**, like a receipt book whose pages can't be torn out.

## How it keeps money safe (in plain words)

| Problem | Real-life example | How this project handles it |
|---|---|---|
| Someone steals your login | A thief copies your session from a public computer | Logins expire quickly and are constantly swapped for new ones. If a stolen, already-used login is ever tried again, the system **logs you out everywhere** to lock the thief out. |
| Sending money while logged in isn't proof enough | You leave your phone unlocked on a table | Payments need a **second, fresh confirmation** that lasts only a few minutes. |
| Two payments at the same moment | You have $100 and two $80 payments arrive at the exact same instant | The system handles **one at a time per account**, so only one goes through and your balance can't go below zero. |
| Tapping "Send" twice | The app freezes and you tap again | Each payment carries a unique ID. If the same ID arrives again, the system replies "already done" instead of paying twice. |
| Rounding errors | $0.10 + $0.20 = $0.30000000004 | Money is stored as **exact decimal amounts**, never approximate numbers. |
| Too many attempts | Someone tries thousands of passwords | **Rate limits** slow down repeated attempts (e.g. at most 5 transfers per minute). |
| Passwords leaking | The database is ever exposed | Passwords are **scrambled with Argon2**, a modern one-way method, so the real password is never stored. |

## Who is this for?

- **Non-technical readers:** the sections above explain what the system does and why. Everything below is for developers who want to run or build on it.
- **Developers:** it's an example of how to build secure logins and money transfers with Python, FastAPI and PostgreSQL.

---

## Features (technical)

- **JWT authentication with three token types**
  - **Access token** (~30 min) for general API access.
  - **Refresh token** (7 days) stored in an `httpOnly` cookie and tracked in the database so it can be revoked.
  - **Step-up token** (~3 min) required on top of a normal session for sensitive actions like transfers.
- **Refresh token rotation with reuse detection:** every refresh issues a new token and revokes the old one. If an already-used token is presented again (a sign of theft), *all* of that user's sessions are revoked.
- **Safe transfers**
  - Row-level locking (`SELECT ... FOR UPDATE`) on the sender and recipient, so concurrent transfers can't corrupt balances.
  - **Idempotency keys:** a retried or double-clicked request returns the original response instead of moving money twice. The idempotency record is committed in the same DB transaction as the transfer.
  - **Immutable transaction ledger** recording the amount and both balances after each transfer.
  - Money is stored as `DECIMAL`, never `float`.
- **Argon2 password hashing** via `pwdlib`.
- **Rate limiting** with `slowapi` (e.g. 5 transfers/min, 10 refreshes/min).
- **Structured JSON logging** with `structlog` and a per-request `X-Request-ID` header.
- **Redis caching and a distributed lock** to prevent cache stampedes on expensive report generation.
- **Alembic migrations** using an async (`asyncpg`) setup.
- **Load testing** with Locust, and **horizontal scaling** behind an Nginx load balancer.

---

## Tech Stack

| Technology | Role |
|---|---|
| Python 3.11 + FastAPI | Async REST API |
| PostgreSQL + SQLAlchemy 2.0 (async) + asyncpg | Persistence, row-level locking |
| Alembic | Schema migrations |
| Redis + aiocache | Caching and distributed locking |
| python-jose | JWT signing and verification |
| pwdlib (Argon2) | Password hashing |
| pydantic-settings | Typed config loaded from `.env` |
| slowapi | Rate limiting |
| structlog | Structured logging |
| Nginx | Load balancer (`least_conn`) in front of the API |
| Docker Compose | Local orchestration |
| pytest + pytest-asyncio + httpx | Unit and integration tests |
| Locust | Load testing |
| uv | Dependency management |

---

## Project Structure

```
payment-system/
├── main.py                  # FastAPI app, router registration, /check_balance, /generate_report, /health
├── db.py                    # Async engine, session, and SQLAlchemy models
├── utils.py                 # structlog config, request-ID middleware, rate limiter
├── authentication/
│   ├── auth.py              # Password hashing, JWT creation/verification, refresh-token storage & revocation, step-up dependency
│   └── settings.py          # pydantic-settings config (SECRET_KEY, token expiry, algorithm)
├── endpoints/
│   ├── login.py             # POST /login
│   ├── refresh.py           # POST /refresh  (rotation + reuse detection)
│   ├── logout.py            # POST /logout
│   ├── transfer.py          # POST /transfer (row locks + idempotency + ledger)
│   ├── verification.py      # Step-up (biometric) verification — in progress
│   └── check_balance.py     # Standalone version of the balance check
├── migrations/              # Alembic environment and versions
├── test/                    # pytest suite (see test/Readme.MD)
├── nginx/nginx.conf         # Load balancer config
├── locustfile.py            # Load test
├── create_tables.py         # Create all tables directly (dev shortcut)
├── drop_tables.py           # Drop auth/transaction tables (dev reset)
├── Documentation.MD         # In-depth technical design documentation
├── docker-compose.yml
├── Dockerfile
└── pyproject.toml / uv.lock
```

---

## Database Schema

| Table | Purpose |
|---|---|
| `user` | `firstname`, `lastname`, `email` (unique), Argon2 `password` hash, `balance` (`DECIMAL(10,2)`) |
| `refresh_tokens` | One row per active refresh token (`jti`), with `expires_at` and `revoked` |
| `step_up_verifications` | Record of recent step-up verifications (`method`: password / biometric / otp) |
| `transactions` | Immutable ledger: `reference`, sender, recipient, `amount`, balances after, `status`, `idempotency_key` |
| `idempotency_records` | Stores the exact response for each idempotency key so retries can replay it |

---

## Getting Started

### Prerequisites

- Python 3.11+
- [uv](https://docs.astral.sh/uv/)
- Docker and Docker Compose

### 1. Clone the repository

```bash
git clone https://github.com/Aniekan-Udo/payment-system.git
cd payment-system
```

### 2. Create a `.env` file

```ini
# Auth
SECRET_KEY=change-me-to-a-long-random-string
ACCESS_TOKEN_EXPIRE_MINUTES=30

# Postgres (used by docker-compose)
POSTGRES_USER=test
POSTGRES_PASSWORD=test
POSTGRES_DB=test

# App connection strings
POSTGRES_URI=postgresql+asyncpg://test:test@localhost:5434/test
REDIS_HOST=localhost
LIMITER_STORAGE=memory://
```

Generate a strong secret with `python -c "import secrets; print(secrets.token_urlsafe(64))"`. Never commit `.env`; it's already in `.gitignore`.

### 3. Start Postgres and Redis

```bash
docker compose up -d postgres redis
```

Postgres is exposed on `localhost:5434` and Redis on `localhost:6380`. If you run the API outside Docker, either set `REDIS_HOST`/port to match or map Redis to `6379`.

### 4. Install dependencies and run migrations

```bash
uv sync
uv run alembic upgrade head
```

(For a quick dev setup you can instead run `uv run python create_tables.py`.)

### 5. Run the API

```bash
uv run uvicorn main:app --reload --port 8001
```

Interactive API docs are then available at <http://localhost:8001/docs>.

### Running the full stack with Docker

```bash
docker compose up --build
```

This starts Postgres, Redis, the API and Nginx. The API is reachable through Nginx at <http://localhost:8002>. To test load balancing across several API instances:

```bash
docker compose up --build --scale api=3
```

---

## API Endpoints

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/login` | POST | — | Verifies email and password; returns an access token and sets the `refresh_token` cookie |
| `/refresh` | POST | Refresh cookie | Rotates the refresh token and returns a new access token. Reusing an old token revokes all sessions |
| `/logout` | POST | Refresh cookie | Revokes the current refresh token and clears the cookie |
| `/transfer` | POST | Access + step-up token | Transfers money to another user, with idempotency and a ledger entry |
| `/check_balance` | POST | — | Looks up a user's balance (cached in Redis for 60s) |
| `/generate_report` | POST | — | Demo of a Redis-lock-protected expensive operation |
| `/health` | GET | — | Liveness check; returns the hostname (useful to see load balancing) |

### Example: login

```bash
curl -X POST http://localhost:8001/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"email": "johndoe@example.com", "password": "secret"}'
```

```json
{ "access_token": "eyJhbGciOi...", "token_type": "bearer" }
```

### Example: transfer

```json
POST /transfer
Authorization: Bearer <access_token>

{
  "recipient_email": "jane@example.com",
  "amount": "25.00",
  "idempotency_key": "b3f1c2e0-7a4d-4c1e-9f7a-2d8e1f0a9c11"
}
```

```json
{
  "message": "Transfer successful",
  "reference": "TXN-4f9c...",
  "amount": "25.00",
  "recipient": "jane@example.com"
}
```

Sending the same `idempotency_key` again returns this same response without moving money a second time.

---

## Running Tests

The tests need a **separate** Postgres database, because their tables are created and dropped automatically.

```bash
export TEST_DATABASE_URL="postgresql+asyncpg://test:test@localhost:5434/test_db"
uv run pytest test/ -v
```

The suite covers:

- password hashing and token creation/verification (unit tests)
- refresh-token storage, rotation and reuse detection
- the full login → refresh → logout flow
- transfers: success, insufficient balance, unknown recipient, idempotent retries, and a **concurrency test** that fires two transfers at once to prove row locking prevents lost updates

See [`test/Readme.MD`](test/Readme.MD) for details.

### Load testing

```bash
uv run locust -f locustfile.py --host http://localhost:8002
```

Then open <http://localhost:8089> to start a test.

---

## Database Migrations

```bash
uv run alembic revision --autogenerate -m "describe your change"
# review the generated file in migrations/versions/ before applying
uv run alembic upgrade head
```

---

## Roadmap / Known Gaps

- [ ] Platform biometric step-up verification (Apple App Attest / Google Play Integrity), plus a password/OTP fallback. The `/verify_setup` route is scaffolded but `verify_platform_assertion` isn't implemented yet.
- [ ] Lock sender and recipient rows in a stable order (by user ID) to rule out deadlocks between opposite-direction transfers.
- [ ] Move `/check_balance` and `/generate_report` to token-based identity (`get_current_user`) instead of trusting fields in the request body.
- [ ] Compare-and-delete when releasing the Redis report lock.
- [ ] `GET /transactions` endpoint for transaction history.
- [ ] Set `secure=True` on the refresh cookie for production (HTTPS).

For the full design rationale (token lifecycles, locking strategy, idempotency and security notes), see [`Documentation.MD`](Documentation.MD).
