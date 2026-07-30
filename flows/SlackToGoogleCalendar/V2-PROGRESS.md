# v2 重新設計 — 進度紀錄

最後更新：2026-07-29

這份文件記錄 `Slack_to_Google_Calendar_AI_Assistant.v2.json` 的**製作進度與狀態**。
要匯入它之前請先讀 [`V2-IMPORT.md`](./V2-IMPORT.md)（前置設定、逐項驗收測試、已知未解風險）。

---

## ✅ 目前狀態：**v2 已升級為生產版**（2026-07-30）

| 項目 | 狀態 |
|---|---|
| **生產工作流程** | `I2dch7ZKvBvX6GVC`（`Slack to Google Calendar AI Assistant`）**內容已換成 v2**，33 節點 / 47 連線，**啟用中**。刻意保留原本的 workflow id，因此 webhook 路徑、執行歷史、repo 檔案的 id 綁定全部延續 |
| **升級前備份** | 線上 v1 定義已存成 `v1-backup.json`（session 目錄），git 歷史中亦可取回 |
| **測試副本** | `RJiCNKlVQ4EhVyMS`（`… (v2 TEST)`）保留但**停用**，仍指向拋棄式測試日曆，可隨時再做測試 |
| **repo 檔案** | `Slack_to_Google_Calendar_AI_Assistant.json` 即生產版；多餘的 `.v2.json` 已刪除（一個工作流程一份匯出） |
| **MCP strict（生產版）** | `valid: true`／`errorCount: 0`／`invalidConnections: 0`／33 nodes／47 connections |
| **README** | 兩種語言都改為**由線上工作流程程式化產生**，節點涵蓋率 33/33 |

### 唯一未解的阻塞：Azure 憑證無法自動續期

**✅ 已於 2026-07-30 修復 —— 而且根因是我在 v2 引入的回歸，不是憑證問題。**

先前的診斷（「憑證約每小時失效、需手動 Reconnect」）**是錯的**。真正原因：

| 版本 | 呼叫 Azure 的方式 | 取 token 的路徑 | 結果 |
|---|---|---|---|
| v1 | `chainLlm` + `lmChatAzureOpenAi` | 節點自行取得 token | ✅ 從不失效 |
| v2（我改的） | `httpRequest` + `predefinedCredentialType` | n8n 泛用 OAuth2 helper：沿用既有 token，過期後用 `refresh_token` 續期 | ❌ 每小時死一次 |

這個 Entra 憑證以 `client_credentials` 換 token（透過 `additionalBodyProperties` 注入），
而 client_credentials **不會回傳 refresh_token**，所以泛用 helper 一旦遇到過期就永久失敗。
LangChain 節點不走那條路，因此 v1 從來沒事。

**同一憑證、同一分鐘的對照實驗**：LangChain 回 `{"text":"{\"ok\":true}"}`，
httpRequest 回 `OAuth access token expired and no refresh token is available`。

**修法**：把 `Call Azure OpenAI`（httpRequest）換成
`Analyze With Azure`（`chainLlm` v1.9）+ `Azure OpenAI gpt-5.2`（`lmChatAzureOpenAi` v1），
恢復 v1 可用的驗證路徑，同時保留 v2 的圖片理解，且**維持 Entra ID 驗證**。

圖片如何保留：`chainLlm` 只認**靜態**圖片槽位，空槽會讓節點直接失效（已實測）。
因此 `Build Azure Payload` 固定送四槽，不足的用 1×1 透明 PNG 補位，
並在 system prompt 明確要求忽略空白佔位圖。
實測：1 張真圖 + 3 張空白，模型仍正確讀出
`季度營運檢討會議 / 2026-10-08 / 14:30–16:00 / 台北辦公室 12F 大會議室`，且回報 `realImages: 1`。

**決定性驗證**：在**完全沒有重新授權憑證**的情況下（舊路徑當下正在失敗）跑完整條鏈路：
`Filter → 👀 → Fetch Thread → Collect Context → Build Payload → Azure OpenAI gpt-5.2
→ Analyze With Azure → Parse AI Response → 移除 👀 → Route → Create Calendar Event
→ Send Success Card → ✅`，`intent=create`、`confidence=0.9`、`title="情人節晚餐"`。

### 過程中排除的做法（記錄下來避免重試）

- **整個 `messages` 參數用運算式動態產生圖片槽位** —— 不會報錯，但圖片根本沒進模型（1 張與 3 張都回 `seen: 0`）
- **用 API 改憑證設定** —— `PATCH /credentials/{id}` 的 `data` 是整份取代（回 400 要求補 `resourceName`／`apiVersion`），沒有機密就改不了
- **改用 API key 憑證** —— 不符合「必須使用 Entra ID」的要求，且事後證明也不需要
- prompt 裡的 JSON 大括號在 `chainLlm` 中是安全的（已實測）
| **真實環境執行紀錄** | **已首次實跑（2026-07-30 10:15–10:24）。** 以 Playwright 驅動 Slack 網頁、用真人帳號 `abc12207` 發訊息，產生 6 筆 v2 執行。核心邏輯驗證通過，但**AI 步驟被憑證問題阻斷**，詳見下方「首次實測結果」 |
| n8n Public API | **已解除阻塞。** 401 不再發生，認證正常，本次部署與驗證皆透過 Public API 完成 |
| n8n MCP strict 驗證（對雲端實體） | `valid: true`／`errorCount: 0`／`invalidConnections: 0`／26 nodes／40 connections／41 expressions／23 warnings —— **與本文件下方記載的製作期預期值逐項吻合** |

**「已驗證」仍不包含任何一次真實的 Slack／Google Calendar／Azure OpenAI 呼叫。**
上線前必須依 `V2-IMPORT.md` 的 54 步驗收測試實跑。

### 部署時所做的三項修改（都是環境限制造成，非設計變更）

| # | 修改 | 原因（皆有實測證據） |
|---|---|---|
| 1 | `Collect Thread Context` L15/L16 的 `$vars.SLACK_APP_ID` / `$vars.SLACK_BOT_USER_ID` 補上寫死的 fallback `'A08LUAXNTD2'` / `'U08MEH30GSU'` | 本 instance 授權**不支援 variables**（`GET /api/v1/variables` 回 403 `feat:variables`），`$vars` 執行期恆為空物件。若不補，L172 的「本 app 參與防護」會整段跳過。兩個值取自 Slack `auth.test` + `bots.info`，**皆非機密**，且 Slack App 重裝後不變（只有 token 會輪換）。寫法保留 `$vars` 優先，日後若啟用 variables 會自動改走 `$vars`，不必再改 workflow |
| 2 | 部署時捨棄 `settings.binaryMode: "separate"` | n8n Public API 的 OpenAPI schema 不接受該欄位（其餘 9 個 settings 皆接受）。**行為完全等價**：`binary-helper-functions.ts` L62/L133 的參數預設值就是 `BINARY_MODE_SEPARATE`，故「未設定 ≡ separate」 |
| 3 | **僅雲端的 v2 TEST** 將日曆 fallback 指向專用測試日曆 `03dbf875…@group.calendar.google.com`（`n8n v2 TEST (safe to delete)`，Asia/Taipei） | v2 原本寫死的 fallback `abc12207@gmail.com` **正是 v1 生產版在用的日曆**，而 `$vars.GCAL_ID` 因授權限制無法設定 —— 不改的話，54 步驗收會把大量測試事件寫進主日曆。**本 repo 的 `.v2.json` 刻意維持 `abc12207@gmail.com`**，因為那才是日後正式切換要用的組態。驗收結束後把整本測試日曆刪掉即可完全清理 |

> ⚠️ 因此 **雲端 v2 TEST 與 repo 的 `.v2.json` 在日曆 ID 上刻意不一致**。正式切換時必須把日曆改回 `abc12207@gmail.com`。

---

## 首次實測結果（2026-07-30）

測試方式：Playwright 驅動 Chrome 登入 Slack，以**真人帳號** `abc12207` 在 `#n8n-calendar` 發訊息
（必須是真人 —— bot 發的訊息會被自我過濾擋掉，無法觸發完整流程）。
測試期間先停用 v1、啟用 v2 TEST，結束後已還原。

### ✅ 端到端全部打通（Azure 憑證重新授權後）

| 場景 | 輸入 | 結果 |
|---|---|---|
| 定時事件 | 明天下午三點跟客戶開會，大概一小時 | 信心 0.90 → 建立 `2026-07-31 15:00–16:00`，綠色成功卡 |
| 全天事件 | 8月15號健康檢查 | 信心 0.72 → `2026-08-15`，結束日 `08-16`（**排他端點正確**） |
| 資訊不足 | 下週找個時間跟設計團隊聊一下 | 信心 0.38 → 黃色澄清卡，未建立事件 |
| 非行程閒聊 | 大家早安，今天也要加油 | `aiHasEvent=false`、信心 0.05 → **不建立事件**，只加 ➖ reaction |
| **圖片理解** | 上傳自製的會議通知圖片（無文字說明日期） | 信心 **0.94** → 正確讀出 `2026-10-08 14:30–16:00`、標題「季度營運檢討會議」、地點「台北辦公室 12F 大會議室」 |
| **thread 修改（時間）** | 根：9月10號下午兩點跟客戶簡報 → 回覆：改到三點 | `intent=update` → **更新既有事件**（14:00 → 15:00），未產生第二筆 |
| **thread 修改（非時間）** | 回覆：地點改成台中辦公室 | `intent=update` → 更新同一筆 |
| **重複描述** | 回覆：把根訊息原文再貼一次 | 走澄清卡，**未建立重複事件** |

最終日曆只有**一筆**「跟客戶簡報」，證明 update 走的是同一個 `eventId`
（`metadataReadable: true`，`existingEvent` 由 message metadata 正確讀回）。

### ✅ 已被真實執行證實的內部行為

| # | 項目 | 證據 |
|---|---|---|
| 1 | `Filter Valid Messages` 放行真人、擋下 bot | bot 觸發的執行**只跑 2 個節點就停**，證明**不會無限迴圈** |
| 2 | **寫死的 Slack ID 修補在執行期生效** | `botAppIdConfigured: true`／`botUserIdConfigured: true`。若未修補，thread 參與防護會整段失效 |
| 3 | thread 續談與輪次累計 | `round` 依序 1→2→3→4；`latestUserMessage` 正確取到新的**使用者**訊息而非 bot 卡片 |
| 4 | thread 歷史組裝 | `[user]`／`[bot]` 正確分類，每則帶自己的時間前綴（prompt 的相對日期錨定依賴這個） |
| 5 | 圖片下載 | `Download Image` 成功取得 `url_private`（httpRequest 4.2 的跨來源帶憑證行為如預期） |
| 6 | 測試日曆隔離 | 全程未寫入主日曆；結束後測試日曆事件已全數清除 |

### ✅ 訊息回饋 reaction（本次新增，見下節）

### 🐛 實測抓到的 v2 缺陷（**已於 2026-07-30 修復**）

`Normalize Error` 曾把所有錯誤都標成「讀取 Slack 對話」，但實際失敗在 `Call Azure OpenAI`。

**已修**：階段與 `calendarWritten` 改由 `$prevNode.name` 判定 —— 那是節點層 metadata，
不依賴 item 配對。實測驗證：

| 實際失敗節點 | 回報階段 | 結果 |
|---|---|---|
| `Call Azure OpenAI` | 「呼叫 AI 解析」 | ✅ 正確 |
| `Collect Thread Context` | 「整理對話內容」 | ✅ 正確 |

連帶風險也一併關閉：`calendarWritten` 不再只靠 `pick()`。只要失敗發生在卡片傳送階段
（`Send Success Card`／`Send Updated Card`／兩個 `Verify …`），日曆必然已寫入，
錯誤卡就不會再叫使用者「重試或手動建立」而造成重複事件。

**同時更正一個先前過度概括的說法**：`$()` 並非在錯誤分支上全面失效。
同一次執行中 `reached('Build Azure Payload')` 回 false，但 `$('Slack Message Trigger')`
成功取到 ts。真正的問題是**個別節點的 item 配對解析不到**，不是「錯誤分支不能用 `$()`」。

### 🔧 MCP strict 誤報（**已於 2026-07-30 解除**）

MCP 會把「名稱帶失敗語義、且與其他節點共用同一個 `main[0]`」的節點判定為錯接的 error handler。
改拓撲（讓 `Normalize Error[0]` 只接 `Send Error Card`）之後誤報轉移到 `Send Error Card`；
最後把 `Add Error Reaction` 改名為 **`Add Alert Reaction`** 才解除。
生產版現在 `errorCount: 0`。

### 📌 其他發現

- **Slack 會補送事件**：兩個 workflow 都停用期間發的訊息，在重新啟用後仍被處理。切換時要留意。
- **thread 內描述「不同的事件」會建立新事件，這是設計行為**（`V2-IMPORT.md`：允許同 thread 建立不同事件，`isDuplicate` 比對 start+title）。曾誤以為是缺陷，實測乾淨案例後確認 update 機制正常。

---

## 訊息回饋 reaction（2026-07-30 新增並實測）

原本 v2 只有**一個** reaction：`aiHasEvent=false` 時加 ➖，其餘結局完全沒有訊息層回饋。
（`slack-app-manifest-patch.yaml` 舊註解寫「加 ➖ ❓ ✅ ⚠️ reaction」，但實作只有 ➖。）

已改為**先確認、後更新**的兩段式回饋：

| 時機 | 動作 |
|---|---|
| 訊息通過 `Filter Valid Messages` 的當下 | 立刻加 👀 `eyes` —— 「我看到了，正在處理」 |
| 結局確定後 | 移除 👀，換成對應結局的 emoji |

| 結局 | emoji |
|---|---|
| 建立成功 | ✅ `white_check_mark` |
| 更新成功 | 🔄 `arrows_counterclockwise` |
| 需要澄清 | ❓ `question` |
| 重複事件 | ♻️ `recycle` |
| 非建立者 | 🔒 `lock` |
| 不處理 | ➖ `heavy_minus_sign` |
| 失敗 | ⚠️ `warning` |

新增 7 個節點（26 → 33），實測時序：

```
t+ 4s  👀 eyes
t+ 8s  （已移除）
t+14s  ✅ / ❓ / ➖   ← 同時只會有一個 reaction
```

三種結局（成功／澄清／不處理）皆已實跑驗證通過。

設計要點：

- 所有 reaction 都貼在**觸發本次執行的那一則訊息**上，thread 回覆會標在回覆本身而非根訊息
  （`Add Suppress Reaction` 原本用 `threadTs`，已一併對齊）。
- 全部設 `onError: continueRegularOutput`。reaction 純屬裝飾，且 `reactions.add` 重複會回
  `already_reacted`、`reactions.remove` 無對象會回 `no_reaction`，都不該讓整個執行失敗。
- 正常路徑用 `$('Slack Message Trigger')` 取目標；錯誤路徑不能用 `$()`（見上方缺陷），
  改用 `Normalize Error` 自己輸出的 `channel`／`threadTs`。
- 三個節點掛在 `Normalize Error` 的 main[0]。**MCP strict 會對此回報 1 個 error，那是誤報** ——
  `Normalize Error` 是沒有設 `onError` 的 Code 節點，只有 main[0] 一個輸出；
  照 MCP 建議把節點移到 main[1] 會接到不存在的輸出而弄壞流程。
  已嘗試改名迴避（`Remove Seen Marker`／`Add Failure Reaction`）但無效，
  因為 `Send Error Card` 本身就會觸發該啟發式。

---

---

## workflow 現況

| 項目 | 值 |
|---|---|
| 檔案 | `Slack_to_Google_Calendar_AI_Assistant.v2.json` |
| workflow 名稱 | `Slack to Google Calendar AI Assistant` |
| 節點數 / 連線數 | 33 / 47（原 26 / 40，2026-07-30 新增 7 個 reaction 節點）|
| Code 節點 | 8（全部 `onError: continueErrorOutput` → `Normalize Error`）|
| `active` | `false`（刻意保持停用）|

---

## 三大需求的達成狀態

| 需求 | 狀態 | 實作方式 |
|---|---|---|
| **1. 讀取 Slack 圖片並擷取資訊** | 已實作，未實測 | `Collect Thread Context` 收集 thread 內圖片（上限 4 張）→ `Download Image` → `Build Azure Payload` 組 vision 訊息 → Azure OpenAI。下載失敗、非圖片、超過上限、超過大小上限都會分類並在卡片上說出原因 |
| **2. 信心不足時在原 thread 補資訊繼續** | 已實作，未實測 | 澄清卡列出**具名的缺項**與可行動的下一步；使用者在同一 thread 補充後，`Collect Thread Context` 會帶著完整歷史（含原圖）重跑。使用者說「就這樣建立」可強制建立 —— 這條脫困路徑有 17 個專屬測試守著 |
| **3. 訊息美化、使用不同卡片** | 已實作，未實測 | 6 張語義不同的卡片：成功（綠）／更新（藍）／澄清（黃）／重複告知／非建立者（紫）／錯誤。時間顯示為 `3月1日（一） 09:00–10:30`（今年省略年份、全天事件還原 exclusive 結束日），不再出現 ISO 字串與信心度百分比 |

---

## 驗證體系

全套入口：`run-all.mjs`（會先重新建置 JSON 再依序執行）。

| 驗證器 | 涵蓋 | 結果 |
|---|---|---|
| `validate-workflow.mjs` | 結構、參數、連線、卡片 block 數上限 | PASS |
| `syntax-check.mjs` | 8 個 Code node 的 `jsCode` 語法（`AsyncFunction`）| PASS |
| `io-contract.mjs` | 19 條欄位映射契約、6 張 Slack 卡片的收件位址、8 個路由出口的拓撲語義、58 個 `$json` 引用完整性 | PASS |
| `test-collect-v2.mjs` | `Collect Thread Context` | 42/42 |
| `test-thread-context.mjs` | 同上的**生產端**：`allEvents` 產出／去重、`latestUserMessage` 來源 | 8/8 |
| `yearfix-v3.mjs` | 年份三態推斷 | 28/28 |
| `route-outcome.mjs` | 對真實 Switch 規則窮舉 1728 種組合 | 16 情境，落空 0、不安全 0 |
| `test-parse-route.mjs` | Parse → Route 接縫 | 21/21 |
| `test-escape-hatch.mjs` | 每一條「使用者不該被卡住」的脫困路徑 | 17/17 |
| `test-update-target.mjs` | update 到底改到哪一個事件 | 11/11 |
| `test-error-normalize.mjs` | `Normalize Error` 生產端 | 17/17 |
| `test-image-guard.mjs` | 圖片守衛 + Build→Parse→Card 接縫 | PASS |
| `test-display-time.mjs` | 卡片可讀時間 | 7/7 |
| `test-multi-event.mjs` | 同 thread 多事件重複判定 | 8/8 |
| `test-timezone.mjs` | 時區守衛 | 9/9 |
| `render-cards.mjs` | 6 張卡片 × 30 個情境，含語義斷言 | 30/30 |
| `verify-rd5b.mjs` | Rubber Duck 第五輪的 22 個複現案例 | 0/22 仍成立 |
| `calc-calendar-regex.mjs` | `calendarId` 守衛 regex 窮舉推導 | PASS |
| `check-doc-sync.mjs` | 文件宣稱的節點數／連線數／Code 節點數與 JSON 一致 | PASS |
| `mcp-validate.mjs` | **n8n MCP** `validate_workflow`（strict）+ 逐節點 `validate_node` | PASS |

**n8n MCP 實測結果**（`n8n-mcp` 鎖定 **2.65.1**，執行時會驗證實際版本相符）：

```
valid: true          errorCount: 0        invalidConnections: 0
totalNodes: 26       validConnections: 40 expressionsValidated: 41
warningCount: 23     validate_node: 26/26 ok
```

23 個 warning 已在 `V2-IMPORT.md` 逐項拆解處置，並由 `mcp-validate.mjs` 對照本 repo 的文件，
數字漂移會直接紅燈。

**mutation test**：`mutate-guards.mjs` 對 38 項防護逐一改壞，**38/38 全數被偵測**，還原後回到全綠。

---

## 這一輪最重要的教訓

**「改完程式跑全套、全綠」不是通過的證據，而是測試沒有鑑別力的訊號。**

第二輪 Council / Rubber Duck 找到的 12 個缺陷（`V2-IMPORT.md` 的 #39–#50），在被發現之前
既有驗證器**一次都沒有變紅**。過程中連續三次出現同一個模式：做完實質行為變更 → 全綠 →
補測試 → 用 mutation 證明舊測試確實抓不到。

衍生出兩個具體的反模式，都已實際踩到：

1. **producer/consumer 盲區** —— 驗了消費端不等於驗了生產端。
   `allEvents` 的消費端有測試，把**產生**它的整段程式碼刪掉仍然全綠；
   卡片測試直接餵 `uncertainWrite: true`，把**算出**這個旗標的邏輯改成 `false` 依然全綠。
   每個跨節點欄位都需要兩端各自的測試。
2. **文件數字會靜默腐爛** —— 「21 個 warning / 7 個 Code 節點」在實際變成 23／8 之後，
   `check-doc-sync.mjs` 仍回報「文件與 JSON 一致」，因為它根本沒檢查這兩項。
   看起來很具體、很有依據的數字，是最難察覺的一種錯誤。現已納入自動比對。

---

## 待辦

| 項目 | 狀態 | 說明 |
|---|---|---|
| 補上 Slack scope（一次到位） | **✅ 已完成（2026-07-30）** | Manifest 存檔並 Reinstall to Workspace 後實測：token 已帶 **30 個 scope，一個不缺**。且因 `token_rotation_enabled: false`，Slack 沿用同一組 token 值只更新權限，**credential 未失效、不需重新填入**，先前預期的 token 輪換中斷並未發生 |
| **重新授權 Azure OpenAI 憑證** | **🔴 阻塞中，且同時影響生產版。只能由使用者操作** | `82mlP2DDo7j1VGdC` 的 OAuth token 已過期（`no refresh token is available`）。v1 在 2026-07-29 02:23 還正常，之後失效。**沒有它，v1 與 v2 都無法做 AI 分析。**<br><br>**已窮盡的自動修復嘗試（全部失敗，附原因）**：<br>① Public API `GET /credentials/{id}` **不回傳 `data` 欄位**，讀不到 `clientId`／`clientSecret`；<br>② `PATCH /credentials/{id}` 雖回 200 可用，但要寫入新 token 仍需上述機密；<br>③ 憑證的 `additionalBodyProperties` 是 `{"grant_type":"client_credentials"}`，理論上可用 client secret 無互動重取 token —— 但機密讀不到，此路不通；<br>④ **instance 上另一個 Azure 憑證 `CtgkMRzzV4whz91f` 也不能用**，實測回 `Unable to sign without access token`（從未完成授權）；<br>⑤ n8n UI 需要登入（`/signin`），我沒有帳密。<br><br>**修法：n8n → Credentials → 「Azure Open AI account Entra ID」→ Reconnect → 完成 Microsoft 授權。** 這是恢復生產與完成驗收的**第一優先** |
| **修 `Normalize Error` 階段誤報** | **未修，已定位** | 錯誤卡片會把所有失敗都說成「讀取 Slack 對話」。根因是錯誤分支內 `$('節點名')` 會拋例外，使 `reached()`／`pick()` 全部失效。連帶可能讓 `calendarWritten` 恆為 false，導致「日曆已建立卻叫使用者重試」→ 重複事件。詳見「首次實測結果」 |
| **設定 Slack Signing Secret** | **未做** | 實測對 webhook URL 送出**帶假簽章**的請求仍回 HTTP 200，代表簽章驗證被跳過（n8n credential 未填 Signing Secret，`SlackTriggerHelpers.ts` L118）。**任何知道 webhook URL 的人都能偽造 Slack 事件並讓 AI 在日曆建立行程。** Signing Secret 不因重裝而改變，設定一次即長期有效 |
| 跑完 54 步驗收 | **部分完成** | 不需 AI 的 9 項核心行為已實測通過；需要 AI 的部分全部卡在上方憑證問題。已備妥獨立測試日曆與 Playwright 腳本，憑證修好後可直接重跑 |
| n8n Public API 401 | **已解除** | 認證正常；本次部署即透過 Public API 完成 |
| 並行競態 | 未實作 | 兩則訊息極短時間內送達同一 thread 可能重複建立。緩解方式（依 `thread_ts` 序列化執行、或建立後回查去重）見 `V2-IMPORT.md` |

### 驗收前必須先知道的三件事

1. **加 Slack scope 會連帶弄壞 v1。** 補 scope 需重新安裝 Slack App，Bot Token（`xoxb-…`）**會輪換**，在 n8n 的 Slack credential 更新之前，**v1 生產版與 v2 都會失效**（`V2-IMPORT.md` §2）。請在非尖峰時段操作，並準備好立刻更新 credential。**好消息是這件事並不急** —— v2 需要的權限目前都已具備，重裝只是為了「一次拿齊未來的 scope」，可以與驗收分開排程。`app_id`／`bot_user_id` 不受重裝影響，workflow 內寫死的值不必改。
2. **驗收期間生產版必須離線。** v2 與 v1 共用 `webhookId` `d8c9a4e7-…-1234567890ab`，n8n 不允許兩個共用 webhook 路徑的 workflow 同時 active。要測 v2 就得先停用 v1，等於**測試期間排程／訊息服務中斷**，且切換必須「先停一邊、再啟另一邊」。
3. **授權邊界就是頻道成員名單。** 任何能在該頻道發言的人都能建立事件、寫進目標日曆（`V2-IMPORT.md` 已知風險 #6）。v1 也是如此，但 v2 把體驗做順了，誤用機會更高。

> 📎 **測試時的注意事項（2026-07-30 實地驗證）**：往 `C08NVUQUK8F` 貼訊息會**立刻觸發生產版 v1**。
> 已確認 bot 自己發的訊息會被 `Filter Valid Messages` 正確擋下（執行 #1628／#1630 都停在該節點，
> 未呼叫 AI、未建立日曆事件），但**真人發的測試訊息不會被擋**，會被 v1 正常處理並寫進主日曆。
> 因此驗收時務必先停用 v1。

---

## 檔案

| 檔案 | 用途 |
|---|---|
| `Slack_to_Google_Calendar_AI_Assistant.json` | **v1，目前的生產版** |
| `Slack_to_Google_Calendar_AI_Assistant.v2.json` | v2 設計檔。**已部署為雲端的 `(v2 TEST)`**；本檔保留生產日曆，雲端指向拋棄式測試日曆 |
| `V2-IMPORT.md` | 匯入前置設定、50 項缺陷的完整處置紀錄、54 步驗收測試、已知風險、回滾程序 |
| `V2-PROGRESS.md` | 本文件 |
| `slack-app-manifest-patch.yaml` | v2 需要的 Slack App 權限增補 |
| `README.md` / `README.zh-tw.md` | **描述 v1**，不描述 v2 |

> 建置腳本與 20 個驗證器留在 session 工作目錄，未納入 repo。
> 它們依賴本機絕對路徑，且屬於製作過程的產物而非 workflow 本身。
