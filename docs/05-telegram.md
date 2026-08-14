# Telegram Integration

This document covers how Telegram was added to Hermes/Gilfoyle, what it is currently used for, and what the live state on `[HERMES_HOST]` says today.

Like the rest of this repo, this is evidence-first. If something is only supported by session history or vault notes rather than a fresh live replay, it is labeled that way.

---

## 1. Current live state

### Gateway service
Verified live on `[HERMES_HOST]`:
- systemd unit: `hermes-gateway.service`
- current state during this draft: `active (running)`
- executable path shown by systemd:
  - `~/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run`

### Telegram-related env keys currently present
Verified by reading key names only from `~/.hermes/.env`:
- `TELEGRAM_ALLOWED_USERS`
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_HOME_CHANNEL`

The values were deliberately not printed in this doc pass, because unlike some people, I do not confuse documentation with credential disclosure.

### Current gateway behavior in logs
Recent `journalctl --user -u hermes-gateway` lines show the Telegram adapter is active and attempting network recovery. Verified examples:
- `[Telegram] Discovering Telegram API fallback IPs via DNS-over-HTTPS…`
- `[Telegram] Connecting to Telegram (attempt 1/8)…`
- `Telegram network error ... Bad Gateway`
- `Telegram polling reconnect failed: Timed out`
- `telegram.error.TimedOut: Timed out`

So Telegram is not hypothetical. The adapter is running right now and hitting real network conditions.

### Pairing state
Current `hermes pairing list` output:
- `No pairing data found. No one has tried to pair yet~`

Safe interpretation:
- there is **no currently stored pairing data** visible via the Hermes pairing command
- that does **not** mean Telegram is unused
- it does mean I cannot claim a currently proven paired conversational Telegram session from live state alone

### Telegram tool surface in current Hermes config
Verified from `~/.hermes/config.yaml` under `platform_toolsets`:
- the `telegram` platform currently includes a broad tool surface, including:
  - `browser`
  - `clarify`
  - `code_execution`
  - `computer_use`
  - `cronjob`
  - `delegation`
  - `file`
  - `homeassistant`
  - `image_gen`
  - `kanban`
  - `memory`
  - `messaging`
  - `session_search`
  - `skills`
  - `spotify`
  - `terminal`
  - `todo`
  - `vision`
  - `web`

That confirms Telegram is configured as a first-class Hermes surface, not merely a single-purpose notifier.

---

## 2. How Telegram was added

### Early gateway state: installed but idle
Session history shows that on **2026-06-03**, the Hermes gateway service was installed and running, but no messaging platform credentials had been configured yet.

Session found:
- `@session:default/20260603_042359_c0e90e`

Grounded conclusion from that session:
- `hermes-gateway.service` was active
- platform blocks existed in config
- no tokens were set yet
- gateway log warned:
  - `No messaging platforms enabled.`

That is the clean pre-Telegram baseline.

### Telegram setup path documented on June 5
Session history shows a dedicated Telegram setup session on **2026-06-05**.

Session found:
- `@session:default/20260605_153112_970abf`

That session explicitly documented:
- running `hermes gateway setup`
- obtaining a Telegram bot token via `@BotFather`
- DMing the bot for pairing/interaction
- restarting the gateway after setup

This is the clearest session-backed evidence for how Telegram was initially enabled.

### Current env confirms setup persisted
Whatever exact wizard steps were taken, the current `.env` still contains Telegram keys:
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_HOME_CHANNEL`
- `TELEGRAM_ALLOWED_USERS`

So Telegram was not just discussed. It was actually wired into the running host config and stayed there.

---

## 3. What Telegram is used for

### A. Infrastructure alert delivery
This is the strongest current evidence.

From the live cron job list, Telegram is the delivery target for multiple watchdogs and reports, including:
- `Disk Space Alert (Proxmox >85%)`
- `VM Health Check (Proxmox)`
- `PBS Backup Verification`
- `AdGuard Stats`
- `Docker Watchdog (Portainer)`
- `Uptime Kuma Digest`
- `URBackup Report`
- `Docker Compose Vault Sync`

Additional paused/older Telegram-delivered jobs exist too, such as:
- `MRDTech Morning Briefing`
- `Wazuh Alert Digest`
- `CrowdSec Digest`
- `UniFi Threat Digest`
- `Cert Expiry Alert`
- `Weekly Infrastructure Report`

Safe conclusion:
- Telegram is currently a **primary outbound alert/reporting channel** for Hermes cron automation.

### B. Home/owner notification channel
The presence of `TELEGRAM_HOME_CHANNEL` and multiple cron deliveries to `telegram` indicates Telegram is being used as the primary home delivery path for owner-facing notifications.

This is stronger than generic “Telegram support exists.” It is wired into the actual alerting plane.

### C. Wazuh alerting and heartbeat history
Session history from June 2026 shows Telegram was also used in the Wazuh alerting path.

Relevant sessions include:
- `@session:default/bg_215715_e9592b`
- `@session:default/20260606_230036_8c6106`
- `@session:default/bg_003737_27eb0a`

Those sessions document:
- direct Telegram integration for Wazuh manager alerts
- a real `api.telegram.org` delivery confirmation (`ok: true`, message id returned)
- a weekly Telegram heartbeat script created to detect DNS/filtering regressions
- AdGuard-side issues previously affecting Telegram API reachability

That is historical evidence, not freshly replayed in this documentation pass, but it is concrete retained session evidence.

### D. Telegram as a conversation surface
Current live logs prove the Telegram platform adapter is active and polling.

However:
- live `pairing list` shows no stored pairing data
- I did not fresh-test an inbound Telegram conversation during this pass

So the safe statement is:
- **Telegram is definitely active as an outbound/adapter surface**
- **inbound conversational pairing is not fully re-proven in this pass**

---

## 4. Voice-note / voice-message history

The user specifically asked to note voice-note incident history **if relevant**.

### What is supported by evidence
- Session history includes Telegram-source sessions.
- Session search results include references to `voice message` queries and Telegram interactions across June/August sessions.
- The current gateway toolset for Telegram includes rich capabilities, but I did **not** find fresh live proof in this pass that Telegram voice notes are a currently active workflow.

### What is not fully proven here
I did **not** replay or verify:
- inbound Telegram voice-note transcription end-to-end
- a dedicated Telegram-only read-aloud or TTS mechanism
- whether recent voice-note incidents were Telegram-specific versus desktop/live-voice-surface behavior

So the safe documentation stance is:
- **Telegram voice-note handling may exist in history/context, but it was not freshly proven in this draft pass**.

That distinction matters. Different surface, different failure mode, different blame target.

---

## 5. Network and reliability notes

### Current live reliability issue
Current logs show repeated Telegram adapter network recoveries:
- `Bad Gateway`
- `Timed out`
- reconnect attempts and bootstrap retry failures

This does **not** prove Telegram is down in a durable sense, but it does prove recent instability in the polling path.

### Historical DNS failure path
Session history documents a prior failure mode where:
- `api.telegram.org` was being sinkholed / blocked by DNS filtering
- Telegram alerting silently degraded until allowlist/remediation was applied
- a weekly heartbeat was then added as cheap insurance

That is exactly the sort of operational scar tissue that belongs in this doc.

---

## 6. Files, commands, and surfaces of record

### Host files
- live env keys stored in:
  - `~/.hermes/.env`
- Telegram-capable toolsets configured in:
  - `~/.hermes/config.yaml`
- gateway service unit:
  - `~/.config/systemd/user/hermes-gateway.service`

### Live commands used in this draft
- `hermes gateway status`
- `hermes pairing list`
- `journalctl --user -u hermes-gateway -n 120 --no-pager`
- `cronjob list`

### Historical sessions used as evidence
- `@session:default/20260603_042359_c0e90e`
- `@session:default/20260605_153112_970abf`
- `@session:default/20260605_205517_23b5df`
- `@session:default/bg_215715_e9592b`
- `@session:default/20260606_230036_8c6106`
- `@session:default/bg_003737_27eb0a`

---

## 7. Open issues and review notes

### Fully grounded current-state findings
- Telegram env keys exist today
- Telegram adapter is active in gateway logs today
- Telegram delivery is used by many live cron jobs today
- no pairing data is currently visible via `hermes pairing list`
- current logs show adapter reconnect / timeout behavior
- Telegram has a full `platform_toolsets.telegram` entry in current config

### Best-effort / needs review
- exact original day/time when Telegram credentials were first entered into the wizard
- whether the current active Telegram surface is conversationally paired versus primarily outbound delivery
- whether voice-note handling is currently active on Telegram specifically
- whether the historical weekly heartbeat job is still enabled in its original form today

---

## 8. Fully grounded vs. best-effort summary

### Fully grounded
- live gateway service running
- Telegram env key presence in `.env`
- current Telegram adapter timeout / reconnect log evidence
- current absence of pairing data from `hermes pairing list`
- current Telegram toolset scope in `config.yaml`
- current and paused Telegram-delivered cron job inventory
- session-backed fact that Telegram setup was routed through `hermes gateway setup` with BotFather token flow on June 5

### Best-effort / needs review
- exact first successful inbound pairing moment
- any current Telegram voice-note workflow
- whether every historical Telegram automation referenced in sessions is still active today
