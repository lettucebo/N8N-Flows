# Project Guidelines

## Overview

n8n workflow automation repository. All workflows are stored as JSON files under `flows/` and managed via n8n MCP tools + Git.

- **n8n Version**: 2.14.2 (self-hosted Docker)
- **Primary Timezone**: Asia/Taipei
- **Language**: Workflow names and node names in English; prompts and user-facing messages in Traditional Chinese (zh-TW)

## Architecture

```
flows/
  {WorkflowName}/
    {Workflow_Name}.json    # n8n workflow export (single JSON file per workflow)
    README.md               # English documentation
    README.zh-tw.md         # zh-TW documentation
.github/
  skills/                   # Copilot skills for n8n development
```

Each workflow is a self-contained JSON file. No shared code between workflows.

## n8n Workflow Development

### MCP-First Workflow

Always use n8n MCP tools for workflow operations:

1. **Read**: `mcp_n8n-mcp_n8n_get_workflow` (mode: structure/details)
2. **Validate**: `mcp_n8n-mcp_n8n_validate_workflow` (profile: strict)
3. **Update**: `mcp_n8n-mcp_n8n_update_partial_workflow` for node changes, `mcp_n8n-mcp_n8n_update_full_workflow` when adding/removing nodes or changing connections
4. **Sync local**: After cloud update, always sync cloud → local JSON

### Node Naming

- Use descriptive English names (e.g., `Parse AI Response`, `Check Confidence Score`)
- Never use default names like `IF`, `Code`, `HTTP Request1`

### Node Versions

Always use latest supported versions:

| Node Type | Target Version |
|-----------|---------------|
| IF | 2.3 |
| Slack | 2.4 |
| Google Calendar | 1.3 |
| Code | 2 |
| Basic LLM Chain | 1.9 |

### Code Node Conventions

- Use `n8n-nodes-base.code` v2 (not deprecated `function` v1)
- Parameter: `jsCode` + `mode: "runOnceForAllItems"`
- Return format: `[{ json: { ... } }]` (never bare objects or primitives)
- Every `catch` block must have `console.log` with error details — never silently swallow errors
- Reference other nodes with `$('Node Name').first().json`

### Expression Syntax

- n8n expressions do NOT support optional chaining (`?.`) — use `$json.field && $json.field.prop` instead
- Slack messages use `<url|text>` format for links (not Markdown `[text](url)`)
- For timezone-aware formatting: `DateTime.fromISO(value).setZone('Asia/Taipei').toFormat(...)`

### Workflow Validation Checklist

After every change:
1. MCP validate with `profile: "strict"`
2. Check `errorCount` = 0 (ignore `"Cannot return primitive values directly"` — MCP static analysis false positive for Code nodes)
3. Check `invalidConnections` = 0
4. Verify `validConnections` matches expected count
5. Sync cloud → local JSON
6. Provide test messages for manual Slack verification

### Cloud ↔ Local Sync

After updating cloud via MCP, sync to local with:
```powershell
$raw = Get-Content "<mcp-result-file>" -Raw | ConvertFrom-Json
$wf = $raw.data.workflow
$export = [ordered]@{
  name = $wf.name; nodes = $wf.nodes; pinData = [ordered]@{}
  connections = $wf.connections; active = $wf.active
  settings = $wf.settings; versionId = $wf.versionId
  meta = [ordered]@{ templateCredsSetupCompleted = $true; instanceId = "..." }
  id = $wf.id; tags = @()
}
$export | ConvertTo-Json -Depth 20 | Set-Content "<local-json-path>" -Encoding UTF8
```

## Change Process

1. **Plan** — Describe changes with risk assessment, dependency analysis, and rollback strategy
2. **Implement** — Execute in waves (low risk → high risk), validate after each wave
3. **3-Round Verify** — (1) structural review, (2) logic/edge case review, (3) MCP + cross-check
4. **Test** — Provide Slack test messages covering: all-day event, timed event, non-schedule message
5. **Sync** — Cloud → local JSON
6. **Commit** — Conventional commit format: `refactor(WorkflowName): description`

## Skills Reference

This repo has 7 n8n-specific skills under `.github/skills/`. They are auto-loaded by Copilot when relevant:

- `n8n-code-javascript` — Code node JS patterns
- `n8n-code-python` — Code node Python patterns
- `n8n-expression-syntax` — Expression validation
- `n8n-mcp-tools-expert` — MCP tool usage guide
- `n8n-node-configuration` — Node config patterns
- `n8n-validation-expert` — Validation error interpretation
- `n8n-workflow-patterns` — Workflow architecture patterns
