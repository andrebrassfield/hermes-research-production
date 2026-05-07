# M4-Fleet — Hermes + Goose Agent Harness Infrastructure

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    M4 MacBook Air (Dre's Mac)                      │
│                                                                      │
│   ┌─────────────┐     Brain-Body Protocol      ┌───────────────┐   │
│   │ Hermes      │◄──── goose-safe-exec ────────│ Goose 1.33.1  │   │
│   │ Research    │     (read-only, stateless)    │ (Executor)    │   │
│   │ Gateway     │                              │               │   │
│   │ PID 66423   │                              │ ~/.local/bin/ │   │
│   └──────┬──────┘                              └─────────┬─────┘   │
│          │                                               │          │
│   ┌──────▼──────┐     Vault Sync           ┌────────────▼──────┐   │
│   │ Brain API   │◄─────────────────────────►│ Obsidian Vault    │   │
│   │ port 3456   │                          │ ~/.dre/brain/     │   │
│   │ LaunchAgent │                          │                   │   │
│   └─────────────┘                          └───────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## GitHub Topology

```
NousResearch/hermes-agent (UPSTREAM — read-only)
    └── Public base; read with SSH key gary-dres-openclaw

andrebrassfield/hermes-research-dev (DEV — read-write)
    └── SSH key: andre@brassfield.com
    └── Push: git@github.com-andre:andrebrassfield/hermes-research-dev.git

andrebrassfield/hermes-research-production (PROD — read-write)
    └── SSH key: andre@brassfield.com
    └── Push: git@github.com-andre:andrebrassfield/hermes-research-production.git
```

## Local Branches

| Branch | Tracks | Purpose |
|--------|--------|---------|
| `fix/defensive-model-config-hardling-v2` | upstream/main + local patches | Active development |
| `main` | dev/main | Dev integration branch |

## Infrastructure Files

| Path | Purpose |
|------|---------|
| `~/.hermes/sync/sync-from-upstream.sh` | Daily upstream sync (05:00 UTC) |
| `~/.hermes/bin/goose-safe-exec` | Stateless read-only Goose executor |
| `~/.hermes/bin/watchdog.sh` | Alert-only gateway monitor (no auto-restart) |
| `~/.hermes/bin/restart-research.sh` | Manual gateway restart |
| `~/.hermes/skills/goose-handshake/` | Hermes↔Goose brain-body protocol |
| `~/.hermes/skills/m4-system-monitor/` | macOS health monitoring skill |

## Root Cause: Hermes Self-Destruct (Session 4)

**What happened:** Snapshot script (`hermes-snap-*.sh`) contained `eval 'hermes gateway restart'` which triggered a restart loop. When gateway finally exited gracefully, `launchd KeepAlive: SuccessfulExit: false` prevented auto-restart.

**Permanent fix applied:**
- `checkpoints: {enabled: false}` in both configs
- `agent.restart_drain_timeout: 0` in both configs
- `agent.gateway_auto_continue_freshness: 0` in both configs
- Snapshot scripts removed

## Active Commits

### Patch 1: Model Config Defensive Hardening (0fca31a45)
Files: `runtime_provider.py`, `status.py`, `doctor.py`, `fallback_cmd.py`, `main.py`

**Problem:** Legacy `model.default` was a `dict` (`{provider: nous, model: stepfun/step-3.5-flash}`) instead of a string. Calling `.strip()` on a dict caused `AttributeError`.

**Fix:** `normalize_model_default()` helper handles both dict and string inputs.

### Patch 2: M4-Fleet Infrastructure README (40b385e23)
This document.

## Safety Configuration

```yaml
# ~/.hermes/config.yaml (main)
approvals:
  mode: manual
checkpoints:
  enabled: false
agent:
  restart_drain_timeout: 0
  gateway_auto_continue_freshness: 0

# ~/.hermes/profiles/research/config.yaml (research)
[Same settings]
```

## Cron Jobs (Hermes Research Profile)

| Job | Schedule | Purpose |
|-----|----------|---------|
| Hermes Health Monitor | `*/15 * * * *` | Gateway PID, ports, status |
| Resource Monitor | `*/5 * * * *` | Disk, RAM, CPU |
| Upstream Sync | `0 5 * * *` | Pull NousResearch/main → push dev/prod |

## Brain API

- **URL:** `http://localhost:3456`
- **Vault:** `~/.dre/brain/`
- **LaunchAgent:** `com.dre.brain-api` (PID managed, auto-restart)
- **Health:** `curl http://localhost:3456/api/v1/health`
- **Write to vault:** Edit files directly in `~/.dre/brain/`

## SSH Keys

| Key | Account | Used For |
|-----|---------|---------|
| `~/.ssh/id_hermes_push` | gary-dres-openclaw | NousResearch read-only |
| `~/.ssh/id_hermes_andre` | andre@brassfield.com | dev/prod push |
| (default agent key) | gary-dres-openclaw | NousResearch read |

## Sync Workflow

```bash
# Manual sync
bash ~/.hermes/sync/sync-from-upstream.sh

# Auto (daily 05:00 UTC via Hermes cron)
# Pulls upstream/main → rebases fix branch → pushes to dev → opens PR → pushes to prod
```

## After Upstream Update

1. Run: `bash ~/.hermes/sync/sync-from-upstream.sh`
2. Review PR on dev repo
3. Test on dev before prod
4. Prod is the stable, tested target
