# novps.yaml manifest schema

The manifest is consumed by `novps apps apply <app-name> -f novps.yaml`. The app name is the CLI argument, not a field in the YAML. (The platform is Granite; the CLI and the manifest file kept the `novps` name.)

## Top level

| Field | Type | Notes |
|---|---|---|
| `envs` | `list<{key,value}>` | App-level environment variables. Merged into every resource; resource-level `envs` override on conflict. |
| `resources` | `list<resource>` | One entry per deployable unit (web-app, worker, cron-job). |

## Resource

| Field | Type | Notes |
|---|---|---|
| `name` | string | Unique within the app. Used to match resources across applies. Duplicates in one manifest → 422. |
| `type` | `web-app` \| `worker` \| `cron-job` | Determines which `config` fields are meaningful. |
| `source_type` | `docker` \| `github` | How Granite gets the image. |
| `source` | object | Shape depends on `source_type`. See below. |
| `config` | object | Runtime config. See below. |
| `replicas` | `{type, count}` | `type`: `xs` \| `sm` \| `md` \| `lg` \| `xl`. `count`: integer. Both are plan-limited — a too-large size or count is rejected with 422. |
| `envs` | `list<{key,value}>` | Resource-level env vars. Use `${VAR}` for `--env-file` substitution. |
| `volumes` | `list` | Persistent volume mount, at most one. See "Volumes". |

## `source` — docker

Pull a pre-built image from any registry.

```yaml
source_type: docker
source:
  type: docker
  credentials: ''              # 'user:password', a stored credential UUID, or '' for public
  name: ghcr.io/org/repo       # full image path
  tag: latest                  # image tag (prefer a git sha in CI)
```

- **`credentials` is honoured only when the resource is created.** On later applies it's ignored — change credentials with
  `novps resources set-image <id> --docker-credentials "user:token"` (or `resources update ... --docker-credentials`),
  and clear them with `--docker-credentials ""`. Full guidance in `registry-credentials.md`.
- `apps export` always writes `credentials: ''` — stored secrets are never returned.
- `tag` may also carry a digest: `sha256:...`, `@sha256:...`, or `name@sha256:...` pins the exact image.
- `tag` may be a `$VAR` reference (e.g. `$IMAGE_TAG`) resolved from the app's env vars at deploy time — distinct from `${VAR}`, which the CLI substitutes locally from `--env-file`.

## `source` — github (build from source)

Granite builds the image on each deploy. Requires:
- the Granite GitHub App installed on the repo (`https://github.com/apps/granite-so/installations/new`)
- the user's GitHub OAuth linked on Granite

Verify with `novps github list` — empty means the integration isn't set up yet, and the CLI cannot fix it (dashboard/GitHub only).

```yaml
source_type: github
source:
  type: github
  repository: owner/repo
  branch: main
  source_dir: ./               # path inside the repo (e.g. ./backend for monorepos)
  build_command: docker build .
  build_envs:                  # build-time env (optional)
    - key: NODE_ENV
      value: production
```

A push to `branch` triggers a deploy of **every** resource bound to that (installation, repository, branch) — `source_dir` scopes the build context, not the trigger. In a monorepo that means one frontend commit also rebuilds the backend, so prefer `docker` mode there.

Prefer `docker` mode when the project's CI already publishes an image — see SKILL.md, "Step 0.5" and "Step 0.6".

## `config`

| Field | Applies to | Notes |
|---|---|---|
| `command` | worker, cron-job | Startup command override. Empty = use image default. |
| `port` | web-app | HTTP port the app listens on (string, e.g. `"8080"`). |
| `internal_ports` | any | Additional exposed ports (strings). Also the allow-list for `port-forward`. |
| `schedule` | cron-job | Cron expression (e.g. `* * * * *`). |
| `allow_overlapping` | cron-job | Whether runs can overlap. |
| `restart_policy` | any | `always` (default, omitted from exports), `on-failure`, `never`. |

## Volumes

```yaml
volumes:
  - path: /data
```

- At most **one** volume per resource — more → 422.
- Size is taken from the subscription plan; a `size` key in the manifest is ignored.
- **Omitting** `volumes` entirely = "don't touch the volume". `volumes: []` = **detach** (data-loss shaped — confirm with the user).
- Changing `path` re-mounts the same volume elsewhere.
- Resize and delete are dashboard-only (Volumes page); attaching a pre-existing volume by name is also dashboard-only — the manifest can only attach a new one.

## Env var substitution

Values like `${DB_URL}` are resolved at apply time from:
1. The shell environment
2. A `.env` file passed via `--env-file`

This is how secrets stay out of the manifest. Commit the YAML; keep `.env` gitignored. Registry credentials are the exception — they don't belong in the manifest at all; set them via the CLI after create.

## Databases

Managed databases are **not** declared in this manifest. Create them with `novps databases create --engine <postgres|redis|mysql|mongodb>`, then reference their connection strings through `envs` + `--env-file`, and grant access with `novps databases allow-apps <db-id> <app-name>`.

## Example — minimal web-app from GitHub

```yaml
envs:
  - key: LOG_LEVEL
    value: info

resources:
  - name: api
    type: web-app
    source_type: github
    source:
      type: github
      repository: owner/repo
      branch: main
      source_dir: ./
      build_command: docker build .
    config:
      port: "8080"
      restart_policy: always
    replicas:
      type: sm
      count: 2
    envs:
      - key: DB_URL
        value: ${DB_URL}
    volumes: []
```

## Example — worker from a private docker image

```yaml
resources:
  - name: worker
    type: worker
    source_type: docker
    source:
      type: docker
      credentials: ''          # set after create: resources set-image <id> --docker-credentials "$U:$T"
      name: ghcr.io/org/repo
      tag: v1.2.3
    config:
      command: "python -m app.worker"
      restart_policy: always
    replicas:
      type: sm
      count: 1
    envs: []
    volumes:
      - path: /var/lib/app
```
