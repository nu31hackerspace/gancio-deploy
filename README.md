# Gancio Deploy for NU31 Infra

This repository deploys [`gancio`](https://framagit.org/les/gancio) to the existing [`infra`](https://github.com/nu31hackerspace/infra).

It contains:

- `docker-stack.yml`
- GitHub Actions deployment workflow
- minimal infra integration notes

## Files in this repository

- [`docker-stack.yml`](./docker-stack.yml) Swarm stack for the application
- [`.github/workflows/publish.yml`](./.github/workflows/publish.yml) build and deployment workflow

## Infra dependencies

Before deploying, [`infra`](https://github.com/nu31hackerspace/infra) must already be running.

Required external networks:

- `infra_reverse-proxy`
- `infra_postgres-net`

Required services:

- `caddy`
- `postgres-1`

## How to deploy Gancio on infra

### 1. Deploy infra

`infra` must already be deployed.

### 2. Create a PostgreSQL database and user

Create a dedicated PostgreSQL user and database:

```sql
CREATE USER gancio WITH PASSWORD 'change-me';
CREATE DATABASE gancio OWNER gancio;
GRANT ALL PRIVILEGES ON DATABASE gancio TO gancio;
```

Expected deployment parameters:

- `DB_HOST=infra-postgres-1`
- `DB_PORT=5432`
- `DB_DATABASE=gancio`
- `DB_USERNAME=gancio`
- `DB_PASSWORD=<your-password>`

### 3. Choose a domain

This setup uses `events.nu31.space`.

DNS must point to the server running `infra`.

### 4. Add a Caddy route

Add this block to `infra/caddy/CaddyFile`:

```caddyfile
events.nu31.space {
    reverse_proxy * gancio_app:13120
}
```

Then redeploy `infra`.

### 5. Create a GitHub repository for deployment

Push this directory into a separate GitHub repository, for example `nu31hackerspace/gancio-deploy`.

### 6. Configure GitHub Secrets

Required secrets:

- `DEPLOY_HOST`
- `DEPLOY_USER`
- `DEPLOY_SSH_PRIVATE_KEY`
- `BASEURL`
- `DB_HOST`
- `DB_PORT`
- `DB_DATABASE`
- `DB_USERNAME`
- `DB_PASSWORD`
- `NUXT_SESSION_PASSWORD`

Optional:

- `LOG_LEVEL`

Example:

- `BASEURL=https://events.nu31.space`
- `DB_HOST=infra-postgres-1`
- `DB_PORT=5432`
- `LOG_LEVEL=info`

`NUXT_SESSION_PASSWORD` must be at least 32 characters long.

Example generation:

```bash
openssl rand -base64 32
```

### 7. Configure GitHub Variables

Optional:

- `GANCIO_REPOSITORY`
- `GANCIO_REF`

By default the workflow uses:

- repository: `https://framagit.org/les/gancio.git`
- ref: `v1.28.2`

If you want to pin a specific version, you can set:

- `GANCIO_REF=v1.28.2`
- or a commit SHA

### 8. Run the deployment

Push to `main` or run the `Build and deploy Gancio` workflow manually.

### 9. Verify the result

Checks:

- GitHub Actions status
- `docker service ls`
- `docker service ps gancio_app`
- `gancio` logs
- `https://events.nu31.space`

## Important notes

- `gancio` runs database migrations automatically on startup
- a separate migration job is not required for the basic setup
- user uploads are stored in the `gancio-uploads` Docker volume
- logs are stored in the `gancio-logs` Docker volume
- `user_locale` is stored in the `gancio-user-locale` Docker volume
