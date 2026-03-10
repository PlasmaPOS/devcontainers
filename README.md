# PlasmaPOS DevContainer Images

Pre-built Docker images for DevPod cloud development environments. These images are the base layer — [PlasmaPOS/dotfiles](https://github.com/PlasmaPOS/dotfiles) handles runtime configuration (env vars, git identity, shell aliases).

## Repository Structure

```
devcontainers/
├── Dockerfile.light            # Backend/tooling projects
├── Dockerfile.full             # App projects with mobile + web testing
└── .github/workflows/build.yml # Auto-builds on push to main
```

## Images

### `light` — Backend & Tooling

**Registry:** `ghcr.io/plasmapos/devcontainers/light:latest`
**For:** dev-agent, seal, aqua

| Layer | What's installed |
|-------|-----------------|
| Base | Debian (devcontainers base image) |
| Runtime | Node.js 22 (system-wide), Bun (latest) |
| AI CLIs | Claude Code (`claude`), Codex (`codex`) — in `/usr/local/bin` |
| Dev tools | Convex CLI (via bun global) |
| System | Bun symlinked to `/usr/local/bin` for non-interactive SSH |

### `full` — Apps with E2E Testing

**Registry:** `ghcr.io/plasmapos/devcontainers/full:latest`
**For:** plasma, pile, roster, veil

Everything in `light`, plus:

| Layer | What's installed |
|-------|-----------------|
| Browser | Playwright + Chromium (browsers in `/home/vscode/.cache/ms-playwright`) |
| Mobile | Android SDK 34, platform-tools, emulator |
| AVD | Pre-configured Pixel 6 AVD (`test-device`) with Google APIs x86_64 |
| Dev tools | agent-device CLI (via bun global) |
| System | Java JDK (headless, required by Android SDK) |

## VM Sizing

| Image | Projects | GCP Machine Type | vCPU | RAM |
|-------|----------|-----------------|------|-----|
| `light` | dev-agent, seal, aqua | `n2d-standard-4` | 4 | 16 GB |
| `full` | plasma, pile, roster, veil | `n2d-standard-8` | 8 | 32 GB |

Full-image projects need 8 vCPU because running emulator + dev server + AI CLIs concurrently on 4 vCPU causes OOM.

Override per workspace:
```bash
devpod up <repo> --provider-option MACHINE_TYPE=n2d-standard-8
```

## Usage in `devcontainer.json`

Each project repo has a `.devcontainer/devcontainer.json` that references one of these images:

```json
{
  "name": "my-project",
  "image": "ghcr.io/plasmapos/devcontainers/light:latest",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "postCreateCommand": "bun install",
  "runArgs": ["--shm-size=2g"]
}
```

**Note:** Node.js is already baked into the image — do NOT add the `ghcr.io/devcontainers/features/node:1` feature (it would install a duplicate).

## Building

Images are auto-built on push to `main` via GitHub Actions.

```bash
# Manual trigger
gh workflow run build.yml

# Build locally for testing
docker build -f Dockerfile.light -t devcontainers-light .
docker build -f Dockerfile.full -t devcontainers-full .
```

## Design Decisions

### Why install AI CLIs via npm as root?

`npm install -g` as root places binaries directly in `/usr/local/bin`. This means `claude` and `codex` work in non-interactive SSH sessions without any PATH manipulation. Bun global installs go to `~/.bun/bin` which requires shell profile sourcing — fine for interactive use, but breaks when DevPod or scripts run commands via `ssh <host> 'command'`.

### Why symlink bun into `/usr/local/bin`?

Same reason — non-interactive SSH sessions don't source `.bashrc`/`.profile`. The symlink ensures `bun` and `bunx` are available everywhere. The actual bun installation lives at `/home/vscode/.bun/bin/bun`.

### Why Chromium only (no Firefox/WebKit)?

Chromium covers our testing needs. Each additional browser adds ~200 MB to the image and increases build time. Add others if needed:
```dockerfile
RUN npx playwright install --with-deps chromium firefox
```

## For Coding Agents

If you are an AI agent and need to understand the environment:

- **Bun**: `/home/vscode/.bun/bin/bun` (also at `/usr/local/bin/bun`)
- **Node**: `/usr/bin/node` (v22, system-wide)
- **Claude Code**: `/usr/bin/claude`
- **Codex**: `/usr/bin/codex`
- **Convex CLI**: `~/.bun/bin/convex` (also via `bunx convex`)
- **Playwright browsers** (full image only): `/home/vscode/.cache/ms-playwright`
- **Android SDK** (full image only): `/opt/android-sdk`
- **Android emulator AVD** (full image only): `test-device` (Pixel 6, API 34, Google APIs x86_64)
- **Default user**: `vscode` (the standard devcontainers user)
- **Runtime config** (env vars, git, aliases) is handled by [PlasmaPOS/dotfiles](https://github.com/PlasmaPOS/dotfiles), not this image
