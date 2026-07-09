# svc-hub

Central hub for the configuration and code of the Docker services deployed across a fleet of
Raspberry Pi boards. See `README.md` for a getting-started cheatsheet.

## Raspberry Pi fleet

| Name | Role |
|---|---|
| `lab53` | Komodo Core + Periphery + cloudflared |
| `lab54` | Periphery (fleet) |
| `lab55` | Periphery (fleet) |
| `lab56` | Periphery — runs n8n |
| `lab57` | Periphery (fleet) |
| `lab58` | Periphery (fleet) |

Declared as code in `komodo/servers.toml`, addressed by `<name>.local` (mDNS/Avahi, on by
default on Raspberry Pi OS) rather than raw LAN IPs — this repo is public, so no real network
details are committed. Real IPs live only on your own LAN/router config, never in git.

## Structure

```
n8n/                       # n8n: official image, runs on lab56
cloudflared/               # Cloudflare Tunnel: the hub's only public ingress, runs on lab53
komodo/                    # Resource Sync (servers.toml, stacks/<name>.toml)
komodo/core/               # How to deploy Komodo Core+Mongo+Periphery itself on lab53
komodo/CONNECT-SERVERS.md  # How to connect the rest of the Raspberry Pi fleet via Periphery
renovate.json               # Automatic image version updates
.claude/agents/             # Review/scaffolding subagents
```

**Every service in this hub is deploy-only.** There's no build step, no Dockerfile, no
registry we publish to: each service is just a folder at the repo root (`n8n/`, `cloudflared/`,
`komodo/core/`) with a `docker-compose.yml` + `.env.example` referencing an image already
published upstream (n8n's, Cloudflare's, Komodo's own). Redeploy is triggered by a GitHub
webhook pointed directly at that Stack's `/deploy` endpoint in Komodo — no GitHub Actions
workflow involved anywhere in this repo.

## Adding a new service

Use the `service-scaffolder` subagent, or by hand:
1. Create `<name>/{docker-compose.yml,.env.example}` at the repo root, referencing the
   official upstream image directly, with the version **pinned as a literal tag in
   `docker-compose.yml`** (not behind an env var — see the Renovate section below for why).
   Use `n8n/docker-compose.yml` and `n8n/.env.example` as the reference template.
2. Create `komodo/stacks/<name>.toml` following `komodo/stacks/n8n.toml` as the schema
   reference (same `git_provider`/`repo`/`branch`; change `file_paths` and `server`).
3. Configure a GitHub webhook pointing directly at that Stack's `/deploy` (copied from the
   Komodo UI, Stack Config > Webhooks) — there's no CI to trigger it from.

## Compose resource-limit trap

Always use `mem_limit`/`cpus` (Compose shorthand, always applied with `docker compose up`).
**Don't** use `deploy.resources.limits`: the Compose CLI ignores that field unless `--compatibility`
is passed — it's only native in Swarm mode. On a resource-constrained Raspberry Pi, a limit that
"looks" configured but doesn't apply is worse than no limit at all.

## Renovate → Komodo flow

1. Renovate detects an outdated image (native `docker-compose` manager, no hand-written regex)
   and opens a PR that bumps the pinned tag directly in the relevant `docker-compose.yml`.
   Patch/minor auto-merge; major requires manual review.
2. On merge to `main`, the GitHub webhook toward that Stack's `/deploy` fires, and Komodo
   redeploys with the already-updated tag. No build, no registry push — Komodo pulls the
   image straight from wherever it's published upstream.

**Renovate coverage rule:** Renovate's docker-compose manager only reads image references from
`docker-compose.yml` itself — never `.env.example`. So every service's image tag is pinned as a
**literal tag directly in `docker-compose.yml`**, not behind an env var; `.env`/`.env.example`
is reserved for secrets and per-deploy config only. This is what makes Renovate's patch/minor
automerge actually fire. `komodo/core/docker-compose.yml` pins both the `core` and `periphery`
images to the same `major.minor.patch` tag — they must move together, since Komodo releases
Core and Periphery as a matched pair; the shared `groupName` in `renovate.json`'s patch/minor
`packageRule` bundles both into the same PR so they bump in lockstep.

Distinction that matters in Komodo: the GitHub webhook toward the `ResourceSync` triggers
`/sync` (re-reads `servers.toml`/`stacks/*.toml`, topology changes); the webhook toward a
specific Stack is `/deploy` (new image or updated compose for one service). They are not
the same thing.

## Useful commands

```sh
docker compose -f <folder>/docker-compose.yml config   # validate syntax
```

## Available subagents

- `docker-compose-reviewer`: reviews compose files against best practices for Raspberry Pi.
- `deployment-security-auditor`: audits security exposure before deploying (services sit
  behind the `cloudflared/` tunnel but are still reachable from the wider internet through it).
- `service-scaffolder`: generates the structure for a new deploy-only service following the
  hub's pattern.

## Renovate

Decision: **GitHub App hosted**, not self-hosted. `renovate.json` at the root is all the config
needed; GitHub runs the scans, no container to maintain. It covers by default any
`docker-compose.yml` in the repo (`komodo/core/`, `n8n/`, `cloudflared/`, and any future
service folder) — see the coverage rule above for why versions must be pinned as literals in
compose. For Komodo, majors (e.g. `:2` → `:3`) still skip automerge via the existing
`packageRule` — intentional, since a Komodo major can bring incompatible changes that need
manual review.

**Manual step not yet done:** `platformAutomerge: true` in `renovate.json` requires "Allow
auto-merge" to be enabled in the GitHub repo settings (Settings > General > Pull Requests).
Without it, Renovate's patch/minor PRs will open but never actually merge themselves.

## Deploying Komodo Core (lab53)

See `komodo/core/docker-compose.yml` + `komodo/core/.env.example` (copy to
`.env`, never committed — see `.gitignore`). To connect the rest of the fleet,
see `komodo/CONNECT-SERVERS.md` (Periphery via systemd, Komodo's recommended method).

Mongo is pinned to `4.4.18`, not a newer/unpinned tag: MongoDB 5.0+ requires ARMv8.2-A, which
Raspberry Pi 4's Cortex-A72 doesn't have — `:latest` crashes on boot with `SIGILL`. See the
`ponytail:` comment on that image line for the upgrade path if this ever needs to change.

## Deploying n8n (lab56)

See `n8n/docker-compose.yml` + `n8n/.env.example`. Official image, SQLite (enough for a
single instance), reachable only over the LAN by mDNS hostname (`http://lab56.local:5678`) —
the `cloudflared/` tunnel on lab53 is what exposes it publicly, not this compose file. The first
user is created via n8n's own signup UI (since v1.0 there's no basic auth or way to disable
login). `N8N_ENCRYPTION_KEY` is critical: generate it once and never touch it again — changing
it breaks decryption of already-saved credentials.

## Deploying cloudflared (lab53)

See `cloudflared/docker-compose.yml` + `cloudflared/.env.example`. This is meant to be the
hub's only public ingress: it opens an outbound-only connection to Cloudflare, so no router
port forwarding is needed for any service. Hostname-to-internal-service routing (which public
hostname maps to which Pi's `<name>.local:port`) is configured in the Cloudflare Zero Trust
dashboard, not in this repo. Individual services still bind their port to the host (`ports:` in
their own compose file) so cloudflared can reach them over the LAN — that binding is LAN-only
reachability, not public exposure, since nothing forwards it at the router.

**Not deployed yet, on purpose:** `hermes`/`lab53` already runs a cloudflared tunnel started
manually outside this repo (unpinned `:latest`, predates this hub). `komodo/stacks/cloudflared.toml`
is declared so it shows up in Komodo, but its `/deploy` webhook is deliberately not registered
in GitHub — don't deploy it without migrating the real `TUNNEL_TOKEN` from the manual container
into `cloudflared/.env` first and retiring the manual one, or you'll end up running two tunnels.

## Manual steps outside git (not automatable from this repo)

- `KOMODO_WEBHOOK_SECRET` is not a GitHub Actions secret (there's no workflow to consume it):
  it's the value Komodo Core validates incoming webhooks against (set in
  `komodo/core/.env.example`), and must match the "Secret" field entered when creating each
  Stack's native GitHub webhook below.
- Enable "Allow auto-merge" in GitHub repo settings, required for Renovate's
  `platformAutomerge: true` to actually merge patch/minor PRs.
- Install the Renovate GitHub App on the repo.
- On `lab53`: copy `komodo/core/.env.example` to `.env`, fill in real
  secrets, and `docker compose up -d`.
- Connect the rest of the fleet following `komodo/CONNECT-SERVERS.md`.
- Manual bootstrap of the first `ResourceSync` in the Komodo UI (it can't self-create from a
  file it isn't reading yet) + register the GitHub webhook toward its listener.
- On `lab56`: copy `n8n/.env.example` to `.env`, generate `N8N_ENCRYPTION_KEY`,
  `mkdir -p local-files`, and `docker compose up -d`.
- Configure the GitHub webhook toward the `n8n` Stack's `/deploy` in Komodo.
- `cloudflared` is intentionally on hold — see "Not deployed yet, on purpose" above. When
  ready to migrate: copy the real `TUNNEL_TOKEN` from the manual container into
  `cloudflared/.env`, `docker compose up -d` on `lab53`, retire the manual container, then
  configure the GitHub webhook toward the `cloudflared` Stack's `/deploy` in Komodo.
