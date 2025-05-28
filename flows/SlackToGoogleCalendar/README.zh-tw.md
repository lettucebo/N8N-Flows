# n8n Slack 日曆助手 - 完整實施指南

## 📋 系統概述

這是一個基於 n8n 的自動化工作流，能夠：
- 監聽 Slack 頻道的訊息
- 使用 Azure OpenAI 智能分析訊息中的日程信息
- 自動創建 Google Calendar 事件
- 根據信心度決定自動創建或需要確認
- 支援全天活動和定時活動
- 提供詳細的 Slack 通知反饋

## 🏗️ 系統架構

```
Slack 訊息 → Webhook → Filter Channel Messages → Basic LLM Chain (Azure OpenAI) 
    ↓
Process AI Response → Confidence Filter → Event Type Filter
    ↓                    ↓                    ↓
Error Notification  High Confidence    Low Confidence
                         ↓                    ↓
                   Create Calendar    Confirmation Request
                         ↓
                   Success Notification
```

## 🔧 節點配置詳情

### 1. Webhook 節點
**名稱**: `Webhook`
**類型**: Webhook Trigger

**配置**:
```yaml
HTTP Method: POST
Path: slack-calendar
Response Mode: Respond With
Response Data: ={{$json.challenge || 'OK'}}
```

**用途**: 接收 Slack Event Subscriptions 的 webhook 請求

---

### 2. Filter Channel Messages 節點
**名稱**: `Filter Channel Messages`
**類型**: IF

**關鍵配置**:
```javascript
// 條件 1: 檢查頻道
Value 1: ={{$json.channel}}
Operation: Equal
Value 2: C08NVUQUK8F  // 替換為你的實際頻道 ID

// 條件 2: 檢查訊息類型
Value 1: ={{$json.type}}
Operation: Equal
Value 2: message

// 條件 3: 排除 Bot 訊息（防止無限循環）
Value 1: ={{$json.user}}
Operation: Not Equal
Value 2: YOUR_BOT_USER_ID  // 替換為你的 Bot User ID

// 條件 4: 排除系統訊息
Value 1: ={{$json.subtype}}
Operation: Is Empty

// 條件 5: 排除回覆訊息
Value 1: ={{$json.thread_ts}}
Operation: Is Empty

// 邏輯
Combine: AND
```

**用途**: 過濾只處理目標頻道的用戶訊息，避免無限循環

---

### 3. Basic LLM Chain 節點
**名稱**: `AI Message Analyzer`
**類型**: Basic LLM Chain

**Language Model 設定**:
- Model: Azure OpenAI Chat Model
- Credential: Azure OpenAI API
- Deployment Name: 你的 Azure 部署名稱（如 gpt-4）

**完整 Prompt**:
```
你是一個專業的日程管理助手。請分析用戶的 Slack 訊息，提取可能的日程信息。

🔍 地點偵測規則（重要）：
1. 移動動詞後的地點：
   - 「去XX」、「到XX」、「往XX」、「赴XX」
   - 「前往XX」、「出發到XX」、「飛往XX」
   - 範例：「去名古屋玩」→ location: "名古屋"

2. 位置介詞後的地點：
   - 「在XX」、「於XX」、「位於XX」
   - 「XX舉行」、「XX進行」、「XX召開」
   - 範例：「在台北開會」→ location: "台北"

3. 常見地點類型：
   - 城市/國家：台北、東京、名古屋、新加坡、美國、日本
   - 地標/景點：101大樓、迪士尼、故宮、富士山
   - 場所：會議室A、咖啡廳、餐廳、辦公室、家裡
   - 線上：Zoom、Teams、Google Meet、視訊、線上
   - 建築：XX大樓、XX中心、XX館、XX廳

4. 複合地點表達：
   - 「台北101」→ location: "台北101"
   - 「新竹科學園區」→ location: "新竹科學園區"
   - 「會議室A」→ location: "會議室A"
   - 「線上會議」→ location: "線上"

5. 地點提取優先順序：
   - 具體地址 > 建築名稱 > 城市名稱 > 區域名稱
   - 如果有多個地點，選擇最具體的那個

6. 特殊情況處理：
   - 「XX玩」、「XX旅遊」、「XX出差」→ XX 是地點
   - 「回XX」、「返回XX」→ XX 是地點
   - 「XX見面」、「XX聚餐」→ XX 可能是地點

全天活動識別規則：
1. 旅遊活動：「去XX玩」、「XX旅遊」、「到XX」、「XX出差」
2. 假期/休假：「請假」、「休假」、「放假」
3. 節日/慶典：「生日」、「節日」、「活動日」
4. 研習/課程：「研習營」、「工作坊」、「訓練營」
5. 只提到日期沒有具體時間的活動

部分時間活動識別規則：
1. 會議：「開會」、「會議」、「討論」
2. 簡報：「簡報」、「presentation」、「報告」
3. 有明確時間的活動：「下午2點」、「早上」、「晚上」

時間格式規則：
- 全天活動：startDateTime 設為 "YYYY-MM-DD"（不包含時間部分）
- 部分時間活動：使用完整的 ISO 8601 格式 "YYYY-MM-DDTHH:mm:ss+08:00"

分析規則：
1. 時間關鍵字：明天、下週、今天、具體日期、具體時間
2. 活動關鍵字：會議、討論、簡報、培訓、研習、活動、玩、旅遊  
3. 地點信息：會議室、地址、線上、視訊、城市名稱、建築名稱
4. 參與者：@用戶名、職稱、部門
5. 時間推斷：
   - 「明天下午」→ 明天 14:00-15:00（部分時間）
   - 「下週三開會」→ 下週三 09:00-10:00（部分時間）
   - 「2點開會」→ 14:00-15:00（部分時間）
   - 「去XX玩」→ 全天活動（純日期格式）
   - 「XX研習營」→ 全天活動（純日期格式）
   - 「請假」→ 全天活動（純日期格式）

請按照以下 JSON 格式回覆，不要添加任何其他文字：
{
  "hasEvent": true,
  "events": [
    {
      "title": "事件標題",
      "description": "詳細描述",
      "startDateTime": "2025-05-20",
      "endDateTime": "2025-05-20",
      "isAllDay": true,
      "location": "具體地點名稱",
      "attendees": ["email1@example.com"],
      "confidence": 0.95
    }
  ],
  "reasoning": "分析原因，包含地點識別邏輯"
}

地點偵測範例：
- 「5/20 去名古屋玩」→ location: "名古屋"（從「去XX玩」提取）
- 「明天在會議室A開會」→ location: "會議室A"（從「在XX」提取）
- 「下週到台北出差」→ location: "台北"（從「到XX出差」提取）
- 「6/15 東京迪士尼一日遊」→ location: "東京迪士尼"（複合地點）
- 「線上會議討論專案」→ location: "線上"（虛擬地點）
- 「在家工作」→ location: "家裡"（居家地點）
- 「新竹科學園區參訪」→ location: "新竹科學園區"（園區地點）

活動類型與地點範例：
- 「5/20 去名古屋玩」→ 全天活動，startDateTime: "2025-05-20", endDateTime: "2025-05-20", isAllDay: true, location: "名古屋"
- 「明天下午2點在會議室B簡報」→ 部分時間，startDateTime: "2025-05-21T14:00:00+08:00", endDateTime: "2025-05-21T15:00:00+08:00", isAllDay: false, location: "會議室B"
- 「6/15 台北101研習營」→ 全天活動，startDateTime: "2025-06-15", endDateTime: "2025-06-15", isAllDay: true, location: "台北101"
- 「下週三上午10點線上會議」→ 部分時間，startDateTime: "2025-06-04T10:00:00+08:00", endDateTime: "2025-06-04T11:00:00+08:00", isAllDay: false, location: "線上"

信心度評估：
- 0.9+: 完整時間+地點+活動類型明確
- 0.7-0.9: 明確時間和活動，地點可推斷
- 0.5-0.7: 模糊時間或活動類型，地點不明確
- <0.5: 缺乏關鍵信息

預設設定：
- 時區：Asia/Taipei (+08:00)
- 部分時間活動預設時長：1小時
- 無具體時間的會議：09:00-10:00
- 旅遊/休假/研習等：全天活動
- 地點提取：優先提取最具體的地點信息

現在請分析以下 Slack 訊息：「={{$json.text}}」

⚠️ 重要：請特別注意地點信息的提取，確保不遺漏任何地點相關的詞語。
```

**用途**: 使用 Azure OpenAI 分析 Slack 訊息，提取日程信息

---

### 4. Process AI Response 節點
**名稱**: `Process AI Response`
**類型**: Function

**完整 JavaScript 代碼**:
```javascript
// ===============================================
// Process AI Response - 完整版 JavaScript 代碼
// ===============================================

console.log('🚀 開始處理 AI 回應...');

// 1. 檢查基本輸入
const inputData = $input.first();
if (!inputData || !inputData.json) {
  console.log('❌ 沒有輸入數據，停止執行');
  return [];
}

console.log('📥 輸入數據:', JSON.stringify(inputData.json, null, 2));

// 2. 取得 AI 輸出（從 text 欄位）
const aiResponseText = inputData.json.text || inputData.json.output;
if (!aiResponseText) {
  console.log('❌ AI 沒有輸出文字，停止執行');
  return [];
}

console.log('🤖 AI 原始回應:', aiResponseText);

// 3. 解析 JSON 字串
let aiOutput;
try {
  // 清理可能的 markdown 格式
  const cleanResponse = aiResponseText.replace(/```json\n?|```\n?/g, '').trim();
  aiOutput = JSON.parse(cleanResponse);
  console.log('✅ 解析後的 AI 輸出:', JSON.stringify(aiOutput, null, 2));
} catch (error) {
  console.log('❌ JSON 解析失敗:', error.message);
  return [{
    json: {
      status: 'error',
      error: `AI 回應解析失敗: ${error.message}`,
      rawResponse: aiResponseText,
      slackUser: 'unknown',
      slackChannel: 'unknown',
      slackTimestamp: Date.now().toString()
    }
  }];
}

// 4. 檢查 Slack 原始數據
let slackData = {};
try {
  // 嘗試多種方式取得 Slack 數據
  if ($node && $node[0] && $node[0].json) {
    slackData = $node[0].json;
  } else if ($('Webhook') && $('Webhook').first()) {
    slackData = $('Webhook').first().json;
  } else if ($('Slack Webhook') && $('Slack Webhook').first()) {
    slackData = $('Slack Webhook').first().json;
  } else {
    // 如果無法取得原始數據，使用預設值
    slackData = {
      text: '無法取得原始訊息',
      user: 'unknown',
      channel: 'unknown',
      ts: Date.now().toString()
    };
  }
  console.log('📱 Slack 數據:', JSON.stringify(slackData, null, 2));
} catch (error) {
  console.log('⚠️ 取得 Slack 數據時發生錯誤:', error.message);
  slackData = {
    text: '無法取得原始訊息',
    user: 'unknown', 
    channel: 'unknown',
    ts: Date.now().toString()
  };
}

// 5. 檢查 AI 是否找到事件
if (!aiOutput.hasEvent || !aiOutput.events || aiOutput.events.length === 0) {
  console.log('ℹ️ AI 沒有找到事件');
  return [{
    json: {
      status: 'no_event',
      message: '未找到有效的日程信息',
      originalMessage: slackData.text,
      slackUser: slackData.user,
      slackChannel: slackData.channel,
      slackTimestamp: slackData.ts,
      reasoning: aiOutput.reasoning || '無法識別日程相關內容'
    }
  }];
}

// 6. 處理找到的事件（支援全天活動）
console.log(`📅 找到 ${aiOutput.events.length} 個事件，開始處理`);

try {
  const processedEvents = aiOutput.events.map((event, index) => {
    console.log(`🔄 處理事件 ${index}:`, JSON.stringify(event, null, 2));
    
    // 驗證事件是否有必要欄位
    if (!event.title && !event.startDateTime) {
      console.log(`⚠️ 事件 ${index} 缺少必要欄位，跳過`);
      return null;
    }

    // 🎯 處理全天活動與部分時間活動
    let startDateTime, endDateTime, isAllDay = false;
    let calendarStartDateTime, calendarEndDateTime;
    
    try {
      // 檢查是否為全天活動
      const isAllDayEvent = event.isAllDay === true || 
                           (!event.startDateTime.includes('T') && !event.endDateTime.includes('T'));
      
      if (isAllDayEvent) {
        // 📅 全天活動處理
        console.log(`📅 事件 ${index} 是全天活動`);
        isAllDay = true;
        
        // 確保日期格式正確 (YYYY-MM-DD)
        startDateTime = event.startDateTime.includes('T') 
          ? event.startDateTime.split('T')[0] 
          : event.startDateTime;
        endDateTime = event.endDateTime.includes('T') 
          ? event.endDateTime.split('T')[0] 
          : event.endDateTime;
          
        // Google Calendar 全天活動格式：純日期字串
        calendarStartDateTime = startDateTime;
        calendarEndDateTime = endDateTime;
        
        // 驗證日期格式
        if (!startDateTime.match(/^\d{4}-\d{2}-\d{2}$/)) {
          throw new Error('全天活動日期格式錯誤');
        }
        
        console.log(`✅ 全天活動時間: ${startDateTime} 到 ${endDateTime}`);
        
      } else {
        // ⏰ 部分時間活動處理
        console.log(`⏰ 事件 ${index} 是部分時間活動`);
        isAllDay = false;
        
        const startDate = new Date(event.startDateTime);
        const endDate = new Date(event.endDateTime);
        
        if (isNaN(startDate.getTime()) || isNaN(endDate.getTime())) {
          throw new Error('無效的日期時間格式');
        }
        
        // 確保結束時間在開始時間之後
        if (endDate <= startDate) {
          endDate.setTime(startDate.getTime() + 60 * 60 * 1000);
        }
        
        startDateTime = startDate.toISOString();
        endDateTime = endDate.toISOString();
        
        // Google Calendar 部分時間活動格式：完整 ISO 字串
        calendarStartDateTime = startDateTime;
        calendarEndDateTime = endDateTime;
        
        console.log(`✅ 部分時間活動: ${startDateTime} 到 ${endDateTime}`);
      }
      
    } catch (dateError) {
      console.log(`⚠️ 事件 ${index} 時間處理失敗，使用預設:`, dateError.message);
      
      // 預設為明天的全天活動
      const tomorrow = new Date();
      tomorrow.setDate(tomorrow.getDate() + 1);
      const dateStr = tomorrow.toISOString().split('T')[0];
      
      startDateTime = dateStr;
      endDateTime = dateStr;
      calendarStartDateTime = dateStr;
      calendarEndDateTime = dateStr;
      isAllDay = true;
    }
    
    // 處理參與者列表
    const attendees = (event.attendees || []).filter(email => 
      email && email.includes('@') && email.includes('.')
    );
    
    // 確保 confidence 是浮點數
    const confidence = parseFloat(event.confidence) || 0.5;
    console.log(`📊 事件 ${index} 信心度: ${confidence} (${Math.round(confidence * 100)}%)`);
    
    // 構建詳細描述
    const description = [
      event.description || '',
      '',
      '📋 來源信息：',
      `• Slack 訊息: ${slackData.text}`,
      `• 發送者: <@${slackData.user}>`,
      `• 頻道: <#${slackData.channel}>`,
      `• 活動類型: ${isAllDay ? '📅 全天活動' : '⏰ 部分時間活動'}`,
      `• AI 分析信心度: ${Math.round(confidence * 100)}%`,
      aiOutput.reasoning ? `• 分析說明: ${aiOutput.reasoning}` : ''
    ].filter(line => line !== '').join('\n');
    
    // 🎯 根據信心度決定處理策略
    let eventStatus;
    if (confidence >= 0.7) {
      eventStatus = 'high_confidence';
      console.log(`✅ 事件 ${index} 高信心度，自動處理`);
    } else {
      eventStatus = 'low_confidence';
      console.log(`⚠️ 事件 ${index} 低信心度，需要確認`);
    }
    
    const processedEvent = {
      json: {
        eventIndex: index,
        title: event.title || '未命名事件',
        description: description,
        
        // 📅 時間信息
        startDateTime: calendarStartDateTime,
        endDateTime: calendarEndDateTime,
        isAllDay: isAllDay,
        
        // 📍 地點和參與者
        location: event.location || '',
        attendees: attendees,
        
        // 📊 分析信息
        confidence: confidence,
        reasoning: aiOutput.reasoning || '',
        
        // 📱 原始信息
        originalMessage: slackData.text,
        slackUser: slackData.user,
        slackChannel: slackData.channel,
        slackTimestamp: slackData.ts,
        
        // 🔄 處理狀態
        status: eventStatus
      }
    };
    
    console.log(`✅ 事件 ${index} 處理完成:`, JSON.stringify(processedEvent.json, null, 2));
    return processedEvent;
  }).filter(event => event !== null);

  // 7. 檢查是否有有效的處理結果
  if (processedEvents.length === 0) {
    console.log('❌ 沒有有效的事件可處理');
    return [{
      json: {
        status: 'error',
        error: '所有事件都處理失敗',
        originalMessage: slackData.text,
        slackUser: slackData.user,
        slackChannel: slackData.channel,
        slackTimestamp: slackData.ts
      }
    }];
  }

  console.log(`🎉 成功處理 ${processedEvents.length} 個事件`);
  console.log('📤 最終輸出:', JSON.stringify(processedEvents, null, 2));
  
  return processedEvents;
  
} catch (error) {
  console.log('❌ 處理事件時發生錯誤:', error.message);
  console.log('🔍 錯誤堆疊:', error.stack);
  
  return [{
    json: {
      status: 'error',
      error: `處理失敗: ${error.message}`,
      originalMessage: slackData.text,
      slackUser: slackData.user,
      slackChannel: slackData.channel,
      slackTimestamp: slackData.ts,
      debugInfo: {
        errorStack: error.stack,
        inputData: inputData.json,
        aiOutput: aiOutput
      }
    }
  }];
}
```

**用途**: 解析 AI 輸出，處理時間格式，根據信心度分類事件

---

### 5. Confidence Filter 節點
**名稱**: `Confidence Filter`
**類型**: IF

**配置**:
```javascript
// 使用單一條件（推薦）
Value 1: ={{$json.status === 'high_confidence'}}
Operation: Equal
Value 2: true

// 或者使用分離條件
// 條件 1: 狀態檢查
Value 1: ={{$json.status}}
Operation: Equal
Value 2: high_confidence

// 條件 2: 信心度檢查
Value 1: ={{parseFloat($json.confidence)}}
Operation: Greater than or equal
Value 2: 0.7

// 邏輯: AND
```

**用途**: 根據信心度（≥70%）決定自動創建還是需要確認

---

### 6. Create Calendar Event 節點
**名稱**: `Create Calendar Event`
**類型**: Google Calendar

**配置**:
```javascript
Calendar ID: primary
Resource: Event
Operation: Create

// 基本欄位
Summary: ={{$json.title}}
Start: ={{$json.startDateTime}}
End: ={{$json.endDateTime}}
Description: ={{$json.description}}
Location: ={{$json.location}}

// 🎯 關鍵設定：All Day Event
// 方案一：使用固定值（最穩定）
All Day Event: Yes  // 手動選擇（如果都是全天活動）

// 方案二：使用表達式（需要動態處理）
All Day Event: ={{Boolean($json.isAllDay)}}

// 其他設定
Send Notifications: Yes
Time Zone: Asia/Taipei
```

**已知問題**: All Day Event 欄位的表達式可能不穩定，建議使用分支處理

**分支解決方案**:
如果 All Day Event 表達式有問題，創建兩個專門節點：
1. `Create All Day Event` - All Day: Yes (固定)
2. `Create Timed Event` - All Day: No (固定)
在前面用 Event Type Filter 分流

**用途**: 創建 Google Calendar 事件

---

### 7. Success Notification 節點
**名稱**: `Success Notification`
**類型**: Slack

**配置**:
```javascript
Credential: Slack API
Resource: Message
Operation: Post

// 頻道設定
Select a Channel: By ID
Channel ID: ={{$json.slackChannel || 'C08NVUQUK8F'}}

// 訊息內容
Text: 
✅ **日曆事件已成功創建！**

📅 **{{$json.summary}}**
{{$json.start.date ? 
  '🗓️ ' + $json.start.date + ' (全天活動)' : 
  '🕐 時間: ' + $json.start.dateTime + ' - ' + $json.end.dateTime
}}
📍 地點: {{$json.location || '未指定'}}

🔗 [查看事件]({{$json.htmlLink}})

_🤖 由 AI 助手自動創建_

// 回覆串設定
Thread TS: ={{$json.slackTimestamp}}
```

**用途**: 發送成功創建事件的 Slack 通知

---

### 8. Low Confidence Notification 節點
**名稱**: `Low Confidence Notification`
**類型**: Slack

**配置**:
```javascript
Channel ID: ={{$json.slackChannel}}

Text:
⚠️ **需要確認的日程信息**

我在你的訊息中找到了可能的日程，但需要確認：

📝 **{{$json.title}}**
🕐 時間: {{DateTime.fromISO($json.startDateTime).toFormat('MM/dd HH:mm')}} - {{DateTime.fromISO($json.endDateTime).toFormat('HH:mm')}}
📍 地點: {{$json.location || '未指定'}}
📊 信心度: {{Math.round($json.confidence * 100)}}%

💭 AI 分析: _{{$json.reasoning}}_

請回覆以下選項：
✅ 確認創建
❌ 取消
✏️ 提供更多詳細信息

Thread TS: ={{$json.slackTimestamp}}
```

**用途**: 低信心度事件需要用戶確認

---

### 9. Error Notification 節點
**名稱**: `Error Notification`
**類型**: Slack

**配置**:
```javascript
Channel ID: ={{$json.slackChannel}}

Text:
❌ **處理失敗**

{{$json.status === 'no_event' ? '在你的訊息中沒有找到明確的日程信息。' : '處理你的訊息時遇到錯誤：'}}

{{$json.status === 'error' ? '🐛 錯誤詳情: ' + $json.error : ''}}
{{$json.reasoning ? '🤔 AI 分析: ' + $json.reasoning : ''}}

💡 **建議格式範例：**
• `明天下午2點團隊會議`
• `2025-06-01 14:00 客戶簡報`
• `下週三上午10點在會議室A討論專案`
• `6/15 全天研習營`

原始訊息: _{{$json.originalMessage}}_

Thread TS: ={{$json.slackTimestamp}}
```

**用途**: 處理錯誤和無事件情況的通知

## 🔐 憑證設定

### 1. Slack API 憑證
**類型**: Slack API
**設定**:
```yaml
Access Token: xoxb-your-bot-token-here
```

**取得方式**:
1. 前往 https://api.slack.com/apps
2. 創建新 App 或選擇現有 App
3. OAuth & Permissions → Bot User OAuth Token

### 2. Azure OpenAI 憑證
**類型**: Azure OpenAI
**設定**:
```yaml
API Key: your-azure-openai-api-key
Resource Name: your-resource-name
API Version: 2024-02-15-preview
```

### 3. Google Calendar 憑證
**類型**: Google Calendar OAuth2 API
**設定**: 按照 n8n 指示完成 OAuth2 授權流程

## 🔧 Slack App 設定

### Event Subscriptions 設定
```yaml
Request URL: https://your-n8n-domain.com/webhook-test/slack-calendar
Subscribe to bot events:
  - message.channels
  - app_mention
```

### Bot Token Scopes
```yaml
必要權限:
  - channels:read
  - channels:history
  - chat:write
  - users:read
  - app_mentions:read

可選權限:
  - chat:write.public
  - reactions:read
```

## 🚨 已知問題和解決方案

### 1. 無限循環問題
**問題**: Bot 發送的通知會觸發新的 workflow
**解決**: 在 Filter Channel Messages 中添加 Bot 過濾
```javascript
Value 1: ={{$json.user}}
Operation: Not Equal
Value 2: YOUR_BOT_USER_ID
```

### 2. All Day Event 表達式問題
**問題**: `={{$json.isAllDay ? 'Yes' : 'No'}}` 不被接受
**解決**: 使用分支處理或固定值
```javascript
// 方案 1: 分支處理
Event Type Filter → true → Create All Day Event (All Day: Yes)
                  → false → Create Timed Event (All Day: No)

// 方案 2: 嘗試不同表達式
={{Boolean($json.isAllDay)}}
={{$json.isAllDay === true}}
```

### 3. Confidence 判斷問題
**問題**: 浮點數比較失敗
**解決**: 使用 `parseFloat()` 確保數值類型
```javascript
={{parseFloat($json.confidence) >= 0.7}}
```

### 4. 全天活動時間格式問題
**問題**: Google Calendar 顯示 08:00-08:00 而不是全天
**確認**: 檢查 Google Calendar 輸出格式
```json
// ✅ 正確的全天活動格式
"start": {"date": "2025-05-20"}
"end": {"date": "2025-05-20"}

// ❌ 錯誤的格式
"start": {"dateTime": "2025-05-20T08:00:00+08:00"}
```

## 🧪 測試案例

### 全天活動測試
```
輸入: "5/20 去名古屋玩"
預期輸出:
- title: "去名古屋玩"
- startDateTime: "2025-05-20"
- endDateTime: "2025-05-20" 
- isAllDay: true
- location: "名古屋"
- confidence: 0.9+
```

### 定時活動測試
```
輸入: "明天下午2點團隊會議"
預期輸出:
- title: "團隊會議"
- startDateTime: "2025-05-XX T14:00:00+08:00"
- endDateTime: "2025-05-XX T15:00:00+08:00"
- isAllDay: false
- confidence: 0.8+
```

### 低信心度測試
```
輸入: "可能要開會"
預期結果: 觸發 Low Confidence Notification
```

### 無事件測試
```
輸入: "今天天氣很好"
預期結果: 觸發 Error Notification (no_event)
```

## 🔍 故障排除

### 調試步驟
1. **檢查 Webhook 觸發**: 確認 Slack 事件有到達 n8n
2. **檢查過濾條件**: 確認 Filter Channel Messages 邏輯正確
3. **檢查 AI 輸出**: 查看 Basic LLM Chain 的原始回應
4. **檢查數據處理**: 查看 Process AI Response 的 console 日誌
5. **檢查信心度判斷**: 確認 Confidence Filter 邏輯
6. **檢查 Google Calendar**: 確認事件創建成功

### 常用調試代碼
```javascript
// 在任何 Function 節點中添加調試
console.log('調試數據:', JSON.stringify($json, null, 2));
console.log('數據類型:', typeof $json.confidence);
console.log('條件結果:', $json.status === 'high_confidence');
return [$input.all()];
```

## 📈 性能優化建議

### 1. 減少 API 調用
- 優化 AI prompt 減少重複分析
- 合併相似的處理邏輯

### 2. 改進錯誤處理
- 添加重試機制
- 改進錯誤訊息的用戶友好性

### 3. 增強功能
- 支援週期性事件
- 添加事件編輯功能
- 整合其他日曆服務

## 🆕 擴展功能建議

### 1. 會議室預訂整合
- 檢查會議室可用性
- 自動預訂會議室

### 2. 參與者管理
- 自動邀請相關人員
- 從 Slack 用戶映射到 Gmail

### 3. 智能建議
- 基於歷史數據的時間建議
- 衝突檢測和替代時間建議

### 4. 多語言支援
- 支援英文和其他語言
- 國際化時間格式

## 📝 維護注意事項

### 定期檢查
- [ ] Slack Token 有效性
- [ ] Azure OpenAI Quota 使用量
- [ ] Google Calendar API 限制
- [ ] Workflow 執行日誌

### 更新考量
- [ ] AI Model 版本更新
- [ ] Slack API 變更
- [ ] Google Calendar API 變更
- [ ] n8n 版本兼容性

## 🔗 相關文檔

- [n8n 官方文檔](https://docs.n8n.io/)
- [Slack API 文檔](https://api.slack.com/)
- [Google Calendar API](https://developers.google.com/calendar)
- [Azure OpenAI 文檔](https://docs.microsoft.com/en-us/azure/cognitive-services/openai/)

---

## 📧 支援信息

**系統版本**: n8n 1.93.0
**最後更新**: 2025-05-28
**功能狀態**: ✅ 生產可用

如有問題，請提供：
1. 具體的錯誤訊息
2. 相關的執行日誌
3. 輸入的測試數據
4. 期望的輸出結果