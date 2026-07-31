# Migrating the calendar assistant from Slack to Discord

- **Date**: 2026-07-31
- **Status**: design, awaiting review
- **Supersedes**: nothing. `flows/SlackToGoogleCalendar/` stays live until this is proven.
- **Decision owner**: repo owner. The recommendation below was made autonomously; every alternative is documented so it can be overridden without redoing the research.

## 1. What exists today

`Slack to Google Calendar AI Assistant` (workflow `I2dch7ZKvBvX6GVC`, 35 nodes / 49 connections, active). A human posts in a Slack channel, the bot reacts 👀, reads the thread, sends the text and any images to Azure OpenAI, and writes one Google Calendar event per meeting found, replying with a Block Kit card per event and swapping 👀 for ✅ / 🔄 / ⚠ / ➖.

It touches exactly four Slack Web API endpoints:

| Endpoint | Nodes |
|---|---|
| `conversations.replies` | `Fetch Thread Replies` |
| `chat.postMessage` | 6 card senders |
| `reactions.add` | 6 reaction nodes |
| `reactions.remove` | 2 reaction nodes |

plus `n8n-nodes-base.slackTrigger` for ingestion. 17 of the 35 nodes hold the `slackApi` credential.

Everything else — `Build Azure Payload`, `Analyze With Azure`, `Azure OpenAI gpt-5.2`, most of `Parse AI Response`, `Pair Created Event`, both Google Calendar nodes — is platform-agnostic and carries over unchanged. That is roughly 40% of the logic and effectively all of the hard-won behaviour: year disambiguation, the confidence threshold, multi-event fan-out, duplicate detection, update targeting.

### The one piece of state

There is no database. The bot carries state between turns entirely through Slack message metadata: each success card is posted with

```json
"metadata": { "event_type": "gcal_event_created",
  "event_payload": { "event_id": "...", "calendar_id": "...", "start": "...", "title": "..." } }
```

and `Collect Thread Context` reads those fields back off the bot's own earlier messages (filtered by `app_id`, so another app cannot forge them) to rebuild `ctx.allEvents`. Duplicate detection and "which event does this edit target" both depend on it. **Any Discord design must provide an equivalent, or the assistant loses its memory.**

## 2. Hard constraints discovered

All verified against official documentation and this n8n instance, not assumed.

### 2.1 Discord has no Events API

`MESSAGE_CREATE` is only deliverable over a persistent Gateway WebSocket. Discord's "Interactions Endpoint URL" — the only inbound HTTP path — carries exactly five interaction types: `PING`, `APPLICATION_COMMAND`, `MESSAGE_COMPONENT`, `APPLICATION_COMMAND_AUTOCOMPLETE`, `MODAL_SUBMIT`. A human typing in a channel produces none of them.

> "These two methods are mutually exclusive… the webhook method detailed below does not require a connected client."
> — [Receiving and Responding](https://docs.discord.com/developers/interactions/receiving-and-responding#receiving-an-interaction)

This is the structural difference from Slack and it drives the whole design.

### 2.2 n8n cannot trigger on Discord messages here

`search_nodes` against the n8n MCP documentation server returns exactly two Discord entries: `nodes-base.discord` (v2, action node) and `n8n-nodes-discord-trigger.discordtrigger` (community). `get_node nodes-base.discordTrigger` returns *not found* — there is no built-in trigger.

The community trigger is not usable as things stand: community node installation on this instance has been broken since 2025-11-19. `installed_packages` lists `n8n-nodes-mcp` and `n8n-nodes-firecrawl`, but `/mnt/data/n8n/data/nodes` contains only an empty `package.json` with no `node_modules`, and n8n logs "some packages are missing" at boot.

### 2.3 The built-in Discord node covers most, but not all, of what is needed

`nodes-base.discord` v2 operations: `message` Send / Get / Get Many / Delete / **React with Emoji** / Send and Wait; `channel` CRUD; `member` Get Many / Role Add / Role Remove; plus webhook send. Connection types: Bot Token, OAuth2, Webhook. `message:send` targets "a channel, thread, or member" and supports `embeds` (fields or raw JSON), `files` (binary), and message `flags`.

**There is no reaction-removal operation.** The 👀 → result swap therefore needs a raw HTTP Request node on `DELETE /channels/{channel_id}/messages/{message_id}/reactions/{emoji}/@me`. That is unremarkable here — the current workflow already contains 16 `httpRequest` nodes.

### 2.4 Discord has no invisible message metadata

Evaluated every candidate:

| Mechanism | Capacity | Visible to users | Survives read-back |
|---|---|---|---|
| `embed.fields` | 25 fields, name 256 / value 1024, 6000 total | **yes** | yes |
| `embed.footer.text` | 2048 | **yes** | yes |
| `embed.timestamp` | one ISO 8601 instant | yes, rendered as a localised time | **yes, exact** |
| **component `custom_id`** | **100 chars**, 5 buttons × 5 rows | **no** | **yes** |
| `message.flags` | system-defined bits only | — | not usable |
| `interaction_metadata`, `application_id` | not developer-supplied | — | not usable |
| Second embed + `SUPPRESS_EMBEDS` | large | no | yes, but hides *all* embeds unless the visible card moves to Components v2 |

`custom_id` is the only genuinely hidden field, and 100 characters is enough: a Google Calendar event id is ~26 characters.

## 3. Approaches considered

### Approach A — message context-menu command, plus buttons and modals *(recommended)*

The user writes naturally in the channel, then right-clicks the message → **Apps → 加入行事曆**. Discord POSTs an `APPLICATION_COMMAND` interaction to an n8n Webhook node. The workflow verifies the Ed25519 signature, answers inside 3 seconds, then does the real work and posts the card into a thread on the source message. Edits happen through buttons on the card.

**Pros**
- No new infrastructure. Everything stays inside n8n.
- No `MESSAGE_CONTENT` privileged intent, so no annual reapplication.
- **The metadata problem solves itself**: the card needs buttons anyway, and `custom_id` is hidden.
- Ed25519 verification is mandatory, which closes a real gap — the current Slack webhook accepts forged signatures and still returns 200.
- Interaction endpoints are exempt from the global 50 req/s limit.
- Explicit invocation removes the "is this message even about scheduling?" guess, which is where the low-confidence and suppress branches spend their effort.

**Cons**
- The UX changes: one extra right-click instead of pure passive listening.
- "Reply in the thread to change the event" stops working as-is, because thread replies never reach an HTTP endpoint. Replaced by buttons + a second context-menu command (see §4.6).
- The 3-second acknowledgement deadline is a real design constraint inside n8n.

### Approach B — a Gateway relay service on the same VM

A small always-on process (roughly 100 lines over `discord.js` or a raw WebSocket) subscribes to `MESSAGE_CREATE` and re-shapes each event into the payload the current workflow already expects, POSTing it to an n8n webhook.

**Pros**
- Preserves today's experience exactly: type in the channel, 👀 appears immediately, no extra gesture.
- The n8n workflow stays structurally identical; only the outbound API calls change.
- The relay is stateless and independently restartable.

**Cons**
- Adds a permanently running service to a VM that `AGENTS.md` records as having **no monitoring**, a single copy of every backup and the n8n encryption key, and a 25-minute total outage in July caused by running `docker compose` from the wrong directory.
- Requires the `MESSAGE_CONTENT` privileged intent. Self-serve below 10,000 users as of the 10 June 2026 policy change, but **reapplication is now annual** — a silent expiry would kill ingestion.
- New failure modes to own: reconnect/resume, heartbeats, and the 1000-IDENTIFY-per-24h cap that resets the bot token when exceeded.

### Approach C — repair community nodes and install `n8n-nodes-discord-trigger`

**Pros**
- No extra service; everything inside n8n; today's UX preserved.

**Cons**
- Requires repairing a subsystem that has been broken for eight months, which would also reinstate `n8n-nodes-firecrawl` and `n8n-nodes-mcp`.
- Puts a third-party package in charge of a persistent Gateway connection inside the production n8n process; every n8n restart reconnects.
- Largest blast radius of the three on an instance with the operational history above.

### Recommendation

**Approach A**, for three reasons in order of weight:

1. **It removes the hardest problem instead of working around it.** The absence of hidden metadata is the single biggest gap; A needs buttons anyway, so `custom_id` becomes the natural carrier. B and C must either print the event id where users can see it or adopt Components v2 plus the `SUPPRESS_EMBEDS` trick.
2. **It does not touch the VM.** Given no monitoring, single-copy backups, and a recent self-inflicted outage, adding a daemon costs more than a right-click saves.
3. **It fixes a live security gap as a side effect.** Discord refuses to keep an interactions endpoint that does not reject deliberately invalid signatures, so verification cannot be quietly skipped the way it was on the Slack side.

The trade-off to accept consciously: **passive listening is lost.** If that is non-negotiable, take Approach B. Approach C is not recommended under any circumstance.

## 4. Design (Approach A)

### 4.1 Discord application setup

| Item | Value |
|---|---|
| Bot scopes | `bot`, `applications.commands` |
| Bot permissions | `VIEW_CHANNEL`, `SEND_MESSAGES`, `SEND_MESSAGES_IN_THREADS`, `CREATE_PUBLIC_THREADS`, `READ_MESSAGE_HISTORY`, `ADD_REACTIONS`, `EMBED_LINKS`, `ATTACH_FILES` |
| Privileged intents | **none** |
| Interactions Endpoint URL | the n8n production webhook URL |
| Commands | message command `加入行事曆`; message command `更新行事曆事件` |

Both are `type: 3` (MESSAGE) application commands, registered once via `PUT /applications/{app_id}/guilds/{guild_id}/commands`. Guild-scoped commands appear immediately; global commands take up to an hour to propagate.

### 4.2 Ingestion and the 3-second deadline

```
Discord Interaction Webhook (n8n Webhook, responseMode: responseNode, raw body enabled)
  → Verify Signature        (Code: Ed25519 over timestamp + raw body)
  → Route Interaction       (Switch on interaction type)
      type 1 PING           → Respond {type: 1}
      type 2 COMMAND        → Respond {type: 4, ephemeral 「👀 收到，正在解析…」}  ← under 3s
      type 3 COMPONENT      → Respond {type: 9, modal}   (edit button)
      type 5 MODAL_SUBMIT   → Respond {type: 5, deferred ephemeral}
  → …work continues after the response node…
```

Three points that matter:

- The webhook must expose the **raw** request body. Ed25519 is verified over `timestamp + raw_body`; re-serialising parsed JSON changes byte order and the check fails.
- `Respond to Webhook` is placed immediately after verification and routing, before anything slow. n8n continues executing the branch after responding, so the AI call and calendar writes happen after the deadline has already been met.
- Answering with `type: 4` and an ephemeral message rather than `type: 5` deferral avoids a spinner that must later be filled in, and keeps the 15-minute interaction token free for a final follow-up.

Signature verification uses `crypto.verify('ed25519', …)` from Node's standard library, which is available inside a Code node without any import. **Discord periodically sends deliberately invalid signatures and will remove the endpoint if they are accepted.**

### 4.3 What the interaction payload provides

For a message command, `data.resolved.messages[data.target_id]` contains the full target message: `content`, `attachments[]` with signed CDN `url`, `author`, `id`, `timestamp`. `channel_id`, `guild_id` and `member.user.id` (the invoker) sit at the top level.

> **This is the single assumption the whole approach rests on.** Discord documents `MESSAGE_CONTENT` as not required for interaction payloads, but that must be proven empirically in Phase 1 before any further work. If `content` arrives empty, Approach A collapses and the fallback is Approach B.

### 4.4 Threads

Discord threads are channels whose id equals the source message id, so:

1. If the resolved message already has a `thread`, use it.
2. Otherwise `POST /channels/{channel_id}/messages/{message_id}/threads` with `{ name, auto_archive_duration: 1440 }`.
3. Post cards with `POST /channels/{thread_id}/messages`.

Thread history — the `conversations.replies` replacement — is `GET /channels/{thread_id}/messages?limit=100`, which returns **newest first** and must be reversed. More than 100 messages means paginating with `before`; the current `threadTruncated` flag maps onto "a full page came back".

A message can only ever have one thread, so step 1 and 2 are idempotent together.

### 4.5 The metadata replacement

Each success card is an embed plus one action row:

| Carrier | Content | Replaces |
|---|---|---|
| `embed.timestamp` | event start, ISO 8601, exact round-trip | `event_payload.start` |
| `embed.fields[0].value` | event title | `event_payload.title` |
| button `custom_id` | `gc1:{eventId}` | `event_payload.event_id` |
| workflow constant | calendar id | `event_payload.calendar_id` |
| `author.id === BOT_USER_ID && author.bot === true` | provenance | Slack's `app_id` check |

`gc1:` is a format version prefix so the reader can evolve without misparsing old cards. `calendar_id` moves to a workflow constant because it is invariant here; the existing `pickCalendarId` fallback and `calendarWarning` behaviour are preserved.

The provenance check is *stronger* than the Slack original: only the bot itself can author a message as the bot, whereas Slack's `app_id` was a field on a message the reader had to trust.

### 4.6 The update path

Two routes, because one alone is worse than what Slack offered:

1. **Buttons on the card** — 「修改時間」 opens a modal (`type: 9`) with a free-text field. The submitted text goes through the same AI chain with `intent: update` and the event id taken from `custom_id`. This is *better* than Slack's flow: it is unambiguous about which event is being edited, which is precisely the failure mode the `updateAmbiguous` logic exists to mitigate.
2. **A second context-menu command** 「更新行事曆事件」 — the user writes a correction in the thread as they do today, then right-clicks it. Thread history supplies context exactly as now, and existing events are recovered from the cards' `custom_id` values.

### 4.7 Reactions

Same behaviour, different transport. Emoji must be percent-encoded in the path.

| Meaning | Slack name | Discord | Encoded |
|---|---|---|---|
| seen | `eyes` | 👀 | `%F0%9F%91%80` |
| created | `white_check_mark` | ✅ | `%E2%9C%85` |
| updated | `arrows_counterclockwise` | 🔄 | `%F0%9F%94%84` |
| notice | `heavy_minus_sign` | ➖ | `%E2%9E%96` |
| error | `warning` | ⚠️ | `%E2%9A%A0%EF%B8%8F` |

Add: `PUT /channels/{channel_id}/messages/{message_id}/reactions/{emoji}/@me`. Remove: the same path with `DELETE`. Both return `204 No Content`, so the existing "only `ok === true` counts as delivered" check becomes "only `204` counts".

### 4.8 Images

Attachments arrive as signed CDN URLs (`?ex=…&is=…&hm=…`) that need **no** Authorization header and expire. The existing download-and-aggregate path works unchanged apart from the field names: `files[].url_private` → `attachments[].url`, and `mimetype` → `content_type`. Fetch promptly; do not persist the URLs.

When passing an image URL *into* a Discord embed, strip the query parameters and let Discord refresh the signature itself.

### 4.9 Card formatting

Block Kit → embeds. Limits: 10 embeds per message, 25 fields each, title 256, description 4096, field value 1024, footer 2048, **6000 characters combined**. The current cards are far below all of these.

Link syntax changes from `<url|text>` to `[text](url)`, and — important — markdown links only render **inside embeds**, not in a plain `content` string. Every card here is an embed, so this is a formatting change rather than a behavioural one.

Components v2 is deliberately not adopted: the `IS_COMPONENTS_V2` flag cannot be removed from a message once set, and nothing here needs it.

### 4.10 Node-level migration map

| Today | Becomes | Notes |
|---|---|---|
| `Slack Message Trigger` | `Discord Interaction Webhook` + `Verify Signature` + `Route Interaction` + `Respond to Interaction` | 1 node → 4 |
| `Filter Valid Messages` | folded into `Route Interaction` | bot filtering by `author.bot` / `webhook_id` |
| `Fetch Thread Replies` | `Fetch Thread Messages` (`GET /channels/{thread_id}/messages`) | reverse order; paginate with `before` |
| `Collect Thread Context` | same node, rewritten reader | `metadata.event_payload` → `custom_id` + `embed.timestamp` + field |
| `Should Process` | simplified | explicit invocation removes most gates |
| `Has Images` / `Split Images` / `Download Image` / `Aggregate Images` | unchanged apart from field names | `url_private` → `url` |
| `Build Azure Payload` / `Analyze With Azure` / `Azure OpenAI gpt-5.2` | **unchanged** | platform-agnostic |
| `Parse AI Response` | unchanged except the ctx field names it reads | keeps fan-out, year logic, dedupe |
| `Route Outcome` | unchanged | |
| `Create Calendar Event` / `Update Calendar Event` / `Pair Created Event` | **unchanged** | |
| 6 × `Send * Card` | 6 × Discord message send with embed + action row | Block Kit → embeds |
| 3 × `Verify * Delivery` | same shape, success test becomes HTTP 200/204 | keeps the per-item iteration |
| 8 × reaction nodes | `PUT`/`DELETE` reaction endpoints | no node support for removal |
| `Normalize Error` / `Send Error Card` | unchanged logic, Discord transport | `$prevNode.name` stage detection is unaffected |

Roughly 14 nodes carry over untouched, 12 need field renames only, and 9 are genuinely new.

### 4.11 Error handling

Unchanged in shape: fallible nodes on the main path set `onError: continueErrorOutput` into a single `Normalize Error` → `Send Error Card` sink, and `Normalize Error` continues to derive the failing stage from `$prevNode.name`. The node-name list inside it must be updated for the renamed Discord nodes, including `Pair Created Event`, or `calendarWritten` will mis-report and the error card will tell the user to retry after events were already created.

Discord-specific additions: treat HTTP 429 by honouring `retry_after` rather than retrying blindly — more than 10,000 invalid requests in 10 minutes triggers a Cloudflare IP ban.

## 5. Delivery plan

| Phase | Goal | Done when |
|---|---|---|
| 0 | Discord app, bot invite, permissions, both commands registered | commands visible in the right-click menu |
| **1** | **Webhook + Ed25519 + PING + ephemeral ACK** | **Discord accepts the endpoint; a real invocation returns inside 3 s; the resolved message carries non-empty `content` and attachments** |
| 2 | Thread creation, embed card, reactions | a card lands in a thread and 👀 becomes ✅ |
| 3 | AI chain transplanted verbatim | a single-event message produces one calendar event |
| 4 | `custom_id` metadata written and read back | a second invocation in the same thread detects the existing event |
| 5 | Update via button/modal and via the second command | an event is edited, not duplicated |
| 6 | Error routing, multi-event fan-out, docs, CHANGELOG | MCP strict clean; both READMEs regenerated |

**Phase 1 is a go/no-go gate.** If the resolved message content is empty without the `MESSAGE_CONTENT` intent, stop and switch to Approach B rather than working around it.

## 6. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Interaction payload omits message content without the privileged intent | low | fatal to Approach A | Phase 1 gate; fall back to Approach B |
| n8n cannot answer within 3 s under cold start | medium | interaction token invalidated, user sees a failure | respond before any slow node; measure in Phase 1; if marginal, a static 200 from a front node |
| `custom_id` outgrows 100 characters | low | metadata truncated | only the event id is stored (~26 chars); version prefix allows a move to `embed.footer` |
| Ed25519 verification wrong, endpoint removed by Discord | medium | ingestion stops | test with deliberately invalid signatures in Phase 1, exactly as Discord does |
| CDN URL expires before download | low | image lost | download immediately; existing `imagesAllFailed` path already degrades gracefully |
| Rate limiting on multi-event fan-out (up to 5 cards + reactions at once) | medium | 429s | per-channel buckets; honour `retry_after`; never blind-retry |
| Slack and Discord flows diverge as bugs are fixed in one | **high** | the platform-agnostic logic drifts | keep node names and code identical wherever the logic is shared; note it in `AGENTS.md` |

## 7. Explicitly out of scope

- Voice, slash commands with typed arguments, scheduled reminders, multi-guild support.
- Migrating existing Slack thread state. Events already in the calendar stay there; Discord starts with an empty memory.
- Retiring the Slack workflow. It stays active until the Discord one has run for a week.

## 8. Open decisions for the reviewer

1. **Approach A vs B** — A was chosen autonomously. B is the answer if passive listening matters more than operational simplicity.
2. **Command names** — `加入行事曆` / `更新行事曆事件` are placeholders.
3. **Whether the event id may be visible.** If it may, `embed.footer` replaces `custom_id` and the buttons become optional.
4. **Whether the Slack workflow is eventually deleted or kept as a second front end.** Keeping both doubles the maintenance surface described in the risk table.
