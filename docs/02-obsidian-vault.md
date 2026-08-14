# Obsidian Vault

This document covers the **private Obsidian vault**, the separate **public-facing WikiDocs instance**, and how the Gilfoyle/Hermes workflow uses the vault as durable memory, journal, and job-search workspace.

Everything here is grounded in one or more of:

- live git/worktree inspection on `[HERMES_HOST]` and `[INFRA_HOST]`
- live directory inspection on `[INFRA_HOST]` for WikiDocs datasets
- retained Hermes session history from the June/July WikiDocs and vault work
- local Obsidian Brain notes written during that work

Where a detail is reconstructed rather than directly proven, it is labeled that way.

---

## 1. Vault location and git repo

### Local vault on the Hermes VM (`[HERMES_HOST]`)

Verified live on this host:

- Path: `~/obsidian-vault`
- Git worktree: yes
- Remote: `git@github.com:mdziegiel/obsidian-vault.git`
- Branch: `main`
- Local HEAD on `[HERMES_HOST]` at time of writing: `866cf12bc3043931c0c84eb63c044a2540b59012`

Command used:

```bash
cd ~/obsidian-vault
git rev-parse --is-inside-work-tree
git remote get-url origin
git branch --show-current
git rev-parse HEAD
```

### Vault on `[INFRA_HOST]`

Verified live over SSH on `[INFRA_HOST]`:

- Path: `~/obsidian-vault`
- Git worktree: yes
- Remote: `git@github.com:mdziegiel/obsidian-vault.git`
- Branch: `main`
- Remote-side HEAD on `[INFRA_HOST]` at time of writing: `9ab9281f8e7adcd21c0f3d3669f6c61d3787076d`

That means the vault exists in at least two active places:

- the current Hermes VM at `[HERMES_HOST]`
- the `[INFRA_HOST]` host where a large amount of the WikiDocs/Command Center work was historically performed

The two HEADs are **not identical** right now. That is not necessarily an error, but it means you should not casually claim the two copies are perfectly synchronized unless you verify that specific point later.

---

## 2. Current vault structure

### Top-level folders on `[HERMES_HOST]`

Verified live:

```text
Archive/
Brain/
ClaudeHistory/
Clippings/
docker-compose-templates/
JobSearch/
Journal/
Projects/
Sync/
Templates/
Wiki/
```

Also present:

- `.git/`
- `.obsidian/`
- `.trash/`

Command used:

```bash
cd ~/obsidian-vault
find . -maxdepth 1 -type d | sort
```

### Session-backed structure confirmation

A Brain note documents this explicitly:

- `~/obsidian-vault/Brain/2026-06-22_obsidian-vault-structure-confirmation.md`

That note records a confirmed vault structure under `~/obsidian-vault/` with required folders including:

- `Brain/`
- `ClaudeHistory/`
- `ClaudeHistory/import-staging/`
- `JobSearch/`
- `Journal/`
- `Templates/`
- `Wiki/`

Another Brain note:

- `~/obsidian-vault/Brain/2026-06-22_obsidian-vault-and-wikidocs-mirror.md`

records that the vault path was corrected from an earlier mistaken assumption of `[REMOTE_VAULT_PATH]/` to the actual path `~/obsidian-vault/`.

That is useful because it proves this path was not guessed retroactively; it was explicitly corrected during the earlier work.

---

## 3. What the vault is for

The private Obsidian vault is not the same thing as the public docs surface.

### Private vault roles

Grounded from the live folder structure plus Brain notes, the vault serves at least these functions:

1. **Durable operational memory and session journal**
   - `Brain/` stores concise session-derived infrastructure notes.
2. **Job-search tracking and related automations**
   - `JobSearch/` is the workspace for application/job tracking flows.
3. **Sync inputs for dashboard / command-center workflows**
   - `Sync/` currently contains files such as:
     - `calendar-today.md`
     - `inbox-summary.md`
     - `itnews-feed.json`
     - `job-matches.json`
4. **Knowledge capture / imported source material**
   - `Clippings/`, `Wiki/`, `Archive/`, and `Projects/` all indicate broader knowledge-management use beyond pure daily journaling.

### Brain notes as the durable memory layer

A specific session-backed note proves the intentional use of `Brain/` as durable operational memory:

- `~/obsidian-vault/Brain/2026-06-23_obsidian-session-note-capture-skill.md`

That note records creation of a Hermes skill at:

```text
~/.hermes/skills/productivity/obsidian-session-note-capture/SKILL.md
```

The note says the skill instructs future sessions to write notes under:

```text
~/obsidian-vault/Brain/
```

using a consistent format with:

- title
- approximate date
- `[[wikilinks]]`
- concise summary
- `Pending:` line

That is direct proof that the vault is being used as a **deliberate memory/journal surface** for substantive sessions, not just as an incidental notes folder.

---

## 4. How Gilfoyle uses the vault

### Running memory / historical retrieval

The current Gilfoyle `SOUL.md` on the Hermes host contains this standing rule:

> Before answering questions about past incidents, fixes, configurations, or homelab history, run `vault_search.py` with a concise query and ground the answer in results, citing `source_path`.

That makes the vault part of the operating model, not just a documentation afterthought.

Grounded implication:

- the vault is expected to be queried for **historical incident/configuration knowledge** before answering those classes of questions
- live-state checks still take precedence for current status questions

### Session-note capture behavior

The vault is also used as a **post-session memory sink**.

Grounded from the June 23 Brain note:

- substantive sessions are supposed to become concise `Brain/` notes
- trivial sessions are intentionally skipped
- secrets are not supposed to be copied into those notes

That is a strong sign that Gilfoyle/Hermes uses the vault as a **curated memory system**, not a transcript landfill.

### Job-search tracking

Two Brain notes show the vault’s job-search role clearly:

1. `~/obsidian-vault/Brain/2026-06-25_command-center-iconize-quickadd.md`
2. `~/obsidian-vault/Brain/2026-06-25_command-center-rss-scroll-job-watch.md`

Grounded facts from those notes:

- `QuickAdd` and `Commander` were configured for job-search capture.
- A `New Job Application` template flow was set up to write under `JobSearch/`.
- A test note was created under `JobSearch/`, verified, then removed.
- `Track this` behavior from the Command Center workflow was verified to create a correctly named and filled `JobSearch/` note, open it, and then remove the test note.
- `Sync/job-matches.json` is generated from the `job-watch` application and consumed by the Obsidian-side command-center/dashboard workflow.

This proves the vault is being used not just as a notebook, but as an active workspace for job-search operations and automation outputs.

---

## 5. WikiDocs is a separate public-facing business reference

### Important correction: it is not Wiki.js

Session history around June 12 explicitly proved this, and it should be stated bluntly because earlier tasks referred to “Wiki.js” incorrectly.

Grounded from retained session history:

- `http://[INFRA_HOST]:11122` is **not** Wiki.js
- `/graphql` on that service returns HTML, not a Wiki.js GraphQL API
- the running container was identified as:
  - name: `wikidocs-wikidocs-1`
  - image: `zavy86/wikidocs`

### Live WikiDocs repo and path on `[INFRA_HOST]`

Verified live over SSH on `[INFRA_HOST]`:

- Root path: `[WIKIDOCS_REPO_PATH]`
- Git remote: `git@github.com:mdziegiel/wikidocs.git`
- Branch: `main`
- Current HEAD at time of writing: `3a0b1f813ee7f5bb7c4cc1e4164dc697f33e2a18`

Verified dataset/document roots present:

```text
[WIKIDOCS_REPO_PATH]/datasets
[WIKIDOCS_DATASET_PATH]
[WIKIDOCS_REPO_PATH]/dataset
[WIKIDOCS_REPO_PATH]/dataset/documents
```

The presence of both `dataset/` and `datasets/` is live filesystem fact. This document does **not** assert why both exist without further proof, but the operational paths used in the session history consistently pointed at `datasets/documents`.

### Live document tree sample

Verified live under:

```text
[WIKIDOCS_DATASET_PATH]/
```

Sample pages currently present include:

- `about-michael/content.md`
- `architecture-overview/content.md`
- `backup-architecture-pbs-qnap/content.md`
- `daily-command-portal/content.md`
- `docker-compose-templates/content.md`
- `home-lab/content.md`
- `homepage/content.md`
- `infrastructure/content.md`
- `mrdtech-mcp-gateway/content.md`
- `projects/content.md`
- `runbooks/content.md`
- `security/content.md`
- `wikidocs-category-patch-maintenance/content.md`

This is good evidence that WikiDocs is being used as the **rendered knowledge base / business-facing reference layer**, distinct from the private vault.

### Role separation: private vault vs public WikiDocs

Grounded from the June 22 mirror note and the June 12 WikiDocs build/redesign sessions:

- **Private vault** (`~/obsidian-vault`) is the working knowledge space.
- **WikiDocs** (`[WIKIDOCS_DATASET_PATH]`) is the rendered / public-facing business reference.

The June 22 note says WikiDocs content was mirrored from:

```text
[INFRA_HOST]:[WIKIDOCS_DATASET_PATH]
```

into the vault path:

```text
~/obsidian-vault/Wiki/
```

That is direct evidence of the directional relationship:

- WikiDocs content can be mirrored into the private vault
- but the two are still separate surfaces with separate operational roles

---

## 6. Session-backed WikiDocs history

### June 12 build-out and redesign work

Retained session history around June 12 proves several important facts:

- The service at `[INFRA_HOST]:11122` was investigated and correctly identified as WikiDocs.
- WikiDocs page files live at:

```text
/datasets/documents/<slug>/content.md
```

- A builder/uploader workflow was created on the Hermes side:

```text
~/build_wikidocs.py
```

- Main pages including `about-michael`, `infrastructure`, `projects`, `docker-compose-templates`, `security`, `runbooks`, `hacker-culture`, and `home-lab` were built/updated through the flat-file deployment path.
- Later redesign work on June 12 changed the Projects page, merged sections, adjusted layout density, and added custom CSS.

This document is not repeating every June 12 content change in detail, but it is important to record that the public/business reference surface was actively curated through direct dataset edits rather than a proper API.

### Sidebar/homepage regression fix and commit `8705f60`

You asked specifically about commit `8705f60`.

Grounded conclusion:

- commit `8705f60` was **not found in `obsidian-vault`** on either `[HERMES_HOST]` or `[INFRA_HOST]`
- commit `8705f60` **was found in the WikiDocs repo** at `[WIKIDOCS_REPO_PATH]`

Verified live on `[INFRA_HOST]`:

```text
8705f60 Update WikiDocs homepage and sync incident notes
```

And the commit touched:

- `datasets/documents/homepage/content.md`
- `datasets/documents/known-issues-and-quirks/content.md`

That is important because it means the “sidebar regression fix” belongs to the **WikiDocs repo history**, not the Obsidian vault repo history.

### Related Brain note about the sidebar regression

There is also a local Brain note:

- `~/obsidian-vault/Brain/2026-07-28_wikidocs-homepage-sidebar-regression-fix.md`

Grounded facts from that note:

- Michael reported that the WikiDocs homepage Main Sections table was stale.
- The live source path was confirmed at:

```text
[WIKIDOCS_DATASET_PATH]/homepage/content.md
```

- The file timestamp was noted as `2026-07-05`.
- The homepage table was rebuilt from the **current rendered sidebar order**, not guessed from memory.
- A backup was created at:

```text
[WIKIDOCS_DATASET_PATH]/homepage/content.md.bak-sidebar-20260728042031
```

That Brain note does not itself name `8705f60`, but it clearly documents the same class of problem: homepage/sidebar structure drift in WikiDocs and direct file-based repair.

So the safe documentation position is:

- commit `8705f60` is a **WikiDocs repo commit** tied to homepage/incident-note sync
- the **July 28 Brain note** documents a later/related homepage sidebar regression fix workflow
- do **not** claim, without further proof, that `8705f60` alone fully resolved all later sidebar regressions

---

## 7. Obsidian ↔ WikiDocs mirroring

Grounded from the June 22 mirror note:

- WikiDocs content was mirrored from the Docker host source path:

```text
[INFRA_HOST]:[WIKIDOCS_DATASET_PATH]
```

into:

```text
~/obsidian-vault/Wiki/
```

The note says that mirror process:

- flattened nested document paths into filenames using `__` separators
- stripped WikiDocs custom `<style>` artifacts
- preserved Markdown and placeholders
- reported zero failures for 65 Markdown files

That is strong evidence that the private vault can carry a mirrored representation of the public WikiDocs corpus, but it does **not** mean the vault and WikiDocs are the same repository or the same editing surface.

---

## 8. Sync and dashboard integration

The `Sync/` folder is part of the live private vault and currently contains at least:

- `calendar-today.md`
- `inbox-summary.md`
- `itnews-feed.json`
- `job-matches.json`

Grounded from Brain notes:

- `Sync/itnews-feed.json` is fed by a Hermes-side FreshRSS fetch script.
- `Sync/job-matches.json` is fed from the `job-watch` application by a Hermes-side exporter script.
- Those files are consumed by the Obsidian Command Center / Daily Command Portal style workflows.

This supports the larger point that the private vault is not static documentation only; it is an automation-fed workspace.

---

## 9. Open issues / unresolved history relevant to the vault and WikiDocs

### The two vault clones are not at the same HEAD

- `[HERMES_HOST]` local vault HEAD: `866cf12bc3043931c0c84eb63c044a2540b59012`
- `[INFRA_HOST]` vault HEAD: `9ab9281f8e7adcd21c0f3d3669f6c61d3787076d`

This is live fact.

What is **not confirmed** in this doc:

- whether that divergence is intentional
- which host should be treated as the authoritative writer at this moment

### `dataset/` and `datasets/` both exist under `[WIKIDOCS_REPO_PATH]`

That dual-path presence is verified, but this draft does not claim the reason. It only records that the operational session history consistently used `datasets/documents`.

### Command Center note path is not cleanly established in this pass

The Brain notes clearly document Command Center / Daily Command Portal behavior, but in this pass I did not identify one single current canonical Markdown file path for the rendered Command Center dashboard note inside the vault. That needs a targeted follow-up if you want this repo doc to name it precisely.

### The public/private boundary is operationally clear, but governance rules are not fully written down

This draft can prove that the private vault and public WikiDocs are separate surfaces. It does **not** yet prove a formal written policy for:

- which content starts in Obsidian and then gets promoted to WikiDocs
- which content is meant to stay private forever
- who is allowed to edit which surface directly

Those rules may exist implicitly in practice, but I’m not going to invent them.

---

## 10. Minimal grounded commands and paths

### Verify the local vault repo on `[HERMES_HOST]`

```bash
cd ~/obsidian-vault
git remote get-url origin
git branch --show-current
git rev-parse HEAD
```

### Verify the vault repo on `[INFRA_HOST]`

```bash
ssh michaeld@[INFRA_HOST] 'cd ~/obsidian-vault && git remote get-url origin && git branch --show-current && git rev-parse HEAD'
```

### Verify the WikiDocs repo on `[INFRA_HOST]`

```bash
ssh michaeld@[INFRA_HOST] 'cd [WIKIDOCS_REPO_PATH] && git remote get-url origin && git rev-parse HEAD'
```

### Inspect live WikiDocs page files

```bash
ssh michaeld@[INFRA_HOST] 'find [WIKIDOCS_DATASET_PATH] -maxdepth 2 -type f | sort | sed -n "1,80p"'
```

### Check the specific WikiDocs commit you asked about

```bash
ssh michaeld@[INFRA_HOST] 'cd [WIKIDOCS_REPO_PATH] && git show --stat --oneline 8705f60'
```

---

## Grounding assessment for this document

### Fully grounded

These sections are backed directly by live files, live commands, Brain notes, or preserved session history:

- local vault path and git remote on `[HERMES_HOST]`
- vault path and git remote on `[INFRA_HOST]`
- the fact that the two vault clones are currently at different HEADs
- top-level private vault folder structure on `[HERMES_HOST]`
- the use of `Brain/` as the session-memory/journal surface
- the use of `JobSearch/` for job-search tracking flows
- the use of `Sync/` for automation-fed files such as `itnews-feed.json` and `job-matches.json`
- the correction that the public site at `[INFRA_HOST]:11122` is WikiDocs, not Wiki.js
- the WikiDocs repo path `[WIKIDOCS_REPO_PATH]` and remote `mdziegiel/wikidocs`
- the live WikiDocs document tree path `[WIKIDOCS_DATASET_PATH]`
- the existence and meaning of commit `8705f60` in the WikiDocs repo
- the July 28 Brain note about the homepage/sidebar regression fix workflow
- the June 22 mirror note showing WikiDocs content mirrored into `obsidian-vault/Wiki/`

### Best-effort reconstruction / needs review before public release

These points are plausible and consistent with the evidence, but are not fully proven end-to-end in this pass:

- whether `[HERMES_HOST]` or `[INFRA_HOST]` should be treated as the authoritative live writer for the vault right now
- the exact governance boundary for which content stays private in Obsidian versus which gets promoted to WikiDocs
- the exact current canonical Markdown file path for the rendered Command Center / Daily Command Portal note inside the private vault
- whether commit `8705f60` was the first, only, or final fix for all homepage/sidebar regressions, as opposed to one step in a longer sequence

If you want, I’ll stop here and let you review this before I touch `03-mcp-gateway.md`. That would be the sane way to do it.