# Project Guidelines

## Overview

n8n workflow automation repository. All workflows are stored as JSON files under `flows/` and managed via n8n MCP tools + Git.

- **Instance**: self-hosted n8n at `https://n8n.yu.money`, running as one service on a shared Azure VM — read [Hosting & Operations](#hosting--operations) before running anything on the host or reasoning about why a workflow behaves differently there than the JSON suggests. MCP servers are configured in `.mcp.json` (repo root, read by Copilot CLI). Version verified on the box on 2026-07-29 is **2.32.5**; the `1.93.0` in `flows/RaindropKnowledgeManagement/README.md` is from 2025-05 and stale. Re-confirm via MCP rather than trusting any number in a doc — including this one.
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

`flows/SlackToGoogleCalendar/Slack_to_Google_Calendar_AI_Assistant.json` is the most current workflow and the one to copy patterns from (English node names, IF v2.3, Slack v2.4, error outputs, Block Kit notifications, full `settings` block). The other three flows predate those conventions — see [Known drift](#known-drift).

### Workflow inventory

Snapshot — re-read the JSON or query MCP before relying on any count or version here.

| Folder | Trigger → Sink | Notes |
|---|---|---|
| `SlackToGoogleCalendar` | Slack trigger → Azure OpenAI → Google Calendar → Slack | 14 nodes; the reference flow |
| `RaindropKnowledgeManagement` | Schedule (30 min) → Raindrop → Azure AI → Notion | 11 nodes; IF/Merge branch for "AI vs basic analysis". The schedule interval and the dedupe window in `篩選新項目` are coupled: the window is `interval + 5` minutes. Change one, change the other. Also `perpage: 10` on the Raindrop request. |
| `JenDailyScheduleAlert` | Schedule → Google Sheets → Slack | 4 nodes, linear; does its own Taipei/Calgary offset + DST math in the Code node |
| `DailyCurrencyExchangeRateAlert` | Schedule → HTTP → Slack | 4 nodes, linear; the only flow that authenticates via an n8n env var (`{{$env.EXCHANGE_API_KEY}}`) instead of a credential. **Currently broken** — `$env` access is denied at runtime, so every scheduled run has failed since 2026-07-16; see [Environment facts](#environment-facts-that-constrain-workflow-authoring) |

**The instance holds more workflows than this repo does.** As of 2026-07-29 the live database has **6** workflows and **12** credentials; `Github flow backup` and `Spotify Weekly Backup Schedule` exist only on the instance and have never been exported here. Only two are active — `Daily Currency Exchange Rate Alert` and `Slack to Google Calendar AI Assistant`. When auditing "all workflows", query MCP; `flows/` is not the full set, and a repo-only audit will miss a third of them.

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

**3. Authoritative check** — `mcp_n8n-mcp_n8n_validate_workflow` with `profile: "strict"` against the live workflow. Local JSON parsing cannot catch bad node parameters; only MCP validation can. Always read files with `-Encoding UTF8` — every workflow contains zh-TW text and emoji.

There is no automated way to test behaviour. The only functional test is triggering the workflow on the live instance and reading the execution.

## n8n Workflow Development

### MCP-First Workflow

Always use n8n MCP tools for workflow operations:

1. **Read**: `mcp_n8n-mcp_n8n_get_workflow` (mode: structure/details)
2. **Validate**: `mcp_n8n-mcp_n8n_validate_workflow` (profile: strict)
3. **Update**: prefer `mcp_n8n-mcp_n8n_update_partial_workflow` — it has dedicated operations for `addNode`, `removeNode`, `addConnection`, `removeConnection`, `rewireConnection`, and `replaceConnections` (see `.github/skills/n8n-mcp-tools-expert/WORKFLOW_GUIDE.md`), so structural edits do not require a full replace. Use `mcp_n8n-mcp_n8n_update_full_workflow` only when deliberately replacing a whole workflow.
4. **Sync local**: After cloud update, always sync cloud → local JSON

### Node Naming

- Use descriptive English names (e.g., `Parse AI Response`, `Check Confidence Score`)
- Never use default names like `IF`, `Code`, `HTTP Request1`
- **Legacy exception**: `DailyCurrencyExchangeRateAlert`, `JenDailyScheduleAlert`, and most of `RaindropKnowledgeManagement` still use zh-TW node names (`取得即時匯率`, `處理排班資料`). Do not mass-rename them — a rename breaks `connections` keys, `$('Node Name')` lookups, and stored execution history. Rename only when already restructuring that node, and update every reference.

### Node Versions

Currently observed across the repo (not all in one workflow) — treat as the floor for new work, and confirm against MCP before assuming a version is still current. The instance jumped 2.14.2 → 2.32.5 on 2026-07-29, so several of these almost certainly have newer `typeVersion`s available now; the table records what the files contain, not what the instance supports.

| Node Type | Version in repo |
|-----------|---------------|
| Schedule Trigger | 1.1 |
| HTTP Request | 4.1 |
| Code | 2 |
| IF | 2.3 |
| Merge | 2.1 |
| Slack | 2.4 |
| Slack Trigger | 1 |
| Google Calendar | 1.3 |
| Google Sheets | 4 |
| Notion | 2 |
| Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`) | 1.9 |
| Azure OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`) | 1 |

### Known drift

Do not assume the repo is uniform. Currently:

- Slack nodes are v1 in `DailyCurrencyExchangeRateAlert` and `JenDailyScheduleAlert` (plain `"channel": "#n8n-currency"` + `text`), v2.4 only in `SlackToGoogleCalendar` (`resource`/`operation`/`select` + `channelId` resource locator). **New workflows use the v2.4 shape**; do not copy the v1 nodes forward.
- IF is v2 in `RaindropKnowledgeManagement`, v2.3 in `SlackToGoogleCalendar`.
- Only `SlackToGoogleCalendar` carries a full `settings` block: `saveExecutionProgress`, `saveManualExecutions`, `saveDataErrorExecution: "all"`, `saveDataSuccessExecution: "all"`, `executionTimeout: 3600`, `timezone: "Asia/Taipei"`, `executionOrder: "v1"`, `binaryMode: "separate"`, `callerPolicy: "workflowsFromSameOwner"`, `availableInMCP: false`. The other three only have `{"executionOrder": "v1"}`. Copy the reference flow's block verbatim for new or substantially reworked workflows rather than retyping it.
- Only `SlackToGoogleCalendar` and `RaindropKnowledgeManagement` have README pairs, and **`flows/SlackToGoogleCalendar/README.md` is stale** — it documents a `Webhook` trigger, an `AI Message Analyzer` node, and `Function` nodes that no longer exist. Read the JSON, never the README, to learn what a workflow actually does.
- Bumping a node's `typeVersion` is a behavioural change, not a cleanup. Bump only when you are also validating and testing that workflow.

### Code Node Conventions

Target for new and reworked Code nodes:

- Use `n8n-nodes-base.code` v2 (not deprecated `function` v1)
- **JavaScript only.** The Python task runner is not available on this instance — see [Environment facts](#environment-facts-that-constrain-workflow-authoring). A `pythonCode` node fails at runtime, not at validation.
- Parameter: `jsCode` + explicit `mode: "runOnceForAllItems"`. The two legacy alert flows omit `mode` entirely and return a bare `{ json: { ... } }` object instead of an array — that still runs, but new code should use the explicit form.
- Return format: `[{ json: { ... } }]`
- Every `catch` block must have `console.log` with error details — never silently swallow errors
- Reference other nodes with `$('Node Name').first().json`
- Code nodes run real JS: optional chaining (`?.`), `try/catch`, and array methods are used throughout `Parse AI Response` and `篩選新項目`. The `?.` restriction below applies to `{{ }}` expressions, not here.
- Luxon `DateTime` is available without import. zh-TW output uses the locale option: `DateTime.fromISO(v).setZone('Asia/Taipei').toFormat('MM月dd日 (cccc) HH:mm', { locale: 'zh-TW' })`
- Degrade, don't throw. The house style on bad input in `SlackToGoogleCalendar` is to `console.log` and return a status item (`{ status: 'error', error: '...' }`) so downstream IF nodes can route it, rather than failing the execution.

### Expression Syntax

- This repo writes expressions without optional chaining — the established pattern is `$json.field && $json.field.prop` (see `Send Error Notification`). Keep it consistent; the `.github/skills/n8n-expression-syntax` docs do not require `?.` either.
- Expression strings start with `=` in the JSON (`"text": "={{ $json.fallbackText }}"`). A value without the leading `=` is a literal.
- Slack messages use `<url|text>` format for links (not Markdown `[text](url)`)
- For timezone-aware formatting: `DateTime.fromISO(value).setZone('Asia/Taipei').toFormat(...)`
- Channel/calendar/database pickers are resource locators, not plain strings:
  `"channelId": { "__rl": true, "mode": "id", "value": "<channel-id>" }`
- IF v2.3 conditions carry a stable `id` per condition, `operator: { type, operation }`, and `options: { version: 2, typeValidation: "loose", caseSensitive: true }`. Preserve that shape when editing conditions; MCP partial updates add the v2.2+ metadata automatically, so let it, rather than hand-writing a partial condition object.

## Cross-Node Contracts

These conventions span multiple nodes and are the main thing to preserve when editing:

### Error routing

In `SlackToGoogleCalendar`, the four nodes on the main path that can fail — `Analyze Message with AI`, `Create Calendar Event`, `Send Success Notification`, `Send Low Confidence Alert` — set `onError: "continueErrorOutput"` and wire their **second** `main` output (index 1) into one shared Slack `Send Error Notification` node. Terminal notification nodes (`Send No Event Reply`) and AI sub-nodes (`Azure OpenAI gpt-5.2`, connected over `ai_languageModel`) do not. When you add a fallible node to this flow's main path, wire its error output into the same sink rather than letting the execution die. The other three workflows have no error routing at all.

### Slack Block Kit messages

A Code node builds the blocks and returns them **as a JSON string**, plus a plain-text fallback:

```javascript
return [{ json: { blocks: JSON.stringify(blocks), fallbackText: '✅ 日曆事件已成功建立！' } }];
```

The Slack node then uses `messageType: "block"`, `blocksUi: "={{ $json.blocks }}"`, `text: "={{ $json.fallbackText }}"`. Keep the stringified form when editing these pairs — every Block Kit message in this repo uses it. Simple alerts skip the builder and use `messageType: "text"` with an inline expression.

### AI analysis chain

`chainLlm` (prompt) + `lmChatAzureOpenAi` (model) connected over the `ai_languageModel` port, then a Code node parses the output:

- The prompt injects "now" via `{{ DateTime.now().setZone('Asia/Taipei').toFormat('yyyy年MM月dd日 (cccc)', { locale: 'zh-TW' }) }}` and demands raw JSON with no surrounding prose.
- The parser still strips fences defensively: `text.replace(/```json\n?|```\n?/g, '').trim()` before `JSON.parse`, because the model does not always comply.
- Model output flows through a status vocabulary the IF nodes switch on: `error`, `no_event`, `high_confidence`, `low_confidence`, with a numeric `confidence`.
- **The 0.7 confidence threshold is duplicated in two places**: the ternary that sets `status` in `Parse AI Response`, and `rightValue: 0.7` on the `Check Confidence Score` IF node. Changing one without the other makes the status field and the routing disagree.

If you change the prompt's JSON shape, update the parser, the IF conditions, and the block builders in the same change.

## Validation & Sync

### Workflow Validation Checklist

After every change:
1. MCP validate with `profile: "strict"`
2. Check `errorCount` = 0 (ignore `"Cannot return primitive values directly"` — MCP static analysis false positive for Code nodes)
3. Check `invalidConnections` = 0
4. Verify `validConnections` matches expected count
5. Sync cloud → local JSON, then run the [structural check](#local-checks)
6. Trigger the workflow on the live instance and read the execution — for `SlackToGoogleCalendar` that means posting real Slack test messages (all-day event, timed event, non-schedule message); for the scheduled flows it means a manual execution

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

- `.mcp.json` (Copilot CLI): `"N8N_API_KEY": "${N8N_API_KEY}"` inside a stdio server's `env` block. **Verify this actually resolves before relying on it.** Searching the CLI bundle (1.0.73) for placeholder-expansion logic found none, and an unexpanded `${...}` is transmitted literally. The failure is silent in the worst way: the server starts and its tools appear normally, and only the tool *call* fails with an auth error. If n8n MCP tools start returning auth failures, check this first.
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

- If the flow has `README.md` / `README.zh-tw.md`, update **both**, and run the doc-drift check in [Local Checks](#local-checks) before you call the change done. `flows/SlackToGoogleCalendar/` was rewritten from its JSON in 1.0.4 and now passes 14/14 in both languages — keep it that way instead of letting it rot again.
- Add a `CHANGELOG.md` entry under a semver heading (`## [1.0.4] - YYYY-MM-DD`) with subsections matching the existing style (`### Enhancements` / `### Fixes` / `### Documentation`).
- The root `README.md` lists workflows; it currently documents only two of the four, so add yours if you create a new flow.

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