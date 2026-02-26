# NanoClaw

Personal WhatsApp assistant running on Linux (WSL2). See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process that connects to WhatsApp, routes messages to Claude Agent SDK running in Docker containers. Each group has an isolated filesystem and memory.

## Platform

- **OS**: Linux (WSL2)
- **Container runtime**: Docker (converted from Apple Container during setup)
- **Service manager**: systemd user service
- **Node.js**: v24 via nvm (host), Node 24 base image (containers)

## AI Provider

This instance uses **TensorFoundry** instead of Anthropic directly. Configuration is in `.env`:

- **Base URL**: `https://api.tensorfoundry.io/anthropic`
- **Model**: `tf/forge-code-2.0-code` (GLM5 + MiniMax v2.5 + KimiK2 synthetic)
- **Auth**: `ANTHROPIC_AUTH_TOKEN` (not the standard `ANTHROPIC_API_KEY`)

All model env vars (`ANTHROPIC_MODEL`, `ANTHROPIC_SMALL_FAST_MODEL`, `ANTHROPIC_DEFAULT_*_MODEL`) are set to the same model. These are read from `.env` at runtime and passed to containers via stdin — never mounted as files.

## Container Tooling

The Docker container image (`nanoclaw-agent:latest`) includes:

| Tool | Version | Auth Method |
|------|---------|-------------|
| Node.js | 24.x | Built into base image |
| .NET SDK | 10.x | No auth needed |
| `gh` (GitHub CLI) | 2.86+ | `GH_TOKEN` env var from `.env` |
| `jules` (Google AI coding) | 0.1.42+ | Host `~/.jules/` mounted read-only |
| `codex` (OpenAI Codex CLI) | 0.105+ | `AZURE_OPENAI_API_KEY_SIXPIVOT` env var; config from `container/codex-config.toml` |
| `agent-browser` (Chromium) | latest | No auth needed |
| Claude Code SDK | latest | Via `ANTHROPIC_AUTH_TOKEN` |

### Secret Handling

Secrets from `.env` are passed to containers via stdin JSON (never written to disk inside containers). The agent-runner strips `ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`, and `ANTHROPIC_AUTH_TOKEN` from Bash subprocess environments so agent tools can't leak them. `GH_TOKEN` is intentionally NOT stripped since `gh` needs it in the environment.

Jules credentials are mounted from the host's `~/.jules/` directory (read-only) since the Jules CLI uses OAuth, not API keys.

## Registered Groups

| Group | Folder | Trigger Required |
|-------|--------|-----------------|
| Newtbot (main) | `main` | No |
| Newtbot - RLFP | `rlfp` | No |
| Newtbot - WhichwAI | `whichwai` | No |
| Newtbot - MoTuMu | `motumu` | No |

- **Assistant name**: Newtbot
- **Trigger word**: `@Newtbot` (only needed in groups with `requires_trigger = 1`)
- All current groups are solo groups with no trigger required

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/whatsapp.ts` | WhatsApp connection, auth, send/receive |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns Docker containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/Dockerfile` | Agent container image definition |
| `container/agent-runner/src/index.ts` | Code running inside each container |
| `.env` | Secrets and config (gitignored) |

## Skills

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |

## Development

Run commands directly — don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
./container/build.sh # Rebuild agent container (Docker)
```

Service management:
```bash
systemctl --user status nanoclaw      # Check status
systemctl --user restart nanoclaw     # Restart
systemctl --user stop nanoclaw        # Stop
journalctl --user -u nanoclaw -f      # Follow logs
tail -f logs/nanoclaw.log             # Application logs
```

## Container Rebuild

After changing `container/Dockerfile` or `container/agent-runner/`:

```bash
./container/build.sh
```

To force a clean rebuild (cache issues):
```bash
docker builder prune -f
./container/build.sh
```

Verify after rebuild:
```bash
docker run --rm --entrypoint node nanoclaw-agent:latest --version
docker run --rm --entrypoint dotnet nanoclaw-agent:latest --version
docker run --rm --entrypoint gh nanoclaw-agent:latest --version
docker run --rm --entrypoint jules nanoclaw-agent:latest version
```

## Re-authenticating WhatsApp

If the bot stops responding and logs show repeated `405` / `Connection Failure` errors, the WhatsApp session has been invalidated and needs a fresh QR scan:

```bash
systemctl --user stop nanoclaw
cd ~/nanoclaw && rm -rf store/auth store/auth-status.txt store/qr-data.txt && npx tsx src/whatsapp-auth.ts
# Scan the QR code in WhatsApp → Settings → Linked Devices → Link a Device
systemctl --user start nanoclaw
```

The `405` error means WhatsApp's server rejected the noise-protocol handshake — usually caused by the WA Web version drifting (fixed in code by using `fetchLatestBaileysVersion()`) or the session being revoked on the phone.

## Adding a New Group

1. Create the WhatsApp group and send a message in it
2. Trigger a group refresh or wait for the service to see it
3. Register via Node.js (sqlite3 CLI not available):
   ```bash
   node -e "
   const db = require('better-sqlite3')('store/messages.db');
   db.prepare('INSERT OR REPLACE INTO registered_groups (jid, name, folder, trigger_pattern, added_at, container_config, requires_trigger) VALUES (?, ?, ?, ?, ?, ?, ?)').run(
     'JID_HERE', 'name', 'folder-name', '@Newtbot', new Date().toISOString(), null, 0
   );"
   ```
4. Create the group directory: `mkdir -p groups/folder-name/logs`
5. Restart: `systemctl --user restart nanoclaw`

## Notes

- `sqlite3` CLI is not installed on this system — use `node -e "..."` with `better-sqlite3` for DB queries
- No sudo access — system packages can't be installed; add tools to the Dockerfile instead
- Docker-in-Docker is not supported — agents cannot run Docker inside their containers
