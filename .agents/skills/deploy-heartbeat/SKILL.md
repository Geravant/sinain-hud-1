---
name: deploy-heartbeat
description: Deploy HEARTBEAT.md and skill files to the OpenClaw strato server
---

# Deploy Heartbeat

Deploy HEARTBEAT.md, plugin code, and optionally SKILL.md to the OpenClaw server on strato.

> **Note:** The sinain-hud plugin auto-deploys HEARTBEAT.md and SKILL.md from `/mnt/openclaw-state/sinain-sources/` to the agent workspace on every agent start. SCP updates to sinain-sources take effect on the next agent start without a gateway restart. A restart is only needed when plugin code (`index.ts`) or gateway config (`openclaw.json`) changes.

## Source of truth

| File | Source of truth | Also kept in sync |
|------|----------------|-------------------|
| HEARTBEAT.md | `openclaw/skills/sinain-hud/HEARTBEAT.md` | `sinain-hud/sinain-hud-plugin/HEARTBEAT.md` |
| SKILL.md | `openclaw/skills/sinain-hud/SKILL.md` | — |
| Plugin code | `sinain-hud/sinain-hud-plugin/` | — |
| Heartbeat prompt | `openclaw/skills/sinain-hud/openclaw-config-patch.json` | Server `openclaw.json` |

## Step 1: Sync repos

Copy HEARTBEAT.md from openclaw (source of truth) to sinain-hud so both repos stay aligned:
```bash
cp /Users/Igor.Gerasimov/IdeaProjects/openclaw/skills/sinain-hud/HEARTBEAT.md \
   /Users/Igor.Gerasimov/IdeaProjects/sinain-hud/sinain-hud-plugin/HEARTBEAT.md
```

## Step 2: SCP skill files to sinain-sources

Upload skill files to the persistent source directory (the plugin reads from here on each agent start):
```bash
scp -i ~/.ssh/id_ed25519_strato \
  sinain-hud-plugin/HEARTBEAT.md \
  root@85.214.180.247:/mnt/openclaw-state/sinain-sources/HEARTBEAT.md
```
If SKILL.md also changed:
```bash
scp -i ~/.ssh/id_ed25519_strato \
  /Users/Igor.Gerasimov/IdeaProjects/openclaw/skills/sinain-hud/SKILL.md \
  root@85.214.180.247:/mnt/openclaw-state/sinain-sources/SKILL.md
```

> Skill file updates take effect on the next agent start — no restart needed.

## Step 2b: Fix file ownership after SCP

**Always run this after any SCP deploy.** SCP runs as root and creates `root:root` files that the container's `node` user (uid=1000) cannot write:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'chown -R 1000:1000 /mnt/openclaw-state/sinain-sources/ /mnt/openclaw-state/workspace/'
```

## Step 3: SCP plugin files (if changed)

If `sinain-hud-plugin/index.ts` or `openclaw.plugin.json` changed, upload them:
```bash
scp -i ~/.ssh/id_ed25519_strato \
  sinain-hud-plugin/index.ts sinain-hud-plugin/openclaw.plugin.json \
  root@85.214.180.247:/mnt/openclaw-state/extensions/sinain-hud/
```

> Plugin code changes require a gateway restart (Step 5).

## Step 4: Update openclaw.json on server (if config changed)

If the heartbeat prompt or other config in `openclaw-config-patch.json` changed, update the server config. Read the current config, apply changes, verify:
```bash
# Read current config
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'cat /mnt/openclaw-state/openclaw.json'

# Edit with sed (for simple string replacements)
# NOTE: JSON stores em-dashes as \u2014 — match that literal escape, not the UTF-8 character
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'sed -i "s|old prompt text|new prompt text|" /mnt/openclaw-state/openclaw.json'

# Verify the change
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'grep "heartbeat" /mnt/openclaw-state/openclaw.json'
```

> Config changes require a gateway restart (Step 5).

## Step 5: Restart gateway (only if plugin code or config changed)

**IMPORTANT:** Use `docker-compose.openclaw.yml` — not the default `docker-compose.yml`.
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml restart'
```

## Step 6: Verify deployment

```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 '
  echo "=== HEARTBEAT.md ===" &&
  head -5 /mnt/openclaw-state/sinain-sources/HEARTBEAT.md &&
  echo "" &&
  echo "=== Plugin files ===" &&
  ls -la /mnt/openclaw-state/extensions/sinain-hud/ &&
  echo "" &&
  echo "=== heartbeat_tick in plugin ===" &&
  grep heartbeat_tick /mnt/openclaw-state/extensions/sinain-hud/index.ts | head -3 &&
  echo "" &&
  echo "=== heartbeat prompt in config ===" &&
  grep heartbeat /mnt/openclaw-state/openclaw.json
'
```

Check gateway logs for successful plugin registration:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml logs --tail=30 openclaw-gateway 2>&1 | grep -i "sinain\|plugin\|heartbeat\|error"'
```

Expected output:
```
sinain-hud: plugin registered
[heartbeat] started
sinain-hud: service started (heartbeat: /home/node/.openclaw/sinain-sources/HEARTBEAT.md)
```

## Step 7: Commit

Commit in sinain-hud repo (this repo). If the openclaw fork was also updated, commit there too.

## Key details

| Detail | Value |
|--------|-------|
| SSH key | `~/.ssh/id_ed25519_strato` |
| Server | `root@85.214.180.247` |
| Compose file | `/opt/openclaw/docker-compose.openclaw.yml` |
| Source files | `/mnt/openclaw-state/sinain-sources/` |
| Plugin files | `/mnt/openclaw-state/extensions/sinain-hud/` |
| Gateway config | `/mnt/openclaw-state/openclaw.json` |
| Container workspace | `/home/node/.openclaw/workspace/` |
| `uv` / `python3` | Inside container only — not on host |

## When to restart vs. not

| What changed | Restart needed? |
|---|---|
| HEARTBEAT.md or SKILL.md only | No — plugin syncs on next agent start |
| sinain-memory/ scripts | No — plugin syncs on next agent start |
| Plugin code (`index.ts`) | Yes |
| Plugin manifest (`openclaw.plugin.json`) | Yes |
| Gateway config (`openclaw.json`) | Yes |

## Gotchas

- **Compose file**: Always use `-f docker-compose.openclaw.yml`. The default `docker-compose.yml` uses env vars that aren't set on the host and will fail.
- **JSON Unicode**: `openclaw.json` stores em-dashes as `\u2014`. When using `sed`, match the literal `\u2014` escape sequence, not the UTF-8 em-dash character.
- **No python3 on host**: `uv` and `python3` are inside the Docker container only. For host-side JSON edits, use `sed` or SCP the file locally, edit, SCP back.
- **File ownership — CRITICAL**: SCP always runs as `root` on the strato server, so uploaded files land as `root:root`. The container's `node` user is `uid=1000` (`sinain-tunnel` on the host) and **cannot write root-owned files**. This caused the entire pipeline to silently break in March 2026 — heartbeat ticks ran but failed to write playbook-logs for 12 days. **After every SCP deploy, always run:**
  ```bash
  ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
    'chown -R 1000:1000 /mnt/openclaw-state/sinain-sources/ /mnt/openclaw-state/workspace/'
  ```
  The `SITUATION.md` file may appear healthy (it's written via RPC, which runs as uid=1000 inside the container) even when all other files are broken by root ownership.
