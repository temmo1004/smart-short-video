# Smart Short Video - 智能短影片生成器

混合 AI 圖片與原始影片片段，生成 TikTok/Reels/Shorts 短影片。

## 功能特色

- 🎬 **影片切片**: 每 3 秒一段，自動分割
- 🗣️ **Whisper 轉錄**: 支援本地/OpenAI API 語音轉文字
- 🎯 **片段重要性分析**: AI 自動選取重要片段
- ✍️ **AI 文案重寫**: 簡潔有力的短影片文案 (12-15字)
- 🎨 **AI 生圖**: 支援 GLM (glm-image)、Pollinations.ai、DALL-E
- 🔀 **場景隨機混合**: Fisher-Yates 洗牌演算法
- 🎥 **Remotion 渲染**: 輸出高品質短影片

## 安裝

### 選項 1: 下載打包檔案

```bash
# 下載並解壓縮到 Claude Code skills 目錄
tar -xzf packages/smart-short-video-skill.tar.gz -D ~/.claude/skills/
```

### 選項 2: 直接下載 SKILL.md

將 `SKILL.md` 和 `rules/` 目錄放到 `~/.claude/skills/smart-short-video/`

## 使用方式

在 Claude Code 中執行：

```
/smart-short-video /path/to/your/video.mp4
```

## 支援的 AI 生圖服務

| 服務 | 模型 | 說明 |
|------|------|------|
| GLM | glm-image | 智譜 AI，需要 API Key |
| Pollinations.ai | - | 免費，無需 API Key |
| DALL-E | dall-e-3 | OpenAI，需要 API Key |

## GLM API 配置

```bash
# 設定 API Key
export GLM_API_KEY="your_glm_api_key"
```

## 授權

MIT License
