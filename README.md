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

Two things are needed: a message queue and an LLM key.

### Variables to put into the `.env` files

| File | Variable | Value |
|---|---|---|
| `bizstruct-ml/.env` | `OPENAI_API_KEY` | An OpenAI API key (`sk-...`). Required unless you use Azure OpenAI. |
| `bizstruct-ml/.env` | `OPENAI_MODEL` | Optional, default `gpt-4o`. |
| `bizstruct-ml/.env` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_DEPLOYMENT` | Optional alternative to OpenAI. If the endpoint is set, Azure takes priority. |
| `bizstruct-ml/.env` | `SERVICE_BUS_CONNECTION_STRING` | Only with real Azure Service Bus (variant B). Ignored with the emulator. |
| `bizstruct-be/.env` | `SERVICE_BUS_CONNECTION_STRING` | Same value as in `bizstruct-ml/.env`. Only with real Azure (variant B). |
| root `.env` | `SERVICE_BUS_QUEUE_NAME` | Queue name, `generation-tasks` by default. Must match the queue that exists. |
| root `.env` | `INTERNAL_API_KEY`, `POSTGRES_PASSWORD` | Any secret strings; the same key is used by the api and the worker. |

Not needed: Azure Web PubSub (the worker does not use it; an empty value in `bizstruct-be/.env` means no push events), Langfuse (optional tracing), and the judge variables (compose sets `JUDGE_PROVIDER=fake`, so the consistency judge is stubbed).

Start from `cp bizstruct-ml/.env.example bizstruct-ml/.env` and set `OPENAI_API_KEY`.

### Variant A: local Service Bus emulator (no Azure account)

```bash
docker compose -f docker-compose.yml -f docker-compose.emulator.yml --profile ml up -d --build
```

This adds the Microsoft Service Bus emulator and Azure SQL Edge, creates the `generation-tasks` queue from `servicebus-emulator/Config.json` and points the api and the worker at it. `SERVICE_BUS_CONNECTION_STRING` is set by the override file, so leave it as is. Using the emulator means accepting Microsoft's license terms (`ACCEPT_EULA=Y` in the override file).

On the first start the worker may exit once because the emulator is still booting; it restarts by itself (up to 5 times). If it gave up, run the same command again.

### Variant B: real Azure Service Bus

Create a Service Bus namespace, a queue named like `SERVICE_BUS_QUEUE_NAME`, and copy a connection string with Send and Listen rights (Portal: namespace, Shared access policies) into both `bizstruct-be/.env` and `bizstruct-ml/.env`. Then:

```bash
docker compose --profile ml up -d --build
```

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
