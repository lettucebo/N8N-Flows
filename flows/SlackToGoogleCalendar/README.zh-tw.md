# Slack to Google Calendar AI Assistant — 實作說明

> 本文件由生產工作流程 `I2dch7ZKvBvX6GVC` 直接產生，節點清單與拓撲保證與線上一致。

## 📋 概覽

| 項目 | 值 |
|---|---|
| 工作流程名稱 | `Slack to Google Calendar AI Assistant` |
| 工作流程 ID | `I2dch7ZKvBvX6GVC` |
| 節點數 / 連線數 | 35 / 49 |
| Code 節點 | 9 |
| 狀態 | 啟用中 |
| 時區 | `Asia/Taipei` |

這個工作流程讀取 Slack 頻道訊息（含 thread 回覆與圖片），用 Azure OpenAI 判斷是否描述了行程，並在 Google 日曆建立或更新事件，然後把結果以 Block Kit 卡片回覆到原 thread，同時在原訊息上留下對應的 emoji。

## 🔀 處理管線

### 接收

Slack 訊息進來，先濾掉明顯不需要處理的，然後**立刻**加上 👀 —— 讓使用者在任何耗時工作開始前就知道「我看到了」。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Slack Message Trigger` | `slackTrigger` v1 | — | 0→Filter Valid Messages |
| `Filter Valid Messages` | `if` v2.3 | — | 0→Fetch Thread Replies / Add Seen Reaction |
| `Add Seen Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

### 對話脈絡

每次執行都重新讀取整個 thread。這是續談能運作的關鍵：重建 thread 發起人、使用者已經講了幾輪、先前已建立哪些事件（從訊息 metadata 讀回），以及是否該迴避不相關的討論串。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Fetch Thread Replies` | `httpRequest` v4.2 | continueErrorOutput | 0→Collect Thread Context<br>1→Normalize Error |
| `Collect Thread Context` | `code` v2 | continueErrorOutput | 0→Should Process<br>1→Normalize Error |
| `Should Process` | `if` v2.3 | — | 0→Has Images |

### 圖片處理

收集 thread 內任何位置的圖片，透過 Slack 下載後併入 AI 請求。非圖片附件或超過上限的部分會在卡片上說明，不會靜默丟棄。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Has Images` | `if` v2.3 | — | 0→Split Images<br>1→Build Azure Payload |
| `Split Images` | `code` v2 | continueErrorOutput | 0→Download Image<br>1→Normalize Error |
| `Download Image` | `httpRequest` v4.2 | continueRegularOutput | 0→Aggregate Images |
| `Aggregate Images` | `aggregate` v1 | — | 0→Build Azure Payload |

### AI 解析

一次 Azure OpenAI chat completion，帶入 thread 文字、圖片，以及明確的 Asia/Taipei「現在時間」。回應以防禦性方式解析，並清除 👀 標記。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Build Azure Payload` | `code` v2 | continueErrorOutput | 0→Analyze With Azure<br>1→Normalize Error |
| `Analyze With Azure` | `@n8n/n8n-nodes-langchain.chainLlm` v1.9 | continueErrorOutput | 0→Parse AI Response<br>1→Normalize Error |
| `Azure OpenAI gpt-5.2` | `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` v1 | — | — |
| `Parse AI Response` | `code` v2 | continueErrorOutput | 0→Route Outcome / Remove Seen Reaction<br>1→Normalize Error |
| `Remove Seen Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

### 分流

單一 Switch 決定結局。順序有意義：強制建立排在澄清之前，且有無條件的 fallback，確保沒有任何情況會靜默落空。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Route Outcome` | `switch` v3.2 | — | 0→Add Suppress Reaction<br>1→Send Not Owner Card<br>2→Update Calendar Event<br>3→Create Calendar Event<br>4→Send Clarify Card<br>5→Create Calendar Event<br>6→Send Duplicate Notice<br>7→Send Clarify Card |

### 寫入日曆

建立與更新是兩個獨立節點，因此更新永遠不可能誤建第二筆事件。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Create Calendar Event` | `googleCalendar` v1.3 | continueErrorOutput | 0→Pair Created Event<br>1→Normalize Error |
| `Update Calendar Event` | `googleCalendar` v1.3 | continueErrorOutput | 0→Send Updated Card<br>1→Normalize Error |
| `Pair Created Event` | `code` v2 | — | 0→Send Success Card |

### Slack 卡片

六張語義不同的卡片。每次傳送後都接一個投遞確認 —— 因為 Slack 可能回 HTTP 200 但 `ok:false`，而卡片的 metadata 是下一輪讀回 event id 的唯一途徑。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Send Success Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Card Delivery |
| `Send Updated Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Updated Card Delivery |
| `Send Clarify Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Notice Delivery |
| `Send Duplicate Notice` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Notice Delivery |
| `Send Not Owner Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Verify Notice Delivery |
| `Verify Card Delivery` | `code` v2 | continueErrorOutput | 0→Add Success Reaction<br>1→Normalize Error |
| `Verify Updated Card Delivery` | `code` v2 | continueErrorOutput | 0→Add Updated Reaction<br>1→Normalize Error |
| `Verify Notice Delivery` | `code` v2 | continueErrorOutput | 0→Add Notice Reaction<br>1→Normalize Error |

### 結局 reaction

👀 會被換成描述結局的 emoji，讓訊息本身就帶著狀態，不必展開 thread 才知道結果。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Add Success Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Updated Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Notice Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Suppress Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

### 錯誤處理

所有可能失敗的節點都把錯誤輸出導向同一處。失敗階段由 `$prevNode` 判定而非 `$()`，因為錯誤分支上的 item 配對並不可靠。

| 節點 | 型別 | onError | 輸出 |
|---|---|---|---|
| `Normalize Error` | `code` v2 | — | 0→Send Error Card |
| `Send Error Card` | `httpRequest` v4.2 | continueRegularOutput | 0→Remove Seen Marker / Add Alert Reaction |
| `Remove Seen Marker` | `httpRequest` v4.2 | continueRegularOutput | — |
| `Add Alert Reaction` | `httpRequest` v4.2 | continueRegularOutput | — |

## 🧭 `Route Outcome` 的出口

| # | outputKey | 導向 |
|---|---|---|
| 0 | `suppress` | Add Suppress Reaction |
| 1 | `not_owner` | Send Not Owner Card |
| 2 | `update` | Update Calendar Event |
| 3 | `force` | Create Calendar Event |
| 4 | `clarify` | Send Clarify Card |
| 5 | `create` | Create Calendar Event |
| 6 | `duplicate` | Send Duplicate Notice |
| 7 | `extra` | Send Clarify Card |

## 🙂 訊息 reaction

訊息通過過濾的當下就會拿到 👀，結局確定後 👀 會被移除並換成下列其中一個。同一時間只會有一個 reaction。

| 結局 | emoji | 由哪個節點加上 |
|---|---|---|
| 已看到、處理中 | 👀 `eyes` | `Add Seen Reaction` |
| 建立成功 | ✅ `white_check_mark` | `Add Success Reaction` |
| 更新成功 | 🔄 `arrows_counterclockwise` | `Add Updated Reaction` |
| 需要澄清 / 重複 / 非建立者 | ❓ `question`、♻️ `recycle`、🔒 `lock` | `Add Notice Reaction` |
| 不處理 | ➖ `heavy_minus_sign` | `Add Suppress Reaction` |
| 失敗 | ⚠️ `warning` | `Add Alert Reaction` |

所有 reaction 節點都設 `onError: continueRegularOutput`：reaction 純屬裝飾，而 `reactions.add` 重複會回 `already_reacted`、`reactions.remove` 無對象會回 `no_reaction`，都不該讓整個執行失敗。`Remove Seen Marker` 與 `Add Alert Reaction` 掛在 `Send Error Card` 之後，直接取用 `chat.postMessage` 回應裡的 `channel` 與 `message.thread_ts`，因此不需要 `$()`。

## ⚙️ 工作流程設定

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

## 🔐 憑證

| 型別 | 名稱 | 使用節點數 |
|---|---|---|
| `slackApi` | Slack account | 17 |
| `googleCalendarOAuth2Api` | Google Calendar account | 2 |
| `azureEntraCognitiveServicesOAuth2Api` | Azure Open AI account Entra ID | 1 |

> ⚠️ Azure 憑證使用 `client_credentials` 流程，**不會發 refresh token**，因此 access token 到期後 n8n 無法自動續期，需要手動 Reconnect。長期解法是改用 API key 型的 `azureOpenAiApi` 憑證。

## 🔧 Slack App 設定

完整的 scope 與事件訂閱清單見同目錄的 [`slack-app-manifest-patch.yaml`](./slack-app-manifest-patch.yaml)（30 個 bot scope、6 個 bot event）。**Manifest 的 scope 清單是整份取代而非追加**，套用前務必讀該檔開頭的警告。

## 🚨 已知行為

- **同一個 thread 可以有多個事件。** `isDuplicate` 比對 start + title；描述不同事件的回覆會建立新事件，這是設計而非缺陷。
- **要修改既有事件，回覆時要說明修改意圖**（例如「改到三點」「地點改成台中」）。單純把原文重貼一次會走澄清卡，不會重複建立。
- **授權邊界等同頻道成員名單。** 任何能在該頻道發言的人都能建立事件。`isOwner` 只保護「已建立事件的後續修改」。
- **並行競態未處理。** 兩則訊息極短時間內送達同一 thread，可能在對方寫入 metadata 前各自建立事件。
- **Slack 會補送事件。** 工作流程停用期間的訊息，在重新啟用後仍可能被處理。

## 📝 維護

- 修改請走 n8n MCP 或 Public API，改完把雲端讀回並覆寫本目錄的 JSON。
- `connections` 以**節點名稱**為鍵，改名等於要同步改 `connections`、Code 節點內的 `$('節點名')` 以及運算式。
- 本文件由腳本從線上工作流程產生；節點有增減時請重新產生，不要手改表格。
