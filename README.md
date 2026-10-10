# litellm

Docker Compose stack for the [LiteLLM](https://docs.litellm.ai/) proxy gateway, backed by PostgreSQL (models, virtual keys, spend logs) and Valkey (Redis-compatible cache). The stack exposes one port:

- `4000` — LiteLLM proxy (OpenAI-compatible API and admin UI)

## Services

| Service | Container | Image | Purpose |
|---|---|---|---|
| `app` | `litellm-app` | `docker.litellm.ai/berriai/litellm:latest` | LiteLLM proxy, reads `config.yaml` |
| `postgres` | `litellm-postgres` | `postgres:18` | Models, virtual keys, spend logs |
| `valkey` | `litellm-valkey` | `valkey/valkey:9` | Cache, snapshots every 900 s if at least 1 key changed |

## Quick Start

### 1. Create the `.env` file

Create a `.env` file next to `docker-compose.yml` with the following variables:

```env
LITELLM_MASTER_KEY=<master_key>
LITELLM_SALT_KEY=<salt_key>
POSTGRES_PASSWORD=<postgres_password>
REDIS_PASSWORD=<redis_password>   # optional, see Notes
```

- `LITELLM_MASTER_KEY`: admin key for the proxy. Use a long random value, for example `sk-` followed by 32+ random characters.
- `LITELLM_SALT_KEY`: encrypts stored provider credentials. **Do not change it after the first start**, or stored credentials become unreadable.
- `POSTGRES_PASSWORD`: password for the `litellm` database user.

### 2. Configure AWS credentials

The proxy calls AWS Bedrock using the `bedrock` profile from the host's `~/.aws` directory, which is mounted read-only into the container. Make sure the profile exists:

```bash
aws configure --profile bedrock
```

The region is set to `eu-central-1` in `docker-compose.yml`.

### 3. Create the `service_main` Docker network

`docker-compose.yml` declares the `service_main` network as `external`, so it must exist before `up`:

```bash
docker network create service_main
```

> If the network already exists (created by another service), skip this step.

### 4. Start the stack

```bash
docker compose up -d
```

The `app` service waits for `postgres` and `valkey` to start. Then check the proxy:

```bash
curl http://127.0.0.1:4000/health/liveliness
```

## Commands

| Command | Description |
|---|---|
| `docker compose up -d` | Start all services in the background |
| `docker compose logs -f app` | Follow the proxy logs |
| `docker compose restart app` | Restart the proxy after changing `config.yaml` |
| `docker compose down` | Stop and remove containers (data in `./data-postgres` and `./data-valkey` is kept) |
| `docker compose pull && docker compose up -d` | Update images and recreate containers |

## Configuration

### `config.yaml`

Mounted into the proxy at `/app/config.yaml`. The current settings are:

- `general_settings.disable_env_credential_login: true`: disables login with credentials from environment variables.
- `litellm_settings.set_verbose: True`: verbose logging.
- `litellm_settings.cache: True`: response caching, backed by Valkey.
- `litellm_settings.drop_params: true`: drops request parameters that the target provider does not support.

Models are defined in this file or in the database. The compose file sets `STORE_MODEL_IN_DB: "True"`.

### Environment variables (`app` service)

| Variable | Value | Description |
|---|---|---|
| `AWS_PROFILE` | `bedrock` | AWS profile used for Bedrock |
| `AWS_REGION` | `eu-central-1` | AWS region |
| `LITELLM_MASTER_KEY` | from `.env` | Admin key |
| `LITELLM_SALT_KEY` | from `.env` | Encryption salt for stored credentials |
| `STORE_MODEL_IN_DB` | `True` | Store models in PostgreSQL |
| `DATABASE_URL` | built from `POSTGRES_PASSWORD` | Connection to `postgres` |
| `REDIS_HOST` / `REDIS_PORT` | `valkey` / `6379` | Connection to `valkey` |

## Data

- `./data-postgres` — PostgreSQL data (mounted at `/var/lib/postgresql`).
- `./data-valkey` — Valkey snapshot (`dump.rdb`).
- `./config.yaml` — proxy configuration (read-only mount).
- `~/.aws` — AWS credentials (read-only mount into `/root/.aws`).

Back up `./data-postgres` and `./data-valkey` together before upgrading.

## Notes

- `.env` contains secrets. Do not commit it. Keep it in `.gitignore`.
- The `docker-compose.yml` file is set up for local evaluation. Before real use, set strong values for `LITELLM_MASTER_KEY` and `LITELLM_SALT_KEY`, and pin the `app` image to a release tag instead of `latest`.
- PostgreSQL is published on host port `5432`. Valkey is not published to the host and is reachable only on the Docker network. Restrict access to port `5432` at the host firewall if the host is reachable from outside.
- `REDIS_PASSWORD` is set in `.env` but the `REDIS_PASSWORD` line in `docker-compose.yml` is commented out, and `valkey` starts without `requirepass`. Enable both together if you need Valkey authentication.
