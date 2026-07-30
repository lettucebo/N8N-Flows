# v2 匯入與驗收指引

`Slack_to_Google_Calendar_AI_Assistant.v2.json` 是依 spec §14 重新設計的版本。
**現有 `Slack_to_Google_Calendar_AI_Assistant.json`（active 中的生產版）未被覆蓋**，保留作為對照與回滾基準。

---

## ⚠️ 請先讀這段：這份 JSON 的驗證狀態

**它從未在任何真實的 n8n 執行個體上執行過。** 我在製作期間無法連上你的 n8n（Public API 一律回 401），因此：

| 已驗證 | 方式 |
|---|---|
| JSON 可解析、26 節點與 40 條連線一致、全節點皆可從 trigger 到達 | 靜態分析 |
| 節點參數 schema、連線合法性、41 個運算式 | **n8n MCP `validate_workflow`（profile: strict）→ `valid:true`、`errorCount:0`、`invalidConnections:0`、`validConnections:40`**；`validate_node` 26/26 ok |
| 全部 Code node 語法正確 | `AsyncFunction` 編譯（n8n 的 Code node 支援 top-level await，用 `new Function` 會誤報） |
| `Collect Thread Context` 的 thread／圖片／ownership／日曆 ID 退路 | 在 Node.js 中**實際執行** 42 個測試案例（替身直接讀產出 JSON 的 `jsCode`，不讀素材檔） |
| `Parse AI Response` 的年份三態、重複判定、強制建立、事件完整性、全天事件排他端點 | 在 Node.js 中**實際執行**，含壞 JSON 例外 |
| `Filter Valid Messages` / `Should Process` / `Has Images` 的條件 | 對 10 種真實 Slack 事件形狀**實際求值** |
| 路由決策無靜默落空、且不會把殘缺事件寫進日曆 | 對 **workflow JSON 內真正的 Switch 規則**窮舉 **1728** 種輸入組合，落空 0、不安全 0、**靜默抹殺 0** |
| **Parse AI Response → Route Outcome 的接縫**（旗標產出後實際走哪條路） | 從 workflow JSON 讀真實 `jsCode` 與 Switch 規則，`new Function` 實際執行 **21 個端到端案例**，含安全／活性兩條不變式 |
| 6 張卡片的 Slack Block Kit 結構與長度限制 | 每張卡片 × 正常／缺值／超長／含跳脫字元／全 null／日曆已寫入與否／附件被拒／逾時結果未知／多事件更新，共 **30 個情境**實際求值後解析 JSON，並用**語義斷言**檢查文案方向（不只檢查 JSON 合法） |
| 圖片守衛與「被拒附件是否真的顯示給使用者」 | 11 個守衛案例 + **3 個 Build→Parse→Card 接縫測試**（含「卡片顯示的必須是 parse 算出的 `attachmentNotice` 本身」與「拒絕原因要對得上」）|
| **下游 I/O 節點的欄位映射與運算式引用**（Google Calendar 收到的每一個欄位） | **19 條逐欄位契約**斷言 + 實際執行 collect／parse 取得真實輸出欄位集合，驗證 6 個下游節點的 **58 個 `$json.X` 引用全部存在**（先前這一整層完全沒有測試，把 `end` 改成 `$json.startDateTime` 全套仍是綠燈） |
| **Slack 卡片發到哪裡**（channel／thread_ts 各自的取值來源） | 6 張卡片逐一比對收件位址契約。先前把 Not Owner 卡的 `thread_ts` 改成 `$json.channel`（卡片會脫離原 thread，使用者在自己的對話串裡等不到任何回應），18 個驗證器全綠 |
| **路由拓撲的語義**（哪個出口接到哪個節點） | Switch 8 個出口逐一比對期望目標 + 5 條投遞驗證連線。先前把 duplicate 分支改接錯誤卡、連線總數維持 40，全套仍是綠燈 |
| **thread 上下文的生產端**（`allEvents` 有沒有被產出、去重、`latestUserMessage` 的來源） | 8 個案例實際執行 `Collect Thread Context` 的 jsCode。先前只驗消費端，把生產端整段拿掉仍全綠 |
| **每一條「使用者不該被卡住」的脫困路徑** | 17 個案例：force 覆蓋模型矛盾、明示 UTC、附件被拒的 5 種原因、截斷後仍能建立、純閒聊仍靜默 |
| **update 到底改到哪一個事件** | 11 個案例。多事件 thread 中「把 A 改到下週三」過去會靜默更新 B —— 成功卡照出，真實行事曆已被改錯 |
| **錯誤正規化的生產端**（階段判定、錯誤形狀、逾時結果未知） | 17 個案例實際執行 `Normalize Error` 的 jsCode。先前卡片測試直接餵 `uncertainWrite: true`，把算出這個旗標的整段邏輯改成 `false` 依然全綠 |
| **時區守衛**（拒絕 `Z`、保留其他 offset、系統補值不得產生 `Z`） | 9 個案例實際執行，含「Z 的澄清原因必須是 `TIMEZONE_UTC` 而非泛用格式錯誤」的訊息精確度斷言 |
| **卡片可讀時間**（含全天事件 exclusive→inclusive 還原、星期幾、今年省略年份） | 7 個案例實際執行 |
| **同 thread 多事件的重複判定** | 8 個案例實際執行 |
| Rubber Duck 第五輪指出的 8 個問題 | **22 個獨立複現案例**（含正向對照組），逐項確認修正生效且未誤傷正常路徑 |
| 新增防護是否真的有鑑別力 | 對 **38 項**守衛做 **mutation test**（把防護改壞後測試必須變紅），38/38 全數偵測，還原後回到全綠 |
| 「全套」的定義本身不會腐爛 | `run-all.mjs` 自稽核：目錄下每個 `.mjs` 都必須明確歸類為驗證器或已排除（附理由），漏掛與誤掛都會直接紅燈。目前 **20 個驗證器**、7 個已排除 |
| 文件宣稱的數字不會腐爛 | 節點數／連線數／Code 節點數由 `check-doc-sync.mjs` 對照 JSON；MCP 的 `warningCount` 由 `mcp-validate.mjs` 對照本文件。`n8n-mcp` 版本鎖定在 **2.65.1** 並於執行時驗證實際版本相符 |

> mutation test 一律先斷言「變異真的改動了檔案」才判讀結果 —— 否則 `replace` 靜默失敗時，「未偵測」會被誤讀成守衛沒有鑑別力（CRLF 換行就害過一次）。

> **另外兩個踩過的坑，寫下來避免重蹈**：
> 1. `build-workflow-part1.mjs` **不寫檔**，要跑 `part2` 才會產出 JSON。做紅綠驗證時只跑 part1 會拿舊 JSON 去測，得到「mutation 沒被偵測」的假結論 —— 而那看起來剛好就像「測試沒有鑑別力」。一律用 `run-all.mjs`。
> 2. n8n 的 Code node 支援 top-level `await`，用 `new Function` 檢查語法會對它誤報。必須用 `AsyncFunction` 建構子，否則這支驗證器會產生假紅，久了就沒人相信它。

### 驗證體系本身曾經有的盲區（本輪修掉）

這些不是 workflow 的缺陷，是**測試套件的缺陷** —— 它們讓綠燈失去意義，比任何單一 bug 都危險：

| 盲區 | 具體證據 | 處置 |
|---|---|---|
| **下游 I/O 節點完全沒有測試** | 全部驗證器都只驗 `Parse AI Response` 的**輸出**，沒有一支驗證下游怎麼**消費**它。實測：把 `Create Calendar Event` 的 `end` 從 `$json.endDateTime` 改成 `$json.startDateTime`（所有事件長度變成零），**全套仍然全綠** | 新增 `io-contract.mjs`：19 條逐欄位映射契約 + 引用完整性（實際執行 collect／parse 取得真實輸出欄位集合，驗證 54 個 `$json.X` 引用全部存在）。同一個 mutation 現在會以三種方式被抓到 |
| **有一支驗證器驗的是 v1 檔** | `test-b1-date.mjs` 讀 `Slack_to_Google_Calendar_AI_Assistant.json`（無 `.v2`）、尋找的是 v1 才有的節點「Analyze Message with AI」，對 v2 零覆蓋；唯一的 `process.exit(1)` 只檢查自身前置條件；還會呼叫真實 Azure（全套執行時間 3 分鐘的來源，且會因外部因素假紅） | 移出驗證清單並在 `NOT_VERIFIERS` 註明理由。全套執行時間 3 分鐘 → 5 秒 |
| **唯一能對照真實 n8n schema 的工具不會失敗** | `mcp-validate.mjs` 結尾無條件 `process.exit(0)`，而且不在驗證清單內 —— 它被納入流程只是為了讓人用眼睛看輸出，而人是會漏看的 | 改為依 `valid`／`errorCount`／`invalidConnections`／逐節點結果決定 exit code，並納入驗證清單 |
| **合法路由是手寫白名單** | 新增 `not_owner` 後忘了同步，於是 720 個「正確命中 not_owner」的組合被 `route-outcome.mjs` 誤報成「落空」—— 驗證器自己與生產不同步 | 改為從 workflow JSON 的 Switch 規則自動推導 |
| **漏掛的驗證器不會有人發現** | 寫好的驗證器忘了加進清單，等同不存在；掛在清單上卻驗錯對象，同樣沒有機制會發現 | `run-all.mjs` 自稽核：目錄下每個 `.mjs` 都必須明確歸類為驗證器或已排除（附理由），漏掛與誤掛都直接紅燈 |

### n8n MCP 的 23 個 warning —— 逐項處置

`errorCount:0` 但 `warningCount:23`。以下是每一項的判斷與依據，避免下次審查重新推導：

| 數量 | Warning | 處置 | 依據 |
|---|---|---|---|
| 10 | 「Hardcoded nodeCredentialType detected」 | **不處理** | HTTP Request 節點用 `nodeCredentialType: 'slackApi'` 借用 Slack 憑證，這正是 n8n 對「用 HTTP 節點打 Slack API」的建議做法。憑證本體仍在 n8n 憑證庫，JSON 內沒有任何密鑰 |
| 8 | 「Code nodes can throw errors」 | **已處理** | 8 個 Code 節點全部設 `onError:'continueErrorOutput'` 並接到 `Normalize Error`；`validate-workflow.mjs` 的檢查 15 會逐一驗證這條連線存在。MCP 的靜態分析看不到 error output 的接線 |
| 2 | `Create/Update Calendar Event` 的 `calendar` 缺 `cachedResultName` | **刻意不加** | 該欄位的值是運算式 `={{ $json.calendarId }}`，目標日曆**執行期才決定**。填一個固定的顯示名稱會讓 UI 顯示 A、實際寫進 B —— 比顯示 `Choose...` 更糟。MCP 自己也註明 "The workflow will run" |
| 1 | `Slack Message Trigger` 的 `channelId` 缺 `cachedResultName` | **不處理** | 只影響 n8n UI 的下拉顯示，不影響執行 |
| 1 | 「Property 'channelId' won't be used」 | **誤報** | 已逐欄比對：v2 與**生產版**的 Slack Trigger 參數完全相同（`trigger:['message']` + 同一個 `channelId`）。生產版目前 active 且只回應該頻道 —— 真實環境行為勝過靜態分析 |
| 1 | 「DateTime is from Luxon library」 | **資訊性提示** | 非問題 |


| **未驗證** | **風險** |
|---|---|
| **Slack / Google Calendar / Azure OpenAI 的實際 API 回應** | 未發過任何一次真實請求 |
| **端到端流程** | 零次真實執行 |
| **並行競態** | 無 API 無法實測 |

**因此請務必先以測試 workflow + 測試頻道跑完下方驗收，再考慮取代生產版。**

---

## 匯入前必須先做的事

### 1. 以「新 workflow」匯入，不要覆蓋生產版

> ✅ **此步驟已於 2026-07-30 完成。** 已透過 n8n Public API 建立 id `RJiCNKlVQ4EhVyMS`、
> 名稱 `Slack to Google Calendar AI Assistant (v2 TEST)`、`active: false`。
> 生產版 `I2dch7ZKvBvX6GVC` 經前後比對確認未被更動。以下保留原始說明作為原理紀錄。

檔案內 `id` 仍為 `I2dch7ZKvBvX6GVC`、`webhookId` 也與生產版相同。
匯入前請先把 `id` 改成空字串，讓 n8n 產生新 ID：

```powershell
$p = "Slack_to_Google_Calendar_AI_Assistant.v2.json"
$j = Get-Content $p -Raw | ConvertFrom-Json -Depth 30
$j.id = ""
$j.name = "Slack to Google Calendar AI Assistant (v2 TEST)"
$j | ConvertTo-Json -Depth 30 | Set-Content "import-me.json" -Encoding UTF8
```

`active` 已設為 `false`，匯入後不會自動上線。

> ⚠️ **`webhookId` 與生產版相同是刻意保留的** —— 這樣日後正式切換時，Slack App 端的 Request URL 不必變動。
> 代價是**兩個 workflow 絕對不可同時 active**，否則 webhook 路徑會衝突。測試期間請確保只有一個是啟用狀態。

### 2. Slack App 需要新增權限

> 📌 **2026-07-30 實測修正 —— 本節原本的說法有兩處錯誤。**
>
> **錯誤一：不是「五項缺一不可」。** 直接讀 Slack 回應標頭 `x-oauth-scopes`，目前 token 已授予
> 22 個 scope。v2 **今天實際會呼叫**的四個 —— `channels:history`／`chat:write`／`files:read`／
> `reactions:write` —— **全部都已具備**。也就是說，就權限而言 v2 現在就跑得動。
>
> **錯誤二：`metadata.message:read` 並非必要。** 已用真實 Slack API 往返實測：
> 以 `chat.postMessage` 寫入 `metadata`，再以 `conversations.replies?include_all_metadata=true`
> 讀回，**event_id 完整取回**，目前 token 並沒有這個 scope。
> Slack 官方文件也一致：該 scope 屬於 **Events API** 路徑（訂閱 `message_metadata_*` 事件）的要求，
> 而 v2 走的是 **Web API** 讀取路徑，文件對該路徑未要求此 scope。
> 因此「缺它會導致 thread 回覆重複建立事件」的推論**不成立**。
>
> **⛔ 但有一個比上述都嚴重的問題：** App Manifest 的 scope 列表是**整份取代**，不是追加。
> 本檔舊版只列 16 個 scope，而 App 現持有 22 個 —— 直接貼上舊版會**靜默移除 10 個現有權限**
> （`files:write`、`app_mentions:read`、`users.profile:read`、`usergroups:*`、`channels:write.*`、
> `im:read`、`mpim:read`、`remote_files:read`）。
>
> [`slack-app-manifest-patch.yaml`](./slack-app-manifest-patch.yaml) 已改寫為
> **「現有 22 個 ∪ v2 需要的 ∪ 未來可能需要的」的聯集，共 30 個 scope**，
> 每個名稱都對照 Slack 官方 scope 文件逐一查證存在，且 YAML 已通過解析驗證。
> 目的是**只重裝一次**，之後不必再為權限重裝。

以下為原始說明，保留作為設計意圖紀錄（但「缺一不可」的判斷已被上方實測推翻）：

完整清單見同目錄的 [`slack-app-manifest-patch.yaml`](./slack-app-manifest-patch.yaml)。**這五項缺一不可**：

| 用途 | Scope | 缺少的後果 |
|---|---|---|
| 讀 thread 內容 | `channels:history` | `Fetch Thread Replies` 回 `missing_scope`，thread 續談完全失效 |
| **讀回既有事件 ID** | **`metadata.message:read`** | **thread 回覆會被當成新事件，重複建立** |
| 讀取圖片 | `files:read` | `Download Image` 回 HTML 登入頁而非圖片 |
| 發卡片 | `chat:write` | 所有輸出節點失敗 |
| 加 reaction | `reactions:write` | 「不處理」的靜默回饋消失 |

Event 訂閱需含 `message.channels`（若為私人頻道另加 `message.groups`）。

> ⚠️ 重新安裝 App 後 Bot Token（`xoxb-...`）**會變更**，必須同步更新 n8n 的 Slack credential，否則連舊版也會一起失效。建議先在非尖峰時段操作。

### 3. 設定 n8n 變數

> ⛔ **本 instance 做不到這一步，已改用其他方式解決。**
> `GET /api/v1/variables` 回 **HTTP 403 `Your license does not allow for feat:variables`**。
> 實測 Code node 內 `typeof $vars === 'object'` 但 `Object.keys($vars)` 為 `[]` ——
> 也就是**不會拋錯，只會靜默取不到值**，這比直接失敗更難察覺。
>
> 因此部署時採取的替代做法（詳見 `V2-PROGRESS.md`）：
>
> | 變數 | 替代做法 |
> |---|---|
> | `SLACK_APP_ID` | 寫死 `'A08LUAXNTD2'` 為 fallback（非機密，取自 `auth.test`+`bots.info`；App 重裝後不變） |
> | `SLACK_BOT_USER_ID` | 寫死 `'U08MEH30GSU'` 為 fallback（同上） |
> | `GCAL_ID` | 雲端 v2 TEST 直接指向專用測試日曆；repo 的 `.v2.json` 維持生產日曆 |
>
> 寫法一律是 `$vars.X || '寫死值'`，保留 `$vars` 優先 —— 日後若升級授權並設定變數，
> 會自動改走 `$vars`，不需要再改 workflow。

以下為原始設計意圖，保留作為紀錄：

| 變數 | 必填 | 說明 |
|---|---|---|
| `SLACK_APP_ID` | **是** | 用於驗證 metadata 來源與 bot 自我辨識。缺少時他人 app 寫的 metadata 無法被排除 |
| `SLACK_BOT_USER_ID` | 建議 | ✅ reaction 備援需要它才會啟用；未設定時該備援**不啟用**（不會誤判為已建立） |
| `GCAL_ID` | 否 | 目標日曆；未設定時沿用節點內的預設值 |

## 主要變更

| 節點 | 變更 |
|---|---|
| `Filter Valid Messages` | **移除 `thread_ts` 必須為空的條件** —— thread 回覆現在會觸發；新增 `app_id` 防護 |
| `Fetch Thread Replies` | 新增。`conversations.replies?include_all_metadata=true`，用於讀回既有事件 ID |
| `Collect Thread Context` | 新增。thread 發起人、對話輪次、附件分類、既有事件判定、**本 app 參與防護**、**全 thread 圖片收集** |
| `Split/Download/Aggregate Images` | 新增。圖片下載並以 `includeBinaries:true` 聚合 |
| `Build Azure Payload` | 新增。日期基準以 JS 求值後內嵌；圖片轉 base64 併入 `image_url` |
| `Call Azure OpenAI` | 由 `chainLlm` 改為 HTTP Request，`retryOnFail:3` + 錯誤出口 + `response_format` |
| `Parse AI Response` | 內嵌年份三態判定；計算 `forceCreate` / `isDuplicate`；**事件完整性守衛**；**全天事件排他端點** |
| `Route Outcome` | 取代原本三層 IF。**`force` 排在 `clarify` 之前**，且有無條件 fallback |
| `Normalize Error` | 新增。集中處理 8 個節點的例外，以 try/catch 保護 `$()` 引用，回推失敗階段 |
| `Verify Card Delivery` | 新增。攔截 Slack `HTTP 200 + {ok:false}`，避免 metadata 靜默寫入失敗 |
| 所有 Slack 發送 | 一律改為 HTTP Request `chat.postMessage`。真正的原因是 (1) attachment 色條要內嵌 blocks、(2) 需要寫入 message `metadata` —— Slack node 本身其實**有** `messageType:'block'` |

### 本輪（Rubber Duck / Council 審查後）修掉的確定性缺陷

這些都是**會讓功能失效或污染真實日曆**的問題，不是風格建議：

| # | 問題 | 後果 | 修法 |
|---|---|---|---|
| 1 | `Filter Valid Messages` 的 `contains` 方向寫反 | **整個 workflow 一則訊息都不會處理** | 改為 `["message","file_share"].includes(...)` |
| 2 | Code 中寫 `$('Slack Trigger')`，實際節點名是 `Slack Message Trigger` | 執行期必定拋錯 | 修正節點名，並讓測試 harness 從 workflow JSON 讀真實名稱 |
| 3 | `$('未執行節點')` 是**拋例外**而非回傳 falsy，三元運算子攔不到 | 錯誤卡片本身發不出去 | 改由 `Normalize Error` Code node 以 try/catch 保護 |
| 4 | `force` / `update` 路由不檢查 `hasValidEvent` | 使用者回「就這樣建立」但 AI 沒解析出事件時，**送 title=null 的空事件進真實日曆** | 兩條規則都加上 `hasValidEvent`，且該旗標現在要求 title 非空、start/end 可解析、end > start |
| 5 | 全天事件的 `end.date` 未 +1 | Google Calendar 的 `end.date` 是**排他**的，單日事件會變成零長度或被拒 | `isAllDay` 時 end 一律 +1 天（含月底跨月） |
| 6 | Update 節點**沒有 `allday` 欄位** | n8n 的 all-day 分支條件是 `allday==='yes' && start && end`，缺一就會停在 dateTime 分支，把全天事件默默改成 24 小時定時事件 | 補上 `allday` |
| 7 | Code node 例外無 error output | AI 回壞 JSON 時，**使用者永遠等不到任何回應**（最糟的失敗模式，因為是靜默的） | 5 個 Code node 全部導向 `Normalize Error` |
| 8 | Slack `HTTP 200 + {ok:false}` 被當成功 | 成功卡片的 metadata 是 event ID 的唯一回讀來源，靜默寫入失敗 → **下一輪對同一件事重複建立** | 新增 `Verify Card Delivery` |
| 9 | 澄清輪不重送原圖 | Azure OpenAI 每輪都是無狀態呼叫，第二輪必定失去圖片脈絡 —— 正好打在「圖片 + 續談」兩大需求的交集 | 改收全 thread 圖片，依 file id 去重、上限 4 張 |
| 10 | thread 訊息無時間戳 | thread 跨數日時，模型會用本次執行時間重新解讀歷史訊息裡的「下週三」 | 每則訊息附台北時間 |
| 11 | 純圖片訊息被過濾掉 | `filter(m => m.text)` 讓無文字的純圖片訊息消失，round 少算 | 改用「文字或附件任一非空」判斷 |
| 12 | `settings` 未對齊生產版 | v2 新增圖片下載，binary 資料處理需與生產版一致 | 對齊生產版 settings 與 `meta.instanceId`（含 `executionOrder:'v1'`）。⚠️ 註：`binaryMode` 這個 **workflow 層設定並不控制 binary 存在哪裡** —— 真正決定的是 n8n 環境變數 `N8N_DEFAULT_BINARY_DATA_MODE`（`default`／`filesystem`／`s3`）。若你的實例用預設值，圖片會存進資料庫，請自行評估是否改設為 `filesystem`。 |
| **13** | **非全天事件缺 `endDateTime` 時無任何預設值** | 「明天早上開會」這種**最常見的輸入**會讓 `endMs=NaN` → `hasValidEvent=false` → 命中 suppress → **使用者只收到一個 ➖ reaction，完全不知道發生什麼事** | 補 `DEFAULT_DURATION_MS = 60 分鐘`；prompt 中 `endDateTime` 改標**必填**並要求依常識估時長 |
| **14** | **suppress 規則誤用 `hasValidEvent`** | 只要資料不夠完整就靜默抹殺，與「資訊不足時應該追問」的設計意圖直接矛盾 | 拆成兩個語義不同的旗標（見下方說明），suppress 改看 `aiHasEvent` |
| **15** | `Verify Card Delivery` 寫 `$json.ok === false` | Slack 遇 5xx／網路硬失敗時，n8n 的 `continueRegularOutput` 會把 `{error: msg}` 推到**正常輸出**，此時 `ok` 是 `undefined`，`undefined === false` 為 false → **被當成投遞成功**，metadata 未寫入 → 下一輪重複建立事件 | 改為 `j.ok !== true`（白名單而非黑名單） |
| **16** | `rejectedFiles` 完全不告知使用者 | 關鍵資訊剛好在被丟棄的那張圖上時，使用者收到一張看似正常的卡片，卻不知道有東西沒被讀到 | 成功卡片與澄清卡片的 context 尾端 append 提示（不新增 block，避免 Slack 摺疊） |
| **17** | `Send Error Card` 的 `stage` 未截斷 | 錯誤來源字串可能長達數千字元 → section text 超過 Slack 的 3000 上限 → **錯誤卡片本身被拒**，使用者對故障一無所知 | `String($json.stage \|\| 'unknown').slice(0, 80)` |
| **18** | 純日期 + 模型標 `isAllDay:false` | 「明天請假」被 `Date.parse` 當成 UTC 午夜 → 台北 08:00，再補 1 小時預設時長 → 日曆上出現**早上 8 點到 9 點的請假** | `isAllDay` 改以**資料格式**判定：`!!ev.isAllDay \|\| /^\d{4}-\d{2}-\d{2}$/.test(start)`。格式是客觀事實，旗標只是模型的判斷，衝突時信格式 |
| **19** | `Date.parse` 對不存在的日期靜默正規化 | `2026-02-30T09:00` 被 JS 悄悄變成 `2026-03-02`，`isNaN` 檢查完全抓不到 → **日曆上出現使用者從未同意的日期** | 新增 `calendarDateValid()`：把字串拆成年／月／日再與 `Date` 物件逐欄比對，不符即拒（可擋 2/30、非閏年 2/29、13 月等） |
| **20** | 全天卻帶時刻／定時卻無時區 offset | 兩種都會讓 n8n 走錯分支，產生零長度事件或午夜事件 | 新增 `shapeValid()`：全天必須是純 `YYYY-MM-DD`；定時必須是帶 offset 的 RFC3339 |
| **21** | `isDuplicate` 用 `slice(0,10)` 只比日期 | 同一天 09:00 與 11:00 的兩場同名會議，第二場被誤判為重複而擋掉 | 改用 `sameInstant()` 比完整時刻；全天事件才退回比日期 |
| **22** | 圖片下載失敗被當成「本來就沒圖」 | `Build Azure Payload` 完全不檢查下載結果，失敗的圖直接消失 → 使用者不知道自己貼的圖從未被讀過 | 新增下載失敗／格式不符／超過大小三重偵測，全部併入 `rejectedFiles`（缺陷 16 的機制已會顯示給使用者） |
| **23** | 日曆已建立、卡片投遞失敗，錯誤訊息卻叫使用者重試 | 使用者照做 → **日曆上出現兩個重複事件**。這是最惡劣的一種錯誤：格式完全合法，語義完全相反 | `Normalize Error` 回讀 `calendarEventId`／`calendarLink`，錯誤卡片文案依 `calendarWritten` 分岔：已建立 → 附連結並明說「請勿重送」；未建立 → 才請使用者重試 |
| **24** | prompt 命令「所有相對日期一律以上述台北時間計算」 | 與「每則訊息附時間戳」的用意直接矛盾 —— 三天前的「明天」會被算成明天，**舊 thread 的歷史日期整批平移** | 改為「歷史訊息中的相對日期，以**該則訊息自己的時間戳**為基準」 |
| **25** | `calendar` 參數硬編碼，`$vars.GCAL_ID` 不生效 | 文件宣稱可用變數指定日曆，實際上是空話 | 見下方「目標日曆的指定方式」 |
| **26** | 日曆 ID 守衛比 Google 的 `extractValue` 寬 | n8n 的 Google Calendar 節點對 `calendar` 欄位設有 `extractValue` regex，比不到就 `throw`。舊守衛放行 `"'(),:;<>[\]` 等 12 個字元，這些值會過守衛卻在 Google 端炸掉。**更危險的是該 regex 沒有 `$` 錨點**：`a@b@x.com` 被靜默截斷成 `a@b`，事件寫進**錯誤的日曆而完全不報錯** | 用窮舉推導出 Google 字元類的真子集並補上 `$` 錨點；`GCAL_ID` 被拒時 `console.log` 警告（不 fail fast —— 那是一次性設定錯誤，讓每次執行都炸只會擴大災情）。⚠️ 查核時請注意 Google 那條 regex 裡的撇號是 **U+2019 右單引號**、不是 ASCII `'` |
| **27** | 全天事件的原始 `end` 在被驗證之前就已被改寫 | `addOneDay('2027-02-30')` 會被 `Date.parse` 正規化成 `2027-03-03`，帶時刻或無法解析的 `end` 則被三元運算直接換成 `start` —— 兩種情況下原始非法值都在 `shapeValid` 看到它之前就消失了 | 把合法性判定移到覆寫**之前**，並用 `allDayEndRejected` 區分「明確非法」與「真的沒給」。前者走 clarify 請使用者確認，後者才 fallback 成 start+1 |
| **28** | 圖片全數下載失敗時仍照常建立事件 | 使用者貼了圖、AI 一張也沒看到，卻用純文字猜出一個高信心事件直接寫進日曆 —— 使用者以為系統讀懂了圖 | 新增 `imagesAllFailed` 旗標，suppress／update／create 三條規則都加上排除。**刻意不加在 force**：否則使用者第二輪說「就這樣建立」也脫不了困 |
| **29** | 全天事件與定時事件被誤判為重複 | `sameInstant` 寫成「任一側是純日期就比日期」，於是「7/30 全天 出差」存在時，使用者無法再建立「7/30 10:00 出差進度會」 | 改為兩側形狀一致才比較：一側全天、一側定時直接判為不同事件。`intent=update` 的正常更新路徑不受影響（Switch #1 早於 #5，與 `isDuplicate` 無關） |
| **30** | 字串型的 error 被轉成 `unknown error` | HTTP 逾時與 Slack API 失敗常見的形狀是 `{ error: 'connect ETIMEDOUT ...' }`。舊寫法先做 `j.error \|\| {}` 再讀 `.message`，字串沒有該屬性 → `undefined` → 一路落到 `unknown error`，**對排查最有用的那一種訊息剛好被丟掉** | 改為有序候選清單，字串型 `error` 優先採用 |
| **31** | `Send Updated Card` 不寫 metadata | 事件從 09:00 改到 11:00 後，thread 中最新的 metadata 仍停留在建立當下。使用者再說 09:00 會被判重複而擋掉（那已不存在），說 11:00 反而判為新事件而**建立第二筆** | 更新卡片比照成功卡片寫入 `event_id/calendar_id/start/title`，並新增 `Verify Updated Card Delivery`（共用同一段驗證邏輯，避免兩者日後失去同步） |
| **32** | AI 回傳 `Z` 結尾的時間被直接寫入日曆 | prompt 要求 `+08:00`，但模型偶爾把「台北 09:00」寫成 `09:00Z`。**已實證**：該事件會被建立在台北 17:00，而卡片顯示的仍是使用者預期的字串 —— 差 8 小時且完全看不出錯，屬於靜默寫錯而非明顯失敗 | `DATE_TIME_OK` 收緊為必須以 `[+-]HH:MM` 結尾；另設 `UTC_Z_SHAPE` 專門分類此情況，澄清原因給 `TIMEZONE_UTC`（訊息說「請說明這是哪個時區」而非泛用的「格式不正確」）。**不擋其他 offset** —— `+09:00` 是「東京時間 9 點」的合法輸出 |
| **33** | 非建立者被導向澄清卡，形成無解迴圈 | 他人在 thread 中要求修改時，`update`／`create` 都因 `isOwner=false` 不成立，落到 fallback 的 clarify，看到「資訊不足，請補充」。但他補再多資訊 `isOwner` 也不會變 true。把「權限不足」偽裝成「你講得不夠清楚」，使用者只會反覆嘗試然後放棄 | 新增 `not_owner` 路由與專屬卡片，明講原因並給一條真正走得通的路（開新訊息建立自己的行程）。排在 `suppress` 之後 —— 他人單純閒聊仍應安靜略過，不該收到權限卡片 |
| **34** | 澄清卡與重複告知卡的投遞失敗完全靜默 | Slack 常以 HTTP 200 回 `{ok:false}`（權限失效、頻道封存、rate limited），HTTP 節點不會判定失敗。**澄清卡是使用者唯一的脫困路徑**，送不出去等同對話卡死，而 n8n 執行紀錄仍顯示這次「成功」 | 新增 `Verify Notice Delivery`，clarify／duplicate／not_owner 三張卡共用（與成功卡同一段驗證邏輯），失敗導向錯誤卡片 |
| **35** | `Create Calendar Event` 設了 `retryOnFail` | Google Calendar 的 insert 非冪等且無 idempotency key。逾時／5xx 時事件往往已在 Google 端建立成功、只是回應沒回來，重試就建立第二筆；而且只有第二筆的 event id 會寫進 metadata，第一筆從此無法被本流程管理 | 移除該節點的 `retryOnFail`。`Update Calendar Event` **保留**重試 —— 對同一 event id 重送相同內容是冪等的 |
| **36** | 同 thread 多個事件時只比對最後一筆 | `Collect Thread Context` 掃描訊息時無條件覆寫 `existingEvent`。thread 先建立 A 再建立 B 之後，使用者再說一次 A 只會跟 B 比對 → 判定不重複 → **A 被重覆寫進日曆** | collect 另外輸出 `allEvents`（全部），重複判定掃過每一筆；`existingEvent` 仍保留最後一筆供 update 使用（「改成…」指的幾乎必然是最近建立的那個） |
| **37** | 卡片顯示原始 ISO 字串與信心度百分比 | `2027-03-01T09:00:00+08:00` 需要使用者自己在腦中解析，且看不出星期幾 —— 而「那天是星期幾」正是他檢查有沒有排錯最常用的線索。信心度則是模型內部分數，使用者無從解讀也無從行動 | `Parse AI Response` 預先算好 `displayTime`（`2027年3月1日（一） 09:00–10:30`，今年省略年份、全天事件還原 exclusive 結束日）；移除所有卡片的信心度欄位；地點改為有值才顯示 |
| **38** | 澄清卡不說明「具體缺什麼」 | 只講「我對這個行程的把握不夠」，使用者不知道要補哪一項，只能重講一次然後再被擋一次 | `clarifyReasons` 分成 10 種具名原因並依嚴重度排序（`THREAD_TRUNCATED` 最優先 —— 那是系統層阻擋，補資料無用）；卡片對每種原因給對應文案與可行動的下一步。另加 `forceIgnored`：使用者說了「就這樣建立」卻仍不能建立時，卡片必須先承接這句話（`✋ 我收到你說的「就這樣建立」了 —— 但還差一項才能建立：`），否則他會認為系統聽不懂人話 |

### 第二輪 Council / Rubber Duck 之後再修掉的缺陷

第一輪修完之後又跑了一次 Council 與 Rubber Duck。以下 12 項是那一輪的產物。
**其中最值得記住的不是任何單一缺陷，而是一個模式**：這 12 項在被發現之前，
既有的驗證器**一次都沒有變紅**。每做完一組實質行為變更、跑全套、全綠 —— 然後才發現
測試對這些行為根本沒有斷言。因此每一項的處置都同時包含「補上會變紅的測試」。

| # | 缺陷 | 為什麼是缺陷 | 處置 |
|---|---|---|---|
| **39** | 附件被拒的原因有一類沒有分類 | `R_TOO_LARGE`（檔案超過大小上限）不在分類表內，落到「其他」；而分類表本身沒有 catch-all，未來新增的拒絕原因會直接消失 | 補上 `R_TOO_LARGE` 文案，並加 catch-all 分支計數未知原因，確保任何拒絕都說得出一個理由 |
| **40** | 附件全被拒時仍可能靜默 suppress | 使用者傳了一張圖但全部沒讀到 → 模型看不到內容 → 回 `hasEvent:false` → suppress 規則成立 → **他傳了圖，得到的是一個 ➖ reaction**。第 22 步驗收看的是「有讀到但被拒」，這個是「一張都沒讀到」，不同路徑 | suppress 規則加 `!attachmentNotice`；並在 `clarifyReasons` 插入 `ATTACHMENT_FAILED`（排在 `THREAD_TRUNCATED` 之後），卡片明說該重傳還是換格式 |
| **41** | `allEvents` 沒有去重 | 同一個事件被建立卡與更新卡各寫一次 metadata 時會出現兩筆，重複判定與 update 目標挑選都會被這些幽靈項干擾 | 改用 `Map` 以 `eventId` 去重，保留最後一次出現的狀態（那才是事件的當前樣貌） |
| **42** | `latestUserMessage` 讀的是 thread 歷史最後一則 | Slack 官方文件明載 `conversations.replies` 回的是**最舊的 N 筆**；thread 超過 limit 時，**觸發這次執行的那則訊息根本不在回應裡**。使用者在長 thread 說「就這樣建立」，系統讀到的是別人幾十則之前的話 —— `forceCreate` 判定失效，脫困路徑靜默消失 | 改讀 trigger 事件自己的 `text`。刻意**不**做「讀不到就退回 thread 前一則」的 fallback —— 純圖片訊息的 text 為空，退回會拿別人的話去誤觸 `forceCreate` |
| **43** | `force` 無法覆蓋模型自相矛盾的輸出 | 模型偶爾回 `hasEvent:false` 卻在 `events[0]` 給了完整事件。此時 `hasValidEvent` 為 false，使用者說「就這樣建立」也建不了，而他看得到卡片上的事件內容 —— 「你明明知道，卻說你不知道」 | `hasValidEvent` 改為 `(ai.hasEvent \|\| forceCreate) && ev && startDateTime && eventComplete`。**門檻沒有降低**：事件物件必須存在、標題與起訖時間仍逐項檢查 |
| **44** | 使用者明說 UTC 仍被時區守衛擋下 | 缺陷 32 的守衛拒絕所有 `Z` 結尾。但使用者若明白寫「UTC 15:00 的會議」，模型輸出 `Z` 是**正確**的 —— 系統卻回「請說明這是哪個時區」，他已經說了。再說一次還是同樣的卡片，形成迴圈 | 新增 `EXPLICIT_UTC` 偵測（`UTC`／`GMT`／`Zulu`／世界協調時間／格林威治，皆帶單字邊界），命中時把 `Z` 正規化成 `+00:00`（語義完全等價）。`\b` 邊界確保 `outcome` 之類字串裡的 `utc` 不會誤觸 |
| **45** | 逾時錯誤卡叫使用者「稍後重試」 | Google Calendar 逾時的時候，請求很可能**已經抵達並建立成功**，只是回應沒回來。叫他重試就是叫他製造重複事件 —— 這是錯誤訊息本身在製造缺陷（與缺陷 23 同型，但這次在逾時路徑上） | `Normalize Error` 新增 `uncertainWrite`（逾時樣態 **且** 確實走到日曆那一步 **且** 尚未確認寫入）。錯誤卡改口「請先到 Google 日曆確認是否已經建立」，不再叫他重試 |
| **46** | 更新卡沒有 Google 日曆連結 | 成功卡有連結，更新卡完全沒有。使用者剛改完時間，最想做的就是點進去看一眼 | 更新卡加上 `htmlLink`，並比照成功卡加缺值防護（拿不到連結時不顯示壞掉的空連結） |
| **47** | 澄清卡顯示 ISO 原文、空欄位無法行動 | 缺陷 37 把成功卡的時間美化了，澄清卡卻還是 `2027-03-01T09:00:00+08:00`；而缺少的欄位顯示「（尚未取得標題）」—— 那是狀態描述，不是可以做的事 | 澄清卡改用 `displayTime`；空欄位改成可行動指引（`還沒有標題 — 告訴我這是什麼活動`） |
| **48** | `calendarWarning` 算了但沒人顯示 | 日曆 ID 被拒、事件寫進 fallback 日曆時，只有 n8n 的 Console 分頁看得到。使用者拿到的是一張綠色成功卡 | 成功卡與更新卡的頁尾都顯示 `calendarWarning` |
| **49** | `E_BAD_JSON` 會把 AI 原文 500 字寫進 log | 模型回傳的內容可能含使用者貼在 thread 裡的私人資訊；n8n 執行紀錄的保存期限與存取範圍都與 Slack 頻道不同 | 縮到 120 字。仍足以判斷「是不是 JSON 被截斷」這個最常見的原因 |
| **50** | 多事件 thread 的 update 會靜默改錯事件 | `eventId` 一律取 thread 裡最後建立的那一筆。thread 裡已有 A、B 兩場會議時，使用者說「把 A 改到下週三」，系統會安靜地更新 **B** —— 他收到「已更新」，真實行事曆已經壞了，而且沒有任何訊號 | 先用標題比對挑目標（完全相同 → 唯一部分相符）。比對不到時**不擋**（使用者可能正是要改標題，擋住就是死路），改為退回最後一筆並標記 `updateAmbiguous`，由更新卡明說「這個執行緒裡有 N 個事件，我更新的是「X」（原時間 …）。若不是這一個，請告訴我要改哪一個」 |

### 目標日曆的指定方式（缺陷 25 的完整處置）

兩個 Google Calendar 節點的 `calendar` 已改為 `{ "__rl": true, "mode": "id", "value": "={{ $json.calendarId }}" }`，
`calendarId` 由 `Parse AI Response` 的 `pickCalendarId()` 決定，優先序為：

1. 既有事件的 `calendarId`（從卡片 metadata 回讀）—— 確保 update 一定寫回原本那本日曆
2. `$vars.GCAL_ID`
3. 內建 fallback

這是一個**核心路徑上的改動**，決策前已用 n8n **2.32.5** 原始碼逐段查證，證據如下：

| 疑慮 | 原始碼位置 | 結論 |
|---|---|---|
| `mode:'id'` 的 email validation regex 會不會擋掉運算式？ | `packages/workflow/src/node-helpers.ts` L1314-1317：`if (valueToValidate.startsWith('=')) return [];` | **不會**。以 `=` 開頭的值直接跳過整個 validation |
| regex 是對「求值前」還是「求值後」的字串跑？ | `packages/core/.../node-execution-context.ts`：先 `getParameterValue()`（L52x）→ `cleanupParameterData`（L536）→ **`extractValue()`（L555-556）** | **求值在先**，抽取拿到的已是真正的字串 |
| Google Calendar 節點怎麼取這個參數？ | `GoogleCalendar.node.ts` L165 / 217 / 367 / 385 / 431 / 590 皆為 `getNodeParameter('calendar', i, '', { extractValue: true })` | 六處一致 |
| `cachedResultName` 缺失會不會影響執行？ | 執行期路徑完全不讀它；n8n MCP 也明示「The workflow will run」 | **不影響**，只有 UI 下拉顯示 |

⚠️ **但查證過程中發現一個真正的風險**，而且是這個改動自己引入的：

`packages/core/.../utils/extract-value.ts` L46-50 的 `executeRegexExtractValue()`：
```ts
const extracted = regex.exec(value);
if (!extracted) {
  throw new WorkflowOperationError(
    `ERROR: ${parameterDisplayName} parameter's value is invalid. ...`);
}
```
**extractValue regex 比不到就直接拋錯，節點整個失敗。** 而 Google Calendar 的 `mode:'id'`
extractValue regex 要求 email 形式。也就是說只要 `calendarId` 求值後不是 email，建立事件就會爆掉。
已知會踩到的兩種真實情境：

- v1 時代產生的舊卡片 metadata **沒有** `calendar_id` 欄位 → `undefined`
- 使用者把 `GCAL_ID` 設成 `primary`（Google Calendar API 完全合法，但不符 n8n 的 regex）

因此 `pickCalendarId()` 對每個候選值都先用 `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` 過濾，
不合格就往下一個候選走，全部不合格才用 fallback。**寧可寫進預設日曆，也不要讓節點爆掉。**

同樣的過濾也套用在**上游**的 `Collect Thread Context` —— 它原本在 metadata 缺 `calendar_id` 時
退回字面 `'primary'`。`primary` 對 Google Calendar API 完全合法，但**不含 `@`**，
會在更新事件時觸發上述的 `throw`。已改為退回 email 形式的日曆 ID（帳號 primary 日曆的
真實 ID 本來就是它的 email，語義等價）。兩處各有一道過濾，形成雙重防線。

三個方案的取捨：

| 方案 | 優點 | 缺點 |
|---|---|---|
| 回退成硬編碼 | 零風險 | `GCAL_ID` 永遠是空話；update 時無法寫回原本的日曆 |
| 運算式但不過濾 | 程式碼最短 | 舊 metadata 或 `primary` 會讓**建立事件整個失敗** |
| **運算式 + email 過濾（採用）** | `GCAL_ID` 真正生效；update 寫回正確日曆；非法值自動降級不會拋錯 | 不支援 `primary` 這類非 email 的合法 Google ID |

不支援 `primary` 是 Google Calendar 節點本身的限制（它的 extractValue regex 就只認 email），不是本次引入的。
**若你要指定日曆，請填該日曆的完整 email 形式 ID**（在 Google 日曆設定頁可以找到，通常長得像
`xxxxx@group.calendar.google.com`）。

這 9 個案例已編碼進 `validate-workflow.mjs` 的檢查 22，上游那 3 個案例編碼進 `test-collect-v2.mjs`
的 RD-D／D2／D3，兩邊都用 mutation test（拿掉過濾後必須分別出現 6 個與 3 個 FAIL）確認過真的有鑑別力。

> ⚠️ 順帶更正一個容易誤解的點：這個改動**沒有**讓 JSON 不含使用者 email —— fallback 值仍寫在
> `Parse AI Response` 與 `Collect Thread Context` 的程式碼裡，只是從日曆節點搬到了 Code 節點。
> 它的價值在於「`GCAL_ID` 真的能用」與「update 寫回正確的日曆」，不在於隱藏 email。

### `aiHasEvent` 與 `hasValidEvent` —— 為何必須拆成兩個旗標

這是本輪最重要的設計修正。原本用單一旗標同時決定兩件事，導致**兩種相反的災難無法同時避免**：

| 旗標 | 語義 | 用來決定 |
|---|---|---|
| `aiHasEvent` | 模型認為「這則訊息在講一件有時間的事」 | **要不要回應使用者**（suppress 只能看這個） |
| `hasValidEvent` | 資料完整到可以安全寫入日曆 | **能不能寫日曆**（create / update / force 都要求這個） |

不變式：`hasValidEvent = true` 蘊含 `aiHasEvent = true`（反向不成立）。

兩條保證必須同時成立，缺一不可：

- **安全不變式** — 走 create / update / force 的路徑，`hasValidEvent` 必為 true → 防止污染真實日曆
- **活性不變式** — `aiHasEvent = true` 時絕不可走 suppress → 防止靜默抹殺

這兩條都已編碼進自動化測試（`route-outcome.mjs` 窮舉 1728 組合、`test-parse-route.mjs` 端到端 19 案例），任何一條被違反測試就會紅燈。

## 驗收測試（請在測試頻道逐項執行）

| # | 送出內容 | 預期 |
|---|---|---|
| 1 | `明天下午三點開會` | 建立事件 + 綠色成功卡片 |
| 2 | 在 #1 的 thread 回覆 `改成四點` | 更新事件 + 藍色卡片，**不新增第二個事件** |
| 3 | 在 #1 的 thread 回覆 `謝謝` | 無動作（不重複建立） |
| 4 | 在 #1 的 thread 回覆 `對了下週四也要開會` | **建立第二個事件**（這是 B-2 修正的重點） |
| 5 | `3/15 健檢` | 澄清卡片詢問年份，並建議 `2027-03-15` |
| 6 | 在 #5 的 thread 回覆 `就這樣建立` | **建立事件**（這是 B-1 修正的重點；修正前會無限追問） |
| 7 | 上傳一張含時間的截圖 | 卡片需顯示「我從圖片讀到」的轉錄內容供核對 |
| 8 | 由**他人**在你的 thread 回覆 `改成五點` | 應被擋下，不得修改你的事件 |
| 9 | 隨意閒聊一句 | 只加 ➖ reaction，不發訊息 |
| 10 | 觀察 bot 自己發的卡片 | **不得**再次觸發 workflow |
| 11 | 上傳無年份的圖 → 收到澄清卡 → 在同 thread 回 `對，2027` | **第二輪仍須看得到原圖**（rd-json #2 修正的重點）；卡片內容應反映圖上資訊 |
| 12 | `8/1 健檢`（全天） | Google 日曆上應只佔 **8/1 一天**，不是零長度、也不是跨到 8/2（缺陷 20 的形狀驗證） |
| 13 | `8/1 到 8/3 出差`（全天多日） | 日曆上應涵蓋 8/1、8/2、8/3 三天 |
| 14 | 在 #12 的 thread 回 `改成 8/5` | 更新後仍須是**全天事件**，不可變成 00:00–00:00 的定時事件 |
| 15 | 兩位同事互相 thread 回覆、bot 從未參與 | 不應介入（不加 reaction、不發訊息） |
| 16 | 一次上傳 5 張以上的圖 | 只處理前 4 張，且卡片頁尾**必須出現**「⚠ 有 N 個附件未被讀取」的提示（缺陷 16 —— 被丟棄的附件必須讓使用者知道） |
| 17 | `明天早上開會`（只有開始時間、沒說到幾點結束） | **必須有回應**（成功卡或澄清卡），自動補 1 小時；**絕不可**只回一個 ➖ reaction（缺陷 13） |
| 18 | `下週約一下`（有意圖但完全沒時間） | 應收到**澄清卡片**詢問時間，而非靜默略過（缺陷 14 —— suppress 只看「AI 是否認為有事件」，不看資料是否完整） |
| 19 | `明天請假`（純日期、沒說幾點） | 日曆上必須是**整天的請假**；**絕不可**變成「早上 8 點到 9 點」（缺陷 18 的迴歸點） |
| 20 | `2/30 開會` 或 `2027/2/29 開會` | 應收到**澄清卡片**指出日期不存在；**絕不可**被靜默改成 3/2 或 3/1 就建立（缺陷 19） |
| 21 | 同一天送 `9點開會` 與 `11點開會`（同樣主旨） | **兩個事件都要建立**；第二個不可被誤判成重複（缺陷 21） |
| 22 | 上傳一個非圖片檔（如 PDF）或超大圖 | 卡片頁尾**必須出現**「有附件未被讀取」提示，不可假裝沒這個附件（缺陷 22） |
| 23 | 讓卡片投遞失敗（例如暫時把 bot 移出頻道）但日曆已建立 | 錯誤訊息必須說「事件已建立」並附連結、明說**請勿重送**；**絕不可**叫使用者重試（缺陷 23，會造成重複事件）。同時檢查 n8n 執行記錄：`Verify Card Delivery` 必須**攔下**這次投遞（缺陷 15 —— Slack 回 `{ok:false}` 或連線失敗時 `ok` 為 `undefined`，兩者都必須算失敗） |
| 24 | 把 `$vars.GCAL_ID` 故意設成非法值（例如 `my calendar` 或 `a@b@x.com`），送一則正常訊息 | 事件應建立在 **fallback 日曆**，且 n8n 執行記錄的 Console 分頁必須出現 `[GCAL_ID]` 警告。**絕不可**靜默寫進某個非預期的日曆（缺陷 26）。確認後把值改回正確的日曆 ID，再送一則訊息確認事件**確實建立在該日曆上**（缺陷 25 —— `$vars.GCAL_ID` 必須真的生效，而不只是文件上寫著支援） |
| 25 | `2/30 到 3/5 出差`（全天、結束日合法但開始日不存在） | 應收到**澄清卡片**；**絕不可**被 `Date.parse` 正規化成 3/2 後照樣建立（缺陷 27） |
| 26 | 上傳圖片，但讓下載失敗（例如把 Slack token 的 `files:read` 權限暫時移除），訊息文字另外寫得像個事件 | 應收到**澄清卡片**說明圖片讀取失敗；**絕不可**用純文字猜出的內容直接建立事件（缺陷 28） |
| 27 | 承 #26，在同 thread 回覆 `就這樣建立` | **必須能建立**。這是刻意保留的脫困路徑 —— 圖片失敗不得讓使用者卡死（缺陷 28 的反向驗收） |
| 28 | 先送 `8/10 全天 出差`，建立後在**新 thread** 送 `8/10 10點 出差`（同名） | **兩個都要建立**。全天與定時是不同型態的事件，不可互相誤判為重複（缺陷 29） |
| 29 | 承 #28 的全天事件，在**原 thread** 回覆 `改成 10 點` | 應走**更新**而非建立第二筆 —— 確認缺陷 29 的修正沒有連帶擋掉正常更新路徑 |
| 30 | 讓 Azure OpenAI 呼叫逾時（例如暫時把 endpoint 改成不可達的位址） | 錯誤卡片必須顯示**具體的連線錯誤字串**（如 `connect ETIMEDOUT ...`）；**絕不可**顯示 `unknown error`（缺陷 30） |
| 31 | 建立 `8/20 09:00 面試` → 在 thread 回 `改成 11 點` → 更新成功後，於同 thread 再送一次 `8/20 11:00 面試` | 應被判為**重複**並提示；若改送 `8/20 09:00 面試` 則應**建立新事件**（因為 09:00 那筆已不存在）。這驗證更新卡片的 metadata 有跟著更新（缺陷 31） |
| 32 | 讓上游丟出一個超長錯誤（例如把 Azure OpenAI endpoint 改成會回傳大量 HTML 錯誤頁的位址） | **錯誤卡片本身必須送得出來**。`stage` 欄位會被截成 80 字元；若未截斷，section text 會超過 Slack 的 3000 上限而讓錯誤卡片整張被拒，使用者對故障一無所知（缺陷 17） |
| 33 | 隔天再回到某個舊 thread（例如 3 天前建立的），在裡面回覆 `那改到明天好了` | 「明天」必須以**你這則新訊息的時間**為基準，而不是原始訊息的時間，也不是把整個 thread 的歷史日期一起平移（缺陷 24）。可搭配一則 3 天前寫著「下週三」的訊息一起確認：那則的「下週三」應維持它當時算出的日期 |
| 34 | **請另一位同事**在你建立的 thread 裡回覆 `改到下午 3 點` | 他必須收到 🔒 **「這則行程不是你建立的」**卡片，內容指名 thread 建立者並告訴他「開一則新訊息就能建立自己的行程」。**絕不可**收到「資訊不足，請補充」（缺陷 33 —— 那會讓他反覆補資料卻永遠通不過） |
| 35 | 同一位同事在你的 thread 裡單純閒聊（例如 `收到` `好喔`） | 必須**安靜略過**（只有 ➖ reaction），不可跳出權限卡片。權限卡片只在他真的想建立或修改行程時才出現 |
| 36 | 送 `明天早上 10 點開會`（**只講開始時間，不講結束**） | 必須成功建立且時長為 1 小時。卡片時間顯示應為 `3月2日（一） 10:00–11:00` 形式 —— 若顯示 ISO 字串或走進澄清卡，代表系統補的結束時間帶了 `Z` 而被自己的時區守衛擋下（缺陷 32／36 的交互作用，實際發生過） |
| 37 | 送一則會讓 AI 回 UTC 時間的訊息（較難穩定重現；可改為在 n8n 中手動編輯 `Parse AI Response` 的輸入來模擬 `...T09:00:00Z`） | 必須走澄清卡，且文案要提到**時區**（`TIMEZONE_UTC`），不可只說「格式不正確」。**絕不可**直接建立事件 —— 那會是一個差 8 小時、外觀完全正常的錯誤事件（缺陷 32） |
| 38 | 在同一 thread 依序建立事件 A（`8/20 09:00 面試`）與事件 B（`8/21 14:00 簡報`），然後**再送一次 A**（`8/20 09:00 面試`） | 必須判為**重複**。若成功建立第二筆 A，代表重複判定只比對了最後一筆（缺陷 36） |
| 39 | 建立一個**單日**全天事件（`8/20 請假一天`） | 卡片時間必須顯示為 `8月20日（三） 整天`，**不可**顯示成 `8月20日 ～ 8月21日` —— 後者代表沒有把 Google 要求的 exclusive 結束日還原（缺陷 37） |
| 40 | 送一則資訊不足的訊息（`下週開個會`），收到澄清卡後回覆 `就這樣建立` | 第二張卡片必須先承接你這句話（`✋ 我收到你說的「就這樣建立」了`）再說明還差什麼，**不可**原封不動再問一次同樣的問題（缺陷 38）。同時確認卡片**不再顯示信心度百分比** |
| 41 | 承步驟 40 的情境，在送出前先把 bot 移出頻道（讓**澄清卡**投遞失敗），送出後再加回來 | n8n 執行紀錄中 `Verify Notice Delivery` 必須**攔下**這次投遞並導向錯誤卡片。若該次執行顯示成功，代表澄清卡的失敗是靜默的 —— 而澄清卡是使用者唯一的脫困路徑，送不出去等同對話卡死（缺陷 34） |
| 42 | 在 n8n 編輯器中開啟 `Create Calendar Event` 節點的 Settings 分頁 | **Retry On Fail 必須是關閉的**。Google Calendar 的 insert 非冪等，逾時重試會建立第二筆事件，而且只有第二筆的 id 會寫進 metadata（缺陷 35）。對照確認 `Update Calendar Event` 的 Retry On Fail 是**開啟**的 —— 對同一 event id 重送相同內容是冪等的 |
| 43 | 上傳一個超過 Slack 檔案大小上限的圖片 | 卡片頁尾必須說出**具體原因**（太大），不可只說「有附件未被讀取」（缺陷 39） |
| 44 | 上傳一個非圖片檔（如 PDF）**且訊息本身沒有任何文字** | 必須收到**澄清卡**說明附件無法讀取；**絕不可**只回一個 ➖ reaction（缺陷 40 —— 他傳了東西，得到的是沉默）|
| 45 | 在同一 thread 建立事件後、更新它、再送一次同樣的內容 | 應判為重複。若判成「不重複」而建立第二筆，代表 `allEvents` 裡同一個事件出現了兩筆（缺陷 41）|
| 46 | 在一個**超過 15 則回覆**的長 thread 裡，先讓系統回澄清卡，再回覆 `就這樣建立` | **必須能建立**。若沒有反應，代表系統讀的是 thread 歷史最後一則而不是你剛送的這則（缺陷 42 —— Slack 回的是最舊的 N 筆，你的訊息根本不在裡面）|
| 47 | 送一則模型可能會判斷矛盾的訊息（可在 n8n 中手動編輯 `Parse AI Response` 的輸入模擬 `hasEvent:false` 但 `events[0]` 完整），收到澄清卡後回 `就這樣建立` | **必須能建立**（缺陷 43）。反向確認：把 `events` 改成 `[]` 之後同樣回覆 `就這樣建立`，**必須仍然擋下** —— force 不得降低資料完整性門檻 |
| 48 | 送 `UTC 15:00 的線上會議` | **必須建立成功**，且時間等同台北 23:00。若走進澄清卡並要你「說明這是哪個時區」，代表明示 UTC 沒有被辨識（缺陷 44）。反向確認：不提 UTC 的一般訊息若讓模型回 `Z`，仍必須被擋（缺陷 32 不可被這個修正弄鬆）|
| 49 | 讓 Google Calendar 呼叫逾時（例如暫時把憑證改成無效值造成長時間等待） | 錯誤卡必須說「**請先到 Google 日曆確認是否已經建立**」，**絕不可**叫你重試或手動建立（缺陷 45 —— 逾時的請求很可能已經成功寫入）|
| 50 | 在既有 thread 回覆 `改成五點`，更新成功後看卡片 | 更新卡必須有 **Google 日曆連結**可以點進去（缺陷 46）|
| 51 | 送一則資訊不足的訊息讓它走澄清卡，仔細看卡片上的時間與空欄位 | 時間必須是 `3月1日（一） 09:00–10:00` 這種可讀格式，**不可**是 ISO 字串；缺少的欄位必須是可以行動的指引（`還沒有標題 — 告訴我這是什麼活動`），不可只是「（尚未取得標題）」（缺陷 47）|
| 52 | 承步驟 24 把 `$vars.GCAL_ID` 設成非法值後，送一則**正常**訊息 | 除了 Console 的 `[GCAL_ID]` 警告，**成功卡本身的頁尾也必須出現警告**告訴你事件進了預設日曆（缺陷 48 —— 使用者不會去翻 n8n 執行紀錄）|
| 53 | 在同一 thread 依序建立 A（`8/20 09:00 週會`）與 B（`8/21 14:00 面試`），然後回覆 `把週會改到 8/22 09:00` | **必須改到 A**。若被改的是 B，代表 update 目標仍是「最後建立的那一筆」（缺陷 50）。再測一次改標題的情形（`改名叫季度檢討`）：這時系統挑不到目標，**仍必須執行更新**，但卡片要出現「這個執行緒裡有 2 個事件，我更新的是「…」」的說明 |
| 54 | 讓模型回傳一段不是合法 JSON 的內容（可在 n8n 中手動編輯 `Parse AI Response` 的輸入，塞入一長串非 JSON 文字），然後開啟該次執行的 Console 分頁 | `[E_BAD_JSON]` 那一行**最多只能有 120 字**的原文。若印出數百字，代表使用者貼在 thread 裡的內容會被完整寫進 n8n 執行紀錄 —— 而執行紀錄的保存期限與存取範圍都和 Slack 頻道不同（缺陷 49）|

第 4、6、11、12、14、**17**、**19**、**23**、**27**、**31**、**34**、**36** 是核心迴歸點，務必確認。
其中 **#17 是最容易復發的** —— 它是最常見的輸入形式，卻曾經完全靜默失敗。
**#23 是後果最嚴重的** —— 錯誤訊息本身把使用者導向製造重複事件。
**#27 是最容易在修 bug 時被犧牲掉的** —— 加防護很容易把使用者的脫困路徑一併堵死。
**#31 是唯一需要三個步驟才驗得出來的** —— 前兩步都會成功，錯誤只在第三步顯現。
**#36 是這次真的踩到的那種** —— 為缺陷 32 加的時區守衛，擋下了系統自己補出來的結束時間，
讓「明天早上 10 點開會」這種最常見的輸入全部走進澄清卡。防護打到自己人，而且只在整合測試才看得出來。

## 已知未解風險

**1. 並行競態**：兩則訊息在極短時間內送達同一 thread 時，兩個執行都會在對方寫入 metadata 前讀取 thread，可能重複建立事件。
緩解方式為在 n8n 設定依 `thread_ts` 的執行序列化，或建立後回查 thread 去重。此項尚未實作，也尚未實測。

**2. `channelId` 缺 `cachedResultName`**：`Slack Message Trigger` 的 channel 是 `__rl` 資源定位器，`cachedResultName` 為空。
已與生產版逐欄比對，**兩者完全相同**（`{"__rl":true,"mode":"id","value":"C08NVUQUK8F"}`）——這不是 v2 引入的問題。
n8n MCP 也明確指出「workflow will run」，只是 UI 下拉會顯示 `Choose...` 而非頻道名。
若在意顯示效果，在 n8n UI 中重新選一次頻道即可自動填入。

**3. 錯誤卡片本身也可能發不出去**：若 `Verify Card Delivery` 失敗的原因是 `not_in_channel` 或 token 失效，那麼接續的錯誤卡片同樣會失敗。
這種情況下使用者不會收到任何訊息，只能靠 n8n 的執行紀錄察覺 —— 因此 `settings.saveDataErrorExecution` 已設為 `'all'`。

**4. 圖片上限 4 張**：超過的部分會記入 `rejectedFiles`，成功卡片與澄清卡片的頁尾會顯示
「⚠ 有 N 個附件未被讀取（超過 4 張上限或非圖片格式）」。
若你的使用情境常一次貼很多張圖，可調整 `Collect Thread Context` 的 `MAX_THREAD_IMAGES`
（注意這會等比增加 vision token 成本與 Slack 下載次數）。

**5. `typeVersion` 刻意不升到 MCP 建議的最新版**：MCP 建議把 httpRequest 4.2→4.4、switch 3.2→3.4。
**已查證 n8n 2.32.5 確實支援 4.4／3.4**（`HttpRequest.node.ts` L18 `defaultVersion:4.4`、L26-33 版本清單；
`Switch.node.ts` L17 `defaultVersion:3.4`、L20-27 版本清單），但**仍決定維持 4.2／3.2**，理由比「保守」更具體：

| 選項 | 優點 | 缺點 |
|---|---|---|
| 維持 4.2 / 3.2（採用） | 4.x 全部對映同一個 `HttpRequestV3`、3.x 全部對映同一個 `SwitchV3`，**執行碼完全相同**，不缺任何功能；且 4.2 對 `Download Image` **更穩妥**（見下） | 版本號不是最新 |
| 升到 4.4 / 3.4 | 跟上最新版本號 | **會讓 Slack 圖片下載可能 401** |

關鍵在 `HttpRequestV3.node.ts` L340：`defaultSendCredentialsOnCrossOriginRedirect = nodeVersion < 4.4`。
也就是 **4.2 預設「跨來源重導時仍帶憑證」，4.4 預設「不帶」**。
`Download Image` 抓的是 Slack 的 `url_private`，Slack 通常會 302 重導到 `files.slack.com`
（`slack.com` ≠ `files.slack.com`，屬跨來源），這個重導**必須帶授權標頭**才能取得檔案。
4.2 的預設值正好讓它成功；升到 4.4 就得在 `Download Image` 的 `options` 裡手動打開該選項，
否則圖片下載會失敗 —— 而圖片理解正是 v2 的核心新功能。

SwitchV3 的 `nodeVersion` 只影響輸出版本標籤（L234），3.2 與 3.4 **無行為差異**。

> 若日後要升到 4.4，**務必**同時在 `Download Image` 開啟「跨來源重導時傳送憑證」，否則圖片功能會壞。

**6. 授權邊界就是「頻道成員」，沒有更細的控制**：這是必須明白說出來的設計事實，而不是疏漏。

- `isOwner` **只保護既有事件**：它決定「誰能修改這個 thread 已建立的行程」，也只決定這一件事。
- **任何能在該頻道發言的人，都能開一則新訊息建立事件**，寫進 `$vars.GCAL_ID` 指定的那本日曆。
- 也就是說：**授權邊界等同於 Slack 頻道的成員名單**。誰被邀進頻道，誰就能寫你的日曆。

v1 也是如此，不是 v2 引入的回歸；但 v1 不處理 thread、也沒有圖片理解，使用門檻較高，實際被誤用的機會較小。
v2 把體驗做順了，這條邊界就更值得先講清楚再上線。

若這個邊界不符合你的需求，有三種收斂方式（由簡到繁）：

| 做法 | 優點 | 缺點 |
|---|---|---|
| **把頻道設為私人頻道，只邀請該邀的人**（建議先用這個） | 零改動、立即生效、語義最直觀（「進得來就寫得了」） | 需要管理頻道成員；訪客／外部連線成員仍算成員 |
| 在 `Collect Thread Context` 加使用者 allowlist | 精確到人；不影響頻道其他用途 | 名單維護成本；新人加入要記得改 workflow |
| 用 Slack User Group 或 Google Calendar ACL 管理 | 與既有權限系統一致 | 需要額外 API 呼叫與憑證範圍，複雜度最高 |

實作 allowlist 的話：在 `Collect Thread Context` 讀 `$vars.ALLOWED_USERS`（逗號分隔的 Slack user id），
在 `Should Process` 之前擋掉不在名單內的人，並回一張「你沒有使用權限」的卡片 ——
**務必給明確訊息，不要靜默丟棄**，否則就是把缺陷 33 那個無解迴圈重新做一遍。

## 回滾

1. 停用 v2 workflow
2. 重新啟用原 `Slack_to_Google_Calendar_AI_Assistant.json`
3. **注意**：舊版不處理 thread 回覆，因此在既有 thread 中等待回應的使用者不會收到任何回覆，需另行公告「請開新訊息」
4. 由於兩者 `webhookId` 相同，切換時務必**先停用一邊、再啟用另一邊**，不要同時 active
