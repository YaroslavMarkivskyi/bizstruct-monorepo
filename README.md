# BizStruct

A system that turns a business idea into a structured business-model canvas with LLM-generated, validated rows.

| Directory | Role |
|---|---|
| `bizstruct-be` | FastAPI backend (api), PostgreSQL, Alembic migrations |
| `bizstruct-fe` | Next.js web client |
| `bizstruct-ml` | Worker that consumes generation tasks from Azure Service Bus and calls the LLM |
| `bizstruct-domain` | Shared domain schemas (a Python library; `be` and `ml` install it from GitHub, tag `v0.18.0`) |
| `bizstruct-docs` | Methodology and technical decisions |

Each component directory is a snapshot of its own repository (`github.com/bizstruct/bizstruct-<name>`).

## Quick start (db + api + web)

Requirements: Docker with the Compose plugin (v2.20+) and internet access (the images install `bizstruct-domain` from GitHub).

```bash
cp .env.example .env                              # set POSTGRES_PASSWORD and INTERNAL_API_KEY
cp bizstruct-be/.env.example bizstruct-be/.env    # works as is; Azure settings stay empty
docker compose up -d --build                      # first build takes a few minutes
```

| Service | URL |
|---|---|
| Web | http://127.0.0.1:3001 |
| API docs (Swagger) | http://127.0.0.1:18000/docs |
| PostgreSQL | 127.0.0.1:15432 |

Ports can be changed in `.env` (`FE_PORT`, `API_PORT`, `DB_PORT`). The api runs `alembic upgrade head` on every start.

**Create a user and log in** (there is no seeded account):

```bash
curl -X POST http://127.0.0.1:18000/api/users/ -H 'content-type: application/json' \
  -d '{"email":"demo@example.com","password":"Demo12345!","full_name":"Demo User"}'

# login takes form fields: username + password
curl -X POST http://127.0.0.1:18000/api/auth/login \
  -d 'username=demo@example.com&password=Demo12345!'
```

Then sign in on the web page with the same email and password.

## Generation (ml worker, needs Azure and an LLM key)

Generating a canvas needs the `ml` worker, which reads tasks from an Azure Service Bus queue and calls an LLM. It is a separate compose profile because it cannot run without real credentials. Without it the api, the web client, users and projects work, but generation tasks are only logged.

1. `cp bizstruct-ml/.env.example bizstruct-ml/.env` and fill in `SERVICE_BUS_CONNECTION_STRING`, `OPENAI_API_KEY` (or the `AZURE_OPENAI_*` variables).
2. Put the same `SERVICE_BUS_CONNECTION_STRING` into `bizstruct-be/.env`, and keep `SERVICE_BUS_QUEUE_NAME` equal in the root `.env`.
3. `docker compose --profile ml up -d --build`

## Configuration

- Root `.env`: `POSTGRES_*`, the shared `INTERNAL_API_KEY`, `SERVICE_BUS_QUEUE_NAME`, host ports. Values passed by compose override the service `.env` files.
- `bizstruct-be/.env`: JWT and CORS settings, Service Bus and Web PubSub connection strings (both optional: empty means no-op).
- `bizstruct-ml/.env`: Service Bus, LLM and judge credentials.

## Commands

```bash
docker compose logs -f api web        # logs
docker compose down                   # stop, keep the database
docker compose down -v                # stop and delete the database volume
```

## Tests

```bash
cd bizstruct-be && uv sync && uv run pytest          # needs PostgreSQL 15 (see its README)
cd bizstruct-ml && uv sync && uv run pytest
cd bizstruct-domain && uv sync && uv run pytest
cd bizstruct-fe && pnpm install && pnpm test
```
