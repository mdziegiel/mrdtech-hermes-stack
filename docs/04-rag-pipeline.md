# RAG Pipeline

This document covers the **private-vault retrieval pipeline** that supports Gilfoyle/Hermes memory lookups outside the live Hermes gateway itself. The current implementation lives across the `[INFRA_HOST]` host (Qdrant, vault-rag-pipeline repo, private vault copy) and the `[HERMES_HOST]` host (Hermes `SOUL.md` policy that requires retrieval before answering historical questions).

This draft is intentionally evidence-first. Where something is inferred from repo docs or session history instead of live runtime proof, it is labeled that way.

---

## 1. Current architecture at a glance

### Live components verified

**Qdrant stack on `[INFRA_HOST]`**
- Compose path: `~/stacks/qdrant/docker-compose.yml`
- Container name: `qdrant`
- Image pin: `qdrant/qdrant@sha256:0bd98fa7977f1e75694779359ca4e212822e5a71334e28421182f72f209d5286`
- Published ports:
  - `6333:6333`
  - `6334:6334`
- Telemetry explicitly disabled:
  - `QDRANT__TELEMETRY_DISABLED: "true"`
- Restart policy: `unless-stopped`

**Vault RAG code repo on `[INFRA_HOST]`**
- Repo path: `~/repos/vault-rag-pipeline`
- Git remote: `git@github.com:mdziegiel/vault-rag-pipeline.git`
- Live HEAD during this draft: `1fd78d9a8116a0600b99c50195c0c0007d3ef2b5`

**Collection state verified live from Qdrant API**
- URL queried: `http://127.0.0.1:6333/collections/mrdtech_vault`
- Status: `green`
- Optimizer status: `ok`
- Point count: `2191`
- Segment count: `2`

**Hermes retrieval rule on `[HERMES_HOST]`**
- File: `~/.hermes/SOUL.md`
- Retrieval rule lines found in current file:
  - `Before answering questions about past incidents, fixes, configurations, or homelab history, run vault_search.py with a concise query and ground the answer in results, citing source_path.`
  - `If NO_RESULTS, say so rather than guessing.`

### Architecture summary

The current design is split deliberately:

1. **private source corpus**: Obsidian vault on `[INFRA_HOST]`
2. **staging + redaction layer**: `vault-rag-pipeline/staging/` processed by `scripts/redact.py`
3. **embedding layer**: Ollama embedding requests using `nomic-embed-text`
4. **vector store**: Qdrant collection `mrdtech_vault`
5. **query layer**: `scripts/vault_search.py`
6. **agent policy layer**: `[HERMES_HOST]` `SOUL.md` forces retrieval before answering historical questions

That split is sane. You do not want raw private notes embedded before redaction, and you do not want the agent free-associating on history when a retriever exists.

---

## 2. Pipeline behavior from the live repo

### Repo README-stated flow
From `~/repos/vault-rag-pipeline/README.md`, the documented pipeline is:

1. copy vault text into a local `staging/` directory outside the vault
2. run `scripts/redact.py`
3. chunk Markdown/text with `scripts/ingest.py`
4. embed through Ollama using `nomic-embed-text`
5. upsert into Qdrant using deterministic UUID5 IDs
6. query with `scripts/vault_search.py` and cite `source_path`

That repo README is not just marketing copy; the code inspected in this draft matches it closely.

### Redaction code verified
File inspected on `[INFRA_HOST]`:
- `~/repos/vault-rag-pipeline/scripts/redact.py`

Verified behaviors from code:
- scans only text files with extensions:
  - `.md`
  - `.txt`
- identifies RFC1918 private IPv4s using regex and `is_private_ipv4()`
- replaces private IPs with host tokens using final octet preservation:
  - replacement format: `[HOST-<last_octet>]`
  - example shape: `[HERMES_HOST]:11434` → `[HOST-234]:11434`
- runs `gitleaks detect --no-git --source <staging_root>`
- replaces exact secret strings with rule-specific placeholders:
  - `[REDACTED-<RuleID>]`
- writes redaction reports under `reports/`
- does **not** modify the original vault
- only mutates staging files when invoked with `--apply`

### Ingest code verified
File inspected on `[INFRA_HOST]`:
- `~/repos/vault-rag-pipeline/scripts/ingest.py`

Verified behaviors from code:
- default collection name:
  - `mrdtech_vault`
- default Ollama URL logic:
  - `OLLAMA_URL` env or fallback `http://localhost:11434`
  - final embed endpoint becomes `/api/embed`
- default Qdrant URL:
  - `http://localhost:6333`
- creates Qdrant collection if missing
- vector size hard-coded to `768`
- distance metric:
  - `Cosine`
- chunk sizing logic:
  - target chars around `1800`
  - min `1200`
  - max `2000`
  - overlap `200`
- chunks built from Markdown sections, mostly headings level `#` through `###`
- embeddings requested in batches
- embed timeout per batch request:
  - `900` seconds
- deterministic point IDs:
  - UUID5 over `source_path:chunk_index`

That last part matters. Deterministic IDs mean reruns update the same logical chunk instead of spraying duplicates across the collection like some cursed confetti cannon.

### Search code verified
File inspected on `[INFRA_HOST]`:
- `~/repos/vault-rag-pipeline/scripts/vault_search.py`

Verified behaviors from code:
- default embedding model env:
  - `OLLAMA_MODEL` default `nomic-embed-text`
- default collection env:
  - `QDRANT_COLLECTION` default `mrdtech_vault`
- search endpoint:
  - `POST /collections/<collection>/points/search`
- query embedding happens through Ollama first
- printed fields include:
  - `score`
  - `source_path`
  - `heading`
  - text body
- default threshold:
  - `--min-score 0.50`
- if nothing clears threshold, script prints:
  - `NO_RESULTS`

That directly matches the standing `SOUL.md` policy.

---

## 3. Redaction model

### Confirmed current redaction approach
The current code and README agree on two mandatory redaction classes before embedding:

1. **RFC1918 private IPv4 addresses**
2. **secrets detected by gitleaks**

### RFC1918 handling
Confirmed from `redact.py`:
- private ranges covered:
  - `10.0.0.0/8`
  - `172.16.0.0/12`
  - `192.168.0.0/16`
- replacement preserves final octet and optional port
- example placeholder style:
  - `[HOST-237]:6333`

This is a decent compromise. Full deletion would reduce retrieval quality; full retention would be reckless. Preserving only the last octet is ugly but useful.

### gitleaks handling
Confirmed from `redact.py` and README:
- tool invocation is:
  - `gitleaks detect --no-git`
- it scans the working tree of the staging directory rather than Git history
- matched secrets are replaced by exact-string substitution
- replacement marker carries the gitleaks rule ID

This means the redaction pass is designed for **public export of staged corpus content**, not for auditing whether secrets ever existed in the repo’s full commit history. Those are different jobs. Pretending otherwise is how people end up leaking credentials with more confidence than necessary.

### Public-safe export evidence from session history
Brain note found:
- `~/obsidian-vault/Brain/2026-07-28_mrdtech-mcp-gateway-vault-rag-publish.md`

That note records a public-safe refresh after `vault-readonly` was added to the MCP gateway, including:
- internal Ollama/Qdrant endpoints replaced with `<OLLAMA_HOST>` and `<QDRANT_HOST>` placeholders
- gitleaks scans on working tree and fresh clone reported no leaks
- grep scans found no real internal IPs, private paths, raw secret files, or token material

That note is specifically about the public gateway repo update, not this stack-doc repo, but it confirms the same redaction/export pattern was already being used operationally.

---

## 4. Ingestion and incremental re-ingestion

### What is fully proven
From the code, the ingest path is **idempotent**:
- point IDs are deterministic UUID5 values from `source_path` plus `chunk_index`
- rerunning ingest updates existing logical points rather than minting random new IDs

That is enough to say the pipeline supports **safe repeat ingest** and **stable updates** when chunk boundaries do not change.

### What is best-effort rather than fully proven
The repo README and code strongly imply an incremental update model, but I did **not** replay a full delta-only ingest during this documentation pass.

So the safe wording is:
- **confirmed**: reruns are idempotent at point-ID level
- **not fully replayed in this pass**: minimal-delta behavior on a changed subset only

### Current collection scale
Two relevant counts exist right now, and they are not the same thing:

- Qdrant live collection point count: `2191`
- GBrain current local chunk count (separate system, separate doc): `705`

The RAG pipeline and GBrain are not the same corpus/index shape. Anyone conflating them deserves the debugging session they’re about to create.

---

## 5. Ollama embedding architecture

### Confirmed live design intent
From `ingest.py` and `vault_search.py`:
- embedding model default: `nomic-embed-text`
- embedding API path used by pipeline scripts: `/api/embed`
- collection vector width: `768`
- query-time embedding and ingest-time embedding both use the same model family assumptions

### What is not fully proven in this pass
I did **not** replay a fresh full embedding run during this draft window.

However, enough live evidence exists to say the pipeline is presently usable:
- Qdrant collection is present and healthy
- point count is non-zero and substantial (`2191`)
- Hermes `SOUL.md` still points to the retrieval script as an active rule

### Performance posture
The repo’s own lessons-learned section says CPU embedding is the bottleneck, and that matches the broader live environment:
- Ollama on `[HERMES_HOST]` is CPU-only
- embedding latency has been repeatedly treated as a real operational constraint
- memory note in current profile already warns that bulk embedding/import paths may require ~300 second timeouts

That is consistent with the design but was not freshly benchmarked in this doc pass.

---

## 6. Agent policy: mandatory retrieval before historical answers

### Current live rule
Verified in `~/.hermes/SOUL.md`:

> Before answering questions about past incidents, fixes, configurations, or homelab history, run `vault_search.py` with a concise query and ground the answer in results, citing `source_path`.

The same block also requires:
- if retrieval returns `NO_RESULTS`, say so instead of guessing
- live-state questions still require live checks instead of vault retrieval substitution

### What that means operationally
This policy splits knowledge questions into two buckets:

1. **historical / narrative / what-happened questions**
   - use `vault_search.py`
   - cite `source_path`
2. **current-state questions**
   - use live tools (`docker ps`, APIs, service checks, etc.)

That is the correct separation. Historical memory and live state are not interchangeable, no matter how badly people want one tool to do both.

### Direct test in this draft
I ran `vault_search.py` queries on `[INFRA_HOST]` with `--min-score 0` for:
- `qdrant ollama ingest redaction`
- `telegram gateway alert bot`
- `claude code codex delegation`
- `gbrain pglite timeout excluded-gbrain`

Those returned no visible results in this pass.

Safe interpretation:
- the script executed
- no printed hits surfaced for those broad queries under the current corpus / thresholds / query phrasing
- I am **not** claiming retrieval is broken solely from that
- I **am** claiming the `SOUL.md` rule exists and remains authoritative

This is the sort of ambiguity honest docs should preserve instead of airbrushing over it.

---

## 7. Known paths and files

### `[INFRA_HOST]` paths
- Qdrant compose:
  - `~/stacks/qdrant/docker-compose.yml`
- Vault RAG repo:
  - `~/repos/vault-rag-pipeline`
- Redactor:
  - `~/repos/vault-rag-pipeline/scripts/redact.py`
- Ingest script:
  - `~/repos/vault-rag-pipeline/scripts/ingest.py`
- Search script:
  - `~/repos/vault-rag-pipeline/scripts/vault_search.py`

### `[HERMES_HOST]` path
- Hermes soul/rule file:
  - `~/.hermes/SOUL.md`

---

## 8. Open issues and review notes

### Confirmed open issues
- The retrieval script produced no visible hits for the broad ad hoc queries used during this draft.
- No fresh full ingest was replayed during this pass.
- No fresh redaction apply-run was replayed during this pass.
- The repo README describes the intended env contract (`PIPELINE_ROOT`, `VAULT_SOURCE_DIR`, `OLLAMA_URL`, `QDRANT_URL`, etc.), but I did not print the live `.env` values because that would be a great way to sabotage your own redaction effort.

### Best-effort / needs review
- Whether the current staging tree and reports tree on `[INFRA_HOST]` exactly match the README-described layout right now was not fully enumerated during this pass.
- Whether the no-result `vault_search.py` runs reflect sparse matches, query phrasing, or a temporary retrieval-quality issue needs a targeted retrieval test, not assumptions.
- Whether all current point counts in Qdrant map exactly to the latest vault HEAD versus a prior ingest snapshot was not replay-verified in this pass.

---

## 9. Fully grounded vs. best-effort summary

### Fully grounded
- Qdrant compose path, image pin, published ports, telemetry disable, restart policy
- vault-rag-pipeline repo path, remote, and HEAD
- live Qdrant collection status (`green`) and point count (`2191`)
- current `SOUL.md` retrieval rule requiring `vault_search.py`
- `redact.py` private-IP and gitleaks substitution behavior
- `ingest.py` chunk sizing, vector width (`768`), deterministic UUID5 upsert design, embed timeout (`900s`)
- `vault_search.py` output format, default min score, and `NO_RESULTS` behavior
- historical note that public-safe redaction/export was used for the vault-readonly gateway publish

### Best-effort / needs review
- fresh proof that the current staged corpus was re-redacted and re-ingested from scratch in the exact current state
- exact live `.env` runtime values for `OLLAMA_URL`, `QDRANT_URL`, and `PIPELINE_ROOT`
- a replayed proof of incremental-delta ingest on only changed files
- root cause of the no-hit ad hoc `vault_search.py` queries used during this draft
