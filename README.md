# PlasmaPOS DevContainer Images

Pre-built Docker images for DevPod cloud workspaces. Two variants:

## Images

### `ghcr.io/plasmapos/devcontainers/light:latest`
For backend/tooling projects (dev-agent, seal, aqua):
- Debian base + Node 22 (system-wide)
- Bun (latest)
- Convex CLI
- Claude Code CLI + Codex CLI
- Git, GH CLI (via devcontainer features)

### `ghcr.io/plasmapos/devcontainers/full:latest`
For app projects with mobile + web testing (plasma, pile, roster, veil):
- Everything in light, plus:
- Playwright + Chromium
- Android SDK 34 + emulator (Pixel 6 AVD)
- agent-device CLI

## VM Sizing

| Image | Projects | Machine Type | vCPU | RAM |
|-------|----------|-------------|------|-----|
| light | dev-agent, seal, aqua | n2d-standard-4 | 4 | 16GB |
| full | plasma, pile, roster, veil | n2d-standard-8 | 8 | 32GB |

Full-image projects need 8 vCPU because running emulator + dev server + AI CLIs concurrently on 4 vCPU causes OOM.

Override per-workspace:
```bash
devpod up <repo> --provider-option MACHINE_TYPE=n2d-standard-8
```

## Usage in devcontainer.json

```json
{
  "image": "ghcr.io/plasmapos/devcontainers/light:latest",
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },
  "postCreateCommand": "bun install"
}
```

## Building

Images are auto-built on push to main via GitHub Actions.
Manual trigger: `gh workflow run build.yml`
