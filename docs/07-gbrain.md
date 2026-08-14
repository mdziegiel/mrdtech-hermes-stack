# GBrain

This document covers the GBrain install and stabilization work on `[INFRA_HOST]`, including the PGLite choice, the Ollama embedding timeout saga, the workaround via file exclusion, and the current final counted state.

Unlike some software, GBrain did eventually do what it was told. It just needed to be cornered first.

---

## 1. Current live state on `[INFRA_HOST]`

### Install/runtime basics
Verified live on `[INFRA_HOST]`:
- binary on PATH:
  - `~/.bun/bin/gbrain`
- version:
  - `gbrain 0.44.1.0`
- current doctor reports upgrade available to:
  - `0.45.12.0`

### Current engine/config
Verified from `~/.gbrain/config.json`:

```json
{
  "engine": "pglite",
  "database_path": "~/.gbrain/brain.pglite",
  "embedding_model": "ollama:nomic-embed-text",
  "embedding_dimensions": 768,
  "chat_model": "anthropic:claude-3-5-haiku-latest",
  "schema_pack": "gbrain-base-v2",
  "mcp": {
    "publish_skills": true
  },
  "self_upgrade": {
    "mode": "notify",
    "mode_prompted": true
  },
  "protocol_installed_at": "2026-08-12T19:15:42.469Z"
}
```

### Current local database path
Verified live:
- PGLite DB path:
  - `~/.gbrain/brain.pglite`

Files present under that directory include:
- `PG_VERSION`
- `postgresql.conf`
- `postgresql.auto.conf`
- `pg_hba.conf`
- `pg_ident.conf`

So yes, this is a real local PGLite deployment, not somebody’s wishful README state.

### Current stats
Verified live via `gbrain stats`:
- `Pages: 239`
- `Chunks: 705`
- `Embedded: 705`
- `Links: 0`
- `Tags: 7`
- `Timeline: 0`

By type:
- `note: 149`
- `concept: 90`

This is the final closeout count the user explicitly wanted preserved.

### Current doctor state
Verified live via `gbrain doctor`:
- connection: `Connected, 239 pages`
- embeddings: `100% coverage, 0 missing`
- current hard failure still present:
  - `sync_failures → 78 unresolved sync failure(s)`
- current embedding-provider probe warning:
  - `Embedding provider probe failed: [embed(ollama:nomic-embed-text)] Cannot connect to API: Unable to connect.`

So the brain is not empty and not dead, but it is also not “clean” in the literal doctor sense today.

That distinction matters.

---

## 2. Why PGLite was chosen

### User-requested architecture
Session history clearly shows the install was requested on `[INFRA_HOST]` with:
- standalone first
- PGLite engine
- remote Ollama embeddings on `[HERMES_HOST]`
- Anthropic Haiku for chat/synthesis
- source corpus from `~/obsidian-vault` on `[INFRA_HOST]`

Relevant session:
- `@session:default/20260812_183930_473871`

### Current config confirms that choice stuck
Current `~/.gbrain/config.json` proves:
- `engine: pglite`
- `database_path: ~/.gbrain/brain.pglite`
- `embedding_model: ollama:nomic-embed-text`
- `chat_model: anthropic:claude-3-5-haiku-latest`

So PGLite was not merely planned. It is the live persistent engine right now.

---

## 3. Install/version pinning saga

### Evidence of deliberate pinning
Session history around the initial install records that latest was **not** used blindly. The install work explicitly checked issue/changelog context around known Ollama/PGLite breakage before pinning.

Relevant session family:
- `@session:default/20260812_183930_473871`
- `@session:default/20260812_184322_44f15d`

The user requirement in that session explicitly referenced:
- issue `#2051`
- Ollama embedding dimension mismatch concerns on PGLite init
- need to pin a known-good version instead of grabbing latest

### Current pinned result
Current live binary confirms the pinned version remains:
- `0.44.1.0`

That lines up with the recorded install saga and later verification that `providers test`, `init`, and `doctor` eventually passed after local patching.

---

## 4. Provider-path and remote Ollama saga

### Confirmed local patch history
From the preserved session history already used elsewhere in this repo:
- GBrain was patched at:
  - `~/gbrain/src/commands/providers.ts`
- the patch goal was to make no-config provider tests honor the **remote Ollama base URL** instead of incorrectly targeting localhost
- after the patch, `providers test`, `init`, and `doctor` were reported as passing at that point in the workflow

This was already captured in the earlier session summary and is reinforced by later GBrain troubleshooting sessions.

### Why that patch mattered
Multiple session-history fragments show the recurring failure mode was effectively:
- provider test or embedding flow reaching the wrong endpoint, especially local/localhost-style target construction
- remote Ollama lives on `[HERMES_HOST]`, not `[INFRA_HOST]`

Session search for:
- `providers test`
- `localhost:11434/v1/embeddings`
- `v0.44.1.0`

returned the install/troubleshooting session family that documented this explicitly.

### What is confirmed today
Current live config still says:
- `embedding_model: ollama:nomic-embed-text`
- `embedding_dimensions: 768`

But current `gbrain doctor` still shows the provider probe warning:
- `Cannot connect to API: Unable to connect.`

So the cleanest accurate statement is:
- **the install was successfully patched far enough to initialize and build the current 239/705 corpus state**
- **the provider probe is not clean today**

That sounds contradictory only if you assume software is required to be coherent.

---

## 5. The timeout investigation

### Hard evidence of timeout failures
Current live file on `[INFRA_HOST]`:
- `~/.gbrain/sync-failures.jsonl`

Tail entries currently show repeated errors of the form:
- `[embed(ollama:nomic-embed-text)] The operation timed out.`

The failures are attached to specific vault files under paths like:
- `Clippings/...`
- `Brain/...`
- `Activity.md`
- `Archive/...`

This is direct live proof that the timeout saga was real, not mythology built from frustration.

### Current doctor corroboration
Current `gbrain doctor` reports:
- `sync_failures → 78 unresolved sync failure(s)`
- `embedding_provider → Cannot connect to API`

That corroborates the JSONL evidence.

### Best root-cause status
The user asked for the root-cause status to be stated plainly:
- **never fully resolved**
- **worked around via file exclusion**

That statement is still the honest one.

Why:
- the corpus did reach a stable all-accounted-for closeout state (`239/239`, `705 chunks`)
- but the broad timeout class was not eliminated in some eternal, universal sense
- current doctor output still shows unresolved sync/provider issues

In other words, the system was made operational, not perfected. Those are different religions.

---

## 6. File-exclusion workaround and closeout

### Direct session proof of the closeout move
Session search found the exact closeout record for the final exclusion-based cleanup.

Relevant session:
- `@session:default/20260813_131821_ab02e6`

That session explicitly records:
- pending files were moved out of the active import path
- final exclusion archive path became:
  - `~/Excluded-GBrain`
- final pending count reached:
  - `0`
- final `gbrain stats` reported:
  - `Pages: 239`
  - `Chunks: 705`
  - `Embedded: 705`

### Why that workaround mattered
The closeout session explains an important mistake that was corrected:
- files initially moved under `~/obsidian-vault/Excluded-GBrain/` would still remain **inside** the import tree
- the exclusion archive had to be moved **outside** the vault to actually stop import attempts

That is the kind of detail only retained because it hurt once.

### Current live state still aligns with closeout
Verified live now:
- `gbrain stats` still reports the same final counts:
  - `239` pages
  - `705` chunks
  - `705` embedded

So the final reported closeout state is not just session lore. It still matches the current live stats.

---

## 7. Storage/layout details

### Live config and state files
On `[INFRA_HOST]`, current GBrain files verified include:
- `~/.gbrain/config.json`
- `~/.gbrain/import-checkpoint.json`
- `~/.gbrain/sync-failures.jsonl`
- `~/.gbrain/audit/content-sanity-2026-W33.jsonl`
- `~/.gbrain/audit/skill-brain-first-snapshot.json`

### Current gbrain.yml
Verified file:
- `~/gbrain/gbrain.yml`

Current visible storage config declares tracked directories such as:
- `people/`
- `companies/`
- `deals/`
- `concepts/`
- `yc/`
- `ideas/`
- `projects/`

and `db_only` directories such as:
- `media/x/`
- `media/articles/`
- `meetings/transcripts/`

This file is broader than the vault-import saga itself, but it proves the install has real persistent configuration rather than disposable default-only state.

---

## 8. Current health caveats

### Fully grounded current caveats
Current `gbrain doctor` still reports:
- `78 unresolved sync failure(s)`
- embedding-provider probe failure
- warnings around content sanity audit and extraction backlog

So if this doc were to claim “clean init, fully healthy, no remaining issues,” it would be lying.

### What is still true despite that
Also fully grounded:
- database connection is healthy
- schema version is current (`125`)
- embeddings coverage is `100%` for the retained active corpus
- the active counted corpus is still `239 pages / 705 chunks / 705 embedded`

So the correct conclusion is not “broken” or “done.” It is:
- **operational with a constrained corpus and known unresolved sync/provider issues**

Which, frankly, describes half of self-hosted software worth using.

---

## 9. Open issues and review notes

### Fully grounded
- GBrain is installed on `[INFRA_HOST]`
- version remains `0.44.1.0`
- engine is `pglite`
- embedding model is `ollama:nomic-embed-text`
- chat model is `anthropic:claude-3-5-haiku-latest`
- live stats are still `239 / 705 / 705`
- timeout failures are real and visible in `sync-failures.jsonl`
- current doctor still reports unresolved sync/provider issues
- final exclusion workaround path from session history is `~/Excluded-GBrain`

### Best-effort / needs review
- exact full list of every excluded file at final closeout was not reprinted in this draft
- exact current remote Ollama base URL value in the running GBrain environment was not re-derived in this pass from process env
- whether the current 78 unresolved failures are entirely outside the retained 239-page active corpus or reflect later post-closeout drift needs separate focused follow-up

That last point matters. The closeout state and the current doctor warnings can both be true if later content or re-sync attempts reopened the wound.

---

## 10. Fully grounded vs. best-effort summary

### Fully grounded
- current GBrain version and binary path
- current `config.json` contents (`pglite`, `ollama:nomic-embed-text`, `768`, `anthropic:claude-3-5-haiku-latest`)
- current PGLite database path and files
- current `gbrain stats` output (`239`, `705`, `705`)
- current `gbrain doctor` failure/warning state
- current `sync-failures.jsonl` evidence of embed timeouts
- session-backed closeout proving the exclusion workaround and final 239/705 state

### Best-effort / needs review
- exact current remote provider env wiring at process level
- exact chronology of every intermediate failed retry before the final exclusion workaround
- whether today’s 78 unresolved sync failures represent new drift after the 239/705 closeout or just acknowledged historical backlog
