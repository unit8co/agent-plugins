---
name: demo-init
description: Set up the current repo to deploy on unit8 DemoHub — check what already exists (on DemoHub via MCP, and in the repo on disk), scaffold a Dockerfile (and a demo.yaml only when the defaults don't fit), register the demo + deploy key, and drive it through build/start until it's live at <slug>.demo.unit8.io. Use when the user runs /demo-init, or asks to deploy/init/prepare this repo for DemoHub, set up a demo, get this app onto DemoHub, or debug why a DemoHub demo doesn't come up.
---

# demo-init

Gets a git repo from "has an app in it" to "running on DemoHub," doing as much of it as
this session can reach and asking only for what it genuinely can't determine itself. Full
human-facing walkthrough, if you want the wider context: `docs/CREATING_A_DEMO.md` in
`unit8co/demohub` (https://github.com/unit8co/demohub/blob/main/docs/CREATING_A_DEMO.md).

Requires the `demohub` MCP server (this plugin bundles it). If tools like `create_demo` /
`verify_deploy_key` aren't available, stop and tell the user to install/enable the
`demohub` plugin first.

Every demo is exactly one container built from a `Dockerfile`. There is no build shape to
choose and nothing to ask the user about it. `docker-compose.yml` and `k8s/` manifests in
the repo are ignored by DemoHub, so they can't help or hurt.

## The default contract (no demo.yaml)

A repo **without** a `demo.yaml` is the normal case. DemoHub then assumes exactly this, and
the Dockerfile you write or check must satisfy every line of it:

| What | Default DemoHub applies | What the image must do |
|---|---|---|
| Dockerfile | `./Dockerfile` at the repo root, build context `.` | Exist at the repo root. `.dockerignore` is honoured. |
| Build | `docker build --platform linux/amd64` on Azure Container Registry, **classic builder (no BuildKit)**, 30 min limit | No `RUN --mount=…`, no `RUN <<EOF` heredocs, no `COPY --chmod`, no `--platform=linux/arm64` / `$TARGETARCH` tricks. Plain `FROM`/`COPY`/`RUN`/`ENV`/`CMD` only; multi-stage builds are fine. |
| Port | **8000**, the only port routed | Listen on **`0.0.0.0:8000`**: not `127.0.0.1`/`localhost`, not 80, 3000, 5000, 8080, 8501 or `$PORT`. DemoHub does **not** set a `PORT` env var, so hardcode 8000 in `CMD`, or set `ENV PORT=8000` if the app reads `$PORT`. `EXPOSE` routes nothing; it only feeds a warning. |
| Health check | Plain TCP connect to port 8000 every 10 s | Accept connections on 8000. An HTTP check happens only if `demo.yaml` sets `health_path` to something other than `/`. |
| Start deadline | The pod must pass that check within **300 s** of the rollout, or the start fails with `ReplicaSet "app-…" has timed out progressing` | Start fast. Don't download models or datasets or run long migrations at boot; bake them into the image at build time instead. |
| Size | Fixed platform-wide: **0.25 vCPU, 512 MiB RAM**. A repo can't change it (`cpu`/`memory` in `demo.yaml` are ignored). | Run a production server, not a dev server (`vite`, `next dev` and `flask run` are heavy and/or bind localhost). Going over 512 MiB means OOM-kill and a restart loop. |
| User | Whatever the image's `USER` is; root is allowed | Nothing special. The stock `nginx`, `python` and `node` images work as they are. |
| Filesystem | Writable but **wiped on every stop/restart**. `/data` is a persistent disk only with `persist: true`. | Keep state you need across restarts under `/data` and set `persist: true` in `demo.yaml`. |
| Traffic | HTTPS ends at the ingress, which sends plain HTTP to the container. Served at the **root path** of `https://<slug>.demo.unit8.io`, behind Okta SSO. | Serve the app at `/`, with no path prefix. Don't hardcode `localhost` URLs in the frontend; use relative URLs. |
| Env vars | Only what's set via `set_env`/the Environment tab, plus `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_DEPLOYMENT` (shared resource) | Read config from env vars. Never commit `.env` files or real secrets. |
| Network | Outbound internet works; private IP ranges and the Azure metadata endpoint are blocked | Don't depend on a database or service you don't run inside the container yourself. |

**Reading DemoHub's state:** `get_demo` returning `"ports": []` means *nothing stored yet*,
so the default **8000** applies. It does **not** mean "any port" or "the image's `EXPOSE`".
Don't guess 8080 or 80 from it. Effective port = `demo.yaml`'s first `ports` entry if the
file sets one, otherwise 8000.

### Default Dockerfile shapes

Pick the shape that matches the repo's stack, then adapt the entry point (module path,
script name, build output dir) to what the repo actually contains. Each one already
satisfies the contract above.

Python with uv (`pyproject.toml` + `uv.lock`); FastAPI/uvicorn shown:

```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY . .
RUN uv sync --frozen --no-dev
EXPOSE 8000
CMD ["uv", "run", "--no-sync", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Python with `requirements.txt`: same, with `RUN pip install --no-cache-dir -r requirements.txt`
in place of the `uv` lines and `CMD` starting the server directly. Port flags for other
servers:
- Streamlit: `streamlit run app.py --server.port 8000 --server.address 0.0.0.0 --server.headless true`
- Gradio: `demo.launch(server_name="0.0.0.0", server_port=8000)`
- Flask (use gunicorn, not `flask run`): `gunicorn -b 0.0.0.0:8000 app:app`

Node server (Express, Next.js, etc.):

```dockerfile
FROM node:22-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build --if-present
ENV NODE_ENV=production PORT=8000 HOST=0.0.0.0
EXPOSE 8000
CMD ["npm", "start"]
```

Check that `npm start` really honours `PORT`/`HOST`. If it doesn't (Next.js wants
`next start -p 8000 -H 0.0.0.0`), put the explicit command in `CMD`.

Static site or SPA build (one HTML file, a Vite/React build output, etc.): serve it with
nginx **on 8000**. The stock nginx image listens on 80, which DemoHub doesn't route to.

```dockerfile
# (for a build step, add a `FROM node:22-slim AS build` stage and COPY --from=build its dist/)
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/
RUN printf '%s\n' \
  'server {' \
  '  listen 8000;' \
  '  root /usr/share/nginx/html;' \
  '  location / { try_files $uri /index.html; }' \
  '}' > /etc/nginx/conf.d/default.conf
EXPOSE 8000
```

Before moving on, re-check the finished Dockerfile against the table: the port is 8000
in every place it appears (`CMD`, server config, `ENV`, `EXPOSE`), it binds `0.0.0.0`, and
there's no BuildKit-only syntax.

## When to add a demo.yaml

Only when a default above genuinely doesn't fit. Making the app listen on 8000 is almost
always simpler than overriding the port. Reasons to write one:

- The app can't reasonably be moved to 8000 (a third-party image with a fixed port).
- It needs more than one port (up to 5; extra ports get `<slug>-<port>.demo.unit8.io`).
- It must keep files across restarts (`persist: true`, mounted at `/data`).
- A plain TCP check isn't enough and an HTTP readiness path is wanted (`health_path`).
- The Dockerfile isn't at the repo root (`dockerfile`/`context`).

These are the only keys DemoHub reads. Set only the ones you need; anything omitted keeps
its default:

```yaml
# Dockerfile path, relative to `context`. Default: Dockerfile
dockerfile: Dockerfile
# Build context, relative to the repo root. Default: .
context: .
# Ports the app listens on. First = primary, served at <slug>.demo.unit8.io. Default: 8000
ports:
  - container_port: 8000
    description: Web UI   # optional label, max 80 chars
# "/" (default) = TCP check only; any other path = HTTP GET that must return 2xx/3xx
health_path: /
# Persistent 8 GiB disk at /data. Default: false
persist: false
```

Don't write `cpu`, `memory` or `requires_sso`: they're ignored with a warning. Sizing is
fixed, and every port is behind SSO unless a DemoHub admin opens it. A `demo.yaml` key
also locks that field in the portal for as long as the file sets it.

## 1. Figure out what already exists

Before touching anything, work out where this repo currently stands:

1. **On DemoHub**: call `list_demos` and look for one whose `repo_ssh_url`/repo matches
   this repo's `git remote get-url origin` (compare loosely: same owner/repo regardless of
   ssh vs https form). If you find one, `get_demo`/`get_status` it to see how far it
   already got (confidentiality pending, unverified key, never built, stopped, errored,
   running) and resume from there instead of starting over.
2. **In the repo**: check the root for `Dockerfile` and `demo.yaml`. Don't overwrite
   working files, but an existing Dockerfile still has to meet the contract above. If it
   listens on the wrong port, binds localhost, or uses BuildKit-only syntax, fix it (it's
   the most common reason a demo never comes up).

Skip straight to whichever step matches the state you find.

## 2. Scaffold what's missing

1. **`Dockerfile`**, if absent: detect the stack (`pyproject.toml`/`requirements.txt` →
   Python, `package.json` → Node, a bare `index.html` → static, `go.mod` → Go, …) and
   write one from the shapes above. If you can, run `docker build` locally and start the
   container to confirm it answers on `localhost:8000`, but that's optional.
2. **`demo.yaml`**: only for one of the reasons listed under "When to add a demo.yaml".
   Otherwise don't create one.
3. Commit and push both to the branch DemoHub will build. DemoHub builds from GitHub,
   not from the local working tree, so unpushed changes don't exist for it.

## 3. Register the demo and the deploy key

Skip anything already done per the step 1 check.

1. `create_demo(name, repo_ssh_url, branch)`: `repo_ssh_url` can be the repo's HTTPS or
   SSH URL, and the repo must be in the `unit8co` GitHub org. If a sensible `name` isn't
   obvious (repo name is generic, taken, or the user hasn't said), ask the user for one
   rather than guessing. Returns `deploy_key_public`.
2. Add that key to the GitHub repo as a **read-only** deploy key. Prefer the GitHub CLI if
   it's installed and authenticated:
   ```
   gh repo deploy-key add <(printf '%s' "$DEPLOY_KEY_PUBLIC") --title demohub -R <owner>/<repo>
   ```
   (Omit `-w`/`--allow-write`: read access is all DemoHub needs, and that flag would
   grant more.) If `gh` isn't available or authenticated, print the key and tell the user
   to add it themselves: repo → **Settings → Deploy keys → Add deploy key**, "Allow write
   access" left unchecked.
3. `verify_deploy_key(slug)`: confirms the key actually landed via a real `git
   ls-remote`. Don't proceed until it succeeds. This is also the first time DemoHub reads
   the repo's spec, so read the returned `spec_error`/`spec_warnings` and fix whatever
   they name before building.
4. **Data notice.** A newly created demo is `confidentiality_pending`: hidden, and
   `start`/`build`/`restart` return HTTP 409 until the **owner** opens the demo's page in
   the portal (`https://demo.unit8.io/demos/<slug>`) and ticks "I understand" (no client,
   production or personal data). No tool can do this for them; don't try to work around
   it. Tell the user, wait for them to confirm, then continue.

## 4. Build, start, and confirm it's live

Don't stop at "registered". The job isn't done until the demo is actually running.

1. `start(slug)`: builds an image of the branch's newest commit first if needed, then
   runs the demo. If the result carries a `build_id`, poll `get_build(slug, build_id)`
   until it has succeeded or failed, keeping the user posted; a finished build also starts
   the demo.
2. Confirm with `get_status(slug)` / `get_demo(slug)` that `status` is `running` and the
   target is healthy (1/1 ready) before declaring done.
3. If it fails, find the cause from `get_build` logs, `get_logs(slug)` and `status_detail`,
   fix it in the repo, push, then `start`/`restart` again (`restart` rebuilds from the
   latest commit when auto-deploy is on):

   | Symptom | Usual cause |
   |---|---|
   | Build fails on `--mount`, `<<EOF` or `--chmod` | BuildKit-only syntax; rewrite with plain `RUN`/`COPY` |
   | `ReplicaSet "app-…" has timed out progressing`, logs show the server started fine | It listens on another port than the effective one (8000 by default) or on `127.0.0.1` |
   | Same timeout, logs empty or show a long boot | Startup takes longer than 300 s, or a wrong `health_path` returns 4xx/5xx |
   | Restart loop, `OOMKilled` | Over 512 MiB: use a production server or a lighter model |
   | Crash at boot with a missing config error | A required env var isn't set: `set_env(slug, vars)`, asking the user for values you can't infer |
   | 409 on start/build | Data notice not yet confirmed (step 3.4) |

Only stop short of a running demo if you hit something you genuinely can't resolve
yourself (missing secret, data notice not confirmed, deploy key waiting on the user, a
failure with an unclear cause). Say exactly what's blocking it and what the user needs to
do.

## 5. Report back

Tell the user what you did in your own words: which files you wrote or found (and whether
you relied on the defaults or added a `demo.yaml`, and why), the demo's slug, whether the
deploy key verified, and the URL it's now live at (`https://<slug>.demo.unit8.io`), or
exactly what's still blocking that.
