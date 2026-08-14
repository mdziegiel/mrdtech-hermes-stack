# Dual-Model Delegation

This document covers the **delegation/runtime split** between Claude Code and Codex-related runtime support in the MRDTech Hermes install.

This is the least cleanly provable part of the stack-doc series. The live host proves some things decisively and only partially proves others. So this document separates:

- what is **definitely installed and active now**
- what is **present in Hermes core as a supported runtime path**
- what is **only reconstructable from older config snapshots/session history**

Anything else would be dishonest.

---

## 1. Current live state on `[HERMES_HOST]`

### Claude Code CLI
Verified live:
- binary path:
  - `~/.hermes/node/bin/claude`
- version:
  - `2.1.226 (Claude Code)`
- auth status:
  - `Login method: Claude Pro account`
  - `Organization: michael.dziegiel@gmail.com's Organization`
  - `Email: michael.dziegiel@gmail.com`

That is hard proof that Claude Code is installed and authenticated right now.

### Codex CLI
Verified live:
- `command -v codex` returned no installed binary
- `codex --version` produced no version output

So there is **no current standalone Codex CLI binary** provable on this host during this draft.

### Current delegation block in config
Verified in `~/.hermes/config.yaml`:

```yaml
delegation:
  model: ''
  provider: ''
  base_url: ''
  api_key: ''
  api_mode: ''
  inherit_mcp_toolsets: true
  max_iterations: 50
  child_timeout_seconds: 600
  reasoning_effort: ''
  max_concurrent_children: 3
  max_spawn_depth: 1
  orchestrator_enabled: true
  subagent_auto_approve: false
```

This matters because it proves that **current child-agent delegation is not pinned to a dedicated provider/model in config**. In other words: no explicit live “send delegation to Codex” or “send delegation to Claude Code” setting is currently configured here.

### Current selected runtime vs. delegation
The current live delegation block is blank, but the Hermes install still contains Codex runtime support code and earlier config snapshots show Codex-provider usage at the main model layer. That is a runtime story, not a clean current delegation-target story.

---

## 2. Evidence of Codex support in Hermes core

### On-disk Codex runtime code
Verified files exist under the local Hermes source tree:
- `~/.hermes/hermes-agent/agent/codex_runtime.py`
- `~/.hermes/hermes-agent/agent/codex_responses_adapter.py`
- `~/.hermes/hermes-agent/hermes_cli/codex_runtime_switch.py`
- `~/.hermes/hermes-agent/hermes_cli/codex_models.py`
- `~/.hermes/hermes-agent/hermes_cli/codex_runtime_plugin_migration.py`

### What the runtime switch file proves
From `hermes_cli/codex_runtime_switch.py`:
- Hermes supports toggling `model.openai_runtime` between:
  - `auto`
  - `codex_app_server`
- the user-facing command accepts synonyms like:
  - `on`
  - `codex`
  - `enable`
  - `off`
  - `disable`
- enabling `codex_app_server` is blocked unless Codex CLI is present
- if enabled, Hermes can migrate MCP servers and curated plugins into Codex runtime config
- the migration code explicitly mentions registering the Hermes tool callback so Codex can use selected Hermes/MCP tools

That is real implementation, not theory.

### What the runtime file proves
From `agent/codex_runtime.py`:
- Hermes has a dedicated Codex API/runtime path
- comments explicitly distinguish:
  - `codex_app_server` subprocess path
  - `codex_responses` API path
- the runtime exists for both app-server and Responses-style Codex interactions

### What the adapter file proves
From `agent/codex_responses_adapter.py`:
- Hermes has explicit normalization logic for OpenAI Responses-compatible backends
- the file header says this path is used by:
  - OpenAI Codex
  - xAI
  - GitHub Models
  - other Responses-compatible endpoints

So Codex support is not imaginary. It is real code in the local Hermes core.

---

## 3. Historical config evidence for Codex usage

### Snapshot evidence
Verified in:
- `~/.hermes/state-snapshots/20260721-025845-pre-update/config.yaml`

That snapshot shows:
- `model.default: gpt-5.4`
- `model.provider: openai-codex`
- `fallback_providers` present
- delegation block still blank for explicit provider/model pinning

Safe conclusion:
- by July 21, the **main model runtime** had been configured to use `openai-codex`
- that does **not** by itself prove child delegation was separately routed to Codex
- it **does** prove Codex was not just an abandoned codepath; it was part of the actual model/runtime configuration history

### Earlier session evidence
Session search found a June 5 session indicating that `OpenAI Codex` was still considered an optional/unconfigured provider at that earlier point.

Session found:
- `@session:default/20260605_183844_9231a2`

That session’s snippet states that optional providers such as `OpenAI Codex` were not yet configured at that time.

So the clean timeline appears to be:
1. early June: Codex support existed but was not configured
2. by July 21 snapshot: `openai-codex` was the active top-level model provider
3. today: live config delegation block is unpinned, Codex runtime support code remains, but no standalone `codex` binary is currently installed on-host

That is not a contradiction. It is what drift looks like when systems evolve faster than their paperwork.

---

## 4. Claude Code side of the split

### What is proven now
- Claude Code CLI is installed
- Claude Code CLI is authenticated
- auth method is account-based (`Claude Pro account`), not an API key string printed from config

### What that implies about the auth decision
The live auth status strongly supports this statement:
- **Claude Code was wired through account/OAuth-style login rather than a raw Anthropic API key-only CLI setup**

I am labeling that phrasing carefully because the CLI prints `Claude Pro account`, not “OAuth token,” but it is clearly account-authenticated rather than anonymous or absent.

### What is not proven in this pass
I did **not** capture a fresh live child-agent run explicitly executed through Claude Code CLI during this documentation pass.

So the safe wording is:
- Claude Code is **installed and ready as a real local runtime surface**
- explicit current delegation targeting to Claude Code is **not directly pinned in current config.yaml**

---

## 5. Security scoping decisions visible in live config/code

The user specifically asked about:
- OAuth vs API key
- permission flags
- `--max-turns` cap

This section splits those by proof level.

### A. OAuth vs API key

#### Fully grounded
- Claude Code auth is live and account-based (`Claude Pro account`)
- current `delegation.api_key` in config is blank
- current `delegation.provider` and `delegation.model` are blank
- current top-level config snapshot history shows `openai-codex` as a model provider in a prior snapshot

#### Best-effort reading
- Claude Code side appears to have been intentionally authenticated through account login rather than through raw per-delegation API key config
- Codex-side auth method is **not fully proven in this pass** because no live standalone `codex` binary is present and no live Codex CLI auth state was available to inspect

### B. Permission defaults / approval posture

#### Fully grounded
Current live delegation config on `[HERMES_HOST]` shows:
- `subagent_auto_approve: false`
- `max_spawn_depth: 1`
- `max_concurrent_children: 3`
- `inherit_mcp_toolsets: true`

From Codex runtime migration code:
- the migration logic can write a default sandbox/permissions setting for Codex runtime
- comments explicitly say this helps avoid approval prompts on every write
- Hermes tool callback registration is selective and documented in the code comments

That proves the system was designed with **approval/sandbox posture as a first-class concern**.

#### Not fully proven in this pass
- I did **not** inspect a live generated `~/.codex/config.toml` or equivalent current Codex CLI config on this host
- I did **not** capture a live permission flag value from a working Codex CLI install

So I cannot honestly claim the exact current Codex permission string in production.

### C. `--max-turns` cap / turn limits

#### Fully grounded
Current live config shows:
- `delegation.max_iterations: 50`
- global goals section includes:
  - `goals.max_turns: 20`

That is not the same thing as a literal CLI `--max-turns` flag capture, but it is a real current turn/iteration governance setting in the live host config.

#### Best-effort only
- I do **not** have a fresh captured invocation line proving a current external Claude Code or Codex delegate command was launched with literal `--max-turns <n>` during this doc pass
- session search found `--max-turns` in historical context, but I am not elevating that to “current production proof” without a live invocation or exact historical command capture

---

## 6. What the split appears to have been

### Most defensible reading from all evidence
The cleanest reading of the system is:

1. **Claude Code** became the locally installed, authenticated external coding runtime
2. **Codex** became a supported Hermes runtime/provider path and at one point a configured top-level model provider (`openai-codex`)
3. **Delegation itself** was intentionally left flexible in current config rather than hard-pinned under `delegation.provider/model`
4. approval and spawn controls were kept explicit:
   - `subagent_auto_approve: false`
   - `max_spawn_depth: 1`
   - `max_concurrent_children: 3`
   - `max_iterations: 50`

That is not a pretty “two perfect delegation targets fully pinned forever” story. It is a real operational story: one local CLI path clearly active, one core-runtime path clearly implemented, and current config leaving room for runtime/provider resolution rather than forcing a single hardcoded child route.

---

## 7. Open issues and drift

### Confirmed drift
- current host has Claude Code installed but no live `codex` binary
- Hermes core still contains substantial Codex runtime support
- older config snapshot shows active `openai-codex` provider use
- current `delegation` block is unpinned

That is operational drift. The codebase remembers more than the host currently exposes.

### Review items before public release
This doc should likely be reviewed before publication because the title “dual-model delegation” is stronger than the current live proof.

Safer interpretations supported by evidence are:
- “Claude Code + Codex runtime integration surfaces”
- “delegation/runtime split between Claude Code and Codex support”

Those are more honest than pretending both are currently symmetrical live targets.

---

## 8. Fully grounded vs. best-effort summary

### Fully grounded
- Claude Code binary path and version
- Claude Code live auth status (`Claude Pro account`)
- absence of a live `codex` binary today
- current live delegation block values in `config.yaml`
- existence of Codex runtime, adapter, and runtime-switch files in Hermes core
- runtime-switch behavior supporting `codex_app_server`
- July 21 snapshot showing `model.provider: openai-codex`
- current spawn/approval controls (`subagent_auto_approve: false`, `max_iterations: 50`, etc.)

### Best-effort / needs review
- exact historical moment when child delegation was first routed through Claude Code versus Codex
- exact current Codex auth method, since no live Codex CLI is installed now
- exact live Codex permission/default sandbox config on this host
- any claim that both targets are presently equally active as delegation backends
