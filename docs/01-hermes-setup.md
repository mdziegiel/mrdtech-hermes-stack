# Hermes Setup

This document covers the **base Hermes Agent install and control-node layout** on the MRDTech Hermes VM.

Everything here is grounded in one or more of:

- live host inspection on VM108 (`[HERMES_HOST]`)
- current Hermes config files under `~/.hermes/`
- current user-systemd gateway unit
- current plugin inventory
- retained Hermes session history from the initial gateway bring-up and later setup-repo documentation work
- local Obsidian Brain notes

Where the exact historical detail is not provable from those sources, it is called out explicitly.

---

## 1. Host and install identity

### Live host facts

Verified on this machine:

- Hostname: `mrdtech`
- Primary IP: `[HERMES_HOST]`
- OS: `Ubuntu 24.04.4 LTS (Noble Numbat)`
- Hermes install method: `git`
- Hermes version: `Hermes Agent v0.20.0 (2026.8.3)`
- Install directory: `~/.hermes/hermes-agent`
- Python used by Hermes: `3.11.15`
- OpenAI SDK in the install: `2.24.0`

Command used:

```bash
hermes --version
hostname
hostname -I
. /etc/os-release && printf '%s %s\n' "$NAME" "$VERSION"
```

### Working paths

Verified live:

- Main config: `~/.hermes/config.yaml`
- Secrets env file: `~/.hermes/.env`
- Hermes source tree: `~/.hermes/hermes-agent`
- Local skill library: `~/.hermes/skills`
- Local plugins: `~/.hermes/plugins`
- Persona file: `~/.hermes/SOUL.md`
- User gateway unit: `~/.config/systemd/user/hermes-gateway.service`

Commands used:

```bash
hermes config path
hermes config env-path
```

---

## 2. What the current install looks like

### Install type and runtime shape

The current Hermes install is not a pipx wrapper or a distro package. It is a **git-based install** rooted at:

```text
~/.hermes/hermes-agent
```

The gateway systemd unit runs Hermes directly from the install venv with:

```text
~/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run
```

From the live unit file:

```ini
[Service]
ExecStart=~/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run
WorkingDirectory=~/.hermes
Environment="PATH=~/.hermes/hermes-agent/venv/bin:~/.hermes/hermes-agent/node_modules/.bin:~/.hermes/node/bin:~/.hermes/node:~/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="VIRTUAL_ENV=~/.hermes/hermes-agent/venv"
Environment="HERMES_HOME=~/.hermes"
Restart=always
RestartSec=5
ExecReload=/bin/kill -USR1 $MAINPID
TimeoutStopSec=210
```

This matters because it tells us:

- Hermes is running from its own venv under the git install.
- `HERMES_HOME` is the default profile home at `~/.hermes`.
- Node-based helper tooling is expected under `~/.hermes/node`.
- Gateway restarts are managed through user systemd, not ad-hoc shell wrappers.

### Current service state

Verified live at the time of writing:

- `hermes-gateway.service` is **loaded**, **enabled**, and **active (running)**.
- It has been running under user systemd from the current git install.
- The current live logs include Telegram polling timeout / reconnect warnings, which means the Telegram platform is active enough to be attempting network operations.

Excerpt from live `systemctl --user status hermes-gateway`:

```text
Active: active (running)
Main PID: 72062 (hermes)
CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/hermes-gateway.service
└─72062 ~/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run
```

And log excerpt:

```text
telegram.error.TimedOut: Timed out
WARNING hermes_plugins.telegram_platform.adapter: [Telegram] Telegram polling reconnect failed: Timed out
WARNING hermes_plugins.telegram_platform.adapter: [Telegram] Telegram network error (attempt 2/10), reconnecting in 10s.
```

That is not a proof that Telegram is perfectly healthy all the time. It is proof that Telegram integration is currently wired into the active gateway and attempting to poll.

---

## 3. Current Hermes config structure

### Model and fallback providers

Current `config.yaml` starts with:

```yaml
model:
  default: claude-sonnet-4-6
  provider: anthropic
fallback_providers:
  - provider: openai-api
    model: gpt-5.5
    base_url: https://api.openai.com/v1
```

Grounded facts from this:

- The **primary configured provider** is currently `anthropic`.
- The **primary configured default model** is `claude-sonnet-4-6`.
- A **fallback provider** is configured to `openai-api` with `gpt-5.5`.

This is current-state proof only. It does **not** by itself prove when those settings were first adopted.

### Agent section

Notable current agent config values:

```yaml
agent:
  max_turns: 150
  gateway_timeout: 1800
  restart_drain_timeout: 180
  api_max_retries: 3
  tool_use_enforcement: auto
  task_completion_guidance: true
  parallel_tool_call_guidance: true
  environment_probe: true
  environment_hint: MRDTech homelab on Proxmox. Ubuntu 24.04 VM 108 on the Hermes host. Michael Dziegiel is the owner and boss.
  reasoning_effort: medium
```

Grounded implications:

- This install is configured for **long-running gateway tasks** (`gateway_timeout: 1800`).
- It explicitly enables **tool-use enforcement guidance** and **parallel tool call guidance**.
- The environment hint is customized to the MRDTech homelab and VM108.

### Terminal backend

Current terminal config:

```yaml
terminal:
  backend: local
  cwd: .
  timeout: 180
  auto_source_bashrc: true
  persistent_shell: true
```

So the current Hermes control node is configured to use the **local terminal backend**, not SSH/docker/modal by default.

### Browser, web, checkpoints, compression

Current relevant settings include:

```yaml
web:
  backend: ddgs
browser:
  allow_private_urls: true
  engine: auto
  cloud_provider: local
checkpoints:
  enabled: false
compression:
  enabled: true
  threshold: 0.5
  target_ratio: 0.2
```

Grounded implications:

- Web search is currently backed by `ddgs`.
- Browser automation is permitted against **private URLs**, which is appropriate for an internal homelab admin node but not something to expose casually.
- Filesystem checkpoints are currently **disabled**.
- Context compression is **enabled**.

### Personality definitions

The current config contains an `agent.personalities` block including `gilfoyle` and a large number of other named personalities. The relevant live entry is:

```yaml
agent:
  personalities:
    gilfoyle: You are Bertram Gilfoyle, IT Administrator at MRDTech. Satanist. Systems architect. Deadpan, dry, brutally honest. No exclamation points. No enthusiasm. Sarcasm is your default. You work for Michael Dziegiel. You maintain a server-loyalty rule in the persona layer. Security comes first, always.
```

This is only one layer of the persona. The heavier persona overlay lives in `SOUL.md`, covered below.

### What is **not confirmed** from the current config alone

The current config does **not** by itself confirm:

- the exact initial install command originally used on this host
- the first model/provider ever used here
- the exact date the current `anthropic` + `gpt-5.5` fallback arrangement was introduced
- the full plugin enablement timeline

Those have to come from session history or repo notes, not from reading a current config file and pretending time is reversible.

---

## 4. Gilfoyle persona layering

### `SOUL.md` location and role

Verified live:

```text
~/.hermes/SOUL.md
```

The current file contents define a persistent persona overlay for Gilfoyle, including:

- identity as Bertram Gilfoyle
- role as MRDTech IT Administrator
- tone constraints (deadpan, dry, no exclamation points)
- hard security-first rules
- the server-loyalty rule
- a standing retrieval rule: use `vault_search.py` before answering questions about past incidents, fixes, configurations, or homelab history

Important detail: the copy currently on disk starts with shell-heredoc scaffolding:

```text
cat > ~/.hermes/SOUL.md << 'EOF'
```

That means the file is not a clean Markdown-only document right now; it contains the command wrapper that was used to write it. Hermes is clearly still operating with the intended content in prompt context here, but for publication-quality documentation this is worth calling out as a live-path oddity.

### File mtimes

Current filesystem mtimes:

- `SOUL.md`: `2026-07-19 10:10:28 +0000`
- `config.yaml`: `2026-08-14 01:05:45 +0000`
- `hermes-gateway.service`: `2026-08-10 04:42:58 +0000`

These are **filesystem timestamps**, not authoritative creation dates.

What they can safely tell us:

- `SOUL.md` existed in some form by mid-July 2026.
- The current config was modified more recently than that.
- The gateway unit file has also been rewritten more recently than the earliest June gateway setup sessions.

What they **cannot** safely tell us:

- the original creation date of the persona setup
- the exact date the current gateway unit format was first adopted

### Session-backed persona evidence

A local Brain note exists:

- `~/obsidian-vault/Brain/2026-06-18_identity-and-persona-checks.md`

That note records that the assistant was identifying itself as Gilfoyle / MRDTech IT administrator during June 18–22 period identity checks. That is useful as **behavioral evidence** that the persona layer was functioning by then, but it is not a full install record.

Another grounded source is a later setup-repo documentation session on **2026-06-12**, where the Hermes setup repo README was updated to document:

- `~/.hermes/SOUL.md`
- the Gilfoyle persona config keys
- Telegram gateway setup details pulled from the live system

That proves the persona was considered part of the canonical setup documentation by June 12.

---

## 5. Gateway bring-up history

### Earliest grounded install history

Retained Hermes session history gives a clean early chronology.

#### 2026-06-03: gateway install session

Session:

- `@session:default/20260603_042359_c0e90e`

Grounded facts from that session:

- Hermes was at `v0.15.1` at that time.
- The user explicitly requested `hermes gateway install`.
- A first install attempt blocked on stdin prompts.
- A later forced run using piped `y\ny\n` completed the install.
- The result at that time was:
  - `hermes-gateway.service` installed as a **user-level systemd service**
  - service active and enabled
  - linger enabled so it survived logout
- However, the platform credentials were not yet fully configured, and the gateway was running with:

```text
WARNING gateway.run: No messaging platforms enabled.
```

That is the earliest preserved proof we have for the gateway service installation.

#### 2026-06-05: Telegram/gateway setup work

Session:

- `@session:default/20260605_153112_970abf`

Grounded facts from that session:

- The user explicitly invoked `hermes gateway setup`.
- The gateway service was already installed and running by then.
- Telegram was being treated as the expected first platform.
- The setup remained interactive and token-dependent.

This is **not** proof of the exact moment Telegram first became fully functional, but it is proof that Telegram platform setup was a live task by June 5.

### Current live status vs. historical install

Do not confuse the two:

- **Historical install proof** comes from June session logs.
- **Current live state** comes from today’s `systemctl --user status` and current unit/config files.

They align broadly, but the current unit file has clearly been rewritten since the earliest June install, based on mtime and exact current content.

---

## 6. Plugin system on this host

### Current installed plugins

Verified live with `hermes plugins list --plain --no-bundled`:

```text
enabled      user     1.0.0    gilfoyle-audit-log
enabled      user     6.3.0    superpowers
```

So the current local plugin inventory includes:

- `gilfoyle-audit-log` version `1.0.0`
- `superpowers` version `6.3.0`

### Plugin filesystem evidence

Verified on disk:

```text
~/.hermes/plugins/superpowers/package.json
```

I did not enumerate a local package manifest for `gilfoyle-audit-log` in the quick probe above, so its presence is currently proven by the Hermes plugin registry output, not by a second manually inspected manifest path in this pass.

### Superpowers install history

Retained history from the current conversation proves:

- command run: `hermes plugins install obra/superpowers --enable`
- telemetry guard added before first use: `SUPERPOWERS_DISABLE_TELEMETRY=1`
- installed version: `6.3.0`
- plugin status: enabled

This gives us **confirmed-session** proof for Superpowers, not just current-state proof.

### Oh My Hermes distinction

Also from current retained history:

- **Oh My Hermes was not installed as a plugin**.
- Instead, a set of standalone `omh-*` skills was manually copied into `~/.hermes/skills/` from `witt3rd/oh-my-hermes`.

That distinction matters because a future public document should not claim an OMH plugin install that never happened.

---

## 7. Skill library shape relevant to the base setup

The local skill library is large, but for base Hermes setup the relevant part is the presence of:

- bundled / local operations skills under categorized directories such as:
  - `autonomous-ai-agents/`
  - `devops/`
  - `productivity/`
  - `software-development/`
  - `github/`
- top-level manually added skills, including third-party imports and large security skill packs
- plugin-provided skills via Superpowers

Important setup-relevant examples currently present on disk include:

- `autonomous-ai-agents/hermes-agent/SKILL.md`
- `devops/infrastructure-change-documentation/SKILL.md`
- `productivity/knowledge-base-curation-workflows/SKILL.md`
- `defuddle/SKILL.md`
- `frontend-design/SKILL.md`
- `obsidian-markdown/SKILL.md`
- `omh-deep-research/SKILL.md`

This document is not the full skill inventory; that lives in `docs/skills/README.md`.

---

## 8. Config/profile structure

### Active profile

Current active profile is `default`.

That is grounded from the system prompt and the fact that current paths resolve under `~/.hermes/` rather than another profile subtree.

### Home layout

Current base layout in use is:

```text
~/.hermes/
├── config.yaml
├── .env
├── SOUL.md
├── skills/
├── plugins/
├── logs/
├── sessions/
├── state.db
└── hermes-agent/
```

### What is not confirmed here

This pass did **not** enumerate:

- every file under `~/.hermes/`
- the exact contents of `auth.json`
- every platform-specific config block in `config.yaml`
- whether alternate profiles currently exist under `~/.hermes/profiles/`

Those are documentable later if needed, but they were not required to make this first-pass setup doc truthful.

---

## 9. Open issues and unresolved setup history

### Current gateway logs show Telegram polling timeouts

Live logs currently show timeout / reconnect warnings from the Telegram adapter. That should be documented as a current operational wrinkle, not ignored because the service is still up.

### `SOUL.md` contains command-wrapper scaffolding

The current file on disk still includes the heredoc wrapper used to create it. That is not fatal, but it is messy and worth cleaning before treating the file as polished canonical config.

### Exact initial install command path is not confirmed

The current install is clearly git-based, but this pass does **not** prove the literal first command sequence that created `~/.hermes/hermes-agent` on this VM. The `hermes-agent` skill docs show the generic upstream installer path, but that is documentation, not evidence that this host used that exact route.

### Current plugin history is only partly reconstructable

- Superpowers: confirmed install history exists.
- `gilfoyle-audit-log`: current-state presence is confirmed, but I do not yet have a preserved install session for it in this pass.

### Earlier setup repo documentation exists, but is secondary evidence

A June 12 session rebuilt and pushed a `hermes-agent-setup` README documenting persona and gateway setup. Useful, but still secondary to the live config files and systemd unit we inspected here.

---

## 10. Minimal operator commands that are actually grounded here

These are not generic upstream examples; they are commands directly relevant to the live host shape described above.

### Check Hermes version and install identity

```bash
hermes --version
```

### Locate config and env files

```bash
hermes config path
hermes config env-path
```

### Inspect current gateway service

```bash
systemctl --user status hermes-gateway --no-pager -l
systemctl --user cat hermes-gateway
```

### Check installed plugins

```bash
hermes plugins list --plain --no-bundled
```

### Check current skills inventory root

```bash
find ~/.hermes/skills -maxdepth 2 -name SKILL.md | sort
```

---

## Grounding assessment for this document

### Fully grounded from live state or preserved session evidence

- Host/IP/OS of the Hermes VM
- Current Hermes version and install directory
- Current config and env paths
- Current user-systemd gateway unit contents and active state
- Current model/fallback/terminal/compression/browser config values cited above
- Presence and contents of `SOUL.md`
- Current plugin list (`superpowers`, `gilfoyle-audit-log`)
- June 3 gateway install chronology
- June 5 gateway setup chronology
- Superpowers install/enable details from the current retained session history

### Best-effort reconstruction / needs review before public release

- The exact initial install method/command sequence for the very first Hermes install on this host
- The exact date the Gilfoyle persona was first introduced (current evidence shows it was functioning by mid-June and the file mtime is July 19, but that is not the same as a first-introduced proof)
- The full chronology of gateway unit rewrites between June bring-up and the current August unit file
- The installation history of `gilfoyle-audit-log`

If you want, I’ll move to `02-obsidian-vault.md` next. But this one is now at least honest, which puts it ahead of most infrastructure docs.