# svc-hub

Central hub for configuring and deploying Docker services across a fleet of Raspberry Pi
boards, managed with [Komodo](https://komo.do) and kept up to date by Renovate. Every service
is deploy-only — no Dockerfile, no build, no registry this repo owns; each one just pins an
already-published image and Komodo deploys it straight from `docker-compose.yml`.

## What's here

One folder per service at the repo root, each with a `docker-compose.yml` + `.env.example` +
`<name>.toml` (its Komodo Stack definition). `komodo/resources.toml` declares the fleet, synced
via Komodo's ResourceSync. `komodo/` is also where Komodo itself lives (Core + Mongo + local
Periphery).

## Getting started

1. **Deploy Komodo Core**: `git clone` this repo onto that board, `cd komodo`,
   `cp .env.example .env` and fill in real secrets, then `docker compose up -d`.
2. **Create your account** via the signup button on Core's UI (no admin is auto-created).
3. **Bootstrap the first ResourceSync** in the Komodo UI, pointed at this repo — from then on,
   `komodo/resources.toml` and every service's `<name>.toml` are managed entirely through git.
4. **Connect the rest of the fleet** following `komodo/CONNECT-SERVERS.md`.
5. **Deploy each service**: `.env.example` → `.env` → fill in → `docker compose up -d` →
   register that Stack's `/deploy` webhook in GitHub. Any secrets a service needs go through
   Komodo Variables instead (see `CLAUDE.md`'s "Secrets via Komodo Variables"); any first-boot
   step specific to that service is documented in its own `docker-compose.yml`/`.env.example`.

Full architecture, conventions, and the Renovate/webhook mechanics are in `CLAUDE.md`.

## Add a new service

Use the `service-scaffolder` subagent, or see "Adding a new service" in `CLAUDE.md`.

## Everyday commands

```sh
docker compose -f <folder>/docker-compose.yml config   # validate syntax before committing
```
