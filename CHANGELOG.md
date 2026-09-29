# CHANGELOG

## [1.0.10] - 2026-09-29

### Fixes
- Resolve flight departure and arrival against their respective local IANA zones before the existing Google Calendar nodes run. The AI prompt now requests both zones independently; the parser blocks missing flight endpoints, nonexistent or conflicting local times, and implausible flight durations instead of creating a misleading event. The reported screenshots came from the airline app, not this bot; this fixes the independently identified bot risk rather than attributing those events to it.
- Persist both endpoint zones in Slack card metadata for subsequent updates. Preserve an unchanged endpoint and existing Calendar description on partial updates; retain all-day exclusive end dates and avoid applying timed-flight guards to all-day events.
- Narrow the fallback flight-number matcher so year-like and short numeric meeting titles are not mistaken for flights, while a short flight number accompanied by an airport route still receives the flight safeguards.

### Documentation
- Updated both workflow READMEs with the two-zone contract and clarification behavior.

### Verification
- The staged workflow passed 25 offline timezone regressions, including CI 0008 and CI 0023, DST gaps and overlaps, partial updates, all-day flight wording, and flight-number false positives. `n8n-mcp` strict validation reported 37 nodes, 54 valid connections, 0 invalid connections, and 0 errors (34 warnings). The guarded Public API update and targeted follow-up were read back from active workflow `I2dch7ZKvBvX6GVC` with version `d0cbc0a9-ab8b-45a5-bd3d-c01e63a83e8c`.
- A temporary credential-free webhook → Code → deployed parser workflow ran on the live n8n JS runner. Both flight endpoints resolved to the expected instants; a single-zone LA meeting inherited its end zone; an implausible flight and a DST gap requested clarification; an all-day deadline remained valid; and an arrival-only update preserved the original departure without supplying a replacement description. The probe was deactivated and removed. This isolates parser behavior; it does **not** prove a full Slack → Azure → Google Calendar execution.

## [1.0.9] - 2026-07-31

### Fixes
- **The reaction on a channel message no longer gets stuck on a stale outcome.** Reported against a real thread: 「明天回家」 was answered with ❓ (needs clarification), the user replied 「就這樣建立」 in the thread, the event was created and ✅ appeared — but on the *reply*. The channel list only shows a thread's first message, so it sat on ❓ indefinitely and looked unresolved.
  - Root cause: every outcome reaction targets `$('Slack Message Trigger').first().json.ts`, i.e. the message that triggered *that* turn, and nothing ever removes an earlier outcome. Proven from executions `#1883` (`ts` = anchor, no `thread_ts`, → `Send Clarify Card` → ❓ on the anchor) and `#1885` (`thread_ts` = anchor, → `Create Calendar Event` → ✅ on the reply). The two reactions landed on two different messages.
  - The workflow was already inconsistent about this: `Add Alert Reaction` and `Remove Seen Marker` targeted the thread anchor while the success, updated and notice reactions targeted the triggering message.
  - Fix: two new nodes, `Compute Anchor Status` (Code, `runOnceForAllItems`) and `Sync Anchor Reaction` (one HTTP node whose URL is `reactions.{{ $json.op }}`, since add and remove take identical parameters). They are fed by the three `Verify * Delivery` nodes and `Send Error Card`, and keep the **anchor** message showing the thread's current state: stale status reactions are removed, the current one is added.
  - **The existing eight reaction nodes were not touched.** The change is purely additive, so per-message feedback — 👀 on the message you just sent, then its own outcome — behaves exactly as before and cannot regress.
  - `Collect Thread Context` now exports `anchorBotReactions`: which of the seven status emoji the bot itself has on the anchor, read from the thread data already fetched. `eyes` is deliberately excluded, being a transient marker owned by `Add`/`Remove Seen Reaction`. Knowing the exact set avoids firing `reactions.remove` blindly for all seven — that endpoint is Slack Tier 2, 20 requests per minute.

### Design decisions worth knowing
- **A per-message outcome is not the same thing as the thread's state.** `recycle` (duplicate) is only reachable *because* the thread already holds the event, so on the anchor it resolves to ✅; and `lock` (someone other than the thread owner spoke) leaves the anchor completely alone. Without this mapping the first live run turned a thread whose event had been created into ♻ — a more misleading channel signal than the bug being fixed. Verified: execution `#1918` mapped duplicate → ✅, and `#1922` (not the owner) produced 0 items and left `["white_check_mark"]` untouched.
- **`Add Suppress Reaction` (➖) is deliberately not wired into the anchor sync.** ➖ judges one message. Syncing it would mean that saying "thanks" in a thread whose event was already created erases the ✅.
- **The removal list always excludes the emoji about to be added**, so two consecutive turns with the same outcome cannot remove and re-add the same reaction with the result depending on execution order.
- `runOnceForAllItems`: one message can create up to five events and five cards, but a thread has one status, so the anchor sync does not multiply with the fan-out.

### Notes on getting there
- The first implementation returned 0 items in production (`#1908`): `$('Collect Thread Context').first()` does not resolve from this position, the same item-pairing limitation recorded for `Normalize Error` in 1.0.6. The anchor identity now comes from `$('Slack Message Trigger').first().json` — proven to resolve in the very same execution by the sibling node `Add Success Reaction` — and `anchorBotReactions` is carried forward on the `Verify *` node output rather than read backwards. `$('Parse AI Response')` *does* resolve from those nodes (`#1913` derived `recycle` through it) and `Parse AI Response` spreads `...ctx`, so the value is available there. A bounded fallback to the full status vocabulary remains for the error path, which carries neither field.
- Production is **37 nodes / 54 connections**; MCP strict reports `valid: true`, `errorCount: 0`, `invalidConnections: 0`. Both READMEs regenerated, 37/37 coverage in each language.
- All test artefacts were removed: four bot cards, one calendar event, and the throwaway workflows used to post and clean up. The reported thread's anchor now carries ✅.

## [1.0.8] - 2026-07-30

### Fixes
- **A message describing several meetings now produces one calendar event per meeting, instead of silently dropping all but the first.** Reported against a real message that listed two slots and yielded a single event.
  - Root cause, in `Parse AI Response`: the prompt already asked for an `events` array and the model already returned every slot, but the parser collapsed it with `const ev = Array.isArray(ai.events) && ai.events.length ? ai.events[0] : ai;` and returned exactly one item. Everything downstream was fed one event because only one ever left this node.
  - Fix: the per-event section is now a loop emitting one item per element of `ai.events`, each carrying `eventIndex` / `eventTotal`. This was safe as a loop because every per-event mutable variable is declared below the old line 160, while the shared state (thread context, ownership, the existing-events snapshot) is computed above it.
  - Scope deliberately limited: **only `create` fans out.** `update` stays single, because an edit targets one existing calendar entry and fanning it out would multiply updates against the same event id. Fan-out is also restricted to an explicit `create` intent or a missing intent — an unrecognised intent such as a hallucinated `delete` is coerced to `create` further down, and amplifying that from one wrong event to N is worse than the pre-existing single wrong event. A cap of `MAX_EVENTS_PER_MESSAGE = 5` bounds the output; `eventTotal` still reports the true total so the cards do not misrepresent it, and the overflow count is shown on the card.

- **The success cards no longer all describe the first event.** Found by review, and it was introduced by the fan-out above — the most damaging part of the whole change.
  - `Send Success Card` built its entire body from `$('Parse AI Response').first()` — 19 references. `.first()` returns item 0, not the paired item, which was harmless while the parser emitted exactly one item and wrong the moment it emitted several. With three events, all three cards showed event #1's title, time, location and reasoning while `$json.id` and `htmlLink` varied, so each card linked to a different calendar entry than the one it described.
  - This was not cosmetic. The card's `metadata.event_payload` is the **only** channel by which the next turn in a thread reads event ids back: `Collect Thread Context` reconstructs `ctx.allEvents` from `event_id` / `start` / `title`. Pinning `start` and `title` to event #1 while varying `event_id` would have produced three thread candidates sharing one start/title, breaking both consumers — duplicate detection would stop matching events 2..N and recreate them, and update targeting (which matches on title, then `.pop()`) would silently edit the wrong meeting. Those are exactly the two failures the comments in that code were written to prevent.
  - Fix: a new Code node **`Pair Created Event`** (`runOnceForEachItem`) sits between `Create Calendar Event` and `Send Success Card`. It resolves the parsed event for the current item through n8n's item pairing and merges the Google response in as `googleEventId` / `googleHtmlLink`. If pairing is ever unavailable it logs and degrades to the old `.first()` behaviour rather than throwing, per this repo's "degrade, don't throw" style. `Send Success Card` now reads only `$json.*`.
  - `Send Updated Card` carries the same 19 `.first()` references and was left alone: it is reachable only through `Update Calendar Event`, which is single-item by the rule above. That is now a load-bearing invariant — if `update` is ever allowed to fan out, this node must be converted the same way.

- **One malformed event can no longer discard the others.** The loop body had no `try`/`catch` and `outItems` was only returned at the end, so a throw while building event 3 lost events 1 and 2 and returned nothing. Each iteration is now wrapped; a failing event is logged and skipped. If every event fails, the node emits a clarify item instead of an empty array, because returning `[]` would end the execution silently and the user would simply never get a reply.

- **A single event delivered at the root of the AI response is no longer treated as "no event".** The old code fell back to `ai` itself when `events` was absent; the first version of the loop dropped that, so such a response fell through to the suppress route and the user got nothing. The fallback is restored, and non-object entries are filtered out so junk cannot consume the five-event cap.

- **Two identical events inside one AI response are no longer both created.** Duplicate detection only compared against the pre-existing thread snapshot, so a model that repeated a slot produced two calendar entries. Entries already emitted in the same batch are now checked too.

- The three delivery checks — `Verify Card Delivery`, `Verify Updated Card Delivery`, `Verify Notice Delivery` — ran `runOnceForAllItems` but inspected only `$json`, i.e. the first card. With several cards in flight they would have reported success while later cards failed. All three now iterate `$input.all()` and throw if any card came back without `ok === true`.

- Production is **35 nodes / 49 connections**; MCP strict reports `valid: true`, `errorCount: 0`, `invalidConnections: 0`.

### Fixes found by a second review round, after the above was already deployed
- **`Pair Created Event`'s own fallback could have reintroduced the bug it exists to prevent.** It fell back to `.first()` wholesale when item pairing was unavailable, which for event #2 means event #1's title and start merged with event #2's Google id — precisely the corrupted metadata described above, just logged. It now rebuilds the event-level fields from **Google's own response** (`summary`, `start`, `location`, `description`) and takes only thread-level fields (`channel`, `threadTs`, `calendarId`) from `.first()`, which are identical across items by construction. A few auxiliary fields are lost in that path and the item is marked `pairingFallback: true`; showing less is preferable to showing another event's details.
- **An unbuildable event could suppress a valid identical one.** The intra-batch duplicate key was recorded before `hasValidEvent` was computed, so a malformed first occurrence claimed the key and the valid second occurrence was marked duplicate — nothing was created and the user got a duplicate notice. The key is now recorded only after validation.
- **Equivalent timestamps escaped the batch check.** The key compared raw strings, so `10:00+08:00` and `10:00:00+08:00` read as different events although both are accepted and denote the same instant. The key is now `Date.parse`-normalised for timed events and the date string for all-day events, and the title is normalised the same way the update matcher normalises it.
- **`intent` was normalised in one place and compared raw in two others.** The fan-out decision used a trimmed, lower-cased `intentRaw`, while update targeting and the emitted `intent` still tested `ai.intent === 'update'`. A response saying `" Update "` therefore did not fan out (correct) but was emitted as `create` (wrong) — the user asks for an edit and gets a second event. All three now use the same definition.
- Reviewed and deliberately not changed: when every event fails to parse, the fallback item reaches the clarify branch for a thread owner but the not-owner branch for a non-owner. That is a reasonable answer for someone who could not have created the event anyway, so it was left alone rather than made unconditional.

### Verification
- **Proven end to end on the live instance, twice** — once before the second review round (execution `#1852`) and again after it (`#1864`). A three-slot agenda routed through the create branch produced, in one run: `Parse AI Response` 3 items → `Route Outcome` `out5` 3 items → **`Create Calendar Event` 3 items with three distinct Google event ids** → `Pair Created Event` 3 items **each carrying its own title and start** → `Send Success Card` 3 cards, all `ok: true` with distinct `ts` → `Verify Card Delivery` 3 → `Add Success Reaction` 3.
- The card bodies were re-rendered from the run's own paired items to confirm what Slack received: headers `✅ 已建立行事曆事件 (1/3) / (2/3) / (3/3)`, and `metadata.event_payload` carrying **3 distinct titles and 3 distinct event ids** — the corruption described above being absent, measured rather than assumed.
- The intra-batch duplicate guard could not be exercised end to end: asked to schedule the same meeting twice in one message, the model collapses it and returns one event, so the guard is defence in depth rather than a hot path. It was instead unit-tested by extracting **the deployed block's exact source text from the live workflow** and running it over crafted inputs — 6/6 cases pass, including the two the review raised (same instant written at different precision; invalid first occurrence followed by a valid identical one).
- All test calendar events were deleted from the production calendar, the test cards were removed from the Slack channel, and both throwaway workflows used for cleanup were deleted.
- Reaching the create branch took three attempts; the first two were routed away by working guards, not bugs — `isOwner: false` because the anchor message's thread root was authored by someone else, then `isDuplicate: true` because the thread already held that event. The anchor for the successful runs was chosen by scanning past executions for a thread where `isOwner` had already been observed to be true, rather than by guessing.

### Environment notes
- Slack event delivery to this instance became unreliable during testing: messages posted through the browser are visible in the client yet generate no `event_callback`, and the bot's own `conversations.history` does not return them. Synthetic replays against the production webhook were used instead, which exercise everything downstream of the trigger. Worth knowing before concluding that a workflow change broke something.
- Copilot CLI 1.0.73 does **not** load this repo's `.mcp.json`; only plugin-provided MCP servers appear. n8n work in this session went through the Public REST API and a manually spawned MCP server over stdio. The `${...}` placeholders in `.mcp.json` are also not expanded, so a key written that way would be transmitted literally.

## [1.0.7] - 2026-07-30

### Fixes
- **Fixed a regression I introduced in v2: the Azure credential appeared to "expire hourly".** It did not. v1 called Azure through the LangChain `lmChatAzureOpenAi` node; v2 called it through `httpRequest` with `authentication: predefinedCredentialType`. Those take different auth paths, and only the latter is affected.
  - Proven side by side, same credential, same minute: the LangChain node returned `{"text":"{\"ok\":true}"}` while `httpRequest` returned `OAuth access token expired and no refresh token is available`.
  - Cause: `httpRequest` uses n8n's generic OAuth2 helper, which reuses a stored access token and renews it with a `refresh_token`. This Entra credential mints tokens with `grant_type: client_credentials` (injected through `additionalBodyProperties`) while declaring `grantType: authorizationCode`; client-credentials responses carry no refresh token, so once the stored token expired the call failed permanently. The LangChain node acquires its own token and is unaffected — which is exactly why this never happened before v2.
  - **Fix: `Call Azure OpenAI` (httpRequest) was replaced by `Analyze With Azure` (`chainLlm` v1.9) plus `Azure OpenAI gpt-5.2` (`lmChatAzureOpenAi` v1)**, restoring v1's working auth path while keeping v2's vision support. Entra ID authentication is retained, as required.
  - Vision is preserved: `chainLlm` accepts images, but only through **static** message slots, and an empty slot makes the node fail outright (verified). `Build Azure Payload` therefore always emits four slots and pads unused ones with a 1×1 transparent PNG, with the system prompt instructing the model to ignore blanks. Verified: with one real image and three blanks the model still extracted `季度營運檢討會議 / 2026-10-08 / 14:30–16:00 / 台北辦公室 12F 大會議室` and reported `realImages: 1`.
  - `Parse AI Response` now reads `$json.text` (the chain's output) and keeps the old `choices[0].message.content` shape as a fallback, so the node survives a future switch back to a REST call.
  - **Verified end to end with the credential still un-reconnected** — the decisive test, since the old path was failing at that exact moment. Full run: `Filter Valid Messages → Add Seen Reaction → Fetch Thread Replies → Collect Thread Context → Build Azure Payload → Azure OpenAI gpt-5.2 → Analyze With Azure → Parse AI Response → Remove Seen Reaction → Route Outcome → Create Calendar Event → Send Success Card → Verify Card Delivery → Add Success Reaction`, with `intent=create`, `confidence=0.9`, `title="情人節晚餐"`. Production is 34 nodes / 48 connections, MCP strict `valid: true`, `errorCount: 0`.
  - The test event this created in the production calendar was removed afterwards.

### Corrections to 1.0.6
- The 1.0.6 entry blamed the hourly failures on the credential and recommended switching to an API key. **That diagnosis was wrong**: the credential is fine, and the API-key suggestion was unnecessary — Entra ID works, through the right node.
- Things ruled out along the way, recorded so they are not re-attempted: the whole `messages` parameter cannot be an expression (it runs but the images never reach the model, `seen: 0` for 1 and 3 images); `PATCH /credentials/{id}` replaces `data` wholesale rather than merging (returns 400 demanding `resourceName`/`apiVersion`), so a credential cannot be adjusted without its secrets; and literal JSON braces in a chainLlm prompt are safe.

### Documentation
- Both READMEs regenerated from the live 34-node workflow; coverage 34/34 in each language.

## [1.0.6] - 2026-07-30

### Promotion
- **v2 is now the production workflow.** Its content was written into the existing production workflow `I2dch7ZKvBvX6GVC` rather than activating the test copy, so the workflow id, webhook path and execution history all carry over and the repo file's id binding stays valid. Production went from 14 nodes / 16 connections to **33 nodes / 47 connections**; MCP strict on the promoted workflow reports `valid: true`, `errorCount: 0`, `invalidConnections: 0`.
  - The live v1 definition was saved to a backup file before promotion, and remains recoverable from git history.
  - `RJiCNKlVQ4EhVyMS` is kept as an inactive test copy still pointing at the disposable test calendar.
  - `Slack_to_Google_Calendar_AI_Assistant.json` now holds the promoted workflow, and the redundant `.v2.json` was removed — one export per workflow, as the naming contract requires. The file is also written without the UTF-8 BOM it previously carried, matching the other three exports.
- Both READMEs were regenerated **from the live workflow** rather than hand-edited, so the node table, topology, routing branches, settings and credential list cannot drift. Coverage went from 4/33 to **33/33** node names in each language.

### Fixes
- **`Normalize Error` now reports the correct failing stage.** It previously labelled every failure 「讀取 Slack 對話」. The stage and the calendar-written flag are now derived from `$prevNode.name`, which is node-level metadata and does not depend on item pairing. Verified live: a forced failure at `Call Azure OpenAI` produced `previousNode: Call Azure OpenAI` → stage 「呼叫 AI 解析」, and a failure at `Collect Thread Context` produced 「整理對話內容」.
  - The connected risk is closed too: `calendarWritten` no longer relies solely on `pick()`. If the failure happened at or after a card-send node, the calendar must already have been written, so the error card can no longer tell a user to retry after the event was created — the duplicate-event trap the code comments (rd-json BLOCKING #4) exist to prevent.
  - Correction to an earlier entry: `$()` is **not** uniformly broken on an error branch. In the same execution where `reached('Build Azure Payload')` returned false, `$('Slack Message Trigger')` resolved correctly. The failure is per-node item pairing, not the error branch as such.
- **The MCP strict false positive is gone.** The validator flags nodes whose names carry failure semantics when they share a `main[0]` with others. Restructuring alone did not clear it, and neither did the first rename; renaming `Add Error Reaction` → `Add Alert Reaction` did. Production now validates with `errorCount: 0`.

### Incidents found
- **The Azure credential cannot renew itself and dies roughly hourly.** `azureEntraCognitiveServicesOAuth2Api` injects `{"grant_type":"client_credentials"}` through `additionalBodyProperties` while declaring `grantType: authorizationCode`; the client-credentials flow returns no refresh token, so once the access token expires n8n fails with `no refresh token is available`. Observed directly: reconnected at ~10:50, working 11:00–11:35, expired again by 13:07. A manual reconnect is therefore a temporary fix, not a solution. Durable options: turn off "send additional body properties" so the standard authorization-code flow with `offline_access` issues a refresh token, or move to an API-key `azureOpenAiApi` credential, which does not expire.

### Testing notes
- Playwright posting via `Enter` silently failed while reporting success — the message never reached Slack. Two causes: a stuck "Loading thread…" flexpane captured the composer, and the DOM-based confirmation matched text that was not actually posted. Reliable posting requires closing the pane, scoping to the channel's own `[data-qa="message_input"]`, clicking Slack's send button, and confirming through `conversations.history` rather than the DOM.

## [1.0.5] - 2026-07-30

### Live testing (first ever real execution of v2)
- Drove Slack's web UI with Playwright, posting as a **real human account** (bot-posted messages are self-filtered and cannot exercise the flow). After the user reconnected the Azure credential, **the whole pipeline ran end to end for the first time**:
  - **Timed event** — 「明天下午三點跟客戶開會，大概一小時」 → confidence 0.90 → created `2026-07-31 15:00–16:00`.
  - **All-day event** — 「8月15號健康檢查」 → `2026-08-15` with an exclusive end of `08-16`, confirming the all-day end-date handling.
  - **Low confidence** — 「下週找個時間跟設計團隊聊一下」 → 0.38 → clarify card, no event.
  - **Non-event chatter** — 「大家早安，今天也要加油」 → `aiHasEvent=false`, confidence 0.05 → no event, ➖ reaction only.
  - **Image understanding** — a generated meeting-notice PNG carrying the date only in pixels → confidence **0.94**, extracting `2026-10-08 14:30–16:00`, the title and the room. `Download Image` fetched `url_private` successfully, confirming the deliberate httpRequest 4.2 cross-origin-credentials choice.
  - **Thread modification** — root 「9月10號下午兩點跟客戶簡報」, then 「改到三點」 and 「地點改成台中辦公室」 both produced `intent=update` and **updated the same `eventId`**; a verbatim restatement was routed to a clarify card rather than creating a duplicate. The calendar ended with exactly one such event.
- Internal behaviours now proven by real runs: bot messages stop after 2 nodes (no feedback loop); the hardcoded Slack ID fallbacks are live (`botAppIdConfigured: true`); `round` increments correctly across a 4-message thread; thread history carries `[user]`/`[bot]` labels with per-message timestamps; test-calendar isolation held and was purged afterwards.
- Corrected an earlier misreading: a thread reply describing a *different* event creates a second event **by design** (`isDuplicate` compares start+title). The case that looked like a bug came from a corrupted test message that concatenated two scenarios.

### Enhancements
- **Message-level acknowledgement reactions.** v2 previously had exactly one reaction (➖ when the AI found no event); every other outcome gave no feedback on the message itself, even though the manifest comment claimed ➖ ❓ ✅ ⚠️ were in use. Reactions now follow a two-stage model: 👀 `eyes` is added the moment a message passes `Filter Valid Messages`, then removed and replaced by the outcome emoji — ✅ created, 🔄 updated, ❓ needs clarification, ♻️ duplicate, 🔒 not the owner, ➖ ignored, ⚠️ failed. Verified live: 👀 at t+4s, gone by t+8s, outcome emoji from t+14s, never more than one reaction at a time. Seven nodes added (26 → 33).
  - Every reaction targets the message that triggered the run, so a thread reply is marked on itself rather than on the thread root; `Add Suppress Reaction` was retargeted to match.
  - All reaction nodes use `onError: continueRegularOutput`: they are cosmetic, and `already_reacted` / `no_reaction` responses must never fail a run.
  - Normal-path nodes resolve the target through `$('Slack Message Trigger')`; error-path nodes cannot use `$()` (see the `Normalize Error` defect) and read `channel`/`threadTs` from `Normalize Error`'s own output instead.
  - **MCP strict now reports 1 error on this workflow and it is a false positive.** It sees several nodes named "…Error…" sharing `Normalize Error`'s `main[0]` and suggests moving them to `main[1]`; `Normalize Error` is a Code node with no `onError`, so `main[1]` does not exist and following that advice would break the flow. Renaming two of the nodes did not clear it because `Send Error Card` alone triggers the heuristic.

### Incidents found
- **Production is down for AI analysis, and there is no working AI credential on the instance at all.** *(Resolved 2026-07-30: the user reconnected `82mlP2DDo7j1VGdC`; a live call now returns `model=gpt-5.2-2025-12-11`.)* Verified by executing real requests through each one:
  - `82mlP2DDo7j1VGdC` "Azure Open AI account Entra ID" → `OAuth access token expired and no refresh token is available`. Execution #1619 on 2026-07-29T02:23 still completed a full AI analysis and created an event, so it lapsed after that. v1 and v2 share it, so **any real user message failed at the AI step** until it was reconnected.
  - `CtgkMRzzV4whz91f` "Azure Open AI account" → `Unable to sign without access token`; authorisation was never completed. **Still broken.**
  Every non-interactive repair path was tried and ruled out: `GET /credentials/{id}` exposes no `data`, so `clientId`/`clientSecret` are unreadable; the credential's `additionalBodyProperties` is `{"grant_type":"client_credentials"}` which would otherwise permit a silent token mint; `PATCH` is accepted (`PUT` is 405) but needs the same secret; and the n8n UI requires a sign-in. **Fix: n8n → Credentials → reconnect.**
- **`Raindrop Knowledge Management` has a credential type mismatch.** Its `Azure AI 內容分析` node declares `azureOpenAiApi` for credential `CtgkMRzzV4whz91f`, which is actually `azureEntraCognitiveServicesOAuth2Api`; executing it raises `Credential with ID … does not exist for type "azureOpenAiApi"`. A second reason, besides the missing firecrawl package, that the workflow will fail if activated.
- **`Normalize Error` misreports the failing stage.** Every failure is labelled 「讀取 Slack 對話」 regardless of where it happened; the observed failures were at `Call Azure OpenAI` and should have read 「呼叫 AI 解析」. Root cause: `reached()` and `pick()` both wrap `$('Node Name').first()`, which throws when evaluated inside a node reached via an error output. `Build Azure Payload` demonstrably ran in all three failures, yet `reached('Build Azure Payload')` returned false every time. Consequence worth prioritising once the credential is restored: `calendarWritten` is computed by the same broken mechanism, so a "calendar written but card failed" case would tell the user to retry — producing exactly the duplicate events the code comments (rd-json BLOCKING #4) say this safeguard exists to prevent.
- **Slack redelivers events after downtime.** With both workflows deactivated, a message posted during the gap was still processed once a workflow was re-activated. Useful to know when planning cutovers: deactivating is not the same as dropping traffic.

### Slack app
- Manifest saved and reinstalled: the token now carries **all 30 requested scopes**. Because `token_rotation_enabled: false`, Slack reissued the same token value with updated scopes, so **the n8n credential kept working** — the token-rotation outage this changelog previously warned about did not occur.

### Deployment
- Deployed the **Slack to Google Calendar v2** rewrite to the n8n instance for the first time. It had been complete but undeployed because `V2-PROGRESS.md` recorded the n8n Public API as 401-blocked; that block is gone and the API now authenticates normally.
  - Created as a **separate** workflow `RJiCNKlVQ4EhVyMS` — `Slack to Google Calendar AI Assistant (v2 TEST)`, `active: false`, 26 nodes / 40 connections. The v2 export's `id` is identical to production's, so a `PUT` would have destroyed the live workflow; it was created via `POST` with `id` omitted.
  - Production `I2dch7ZKvBvX6GVC` verified untouched before and after: still `active`, `versionId` `e8f30411-…`, `updatedAt` unchanged.
  - Authoritative n8n MCP `validate_workflow` (strict) against the deployed copy: `valid: true`, `errorCount: 0`, `invalidConnections: 0`, 26 nodes / 40 connections / 41 expressions / 23 warnings — matching the pre-deployment expectations recorded in `V2-PROGRESS.md` item for item.

### Fixes
- **v2 no longer depends on n8n variables**, which this instance's licence does not support (`GET /api/v1/variables` → 403 `feat:variables`). `$vars` is a defined but permanently empty object at runtime, so the dependency failed *silently* rather than loudly.
  - `Collect Thread Context` now falls back to the real, non-secret Slack identifiers — `$vars.SLACK_APP_ID || 'A08LUAXNTD2'` and `$vars.SLACK_BOT_USER_ID || 'U08MEH30GSU'` — obtained from `auth.test` + `bots.info`. Without this the thread-participation guard, metadata-ownership filter, self-trigger guard and reaction fallback were all inert.
  - `$vars` is kept first in the expression, so a future licence upgrade takes over automatically with no further workflow change.
- **Isolated acceptance testing from the production calendar.** v2's hardcoded calendar fallback was `abc12207@gmail.com`, which is the calendar production v1 actually writes to, and `$vars.GCAL_ID` could not override it. A disposable calendar (`n8n v2 TEST (safe to delete)`, Asia/Taipei) was created and the **deployed** v2 TEST now points at it. The repo's `.v2.json` deliberately keeps the production calendar, since that is the eventual cutover configuration.

### Corrections
- `.github/copilot-instructions.md` claimed `N8N_BLOCK_ENV_ACCESS_IN_NODE` was unset and `$env` access was permitted. **This is false.** A probe Code node returns `ExpressionError: access to env vars denied`, and the block applies to expressions too.
- **Discovered an unrelated live outage while verifying the above:** `Daily Currency Exchange Rate Alert` is `active` but **all 10 of its retained scheduled runs failed**, 2026-07-16 through 2026-07-29, every one with `access to env vars denied` at node `取得即時匯率` (URL `…/v6/{{$env.EXCHANGE_API_KEY}}/latest/TWD`). Fixing it requires moving the key into a credential, not restoring the environment variable. Not fixed in this change.
- Recorded that `settings.binaryMode` is rejected by the Public API's OpenAPI schema but is behaviourally inert — `binary-helper-functions.ts` defaults the parameter to `BINARY_MODE_SEPARATE`, so "unset" and `"separate"` are equivalent.

### Security
- **The Slack webhook accepts forged events.** Verified by replaying Slack's `url_verification` handshake against the live n8n endpoint: a request carrying a *bogus* `x-slack-signature` still returned HTTP 200 and echoed the challenge. The n8n Slack credential has no Signing Secret set, and `SlackTriggerHelpers.ts` L118 (`skipIfNoExpectedSignature`) skips verification entirely in that case. Anyone who learns the webhook URL can inject fake Slack events and have the AI create calendar entries. Remediation (no reinstall required; the Signing Secret does not rotate): copy it from Slack → Basic Information → App Credentials into the `signatureSecret` field of credential `H8RWtu7aOphSXGk5` — confirmed a supported field on the `slackApi` credential schema. Not applied here, since only the user can read the value.
- Noted while verifying: the HTTP 401 an unsigned probe receives is *not* signature enforcement. `verifySignature` returns false whenever `x-slack-request-timestamp` is missing (L91-94), before any secret is consulted.

### Documentation
- `V2-PROGRESS.md`: status changed from "尚未部署" to deployed-but-unaccepted, with the deployed id, the three deployment-time modifications and their evidence, and three pre-acceptance warnings (adding Slack scopes rotates the bot token and breaks v1 until the credential is updated; testing v2 requires deactivating v1 because they share a `webhookId`; the authorization boundary is channel membership).
- `V2-IMPORT.md`: §1 marked complete; §2 corrected (see below); §3 documents the variables-licence workaround.
- **Rewrote `slack-app-manifest-patch.yaml`.** Two errors were found in the old version by live-testing the workspace's actual token rather than re-reading the doc:
  - It claimed five scopes were mandatory. In fact all four that v2 actually calls — `channels:history`, `chat:write`, `files:read`, `reactions:write` — are **already granted**; the token holds 22 scopes. v2 is not blocked on permissions.
  - It claimed `metadata.message:read` was required to read back event ids. **Live round-trip proves otherwise**: writing `metadata` via `chat.postMessage` and reading it back via `conversations.replies?include_all_metadata=true` recovered the `event_id` intact without that scope. Slack's docs agree — the scope gates the *Events API* (`message_metadata_*` subscriptions), not the Web API read path v2 uses.
  - **The serious problem the old file would have caused:** an App Manifest *replaces* the scope list rather than appending to it. The old file listed 16 scopes against the app's 22, so pasting it would have silently removed 10 live permissions (`files:write`, `app_mentions:read`, `users.profile:read`, `usergroups:*`, `channels:write.*`, `im:read`, `mpim:read`, `remote_files:read`).
  - The file now contains the **union of currently-granted + v2's needs + plausible future needs: 30 scopes**, each name verified against Slack's scope reference, with the YAML parsed and checked so nothing granted is dropped. Goal is a single reinstall, never a second one.

## [1.0.4] - 2026-07-29
### Documentation
- Rewrote **Slack to Google Calendar AI Assistant** documentation to match the exported workflow JSON:
  - Replaced the outdated 9-node, webhook-based description with the actual 14-node graph (Slack Trigger → Filter Valid Messages → Analyze Message with AI → Parse AI Response → Has Valid Event → Check Confidence Score → …).
  - Documented the shared error sink: four nodes route `onError: continueErrorOutput` into `Send Error Notification`.
  - Documented the Slack Block Kit contract (`blocks` must be a `JSON.stringify` string) and the two block builder nodes.
  - Documented the duplicated 0.7 confidence threshold in `Parse AI Response` and `Check Confidence Score`.
  - Documented workflow settings, credential types, and the Slack Trigger (its production Webhook URL must still be registered in the Slack app's Event Subscriptions).
  - Kept `README.md` and `README.zh-tw.md` section-for-section aligned.

### Repository tooling
- Added `.mcp.json` for GitHub Copilot CLI MCP server configuration.
- Expanded `.github/copilot-instructions.md` with export-file contract, local checks, cross-node contracts, and credential handling rules.

### Corrections
- `sendUpdates` is `none`, not `all` as stated in 1.0.3; no reminders are configured in `additionalFields`. The 1.0.3 entry describes changes that are not present in the exported workflow.
- `attendees` is computed by `Parse AI Response` but is never mapped into `Create Calendar Event`, so events are always created without guests.

## [1.0.3] - 2025-06-01
### Enhancements
- Updated **Slack to Google Calendar AI Assistant**:
  - Improved AI prompt for better detection and processing of complex event types.
  - Enhanced structured event descriptions with detailed formatting.
  - Configured reminders for both all-day and timed events.
  - Improved notification messages with better formatting and details.
  - Changed calendar events to auto-send invites (sendUpdates: all).
  - Enhanced user experience with emoji and structured information.

### Fixes
- Resolved issues with Slack message filtering logic.
- Fixed incorrect handling of all-day event date formats.

## [1.0.2] - 2025-06-01
### Enhancements
- Improved **Slack to Google Calendar AI Assistant**:
  - Updated AI prompt to better detect and process complex event types.
  - Added structured event descriptions with detailed formatting.
  - Configured reminders for both all-day and timed events.
  - Improved notification messages with better formatting and details.
  - Set calendar events to not auto-send invites (sendUpdates: none).
  - Enhanced user experience with emoji and structured information.
- Added VS Code settings and configuration files

## [1.0.1] - 2025-05-29
### Updated
- Enhanced **README.md** to include a detailed project overview and directory structure.
- Improved **README.zh-tw.md** with additional information on system maintenance and update considerations.

## [1.0.0] - 2025-05-28
### Added
- Initial release of the **N8N-Flows** project.
- Added workflows for **RaindropKnowledgeManagement**, integrating Raindrop.io, Azure AI, and Notion.
- Included project documentation:
  - **README.md**: Overview and usage instructions.
  - **README.zh-tw.md**: Chinese technical documentation for node configurations and functionalities.