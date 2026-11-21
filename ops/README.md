# Antigravity Pooling V1.2 — Operations Guide

Repository: `~/work/antigravity-pooling/antigravity-swap` (branch `v1.2-quota-pools`)  
Wrapper: `~/agy-pool/bin/agy-pool`  
Data Directory: `~/agy-pool` (`ASWAP_HOME`)  
Runtime Interpreter: `~/antigravity-pooling-audit/venv/bin/python`

---

## 1. Six Daily Invocations

### (1) Unattended / Scripted Claude/GPT Execution with Bounded Failover
```bash
~/agy-pool/bin/agy-pool exec -- -p "your prompt here" --model claude-sonnet-4-6 --max-hops 2
```
*Purpose*: Runs print-mode tasks against third-party models. If an account runs out of Claude/GPT quota, it automatically carries conversation state and continues on the next eligible account (up to 2 hops).

### (2) Interactive Supervised Session
```bash
~/agy-pool/bin/agy-pool run
```
*Purpose*: Launches interactive TUI. If the active account hits quota during a turn, it cleanly restores terminal mode and fails over to an available account while preserving conversation context.

### (3) List Accounts and Quota Pools
```bash
~/agy-pool/bin/agy-pool list
```
*Purpose*: Inspects pooled accounts, slot numbers, active aliases, account-wide holds, per-pool holds (`Claude/GPT out until ...`), and quarantine statuses.

### (4) Clear Account Hold or Quarantine
```bash
~/agy-pool/bin/agy-pool limit <slot|alias> --clear
```
*Purpose*: Wipes manual holds, per-pool holds, and quarantine flags on an account when quotas have reset or login issues are resolved.

### (5) Manual Account Hold (e.g., for Maintenance)
```bash
~/agy-pool/bin/agy-pool limit <slot|alias> 4h
```
*Purpose*: Places a temporary account-wide hold on a slot (e.g. for 4 hours), safely excluding it from all pools without deleting credentials.

### (6) Proactive Token & Quota Refresh
```bash
~/agy-pool/bin/agy-pool refresh
```
*Purpose*: Refreshes OAuth access tokens and updates cached quota usage from Google API for all managed accounts.

---

## 2. Decision Table: Move vs. Refuse

| Scenario | Pool | Condition | Action | Outcome |
|---|---|---|---|---|
| Claude Quota Exhausted | `3p` | Account A hits quota; Account B has Claude quota | **Move** | Rotates to B; carries conversation; sets A's `pool_limits["3p"]`. |
| Claude Out on All Accounts | `3p` | All accounts hit Claude quota or are cached at 100% | **Refuse** | Exits with code 3; lists per-account reset times; no quota wasted. |
| Gemini Work on Claude-Held Account | `gemini` | Account A held for `3p`; Gemini requested | **Stay / Serve** | Account A serves Gemini request directly; no rotation needed. |
| Generic 403 Forbidden | Any | HTTP 403 (project/permission error) | **Refuse / Abort** | Exits non-zero; **no swap, no quarantine**; store file byte-identical. |
| Verification Required 403 | Any | "Verify your account to continue" | **Quarantine** | Quarantines account with reason; rotates to next eligible account. |
| Auth Expired (401) | Any | Token rejected | **Retry In-Place** | Renews token via official `agy models` path once; aborts if still 401. |
| Transient 5xx / Connection Error | Any | Upstream 500/502/503 or network drop | **Retry In-Place** | Bounded jittered exponential backoff (max 3 attempts); no rotation. |
| Unknown Model | None | Model not in recognized pool patterns | **Refuse / Account-Wide Hold** | Does not guess pool; places account-wide hold if quota hits. |
| Exceeded Max Hops | Any | Hops counter > `--max-hops` | **Refuse** | Exits code 3; stops rotation cascade to prevent quota cycling. |

---

## 3. Structural Limitations (Consumer Accounts)

1. **Quota RPC Scope (`aicode-consumers`)**:
   Google's `retrieveUserQuotaSummary` endpoint returns **only the Claude and GPT group** (`3p-weekly` / `3p-5h`) for standard consumer accounts. The Gemini model quota buckets are not exposed via this API surface.
2. **Reactive Gemini Failover**:
   Because Gemini quota readings remain unknown (`None`), Gemini usage cannot be evaluated proactively. Gemini failover is strictly **reactive** (triggered upon receiving an actual 429 quota exhaustion error during a run). Conversely, Claude/GPT failover is **proactive** (skips 100% exhausted accounts prior to launch).
3. **Running TUI Session Confinement**:
   Independent, already-running agy TUI processes (holding tokens in process memory) will not dynamically switch mid-stream unless launched via `agy-pool run`.
4. **Service Daemon Status**:
   `agy-pool-failover.service` is provided as a sketch only. Enabling systemd or cron services requires human administrative authorization.
