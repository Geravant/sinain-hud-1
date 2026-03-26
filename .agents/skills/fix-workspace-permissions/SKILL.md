---
name: fix-workspace-permissions
description: Fix file ownership on the OpenClaw server workspace after SCP deploys break the heartbeat pipeline
---

# Fix Workspace Permissions

Use this when the sinain heartbeat pipeline stops writing data (no new playbook-logs, heartbeat script failures, eval gaps) after a deploy.

## Background

The OpenClaw container runs as `node` (uid=1000, same as `sinain-tunnel` on the host). SCP always runs as `root`, so any file uploaded via SCP lands as `root:root`. The container cannot write to root-owned files, which causes silent failures:

- `EACCES: permission denied` on `playbook-logs/YYYY-MM-DD.jsonl`
- `heartbeat script failed: sinain-memory/signal_analyzer.py (code 2)`
- `failed to sync HEARTBEAT.md: EACCES: permission denied`

**The deceptive symptom**: `SITUATION.md` keeps updating (written via RPC inside the container as uid=1000), so the connection looks healthy while everything else is broken.

This caused a 12-day gap in evaluation data in March 2026 (Mar 1 → Mar 13).

## Diagnosis

Check if files are root-owned:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'ls -la /mnt/openclaw-state/workspace/ | head -10 && ls -la /mnt/openclaw-state/sinain-sources/'
```

Check gateway logs for the EACCES pattern:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml logs --tail=100 openclaw-gateway 2>&1 | grep -i "EACCES\|permission denied\|heartbeat script failed"'
```

## Fix

One command repairs both directories:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'chown -R 1000:1000 /mnt/openclaw-state/sinain-sources/ /mnt/openclaw-state/workspace/'
```

No gateway restart needed — the next heartbeat tick will succeed.

## Verify

Run the tick evaluator manually to confirm it can write:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 '
  cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml exec -T openclaw-gateway bash -c "
    cd /home/node/.openclaw/workspace &&
    uv run --with requests python3 sinain-memory/tick_evaluator.py --memory-dir memory/ 2>&1
  "
'
```

Expected output:
```
[tick-eval] level=sampled sampleRate=0.3
[tick-eval] N unevaluated ticks found
[tick-eval] PASS tick=... passRate=1.0
[tick-eval] wrote N eval entries to memory/eval-logs/YYYY-MM-DD.jsonl
```

Check that a new playbook-log file was written today:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 \
  'ls -la /mnt/openclaw-state/workspace/memory/playbook-logs/ | tail -3'
```

## Prevention

**After every SCP deploy**, always run the chown fix. It is part of the `deploy-heartbeat` skill as Step 2b.

The container user (uid=1000) maps to `sinain-tunnel` on the host. Any file written from outside the container (SCP, host-side cp, etc.) must be re-owned to uid=1000 before the container can modify it.

## Key details

| Detail | Value |
|--------|-------|
| SSH key | `~/.ssh/id_ed25519_strato` |
| Server | `root@85.214.180.247` |
| Container user | `node` (uid=1000) |
| Host user | `sinain-tunnel` (uid=1000) |
| Workspace on host | `/mnt/openclaw-state/workspace/` |
| Sources on host | `/mnt/openclaw-state/sinain-sources/` |
| Compose file | `docker-compose.openclaw.yml` |
