# Slack to Google Calendar AI Assistant — Implementation Guide

> Generated from the live production workflow `I2dch7ZKvBvX6GVC`, so the node list and topology cannot drift from what is deployed.

## 📋 Overview

| Item | Value |
|---|---|
| Workflow name | `Slack to Google Calendar AI Assistant` |
| Workflow ID | `I2dch7ZKvBvX6GVC` |
| Nodes / connections | 34 / 48 |
| Code nodes | 8 |
| State | active |
| Timezone | `Asia/Taipei` |

It reads Slack channel messages (including thread replies and images), asks Azure OpenAI whether a schedule is being described, creates or updates a Google Calendar event, replies into the originating thread with a Block Kit card, and leaves an emoji on the original message describing the outcome.

## 🔀 Pipeline

### Intake

A Slack message arrives, obvious non-candidates are dropped, and the message is marked 👀 immediately so the user knows it was seen before any slow work begins.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Slack Message Trigger` | `slackTrigger` v1 | — | 0→Filter Valid Messages |
| `Filter Valid Messages` | `if` v2.3 | — | 0→Fetch Thread Replies / Add Seen Reaction |
| `Add Seen Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

### Thread context

The whole thread is re-read on every run. This is what makes follow-up messages work: the bot reconstructs who started the thread, how many user turns there have been, which events it already created (from message metadata), and whether it should stay out of an unrelated conversation.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Fetch Thread Replies` | `httpRequest` v4.2 | continueErrorOutput | 0→Collect Thread Context<br>1→Normalize Error |
| `Collect Thread Context` | `code` v2 | continueErrorOutput | 0→Should Process<br>1→Normalize Error |
| `Should Process` | `if` v2.3 | — | 0→Has Images |

### Image handling

Images anywhere in the thread are collected, downloaded through Slack and attached to the AI request. Attachments that are not images, or that exceed the cap, are reported on the card rather than dropped silently.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Has Images` | `if` v2.3 | — | 0→Split Images<br>1→Build Azure Payload |
| `Split Images` | `code` v2 | continueErrorOutput | 0→Download Image<br>1→Normalize Error |
| `Download Image` | `httpRequest` v4.2 | continueRegularOutput | 0→Aggregate Images |
| `Aggregate Images` | `aggregate` v1 | — | 0→Build Azure Payload |

### AI analysis

A single Azure OpenAI chat completion receives the thread text, any images, and an explicit "now" in Asia/Taipei. The response is parsed defensively and the 👀 marker is cleared.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Build Azure Payload` | `code` v2 | continueErrorOutput | 0→Analyze With Azure<br>1→Normalize Error |
| `Analyze With Azure` | `@n8n/n8n-nodes-langchain.chainLlm` v1.9 | continueErrorOutput | 0→Parse AI Response<br>1→Normalize Error |
| `Azure OpenAI gpt-5.2` | `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` v1 | — | — |
| `Parse AI Response` | `code` v2 | continueErrorOutput | 0→Route Outcome / Remove Seen Reaction<br>1→Normalize Error |
| `Remove Seen Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

### Routing

One Switch decides the outcome. Order matters: forced creation is evaluated before clarification, and there is an unconditional fallback so nothing can fall through silently.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Route Outcome` | `switch` v3.2 | — | 0→Add Suppress Reaction<br>1→Send Not Owner Card<br>2→Update Calendar Event<br>3→Create Calendar Event<br>4→Send Clarify Card<br>5→Create Calendar Event<br>6→Send Duplicate Notice<br>7→Send Clarify Card |

### Calendar write

Creation and update are separate nodes so an update can never accidentally create a second event.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Create Calendar Event` | `googleCalendar` v1.3 | continueErrorOutput | 0→Send Success Card<br>1→Normalize Error |
| `Update Calendar Event` | `googleCalendar` v1.3 | continueErrorOutput | 0→Send Updated Card<br>1→Normalize Error |

### Slack cards

Six semantically distinct cards. Each send is followed by a delivery check, because a card can return HTTP 200 with `ok:false`, and the card metadata is the only way the next run recovers the event id.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Send Success Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Card Delivery |
| `Send Updated Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Updated Card Delivery |
| `Send Clarify Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Notice Delivery |
| `Send Duplicate Notice` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Notice Delivery |
| `Send Not Owner Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Notice Delivery |
| `Verify Card Delivery` | `code` v2 | continueErrorOutput | 0→Add Success Reaction<br>1→Normalize Error |
| `Verify Updated Card Delivery` | `code` v2 | continueErrorOutput | 0→Add Updated Reaction<br>1→Normalize Error |
| `Verify Notice Delivery` | `code` v2 | continueErrorOutput | 0→Add Notice Reaction<br>1→Normalize Error |

### Outcome reactions

The 👀 marker is replaced by an emoji describing the outcome, so the message itself carries the status without opening the thread.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Add Success Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Updated Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Notice Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Suppress Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

### Error handling

Every fallible node routes its error output to one place. The failing stage is derived from `$prevNode`, not from `$()`, because item pairing is unreliable on an error branch.

| Node | Type | onError | Outputs |
|---|---|---|---|
| `Normalize Error` | `code` v2 | — | 0→Send Error Card |
| `Send Error Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Remove Seen Marker / Add Alert Reaction |
| `Remove Seen Marker` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Alert Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

## 🧭 `Route Outcome` branches

| # | outputKey | Goes to |
|---|---|---|
| 0 | `suppress` | Add Suppress Reaction |
| 1 | `not_owner` | Send Not Owner Card |
| 2 | `update` | Update Calendar Event |
| 3 | `force` | Create Calendar Event |
| 4 | `clarify` | Send Clarify Card |
| 5 | `create` | Create Calendar Event |
| 6 | `duplicate` | Send Duplicate Notice |
| 7 | `extra` | Send Clarify Card |

## 🙂 Message reactions

A message gets 👀 the moment it passes the filter. Once the outcome is known, 👀 is removed and replaced by one of the following. Only one reaction is present at a time.

| Outcome | emoji | Added by |
|---|---|---|
| Seen, working on it | 👀 `eyes` | `Add Seen Reaction` |
| Created | ✅ `white_check_mark` | `Add Success Reaction` |
| Updated | 🔄 `arrows_counterclockwise` | `Add Updated Reaction` |
| Needs clarification / duplicate / not owner | ❓ `question`, ♻️ `recycle`, 🔒 `lock` | `Add Notice Reaction` |
| Ignored | ➖ `heavy_minus_sign` | `Add Suppress Reaction` |
| Failed | ⚠️ `warning` | `Add Alert Reaction` |

Every reaction node sets `onError: continueRegularOutput`. Reactions are cosmetic, and `reactions.add` returns `already_reacted` on a repeat while `reactions.remove` returns `no_reaction` when there is nothing to remove; neither should fail a run. `Remove Seen Marker` and `Add Alert Reaction` hang off `Send Error Card` and read `channel` and `message.thread_ts` straight from the `chat.postMessage` response, so they need no `$()`.

## ⚙️ Workflow settings

```json
{
  "saveExecutionProgress": true,
  "saveManualExecutions": true,
  "saveDataErrorExecution": "all",
  "saveDataSuccessExecution": "all",
  "executionTimeout": 3600,
  "timezone": "Asia/Taipei",
  "executionOrder": "v1",
  "binaryMode": "separate",
  "callerPolicy": "workflowsFromSameOwner",
  "availableInMCP": false
}
```

## 🔐 Credentials

| Type | Name | Nodes using it |
|---|---|---|
| `slackApi` | Slack account | 17 |
| `googleCalendarOAuth2Api` | Google Calendar account | 2 |
| `azureEntraCognitiveServicesOAuth2Api` | Azure Open AI account Entra ID | 1 |

> ⚠️ The Azure credential uses the `client_credentials` flow, which returns **no refresh token**, so n8n cannot renew the access token once it expires and it has to be reconnected by hand. The durable fix is an API-key `azureOpenAiApi` credential instead.

## 🔧 Slack app configuration

The full scope and event list lives in [`slack-app-manifest-patch.yaml`](./slack-app-manifest-patch.yaml) (30 bot scopes, 6 bot events). **An app manifest replaces the scope list rather than appending to it** — read the warning at the top of that file before applying it.

## 🚨 Known behaviours

- **A thread can hold several events.** `isDuplicate` compares start + title, so a reply describing a different event creates a new one. That is by design.
- **Modifying an existing event requires stating the change** (for example 「改到三點」 or 「地點改成台中」). Simply restating the original text routes to a clarification card rather than creating a duplicate.
- **The authorization boundary is channel membership.** Anyone who can post in the channel can create events; `isOwner` only guards later modification of an event already created.
- **Concurrency is not handled.** Two messages arriving in the same thread within a moment of each other can each create an event before the other writes its metadata.
- **Slack redelivers events.** Messages sent while the workflow is deactivated may still be processed once it is re-activated.

## 📝 Maintenance

- Make changes through n8n MCP or the Public API, then read the cloud copy back and overwrite the JSON in this folder.
- `connections` is keyed by **node name**, so renaming a node means updating `connections`, any `$('Node Name')` calls inside Code nodes, and expressions.
- This document is generated from the live workflow; regenerate it when nodes change rather than hand-editing the tables.
