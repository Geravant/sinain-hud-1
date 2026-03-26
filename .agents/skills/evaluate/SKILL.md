---
name: evaluate
description: Run Sinain evaluation pipeline server-side and display the report
---

# Evaluate Sinain Pipeline

Run the sinain-memory evaluation pipeline on the OpenClaw server and display the results.

## Arguments

- `$ARGUMENTS` — optional: number of days to evaluate (default: 1). E.g. `/evaluate 3` for 3-day report.

Parse the days argument:
```
DAYS = parse "$ARGUMENTS" as integer, default 1 if empty or non-numeric
```

## Step 1: Run tick evaluator (Tier 1)

Evaluates any un-evaluated ticks from today's playbook-logs. Uses `sampled` level by default (30% LLM judges).

```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 '
  cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml exec -T openclaw-gateway bash -c "
    cd /home/node/.openclaw/workspace &&
    uv run --with requests python3 sinain-memory/tick_evaluator.py --memory-dir memory/ 2>&1
  "
'
```

## Step 2: Run eval reporter (Tier 2)

Aggregates eval-logs over DAYS days, computes quality metrics, detects regressions, and generates a markdown report via LLM analysis.

```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 "
  cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml exec -T openclaw-gateway bash -c '
    cd /home/node/.openclaw/workspace &&
    uv run --with requests python3 sinain-memory/eval_reporter.py --memory-dir memory/ --days $DAYS 2>&1
  '
"
```

If this fails with PermissionError, fix permissions and retry:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 '
  cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml exec -u root -T openclaw-gateway bash -c "
    chmod -R 777 /home/node/.openclaw/workspace/memory/eval-reports/ /home/node/.openclaw/workspace/memory/eval-logs/
  "
'
```

## Step 3: Fetch and display the report

Read today's generated report:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 '
  cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml exec -T openclaw-gateway \
    cat /home/node/.openclaw/workspace/memory/eval-reports/$(date -u +%Y-%m-%d).md
'
```

## Step 4: Fetch recent eval-log entries

Show last 5 tick-level evaluation entries for granularity:
```bash
ssh -i ~/.ssh/id_ed25519_strato root@85.214.180.247 '
  cd /opt/openclaw && docker compose -f docker-compose.openclaw.yml exec -T openclaw-gateway bash -c "
    tail -5 /home/node/.openclaw/workspace/memory/eval-logs/$(date -u +%Y-%m-%d).jsonl 2>/dev/null || echo \"No eval-logs for today\"
  "
'
```

## Step 5: Display summary

Present the results to the user:
1. Show the full markdown report from Step 3
2. Show the last 5 tick-level results with passRate and judgeAvg (parsed from Step 4 JSONL)
3. Highlight any regressions or quality gate failures (schema validity < 85%, assertion pass rate < 85%, skip rate > 80%)
4. If the schema validity is low, note the most common schema failure reasons from the eval-log entries

## Key details

| Detail | Value |
|--------|-------|
| SSH key | `~/.ssh/id_ed25519_strato` |
| Server | `root@85.214.180.247` |
| Compose file | `docker-compose.openclaw.yml` |
| Container workspace | `/home/node/.openclaw/workspace/` |
| Playbook logs | `memory/playbook-logs/YYYY-MM-DD.jsonl` |
| Eval logs | `memory/eval-logs/YYYY-MM-DD.jsonl` |
| Eval reports | `memory/eval-reports/YYYY-MM-DD.md` |
| Eval config | `sinain-memory/memory-config.json` → `eval` section |
| Runtime override | `memory/eval-config.json` |

## Eval levels

| Level | Description |
|-------|-------------|
| `mechanical` | Schema validation + behavioral assertions only (no LLM cost) |
| `sampled` | mechanical + LLM judges on 30% of ticks (default) |
| `full` | mechanical + LLM judges on every tick |

## Quality gates

| Gate | Threshold | Status symbol |
|------|-----------|---------------|
| Schema validity | >= 85% | Pass/Fail |
| Assertion pass rate | >= 85% | Pass/Fail |
| Mean judge score | >= 3.0/4.0 | Info |
| Skip rate | <= 80% | Warning |
