# 📚 自動化知識管理系統 - 完整技術文檔

## 🎯 專案概述

### 系統目標
建立全自動的知識收藏與整理工作流程：**Raindrop.io 收藏 → AI 智能分析 → Notion 結構化儲存**

### 核心功能
- ⏰ 每 30 分鐘自動檢查新收藏
- 🌐 智能網頁內容抓取
- 🤖 Azure AI 內容分析與分類
- 📋 結構化 Notion 頁面生成
- 🏷️ 智能標籤清理與標準化

## 🏗️ 系統架構

### 技術棧
- **自動化平台**: N8N Self-hosted (v1.93.0)
- **收藏來源**: Raindrop.io (OAuth2 API)
- **AI 分析**: Azure AI Foundry (GPT-4o-mini)
- **資料儲存**: Notion API
- **部署環境**: Self-hosted

### 工作流程圖
```
Schedule Trigger (30分鐘)
    ↓
取得 Raindrop 收藏
    ↓
篩選新項目 (35分鐘時間窗口)
    ↓
抓取網頁內容 (多筆資料處理)
    ↓
處理內容 (文本清理 + AI判斷)
    ↓
是否使用 AI 分析 (IF節點 - 目前強制為true)
    ├── True → Azure AI 內容分析 → 整合 AI 分析結果
    └── False → 基本分析
    ↓
合併分析結果 (Merge節點)
    ↓
存入 Notion 資料庫 (包含豐富頁面內容)
```

## ⚙️ 詳細節點配置

### 1. Schedule Trigger
```yaml
觸發間隔: 30分鐘
規則: 分鐘間隔
狀態: 啟用
```

### 2. 取得 Raindrop 收藏
```yaml
認證: Raindrop OAuth2 API
端點: https://api.raindrop.io/rest/v1/raindrops/0
參數:
  - sort: -created
  - perpage: 10
回應格式: JSON
```

### 3. 篩選新項目 (Code節點)
```javascript
// 關鍵邏輯：使用時間窗口篩選 (解決 Self-hosted staticData 限制)
const scheduleInterval = 30; // 分鐘
const timeWindow = (scheduleInterval + 5) * 60 * 1000; // 35分鐘緩衝
const cutoffTime = Date.now() - timeWindow;
const maxItems = 5; // 每次最多處理5個項目

// 過濾邏輯
const newItems = sortedItems.filter(item => {
  const createdTime = new Date(item.created).getTime();
  return createdTime > cutoffTime;
}).slice(0, maxItems);
```

### 4. 抓取網頁內容 (Code節點)
```javascript
// 支援多筆資料處理，針對不同網站特殊處理
// 包含 Reddit、Twitter、LinkedIn 特殊 User-Agent
// 失敗時使用 Raindrop 摘要作為後備內容

主要功能:
- 多網站 User-Agent 策略
- Reddit old.reddit.com 後備
- 失敗處理與後備內容
- 支援批次處理
```

### 5. 處理內容 (Code節點)
```javascript
// 強化文本清理 + AI判斷邏輯
function sanitizeForJson(text) {
  return text
    .replace(/[\u0000-\u001F\u007F-\u009F]/g, '') // 控制字符
    .replace(/\\/g, '\\\\')  // 轉義反斜線
    .replace(/"/g, '\\"')    // 轉義雙引號
    .replace(/'/g, "\\'")    // 轉義單引號
    .replace(/[\r\n\t]/g, ' ') // 統一空白字符
    .replace(/\s+/g, ' ')      // 合併多餘空格
    // Markdown 清理
    .replace(/^#+\s*/gm, '')   // 移除標題符號
    .replace(/\*\*(.*?)\*\*/g, '$1') // 移除粗體
    .replace(/\*(.*?)\*/g, '$1')     // 移除斜體
    .replace(/`(.*?)`/g, '$1')       // 移除程式碼標記
    .trim();
}

// AI判斷邏輯（目前被後續節點強制覆蓋）
const worthAnalyzing = cleanContent.length > 300 && 
                      cleanContent.length < 8000 && 
                      !originalData.domain.includes('twitter.com');
```

### 6. 是否使用 AI 分析 (IF節點)
```yaml
設定: 目前強制所有項目使用 AI
實際條件: ={{ true }}  # 強制通過
原始邏輯: {{ $json.useAI === true }}
```

### 7. Azure AI 內容分析 (HTTP Request節點)
```yaml
URL: https://demo-ai-svc-res929348950322.cognitiveservices.azure.com/openai/deployments/o4-mini/chat/completions?api-version=2025-01-01-preview
方法: POST
認證: Azure OpenAI API (Header Auth)
關鍵參數: max_completion_tokens: 1000 (非 max_tokens)

JSON Body:
{
  "messages": [
    {
      "role": "system",
      "content": "你是一個專業的內容分析師。請分析網頁內容並提供結構化分析..."
    },
    {
      "role": "user", 
      "content": "={{ '標題：' + ($json.title || 'N/A') + '\\n網站：' + ($json.domain || 'N/A') + '\\n內容：\\n' + ($json.cleanContent || 'N/A').substring(0, 3500) }}"
    }
  ],
  "max_completion_tokens": 1000
}
```

### 8. 整合 AI 分析結果 (Code節點)
```javascript
// 支援多筆資料處理
const azureItems = $input.all();
const processedItems = $('處理內容').all();

// JSON 解析邏輯（處理 ```json``` 包裝）
const jsonMatch = aiContent.match(/```json\s*([\s\S]*?)\s*```/);
const jsonString = jsonMatch ? jsonMatch[1] : aiContent;

// 智能標籤清理
function cleanAndProcessTags(originalTags, suggestedTags) {
  // 移除 r/ 前綴、特殊字符清理、標籤標準化
  // 支援 ChatGPT, JavaScript, Python 等標準化
}
```

### 9. 基本分析 (Code節點)
```javascript
// 針對不使用 AI 的項目（目前未使用）
// 包含 Reddit、Facebook 特殊分析邏輯
```

### 10. 存入 Notion 資料庫
```yaml
資料庫ID: 1ff930aa1b1280278ff5d95c98ce3ffd
資料庫名稱: 📚 學習資源庫

屬性設定:
- 標題|title: ={{$json.title}}
- 連結|url: ={{$json.url}} 
- 摘要|rich_text: ={{$json.summary}}
- 重點|rich_text: ={{$json.keyPoints}}
- 標籤|multi_select: ={{$json.tags}}
- 優先級|select: ={{$json.priority}}
- 預估閱讀時間|number: ={{$json.readingTime}}
- 分類|select: ={{$json.category}}
- 難度|select: ={{$json.difficulty}}
- 來源網站|rich_text: ={{$json.domain}}
- 收藏日期|date: ={{$json.collectedDate}}
- 狀態|select: ={{$json.status}}
```

### 11. Notion 頁面內容 (Block Contents)
```yaml
支援的Block類型:
- Heading 1, 2, 3
- Paragraph  
- Bulleted List Item
- To-Do
- Toggle (但不支援Children)

主要內容結構:
1. 標題 (Heading 1 + 藍色)
2. 元資訊段落 (來源、日期、時間)
3. 分類資訊段落 (分類、優先級、難度)
4. 分隔線 (特殊字符)
5. 內容摘要 (Heading 2 + Paragraph)
6. 重點摘要 (Heading 2 + Paragraph)
7. 學習目標 (Heading 2 + Paragraph)
8. 內容資訊 (Heading 2 + Bulleted List Items)
9. 標籤 (Heading 2 + Code格式Paragraph)
10. 學習進度追蹤 (Heading 2 + 6個To-Do項目)
11. 學習筆記 (Heading 2 + 空白Paragraphs + Heading 3子區域)
12. 原始連結 (Heading 2 + 連結Paragraph)
13. 系統資訊 (Heading 3 + Bulleted List Items)
```

## 🔧 關鍵技術解決方案

### 1. N8N Self-hosted 限制
**問題**: 無法使用 `$workflow.staticData` 和 `fs` 模組
**解決**: 改用時間窗口篩選 (35分鐘) 來識別新項目

### 2. Azure OpenAI API 更新
**問題**: `max_tokens` 參數已廢棄
**解決**: 使用 `max_completion_tokens` 參數

### 3. 多筆資料處理
**問題**: 部分節點只處理第一筆資料
**解決**: 使用 `$input.all()` 和 for 迴圈處理所有項目

### 4. JSON 格式錯誤
**問題**: 特殊字符導致 Azure AI 請求失敗
**解決**: `sanitizeForJson()` 函數全面清理文本

### 5. Notion Block 限制
**問題**: N8N 不支援 divider, callout, table, quote, bookmark
**解決**: 使用基礎類型和特殊字符模擬效果

### 6. Toggle Children 不支援
**問題**: N8N Notion 節點不支援 Toggle 的 Children 設定
**解決**: 改用 Heading 3 + Bulleted List Items

## 📊 系統狀態總覽

### ✅ 正常運作功能
- [x] 自動收藏檢測 (30分鐘間隔)
- [x] 多網站內容抓取 (含特殊處理)
- [x] AI 內容分析 (GPT-4o-mini)
- [x] 智能標籤清理與標準化
- [x] Notion 頁面自動生成
- [x] 多筆資料批次處理
- [x] 學習進度追蹤功能

### ⚠️ 已知限制
- Reddit/Twitter 等網站抓取成功率較低 (使用後備內容)
- Toggle 摺疊功能無法實現 (改用平鋪結構)
- AI 分析目前強制啟用 (IF 節點設為 true)

### 🔧 系統配置
- **時間窗口**: 35 分鐘
- **批次大小**: 最多 5 個項目/次
- **AI 強制**: 目前為 true
- **內容長度限制**: 3500 字符

## 🚨 故障排除指南

### 常見問題 1: 沒有新項目被處理
```
檢查項目:
1. Schedule Trigger 是否啟用
2. Raindrop OAuth2 認證是否有效
3. 時間窗口設定 (35分鐘)
4. 檢查 "篩選新項目" 節點的 console.log
```

### 常見問題 2: Azure AI 分析失敗
```
檢查項目:
1. Azure AI 認證是否有效
2. API 端點 URL 是否正確
3. max_completion_tokens 參數
4. 文本清理是否正確 (sanitizeForJson)
```

### 常見問題 3: Notion 頁面沒有內容
```
檢查項目:
1. Notion 資料庫 ID 是否正確
2. Block Contents 是否正確設定
3. 表達式語法是否正確
4. 點擊記錄左側圖標開啟完整頁面 (非文字內容)
```

### 常見問題 4: 多筆資料變單筆
```
檢查項目:
1. 使用 $input.all() 而非 $input.first()
2. for 迴圈處理所有項目
3. 返回 results 陣列而非單一物件
```

## 🎯 效能監控指標

### 每日統計
- 處理成功率: >95%
- AI 分析準確度: 根據用戶反饋
- API 配額使用: Azure AI tokens
- 錯誤類型分佈: 網站抓取 vs AI 分析

### 成本控制
- GPT-4o-mini Token 價格: ~$0.00015/1000 tokens
- 每月預估成本: 基於處理量計算
- Token 使用追蹤: 記錄在 Notion 頁面中

## 🚀 優化建議與擴展

### 短期優化
1. **成本監控**: 添加月度 Token 使用統計
2. **內容品質**: 基於長度和來源調整 AI 觸發條件
3. **標籤優化**: 根據使用頻率調整標籤標準化規則

### 中期擴展
1. **多來源整合**: GitHub Stars, Twitter Bookmarks
2. **智能分類**: 基於內容自動分配到不同資料庫
3. **學習追蹤**: 閱讀進度和複習提醒

### 長期願景
1. **知識圖譜**: 內容間關聯性分析
2. **個人化推薦**: 基於閱讀歷史的智能推薦
3. **協作功能**: 團隊知識共享機制

## 📝 維護檢查清單

### 每週檢查
- [ ] API 配額使用情況
- [ ] 錯誤日誌審查
- [ ] 新收藏處理狀況

### 每月檢查  
- [ ] 標籤清理規則效果評估
- [ ] AI 分析品質評估
- [ ] 成本使用統計

### 季度檢查
- [ ] 整體系統效能評估
- [ ] 用戶需求變化分析
- [ ] 技術債務清理

## 🔗 重要連結與資源

### API 文檔
- [Raindrop.io API](https://developer.raindrop.io/)
- [Azure OpenAI API](https://docs.microsoft.com/azure/cognitive-services/openai/)
- [Notion API](https://developers.notion.com/)

### N8N 相關
- [N8N Notion 節點文檔](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion/)
- [N8N Code 節點指南](https://docs.n8n.io/code-examples/)

---

## 📋 Debug 資訊範本

當遇到問題時，請提供以下資訊：

```
問題描述: [簡要描述問題]

影響範圍: [哪個節點或功能受影響]

錯誤訊息: [完整的錯誤訊息]

輸入資料: [相關節點的輸入資料]

系統狀態: [其他節點是否正常運作]

最近變更: [是否有修改過設定]
```

**此文檔版本**: v2.0  
**最後更新**: 2025-05-27  
**系統狀態**: ✅ 正常運作