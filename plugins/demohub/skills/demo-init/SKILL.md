---
name: demo-init
description: Set up the current repo to deploy on unit8 DemoHub — ask whether it's a single-container or multi-container app, scaffold whichever of Dockerfile/demo.yaml/docker-compose.yml is missing, and register the demo + deploy key via the demohub MCP tools. Use when the user runs /demo-init, or asks to deploy/init/prepare this repo for DemoHub, set up a demo, or get this app onto DemoHub.
---

# demo-init

Gets a git repo from "has an app in it" to "registered on DemoHub with a verified deploy
key," doing as much of it as this session can reach and asking for the rest. Full
human-facing walkthrough, if you want the wider context: `docs/CREATING_A_DEMO.md` in
`unit8co/demohub` (https://github.com/unit8co/demohub/blob/main/docs/CREATING_A_DEMO.md).

Requires the `demohub` MCP server (this plugin bundles it). If tools like `create_demo` /
`verify_deploy_key` aren't available, stop and tell the user to install/enable the
`demohub` plugin first.

## 0. Look before asking

Check the repo root for `Dockerfile`, `demo.yaml`, `compose.yaml`, `compose.yml`,
`docker-compose.yaml`, or `docker-compose.yml` before asking anything. If a Compose file
already exists, skip straight to step 2 (compose path) using it. If `Dockerfile` already
exists with no Compose file, skip straight to step 2 (single-container path) — don't
re-litigate a decision the repo has already made. Only ask when the repo genuinely has
neither.

## 1. Ask which shape fits

Ask the user (don't guess): does this app need more than one process — a database, a
worker, a second service it talks to — or is it one container?

- **Single container** (default/recommended for most demos): one `Dockerfile`, optionally
  paired with a `demo.yaml` for port/health/sizing/persistence. Simpler, and what
  `unit8co/demohub-example` uses.
- **Multi-container**: a `docker-compose.yml`, inside DemoHub's supported subset (below).
  Pick this only if the app genuinely needs more than one running container — DemoHub
  charges disk-backed volumes and extra containers against the same per-demo quota, so
  don't reach for Compose just to organize a single service.

## 2a. Single container — scaffold if missing

1. **`Dockerfile`**, only if absent: detect the stack from what's in the repo
   (`pyproject.toml`/`requirements.txt` → Python, `package.json` → Node, `go.mod` → Go,
   etc.) and write a Dockerfile that builds and runs it. One hard constraint: **the image
   must run on `linux/amd64`** — never hardcode `--platform=linux/arm64` in the Dockerfile
   or its base image, that's a real failure mode on this platform, not a style preference.
   Look at `unit8co/demohub-example`'s `Dockerfile` for a Python/uv reference shape if
   useful, but match whatever this repo's own stack actually is.
2. **`demo.yaml`**, only if absent — every field optional, DemoHub defaults fill the rest:

   ```yaml
   # Dockerfile to build, relative to `context`.
   dockerfile: Dockerfile
   # Build context, relative to the repo root.
   context: .
   # Ports the app listens on. First port is primary and gets the clean URL,
   # <slug>.demo.unit8.io; extra ports get <slug>-<container_port>.demo.unit8.io.
   # requires_sso: false only for something that must be reachable without a login
   # (e.g. a webhook target) — leave true unless you have a specific reason not to.
   ports:
     - container_port: 8000
       requires_sso: true
       description: Web UI
   # 256 CPU units = 0.25 vCPU; memory in MiB.
   cpu: 256
   memory: 512
   # Path the platform health check hits on the primary port.
   health_path: /
   # Mount a per-demo Azure Files directory at /data. Default off.
   persist: false
   ```

   Set `container_port`/`health_path` to match what the app you scaffolded/found actually
   does — don't just paste the template unexamined.

## 2b. Multi-container — scaffold if missing

Write a `docker-compose.yml` covering each service the app actually needs, staying inside
this subset (full spec: `docs/COMPOSE_TRANSLATION.md` in `unit8co/demohub`):

**Supported:** `build` (or `image` without `build`), `ports`/`expose` (every service needs
one or the other — DemoHub can't infer a port from nothing), `environment`, `healthcheck`
(→ readiness probe; `start_period` becomes a startup grace period, not an initial delay),
`depends_on: {condition: service_healthy}` (→ an init container that waits for it),
`volumes` (named → a PVC; one mounter is `azuredisk`/RWO, two-or-more mounters is
`azurefile-csi`/RWX — **note SQL databases like Postgres/MySQL/SQLite don't work over the
RWX/SMB backing, so keep a database volume single-mounter**), `deploy.resources` (CPU/mem
requests+limits), `user` (numeric only).

**Rejected outright, don't write these:** bind mounts (`./src:/app` — no host filesystem on
the cluster; use a named volume or bake files into the image), `env_file` (move the values
into DemoHub's `.env` editor and reference as `${VAR}` in the compose file instead), raw
`k8s/` manifests alongside it.

**Ignored with a warning, so avoid relying on them:** `restart`, `container_name`, `tty`,
`networks`, `profiles` (only the default profile deploys).

**The two inverted-name traps**, if any service sets a custom entrypoint/command:
Compose's `entrypoint` maps to Kubernetes `command`, and Compose's `command` maps to
Kubernetes `args` — backwards from what the names suggest. Get this wrong and the
container silently runs the wrong thing.

No `demo.yaml` alongside a Compose file — Compose is the whole spec once it's present.

## 3. Register the demo and the deploy key

Skip anything already done (e.g. `create_demo` was already called earlier in this
session, or the user says the demo already exists — then just resume from wherever
`get_demo` shows it stopped).

1. `create_demo(name, repo_ssh_url, branch)` — `repo_ssh_url` can be the repo's HTTPS or
   SSH URL. Returns `deploy_key_public`.
2. Add that key to the GitHub repo as a **read-only** deploy key. Prefer the GitHub CLI if
   it's installed and authenticated:
   ```
   gh repo deploy-key add <(printf '%s' "$DEPLOY_KEY_PUBLIC") --title demohub -R <owner>/<repo>
   ```
   (Omit `-w`/`--allow-write` — read access is all DemoHub needs, and this is the flag
   that would grant more.) If `gh` isn't available or isn't authenticated, print the key
   and tell the user to add it themselves: repo → **Settings → Deploy keys → Add deploy
   key**, "Allow write access" left unchecked.
3. `verify_deploy_key(slug)` — confirms the key actually landed via a real `git
   ls-remote`. Don't proceed to build/start until this returns success; nothing else about
   the demo is editable until it does, and it's also the first time DemoHub reads the
   repo's spec — a mistake in step 2a/2b's files shows up here, not before.

## 4. Hand back control

Stop here unless the user has clearly asked you to also build/start/deploy — this skill's
job is getting the repo *ready* to deploy, not necessarily deploying it. If they do want it
running now: `start(slug)` builds automatically on a first start (poll `get_build(slug,
build_id)` if a `build_id` comes back) and the demo is live at `<slug>.demo.unit8.io` once
that lands.

Tell the user what you did and what's left in your own words — which files you
wrote/found, the demo's slug, whether the deploy key verified, and whether you also started
it or left that for them.
