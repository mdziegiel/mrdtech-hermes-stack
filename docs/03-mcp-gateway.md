# MCP Gateway

This document covers the **read-only multi-server Docker MCP Gateway** currently running on the MRDTech `[INFRA_HOST]` host and how Hermes/Gilfoyle on `[HERMES_HOST]` reaches it.

Everything here is grounded in one or more of:

- live inspection of the gateway stack on `[INFRA_HOST]`
- live inspection of Hermes MCP client config on `[HERMES_HOST]`
- retained session history from the July 2026 gateway build-out
- local Brain notes summarizing the later publish/update work
- session-derived operating notes for post-deploy auth drift and hardening

Where something is only reconstructable from phase history rather than directly re-proven in this pass, it is labeled that way.

---

## 1. Current live location and transport path

### Gateway host

Verified live on `[INFRA_HOST]`:

- stack root: `~/mcp-gateway-portainer-ro`
- main compose file: `~/mcp-gateway-portainer-ro/docker-compose.yml`
- gateway catalog: `~/mcp-gateway-portainer-ro/gateway/catalog.yaml`

Relevant secret/config files present under that stack root:

- `.env`
- `secret.env`
- `secrets/mcp_secret.env`
- `secrets/github.env`
- `secrets/pbs.env`
- `secrets/proxmox.env`
- `secrets/portainer_api_token`

All of the env/secret files I checked are mode `600` and owned by `michaeld:michaeld` on `[INFRA_HOST]`.

### Live gateway container

Verified live via `docker inspect` and `docker ps` on `[INFRA_HOST]`:

- container: `docker-mcp-gateway-portainer-ro`
- image: `docker/mcp-gateway:latest`
- restart policy: `unless-stopped`
- root filesystem: `ReadOnly=true`
- published port: `127.0.0.1:8811->8811/tcp`

The important part is not the image name. The important part is that the gateway only binds to **loopback on `[INFRA_HOST]`**.

### Hermes-side access path

Verified live on `[HERMES_HOST]`:

- Hermes MCP client entries point to: `http://127.0.0.1:18811/mcp`
- local listener: `127.0.0.1:18811`
- owning process:

```text
/usr/bin/ssh -N -T \
  -o BatchMode=yes \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o StrictHostKeyChecking=accept-new \
  -L 127.0.0.1:18811:127.0.0.1:8811 \
  user@[INFRA_HOST]
```

That proves the current path is:

```text
Hermes on [HERMES_HOST]
  -> local MCP URL http://127.0.0.1:18811/mcp
  -> SSH local port forward on [HERMES_HOST]
  -> 127.0.0.1:8811 on [INFRA_HOST]
  -> docker-mcp-gateway-portainer-ro
  -> selected backend MCP servers on the internal Docker bridge network
```

### Exposure conclusion

Based on live state, the gateway is **not directly exposed on the LAN or WAN**.

What is directly proven:

- `[INFRA_HOST]` publishes the gateway only on `127.0.0.1:8811`
- `[HERMES_HOST]` consumes it through an SSH local forward on `127.0.0.1:18811`
- Hermes MCP config points to that local forwarded endpoint, not to `[INFRA_HOST]` directly

So the current deployment matches the claim:

- **SSH-tunnel-only access path for Hermes/Gilfoyle**
- **no direct gateway network exposure**

---

## 2. Current server inventory exposed through the gateway

The gateway currently registers **six read-only backends**.

This is grounded by:

- `docker-compose.yml` `--servers=` and `--tools=` arguments
- `gateway/catalog.yaml`
- Hermes `mcp_servers:` config on `[HERMES_HOST]`
- a July 28 Brain note confirming the six-server state and 202-tool total

### 2.1 Portainer (`portainer_ro`)

Current Hermes MCP entry:

- name: `portainer_ro`
- local URL: `http://127.0.0.1:18811/mcp`

Allowed tools in Hermes config:

- `list_stacks`
- `get_stack_status`
- `list_containers`

Live backend container:

- `mcp-portainer-readonly`
- image: `mrdtech/portainer-readonly-mcp:phase1`
- container root fs: `ReadOnly=true`
- container user: `1000:1000`

Compose-level constraints:

- `PORTAINER_URL=https://[INFRA_HOST]:9005`
- `PORTAINER_ALLOWED_ENDPOINTS=2,3`
- token supplied from file: `/run/secrets/portainer_api_token`
- secret bind mount is read-only
- bridge network only; no published ports

Interpretation grounded by config:

- this server is intentionally scoped to Portainer endpoints `2` and `3`
- it is not a generic Portainer admin surface
- it exposes only the three minimal read-only tools listed above

### 2.2 GitHub (`github_ro`)

Current Hermes MCP entry:

- name: `github_ro`
- local URL: `http://127.0.0.1:18811/mcp`

Live backend container:

- `mcp-github-readonly`
- image: `ghcr.io/github/github-mcp-server:latest`
- `ReadOnly=true`
- restart policy: `unless-stopped`

Compose-level constraints:

- `env_file: ./secrets/github.env`
- `GITHUB_READ_ONLY='1'`
- `GITHUB_TOOLSETS=context,repos,issues,pull_requests,actions`
- secret file mounted read-only at `/run/secrets/github_pat`
- no published ports

Hermes-side allowed tools include read-only repo / PR / issues / actions surfaces such as:

- `get_me`
- `get_file_contents`
- `list_commits`
- `list_pull_requests`
- `search_repositories`
- `actions_get`
- `actions_list`

No GitHub write tool is intentionally listed in the Hermes config shown in this pass.

### 2.3 Filesystem (`fs_ro`)

Current Hermes MCP entry:

- name: `fs_ro`
- local URL: `http://127.0.0.1:18811/mcp`

Live backend container:

- `mcp-filesystem-readonly`
- image: `mrdtech/filesystem-readonly-mcp:phase1`
- `ReadOnly=true`

Live mount verified:

- host path: `/data/compose`
- container path: `/projects`
- bind is read-only (`RW=false`)

Allowed tools in Hermes config:

- `read_text_file`
- `read_media_file`
- `read_multiple_files`
- `list_directory`
- `list_directory_with_sizes`
- `directory_tree`
- `search_files`
- `get_file_info`
- `list_allowed_directories`

Grounded conclusion:

- filesystem visibility is intentionally narrow: `/data/compose` only
- this is not a general host filesystem bridge
- the current mount posture is OS-level read-only in addition to tool allowlisting

### 2.4 Proxmox (`proxmox_ro`)

Current Hermes MCP entry:

- name: `proxmox_ro`
- local URL: `http://127.0.0.1:18811/mcp`

Live backend container:

- `mcp-proxmox-readonly`
- image: `mcp/proxmox:latest`
- `ReadOnly=true`

Catalog description states:

- it uses a `PVEAuditor` token
- gateway hard allowlisting is used
- `proxmox_api_raw` is excluded

Hermes config and compose/cached catalog show a very large read/list/get surface including examples such as:

- `list_vms`
- `get_vm_status`
- `get_vm_config`
- `list_nodes`
- `get_cluster_status`
- `list_storage`
- `list_backup_jobs`
- `get_node_report`
- `get_acl`
- `list_ha_resources`

The current system prompt tool catalog in this session also exposes only the `proxmox_ro` family of read-oriented tools, consistent with that configuration.

Important caveat:

- this backend is **third-party** (`mcp/proxmox:latest` / GethosTheWalrus lineage from the earlier discovery work), so the read-only guarantee here depends on **token scope + tool allowlisting**, not on native server virtue.

### 2.5 PBS (`pbs_ro`)

Current Hermes MCP entry:

- name: `pbs_ro`
- local URL: `http://127.0.0.1:18811/mcp`

Live backend container:

- `mcp-pbs-readonly`
- image: `mrdtech/pbs-readonly-mcp:phase1`
- `ReadOnly=true`

Compose-level notes:

- built from local context `./pbs`
- secrets loaded from `./secrets/pbs.env`
- no published ports

Allowed tools in Hermes config:

- `list_datastores`
- `get_datastore_status`
- `list_snapshots`
- `get_snapshot_verification_status`
- `get_gc_status`
- `list_gc_tasks`
- `get_task_log`

Catalog description explicitly says:

- `GET-only API wrapper`
- `no GC/verify/prune/restore/write tools`

### 2.6 Vault RAG (`vault_ro`)

Current Hermes MCP entry:

- name: `vault_ro`
- local URL: `http://127.0.0.1:18811/mcp`

Live backend container:

- `mcp-vault-readonly`
- image: `mrdtech/vault-readonly-mcp:phase1`
- `ReadOnly=true`

Allowed tools in Hermes config:

- `search_vault`
- `get_document`

Catalog description says this backend is:

- a vault RAG read-only server
- fixed to Ollama embed + Qdrant search/scroll endpoints only
- with **no write tools**

A July 28 Brain note confirms that `vault-readonly` was the **sixth backend** added to the gateway and that the resulting live tool total was **202**:

- Portainer 3
- GitHub 28
- Filesystem 9
- Proxmox 153
- PBS 7
- Vault 2

Those numbers match the live shape of the gateway better than hand-counting by eye like a lunatic.

---

## 3. Read-only vs read-write scoping

### Does the distinction exist?

Yes. Very much yes.

Everything currently documented in the live stack is explicitly **read-only first-pass** or **read-only only**.

That distinction is grounded by multiple layers:

1. **Gateway tool allowlists** in `docker-compose.yml`
2. **Catalog descriptions** calling out read-only posture
3. **Hermes-side include lists** in `[HERMES_HOST]` `config.yaml`
4. **Container hardening** (`read_only: true`, `cap_drop: [ALL]`, `no-new-privileges:true`)
5. **Scope-reduced credentials** or secret-file isolation per backend

### Per-server read-only enforcement model

| Server | Read-only enforcement model | Notes |
|---|---|---|
| Portainer | custom minimal wrapper + endpoint allowlist + only 3 tools | strongest intentional minimization |
| GitHub | official server + `GITHUB_READ_ONLY=1` + toolset limits + Hermes include list | still depends on env/config discipline |
| Filesystem | read-only bind mount of `/data/compose` + read-only tool allowlist | narrow path exposure |
| Proxmox | restricted API token (`PVEAuditor` per catalog) + hard allowlist + explicit raw API exclusion | highest-risk backend because upstream server is broad |
| PBS | custom GET-only wrapper + explicit no-write/no-prune/no-restore tool surface | intentionally narrow |
| Vault RAG | custom search/get-only server | no mutation surface listed |

### What I can verify directly vs what I am not claiming

Directly verified in this pass:

- the allowlisted tools are present in config
- the containers are hardened and running
- the gateway/caller path is loopback + SSH tunnel
- the custom/backend descriptions in catalog explicitly state read-only intent

Not directly re-tested in this pass:

- attempting a forbidden tool call for each backend right now and showing an `unknown tool` style failure

That kind of negative test was described in prior phase work, but I did not rerun every one of those live during this documentation pass. So I am not pretending I just proved every denial path again.

---

## 4. Phased hardening and build history

The current stack name still says `portainer-ro`, but the live stack has expanded far beyond the initial Portainer-only phase. Naming drift is one of the recurring themes here.

### Phase 0 — discovery / scoping (July 25, 2026)

Retained session history shows a `Docker MCP Gateway, Phase 0 Discovery` task.

Grounded outcomes from that phase, as reflected in retained history and current deployment:

- Docker CLI / Docker MCP plugin was **not available on Hermes `[HERMES_HOST]`**, so the gateway was not hosted locally on Hermes.
- The design direction became: host the gateway on `[INFRA_HOST]`, and let Hermes reach it over a constrained connection path.
- Early scoping was explicitly read-only first.
- Credential isolation was a first-class requirement: gateway-owned secret storage on `[INFRA_HOST]`, not Hermes `.env` for backend credentials.

### Phase 1 — Portainer read-only gateway

Retained session history shows a `Docker MCP Gateway, Phase 1 Build` task approved for:

- a dedicated Portainer service account / token
- deployment on `[INFRA_HOST]`, not Hermes
- no write/restart/delete tools
- read-only rootfs, dropped caps, no-new-privileges, no exposed ports

The current live Portainer backend matches that design:

- dedicated `mcp-portainer-readonly` container
- only 3 tools
- token from a gateway-owned secret file on `[INFRA_HOST]`
- endpoint restriction `2,3`

### Phase 2 — additional read-only backends on the same gateway

Subsequent retained history around July 25–27 and the current compose file show the gateway widened to additional backends under the same pattern:

- GitHub read-only
- Filesystem read-only
- Proxmox read-only
- PBS read-only
- Vault RAG read-only

The current live compose file confirms all six run behind the same gateway container and the same tunnel path.

### What appears completed now

Grounded from current live state:

- gateway hosted on `[INFRA_HOST]`
- no direct gateway exposure beyond loopback
- Hermes reaches it through SSH local forward only
- six backends registered and running
- per-backend allowlisted tool surfaces in compose and Hermes config
- secret files on `[INFRA_HOST]` mode `600`
- hardened container posture across all backend services checked in this pass:
  - read-only rootfs
  - `cap_drop: ALL`
  - `no-new-privileges:true`
  - no per-backend published ports

### What still looks pending or at least not fully resolved/documented

These are the important caveats, because “read-only” does not mean “finished forever.”

1. **Naming drift**
   - stack root is still `mcp-gateway-portainer-ro`
   - live system is now six backends, not Portainer-only
   - that is documentation/config naming debt

2. **Negative-write verification is not continuously enforced in this doc pass**
   - prior build work reportedly tested blocked write paths
   - this pass verified config posture, not every current blocked-call path live

3. **Secret rotation procedures remain an operational risk area**
   - see GitHub PAT drift incident below

4. **Backend breadth differs sharply by server**
   - Portainer/PBS/Vault are intentionally narrow
   - Proxmox remains broad even when read-oriented, so its risk floor is still higher

---

## 5. Incidents, drift, and quirks relevant to current state

### 5.1 GitHub PAT rotation / auth drift incident

This is the most concrete documented gateway-ops incident in the retained references.

Session-derived operating note `github-pat-rotation-and-gateway-auth-hardening.md` records the following class of failure:

- GitHub PAT on disk was correct
- direct upstream GitHub API auth returned `HTTP 200`
- but `mcp__github_ro__get_me` still returned `401 Bad credentials`
- the backend container still had the old env loaded

The fix was **not** `docker compose restart`.

The working fix was:

```bash
docker compose up -d --force-recreate --no-deps mcp-github-readonly
```

Why it matters for the doc:

- the gateway stack uses `env_file` / secret-file driven config
- compose restart can leave stale container env in place
- auth rotations must be treated as **recreate**, not merely **restart**, when the backend consumes file-backed env at container creation time

That is not theoretical. It already bit the deployment once.

### 5.2 Auth hardening drift between docs and live state

The same operating reference notes that docs had to be brought back in line with live posture after hardening.

Grounded operational lessons recorded there:

- unauthenticated mode was removed from live deployment
- Hermes uses Bearer auth on all six MCP entries
- public/internal docs had to be updated to reflect the enforced token posture

This matters because the current `[HERMES_HOST]` Hermes config clearly does include:

```yaml
headers:
  Authorization: Bearer ...
```

for all six MCP server entries.

So if any old docs or examples imply “open local-only gateway without auth,” they are stale relative to the live system.

### 5.3 Stack-name drift

The stack path and container naming still center `portainer-ro`, but the stack now fronts:

- Portainer
- GitHub
- Filesystem
- Proxmox
- PBS
- Vault RAG

That is not a runtime failure, but it is real configuration/documentation drift and should be treated as such.

### 5.4 WikiDocs/doc count drift corrected later

A July 28 Brain note records a later publish/update pass after the sixth backend was added.

Grounded facts from that note:

- the gateway had grown to six backends
- the real total was **202 tools**, not `207`
- public repo and WikiDocs content had to be corrected to reflect the real count and the new `vault-readonly` backend

So another drift class here is simply **inventory drift**: the gateway changed faster than the docs.

---

## 6. Current hardening posture visible on disk/live state

The live deployment shows a consistent hardening baseline across the gateway and backends.

### Gateway container hardening

Verified in compose and/or inspect:

- `read_only: true`
- `cap_drop: [ALL]`
- `security_opt: [no-new-privileges:true]`
- tmpfs scratch locations instead of normal writable rootfs
- bind of Docker socket is read-only
- catalog bind is read-only
- auth-token secret bind is read-only
- published only on `127.0.0.1:8811`

### Backend container hardening

Every backend shown in the current compose file uses the same general pattern:

- `read_only: true`
- `cap_drop: [ALL]`
- `no-new-privileges:true`
- no published ports
- attached only to internal bridge network `mcp_portainer_ro`
- stdio transport through the docker-mcp bridge

### Secret storage posture

Verified live on `[INFRA_HOST]`:

- secret/config files live under the gateway stack root
- secret files checked in this pass are mode `600`
- Portainer token file exists separately as:
  - `~/mcp-gateway-portainer-ro/secrets/portainer_api_token`
  - mode `600`

What this does **not** prove:

- that all credential rotation runbooks are perfectly documented
- that every secret file has never drifted in contents historically

What it does prove:

- the current storage pattern keeps backend credentials on `[INFRA_HOST]` under the gateway-owned stack area rather than the Hermes host-level public-facing docs tree

---

## 7. Relationship to Hermes tool exposure

Hermes `[HERMES_HOST]` does not point each MCP family to a separate remote host/port.

Instead, all six configured MCP server entries use the same local forwarded URL:

```text
http://127.0.0.1:18811/mcp
```

and each entry narrows scope using a per-server `tools.include` list.

Current configured entries are:

- `portainer_ro`
- `github_ro`
- `fs_ro`
- `proxmox_ro`
- `pbs_ro`
- `vault_ro`

That means the exposure model is:

- one local forwarded gateway endpoint
- multiple logical MCP server registrations in Hermes
- per-registration tool filtering on the Hermes side
- per-backend tool filtering on the gateway side

This layered filtering is good. It is also exactly the sort of thing that becomes dangerous if one side drifts and the other side is assumed to be authoritative without re-checking.

---

## 8. Minimal grounded commands and paths

### Verify the gateway is loopback-only on `[INFRA_HOST]`

```bash
ssh user@[INFRA_HOST] \
  'docker inspect docker-mcp-gateway-portainer-ro --format "Ports={{json .NetworkSettings.Ports}}"'
```

Expected current shape:

```text
{"8811/tcp":[{"HostIp":"127.0.0.1","HostPort":"8811"}]}
```

### Verify the local SSH tunnel on `[HERMES_HOST]`

```bash
ss -ltnp | grep ':18811 '
ps -ef | grep '18811:127.0.0.1:8811'
```

### Verify the Hermes MCP client entries

```bash
sed -n '808,1080p' ~/.hermes/config.yaml
```

### Inspect the live gateway stack files on `[INFRA_HOST]`

```bash
ssh user@[INFRA_HOST] \
  'sed -n "1,520p" ~/mcp-gateway-portainer-ro/docker-compose.yml'

ssh user@[INFRA_HOST] \
  'sed -n "1,320p" ~/mcp-gateway-portainer-ro/gateway/catalog.yaml'
```

### Verify current secret-file ownership/modes

```bash
ssh user@[INFRA_HOST] \
  'stat -c "%n %a %U:%G %s bytes %y" ~/mcp-gateway-portainer-ro/secrets/* ~/mcp-gateway-portainer-ro/.env ~/mcp-gateway-portainer-ro/secret.env'
```

---

## Grounding assessment for this document

### Fully grounded

These sections are backed directly by live config, live service state, or retained session-derived notes/references:

- gateway stack path on `[INFRA_HOST]`
- live gateway container name, image, restart policy, read-only rootfs, and loopback port bind
- Hermes-side MCP endpoint URL `127.0.0.1:18811/mcp`
- live SSH tunnel process forwarding `18811 -> 127.0.0.1:8811` on `[INFRA_HOST]`
- the conclusion that the gateway is currently reached by SSH tunnel only and not directly exposed on the network
- current six-server inventory: Portainer, GitHub, Filesystem, Proxmox, PBS, Vault RAG
- current per-server tool scopes shown in `[HERMES_HOST]` Hermes config and `[INFRA_HOST]` gateway compose/catalog
- current read-only hardening posture (`read_only`, `cap_drop ALL`, `no-new-privileges`, no per-backend published ports)
- secret/config files living under the `[INFRA_HOST]` gateway stack root with mode `600` for the files checked in this pass
- Portainer endpoint restriction `2,3`
- Filesystem restriction to `/data/compose:ro`
- Proxmox catalog statement that `proxmox_api_raw` is excluded and `PVEAuditor` token is used
- July 28 Brain note that the sixth backend was `vault-readonly` and the live total was 202 tools
- GitHub PAT rotation incident pattern and the `force-recreate --no-deps` fix from the retained operating reference
- auth-hardening drift lesson that live deployment requires Bearer auth on the six Hermes MCP entries

### Best-effort reconstruction / needs review before public release

These points are consistent with the evidence but were not fully re-proven in this pass:

- the exact phase boundaries/names beyond the retained session prompts and current resulting stack
- the exact deployment date for PBS relative to GitHub/Filesystem/Proxmox, since I did not replay every intermediate build transcript end-to-end in this pass
- end-to-end live rejection of every single forbidden write tool right now; this draft documents the configuration posture and prior phase intent, not a fresh full denial-test matrix
- whether the current stack name drift (`portainer-ro`) is deliberate or simply uncleaned technical debt

If you want the short version: the current gateway is real, loopback-bound on `[INFRA_HOST]`, consumed only through an SSH local forward on `[HERMES_HOST]`, and built as a layered read-only surface. The weakest part is not the tunnel. It is the usual thing: config drift and people assuming a restart is the same as a recreate when secrets change.