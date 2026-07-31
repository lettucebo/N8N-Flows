# Project Guidelines

## Overview

n8n workflow automation repository. All workflows are stored as JSON files under `flows/` and managed via the n8n API + Git. Read [Talking to the instance](#talking-to-the-instance) first — which tools actually work is the thing most likely to waste your time here.

- **Instance**: self-hosted n8n at `https://n8n.yu.money`, running as one service on a shared Azure VM — read [Hosting & Operations](#hosting--operations) before running anything on the host or reasoning about why a workflow behaves differently there than the JSON suggests. MCP servers are declared in `.mcp.json` at the repo root and are loaded by Copilot CLI, but see [Talking to the instance](#talking-to-the-instance) for which of their tools actually authenticate. Version verified on the box on 2026-07-29 is **2.32.5**; the `1.93.0` in `flows/RaindropKnowledgeManagement/README.md` is from 2025-05 and stale. Re-confirm against the instance rather than trusting any number in a doc — including this one.
- **Primary Timezone**: Asia/Taipei
- **Language**: workflow names in English; node names in English for new work (legacy flows use zh-TW — see [Node Naming](#node-naming)); AI prompts, Slack messages, and `console.log` output in Traditional Chinese (zh-TW)
- **No build system**: no `package.json`, no CI, no test runner. The JSON files are data, not code — "correctness" means the n8n instance accepts and runs them. See [Local Checks](#local-checks).

## Architecture

```
flows/
  {WorkflowName}/
    {Workflow_Name}.json    # n8n workflow export (single JSON file per workflow)
    README.md               # optional English documentation (only 2 of 4 flows have it)
    README.zh-tw.md         # optional zh-TW documentation
.github/
  skills/                   # Copilot skills for n8n development
```

Each workflow is a self-contained JSON file. No shared code between workflows — a fix in one flow's Code node is not inherited by any other, so search all four when a pattern is wrong.

### Naming contract

- Folder: `PascalCase` with no separators (`SlackToGoogleCalendar`).
- File: the workflow's `name` field with spaces replaced by `_` (`Slack to Google Calendar AI Assistant` → `Slack_to_Google_Calendar_AI_Assistant.json`).
- If you rename a workflow in n8n, rename the local file and folder to match.

### The local JSON is an n8n export, not hand-written config

Every `flows/**/*.json` has exactly these top-level keys, in this order:

`name`, `nodes`, `pinData`, `connections`, `active`, `settings`, `versionId`, `meta`, `id`, `tags`

- `id` and `versionId` bind the file to the live workflow on the n8n instance. Never invent or hand-edit them — they come from the MCP read after a cloud update.
- `meta.instanceId` is the same for all workflows in this repo. When rewriting a file, copy the value already in that file rather than typing one.
- `connections` is keyed by **node name** (not node id), so renaming a node means rewriting every reference to it in `connections`, in `$('Node Name')` calls inside Code nodes, and in expressions.

### Reference implementation

`flows/SlackToGoogleCalendar/Slack_to_Google_Calendar_AI_Assistant.json` is the most current workflow and the one to copy patterns from: English node names, IF v2.3, Switch v3.2, raw `httpRequest` for all Slack calls, `onError` routing into a single error sink, per-item delivery verification, the reaction lifecycle, and the full `settings` block. The other three flows predate all of it — see [Known drift](#known-drift). Its [Cross-Node Contracts](#cross-node-contracts) are the part that most needs preserving.

### Workflow inventory

Snapshot — re-read the JSON or query MCP before relying on any count or version here.

| Folder | Trigger → Sink | Notes |
|---|---|---|
| `SlackToGoogleCalendar` | Slack trigger → Azure OpenAI → Google Calendar → Slack | **37 nodes / 54 connections**; the reference flow. Thread context, vision, reaction lifecycle, multi-event fan-out, anchor status sync. Every Slack call is an `httpRequest` node — it does not use the Slack node at all |
| `RaindropKnowledgeManagement` | Schedule (30 min) → Raindrop → Azure AI → Notion | 11 nodes; IF/Merge branch for "AI vs basic analysis". The schedule interval and the dedupe window in `篩選新項目` are coupled: the window is `interval + 5` minutes. Change one, change the other. Also `perpage: 10` on the Raindrop request. |
| `JenDailyScheduleAlert` | Schedule → Google Sheets → Slack | 4 nodes, linear; does its own Taipei/Calgary offset + DST math in the Code node |
| `DailyCurrencyExchangeRateAlert` | Schedule → HTTP → Slack | 4 nodes, linear; the only flow that authenticates via an n8n env var (`{{$env.EXCHANGE_API_KEY}}`) instead of a credential. **Currently broken** — `$env` access is denied at runtime, so every scheduled run has failed since 2026-07-16; see [Environment facts](#environment-facts-that-constrain-workflow-authoring) |

**The instance holds more workflows than this repo does.** As of 2026-07-31 the live database has **7** workflows and **12** credentials. `Github flow backup`, `Spotify Weekly Backup Schedule` and `Slack to Google Calendar AI Assistant (v2 TEST)` (`RJiCNKlVQ4EhVyMS`, an inactive copy kept as a rollback reference) exist only on the instance and have never been exported here. When auditing "all workflows", query the API; `flows/` is not the full set.

**`active` in the exported JSON is not trustworthy.** All four local files say `"active": true`, but only **two** workflows are actually active on the instance: `Daily Currency Exchange Rate Alert` and `Slack to Google Calendar AI Assistant`. The flag is a snapshot from whenever the file was last synced. Check the instance, not the file.

## Hosting & Operations

This instance is not managed infrastructure. It is a single Azure VM running one Docker Compose project that also hosts unrelated services. Everything below was verified on the host on 2026-07-29 while recovering from a production outage. The failure mode in [The 2026-07-29 outage](#the-2026-07-29-outage--do-not-repeat-this) is trivially easy to repeat.

### Where it runs

| | |
|---|---|
| Host | Single Azure VM, Ubuntu 24.04, accessed over SSH. **Hostname and login user are intentionally omitted — this repo is public.** Ask the repo owner, or read them from your SSH config. |
| Deployment repo | `github.com/lettucebo/CommonVM` — **public**; never commit secrets or DB dumps there |
| Compose file | `~/commonVM/merged-services/src/docker-compose.yml` — host paths below are written relative to the deploy user's home for the same reason |
| Compose project | `src` (now pinned via top-level `name: src`) |
| Ingress | Caddy, `reverse_proxy n8n:5678`, terminating TLS for `n8n.yu.money` and the other services on the box |
| Networks | `src_web` (Caddy ↔ n8n), `src_backend` (n8n ↔ postgres) |

Containers in the project: `src-n8n-1`, `src-n8n-db-1`, `src-caddy-1`, `src-codimd-1`, `src-codimd-db-1`, `rustdesk-hbbs`, `rustdesk-hbbr`. n8n shares the stack with CodiMD and a RustDesk server, so breaking the compose project is never "just an n8n outage".

**All persistent state is bind-mounted from `/mnt/data`** (a separate Azure data disk), not from Docker named volumes:

| Path | Contents |
|---|---|
| `/mnt/data/n8n/db` | PostgreSQL PGDATA — workflows, credentials, executions |
| `/mnt/data/n8n/data` | n8n home (`/home/node/.n8n`) — `config` (encryption key), `binaryData/`, `nodes/` |
| `/mnt/data/backup` | Incident backups, mode 600, directory 700 |

That bind-mount design is the only reason the outage below cost no data: the compose *definition* was replaced, but `/mnt/data` was never touched.

### The 2026-07-29 outage — do not repeat this

**Never run `docker compose` from `~/commonVM/n8n-azure-vm-starter/src/`.** That directory is an Azure deployment *template*, not the running configuration.

Compose derives its project name from the directory basename. Production and the starter template both lived in a directory named `src`, so both resolved to project `src`, and both would name their n8n container `src-n8n-1`. Running `docker compose up -d --pull always` in the starter directory therefore **silently replaced the production n8n container** with the template's definition: pointed at a brand-new empty postgres, mounted an empty named volume as `/home/node/.n8n` (so n8n auto-generated a *fresh* encryption key), and joined only `src_default` instead of `src_web` — Caddy could no longer resolve `n8n` and served 502 for 25 minutes.

Fixed by declaring explicit project names: production carries `name: src`, the template carries `name: n8n-starter`.

Health check for recurrence — `docker compose ls -a` must show exactly one config file for project `src`:

```
NAME   STATUS       CONFIG FILES
src    running(7)   /home/<user>/commonVM/merged-services/src/docker-compose.yml
```

Two comma-separated paths on that line means a foreign stack has merged into production again.

### Updating the n8n image safely

The recovery upgrade ran 2.14.2 → 2.32.5 and applied **72 irreversible schema migrations**. n8n does not support downgrade — rolling back means restoring the database, not swapping an image tag. Sequence:

1. Confirm the working directory is `~/commonVM/merged-services/src` and that `docker compose config --format json` resolves `name` to `src`.
2. Preserve a rollback image *before* pulling. Replaced images become dangling and one `docker image prune` deletes them:
   `docker tag <old-image-id> n8nio/n8n:<old-version>`, then `docker save` it to `/mnt/data/backup` so it survives a prune.
3. Back up **and verify**:
   - `docker exec src-n8n-db-1 pg_dump -U n8n -Fc n8n > <file>` — **no `-t`**; a TTY corrupts the binary dump.
   - `tar czf <file> -C /mnt/data/n8n data`
   - Cold physical copy: stop `src-n8n-db-1`, `cp -a /mnt/data/n8n/db <dest>`, start it again. A hot copy of a running PGDATA is torn and will not restore.
4. `docker compose up -d --no-deps --pull never n8n`
   **`--no-deps` is mandatory.** Without it, `depends_on: n8n-db` lets Compose recreate the live database container, because the `postgres:14-alpine` tag has drifted away from the image `src-n8n-db-1` actually runs.
5. Verify before declaring success: networks are `src_web` + `src_backend`, the mount is `bind /mnt/data/n8n/data`, `docker compose ls -a` is clean, `/healthz` returns 200, and row counts match the pre-upgrade backup.

Pin the image tag. `:latest` is what turned a routine command into an unplanned version change.

### Proving a restore actually worked

Row counts prove nothing about fidelity. These do:

```bash
# 1. Credentials genuinely decrypt (contents are written to a temp file, not printed)
docker exec src-n8n-1 n8n export:credentials --all --decrypted --output=/tmp/c.json
```

```sql
-- 2. Migrations mutated nothing: restore the pre-upgrade dump into a throwaway
--    postgres and compare these against the live DB — they must be identical
select md5(string_agg(id||name||type||data, '' order by id)) from credentials_entity;
select name, json_array_length(nodes::json), length(connections::text)
from workflow_entity order by id;
```

Workflow activation succeeding is **not** evidence that credentials work — schedule triggers activate without ever decrypting anything, and an OAuth token can decrypt perfectly while being expired or revoked. Only an actual execution proves that.

A backup that has never been booted is not a rollback plan. Boot the cold PGDATA copy in a throwaway container — against a *copy*, since starting postgres writes into the directory — and confirm it logs `database system is ready to accept connections`.

### Guard rails currently in place

- `docker.service` carries `RequiresMountsFor=/mnt/data` via `/etc/systemd/system/docker.service.d/require-data-mount.conf`. Without it, `fstab`'s `nofail` combined with `restart: always` meant that any boot where Docker won the race against the data disk would let PostgreSQL `initdb` an empty database into a directory on the OS disk — silently, and far harder to diagnose than the outage above.
- n8n image pinned to `n8nio/n8n:2.32.5`.
- `.env` is mode 600 and has never been tracked by git.
- Database dumps sitting in the CommonVM working tree are gitignored — they contain encrypted credential blobs and that repo is public.

### Outstanding risks

- `src-n8n-db-1` runs image `64ce25a0bb68` (PG 14.22) while the `postgres:14-alpine` tag now resolves to PG 14.23. A full `docker compose up -d` **will** recreate the database container. Data survives — it is a bind mount — but expect an unplanned restart, and follow it with `REINDEX DATABASE n8n`: the database collation is `en_US.utf8`, the two images differ in musl version, and 261 indexes sit on text/varchar columns.
- The compose file on disk carries changes not yet applied to running containers (Caddyfile, rustdesk services, `NODE_OPTIONS`). The next full `up` reconciles all of them simultaneously.
- Every backup and the n8n encryption key live on one disk on one VM, with no off-box copy. Losing the key means re-authorising all 12 credentials by hand.
- Nothing monitors the instance. The 2026-07-29 outage was discovered by a human loading the page.

### Environment facts that constrain workflow authoring

- **Python Code nodes will not run on this instance.** The runtime logs `Failed to start Python task runner ... Python 3 is missing from this system`; the JS runner registers normally. All 23 Code nodes across the 6 live workflows are `jsCode`. Keep it that way and treat `.github/skills/n8n-code-python` as inapplicable here.
- **Community nodes are broken, and have been since 2025-11-19** (predates the outage — the pre-incident backup shows the same state). `installed_packages` lists `n8n-nodes-mcp@0.1.28` and `n8n-nodes-firecrawl@0.3.0`, but `/mnt/data/n8n/data/nodes` holds only an empty `package.json` with no `node_modules`, and n8n logs "some packages are missing" at boot. `Raindrop Knowledge Management` references `n8n-nodes-firecrawl.fireCrawl` and will fail if activated before the package is reinstalled.
- **`$env` is BLOCKED at runtime, and this is actively breaking production.** Verified 2026-07-30 by executing a probe Code node on the instance: `$env` access throws `ExpressionError: access to env vars denied`. This is not theoretical — `Daily Currency Exchange Rate Alert` is `active` and **every one of its 10 retained scheduled runs failed**, from 2026-07-16 through 2026-07-29, all with that same error at node `取得即時匯率`, whose URL is `=https://v6.exchangerate-api.com/v6/{{$env.EXCHANGE_API_KEY}}/latest/TWD` (execution #1621 and 9 earlier). The block applies to expressions as well as Code nodes. **Do not author new `$env` references**; move the secret into an n8n credential (e.g. HTTP Request with a generic header/query auth credential) instead. An earlier revision of this file claimed `N8N_BLOCK_ENV_ACCESS_IN_NODE` was unset and `$env` was permitted — that claim is refuted by the runtime evidence above. The historical mapping still holds for context: `EXCHANGE_API_KEY=${N8N_EXCHANGE_API_KEY}` in `docker-compose.yml`, sourced from `merged-services/src/.env`, with `THREADS_ACCESS_TOKEN` following the same pattern.
- **`$vars` is licence-gated and always empty here.** `GET /api/v1/variables` returns HTTP 403 `Your license does not allow for feat:variables`. In a Code node `$vars` is still a defined object, so `$vars.FOO` yields `undefined` rather than throwing — verified by probe: `typeof $vars === 'object'`, `Object.keys($vars) === []`. Consequence: any workflow that depends on `$vars` silently degrades instead of failing loudly. Never treat an n8n variable as a way to configure a workflow on this instance; inline the value (if non-secret) or use a credential (if secret).
- **There is currently NO working AI credential on this instance** (verified 2026-07-30 by executing real requests through each one). Both Azure credentials are type `azureEntraCognitiveServicesOAuth2Api`:
  - `82mlP2DDo7j1VGdC` "Azure Open AI account Entra ID" → `OAuth access token expired and no refresh token is available`. It worked as recently as execution #1619 on 2026-07-29T02:23. Used by **both** `Slack to Google Calendar AI Assistant` (active) and the v2 TEST copy, so AI analysis is down in production.
  - `CtgkMRzzV4whz91f` "Azure Open AI account" → `Unable to sign without access token`, i.e. authorisation was never completed.
  Neither can be repaired from outside the UI: the Public API's `GET /credentials/{id}` returns no `data` field, so `clientId`/`clientSecret` cannot be read, and although the credential's `additionalBodyProperties` is `{"grant_type":"client_credentials"}` — which would normally allow a non-interactive token mint — that requires the secret. `PATCH /credentials/{id}` is accepted (`PUT` returns 405) but is useless without it. **Reconnecting requires signing in to the n8n UI.**
- **`Raindrop Knowledge Management` has a credential type mismatch.** Its `Azure AI 內容分析` node declares `credentials: { azureOpenAiApi: { id: "CtgkMRzzV4whz91f" } }`, but that credential is actually `azureEntraCognitiveServicesOAuth2Api`. Executing it raises `Credential with ID "CtgkMRzzV4whz91f" does not exist for type "azureOpenAiApi"`. The workflow is inactive so this has gone unnoticed; it is a second reason (besides the missing firecrawl package) that it will fail if activated.
- **Deprecations logged by 2.32.5** — none breaking yet, but do not add new usage: `WEBHOOK_URL` → `N8N_WEBHOOK_URL` (the latter sets the base URL for both test and production webhooks); future default changes for `N8N_UNVERIFIED_PACKAGES_ENABLED`, `N8N_RUNNERS_TASK_TIMEOUT` (300s → 60s), and the compression-node limits; `/home/node/.n8n/binaryData` is renamed to `storage` in v3, migratable early with `N8N_MIGRATE_FS_STORAGE_PATH=true`.
- **Scheduled runs are never backfilled.** Any outage silently skips them, and inbound webhooks return 502 and are lost — although Slack retries a failed event for roughly an hour, so some may redeliver after recovery.

## Local Checks

There is nothing to build, lint, or unit-test. The three checks that exist are:

**1. Single-workflow structural check (the closest thing to "run one test")** — parses the export, verifies node-name uniqueness, that every `connections` source/target resolves to a real node, and that the top-level key sequence has not drifted:

```powershell
function Test-N8nWorkflow([string]$Path) {
  $wf = Get-Content $Path -Raw -Encoding UTF8 | ConvertFrom-Json
  $names = @($wf.nodes.name); $bad = @()
  if ($names.Count -ne ($names | Sort-Object -Unique).Count) { $bad += 'duplicate node names' }
  foreach ($src in $wf.connections.PSObject.Properties) {
    if ($names -notcontains $src.Name) { $bad += "orphan source: $($src.Name)" }
    foreach ($port in $src.Value.PSObject.Properties) {
      foreach ($branch in $port.Value) { foreach ($c in $branch) {
        if ($c -and $names -notcontains $c.node) { $bad += "missing target: $($c.node)" } } }
    }
  }
  $expected = 'name,nodes,pinData,connections,active,settings,versionId,meta,id,tags'
  if ((@($wf.PSObject.Properties.Name) -join ',') -ne $expected) { $bad += 'top-level key drift' }
  '{0,-42} {1,2} nodes  {2}' -f $wf.name, $names.Count, $(if ($bad) { 'FAIL -> ' + ($bad -join '; ') } else { 'OK' })
}

# one workflow
Test-N8nWorkflow 'flows\SlackToGoogleCalendar\Slack_to_Google_Calendar_AI_Assistant.json'
# all of them
Get-ChildItem flows -Recurse -Filter *.json | ForEach-Object { Test-N8nWorkflow $_.FullName }
```

**2. Doc-drift check** — verifies that every node name in an export appears in both READMEs. This is what catches documentation rotting away from the JSON, which has happened before:

```powershell
function Test-N8nDocs([string]$Path) {
  $wf = Get-Content $Path -Raw -Encoding UTF8 | ConvertFrom-Json
  $dir = Split-Path $Path -Parent
  foreach ($doc in 'README.md','README.zh-tw.md') {
    $file = Join-Path $dir $doc
    if (-not (Test-Path $file)) { '{0,-16} absent' -f $doc; continue }
    $text = Get-Content $file -Raw -Encoding UTF8
    $missing = @($wf.nodes.name | Where-Object { -not $text.Contains($_) })
    '{0,-16} {1,2}/{2} nodes documented  {3}' -f $doc, ($wf.nodes.Count - $missing.Count), $wf.nodes.Count, $(if ($missing) { 'MISSING -> ' + ($missing -join '; ') } else { 'OK' })
  }
}
Get-ChildItem flows -Recurse -Filter *.json | ForEach-Object { "== $($_.Directory.Name)"; Test-N8nDocs $_.FullName }
```

It matches on literal node names, so it only means something when node names are English — which is the naming contract above. `RaindropKnowledgeManagement` reports `1/11` for its English README because that flow's nodes are named in Chinese and the English doc translates them; that is a pre-existing naming-contract violation, not a false positive to suppress.

**3. Authoritative check** — the MCP `validate_workflow` tool with `profile: "strict"`, passing the workflow JSON inline. Local JSON parsing cannot catch bad node parameters; only this can. Use `validate_workflow` (takes the JSON) rather than `n8n_validate_workflow` (takes an ID and needs API auth — see [Talking to the instance](#talking-to-the-instance)). Always read files with `-Encoding UTF8` — every workflow contains zh-TW text and emoji.

Expect ~30 warnings on `SlackToGoogleCalendar` (`Hardcoded nodeCredentialType detected`, `Code nodes can throw errors`). Those are inherent to the raw-HTTP Slack pattern and are not actionable. Only `errorCount` matters.

There is no automated way to test behaviour. See [Exercising a workflow](#exercising-a-workflow).

### Exercising a workflow

The only functional test is running it on the live instance and reading the execution. Three techniques this repo relies on:

**Synthetic webhook replay.** Slack event delivery to this instance is unreliable — messages posted in the client are sometimes visible yet generate no `event_callback`, and the bot's own `conversations.history` may not return them. POSTing a hand-built `event_callback` to the production webhook exercises everything downstream of the trigger and is far more dependable:

```javascript
POST https://n8n.yu.money/webhook/<path>/webhook
{ type: 'event_callback', team_id, api_app_id,
  event: { type: 'message', channel, user, text, ts, event_ts, thread_ts, channel_type: 'channel' } }
```

`ts` must be a **real** message timestamp, because reaction calls target it. Include `thread_ts` when simulating a thread reply — omitting it makes the workflow treat the reply as a new top-level message and the anchor logic silently tests nothing.

**Reading the result.** `GET /executions?limit=&workflowId=` then `GET /executions/{id}?includeData=true`. `data.resultData.runData` is keyed by node name; per-branch item counts live at `runData[name][run].data.main[branch].length`. A node that ran but emitted nothing shows `main: [[]]` — that is how you tell "did not run" from "returned zero items", and the two have very different causes.

**A throwaway workflow for API calls.** Credential secrets are not readable through the Public API, so to call Slack or Google with the *production* credential, `POST /workflows` a small webhook-triggered workflow that reuses the same `credentials` object copied from a production node, activate it, call it, then deactivate and `DELETE` it. This is how test messages get posted and how test calendar events and cards get cleaned up afterwards. Always clean up: the calendar and the Slack channel are real.

Beware routing guards when constructing a test. Reaching the `create` branch requires `isOwner` (the thread's first message must be authored by the person whose id you put in the event) and `!isDuplicate` (the thread must not already contain that event). Scan past executions for a thread where `isOwner` was already observed true rather than guessing.

## n8n Workflow Development

### Talking to the instance

**The n8n MCP tools are only half-usable here, and the failure is silent.** Verified 2026-07-31.

`.mcp.json` passes `"N8N_API_KEY": "${N8N_API_KEY}"`, and **nothing expands that placeholder** — neither Copilot CLI nor the n8n-mcp server. The literal string `${N8N_API_KEY}` is sent as the API key. `N8N_API_KEY` is not set in the shell either, so there is nothing to expand from.

The result splits cleanly in two:

| Tool group | Status | Examples |
|---|---|---|
| **Documentation tools** — never touch the instance | **work** | `search_nodes`, `get_node`, `validate_node`, `validate_workflow` (takes workflow JSON), `search_templates`, `get_template`, `tools_documentation` |
| **Management tools** — call the n8n API | **fail: `AUTHENTICATION_ERROR`** | `n8n_list_workflows`, `n8n_get_workflow`, `n8n_update_partial_workflow`, `n8n_update_full_workflow`, `n8n_executions`, `n8n_validate_workflow` (takes an ID), `n8n_manage_credentials`, … |

**`n8n_health_check` lies.** It reports `"connected": true` and `"N8N_API_KEY": "***configured***"` while every management call fails. Do not use it to decide whether the API works — make a real call such as `n8n_list_workflows`.

To make the management tools work, export the real key into the environment **before** launching Copilot CLI, so the placeholder has something to resolve to:

```powershell
$env:N8N_API_KEY = '<key>'   # never commit it; this repo is public
```

Until then, drive the instance through its Public REST API from a script:

```javascript
const BASE = 'https://n8n.yu.money/api/v1';
const H = { 'X-N8N-API-KEY': KEY, accept: 'application/json', 'content-type': 'application/json' };
// GET /workflows, GET /workflows/{id}, PUT /workflows/{id},
// GET /executions?limit=&workflowId=&status=, GET /executions/{id}?includeData=true
```

Two API quirks that will bite:

- `PUT /workflows/{id}` **rejects `settings.binaryMode`** even though `GET` returns it. Strip it from the payload or the update 400s.
- `PUT` takes only `{ name, nodes, connections, settings }`. Sending `id`, `active`, `versionId` or `tags` is rejected.

### Workflow operations

1. **Read**: `GET /workflows/{id}` (or `n8n_get_workflow` once the key resolves)
2. **Validate**: pass the workflow JSON to the MCP `validate_workflow` tool with `profile: "strict"` — this one works today because it takes the JSON inline rather than fetching by ID
3. **Update**: `PUT /workflows/{id}` with the four accepted keys. Prefer `n8n_update_partial_workflow` once auth works — it has dedicated `addNode`, `removeNode`, `addConnection`, `removeConnection`, `rewireConnection` and `replaceConnections` operations (see `.github/skills/n8n-mcp-tools-expert/WORKFLOW_GUIDE.md`), so structural edits do not require a full replace
4. **Sync local**: after every cloud update, sync cloud → local JSON (see [Cloud ↔ Local Sync](#cloud--local-sync))

**Write deployment scripts that assert their preconditions before mutating.** The established pattern in this repo is a `must(condition, message)` helper that throws if an expected substring is missing, plus `new Function(code)` as a syntax gate before pushing a Code node. A `PUT` that half-applies because an anchor string moved is far more expensive than a script that refuses to run.

### Node Naming

- Use descriptive English names (e.g., `Parse AI Response`, `Compute Anchor Status`, `Verify Card Delivery`)
- Never use default names like `IF`, `Code`, `HTTP Request1`
- **Legacy exception**: `DailyCurrencyExchangeRateAlert`, `JenDailyScheduleAlert`, and most of `RaindropKnowledgeManagement` still use zh-TW node names (`取得即時匯率`, `處理排班資料`). Do not mass-rename them — a rename breaks `connections` keys, `$('Node Name')` lookups, and stored execution history. Rename only when already restructuring that node, and update every reference.

### Node Versions

Currently observed across the repo (not all in one workflow) — treat as the floor for new work, and confirm with the MCP `get_node` tool (which works) before assuming a version is still current. The instance jumped 2.14.2 → 2.32.5 on 2026-07-29, so several of these almost certainly have newer `typeVersion`s available now; the table records what the files contain, not what the instance supports.

| Node Type | Version(s) in repo |
|-----------|---------------|
| Schedule Trigger | 1.1 |
| HTTP Request | 4.1 (legacy flows), **4.2** (SlackToGoogleCalendar) |
| Code | 2 |
| IF | 2 (Raindrop), **2.3** (SlackToGoogleCalendar) |
| Switch | 3.2 |
| Merge | 2.1 |
| Aggregate | 1 |
| Slack | 1 — legacy alert flows only |
| Slack Trigger | 1 |
| Google Calendar | 1.3 |
| Google Sheets | 4 |
| Notion | 2 |
| Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`) | 1.9 |
| Azure OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`) | 1 |

### Known drift

Do not assume the repo is uniform. Currently:

- **The Slack node exists only at v1, in `DailyCurrencyExchangeRateAlert` and `JenDailyScheduleAlert`** (plain `"channel": "#n8n-currency"` + `text`). `SlackToGoogleCalendar` does **not** use the Slack node at all — every Slack interaction is a raw `httpRequest` node against `chat.postMessage`, `conversations.replies`, `reactions.add` and `reactions.remove` with `authentication: predefinedCredentialType` + `nodeCredentialType: slackApi`. That is the pattern to copy: the node cannot express message `metadata`, reaction removal, or `include_all_metadata`, all of which this flow depends on.
- IF is v2 in `RaindropKnowledgeManagement`, v2.3 in `SlackToGoogleCalendar`. Only `SlackToGoogleCalendar` uses Switch (v3.2) and Aggregate.
- Only `SlackToGoogleCalendar` carries a full `settings` block: `saveExecutionProgress`, `saveManualExecutions`, `saveDataErrorExecution: "all"`, `saveDataSuccessExecution: "all"`, `executionTimeout: 3600`, `timezone: "Asia/Taipei"`, `executionOrder: "v1"`, `binaryMode: "separate"`, `callerPolicy: "workflowsFromSameOwner"`, `availableInMCP: false`. The other three only have `{"executionOrder": "v1"}`. Copy the reference flow's block verbatim for new or substantially reworked workflows rather than retyping it.
- Only `SlackToGoogleCalendar` and `RaindropKnowledgeManagement` have README pairs. `flows/SlackToGoogleCalendar/README.*` are now generated from the live workflow and are accurate (37/37 node coverage); `RaindropKnowledgeManagement`'s are hand-written and older. Prefer the JSON when the two disagree.
- Bumping a node's `typeVersion` is a behavioural change, not a cleanup. Bump only when you are also validating and testing that workflow.

### Code Node Conventions

Target for new and reworked Code nodes:

- Use `n8n-nodes-base.code` v2 (not deprecated `function` v1)
- **JavaScript only.** The Python task runner is not available on this instance — see [Environment facts](#environment-facts-that-constrain-workflow-authoring). A `pythonCode` node fails at runtime, not at validation.
- Parameter: `jsCode` + explicit `mode: "runOnceForAllItems"`. The two legacy alert flows omit `mode` entirely and return a bare `{ json: { ... } }` object instead of an array — that still runs, but new code should use the explicit form.
- Return format: `[{ json: { ... } }]`
- Every `catch` block must have `console.log` with error details — never silently swallow errors
- Reference other nodes with `$('Node Name').first().json` — but read [Data flow between nodes](#data-flow-between-nodes) first; this is the single most common way a change here fails silently
- Code nodes run real JS: optional chaining (`?.`), `try/catch`, and array methods are used throughout `Parse AI Response` and `篩選新項目`. The `?.` restriction below applies to `{{ }}` expressions, not here.
- Luxon `DateTime` is available without import. zh-TW output uses the locale option: `DateTime.fromISO(v).setZone('Asia/Taipei').toFormat('MM月dd日 (cccc) HH:mm', { locale: 'zh-TW' })`
- Degrade, don't throw. The house style on bad input in `SlackToGoogleCalendar` is to `console.log` and return a status item (`{ status: 'error', error: '...' }`) so downstream IF nodes can route it, rather than failing the execution.
- **Never `return []` from a node on the happy path.** An empty array ends that branch with no error and no output, so every downstream node is skipped and the user simply never gets a reply. When there is genuinely nothing to emit, return one item that downstream routing can recognise (`Parse AI Response` emits a clarify item when every event fails to parse).

### Data flow between nodes

This is where changes in this repo go wrong. Three rules, each learned from a production bug.

**1. `$('Node Name')` reach-back is position-dependent and fails silently.** It is not a global lookup. Whether it resolves depends on where the calling node sits relative to the referenced one, and when it fails you get an exception (caught → empty object) rather than a warning.

Verified in single executions of the same workflow:

- `$('Collect Thread Context').first()` returned nothing from `Compute Anchor Status` (execution `#1908` produced 0 items) — while in that *same* execution the sibling node `Add Success Reaction` resolved `$('Slack Message Trigger').first()` correctly.
- `$('Parse AI Response').first()` *does* resolve from the `Verify * Delivery` nodes (`#1913` derived the right emoji through it).
- On an error branch, `$('Slack Message Trigger')` resolves but `$('Build Azure Payload')` does not. The failure is per-node item pairing, not "error branches break `$()`".

So: **prefer carrying a value forward on the item over reaching back for it.** `Parse AI Response` spreads `...ctx` into every item precisely so downstream nodes do not have to. When you must reach back, wrap it in `try/catch`, log, degrade — and prove it works by reading a real execution, not by reasoning about it.

**2. `.first()` versus `.item`.** `.first()` returns item 0 of the referenced node, *not* the item paired with the current one. That is invisible while a node emits exactly one item and catastrophic the moment it emits several: `Send Success Card` held 19 `$('Parse AI Response').first()` references, so when one message produced three calendar events all three cards described event #1 while linking to three different events. Use `.item` for per-item correlation, or insert a Code node (`Pair Created Event`) that resolves the pairing once and merges the fields onto the item.

**3. A node's mode decides how often it runs.** Everything downstream of a node that emits N items runs N times. `runOnceForEachItem` is correct for per-item correlation; `runOnceForAllItems` is correct for anything that should happen once per execution regardless of fan-out — for example `Compute Anchor Status`, because a thread has one status no matter how many events were created, and `reactions.*` is a Slack Tier 2 endpoint capped at 20 requests per minute.

### Expression Syntax

- This repo writes expressions without optional chaining — the established pattern is `$json.field && $json.field.prop` (see `Send Error Card`). Keep it consistent; the `.github/skills/n8n-expression-syntax` docs do not require `?.` either.
- Expression strings start with `=` in the JSON (`"text": "={{ $json.fallbackText }}"`). A value without the leading `=` is a literal.
- Slack messages use `<url|text>` format for links (not Markdown `[text](url)`)
- For timezone-aware formatting: `DateTime.fromISO(value).setZone('Asia/Taipei').toFormat(...)`
- Channel/calendar/database pickers are resource locators, not plain strings:
  `"channelId": { "__rl": true, "mode": "id", "value": "<channel-id>" }`
- IF v2.3 conditions carry a stable `id` per condition, `operator: { type, operation }`, and `options: { version: 2, typeValidation: "loose", caseSensitive: true }`. Preserve that shape when editing conditions; MCP partial updates add the v2.2+ metadata automatically, so let it, rather than hand-writing a partial condition object.

## Cross-Node Contracts

These conventions span multiple nodes and are the main thing to preserve when editing. All of the following describes `SlackToGoogleCalendar`; the other three flows have none of it.

### Error routing

Two different `onError` settings, used deliberately:

- **`continueErrorOutput`** on the eleven nodes whose failure means the request cannot continue — `Fetch Thread Replies`, `Collect Thread Context`, `Split Images`, `Build Azure Payload`, `Analyze With Azure`, `Parse AI Response`, `Create Calendar Event`, `Update Calendar Event`, and the three `Verify * Delivery` nodes. Their **second** `main` output (index 1) all converge on one `Normalize Error` → `Send Error Card` sink. Add a fallible node to the main path and wire it there too, rather than letting the execution die.
- **`continueRegularOutput`** on every Slack-facing node (the six card senders, all reaction nodes, `Download Image`). A Slack hiccup must not abort a run that has already written to the calendar. The catch is that n8n then pushes `{ error: msg }` to the *regular* output, so the card senders are followed by a `Verify * Delivery` Code node that treats anything other than `ok === true` as failure. Without that, a failed card would pass silently.

`Normalize Error` derives the failing stage from **`$prevNode.name`**, which is node-level metadata and does not depend on item pairing — an earlier version used `$()` reach-back and mislabelled every failure. If you rename or insert a node on the main path, update its stage map, including whether that stage implies the calendar was already written; otherwise the error card can tell a user to retry after events were created.

### Routing vocabulary

`Route Outcome` is a Switch (v3.2) with eight outputs. The keys are the contract between `Parse AI Response` and everything after it:

| Output | Key | Target |
|---|---|---|
| 0 | `suppress` | `Add Suppress Reaction` (➖, not a scheduling message) |
| 1 | `not_owner` | `Send Not Owner Card` (🔒) |
| 2 | `update` | `Update Calendar Event` |
| 3 | `force` | `Create Calendar Event` (user said "just create it") |
| 4 | `clarify` | `Send Clarify Card` (❓) |
| 5 | `create` | `Create Calendar Event` |
| 6 | `duplicate` | `Send Duplicate Notice` (♻) |
| 7 | fallback | `Send Clarify Card` |

Note outputs 3 and 5 share a target, so you cannot infer the branch from the destination node. The confidence threshold (`0.7`) now lives in **exactly one place**, on `Route Outcome`; keep it that way.

Only `create` fans out to several items. `update` stays single, because an edit targets one existing calendar entry and fanning it out would multiply updates against the same event id. `MAX_EVENTS_PER_MESSAGE = 5` bounds a hallucinating model.

### Slack cards

Cards are **not** built by a Code node and **not** sent by the Slack node. Each `Send * Card` is an `httpRequest` node whose `jsonBody` is a single expression that builds the whole `chat.postMessage` payload inline — `attachments[0].blocks[]`, colour, and `metadata` — from `$json`:

```
"jsonBody": "={{ JSON.stringify({ channel: $json.channel, thread_ts: $json.threadTs, text: …,
                metadata: { event_type: 'gcal_event_created', event_payload: { … } },
                attachments: [{ color: '#2eb886', blocks: [ … ] }] }) }}"
```

Read fields from `$json`, never `$('Parse AI Response').first()` — see [Data flow between nodes](#data-flow-between-nodes) for why that broke the multi-event case.

**`metadata.event_payload` is the only state store in the whole system.** There is no database. `Collect Thread Context` reads `event_id` / `calendar_id` / `start` / `title` back off the bot's own earlier cards (filtered by `app_id`, so another app cannot forge them) to rebuild `ctx.allEvents`, which drives duplicate detection and update targeting. Break that field and the bot loses its memory: it re-creates events it already made, and edits the wrong meeting. `Fetch Thread Replies` must keep `include_all_metadata: true`.

### Reaction lifecycle

Two independent layers — do not merge them:

- **Per-message**, on the message that triggered *this* turn: 👀 `eyes` on arrival, removed once parsing finishes, then the outcome (✅ `white_check_mark`, 🔄 `arrows_counterclockwise`, ➖ `heavy_minus_sign`, ⚠ `warning`, or a dynamic ❓/♻/🔒 from `Add Notice Reaction`).
- **Per-thread**, on the thread's anchor (first) message: `Compute Anchor Status` → `Sync Anchor Reaction`, a single `httpRequest` whose URL is `reactions.{{ $json.op }}` because add and remove take identical parameters. The channel list only shows a thread's first message, so the anchor is the status board; stale outcomes are removed before the current one is added.

The anchor mapping is deliberately not the identity: `duplicate` resolves to ✅ (it is only a duplicate because the event exists) and `not_owner` leaves the anchor untouched, so a stranger cannot rewrite a thread's status. `Add Suppress Reaction` is intentionally not wired into the anchor sync — saying "thanks" in a resolved thread must not erase its ✅.

### AI analysis chain

`chainLlm` v1.9 (prompt) + `lmChatAzureOpenAi` v1 (model) over the `ai_languageModel` port, then `Parse AI Response` parses the output.

- **Use the LangChain node pair, not `httpRequest` + `predefinedCredentialType`.** The Azure credential is `azureEntraCognitiveServicesOAuth2Api` minting tokens with `grant_type: client_credentials`, which returns no refresh token; n8n's generic OAuth2 helper (used by `httpRequest`) can then only fail once the access token expires. `lmChatAzureOpenAi` acquires its own token and is unaffected. This was proven side by side with the same credential in the same minute.
- Images reach the model only through **static** `messages.messageValues` slots. Making the whole `messages` parameter an expression runs without error but the images never arrive, and an empty slot makes the node fail outright. `Build Azure Payload` therefore always emits four slots, padding unused ones with a 1×1 transparent PNG and instructing the model to ignore blanks.
- The prompt injects "now" via `{{ DateTime.now().setZone('Asia/Taipei').toFormat('yyyy年MM月dd日 (cccc)', { locale: 'zh-TW' }) }}` and demands raw JSON. The parser still strips fences defensively — `text.replace(/```json\n?|```\n?/g, '').trim()` — because the model does not always comply.
- `Parse AI Response` reads `$json.text` (the chain's output) and keeps the old `choices[0].message.content` shape as a fallback.

If you change the prompt's JSON shape, update the parser, the `Route Outcome` conditions and the card builders in the same change.

## Validation & Sync

### Workflow Validation Checklist

After every change:
1. Validate with the MCP `validate_workflow` tool, `profile: "strict"`, passing the workflow JSON
2. Check `errorCount` = 0 (ignore `"Cannot return primitive values directly"` — a static-analysis false positive for Code nodes)
3. Check `invalidConnections` = 0
4. Verify `totalNodes` and `validConnections` match what you expect — currently **37 / 54** for `SlackToGoogleCalendar`. A number that moved when you did not add or remove anything means the update did more than you intended
5. Sync cloud → local JSON, then run the [structural check](#local-checks)
6. Exercise it on the live instance and read the execution — see [Exercising a workflow](#exercising-a-workflow). Cover more than the happy path: for `SlackToGoogleCalendar`, at minimum a single-event message, a multi-event message, a follow-up turn in an existing thread, and a message from someone who is not the thread owner
7. Check the git diff shape before committing: a workflow change should touch only the `jsCode`/`jsonBody` strings you edited, any nodes you added, their connections, and `versionId`. Hundreds of reordered lines means the export was rebuilt wrong

One MCP validator quirk worth knowing: it flags nodes whose **names** carry failure semantics when they share a `main[0]` with other nodes. Restructuring will not clear it; renaming will (`Add Error Reaction` → `Add Alert Reaction` did). Do not contort the graph to satisfy it.

### Cloud ↔ Local Sync

The n8n instance is the source of truth; the repo is the audit trail. Push changes to the cloud via MCP first, read the result back, then rewrite the local file — never hand-edit local JSON and expect the cloud to follow.

The MCP response shape differs between tools and modes (some wrap the workflow under `data.workflow`, some put it directly under `data`), so the snippet below probes for it and refuses to write anything it cannot recognise:

```powershell
$localPath = 'flows\<Folder>\<Workflow_Name>.json'
$raw = Get-Content '<mcp-result-file>' -Raw -Encoding UTF8 | ConvertFrom-Json

$wf = $raw.data.workflow; if (-not $wf) { $wf = $raw.data }; if (-not $wf) { $wf = $raw }
if (-not ($wf.id -and $wf.nodes -and $wf.connections)) { throw "Unexpected MCP payload - refusing to overwrite $localPath" }

$prev = $null
if (Test-Path $localPath) { $prev = Get-Content $localPath -Raw -Encoding UTF8 | ConvertFrom-Json }
$pinData = [ordered]@{}; if ($wf.pinData) { $pinData = $wf.pinData }
$meta = $wf.meta;        if (-not $meta -and $prev) { $meta = $prev.meta }   # keep meta.instanceId
$tags = @();             if ($wf.tags)    { $tags = @($wf.tags) }

$export = [ordered]@{
  name = $wf.name; nodes = $wf.nodes; pinData = $pinData
  connections = $wf.connections; active = $wf.active
  settings = $wf.settings; versionId = $wf.versionId
  meta = $meta; id = $wf.id; tags = $tags
}
$export | ConvertTo-Json -Depth 20 | Set-Content $localPath -Encoding UTF8
```

Two PowerShell traps this snippet avoids — do not "simplify" them back:

- Assigning an array from an `if` expression (`$tags = if (...) { @() }`) unrolls to `$null` and writes `"tags": null` instead of `"tags": []`. Use plain statements, as above.
- `-Depth 20` is required. Without it `ConvertTo-Json` truncates at depth 2 and writes node parameters as PowerShell strings (`"parameters": "@{rule=}"`), which destroys the workflow. It only prints a warning, so it is easy to miss.

Key order matters — it keeps `git diff` readable, and the [structural check](#local-checks) fails on drift. After syncing, confirm the diff contains only what you changed plus `versionId`; a diff that reorders hundreds of lines means the export was rebuilt wrong.

### Credentials & secrets

Workflow JSON references credentials by `{ id, name }` only (e.g. `"slackApi": { "id": "<credential-id>", "name": "Slack account" }`) — the secrets live in n8n. Never inline a token, key, or webhook secret into a workflow file.

`.mcp.json` is committed and may never contain a literal key — this repo is public.

- `.mcp.json` (Copilot CLI): `"N8N_API_KEY": "${N8N_API_KEY}"` inside a stdio server's `env` block. **This placeholder is confirmed not to expand** (verified 2026-07-31) — the literal string is sent as the key, so every n8n management tool returns `AUTHENTICATION_ERROR` while `n8n_health_check` still claims `"connected": true`. Keep the placeholder — the key must never be committed to this public repo — and supply the real value through the environment instead. See [Talking to the instance](#talking-to-the-instance).
- There is intentionally **no** `.vscode/mcp.json`. It was removed on 2026-07-29; every server it declared was already present in `.mcp.json`, so it only added a second copy to keep in sync. Do not recreate it unless asked.

`context7` needs no key at all. It is an HTTP server pointed at `https://mcp.context7.com/mcp/oauth`, which authenticates via MCP OAuth — browser consent on first connection, tokens cached under `~/.copilot/mcp-oauth-config`. Two things worth knowing if it misbehaves:

- **Fallback**: drop the `/oauth` suffix. `https://mcp.context7.com/mcp` works anonymously (verified: a real `resolve-library-id` call returns results without any credential), just at a lower rate limit. That is a zero-cost rollback if OAuth consent fails.
- **Do not "fix" it by adding an API key header.** Besides being a secret in a public repo, `headers` are not placeholder-expanded either, and Context7 rejects an unexpanded value with `Invalid API key` — which is *worse* than sending nothing, because sending nothing succeeds anonymously.

One exception to know about: `DailyCurrencyExchangeRateAlert` puts its API key in the request URL via the n8n environment variable `{{$env.EXCHANGE_API_KEY}}`. **This no longer works.** `$env` access is denied at runtime on this instance, so the workflow fails on every scheduled run even though the variable *is* injected by the container as `EXCHANGE_API_KEY=${N8N_EXCHANGE_API_KEY}` from the deployment repo's `.env`. Fixing it requires moving the key into an n8n credential, not restoring the env var; see [Environment facts that constrain workflow authoring](#environment-facts-that-constrain-workflow-authoring).

## Change Process

1. **Plan** — Describe changes with risk assessment, dependency analysis, and rollback strategy
2. **Implement** — Execute in waves (low risk → high risk), validate after each wave
3. **Verify** — Run the three review rounds in [Principles](#reviews)
4. **Test** — Exercise the workflow on the live instance (see checklist step 6)
5. **Sync** — Cloud → local JSON
6. **Commit** — Conventional commit format: `refactor(WorkflowName): description`

Commit scope is the workflow folder name (`feat(SlackToGoogleCalendar): ...`); use `docs:` / `chore:` for repo-level changes. Subjects are written in English, and `.vscode/settings.json` configures Conventional Commits for Copilot commit generation. Note the existing history is mixed (`add JenDailyScheduleAlert/...`, `Update 2.txt`) — follow the convention going forward rather than matching the worst of the log.

### Documentation duty

A workflow change is not finished when the JSON is synced. These are requirements going forward, not a description of how tidy the repo currently is:

- If the flow has `README.md` / `README.zh-tw.md`, update **both**, and run the doc-drift check in [Local Checks](#local-checks) before you call the change done. `flows/SlackToGoogleCalendar/`'s pair is **generated from the live workflow** rather than hand-edited, which is why it holds at 37/37 in both languages; regenerate rather than patching by hand, and add any new node to the generator's node grouping or it will refuse to run.
- Add a `CHANGELOG.md` entry under a semver heading (`## [1.0.9] - YYYY-MM-DD`) with subsections matching the existing style (`### Fixes` / `### Enhancements` / `### Documentation` / `### Verification`). The house style records the *evidence* — execution numbers, what was measured versus inferred — not just what changed.
- The root `README.md` lists workflows; it currently documents only two of the four and was last touched 2025-05-29, so add yours if you create a new flow.

## Skills Reference

Seven n8n-specific skills live under `.github/skills/` and are auto-loaded by Copilot when relevant. Consult them instead of duplicating their content here — they are the deeper reference for MCP tool usage, expression syntax, node configuration, and validation errors:

- `n8n-code-javascript` — Code node JS patterns
- `n8n-code-python` — Code node Python patterns. **Not usable on this instance** — the Python runner is missing; see [Environment facts](#environment-facts-that-constrain-workflow-authoring).
- `n8n-expression-syntax` — Expression validation
- `n8n-mcp-tools-expert` — MCP tool usage guide
- `n8n-node-configuration` — Node config patterns
- `n8n-validation-expert` — Validation error interpretation
- `n8n-workflow-patterns` — Workflow architecture patterns

## Principles

### Reviews
完成後進行三輪驗證：

1. 第一輪：重新審視你所有要做的事情，確認你都百分之百滿意。若沒有到百分之百滿意，就請持續修正，直到百分之百滿意為止。
2. 第二輪：進行第二輪審核，看看有任何你認為需要再審核的地方。
3. 第三輪：針對第一輪與第二輪的審核結果，再重新審核一遍。