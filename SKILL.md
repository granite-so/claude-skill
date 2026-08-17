---
name: novps
description: Deploy applications to Granite (granite.so) and run observability/infra tasks against Granite resources via the `novps` CLI. Use when the user mentions Granite, granite.so, novps, asks to deploy, ship, or release to Granite, wants logs, a shell into a pod, to port-forward to a managed database, or to edit a novps.yaml manifest. Also use when a novps.yaml is present in the repo.
---

# Granite (novps CLI)

Native deploy and observability for [Granite](https://granite.so) via the `novps` CLI (v0.3.0+). Wraps the CLI in a set of clear flows so an agent can take intents like "deploy to Granite", "tail prod logs", or "bump the image" and turn them into the right command chain.

**Naming:** the platform is **Granite** (dashboard at `dashboard.granite.so`). The CLI binary is `novps`, the manifest file is `novps.yaml`, and PATs are prefixed `nvps_` — those names have not changed, so don't "fix" them.

## When to use this skill

Trigger on any of:
- The user says *Granite*, *granite.so*, *novps*, "deploy to Granite", "ship it", "push to Granite".
- The user asks for logs, a shell, a port-forward, or secrets tied to Granite.
- A `novps.yaml` exists in the repo.
- The user wants to scale, rollback, or restart a Granite resource.

## Prereqs — always check first

Run these before any action. If any fail, stop and surface the exact fix.

```bash
novps version          # must be >= 0.3.0
novps auth status      # must show authenticated
```

- CLI missing → install from `cli.novps.io`.
- Not authenticated → `novps auth login` (the CLI prompts for the token interactively; prefer this over passing `--token` in a shared terminal, since the token lands in shell history). The user must generate a PAT (prefix `nvps_`) in the Granite dashboard first.
- Wrong project → every command accepts `--project/-p <alias>`; default alias is `default`.

## Step 0 — Project discovery (run before asking anything)

A good deploy UX means **investigating first, asking last**. Before any question to the user, inspect the working directory and propose a plan they can accept or tweak. Never dump a list of questions cold.

Run this discovery sequence end-to-end, then summarise findings and show a draft manifest:

1. **Git / GitHub signals**
   - `git rev-parse --is-inside-work-tree` — is this a git repo?
   - `git remote get-url origin` — is there a GitHub remote? (Enables build-from-source.)
   - `git rev-parse --short HEAD` — default image tag when git is available.
2. **Existing image builds — check this before deciding anything else.** If the project already builds and pushes Docker images, deploying *that* image is the recommended path. See "Step 0.5" below for the exact signals and what to do with them.
3. **Repo shape — one deployable or a monorepo?** A repo holding a frontend *and* a backend (or several services) changes the source-mode decision on its own. See "Step 0.6".
4. **Service topology**
   - `docker-compose.yml` / `docker-compose.yaml` / `compose.yaml` — **authoritative** when present. Every service becomes a candidate resource. Read `image`, `build`, `command`, `ports`, `environment`.
   - `Procfile` — `web:` → web-app, `worker:` → worker, `release:` → one-off (pre-deploy).
   - `Dockerfile` — single-service fallback. Read `EXPOSE` for port, `CMD`/`ENTRYPOINT` for command hints.
5. **Framework conventions** (only after the above — don't invent services)
   - **Laravel** (`composer.json` contains `laravel/framework`): typical resources are `web` (nginx/php-fpm), `horizon` or `queue` worker (`php artisan queue:work` / `horizon`), `scheduler` cron (`* * * * *` → `php artisan schedule:run`).
   - **Django** (`requirements.txt` or `pyproject.toml` has `django`): `web` (gunicorn/uvicorn), plus `celery` + `celery-beat` workers if celery is present.
   - **Rails**: `web` (puma), `sidekiq` worker if `sidekiq` in Gemfile.
   - **Next.js / Node**: single `web` from `npm start`, unless `package.json` scripts hint at workers.
6. **Data services** — postgres / redis / mysql / mongodb detected as compose services?
   - Granite doesn't auto-provision from the manifest. The skill **creates managed instances** by default (don't ask whether to provision — ask *which size*, only if non-trivial).
   - Supported engines: `postgres` (versions 14, 15, 16), `redis`, `mysql`, `mongodb`. For anything else (clickhouse, elasticsearch, …), flag it — the user must either run it as a resource in the manifest or point to an external host.
   - See the "Provisioning databases" section below for the exact flow.
7. **Persistent state** — does any compose service mount a named volume (uploads, sqlite, caches)? That maps to a Granite volume on the resource; see "Volumes" below.
8. **Existing envs**
   - If `.env` or `.env.example` exists, parse keys and mirror them into the manifest's `envs` as `${KEY}` placeholders. Don't copy values.

### Discovery output — what to present to the user

After discovery, output exactly this:

```
Detected: <stack>, <N> services: <names>
Repo shape: <single deployable | monorepo: <dirs>>
Source mode: <github | docker> because <reason>
Image: <registry/image:tag>            (docker mode only)
Registry auth: <public | resolved from <source> | needs your token>
Manifest draft written to novps.yaml (review before apply)
```

Then show the draft and ask a single yes/no: *"Apply this as `<app-name>`?"* If the user rejects specific parts, edit in place rather than re-asking from scratch.

## Step 0.5 — Already building Docker images? Deploy the image.

**If the project already has an image build, recommend docker mode.** Rebuilding from source on Granite duplicates work the CI already does, and the image the team tests is the image that should run.

### Signals to look for (cheap, run them all)

```bash
ls .github/workflows/ 2>/dev/null && grep -rlE "docker/build-push-action|docker (buildx )?build|docker push|kaniko|ko build|jib" .github/workflows/
grep -nE "image:|registry|IMAGE|CI_REGISTRY" .gitlab-ci.yml 2>/dev/null
ls .drone.yml bitbucket-pipelines.yml .circleci/config.yml skaffold.yaml cloudbuild.yaml 2>/dev/null
grep -nE "docker (buildx )?build|docker push" Makefile justfile Taskfile.yml 2>/dev/null
grep -nE "^\s*image:" docker-compose.y*ml compose.y*ml 2>/dev/null   # a remote image, not a local `build:`
```

Extract the **image reference** from whatever you find:
- GitHub Actions: `tags:` / `images:` in `docker/build-push-action` — usually `ghcr.io/${{ github.repository }}` → resolve `owner/repo` from `git remote get-url origin`.
- GitLab CI: `$CI_REGISTRY_IMAGE` → `registry.gitlab.com/<group>/<project>`.
- Compose: an `image:` without a sibling `build:` is already a published image.
- Otherwise ask which registry the images land in — but ask *after* you've shown what you found.

Sanity-check the reference exists before drafting the manifest around it:

```bash
docker manifest inspect <image>:<tag>     # or: skopeo inspect docker://<image>:<tag>
```

### What to do with it

- `source_type: docker`, `source.name: <image>`, `source.tag: <tag>` (default `latest` on a first deploy, `$(git rev-parse --short HEAD)` afterwards — but only if CI actually tags by sha; check the workflow's `tags:` list first).
- Then resolve registry credentials — **next section**. A private image without credentials deploys to `ImagePullBackOff`, which is the single most common first-deploy failure.
- Tell the user *why*: "your CI already publishes `ghcr.io/org/app`, so Granite will pull that image instead of rebuilding from source."

**When to still pick github mode:** no image build exists anywhere, the repo holds exactly one deployable, and the user explicitly wants Granite to build on every push (no CI of their own). Build-from-source needs the Granite GitHub App — see below.

## Step 0.6 — Monorepo? Recommend docker mode.

**A repo that holds a frontend *and* a backend (or several services) should deploy from images, not from source.** In github mode a push fans out to **every resource matching (installation, repo, branch)** — there is no path filter. Change one CSS file in `web/` and the Go API in `api/` rebuilds and redeploys too: double build minutes, an unnecessary rollout on a service nobody touched, and a deploy queue that serialises behind itself.

### Signals

```bash
find . -name Dockerfile -not -path "*/node_modules/*" -not -path "*/.git/*" | head   # >1 → monorepo
ls pnpm-workspace.yaml lerna.json nx.json turbo.json rush.json go.work 2>/dev/null
jq -e '.workspaces' package.json 2>/dev/null                                        # npm/yarn workspaces
grep -nE '^\s*\[workspace\]' Cargo.toml 2>/dev/null
ls -d frontend backend web api client server apps packages services 2>/dev/null      # conventional layout
grep -nA2 "build:" docker-compose.y*ml 2>/dev/null | grep -E "context:"              # several build contexts
```

Two or more independently buildable services in one git repo = monorepo. A `packages/` tree of libraries consumed by one app is *not* — that's still a single deployable.

### What to recommend

1. **`source_type: docker` for every resource**, one image per service: `ghcr.io/org/repo-web`, `ghcr.io/org/repo-api`. Separate images are what make the services independently deployable.
2. **Path-filtered CI** so only the touched service rebuilds. If the user already has workflows, check they carry `paths:` filters — a monorepo workflow without them rebuilds everything anyway, and that's worth pointing out. If they have no CI, offer to scaffold one per service:
   ```yaml
   # .github/workflows/api.yml
   on:
     push:
       branches: [main]
       paths: ['api/**', '.github/workflows/api.yml']
   ```
   (Build + push `ghcr.io/org/repo-api:${{ github.sha }}` and `:main` from `./api`.)
3. **Keep push-to-deploy.** Docker mode doesn't cost the user automatic deploys — pick one:
   - **Moving tag** (`main`, `latest`): Granite watches the registry and redeploys on its own when the digest behind the tag changes. Nothing to wire up. (Not for cron-job resources — those are never auto-deployed.)
   - **Immutable sha tag**: the CI job ships it explicitly after pushing the image:
     ```bash
     novps resources set-image <resource-id> --tag ${GITHUB_SHA::7}
     novps resources deploy <resource-id>
     ```
     Deterministic and auditable — prefer it for production.

Say the trade-off out loud, briefly: *"`web/` and `api/` live in one repo, so build-from-source would rebuild both on every push. I'll deploy each from its own image and let CI rebuild only what changed."*

**When github mode is still fine in a monorepo:** only one directory is actually deployed to Granite (the rest is libraries, docs, or infra), or the user wants it despite the rebuild cost. Each resource gets its own `source_dir` — that scopes the *build context*, not the *trigger*.

## Choosing `source_type` — decide, don't ask

```
Does the project already build & push a Docker image? (Step 0.5 signals)
├── Yes → source_type: docker   ← RECOMMENDED: reuse the image CI already publishes
│         └── resolve registry credentials (see "Docker registry credentials")
└── No
    ├── Monorepo — 2+ deployable services in one repo? (Step 0.6 signals)
    │   └── Yes → source_type: docker  ← RECOMMENDED: one image per service, so a
    │             push to the frontend doesn't rebuild the backend. Offer to set up
    │             path-filtered CI; keep auto-deploy via a moving tag or set-image.
    ├── Has GitHub remote (git remote get-url origin → github.com/...)?
    │   └── novps github list → non-empty?
    │       ├── Yes → source_type: github  ← default for a single-deployable repo with no CI
    │       └── No  → offer to open the GitHub App install page in the browser.
    │                 After install, rerun `novps github list` and continue.
    └── No GitHub remote → source_type: docker
        ├── novps registry list → has a namespace?
        │   ├── Yes → use it (single auth, no cross-registry creds)
        │   └── No  → direct user to create a namespace in the Granite dashboard
        └── Or any external registry the user already uses (ghcr, Docker Hub, GitLab).
```

**GitHub App install — open the browser for the user.** When `novps github list` is empty, don't just print the URL. Offer to open it:

```bash
# macOS
open https://github.com/apps/granite-so/installations/new
# Linux
xdg-open https://github.com/apps/granite-so/installations/new
# Windows (PowerShell)
start https://github.com/apps/granite-so/installations/new
```

Pick the right command for the user's platform (`uname -s` → `Darwin` | `Linux` | other). Say something like *"Opening the Granite GitHub App install page — pick the repo, hit install, then tell me to continue."* After they say go, rerun `novps github list` to confirm, then proceed with the deploy.

**Default image tag:**
- First deploy (no previous image): `latest`.
- Subsequent deploys in a git repo: `$(git rev-parse --short HEAD)`.
- Only ask the user if they explicitly want to pick the tag.

## Docker registry credentials

Needed for any private image. Format is **`username:password`** (a single string), or the UUID of a credential the project already stores. Granite dedupes credentials by hash, so re-passing the same `user:token` never creates a duplicate.

### ⚠️ Manifest only sets credentials on **create**

`source.credentials` in `novps.yaml` is read **only when the resource is first created**. On every later `apps apply` it is ignored — editing that field does nothing. To change or remove credentials on an existing resource, use the CLI:

```bash
novps resources set-image <resource_id> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"
novps resources update   <resource_id> --tag 1.4.2 --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"
novps resources set-image <resource_id> --docker-credentials ""     # remove stored credentials
```

If a deploy fails with `ImagePullBackOff` / `401 Unauthorized` on an existing resource, this is almost always the cause — don't re-apply the manifest, run `set-image` and then `novps resources deploy <resource_id>`.

### Recommended flow — keep secrets out of the manifest

1. Draft the manifest with `credentials: ''`.
2. `novps apps apply <app> -f novps.yaml --env-file .env --wait` (the first pull may fail — expected).
3. `novps resources set-image <resource_id> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"`
4. `novps resources deploy <resource_id>` and tail logs until healthy.

Inline `credentials: "user:token"` in the manifest is only acceptable when `novps.yaml` is gitignored and the user asked for it explicitly. Say so out loud before doing it.

### Try to resolve credentials yourself — in this order

1. **Is the image public?** `docker manifest inspect <image>:<tag>` with no login. Success → `credentials: ''`, stop here. Don't ask for a token that isn't needed.
2. **Environment variables already in the shell:** `DOCKER_USER`/`DOCKER_USERNAME`, `DOCKER_TOKEN`/`DOCKER_PASSWORD`, `GHCR_TOKEN`, `CR_PAT`, `GITHUB_TOKEN`, `GITLAB_TOKEN`, `REGISTRY_USER`/`REGISTRY_PASSWORD`. Check names only (`env | grep -iE '^(docker|ghcr|cr_pat|registry|gitlab)' | cut -d= -f1`) — never echo values.
3. **ghcr.io + `gh` CLI installed:**
   ```bash
   gh auth status && gh api user -q .login          # username
   gh auth token                                     # password
   ```
   If the token lacks `read:packages`, the pull 401s → `gh auth refresh -h github.com -s read:packages`.
4. **Local docker login:** `~/.docker/config.json` → `auths["<registry>"].auth` is base64 `user:pass`. If the entry is empty and `credsStore`/`credHelpers` is set, ask the helper: `echo <registry> | docker-credential-<helper> get`.
5. **Granite's own registry** (`novps registry list` shows a namespace): the robot credentials live in the dashboard → Registry → the namespace. Point the user there rather than guessing.
6. **CI config as a hint:** the workflow tells you *which* secret it uses (`secrets.GHCR_TOKEN`, `${CI_REGISTRY_PASSWORD}`, …). That names what the user needs to mint — note that GitHub Actions' built-in `GITHUB_TOKEN` is ephemeral and cannot be reused here.

**Verify before wiring it in** — a bad credential costs a failed rollout:

```bash
echo "$DOCKER_TOKEN" | docker login <registry> -u "$DOCKER_USER" --password-stdin
docker manifest inspect <image>:<tag>
```

### If you can't resolve them — ask, with instructions

Don't ask "what are your registry credentials?". Tell the user exactly what to mint, with the scope, and how to hand it over. Per-registry step-by-step instructions live in `reference/registry-credentials.md` — read it and paste the relevant block. Short form for ghcr.io:

> I need a read-only token for `ghcr.io/org/app`:
> 1. https://github.com/settings/tokens → *Generate new token (classic)*
> 2. Scope: **`read:packages`** only. Expiry: your call.
> 3. Then run, in this terminal (leading space keeps it out of history):
>    ```bash
>     export DOCKER_USER=<your-github-login>
>     export DOCKER_TOKEN=<the-token>
>    ```
> Tell me when it's set — I'll read only the variable names, never the value.

Safety rules when handling tokens:
- Prefer `export` in the user's shell over pasting the token into chat; if they do paste it, note that it now lives in the session transcript and should be rotated after.
- Never echo a token, never write it into `novps.yaml`, never commit it.
- `set -o history` caveat: suggest `HISTCONTROL=ignorespace` + a leading space, or `read -rs DOCKER_TOKEN`.

## Provisioning databases (postgres / redis / mysql / mongodb)

When discovery detects a database service in `docker-compose.yml` (or via framework conventions like Laravel needing a DB), the skill **provisions the managed instance automatically** as part of the deploy flow — don't stop and ask whether to create it.

### Flow

1. **Create** (default size `sm`, bump only if the user said "prod" or similar):
   ```bash
   novps databases create --engine postgres -s sm --postgres-version 16 --wait --json
   novps databases create --engine redis    -s sm --wait --json
   ```
   Parse the returned `id` from JSON.
2. **Read credentials** in env format:
   ```bash
   novps databases get <db-id> --format env --show-password
   ```
   Append the output lines directly into the repo's `.env` (creating it if missing). This gives `DATABASE_URL`, `REDIS_URL`, etc. — whatever the CLI emits, pass through as-is, don't rename.
3. **Grant the app access**:
   ```bash
   novps databases allow-apps <db-id> <app-name>
   ```
   Required — databases are isolated by default. Do this before `apps apply`, or the first deploy will come up unable to connect.
4. **Mirror into the manifest**: ensure the resource `envs` reference the keys as `${DATABASE_URL}`, `${REDIS_URL}`, etc. Apply picks them up via `--env-file .env`.

### Defaults

- Size: `sm` for new/dev apps. Ask only if the user mentions production/scale.
- Postgres version: `16` (latest supported) unless the project pins a version in compose or docs.
- Node count: `1` — replicas are a separate decision the user should make explicitly. MySQL HA runs 1 or 3 nodes; Postgres read replicas are a separate `databases replica` command.

### When *not* to auto-provision

- An existing Granite database with a matching name is already in `databases list` — use it, don't create a duplicate. Offer the user: "found existing `<name>` postgres, wire to it? (y/n)".
- The user has external managed DBs (RDS, Neon, Upstash) — let them paste the URL, don't create anything.
- The engine isn't supported (clickhouse, elasticsearch, …) — tell the user, suggest either self-hosting as a resource or an external provider.

## Volumes (persistent storage)

- A resource may have **at most one volume**, declared as `volumes: [{ path: /data }]`. Size comes from the subscription plan — don't put a size in the manifest, it's ignored.
- **Omitting** `volumes` means "leave the volume alone". An explicit `volumes: []` **detaches** it. Never emit `[]` on an existing stateful resource unless the user asked to detach — that's a data-loss-shaped action.
- Changing `path` re-mounts the same volume at the new path.
- Resize and delete are dashboard-only (Volumes page); the manifest can't do them. Deleting an orphaned volume destroys its data.
- Workers/web-apps that write uploads, sqlite files, or caches to disk need this — a compose `volumes:` entry is the signal.

## The deploy flow

Granite is **manifest-driven**. The source of truth is `novps.yaml` at the repo root. The app name is not in the YAML — it's the CLI argument.

### 1. Has a `novps.yaml`?

```bash
novps apps apply <app-name> -f novps.yaml --wait
```

- `apps apply` **creates the app if it doesn't exist** and updates it if it does. No separate "create" step needed.
- Resolve the manifest path from the repo root, not cwd — if the agent is invoked from a subdirectory, `-f` must still point at the real location.
- Add `--env-file .env` if a `.env` exists in the repo, so `${VAR}` placeholders resolve. `${VAR}` is filled from the merged shell env + `.env` file.
- Add `--dry-run` first any time `--prune` is being used or when the manifest was edited significantly. Show the user the dry-run output, get confirmation, then re-run without `--dry-run`.
- `--wait` blocks until rollout finishes. Pair with `--json` when the agent needs to parse the result rather than show it to the user.
- Remember: **apply never updates docker credentials** on an existing resource. Use `resources set-image --docker-credentials`.

### 2. No manifest yet? Bootstrap.

```bash
novps apps list                         # pick the app
novps apps export <app-name> -o novps.yaml
```

Do **not** pass `--include-secrets` by default. If the user explicitly wants secret values in the file, warn first — that value lands in git unless they add `novps.yaml` to `.gitignore`. Prefer the `${VAR}` + `--env-file` pattern.

Export always emits `credentials: ''` for docker sources — stored registry credentials are never handed back. That empty string is not a diff; don't "restore" it by re-applying secrets.

After export, show a diff of what was exported, then let the user edit before applying.

### 3. No app exists yet at all?

Run **Step 0 — Project discovery** (above), pick `source_type` via the decision tree, draft a `novps.yaml` from detected services, show it to the user, then apply. See `reference/novps.docker.example.yaml` and `reference/novps.github.example.yaml` for starter shapes; schema in `reference/manifest-schema.md`.

## Fast paths (no manifest edit needed)

Use these when the intent is narrow — skip `apps apply`, go direct.

| Intent | Command |
|---|---|
| Ship a new image tag | `novps resources set-image <resource-id> --tag <sha>` then `novps resources deploy <resource-id>` |
| Fix / rotate registry auth | `novps resources set-image <resource-id> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"` |
| Drop stored registry auth (image went public) | `novps resources set-image <resource-id> --docker-credentials ""` |
| New tag + new credentials in one go | `novps resources update <resource-id> --tag <sha> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"` |
| Change a single env | `novps resources set-env <resource-id> KEY=VALUE` (merges by default) |
| Replace all envs | `novps resources set-env <resource-id> --replace K1=V1 K2=V2` — **confirm with user first** |
| Scale | `novps resources scale <resource-id> -r sm:2` (sizes: `xs` `sm` `md` `lg` `xl`) |
| Restart (redeploy same image) | `novps resources deploy <resource-id>` or `novps apps deploy <app-id>` |
| Edit image + don't deploy yet | `novps resources update <resource-id> --image X --tag Y --no-deploy` |

Default tag strategy: `latest` for a first deploy, `$(git rev-parse --short HEAD)` once the project has a git repo and an image has been shipped before.

## Observability

| Intent | Command |
|---|---|
| Tail logs | `novps resources logs <resource-id> -f --since 10m` |
| Search logs | `novps resources logs <resource-id> --since 1h --search "error"` |
| Logs for one pod | `novps resources logs <resource-id> --pod <pod-name>` |
| Shell into pod | `novps resources connect <resource-id>` |
| Resource status | `novps resources get <resource-id>` |
| List app resources | `novps apps resources <app-id>` |
| Port-forward a resource | `novps port-forward resource <resource-id> <remote-port> [-l <local-port>]` |
| Port-forward a database | `novps port-forward database <database-id> [-l <local-port>]` |

When debugging "why is X broken", default to: `resources get` → `resources logs --since 15m --search error` → offer `resources connect` if the user wants to poke inside.

**Image-pull failures** (`ImagePullBackOff`, `401`, `manifest unknown`) are not app bugs — check, in order: the tag exists in the registry (`docker manifest inspect`), the image is private (needs credentials), the stored credential is stale (rotate via `set-image --docker-credentials`).

## Resolving IDs from names

The user thinks in names, the CLI wants IDs. Always resolve before acting:

```bash
novps apps list --json                        # find app_id by name
novps apps resources <app-id> --json          # find resource_id by name
novps databases list --json                   # find database_id by name
```

## Databases & storage (read-heavy, some writes)

- `novps databases list|get|create|delete|resize|allow-apps|replica|backups|pool|pg-db|pg-user`
- `novps storage list|create|delete|set-access|files|keys`

For destructive ops (`delete`, `resize`, `set-access`), always show the user what will happen and get explicit confirmation. These are not reversible from the CLI.

## Safety rails

- Never commit real secret values into `novps.yaml`. Use `${VAR}` + `--env-file .env` and ensure `.env` is gitignored.
- Registry credentials never belong in a committed manifest — set them with `resources set-image --docker-credentials` after create.
- `--prune` on `apps apply` deletes resources missing from the manifest — always dry-run first.
- `volumes: []` on an existing resource detaches its persistent volume — confirm before emitting it.
- `set-env --replace` wipes existing envs — merge is the default for a reason.
- `databases delete` and `storage delete` are permanent — require explicit user confirmation and echo the target ID back before running.
- Don't run `auth login` with a token the user pastes in plain chat without noting the token is now in session history.

## Common recipes

**"Deploy this repo to Granite"** (happy path):
1. Check prereqs.
2. Is there a `novps.yaml`? If yes → skip to step 7.
3. **Project discovery** (Step 0), including the image-build check (Step 0.5) and the monorepo check (Step 0.6). Pick `source_type` via the decision tree.
4. **Docker mode**: resolve the image reference + registry credentials; verify the image is pullable.
5. **Provision data services** detected in compose (postgres/redis/mysql/mongodb) — `databases create --wait`, append `databases get --format env --show-password` output to `.env`, `databases allow-apps`.
6. Draft `novps.yaml` from detected application services (web/worker/cron as inferred), referencing `.env` keys as `${VAR}`, `credentials: ''`. Show it, get one yes/no.
7. `novps apps apply <name> -f novps.yaml --env-file .env --wait`.
8. Private image? `novps resources set-image <id> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"` → `novps resources deploy <id>`.
9. Tail logs on changed resources until healthy.

**"Ship the latest commit"**:
1. `sha=$(git rev-parse --short HEAD)`, ensure the image for that sha exists in the registry (`docker manifest inspect`) — CI may still be building it.
2. `novps resources set-image <id> --tag $sha`
3. `novps resources deploy <id>`
4. `novps resources logs <id> -f --since 2m` until ready.

**"Deploy this monorepo (frontend + backend)"**:
1. Discovery → confirm two deployables (two Dockerfiles / workspace config / `frontend` + `backend` dirs).
2. Explain once: build-from-source would rebuild both on every push, so each service deploys from its own image.
3. Per service: image `ghcr.io/org/repo-<svc>`, `source_type: docker`, own `config.port`/`command`; one `novps.yaml` with both resources.
4. CI: one workflow per service with `on.push.paths` scoped to that service's directory. Offer to write them if missing.
5. Apply once, attach registry credentials per resource if the images are private, deploy, tail logs.
6. Tell the user how redeploys happen from here: moving tag → automatic on digest change; sha tag → `set-image --tag` + `deploy` from CI.

**"The image moved to a private registry"**:
1. `novps resources update <id> --image <new-image> --tag <tag> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"`
2. `novps resources logs <id> -f --since 2m` — watch for pull errors.

**"Something's broken in prod"**:
1. `novps apps resources <app-id>` → find unhealthy resource.
2. `novps resources get <id>` → status, recent events.
3. `novps resources logs <id> --since 15m --search error`.
4. Offer `novps resources connect <id>` for deeper debugging.

**"Connect to the prod db locally"**:
1. `novps databases list` → find id.
2. `novps port-forward database <id> -l 5433`.
3. Print the `psql postgres://...@localhost:5433/...` string using `novps databases get <id>`.

## References

- `reference/manifest-schema.md` — full YAML schema, field by field.
- `reference/registry-credentials.md` — how to obtain registry credentials, per registry, with copy-paste user instructions.
- `reference/novps.docker.example.yaml` — pre-built image starter.
- `reference/novps.github.example.yaml` — build-from-source starter.
- Granite docs: https://docs.granite.so
