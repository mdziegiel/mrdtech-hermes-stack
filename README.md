# MRDTech Hermes Stack

Local documentation scaffold for the MRDTech Hermes / Gilfoyle control stack. This repo is being drafted locally first, then intended for redaction review before any public push.

## Scope

This stack currently spans:

- **Hermes Agent** running on VM108 (`[HERMES_HOST]`) as the primary agent runtime
- **Gilfoyle persona** layered through `~/.hermes/SOUL.md`
- **Hermes gateway** for messaging / Telegram delivery
- **Obsidian vault integration** and WikiDocs/Wiki-like documentation workflows
- **MCP / plugin integrations** for infrastructure read/write surfaces
- **RAG pipeline** using Ollama embeddings and Qdrant-backed retrieval patterns
- **Dual-model delegation** patterns involving Claude Code / Codex-style workflows
- **GBrain** as a separate local knowledge / embedding workflow on the infrastructure host
- **Local skill catalog** under `~/.hermes/skills/` plus plugin-provided skill sets

## Grounded environment facts

Known from local session history and on-host state:

- Hermes host: **Ubuntu VM108** at `[HERMES_HOST]`
- Active Hermes profile: `default`
- Skill library lives at `~/.hermes/skills/`
- Plugins currently present include **Superpowers** and local audit logging
- OHM standalone skills were manually installed into the local skill library; no OMH plugin path is currently documented as installed
- `defuddle` CLI is installed separately and the local `defuddle` skill is only the wrapper/invocation guidance

## Architecture overview

```mermaid
flowchart TD
    U[Michael] --> D[Hermes Desktop / Chat Surface]
    D --> H[Hermes Agent on VM108
[HERMES_HOST]]
    H --> P[Gilfoyle Persona
SOUL.md]
    H --> S[Skill Library
~/.hermes/skills]
    H --> G[Gateway / Telegram Delivery]
    H --> O[Obsidian Vault + WikiDocs Workflows]
    H --> M[MCP / Plugins / Infra Integrations]
    H --> R[RAG Pipeline
Ollama + Qdrant]
    H --> X[Dual-Model Delegation
Claude Code / Codex style]
    H --> B[GBrain on [INFRA_HOST]]
```

## Documentation map

- [01 Hermes setup](docs/01-hermes-setup.md)
- [02 Obsidian vault](docs/02-obsidian-vault.md)
- [03 MCP gateway](docs/03-mcp-gateway.md)
- [04 RAG pipeline](docs/04-rag-pipeline.md)
- [05 Telegram](docs/05-telegram.md)
- [06 Dual-model delegation](docs/06-dual-model-delegation.md)
- [07 GBrain](docs/07-gbrain.md)
- [Skills catalog](docs/skills/README.md)

## Known gaps to fill

These items need fuller source-backed writeups before publication:

- Exact MCP server inventory and phased hardening chronology
- Exact Obsidian ↔ WikiDocs boundary and current source-of-truth rules
- Full RAG redaction pipeline sequence and publication-safe audit steps
- Exact dual-model delegation architecture and when each model/provider is used
- GBrain timeline beyond the local pinned install / provider-fix work

## Publication rule

Before any GitHub push, this repo should be reviewed with the same redaction gate used for the RAG pipeline work:

- `gitleaks` scan
- RFC1918 / internal-topology audit
- credential / token / host-name review
- manual proofread for environment leakage
