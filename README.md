# granite-claude-skill

A [Claude Code](https://claude.com/claude-code) skill for deploying and operating applications on [Granite](https://granite.so) without leaving the terminal. Say *"deploy this to Granite"* or *"tail the prod logs"* and the agent runs the right `novps` CLI commands.

> The platform is **Granite** (dashboard: `dashboard.granite.so`). The CLI binary is still `novps`, the manifest is still `novps.yaml`, and PATs still start with `nvps_`.

## What it does

- **Discover** before asking: inspects git remotes, CI workflows, compose files, and framework layout, then proposes a manifest
- **Prefers your existing image**: if the repo already builds and pushes a Docker image, it deploys that image instead of rebuilding from source — and resolves the registry credentials for you
- **Monorepo-aware**: in a repo with a frontend *and* a backend it deploys each service from its own image, so a push to one doesn't rebuild the other (build-from-source has no path filter — every resource on that repo+branch rebuilds)
- **Deploy** via manifests: `novps apps apply <name> -f novps.yaml --wait`
- **Ship** a new image: `novps resources set-image` + `resources deploy`
- **Observe**: tail logs, shell into pods, port-forward databases
- **Manage**: scale, edit envs, rotate registry credentials, redeploy
- **Provision**: managed postgres / redis / mysql / mongodb, wired into `.env` and the manifest
- **Bootstrap**: generates a `novps.yaml` from an existing app via `novps apps export`

Supports both deploy modes:
- **Docker** — pull a pre-built image from any registry (ghcr, Granite registry, Docker Hub, GitLab, …)
- **GitHub** — Granite builds from source on each deploy (requires the Granite GitHub App installed)

### Registry credentials, handled properly

Private images need `user:token`. The skill tries to resolve it from the environment, `gh auth token`, or `~/.docker/config.json` before asking, verifies it with a real `docker login`, and keeps it out of the committed manifest. It also knows the sharp edge:

```bash
# credentials in novps.yaml apply ONLY at resource creation
novps resources set-image <resource_id> --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"
novps resources update    <resource_id> --tag 1.4.2 --docker-credentials "$DOCKER_USER:$DOCKER_TOKEN"
novps resources set-image <resource_id> --docker-credentials ""   # remove stored credentials
```

## Requirements

- `novps` CLI **v0.3.0 or later** — install from [cli.novps.io](https://cli.novps.io)
- A Granite Personal Access Token (`nvps_…`) — `novps auth login`
- Claude Code (or any Claude Agent SDK harness that loads skills)

## Installation

The install directory must be named `novps` — Claude Code uses the directory name as the skill name.

### As a project skill (this repo)

Clone or download this repo into your project's `.claude/skills/novps/` directory:

```bash
mkdir -p .claude/skills
git clone https://github.com/granite-so/claude-skill .claude/skills/novps
```

### As a user-level skill (all projects)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/granite-so/claude-skill ~/.claude/skills/novps
```

Claude Code will auto-load the skill on matching intents.

## Usage

Once installed, just talk to your agent:

- *"Deploy this app to Granite"*
- *"Bump the image tag to the current commit on the api resource"*
- *"The api can't pull its image — fix it"*
- *"Tail the prod logs for the last 15 minutes and grep for errors"*
- *"Give me a shell into the worker pod"*
- *"Port-forward the prod database to localhost"*
- *"Scale the api to sm:4"*

The skill will resolve resource IDs, pick the right CLI flow, surface confirmations for destructive actions, and never commit secrets to the manifest.

## Repository layout

```
.
├── SKILL.md                       # entrypoint loaded by Claude Code
├── README.md                      # this file
├── reference/
│   ├── manifest-schema.md         # full novps.yaml schema
│   ├── registry-credentials.md    # obtaining & rotating docker registry credentials
│   ├── novps.docker.example.yaml  # pre-built image starter
│   └── novps.github.example.yaml  # build-from-source starter
└── LICENSE
```

## License

MIT — see [LICENSE](./LICENSE).
