# svc-hub

Central hub for the configuration and code of the Docker services deployed across a fleet of
Raspberry Pi boards. See `README.md` for a getting-started cheatsheet.

## Raspberry Pi fleet

| Name | Role |
|---|---|
| `lab53` | Komodo Core + Periphery + cloudflared + scanopy |
| `lab54` | Periphery — runs TREK |
| `lab55` | Periphery — idle, nothing scheduled |
| `lab56` | Periphery — runs n8n, cups |
| `lab57` | Periphery — runs Odoo |
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

Services don't set `mem_limit`/`cpus` — a hard `mem_limit` below what a service needs OOM-kills
it outright (happened with n8n at `1g`, below its documented 2GB floor); without a limit, the
Pi's own OOM killer handles it instead. If limits are ever needed, use Compose's
`mem_limit`/`cpus` shorthand, not `deploy.resources.limits` — Compose CLI ignores that field
without `--compatibility` (it's Swarm-only), so it'd look configured without applying.

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

**GitHub App hosted**, not self-hosted — `renovate.json` is all the config needed, no
container to maintain. Covers any `docker-compose.yml` in the repo by default (see the
coverage rule above for why tags must be pinned as literals). Automerge exceptions in
`renovate.json`: Komodo majors (breaking changes need review), scanopy's daemon/server minors
(pre-1.0, semver minors not guaranteed compatible), `bloodhound-ce`'s `neo4j` minors (pinned
to the 4.4 line for BloodHound's graph schema).

**Manual step not yet done:** `platformAutomerge: true` needs "Allow auto-merge" enabled in
GitHub repo settings (Settings > General > Pull Requests), or patch/minor PRs open but never
merge.

## Secrets via Komodo Variables

Default pattern for any secret a service needs: an `environment` block in that service's
`komodo/stacks/<name>.toml`, referencing secrets as `[[VARIABLE_NAME]]` instead of literal
values. Komodo resolves those against Settings > Variables (mark sensitive ones "secret") and
writes the result to `.env` on the host right before `docker compose up -d` — no hand-copied
`.env` needed. Reference examples: `komodo/stacks/n8n.toml`, `komodo/stacks/cups.toml`.

Gotchas:
- `env_file_path` (the written `.env`) is relative to `run_directory`, but a plain
  `env_file: ./.env` in the compose file resolves relative to the compose file's own
  directory. Set `run_directory = "<service>"` + `file_paths = ["docker-compose.yml"]` so
  they land in the same place (not the repo-root `["<service>/docker-compose.yml"]` other
  Stacks use).
- Changes to `run_directory`/`file_paths`/`environment` need the `ResourceSync` to run
  (Execute Sync, or its own `/sync` webhook) — `/deploy` alone reuses whatever Stack config
  Komodo already had saved and silently ignores newer `komodo/stacks/<name>.toml` content.
- A missing/unresolved `[[VAR]]` passes through as literal placeholder text; whether that's
  caught depends on the app (n8n crash-loops on an invalid `N8N_INSTANCE_OWNER_PASSWORD_HASH`,
  others may fail silently) — verify Variables exist in Komodo before the first deploy.
- A secret containing a literal `$` (bcrypt hashes: `$2b$12$...`) gets mangled: `env_file:`
  values go through the same `$`-interpolation as the rest of a compose file, and an
  unresolved `$VAR` reference gets dropped. Wrap the line in single quotes in the
  `environment` block (`SOME_HASH='[[SOME_HASH]]'`) — see `N8N_INSTANCE_OWNER_PASSWORD_HASH`
  in `komodo/stacks/n8n.toml`.
- This pattern only covers env vars. A secret that only has a config-file form (Odoo's
  `admin_passwd`, for example — the official image has no env-var for it) can't go through
  Komodo Variables. `config_files` (a separate Stack field) can track an existing file and
  make it editable in the Komodo UI, but — unlike `environment` — doesn't write/template it
  from a Variable, and registering one that doesn't already exist on the host **blocks the
  Stack's deploy entirely** (fails Validate Files). If nothing can populate the file
  automatically, prefer skipping it and living with the app's own defaults over adding a step
  that breaks automatic deploy — see `odoo/docker-compose.yml`.

## Service reference

Every service is deploy-only (see above) — the authoritative config, secrets handling, and
any hardware/version-pin quirks live as comments in that service's own
`docker-compose.yml`/`.env.example`/`komodo/stacks/<name>.toml`, not duplicated here.

| Service | Board | Deploy notes |
|---|---|---|
| Komodo Core | `lab53` | `komodo/core/`; see `komodo/CONNECT-SERVERS.md` to connect the rest of the fleet |
| n8n | `lab56` | `n8n/`; secrets via Komodo Variables (see above) |
| cups | `lab56` | `cups/`; secrets via Komodo Variables (see above); `network_mode: host` + `privileged: true` |
| Odoo | `lab57` | `odoo/`; own bridge network isolates Postgres from the LAN; no custom `odoo.conf` (see the note in `odoo/docker-compose.yml`) — runs on Odoo's own defaults, change the master password via `/web/database/manager` after first boot |
| TREK | `lab54` | `trek/` |
| scanopy | `lab53` | `scanopy/`; shares the board with Komodo Core/cloudflared; `network_mode: host` + `privileged: true` on `daemon`; LAN-only by decision (maps real network topology) |
| SysReptor | `lab58` | `sysreptor/`; shares the board with BloodHound CE |
| BloodHound CE | `lab58` | `bloodhound-ce/`; shares the board with SysReptor |
| cloudflared | `lab53` | `cloudflared/`; **not deployed yet on purpose** — see the note in `cloudflared/docker-compose.yml` |

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
- On `lab56`: `mkdir -p n8n/local-files`. Create the Komodo Variables that
  `komodo/stacks/n8n.toml` and `komodo/stacks/cups.toml` interpolate (see "Secrets via Komodo
  Variables" above) before the first deploy of either Stack.
- Configure the GitHub webhooks toward the `n8n` and `cups` Stacks' `/deploy` in Komodo.
- On `lab57`: `mkdir -p odoo/addons`. In Komodo, create `ODOO_POSTGRES_PASSWORD` ("secret") —
  interpolated via `komodo/stacks/odoo.toml`. After first boot, set a real master password at
  `http://lab57.local:8069/web/database/manager` (default login is `admin`).
- Configure the GitHub webhook toward the `odoo` Stack's `/deploy` in Komodo.
- On `lab54`: `mkdir -p trek/data trek/uploads`. In Komodo, create `TREK_ENCRYPTION_KEY`
  ("secret") — interpolated via `komodo/stacks/trek.toml`.
- Configure the GitHub webhook toward the `trek` Stack's `/deploy` in Komodo.
- On `lab53`: `mkdir -p scanopy/data`. In Komodo, create `SCANOPY_POSTGRES_PASSWORD`
  ("secret") — interpolated via `komodo/stacks/scanopy.toml`.
- Configure the GitHub webhook toward the `scanopy` Stack's `/deploy` in Komodo.
- In Komodo (Settings > Variables), create `SYSREPTOR_POSTGRES_PASSWORD`,
  `SYSREPTOR_REDIS_PASSWORD`, `SYSREPTOR_SECRET_KEY` (all "secret") — interpolated into the
  `sysreptor` Stack's `environment` via `komodo/stacks/sysreptor.toml`.
- Configure the GitHub webhook toward the `sysreptor` Stack's `/deploy` in Komodo.
- Import [HTB's report templates](https://github.com/Syslifters/HackTheBox-Reporting) — run
  from `sysreptor/` on lab58:
  `curl -s "https://docs.sysreptor.com/assets/htb-designs.tar.gz" | docker compose exec --no-TTY app python3 manage.py importdemodata --type=design`
- In Komodo (Settings > Variables), create `BLOODHOUND_POSTGRES_PASSWORD` and
  `BLOODHOUND_NEO4J_SECRET` (both "secret") — interpolated into the `bloodhound-ce` Stack's
  `environment` via `komodo/stacks/bloodhound-ce.toml`. Grab the randomized admin password
  from `docker compose logs bloodhound` after first deploy.
- Configure the GitHub webhook toward the `bloodhound-ce` Stack's `/deploy` in Komodo.
- `cloudflared` is intentionally on hold — see the note in `cloudflared/docker-compose.yml`.
  When ready to migrate: create `CLOUDFLARED_TUNNEL_TOKEN` in Komodo (the real token from the
  manual container), retire the manual container, then configure the GitHub webhook toward
  the `cloudflared` Stack's `/deploy` in Komodo.
