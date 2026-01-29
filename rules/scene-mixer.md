# 場景混合邏輯

> **SKILL 流程參考文件**
>
> 此文件描述 SKILL 流程中使用的場景混合技術。
> 實際使用時，這些功能已整合在 `VideoProcessor.processShortVideo()` 內。

## 混合策略

### 交錯模式 (預設)
```typescript
/**
 * 交錯混合：圖片和影片場景交替出現
 * 適合：videoRatio = 0.5 (各一半)
 */
function interleaveMix(
  scenes: SceneData[],
  videoRatio: number
): SceneData[] {
  const videoCount = Math.round(scenes.length * videoRatio);
  const videoIndices = selectVideoSceneIndices(scenes.length, videoRatio);

  return scenes.map((scene, i) => ({
    ...scene,
    type: videoIndices.includes(i) ? 'video' : 'image',
  }));
}
```

### 前後分段模式
```typescript
/**
 * 前後分段：前半段用圖片，後半段用影片
 * 適合：開頭吸引，後面展示實際內容
 */
function splitMix(
  scenes: SceneData[],
  videoRatio: number
): SceneData[] {
  const splitPoint = Math.floor(scenes.length * (1 - videoRatio));

  return scenes.map((scene, i) => ({
    ...scene,
    type: i >= splitPoint ? 'video' : 'image',
  }));
}
```

### 規律間隔模式
```typescript
/**
 * 規律間隔：每 N 個場景插入一個影片場景
 * 適合：定期展示實際影片片段
 */
function intervalMix(
  scenes: SceneData[],
  videoRatio: number
): SceneData[] {
  const interval = Math.round(1 / videoRatio);

  return scenes.map((scene, i) => ({
    ...scene,
    type: (i + 1) % interval === 0 ? 'video' : 'image',
  }));
}
```

---

## 場景類型定義

```typescript
interface SceneData {
  id: string;
  index: number;
  type: 'image' | 'video';  // 場景類型

  // 通用欄位
  transcription: string;
  keywords: string[];
  duration: number;
  importanceScore?: number;

  // 圖片場景專用
  imageUrl?: string;
  imageConfig?: {
    primary: {
      url: string | null;  // Base64 或 null
      type: 'irasutoya' | 'ai-generated' | 'fallback';
    };
    icon?: string;
  };

  // 影片場景專用
  useVideo?: boolean;
  videoSegmentPath?: string | null;
  videoConfig?: {
    startTime: number;
    endTime: number;
    opacity: number;
    objectFit: 'cover' | 'contain' | 'fill';
  };
}
```

---

## 混合模式選擇

```typescript
type MixMode = 'interleave' | 'split' | 'interval' | 'random';

function mixScenes(
  scenes: SceneData[],
  videoRatio: number,
  mode: MixMode = 'interleave'
): SceneData[] {
  switch (mode) {
    case 'interleave':
      return interleaveMix(scenes, videoRatio);
    case 'split':
      return splitMix(scenes, videoRatio);
    case 'interval':
      return intervalMix(scenes, videoRatio);
    case 'random':
      return randomMix(scenes, videoRatio);
    default:
      return interleaveMix(scenes, videoRatio);
  }
}
```

---

## 視覺化範例

### videoRatio = 0.5, mode = 'interleave'
```
場景: 0    1    2    3    4    5    6    7    8    9
類型: 🖼️  🎬  🖼️  🎬  🖼️  🎬  🖼️  🎬  🖼️  🎬
```

### videoRatio = 0.25, mode = 'interleave'
```
場景: 0    1    2    3    4    5    6    7    8    9
類型: 🖼️  🖼️  🎬  🖼️  🖼️  🎬  🖼️  🖼️  🎬  🖼️
```

### videoRatio = 0.5, mode = 'split'
```
場景: 0    1    2    3    4    5    6    7    8    9
類型: 🖼️  🖼️  🖼️  🖼️  🖼️  🎬  🎬  🎬  🎬  🎬
       (前半段圖片)        (後半段影片)
```

### videoRatio = 0.33, mode = 'interval'
```
場景: 0    1    2    3    4    5    6    7    8
類型: 🖼️  🖼️  🎬  🖼️  🖼️  🎬  🖼️  🖼️  🎬
       (每 3 個場景插入影片)
```

---

## 動態調整

### 基於重要性調整
```typescript
/**
 * 重要場景用影片，不重要用圖片
 */
function importanceBasedMix(
  scenes: SceneData[],
  importanceScores: number[]
): SceneData[] {
  // 排序並選擇前 50% 重要的場景用影片
  const sorted = scenes
    .map((s, i) => ({ scene: s, score: importanceScores[i] }))
    .sort((a, b) => b.score - a.score);

  const videoCount = Math.floor(scenes.length / 2);
  const videoSceneIds = new Set(
    sorted.slice(0, videoCount).map(s => s.scene.id)
  );

  return scenes.map(scene => ({
    ...scene,
    type: videoSceneIds.has(scene.id) ? 'video' : 'image',
  }));
}
```

### 基於關鍵字調整
```typescript
/**
 * 特定關鍵字的場景優先使用影片
 */
function keywordBasedMix(
  scenes: SceneData[],
  videoKeywords: string[]
): SceneData[] {
  return scenes.map(scene => {
    const hasVideoKeyword = scene.keywords.some(kw =>
      videoKeywords.some(vk => kw.includes(vk))
    );

    return {
      ...scene,
      type: hasVideoKeyword ? 'video' : 'image',
    };
  });
}
```

---

## SKILL 流程中的使用

在 `processShortVideo()` 中，場景混合是自動執行的：

```typescript
// 步驟 5: 生成場景（混合模式）(SKILL 流程自動執行)
const videoSceneIndices = this.selectVideoSceneIndices(scenes.length, videoRatio);

scenes = scenes.map((scene, i) => {
  const isVideoScene = videoSceneIndices.includes(i);
  return {
    ...scene,
    transcription: selectedTexts[i] || scene.transcription,
    keywords: [...new Set([...scene.keywords, ...(selectedHighlights[i] || [])])].slice(0, 5),
    importanceScore: importanceScores[i] || 0.7,
    // 混合模式配置
    useVideo: isVideoScene,
    videoSegmentPath: isVideoScene ? (videoSegmentPaths.get(i) || null) : null,
    imageUrl: isVideoScene ? undefined : scene.imageUrl,
    imageConfig: isVideoScene
      ? { primary: { url: null }, icon: scene.imageConfig?.icon }
      : scene.imageConfig,
  };
});
```

`selectVideoSceneIndices()` 方法使用交錯策略：
1. 根據 `videoRatio` 計算需要的影片場景數量
2. 使用均勻間隔分配影片場景索引
3. 返回影片場景的索引數組
