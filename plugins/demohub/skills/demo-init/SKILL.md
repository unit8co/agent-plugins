---
name: demo-init
description: Set up the current repo to deploy on unit8 DemoHub — check what already exists (on DemoHub via MCP, and in the repo on disk), scaffold a Dockerfile/demo.yaml if missing, register the demo + deploy key, and drive it through build/start until it's live at <slug>.demo.unit8.io. Use when the user runs /demo-init, or asks to deploy/init/prepare this repo for DemoHub, set up a demo, or get this app onto DemoHub.
---

# demo-init

Gets a git repo from "has an app in it" to "running on DemoHub," doing as much of it as
this session can reach and asking only for what it genuinely can't determine itself. Full
human-facing walkthrough, if you want the wider context: `docs/CREATING_A_DEMO.md` in
`unit8co/demohub` (https://github.com/unit8co/demohub/blob/main/docs/CREATING_A_DEMO.md).

Requires the `demohub` MCP server (this plugin bundles it). If tools like `create_demo` /
`verify_deploy_key` aren't available, stop and tell the user to install/enable the
`demohub` plugin first.

Always target the single-container shape — a `Dockerfile`, optionally paired with a
`demo.yaml` for port/health/sizing/persistence. Don't ask the user to choose a build
shape; there's nothing to decide.

## 1. Figure out what already exists

Before touching anything, work out where this repo currently stands:

1. **On DemoHub** — call `list_demos` and look for one whose `repo_ssh_url`/repo matches
   this repo's `git remote get-url origin` (compare loosely: same owner/repo regardless of
   ssh vs https form). If you find one, `get_demo`/`get_status` it to see how far it
   already got (registered but unverified key, verified but never built, built but
   stopped, already running, etc.) and resume from there instead of starting over.
2. **In the repo** — check the repo root for `Dockerfile` and `demo.yaml`. Existing files
   mean that step's done; don't overwrite them.

Skip straight to whichever step matches the state you find — e.g. a demo that's already
registered and verified goes straight to step 4.

## 2. Scaffold what's missing

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

## 3. Register the demo and the deploy key

Skip anything already done per the step 1 check.

1. `create_demo(name, repo_ssh_url, branch)` — `repo_ssh_url` can be the repo's HTTPS or
   SSH URL. If a sensible `name` isn't obvious (repo name is generic, taken, or the user
   hasn't said), ask the user for one rather than guessing. Returns `deploy_key_public`.
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
   repo's spec — a mistake in step 2's files shows up here, not before.

## 4. Build, start, and confirm it's live

Don't stop at "registered" — the job isn't done until the demo is actually running.

1. `start(slug)` — builds an ACR image first if none exists yet, then runs the demo. If
   the result carries a `build_id`, poll `get_build(slug, build_id)` until it's
   succeeded/failed, keeping the user posted on progress; a finished build also starts the
   demo.
2. If the build fails, or the demo doesn't come up healthy, pull `get_logs(slug)` and fix
   the underlying issue (Dockerfile, `demo.yaml` port/health path, missing env var) rather
   than leaving the user with a failed state — then retry `start`/`build`.
3. `get_status(slug)` (or watch the build outcome) to confirm it's actually running before
   declaring done.
4. If the app needs configuration to boot (API keys, feature flags, etc.), use
   `set_env(slug, vars)` — ask the user for values you can't infer.

Only stop short of a running demo if you hit something you genuinely can't resolve
yourself (missing secret, failing health check with an unclear cause, GitHub deploy key
step waiting on the user) — say exactly what's blocking it and what the user needs to do.

## 5. Report back

Tell the user what you did in your own words — which files you wrote/found, the demo's
slug, whether the deploy key verified, and confirm the URL it's now live at
(`<slug>.demo.unit8.io`) or exactly what's still blocking that.
