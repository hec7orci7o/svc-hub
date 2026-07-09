# svc-hub

Central hub for the configuration and code of the Docker services deployed across a fleet of
Raspberry Pi boards. See `README.md` for a getting-started cheatsheet.

## Raspberry Pi fleet

| Name | Role |
|---|---|
| `lab53` | Komodo Core + Periphery + cloudflared |
| `lab54` | Periphery — runs TREK |
| `lab55` | Periphery — runs Odoo |
| `lab56` | Periphery — runs n8n, cups |
| `lab57` | Periphery — runs scanopy |
| `lab58` | Periphery — runs sysreptor, bloodhound-ce |

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

## No resource limits, on purpose

Services don't set `mem_limit`/`cpus` — a hard `mem_limit` below what a service actually needs
OOM-kills it outright (that's exactly what happened with n8n at `1g`, below its own documented
2GB floor); without a limit, the Pi's own memory pressure/OOM killer handles it instead. If a
future need for real limits comes up, use Compose's `mem_limit`/`cpus` shorthand, not
`deploy.resources.limits` — the Compose CLI ignores that field unless `--compatibility` is
passed, since it's only native in Swarm mode, so it would look configured without applying.

## Compose security baseline

Every service gets `security_opt: [no-new-privileges:true]` — it only blocks privilege
escalation via setuid binaries, never breaks anything legitimate. Add `cap_drop: [ALL]` too,
unless the service's own entrypoint needs root-level capabilities at startup (common in
official DB images that `chown`/`setuid` down to an unprivileged user) or it genuinely needs
host-level access by design (Periphery's `docker.sock`/`/proc` mounts). Skipping `cap_drop` for
either reason must come with a one-line comment saying why — a silent skip reads as an
oversight, not a decision, to the next reader.

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
manual review. scanopy's daemon/server images get their own extra `packageRule` that also
excludes minors from automerge — they're pre-1.0, where a semver minor isn't guaranteed
backwards-compatible. `bloodhound-ce/`'s `neo4j` image gets the same minor-automerge exclusion,
scoped to that file only, since it's pinned to the 4.4 line for BloodHound CE's graph schema
rather than for architecture/stability reasons.

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

## Deploying CUPS/AirPrint bridge (lab56)

See `cups/docker-compose.yml` + `cups/.env.example`. Runs alongside n8n on the same board.
Uses `network_mode: host` (AirPrint/Bonjour discovery needs real mDNS broadcasts, which don't
cross a bridge network) and `privileged: true` (USB printer passthrough + Avahi/dbus host
integration) — the hub's usual `no-new-privileges`/`cap_drop: ALL` baseline is skipped here on
purpose, not by oversight (see the comment on the `cups` service). Its web admin panel must
stay LAN-only: never give it a `cloudflared` Public Hostname rule, since `CUPS_ADMIN_PASSWORD`
sits in plaintext in `.env`. This image tags by build date (`focal-YYYYMMDD`), not semver, so
Renovate PRs for it need manual review rather than trusting patch/minor automerge.

## Deploying Odoo (lab55)

See `odoo/docker-compose.yml` + `odoo/.env.example`. Official `odoo` + `postgres` images, own
bridge network (`odoo_network`) so Postgres isn't reachable from the LAN, only Odoo's HTTP
(`8069`) and websocket (`8072`) ports are published. Postgres's memory tuning
(`shared_buffers=512MB`, `effective_cache_size=2GB`) is sized for lab55's 8GB — see the comment
on that `command:` line before moving this to a smaller board. `./addons` and `./config` are
bind mounts for custom modules/config, created by hand on the host (same pattern as n8n's
`local-files`), not committed.

## Deploying TREK (lab54)

See `trek/docker-compose.yml` + `trek/.env.example`. Official image, SQLite (no external DB),
reachable only over the LAN by mDNS hostname (`http://lab54.local:3000`) — same public-exposure
model as n8n, via the `cloudflared/` tunnel on lab53 if ever enabled. `ENCRYPTION_KEY` is
critical (encrypts stored secrets: API keys, MFA, SMTP, OIDC) — generate once with
`openssl rand -hex 32`; unlike n8n, upstream documents a rotation script if you ever need to
change it, but treat it as permanent by default. `ADMIN_EMAIL`/`ADMIN_PASSWORD` only apply on
first boot.

## Deploying scanopy (lab57)

See `scanopy/docker-compose.yml` + `scanopy/.env.example`. Three containers: `daemon` (host
networking + `privileged: true`, needed for real LAN topology discovery — see the comment on
that service), `postgres`, and `server` (published on `:60072`). AGPL-3.0 for self-hosted use —
if this ever needs to run as a commercial/closed offering instead, see the licensing note in
the compose file. Images are pinned to `v0.17.4`; the project is pre-1.0, so
`renovate.json` carves out an explicit exception keeping its minor bumps off automerge
(0.x minor bumps aren't guaranteed non-breaking under semver) — patch bumps still automerge
like everything else.

## Deploying SysReptor (lab58)

See `sysreptor/docker-compose.yml` + `sysreptor/.env.example`. Pentest reporting platform:
`postgres:14` + `redis:8.0-alpine` + the official `syslifters/sysreptor` app image. Upstream's
own `install.sh` generates a compose with `build: ../../` alongside `image:`, which would
build the image locally — deliberately omitted here so this stays deploy-only, since
`image:` alone already pulls the exact tag they publish. Two secrets are critical:
`SECRET_KEY` (session signing, safe to rotate — just logs everyone out) and
`ENCRYPTION_KEYS` (encrypts findings/uploads at rest — losing every listed key makes
existing data unrecoverable, rotate by adding a new key rather than replacing the only one).
No license key needed for community use — `LICENSE` is Professional-tier only. Code license
is "SysReptor Community License 1.1" (source-available, not an OSI-approved license).

## Deploying BloodHound CE (lab58)

See `bloodhound-ce/docker-compose.yml` + `bloodhound-ce/.env.example`. Shares lab58 with
`sysreptor`. No `mem_limit`/JVM heap cap set on `graph-db` (Neo4j), same "no resource limits"
reasoning as the rest of the hub — see that section above. Three containers:
`app-db` (`postgres:18`), `graph-db` (`neo4j:4.4.42`, pinned to the 4.4 line on purpose —
BloodHound CE's graph schema targets it, don't bump ahead of upstream; `renovate.json` blocks
automerge on its minors accordingly), and `bloodhound` (the only one published to the LAN, on
`:8080`). Postgres and Neo4j are intentionally **not** published to the host, unlike upstream's
example compose — Neo4j's browser is a second attack surface with its own credentials on top
of the BloodHound UI, and this is a tool whose whole purpose is mapping AD attack paths, so
treat its own exposure with the same care. The initial admin password is randomized on first
boot and printed to `docker compose logs bloodhound`, not stored in `.env`.

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
- On `lab56`: copy `cups/.env.example` to `.env`, set a real `CUPS_ADMIN_PASSWORD`, and
  `docker compose up -d`.
- Configure the GitHub webhook toward the `cups` Stack's `/deploy` in Komodo.
- On `lab55`: copy `odoo/.env.example` to `.env`, set a real `POSTGRES_PASSWORD`,
  `mkdir -p addons config`, and `docker compose up -d`.
- Configure the GitHub webhook toward the `odoo` Stack's `/deploy` in Komodo.
- On `lab54`: copy `trek/.env.example` to `.env`, generate `ENCRYPTION_KEY`,
  `mkdir -p data uploads`, and `docker compose up -d`.
- Configure the GitHub webhook toward the `trek` Stack's `/deploy` in Komodo.
- On `lab57`: copy `scanopy/.env.example` to `.env`, set a real `POSTGRES_PASSWORD`,
  `mkdir -p data`, and `docker compose up -d`.
- Configure the GitHub webhook toward the `scanopy` Stack's `/deploy` in Komodo.
- On `lab58`: copy `sysreptor/.env.example` to `.env`, set real `POSTGRES_PASSWORD`,
  `REDIS_PASSWORD`, `SECRET_KEY`, and `ENCRYPTION_KEYS`/`DEFAULT_ENCRYPTION_KEY_ID`, then
  `docker compose up -d`.
- Configure the GitHub webhook toward the `sysreptor` Stack's `/deploy` in Komodo.
- Also on `lab58`: copy `bloodhound-ce/.env.example` to `.env`, set real `POSTGRES_PASSWORD`
  and `NEO4J_SECRET`, then `docker compose up -d`. Grab the randomized admin password from
  `docker compose logs bloodhound`.
- Configure the GitHub webhook toward the `bloodhound-ce` Stack's `/deploy` in Komodo.
- `cloudflared` is intentionally on hold — see "Not deployed yet, on purpose" above. When
  ready to migrate: copy the real `TUNNEL_TOKEN` from the manual container into
  `cloudflared/.env`, `docker compose up -d` on `lab53`, retire the manual container, then
  configure the GitHub webhook toward the `cloudflared` Stack's `/deploy` in Komodo.
