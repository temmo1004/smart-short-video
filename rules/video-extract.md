# 影片片段提取規則

> **SKILL 流程參考文件**
>
> 此文件描述 SKILL 流程中使用的影片片段提取技術。
> 實際使用時，這些功能已整合在 `VideoProcessor.processShortVideo()` 內的 `extractVideoSegmentsForMix()` 方法。

## 提取策略

### 根據 videoRatio 決定提取哪些片段

```typescript
/**
 * 選擇要使用影片片段的場景索引
 * @param totalScenes 場景總數
 * @param videoRatio 影片占比 0-1
 * @returns 影片場景的索引數組
 */
function selectVideoSceneIndices(
  totalScenes: number,
  videoRatio: number
): number[] {
  const videoCount = Math.round(totalScenes * videoRatio);
  const videoIndices: number[] = [];

  // 交錯策略：均勻分佈影片場景
  const interval = Math.floor(totalScenes / (videoCount + 1));

  for (let i = 1; i <= videoCount; i++) {
    const idx = Math.min(i * interval, totalScenes - 1);
    videoIndices.push(idx);
  }

  return videoIndices;
}
```

### 分配範例

| videoRatio | 場景數 | 影片場景 | 索引分配 |
|------------|--------|----------|----------|
| 0.0 | 20 | 0 | [] |
| 0.25 | 20 | 5 | [3, 7, 11, 15, 19] |
| 0.5 | 20 | 10 | [1, 3, 5, 7, 9, 11, 13, 15, 17, 19] |
| 0.75 | 20 | 15 | [0, 1, 2, 4, 5, 6, 8, 9, 10, 12, 13, 14, 16, 17, 18] |
| 1.0 | 20 | 20 | [0, 1, 2, ..., 19] |

---

## FFmpeg 提取指令

### 精確時間提取
```bash
# 從原始影片提取指定時間段的片段
ffmpeg -i input.mp4 \
  -ss 00:00:15 \
  -t 00:00:03 \
  -c:v libx264 \
  -c:a aac \
  -y output/segments/segment_0.mp4
```

### 基於片段索引提取
```bash
# 使用 VideoSplitter 生成的片段
ffmpeg -i temp/video-processing/segment_5.mp4 \
  -c copy \
  -y output/segments/segment_5.mp4
```

### 批次提取腳本
```typescript
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

async function extractVideoSegments(
  inputPath: string,
  segments: Array<{ index: number; startTime: number; duration: number }>,
  outputDir: string
): Promise<string[]> {
  const extractedPaths: string[] = [];

  for (const segment of segments) {
    const outputPath = `${outputDir}/segment_${segment.index}.mp4`;

    await execAsync(
      `ffmpeg -i "${inputPath}" ` +
      `-ss ${segment.startTime} ` +
      `-t ${segment.duration} ` +
      `-c:v libx264 -preset fast -crf 23 ` +
      `-c:a aac -b:a 128k ` +
      `-y "${outputPath}"`
    );

    extractedPaths.push(outputPath);
    console.log(`✓ 提取片段 ${segment.index}: ${outputPath}`);
  }

  return extractedPaths;
}
```

---

## 影片片段配置

### 場景中的影片配置
```json
{
  "id": "scene-5",
  "index": 5,
  "type": "video",
  "transcription": "這段用原始影片",
  "duration": 3,
  "imageConfig": {
    "primary": {
      "url": null
    }
  },
  "videoSegmentPath": "output/segments/segment_5.mp4",
  "useVideo": true,
  "videoConfig": {
    "startTime": 0,
    "endTime": 3,
    "opacity": 1,
    "objectFit": "cover"
  }
}
```

---

## Remotion 中的影片播放

### 使用 Video 標籤
```tsx
import { Video } from 'remotion';

// 影片場景組件
export const VideoScene: React.FC<{ scene: SceneData }> = ({ scene }) => {
  if (!scene.videoSegmentPath) {
    return null;
  }

  return (
    <Video
      src={scene.videoSegmentPath}
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
  );
};
```

---

## 優化建議

### 1. 預先提取所有片段
```bash
# 在渲染前一次性提取所有需要的片段
for i in {0..19}; do
  ffmpeg -i input.mp4 -ss $((i * 3)) -t 3 -c copy output/segments/segment_$i.mp4
done
```

### 2. 使用硬連接節省空間
```bash
# 如果不需要重新編碼，使用硬連接
ffmpeg -i input.mp4 -ss 0 -t 3 -c copy -y output/segments/segment_0.mp4
```

### 3. 並行提取
```typescript
// 使用 Promise.all 並行提取
await Promise.all(
  segments.map(seg => extractSegment(seg))
);
```

---

## SKILL 流程中的使用

在 `processShortVideo()` 中，影片片段提取是自動執行的：

```typescript
// 步驟 4.5: 提取影片片段 (SKILL 流程自動執行)
if (videoRatio > 0) {
  videoSegmentPaths = await this.extractVideoSegmentsForMix(
    inputPath,
    script.selectedSegments,
    script.keywords.length,
    videoRatio,
    segmentDuration,
    outputDir
  );
}
```

`extractVideoSegmentsForMix()` 方法會：
1. 根據 `videoRatio` 計算需要提取的影片片段數量
2. 使用交錯策略選擇影片場景索引
3. 使用 ffmpeg 從原始影片提取片段
4. 儲存到 `output/segments/segment_{n}.mp4`
5. 返回 `Map<場景索引, 片段路徑>`
