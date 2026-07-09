# svc-hub

Central hub for configuring and deploying Docker services across a fleet of Raspberry Pi
boards, managed with [Komodo](https://komo.do) and kept up to date by Renovate. Every service
is deploy-only — no Dockerfile, no build, no registry this repo owns; each one just pins an
already-published image and Komodo deploys it straight from `docker-compose.yml`.

## What's here

| Folder | Deploys | Runs on |
|---|---|---|
| `komodo/core/` | Komodo Core + Mongo + local Periphery | `lab53` |
| `cloudflared/` | Cloudflare Tunnel — the hub's only public ingress | `lab53` |
| `n8n/` | n8n | `lab56` |
| `cups/` | CUPS/AirPrint bridge | `lab56` |
| `odoo/` | Odoo ERP + Postgres | `lab55` |
| `trek/` | TREK — collaborative travel planner | `lab54` |
| `scanopy/` | scanopy — network topology documentation | `lab57` |
| `sysreptor/` | SysReptor — pentest reporting platform | `lab58` |
| `bloodhound-ce/` | BloodHound CE — AD attack path analysis | `lab58` |
| `komodo/servers.toml`, `komodo/stacks/*.toml` | Fleet + Stack definitions, synced via Komodo's ResourceSync | — |

## Getting started

1. **Deploy Komodo Core** on `lab53`: `git clone` this repo there, `cd komodo/core`,
   `cp .env.example .env` and fill in real secrets, then `docker compose up -d`.
2. **Create your account** at `http://lab53.local:9120` via the signup button (no admin is
   auto-created).
3. **Bootstrap the first ResourceSync** in the Komodo UI, pointed at this repo — from then on,
   `komodo/servers.toml` and `komodo/stacks/*.toml` are managed entirely through git.
4. **Connect the rest of the fleet** (`lab54`–`lab58`) following `komodo/CONNECT-SERVERS.md`.
5. **Deploy `cloudflared`**: create the tunnel in the Cloudflare Zero Trust dashboard, fill in
   `cloudflared/.env`, `docker compose up -d` on `lab53`, then register its Stack's `/deploy`
   webhook in GitHub.
6. **Deploy `n8n`** the same way on `lab56` (see `n8n/.env.example`), and register its webhook.
7. **Deploy `cups`** the same way, also on `lab56` (see `cups/.env.example`), and register its
   webhook.
8. **Deploy `odoo`** the same way on `lab55` (see `odoo/.env.example`), and register its
   webhook.
9. **Deploy `trek`** the same way on `lab54` (see `trek/.env.example`), and register its
   webhook.
10. **Deploy `scanopy`** the same way on `lab57` (see `scanopy/.env.example`), and register
    its webhook.
11. **Deploy `sysreptor`** and **`bloodhound-ce`** the same way, both on `lab58` (see their
    `.env.example` files), and register their webhooks.

Full architecture, conventions, and the Renovate/webhook mechanics are in `CLAUDE.md`.

## Add a new service

Use the `service-scaffolder` subagent, or see "Adding a new service" in `CLAUDE.md`.

## Everyday commands

```sh
docker compose -f <folder>/docker-compose.yml config   # validate syntax before committing
```
