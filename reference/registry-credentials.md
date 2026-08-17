# Docker registry credentials

Everything about getting a private image to pull on Granite.

## The contract

- Format: a single string `username:password`. The first `:` splits it — passwords may contain `:`.
- Alternative: the **UUID** of a credential the project already stores (shown in the dashboard next to the Docker image). Passing the UUID reuses the stored secret without re-typing it.
- Granite hashes the credential and dedupes it project-wide, so passing the same `user:token` for a second image is safe and creates no duplicate.
- Public image → use `''` (empty). Don't attach credentials you don't need.
- Stored passwords are write-only: the API returns a masked value (`******abc`) and `apps export` emits `credentials: ''`. There is no way to read one back.

## Where credentials can be set

| Path | Creates | Updates | Notes |
|---|---|---|---|
| `novps.yaml` → `source.credentials` | ✅ | ❌ | **Read only when the resource is created.** Ignored on every later `apps apply`. |
| `novps resources set-image <id> --docker-credentials "..."` | — | ✅ | The way to change credentials. |
| `novps resources update <id> --tag X --docker-credentials "..."` | — | ✅ | Tag + credentials in one call. |
| `novps resources set-image <id> --docker-credentials ""` | — | ✅ | Removes the stored credential (image went public). |
| Dashboard → resource → Docker image | — | ✅ | Also where existing credentials are listed per registry. |

```bash
novps resources set-image <resource_id> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"
novps resources update    <resource_id> --tag 1.4.2 --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"
novps resources set-image <resource_id> --docker-credentials ""
```

After changing credentials, redeploy: `novps resources deploy <resource_id>`.

## Resolution order (try before asking)

1. **Public?** `docker manifest inspect <image>:<tag>` with no login. Works → `credentials: ''`.
2. **Shell env** — check *names* only, never values:
   ```bash
   env | grep -iE '^(docker|ghcr|cr_pat|registry|gitlab|harbor)' | cut -d= -f1
   ```
3. **`gh` CLI** (ghcr.io only):
   ```bash
   gh auth status && gh api user -q .login && gh auth token
   ```
   401 on pull → the token lacks `read:packages`: `gh auth refresh -h github.com -s read:packages`.
4. **`~/.docker/config.json`**:
   ```bash
   jq -r '.auths | to_entries[] | "\(.key)\t\(.value.auth // "via-helper")"' ~/.docker/config.json
   # decode one: echo <base64> | base64 -d   → user:pass
   ```
   Empty `auth` + a `credsStore`/`credHelpers` entry → ask the helper:
   ```bash
   echo <registry-host> | docker-credential-<helper> get
   ```
   (macOS default helper is `osxkeychain`, Linux often `secretservice` or `pass`.)
5. **Granite's own registry** — `novps registry list` shows the project's namespaces; the robot account and its secret live in the dashboard under Registry → namespace. Prefer this registry when the user has no registry of their own: one auth, no cross-registry juggling.
6. **CI config** — it names the secret the pipeline uses (`secrets.GHCR_TOKEN`, `$CI_REGISTRY_PASSWORD`, …), which tells the user what to mint. GitHub Actions' built-in `GITHUB_TOKEN` is per-run and cannot be reused outside the workflow.

**Always verify before wiring in:**

```bash
echo "$DOCKER_TOKEN" | docker login <registry-host> -u "$DOCKER_USER" --password-stdin
docker manifest inspect <image>:<tag>
docker logout <registry-host>
```

## Registry host, from an image name

The first path segment is the registry only if it looks like a host (contains `.` or `:`, or is `localhost`); otherwise the image is on Docker Hub. All Docker Hub aliases (`index.docker.io`, `registry-1.docker.io`, `registry.hub.docker.com`) are the same registry — credentials for one work for all.

| Image | Registry host |
|---|---|
| `ghcr.io/org/app` | `ghcr.io` |
| `registry.gitlab.com/group/app` | `registry.gitlab.com` |
| `org/app`, `redis` | `docker.io` |
| `localhost:5000/app` | `localhost:5000` |

## Per-registry instructions to hand the user

Paste the relevant block verbatim. Ask them to `export` the values in their shell rather than pasting the token into chat.

### GitHub Container Registry (ghcr.io)

> 1. https://github.com/settings/tokens → **Generate new token (classic)**
> 2. Scope: **`read:packages`** only (Granite just pulls). Set any expiry you like.
> 3. Copy the token, then in this terminal:
>    ```bash
>     export DOCKER_USER=<your-github-username>
>     export DOCKER_TOKEN=ghp_xxxxxxxx
>    ```
> (The leading space keeps it out of shell history if `HISTCONTROL=ignorespace` is set.)
>
> Fine-grained tokens also work: give the token *Packages: read* on the owning org/user.
> Org packages: make sure the package's *Manage Actions access* / visibility lets your account read it.

### Docker Hub

> 1. https://app.docker.com/settings/personal-access-tokens → **Generate new token**
> 2. Permissions: **Read-only**.
> 3. Then:
>    ```bash
>     export DOCKER_USER=<your-dockerhub-username>
>     export DOCKER_TOKEN=dckr_pat_xxxxxxxx
>    ```
> Use the access token, not your account password — Hub rejects passwords for automated pulls when 2FA is on.

### GitLab Container Registry (registry.gitlab.com)

> Project → **Settings → Repository → Deploy tokens** → *Add token*
> 1. Scope: **`read_registry`**.
> 2. GitLab shows a username (e.g. `gitlab+deploy-token-123`) and a token — both are needed:
>    ```bash
>     export DOCKER_USER=gitlab+deploy-token-123
>     export DOCKER_TOKEN=<the-deploy-token>
>    ```
> A personal access token with `read_registry` also works (username = your GitLab login).

### AWS ECR

> ECR passwords are 12-hour tokens, so a static credential will expire and the next pull fails.
> Options, best first:
> - Push the image to a registry with long-lived credentials (ghcr.io, Docker Hub, the Granite registry).
> - Create a pull-only IAM user and refresh the credential on a schedule:
>   ```bash
>    export DOCKER_USER=AWS
>    export DOCKER_TOKEN=$(aws ecr get-login-password --region <region>)
>   ```
>   then re-run `novps resources set-image <id> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"` before it expires.
>
> Flag the expiry to the user explicitly — a silently expiring credential looks like a random outage weeks later.

### Google Artifact Registry / GCR

> Service-account key, read-only:
> 1. Create a service account with **Artifact Registry Reader**.
> 2. Download a JSON key.
> 3. Then:
>    ```bash
>     export DOCKER_USER=_json_key
>     export DOCKER_TOKEN="$(cat key.json | tr -d '\n')"
>    ```
> The JSON key contains `:` characters — that's fine, only the first `:` splits user from password.

### Self-hosted / Harbor / other

> Create a robot or read-only account in the registry UI, then:
> ```bash
>  export DOCKER_USER=<user>
>  export DOCKER_TOKEN=<password-or-token>
> ```
> If the registry runs on plain HTTP or a self-signed cert, say so — Granite pulls over TLS and an untrusted cert fails the pull.

## Handling tokens safely

- Prefer `export` in the user's shell over a paste into chat. If they paste it, tell them it's now in the session transcript and should be rotated when done.
- Never echo a token, never write one into `novps.yaml`, never commit one.
- `read -rs DOCKER_TOKEN` avoids both the screen and the history file.
- Read-only scopes only. Granite never pushes to the user's registry.
- Rotating a token later: `novps resources set-image <id> --docker-credentials "$DOCKER_USER:$NEW_TOKEN"` then `novps resources deploy <id>`. The old credential stays attached to nothing and is harmless.

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` + `401 Unauthorized` | Missing / wrong / expired credential | `set-image --docker-credentials`, then `deploy` |
| `manifest unknown` / `not found` | Tag doesn't exist yet (CI still building), or typo in the image path | `docker manifest inspect`; wait for CI or fix the tag |
| Pull worked locally, fails on Granite | Local `docker login` session — the image is private and Granite has no credential | Attach credentials explicitly |
| Credentials edited in the manifest, nothing changed | `source.credentials` is create-only | Use `resources set-image --docker-credentials` |
| ECR/GAR pull fails weeks later | Short-lived token expired | Rotate, or move the image to a registry with static credentials |
