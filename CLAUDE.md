# svc-hub

Central hub for the configuration and code of the Docker services deployed across a fleet of
Raspberry Pi boards, managed with [Komodo](https://komo.do) and kept current by Renovate.
See `README.md` for a getting-started cheatsheet.

## Fleet

Declared as code in `komodo/servers.toml`, addressed by `<name>.local` (mDNS/Avahi, on by
default on Raspberry Pi OS) rather than raw LAN IPs — this repo is public, so no real network
details are committed. What runs where: `grep "server = " komodo/stacks/*.toml` — don't
duplicate that mapping here, it changes with every new/moved service.

## Structure

```
komodo/                    # Resource Sync (servers.toml, stacks/<name>.toml)
komodo/core/               # How to deploy Komodo Core+Mongo+Periphery itself
komodo/CONNECT-SERVERS.md  # How to connect the rest of the fleet via Periphery
renovate.json               # Automatic image version updates
.claude/agents/             # Review/scaffolding subagents
<service>/                  # One folder per deploy-only service (docker-compose.yml + .env.example)
```

**Every service in this hub is deploy-only.** No build step, no Dockerfile, no registry we
publish to: each service is a folder at the repo root with a `docker-compose.yml` +
`.env.example` referencing an image already published upstream. Redeploy is triggered by a
GitHub webhook pointed directly at that Stack's `/deploy` endpoint in Komodo — no GitHub
Actions workflow involved anywhere in this repo.

## Adding a new service

Use the `service-scaffolder` subagent, or by hand:
1. Create `<name>/{docker-compose.yml,.env.example}`, referencing the official upstream image
   directly, version **pinned as a literal tag in `docker-compose.yml`** (not behind an env
   var — see the Renovate coverage rule below). Use any existing service folder as a template.
2. Create `komodo/stacks/<name>.toml` following an existing one as the schema reference (same
   `git_provider`/`repo`/`branch`; change `file_paths`/`server`).
3. Configure a GitHub webhook pointing at that Stack's `/deploy` (Komodo UI, Stack Config >
   Webhooks) — there's no CI to trigger it from.

## No resource limits, on purpose

Services don't set `mem_limit`/`cpus` — a hard `mem_limit` below what a service needs OOM-kills
it outright (happened in production, below the image's own documented memory floor); without a
limit, the Pi's own OOM killer handles it instead. If limits are ever needed, use Compose's
`mem_limit`/`cpus` shorthand, not `deploy.resources.limits` — Compose CLI ignores that field
without `--compatibility` (it's Swarm-only), so it'd look configured without applying.

## Compose security baseline

Every service gets `security_opt: [no-new-privileges:true]`. Add `cap_drop: [ALL]` too, unless
the entrypoint needs root-level capabilities at startup (common in DB images that
`chown`/`setuid` down to an unprivileged user) or it genuinely needs host-level access by
design. Skipping `cap_drop` needs a one-line comment saying why — a silent skip reads as an
oversight, not a decision.

## Secrets via Komodo Variables

Default pattern for any secret a service needs: an `environment` block in that service's
`komodo/stacks/<name>.toml`, referencing secrets as `[[VARIABLE_NAME]]` instead of literal
values. Komodo resolves those against Settings > Variables (mark sensitive ones "secret") and
writes the result to `.env` on the host right before `docker compose up -d` — no hand-copied
`.env` needed. Variable names are prefixed per-service (they're global in Komodo, not scoped
per-stack) to avoid collisions.

Gotchas:
- `env_file_path` (the written `.env`) is relative to `run_directory`, but a plain
  `env_file: ./.env` in the compose file resolves relative to the compose file's own
  directory. Set `run_directory = "<service>"` + `file_paths = ["docker-compose.yml"]` so
  they land in the same place.
- Config-driven Stack changes (`run_directory`/`file_paths`/`environment`) need the
  `ResourceSync` to run (Execute Sync, or its own `/sync` webhook) — `/deploy` alone reuses
  whatever Stack config Komodo already had saved and silently ignores newer toml content.
- A missing/unresolved `[[VAR]]` passes through as literal placeholder text; whether that's
  caught depends on the app (some crash-loop with a clear error, others fail silently) —
  verify Variables exist in Komodo before the first deploy of a new Stack.
- A secret containing a literal `$` (bcrypt hashes: `$2b$12$...`) gets mangled: `env_file:`
  values go through the same `$`-interpolation as the rest of a compose file, and an
  unresolved `$VAR` reference gets dropped. Wrap the line in single quotes in the
  `environment` block (`SOME_HASH='[[SOME_HASH]]'`) — the Komodo Variable itself stays raw.
- This pattern only covers env vars. A secret that only has a config-file form can't go
  through Komodo Variables. `config_files` (a separate Stack field) can track an existing
  file and make it editable in the Komodo UI, but — unlike `environment` — doesn't
  write/template it from a Variable, and registering one that doesn't already exist on the
  host **blocks the deploy entirely** (fails Validate Files). If nothing can populate the
  file automatically, prefer skipping it and living with the app's own defaults over adding
  a step that breaks automatic deploy.

## Renovate → Komodo flow

1. Renovate detects an outdated image (native `docker-compose` manager, no hand-written regex)
   and opens a PR that bumps the pinned tag directly in the relevant `docker-compose.yml`.
   Patch/minor auto-merge; major requires manual review.
2. On merge to `main`, the GitHub webhook toward that Stack's `/deploy` fires, and Komodo
   redeploys with the already-updated tag. No build, no registry push.

**Renovate coverage rule:** Renovate's docker-compose manager only reads image references from
`docker-compose.yml` itself — never `.env.example`. So every service's image tag is pinned as a
**literal tag directly in `docker-compose.yml`**; `.env`/`.env.example` is reserved for secrets
and per-deploy config only. This is what makes patch/minor automerge actually fire.

Distinction that matters in Komodo: the GitHub webhook toward the `ResourceSync` triggers
`/sync` (re-reads `servers.toml`/`stacks/*.toml`, topology changes); the webhook toward a
specific Stack is `/deploy` (new image or updated compose for one service). Not the same thing.

**GitHub App hosted**, not self-hosted — `renovate.json` is all the config needed, no
container to maintain. Automerge exceptions (pre-1.0 packages, pinned majors, etc.) live in
`renovate.json` itself with inline reasoning in the relevant `docker-compose.yml`; don't
duplicate the list here.

**Manual step not yet done:** `platformAutomerge: true` needs "Allow auto-merge" enabled in
GitHub repo settings (Settings > General > Pull Requests), or patch/minor PRs open but never
merge.

## Useful commands

```sh
docker compose -f <folder>/docker-compose.yml config   # validate syntax
```

## Available subagents

- `docker-compose-reviewer`: reviews compose files against best practices for Raspberry Pi.
- `deployment-security-auditor`: audits security exposure before deploying (services sit
  behind the tunnel but are still reachable from the wider internet through it).
- `service-scaffolder`: generates the structure for a new deploy-only service following the
  hub's pattern.

## Manual steps outside git (not automatable from this repo)

One-time bootstrap:
- `KOMODO_WEBHOOK_SECRET` is not a GitHub Actions secret (there's no workflow to consume it):
  it's the value Komodo Core validates incoming webhooks against (`komodo/core/.env.example`),
  and must match the "Secret" field entered when creating each Stack's native GitHub webhook.
- Enable "Allow auto-merge" in GitHub repo settings, install the Renovate GitHub App.
- Deploy Komodo Core (`komodo/core/.env.example` → `.env` → `docker compose up -d`), connect
  the rest of the fleet (`komodo/CONNECT-SERVERS.md`), then bootstrap the first `ResourceSync`
  in the Komodo UI (it can't self-create from a file it isn't reading yet) and register its
  webhook.

Per service, before its first deploy:
- Create any bind-mount directories it needs (see the `mkdir` line in its
  `docker-compose.yml` header comment, if any).
- Create the Komodo Variables its `komodo/stacks/<name>.toml` interpolates (see "Secrets via
  Komodo Variables" above) — check that file's `environment` block for the exact names.
- Configure the GitHub webhook toward that Stack's `/deploy` in Komodo.
- Any further first-boot step (setting a master password, importing data, host-level prep) is
  documented in that service's own `docker-compose.yml`/`.env.example` — check there, it's
  not duplicated here.
