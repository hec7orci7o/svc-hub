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

## Secrets via Komodo Variables

Default pattern for any secret a service needs (not just `.env` filled by hand on the host):
declare an `environment` block in that service's `komodo/stacks/<name>.toml`, containing the
full `.env`-style content that service needs, with secrets referenced as `[[VARIABLE_NAME]]`
instead of literal values. Komodo resolves those against Settings > Variables (mark the
sensitive ones "secret" — masks them in logs/UI, doesn't block non-admin access on a
single-admin instance) and writes the result to a `.env` file on the host itself right before
`docker compose up -d`, which the service's own `env_file: ./.env` then reads. No hand-copied
`.env` needed on the host for anything covered this way. `komodo/stacks/n8n.toml` and
`komodo/stacks/cups.toml` are the reference examples.

Two things that must line up for this to work, both non-obvious from Komodo's docs:
- **`run_directory` + `file_paths` must put the run directory, the compose file, and the
  written `.env` all in the same folder.** `file_paths` are relative to `run_directory`
  (default: repo root); the written `.env`'s path (`env_file_path`, default `.env`) is
  *also* relative to `run_directory` — but a plain `env_file: ./.env` in the compose file
  resolves relative to the compose file's own directory, not `run_directory`. Set
  `run_directory = "<service>"` and `file_paths = ["docker-compose.yml"]` (not
  `["<service>/docker-compose.yml"]` from the repo root, the convention every other Stack
  without this pattern still uses) so all three coincide.
- **Config-driven changes to a Stack (`run_directory`, `file_paths`, `environment`) need the
  `ResourceSync` to run** (Execute Sync in the Komodo UI, or its own `/sync` webhook — see
  "Renovate → Komodo flow" above for why that's a different webhook than the Stack's own
  `/deploy`). Hitting `/deploy` alone redeploys with whatever Stack config Komodo already had
  saved, silently ignoring newer `komodo/stacks/<name>.toml` content until a sync catches up.

A missing or unresolved `[[VARIABLE_NAME]]` (Variable never created in Komodo) doesn't fail
loudly at the Komodo level — it gets passed through as the literal placeholder text, and
whether that's caught depends entirely on the app. n8n's `N8N_INSTANCE_OWNER_PASSWORD_HASH`
is a case that crash-loops the container on every boot with a clear error
(`... is not a valid bcrypt hash`) — confirmed in production on this exact deployment. Others
may fail more silently (e.g. an app just treating the placeholder as a literal password).
Always verify Variables actually exist in Komodo before the first deploy of a Stack using
this pattern, don't assume the sync/deploy will surface a missing one for you.

## Service reference

Every service is deploy-only (see above) — the authoritative config, secrets handling, and
any hardware/version-pin quirks live as comments in that service's own
`docker-compose.yml`/`.env.example`/`komodo/stacks/<name>.toml`, not duplicated here.

| Service | Board | Deploy notes |
|---|---|---|
| Komodo Core | `lab53` | `komodo/core/`; see `komodo/CONNECT-SERVERS.md` to connect the rest of the fleet |
| n8n | `lab56` | `n8n/`; secrets via Komodo Variables (see above) |
| cups | `lab56` | `cups/`; secrets via Komodo Variables (see above); `network_mode: host` + `privileged: true` |
| Odoo | `lab55` | `odoo/`; own bridge network isolates Postgres from the LAN |
| TREK | `lab54` | `trek/` |
| scanopy | `lab57` | `scanopy/`; `network_mode: host` + `privileged: true` on `daemon` |
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
- `cloudflared` is intentionally on hold — see the note in `cloudflared/docker-compose.yml`.
  When ready to migrate: copy the real `TUNNEL_TOKEN` from the manual container into
  `cloudflared/.env`, `docker compose up -d` on `lab53`, retire the manual container, then
  configure the GitHub webhook toward the `cloudflared` Stack's `/deploy` in Komodo.
