# Smart Short Video 工作流程

> **純 Bash 命令逐步執行**
>
> 每個步驟都是獨立的 Bash 命令，可以手動執行並檢查中間結果。

---

## 步驟 1: 影片切片

```bash
# 建立臨時目錄
mkdir -p temp/video-processing

# 用 ffmpeg 切片（每 3 秒）
ffmpeg -i "input.mp4" \
  -c copy \
  -f segment \
  -segment_time 3 \
  -segment_start_number 0 \
  "temp/video-processing/segment_%03d.mp4"

# 計算片段數量
ls temp/video-processing/segment_*.mp4 | wc -l
```

---

## 步驟 2: Whisper 轉錄（全部片段）

```bash
# 建立輸出目錄
mkdir -p output

# 批量轉錄（large-v3-turbo 模型）
python3 << 'PYTHON_EOF'
import whisper
import json
import os

model = whisper.load_model('large-v3-turbo')
segment_dir = 'temp/video-processing/'
results = []

segments = sorted([f for f in os.listdir(segment_dir) if f.endswith('.mp4')])
print(f"找到 {len(segments)} 個片段，開始轉錄...")

for i, filename in enumerate(segments):
    filepath = os.path.join(segment_dir, filename)
    result = model.transcribe(filepath, task='transcribe', language='zh', verbose=False)
    text = result['text'].strip()
    results.append({'index': i, 'file': filename, 'text': text})
    print(f"[{i+1}/{len(segments)}] {text[:50]}...")

with open('output/transcripts.json', 'w', encoding='utf-8') as f:
    json.dump(results, f, ensure_ascii=False, indent=2)
print("✓ 轉錄完成！")
PYTHON_EOF
```

---

## 步驟 3: 關鍵字提取

```bash
# 從轉錄結果提取關鍵字
python3 << 'PYTHON_EOF'
import json
import ollama

# 讀取轉錄結果
with open('output/transcripts.json', 'r', encoding='utf-8') as f:
    transcripts = json.load(f)

# 提取關鍵字
keywords = []
for item in transcripts:
    prompt = f"""從以下文字提取 3-5 個關鍵字（用逗號分隔）：

{item['text']}

只輸出關鍵字，不要其他說明。"""

    response = ollama.generate(model='llama3', prompt=prompt)
    kw_list = [k.strip() for k in response['response'].split(',')]
    keywords.append({'index': item['index'], 'keywords': kw_list})
    print(f"[{item['index']}] {kw_list}")

# 保存關鍵字
with open('output/keywords.json', 'w', encoding='utf-8') as f:
    json.dump(keywords, f, ensure_ascii=False, indent=2)
print("✓ 關鍵字提取完成！")
PYTHON_EOF
```

---

## 步驟 4: AI 文案生成

```bash
# 生成短影片文案（選擇重要片段並重寫）
python3 << 'PYTHON_EOF'
import json

# 讀取轉錄和關鍵字
with open('output/transcripts.json', 'r', encoding='utf-8') as f:
    transcripts = json.load(f)
with open('output/keywords.json', 'r', encoding='utf-8') as f:
    keywords = json.load(f)

# TODO: 使用 AI 選擇重要片段並重寫成短影片文案
# 這步需要調用 LLM

script = {
    'selectedSegments': list(range(40)),  # 前 40 個片段
    'scriptTexts': [t['text'] for t in transcripts[:40]],
    'keywords': [k['keywords'] for k in keywords[:40]],
    'duration': 120
}

with open('output/script.json', 'w', encoding='utf-8') as f:
    json.dump(script, f, ensure_ascii=False, indent=2)
print("✓ 文案生成完成！")
PYTHON_EOF
```

---

## 步驟 5: 圖片搜尋

```bash
# 根據關鍵字搜尋圖片
python3 << 'PYTHON_EOF'
import json
from pathlib import Path

# 讀取關鍵字
with open('output/script.json', 'r', encoding='utf-8') as f:
    script = json.load(f)

# 搜尋圖片（使用本地資料庫）
image_db_path = 'data/irasutoya-image-database.json'
with open(image_db_path, 'r', encoding='utf-8') as f:
    image_db = json.load(f)

images = []
for i, kw_list in enumerate(script['keywords']):
    # 簡單匹配：找第一個包含關鍵字的圖片
    found = None
    for img in image_db['images']:
        for kw in kw_list:
            if kw in img.get('tags', []) or kw in img.get('filename', ''):
                found = img['filename']
                break
        if found:
            break
    images.append({'index': i, 'image': found or 'default.png'})
    print(f"[{i}] {kw_list} -> {found or 'default.png'}")

with open('output/images.json', 'w', encoding='utf-8') as f:
    json.dump(images, f, ensure_ascii=False, indent=2)
print("✓ 圖片搜尋完成！")
PYTHON_EOF
```

---

## 步驟 6: 影片片段提取

```bash
# 建立 segments 目錄
mkdir -p output/segments

# 根據 videoRatio=0.5 提取一半的影片片段
python3 << 'PYTHON_EOF'
import json
import subprocess

# 讀取選中的片段
with open('output/script.json', 'r', encoding='utf-8') as f:
    script = json.load(f)

selected = script['selectedSegments']
video_ratio = 0.5
video_count = int(len(selected) * video_ratio)

# 選擇要提取的片段（交錯策略）
video_indices = []
interval = len(selected) // (video_count + 1)
for i in range(1, video_count + 1):
    idx = min(i * interval, len(selected) - 1)
    video_indices.append(idx)

print(f"提取 {len(video_indices)} 個影片片段...")

for i, scene_idx in enumerate(video_indices):
    seg_idx = selected[scene_idx]
    start_time = seg_idx * 3

    # 用 ffmpeg 提取
    subprocess.run([
        'ffmpeg', '-y',
        '-i', '/Users/aes/Downloads/咒術迴戰modulo第19話完整解說：穿血！ - 五个光 (1080p, h264).mp4',
        '-ss', str(start_time),
        '-t', '3',
        '-c:v', 'libx264',
        '-c:a', 'aac',
        f'output/segments/segment_{scene_idx}.mp4'
    ], capture_output=True)
    print(f"[{i+1}/{len(video_indices)}] 提取片段 {scene_idx} (原始 {seg_idx})")

# 保存影片場景索引
with open('output/video-indices.json', 'w', encoding='utf-8') as f:
    json.dump(video_indices, f, ensure_ascii=False, indent=2)
print("✓ 影片片段提取完成！")
PYTHON_EOF
```

---

## 步驟 7: 場景組裝

```bash
# 生成 scenes.json
python3 << 'PYTHON_EOF'
import json
import base64
from pathlib import Path

# 讀取數據
with open('output/script.json', 'r', encoding='utf-8') as f:
    script = json.load(f)
with open('output/images.json', 'r', encoding='utf-8') as f:
    images = json.load(f)
with open('output/video-indices.json', 'r', encoding='utf-8') as f:
    video_indices = json.load(f)

video_set = set(video_indices)

scenes = []
for i in range(len(script['selectedSegments'])):
    is_video = i in video_set

    # 轉換圖片為 base64
    image_url = None
    if not is_video:
        img_path = Path(f"public/ai-images/{images[i]['image']}")
        if img_path.exists():
            with open(img_path, 'rb') as f:
                image_url = f"data:image/png;base64,{base64.b64encode(f.read()).decode()}"

    scene = {
        'id': f'scene-{i}',
        'index': i,
        'type': 'video' if is_video else 'image',
        'transcription': script['scriptTexts'][i],
        'keywords': script['keywords'][i],
        'duration': 3,
        'imageConfig': {
            'primary': {'url': image_url}
        } if not is_video else {'primary': {'url': None}},
        'videoSegmentPath': f'output/segments/segment_{i}.mp4' if is_video else None
    }
    scenes.append(scene)
    print(f"[{i}] {'影片' if is_video else '圖片'}: {script['scriptTexts'][i][:30]}...")

with open('output/scenes.json', 'w', encoding='utf-8') as f:
    json.dump({'scenes': scenes}, f, ensure_ascii=False, indent=2)
print("✓ 場景組裝完成！")
PYTHON_EOF
```

---

## 步驟 8: Remotion 渲染

```bash
# 渲染短影片
npx remotion render src/Root.tsx AIShortVideo output/short-video.mp4 \
  --overwrite \
  --width=1080 \
  --height=1920
```

---

## 快速執行（全部步驟）

```bash
# 1. 切片
ffmpeg -i "input.mp4" -c copy -f segment -segment_time 3 temp/seg_%03d.mp4

# 2. 轉錄
python3 -c "import whisper; m=whisper.load_model('large-v3-turbo'); ..."

# 3. 關鍵字
python3 -c "import ollama; ..."

# 4. 文案
# 5. 圖片
# 6. 提取片段
# 7. 場景
# 8. 渲染
```
