# Gancio Deploy for NU31 Infra

This repository is used to deploy [`gancio`](https://framagit.org/les/gancio) onto the existing infrastructure in [`../infra`](../infra).

This is not a `gancio` fork and not a copy of its source code. The repository only contains deployment logic:

- `docker-stack.yml` for Docker Swarm
- a GitHub Actions workflow for build and deployment
- minimal documentation for integration with `infra`

This approach is useful because it allows you to:

- avoid pulling the full upstream `gancio` code into your own repository
- avoid maintaining a fork unless it is actually needed
- upgrade `gancio` using an upstream branch, tag, or commit
- reuse the existing networks, reverse proxy, and PostgreSQL from `infra`

## What Gancio is

`Gancio` is a self-hosted shared agenda for local communities. In practice, it is a web application for publishing events, pages, and announcements, with federation support.

In this setup, `gancio` runs as a separate Swarm stack, while your `infra` project provides:

- incoming HTTP/HTTPS through `caddy`
- access to shared `postgres`
- Docker Swarm overlay networks

## How deployment works

The flow is:

1. Only this deployment repository is stored in GitHub
2. GitHub Actions fetches the `gancio` sources from the upstream repository
3. The workflow builds a Docker image
4. The image is published to `GHCR`
5. The workflow creates an `envfile`
6. The workflow runs `docker stack deploy`
7. `caddy` from `infra` proxies the domain to the `gancio` service

At runtime this means:

- the `infra` stack is already deployed on the server
- the `gancio` stack is deployed separately
- both stacks are attached to shared external networks

## Files in this repository

- [`docker-stack.yml`](./docker-stack.yml) Swarm stack for the application
- [`.github/workflows/publish.yml`](./.github/workflows/publish.yml) build and deployment workflow
- [`infra-caddy-snippet.example`](./infra-caddy-snippet.example) example `caddy` route

## Infra dependencies

Before deploying, [`../infra`](../infra) must already be up and running.

This deployment expects the following external networks to already exist in `infra`:

- `infra_reverse-proxy`
- `infra_postgres-net`

It also expects the following services from `infra` to already be running:

- `caddy`
- `postgres-1`

## How to deploy Gancio on infra

### 1. Deploy infra

First, the infrastructure from [`../infra`](../infra) must be deployed.

If it is already running, you can skip this step.

### 2. Create a PostgreSQL database and user

For `gancio`, it is better to create a dedicated database user and a dedicated database.

Example:

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

This setup will use:

- `events.nu31.space`

Its DNS must point to the server running `infra`.

### 4. Add a Caddy route

Open [`../infra/caddy/CaddyFile`](../infra/caddy/CaddyFile) and add a block like this:

```caddyfile
events.nu31.space {
    reverse_proxy * gancio_app:13120
}
```

Where:

- `events.nu31.space` is your domain
- `gancio` is the Swarm stack name
- `app` is the service name from [`docker-stack.yml`](./docker-stack.yml)

Inside the Docker Swarm overlay network, the service will be reachable as `gancio_app`.

After editing the file, redeploy `infra` so `caddy` reloads the new configuration.

### 5. Create a GitHub repository for deployment

Create a separate GitHub repository, for example:

- `gancio-deploy`

Then push the contents of this directory into it.

### 6. Configure GitHub Secrets

The repository needs these secrets:

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

Example values:

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

You can optionally define:

- `GANCIO_REPOSITORY`
- `GANCIO_REF`

By default the workflow uses:

- repository: `https://framagit.org/les/gancio.git`
- ref: `main`

If you want to pin a specific version, you can set:

- `GANCIO_REF=v2.0.0-alpha.3`
- or a commit SHA

### 8. Run the deployment

There are two options:

- push to `main`
- manually run the `Build and deploy Gancio` workflow

The workflow will:

- check out the deployment repository
- clone upstream `gancio`
- build the image
- push the image to `ghcr.io/<github-owner>/gancio`
- deploy the `gancio` stack to the server

### 9. Verify the result

After a successful workflow run:

- the `gancio` service should be running in Swarm
- `caddy` should proxy the domain to `gancio_app:13120`
- the application should open at `BASEURL`

Typical checks:

- GitHub Actions status
- `docker service ls`
- `docker service ps gancio_app`
- `gancio` logs
- website availability on the domain

## How to update Gancio

There are two main approaches:

- just push changes to the deployment repository and keep tracking the latest upstream `main`
- pin `GANCIO_REF` to a tag or commit

If you want controlled upgrades, using a tag or commit SHA is better.

## Important notes

- `gancio` runs database migrations automatically on startup
- a separate migration job is not required for the basic setup
- user uploads are stored in the `gancio-uploads` Docker volume
- logs are stored in the `gancio-logs` Docker volume
- `user_locale` is stored in the `gancio-user-locale` Docker volume

## Current limitations

This setup assumes that:

- `infra` is already deployed
- `postgres` from `infra` is reachable through the external Swarm network
- `caddy` routes traffic by domain
- the separate deployment repository approach works for your case

If later you need:

- a custom Dockerfile
- patches on top of upstream
- a custom init or seed step
- dedicated backup jobs

those can be added to this deployment repository without moving to a fork.
