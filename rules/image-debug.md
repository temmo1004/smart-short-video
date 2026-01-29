# 圖片調試規則

> **SKILL 流程參考文件**
>
> 此文件描述 SKILL 流程中使用的圖片調試技術。
> 實際使用時，這些功能已整合在 `VideoProcessor.processShortVideo()` 內。

## 問題診斷

### 常見問題

| 問題 | 原因 | 解決方案 |
|------|------|----------|
| 圖片全部看不見 | Base64 編碼失敗 | 檢查 cacheImage() 函數 |
| 部分圖片看不見 | 關鍵字匹配不到 | 使用 MCP 語義搜尋擴展 |
| 日文檔名 404 | URL 編碼問題 | 已用 Base64 解決 |
| default.png 404 | fallback 圖片不存在 | 創建預設 fallback |

## 調試步驟

### 1. 檢查圖片資料庫
```bash
# 檢查資料庫是否存在
ls -lh data/irasutoya-image-database.json

# 檢查圖片數量
cat data/irasutoya-image-database.json | jq '.images | length'
```

### 2. 測試關鍵字搜尋
```bash
# 測試特定關鍵字
node -e "
import { IrasutoyaCrawler } from './src/ai-video/IrasutoyaCrawler.js';

(async () => {
  const crawler = new IrasutoyaCrawler();
  const results = await crawler.searchImages('GitHub', 3);
  console.log('找到圖片:', results.length);
  console.log('圖片 URL:', results);
})();
"
```

### 3. 檢查 Base64 轉換
```bash
# 測試 Base64 轉換
node -e "
import fs from 'fs';

const testImg = 'public/ai-images/default.png';
if (fs.existsSync(testImg)) {
  const content = fs.readFileSync(testImg);
  const base64 = content.toString('base64');
  console.log('Base64 長度:', base64.length);
  console.log('Data URI 長度:', base64.length + 30);
} else {
  console.log('檔案不存在:', testImg);
}
"
```

### 4. 驗證場景配置
```bash
# 檢查 scenes.json 中的圖片 URL
cat output/scenes.json | jq '.scenes[0].imageConfig.primary.url' | head -c 100
```

## 調試報告格式

```json
{
  "timestamp": "2025-01-27T10:00:00Z",
  "inputVideo": "input.mp4",
  "totalScenes": 21,
  "imageScenes": 10,
  "videoScenes": 11,
  "imageStats": {
    "totalSearched": 10,
    "foundInDatabase": 8,
    "usingFallback": 2,
    "usingMCPExpansion": 3
  },
  "imageStatus": [
    {
      "sceneIndex": 0,
      "keyword": "GitHub",
      "hasImage": true,
      "imageSource": "irasutoya",
      "isBase64": true,
      "imageSize": "245KB",
      "filename": "アイコン_1.png"
    },
    {
      "sceneIndex": 1,
      "keyword": "熱點",
      "hasImage": false,
      "imageSource": "fallback",
      "isBase64": true,
      "fallbackReason": "keyword not found in database"
    }
  ],
  "recommendations": [
    "為 '熱點' 關鍵字添加 MCP 同義詞映射",
    "考慮使用影片片段替代找不到圖片的場景"
  ]
}
```

## 修復建議

### MCP 語義擴展
```typescript
// 在 IrasutoyaCrawler 中使用 MCP 語義搜尋器
const crawler = new IrasutoyaCrawler('public/ai-images', {
  databasePath: path.join(process.cwd(), 'data', 'irasutoya-image-database.json'),
});

// 設置 MCP 語義搜尋器
crawler.setSemanticSearcher(async (query, availableKeywords) => {
  // 使用 MCP 服務擴展關鍵字
  return mcpSemanticSearch(query, availableKeywords);
});
```

### Fallback 策略
```typescript
// 當圖片找不到時的優先順序
1. MCP 語義擴展搜尋
2. 相似關鍵字搜尋
3. 隨機同分類圖片
4. 使用影片片段替代
5. SVG fallback (最後手段)
```

---

## SKILL 流程中的使用

在 `processShortVideo()` 中，圖片搜尋和調試是自動執行的：

```typescript
// 步驟 4: 搜尋圖片 (SKILL 流程自動執行)
const images = await this.crawler.searchBatch(script.keywords, 2);

// 如果需要啟用調試模式，檢查圖片狀態
const imageStatus = script.keywords.map((kw, i) => ({
  sceneIndex: i,
  keyword: kw.join(', '),
  hasImage: !!images[i]?.[0],
  imageSource: images[i]?.[0] ? 'irasutoya' : 'fallback',
}));
```
