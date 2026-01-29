# Remotion 渲染規則

> **SKILL 流程參考文件**
>
> 此文件描述 SKILL 流程中使用的 Remotion 渲染技術。
> 實際使用時，這些功能已整合在 `VideoProcessor.processShortVideo()` 內的 `RemotionRenderer.renderShortVideo()` 方法。

## Composition 配置

### AIShortVideo Composition
```tsx
// src/Root.tsx
import { Composition } from 'remotion';
import { AIShortVideoPlayer } from './ai-video/components/AIShortVideoPlayer';

export const RemotionRoot = () => (
  <>
    <Composition
      id="AIShortVideo"
      component={AIShortVideoPlayer}
      durationInFrames={scenes.reduce((sum, s) => sum + s.duration * 30, 0)}
      fps={30}
      width={1080}
      height={1920}
      defaultProps={{ scenes }}
    />
  </>
);
```

---

## 混合場景組件

### AIShortVideoPlayer
```tsx
import React from 'react';
import { Series, AbsoluteFill } from 'remotion';
import { ImageScene } from './ImageScene';
import { VideoScene } from './VideoScene';
import { SceneData } from '../types';

interface AIShortVideoPlayerProps {
  scenes: SceneData[];
}

export const AIShortVideoPlayer: React.FC<AIShortVideoPlayerProps> = ({ scenes }) => {
  return (
    <AbsoluteFill>
      <Series>
        {scenes.map((scene) => {
          // 根據場景類型選擇組件
          if (scene.type === 'video' || scene.useVideo) {
            return (
              <Series.Sequence
                key={scene.id}
                durationInFrames={Math.round(scene.duration * 30)}
              >
                <VideoScene scene={scene} />
              </Series.Sequence>
            );
          }

          return (
            <Series.Sequence
              key={scene.id}
              durationInFrames={Math.round(scene.duration * 30)}
            >
              <ImageScene scene={scene} />
            </Series.Sequence>
          );
        })}
      </Series>
    </AbsoluteFill>
  );
};
```

---

### ImageScene 組件
```tsx
import React from 'react';
import { AbsoluteFill, Img } from 'remotion';
import { SubtitleOverlay } from './SubtitleOverlay';
import { SceneData } from '../types';

interface ImageSceneProps {
  scene: SceneData;
}

export const ImageScene: React.FC<ImageSceneProps> = ({ scene }) => {
  const imageUrl = scene.imageConfig?.primary?.url;

  return (
    <AbsoluteFill style={{ backgroundColor: '#ffffff' }}>
      {/* 圖片區域 */}
      {imageUrl && (
        <div style={{ position: 'absolute', top: 0, left: 0, width: '100%', height: '65%' }}>
          <Img
            src={imageUrl}
            style={{ width: '100%', height: '100%', objectFit: 'contain' }}
          />
        </div>
      )}

      {/* 字幕區域 */}
      <div style={{ position: 'absolute', bottom: 0, left: 0, width: '100%', height: '40%' }}>
        <SubtitleOverlay text={scene.transcription} />
      </div>
    </AbsoluteFill>
  );
};
```

---

### VideoScene 組件
```tsx
import React from 'react';
import { AbsoluteFill, Video } from 'remotion';
import { SubtitleOverlay } from './SubtitleOverlay';
import { SceneData } from '../types';

interface VideoSceneProps {
  scene: SceneData;
}

export const VideoScene: React.FC<VideoSceneProps> = ({ scene }) => {
  const videoPath = scene.videoSegmentPath;

  if (!videoPath) {
    // Fallback to image scene
    return <ImageScene scene={scene} />;
  }

  return (
    <AbsoluteFill>
      {/* 影片背景 */}
      <Video
        src={videoPath}
        startFrom={0}
        endAt={scene.duration}
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: '100%',
          objectFit: 'cover',
        }}
      />

      {/* 半透明遮罩 */}
      <div
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: '100%',
          backgroundColor: 'rgba(0, 0, 0, 0.3)',
        }}
      />

      {/* 字幕疊加 */}
      <div
        style={{
          position: 'absolute',
          bottom: 60,
          left: 0,
          width: '100%',
          display: 'flex',
          justifyContent: 'center',
        }}
      >
        <SubtitleOverlay
          text={scene.transcription}
          fontSize={42}
          color="#ffffff"
          style={{ textShadow: '0 2px 4px rgba(0,0,0,0.8)' }}
        />
      </div>
    </AbsoluteFill>
  );
};
```

---

## 渲染指令

### 基本渲染
```bash
npx remotion render src/Root.tsx AIShortVideo output/short-video.mp4 \
  --overwrite \
  --codec=h264 \
  --quality=80 \
  --log=verbose
```

### 指定輸出尺寸
```bash
# 直式 9:16 (TikTok/Reels/Shorts)
npx remotion render src/Root.tsx AIShortVideo output/short-video.mp4 \
  --width=1080 \
  --height=1920 \
  --overwrite

# 橫式 16:9
npx remotion render src/Root.tsx AIShortVideo output/short-video.mp4 \
  --width=1920 \
  --height=1080 \
  --overwrite
```

### 並行渲染
```bash
# 使用 4 個並行進程
npx remotion render src/Root.tsx AIShortVideo output/short-video.mp4 \
  --concurrency=4 \
  --overwrite
```

---

## 渲染優化

### 1. 預先處理影片片段
```bash
# 確保所有片段都已提取且格式一致
mkdir -p output/segments

# 標準化所有片段為相同格式
for f in output/segments/*.mp4; do
  ffmpeg -i "$f" \
    -c:v libx264 -preset fast -crf 23 \
    -c:a aac -b:a 128k \
    -y "${f%.mp4}_processed.mp4"
done
```

### 2. 使用 Puppeteer 渲染
```bash
# Remotion 預設使用 Puppeteer
# 確保 Chromium 已安裝
npx remotion render --browser-executable=/path/to/chrome
```

### 3. 幀數優化
```bash
# 減少不必要的幀數
npx remotion render src/Root.tsx AIShortVideo output.mp4 \
  --frames="0-300" \
  --overwrite
```

---

## 常見問題

### 1. 影片載入失敗
```
錯誤: Failed to load video: output/segments/segment_5.mp4
```
**解決方案**: 確保影片片段路徑正確，使用絕對路徑或相對於 public 目錄的路徑

### 2. Base64 圖片過大
```
錯誤: Data URI too large
```
**解決方案**: 壓縮圖片或使用外部 URL

### 3. 渲染中斷
```
錯誤: Render failed at frame 150
```
**解決方案**: 使用 `--frames="0-150"` 重新渲染，然後拼接

---

## 輸出檢查

### 驗證輸出
```bash
# 檢查影片時長
ffprobe -v quiet -show_entries format=duration -of csv=p=0 output/short-video.mp4

# 檢查影片尺寸
ffprobe -v quiet -show_entries stream=width,height -of csv=p=0 output/short-video.mp4

# 檢查影片編碼
ffprobe -v quiet -show_entries stream=codec_name -of csv=p=0 output/short-video.mp4
```

---

## SKILL 流程中的使用

在 `processShortVideo()` 中，Remotion 渲染是自動執行的：

```typescript
// 步驟 6: Remotion 渲染短影片 (SKILL 流程自動執行)
const outputPath = path.join(outputDir, 'short-video.mp4');

const renderResult = await this.remotionRenderer.renderShortVideo(
  scenes,
  outputPath,
  { maxDuration }
);
```

`RemotionRenderer.renderShortVideo()` 方法會：
1. 使用 `remotion render` 指令渲染影片
2. 自動處理場景配置
3. 支援圖片和影片場景混合
4. 返回渲染結果（包含實際時長、使用場景數等）
