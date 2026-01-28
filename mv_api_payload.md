# AutoMV Engine API 資料結構文件

本文件記錄所有功能的 API 回傳資料格式。

---

## 通用格式說明

### 1. 核心 Payload 結構

所有任務提交都使用以下 payload 格式提交到 worker：

```json
{
  "session_id": "xxxxx",
  "effect": "automv",
  "acts": [{
    "id": 0,
    "params": {
      "musicName": "音樂名稱",
      "useVocal": true/false,
      "startNum": 1,
      "endNum": 5,
      "functions": ["storyboard_initial", "actor_setting", "..."],
      "characterNames": ["角色1", "角色2"],
      "versionIndices": {
        "storyboard_initial": 0,
        "storyboard_refiner": 1
      },
      "character_selections": {
        "角色名稱": 0
      },
      "paragraph_selections": {
        "1": 0,
        "2": 1
      },
      "cacheKeys": ["cache_key_1", "cache_key_2"]
    }
  }]
}
```

#### Payload 欄位說明

| 欄位 | 類型 | 必填 | 說明 |
|-----|------|-----|------|
| `session_id` | string | ✅ | 會話 ID，用於標識用戶會話 |
| `effect` | string | ✅ | 固定為 "automv" |
| `acts` | array | ✅ | 動作陣列，包含任務參數 |
| `acts[0].id` | number | ✅ | 動作 ID，固定為 0 |
| `acts[0].params` | object | ✅ | 任務參數物件 |

#### Params 欄位說明

| 欄位 | 類型 | 必填 | 說明 | 使用場景 |
|-----|------|-----|------|---------|
| `musicName` | string | ✅ | 音樂名稱 | 所有功能 |
| `useVocal` | boolean | ✅ | 是否使用人聲 | 所有功能 |
| `startNum` | number | ✅ | 起始段落編號 | 所有功能 |
| `endNum` | number | ✅ | 結束段落編號 | 所有功能 |
| `functions` | array | ✅ | 要執行的功能列表 | 所有功能 |
| `characterNames` | array | ❌ | 角色名稱列表 | `character_gen` |
| `versionIndices` | object | ❌ | 版本索引映射 | 使用特定版本時 |
| `character_selections` | object | ❌ | 角色選擇映射（角色名→索引） | `character_gen_select` |
| `paragraph_selections` | object | ❌ | 段落選擇映射（段落號→索引） | `start_frame_gen_select`, `video_gen_select` |
| `cacheKeys` | array | ❌ | 要清除的 cache keys | `clear_cache` |

#### Functions 列表

| Function Name | 說明 | 前端按鈕 ID |
|--------------|------|------------|
| `storyboard_initial` | 初始故事板生成 + 音樂情感分析 | 1.1 |
| `storyboard_refiner` | 故事板精煉 | 1.3 |
| `actor_setting` | 角色設定 | 2 |
| `character_gen` | 角色圖片生成 | 2.1 |
| `start_frame_gen` | 起始幀生成 | 3 |
| `video_gen_unified` | 統一視頻生成（lip_sync + full_video_gen） | 4 |
| `video_merge` | 視頻合併 | 5 |
| `query_cache` | 查詢 cache | - |
| `clear_cache` | 清除 cache | - |
| `get_cache_json` | 獲取完整 ProjectCache | - |
| `character_gen_select` | 角色圖片選擇 | - |
| `start_frame_gen_select` | 段落圖片選擇 | - |
| `video_gen_select` | 視頻段落選擇 | - |

### 2. Task Status API 標準回傳格式
```json
{
  "job_id": "任務ID",
  "task_id": "邏輯任務ID（可選）",
  "session_id": "Session ID",
  "status": "pending" | "running" | "success" | "error" | "timeout",
  "functions": ["功能名稱"],
  "music_name": "歌曲名稱（可選）",
  "response_data": {
    "return_message": /* 實際資料 */,
    "return_message_original": "原始回傳（可選）"
  },
  "terminal_logs": [
    {
      "time": "HH:MM:SS",
      "message": "狀態或訊息",
      "type": "info" | "status" | "success" | "error" | "warning"
    }
  ],
  "start_time": 1700000000.0,
  "error": "錯誤訊息（僅在失敗時）"
}
```

依 API 不同可能會附加欄位：`query_function`、`character_name`、`paragraph_number`、`paragraphs`、`idx`、`cast` 等。

### 3. Query Cache 提交回傳
Query Cache 會先回傳任務提交結果，實際資料需透過 Task Status/Task Response 取得：
```json
{
  "success": true,
  "job_id": "job-id",
  "session_id": "session-id",
  "query_function": "功能名稱",
  "message": "查詢任務已提交"
}
```

---

## 1. Storyboard 系列

### 1.1 Storyboard Initial

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": [...]
  },
  "query_function": "storyboard_initial"
}
```

#### return_message 內容（陣列）
```json
[
  {
    "paragraph_id": "1",
    "paragraph_content": "段落的歌詞內容",
    "scene_description": "場景描述",
    "action_description": "動作描述",
    "character_emotion": "角色情緒",
    "camera_angle": "鏡頭角度",
    "lighting": "燈光設定",
    "mood": "氛圍",
    "visual_style": "視覺風格",
    "timestamp_start": "00:00",
    "timestamp_end": "00:05"
  },
  {
    "paragraph_id": "2",
    "paragraph_content": "第二段歌詞內容",
    "scene_description": "第二個場景描述",
    "action_description": "第二個動作描述",
    "character_emotion": "第二個角色情緒",
    "camera_angle": "第二個鏡頭角度",
    "lighting": "第二個燈光設定",
    "mood": "第二個氛圍",
    "visual_style": "第二個視覺風格",
    "timestamp_start": "00:05",
    "timestamp_end": "00:10"
  }
]
```

#### Query Cache 結果（Task Response 的 return_message）
```json
[
  {
    "paragraph_id": "1",
    "paragraph_content": "...",
    "scene_description": "...",
    "action_description": "...",
    "...": "..."
  }
]
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `paragraph_id` | 段落編號 | string/number |
| `paragraph_content` | 段落歌詞內容 | string |
| `scene_description` | 場景描述 | string |
| `action_description` | 動作描述 | string |
| `character_emotion` | 角色情緒 | string |
| `camera_angle` | 鏡頭角度 | string |
| `lighting` | 燈光設定 | string |
| `mood` | 氛圍/情緒 | string |
| `visual_style` | 視覺風格 | string |
| `timestamp_start` | 開始時間 | string |
| `timestamp_end` | 結束時間 | string |

#### 顯示方式
- 以 Block 形式顯示每個段落
- 每個 Block 顯示為 key-value 結構

---

### 1.2 Storyboard Emotion

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": [...]
  },
  "query_function": "storyboard_emotion"
}
```

#### return_message 內容（陣列）
```json
[
  {
    "paragraph_id": "1",
    "paragraph_content": "歌詞內容",
    "emotion": "情緒分析",
    "emotion_intensity": "情緒強度（如：high, medium, low）",
    "mood": "氛圍",
    "character_emotion": "角色情緒",
    "emotional_arc": "情緒變化弧線",
    "emotional_keywords": ["關鍵詞1", "關鍵詞2"]
  },
  {
    "paragraph_id": "2",
    "emotion": "...",
    "emotion_intensity": "...",
    "mood": "...",
    // ... 其他欄位
  }
]
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `paragraph_id` | 段落編號 | string/number |
| `paragraph_content` | 歌詞內容 | string |
| `emotion` | 情緒分析 | string |
| `emotion_intensity` | 情緒強度 | string |
| `mood` | 氛圍 | string |
| `character_emotion` | 角色情緒 | string |
| `emotional_arc` | 情緒變化弧線 | string |
| `emotional_keywords` | 情緒關鍵詞 | array |

#### 顯示方式
- 以 Block 形式顯示每個段落
- 每個 Block 顯示為 key-value 結構

---

### 1.3 Storyboard Refiner

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": [...]
  },
  "query_function": "storyboard_refiner"
}
```

#### return_message 內容（陣列）
```json
[
  {
    "paragraph_id": "1",
    "refined_scene": "精煉後的場景描述",
    "refined_action": "精煉後的動作描述",
    "refined_visual": "精煉後的視覺描述",
    "camera_movement": "鏡頭運動",
    "composition": "構圖",
    "visual_effects": "視覺效果",
    "lighting_detail": "燈光細節",
    "color_palette": "色彩配置"
  },
  {
    "paragraph_id": "2",
    "refined_scene": "...",
    "refined_action": "...",
    // ... 其他欄位
  }
]
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `paragraph_id` | 段落編號 | string/number |
| `refined_scene` | 精煉後的場景描述 | string |
| `refined_action` | 精煉後的動作描述 | string |
| `refined_visual` | 精煉後的視覺描述 | string |
| `camera_movement` | 鏡頭運動 | string |
| `composition` | 構圖 | string |
| `visual_effects` | 視覺效果 | string |
| `lighting_detail` | 燈光細節 | string |
| `color_palette` | 色彩配置 | string |

#### 顯示方式
- 以 Block 形式顯示每個段落
- 每個 Block 顯示為 key-value 結構

---

## 2. Actor & Character 系列

### 2. Actor Setting

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": {...}
  },
  "query_function": "actor_setting"
}
```

#### return_message 內容（物件）
```json
{
  "good_designs": [
    "設計風格 1",
    "設計風格 2",
    "設計風格 3"
  ],
  "style_requirement": "整體風格要求描述",
  "user_region": "用戶地區/文化背景",
  "character_depiction": {
    "角色名稱1": {
      "appearance": "外觀描述",
      "personality": "性格描述",
      "age": "年齡",
      "gender": "性別",
      "clothing": "服裝描述",
      "special_features": "特殊特徵",
      "role": "角色定位"
    },
    "角色名稱2": {
      "appearance": "...",
      "personality": "...",
      "age": "...",
      "gender": "...",
      "clothing": "...",
      "special_features": "...",
      "role": "..."
    }
  }
}
```

#### Query Cache 結果（Task Response 的 return_message）
```json
{
  "good_designs": [...],
  "style_requirement": "...",
  "user_region": "...",
  "character_depiction": {...}
}
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `good_designs` | 推薦的設計風格列表 | array |
| `style_requirement` | 整體風格要求 | string |
| `user_region` | 用戶地區/文化背景 | string |
| `character_depiction` | 角色描述物件 | object |
| `character_depiction[角色名].appearance` | 外觀描述 | string |
| `character_depiction[角色名].personality` | 性格描述 | string |
| `character_depiction[角色名].age` | 年齡 | string |
| `character_depiction[角色名].gender` | 性別 | string |
| `character_depiction[角色名].clothing` | 服裝描述 | string |
| `character_depiction[角色名].special_features` | 特殊特徵 | string |
| `character_depiction[角色名].role` | 角色定位 | string |

#### 顯示方式
- 以卡片布局（Card Layout）顯示
- 分區顯示：Good Designs、Style Requirement、User Region、Character Depiction
- Character Depiction 以角色卡片形式展示

---

### 2.1 Character Generation

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": [...]
  },
  "query_function": "character_gen"
}
```

#### return_message 內容（陣列，新格式含 history）
```json
[
  {
    "name": "角色名稱1",
    "history": [
      {
        "url": "s3://bucket/path/to/image1.png",
        "timestamp": "2026-01-19T10:00:00",
        "prompt": "生成圖片的提示詞"
      },
      {
        "url": "s3://bucket/path/to/image2.png",
        "timestamp": "2026-01-19T10:05:00",
        "prompt": "重新生成的提示詞"
      },
      {
        "url": "s3://bucket/path/to/image3.png",
        "timestamp": "2026-01-19T10:10:00",
        "prompt": "第三次生成的提示詞"
      }
    ],
    "selected_idx": 1
  },
  {
    "name": "角色名稱2",
    "history": [
      {
        "url": "s3://bucket/path/to/character2_v1.png",
        "timestamp": "2026-01-19T10:10:00",
        "prompt": "角色2的提示詞"
      }
    ],
    "selected_idx": 0
  }
]
```

#### 舊格式（無 history，僅供參考）
```json
[
  {
    "name": "角色名稱",
    "url": "s3://bucket/path/to/image.png"
  }
]
```

#### Query Cache 結果（Task Response 的 return_message）
```json
[
  {
    "name": "角色名稱",
    "history": [
      {
        "url": "...",
        "timestamp": "...",
        "prompt": "..."
      }
    ],
    "selected_idx": 0
  }
]
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `name` | 角色名稱 | string |
| `history` | 歷史生成記錄 | array |
| `history[].url` | 圖片 S3 URL | string |
| `history[].timestamp` | 生成時間戳 | string (ISO 8601) |
| `history[].prompt` | 生成提示詞 | string |
| `selected_idx` | 當前選中的圖片索引 | number |

#### 顯示方式
- 以按鈕列表形式顯示
- 每個角色一個按鈕，顯示角色名稱和歷史圖片數量
- 點擊按鈕開啟歷史圖片選擇模態框
- 可選擇不同版本的角色圖片
- 提供重新生成（Regenerate）按鈕

---

## 3. Video Generation 系列

### 3. Start Frame Generation

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": {...}
  },
  "query_function": "start_frame_gen"
}
```

#### return_message 內容（物件）
```json
{
  "para_1": {
    "url": "s3://bucket/path/to/frame1.png",
    "prompt": "生成起始幀的提示詞",
    "timestamp": "2026-01-19T10:00:00"
  },
  "para_2": {
    "url": "s3://bucket/path/to/frame2.png",
    "prompt": "第二段的提示詞",
    "timestamp": "2026-01-19T10:01:00"
  },
  "para_3": {
    "url": "s3://bucket/path/to/frame3.png",
    "prompt": "第三段的提示詞",
    "timestamp": "2026-01-19T10:02:00"
  }
}
```

#### Query Cache 結果（Task Response 的 return_message）
```json
{
  "para_1": {
    "url": "...",
    "prompt": "...",
    "timestamp": "..."
  },
  "para_2": {...},
  "para_3": {...}
}
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `para_N` | 段落編號（N 為數字） | object |
| `para_N.url` | 起始幀圖片 S3 URL | string |
| `para_N.prompt` | 生成提示詞 | string |
| `para_N.timestamp` | 生成時間戳 | string (ISO 8601) |

#### 顯示方式
- 以可點擊的段落列表形式顯示
- 每個段落顯示編號和預覽縮圖
- 點擊後在預覽面板顯示完整圖片
- 提供「Confirm Paragraph Selection」按鈕進行選擇確認

---

### 4. Video Gen / Video Gen Unified

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": {...}
  },
  "query_function": "video_gen_unified" | "video_gen"
}
```

#### return_message 內容（物件）
```json
{
  "video_gen_count": 3,
  "lip_sync_count": 0,
  "para_1": {
    "url": "s3://bucket/path/to/video1.mp4",
    "start_frame_url": "s3://bucket/path/to/frame1.png",
    "duration": 5.0,
    "timestamp": "2026-01-19T10:00:00"
  },
  "para_2": {
    "url": "s3://bucket/path/to/video2.mp4",
    "start_frame_url": "s3://bucket/path/to/frame2.png",
    "duration": 4.5,
    "timestamp": "2026-01-19T10:01:00"
  },
  "para_3": {
    "url": "s3://bucket/path/to/video3.mp4",
    "start_frame_url": "s3://bucket/path/to/frame3.png",
    "duration": 6.0,
    "timestamp": "2026-01-19T10:02:00"
  }
}
```

#### Query Cache 結果（Task Response 的 return_message）
```json
{
  "video_gen_count": 3,
  "lip_sync_count": 0,
  "para_1": {...},
  "para_2": {...},
  "para_3": {...}
}
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `video_gen_count` | Video Gen 段落數量 | number |
| `lip_sync_count` | Lip Sync 段落數量 | number |
| `para_N` | 段落編號（N 為數字） | object |
| `para_N.url` | 視頻 S3 URL | string |
| `para_N.start_frame_url` | 起始幀圖片 URL | string |
| `para_N.duration` | 視頻時長（秒） | number |
| `para_N.timestamp` | 生成時間戳 | string (ISO 8601) |

#### 顯示方式
- 以可點擊的段落列表形式顯示（視頻模式）
- 每個段落顯示編號和視頻縮圖
- 點擊後在預覽面板播放視頻
- 提供「Confirm Video Paragraph Selection」按鈕進行選擇確認

---

### Lip Sync（video_gen_unified 的子功能）

#### return_message 內容（物件）
```json
{
  "video_gen_count": 0,
  "lip_sync_count": 3,
  "para_1": {
    "url": "s3://bucket/path/to/lipsync_video1.mp4",
    "audio_url": "s3://bucket/path/to/audio1.mp3",
    "duration": 5.0,
    "timestamp": "2026-01-19T10:00:00"
  },
  "para_2": {
    "url": "s3://bucket/path/to/lipsync_video2.mp4",
    "audio_url": "s3://bucket/path/to/audio2.mp3",
    "duration": 4.5,
    "timestamp": "2026-01-19T10:01:00"
  }
}
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `video_gen_count` | Video Gen 段落數量 | number |
| `lip_sync_count` | Lip Sync 段落數量 | number |
| `para_N` | 段落編號（N 為數字） | object |
| `para_N.url` | Lip Sync 視頻 S3 URL | string |
| `para_N.audio_url` | 音頻檔案 S3 URL | string |
| `para_N.duration` | 視頻時長（秒） | number |
| `para_N.timestamp` | 生成時間戳 | string (ISO 8601) |

---

### 5. Video Merge

#### 完整回傳格式
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": {...}
  },
  "query_function": "video_merge"
}
```

#### return_message 內容（物件）
```json
{
  "url": "s3://bucket/path/to/merged_video.mp4",
  "duration": 25.5,
  "resolution": "1920x1080",
  "fps": 30,
  "file_size": 15728640,
  "timestamp": "2026-01-19T10:05:00",
  "merged_segments": [
    {
      "segment_id": "para_1",
      "duration": 5.0
    },
    {
      "segment_id": "para_2",
      "duration": 4.5
    },
    {
      "segment_id": "para_3",
      "duration": 6.0
    }
  ]
}
```

#### Query Cache 結果（Task Response 的 return_message）
```json
{
  "url": "...",
  "duration": 25.5,
  "resolution": "...",
  "fps": 30,
  "...": "..."
}
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `url` | 合併後視頻的 S3 URL | string |
| `duration` | 總時長（秒） | number |
| `resolution` | 分辨率 | string |
| `fps` | 幀率 | number |
| `file_size` | 檔案大小（bytes） | number |
| `timestamp` | 生成時間戳 | string (ISO 8601) |
| `merged_segments` | 合併的片段資訊 | array |
| `merged_segments[].segment_id` | 片段ID | string |
| `merged_segments[].duration` | 片段時長 | number |

#### 顯示方式
- 單個視頻預覽
- 顯示視頻資訊（時長、分辨率、檔案大小等）
- 提供視頻播放器

---

## 4. Cache Management

### View Cache JSON

#### API 請求
```
GET /api/get_cache_json?music_name={music_name}
```

#### 初始回傳
```json
{
  "success": true,
  "job_id": "job-id",
  "session_id": "session-id",
  "message": "獲取 ProjectCache 任務已提交"
}
```

#### Task Status 回傳
```json
{
  "status": "success",
  "job_id": "job-id",
  "response_data": {
    "return_message": {
      "session_id": "session-abc123",
      "cache_keys": [
        "storyboard_initial",
        "storyboard_emotion",
        "storyboard_refiner",
        "actor_setting",
        "character_gen",
        "start_frame_gen",
        "video_gen_unified",
        "video_merge"
      ],
      "cache_data": {
        "storyboard_initial": [...],
        "storyboard_emotion": [...],
        "storyboard_refiner": [...],
        "actor_setting": {...},
        "character_gen": [...],
        "start_frame_gen": {...},
        "video_gen_unified": {...},
        "video_merge": {...}
      }
    }
  }
}
```

#### return_message 詳細結構
```json
{
  "session_id": "當前 session ID",
  "cache_keys": [
    "已快取的功能名稱列表"
  ],
  "cache_data": {
    "storyboard_initial": [
      {
        "paragraph_id": "1",
        "scene_description": "...",
        // ... storyboard_initial 的完整資料
      }
    ],
    "storyboard_emotion": [
      {
        "paragraph_id": "1",
        "emotion": "...",
        // ... storyboard_emotion 的完整資料
      }
    ],
    "storyboard_refiner": [
      {
        "paragraph_id": "1",
        "refined_scene": "...",
        // ... storyboard_refiner 的完整資料
      }
    ],
    "actor_setting": {
      "good_designs": [...],
      "style_requirement": "...",
      "user_region": "...",
      "character_depiction": {...}
    },
    "character_gen": [
      {
        "name": "角色名稱",
        "history": [...],
        "selected_idx": 0
      }
    ],
    "start_frame_gen": {
      "para_1": {
        "url": "...",
        "prompt": "...",
        "timestamp": "..."
      },
      "para_2": {...}
    },
    "video_gen_unified": {
      "video_gen_count": 3,
      "lip_sync_count": 0,
      "para_1": {...},
      "para_2": {...}
    },
    "video_merge": {
      "url": "...",
      "duration": 25.5,
      // ... 其他欄位
    }
  }
}
```

#### 欄位說明
| Key | 說明 | 類型 |
|-----|------|------|
| `session_id` | 當前 session ID | string |
| `cache_keys` | 已快取的功能名稱列表 | array |
| `cache_data` | 所有快取資料 | object |
| `cache_data[功能名]` | 該功能的完整快取資料 | any |

#### 顯示方式
- 顯示 Session ID
- 顯示快取統計（cache keys 數量、列表）
- 可折疊的完整 JSON 顯示
- 每個 cache key 的詳細資訊（可折疊）

---

## 5. 資料類型對照表

| 功能 ID | 功能名稱 | 後端函數名 | return_message 類型 | 顯示方式 |
|---------|---------|-----------|-------------------|---------|
| 1.1 | Storyboard Initial | storyboard_initial | Array | Blocks |
| 1.2 | Storyboard Emotion | storyboard_emotion | Array | Blocks |
| 1.3 | Storyboard Refiner | storyboard_refiner | Array | Blocks |
| 2 | Actor Setting | actor_setting | Object | Card Layout |
| 2.1 | Character Generation | character_gen | Array | Button List |
| 3 | Start Frame Generation | start_frame_gen | Object | Paragraph List (Image) |
| 4 | Video Gen | video_gen_unified | Object | Paragraph List (Video) |
| 5 | Video Merge | video_merge | Object | Single Video Preview |
| - | View Cache JSON | - | Object | Collapsible Sections |

---

## 6. API 端點清單

### GUI Index
```
GET /
```
回傳 HTML 頁面。

### Session ID
```
GET /api/session_id
Response: {
  "session_id": "session-id"
}
```

### Get Cache JSON
```
GET /api/get_cache_json?music_name={music_name}
Response: {
  "success": true,
  "job_id": "job-id",
  "session_id": "session-id",
  "message": "獲取 ProjectCache 任務已提交"
}
```

### Submit Task
```
POST /api/submit_task
Body: {
  "music_name": "歌曲名稱",
  "use_vocal": true/false,
  "start_num": 1,
  "end_num": 5,
  "functions": ["1.1", "1.3", "2", "2.1", "3", "4", "5"],  // 前端按鈕 ID
  "version_indices": {  // 可選：指定使用的版本
    "storyboard_initial": 0,
    "storyboard_refiner": 1,
    "actor_setting": 0,
    "character_gen": 0
  },
  "character_selections": {  // 可選：角色選擇映射
    "角色名稱1": 0,
    "角色名稱2": 1
  },
  "paragraph_selections": {  // 可選：段落選擇映射
    "1": 0,
    "2": 1,
    "3": 0
  }
}
Response: {
  "success": true,
  "job_id": "job-id",
  "task_id": "task-id",  // 基於任務內容的邏輯 ID
  "session_id": "session-id",
  "functions": ["storyboard_initial", "actor_setting"],  // 轉換後的函數名稱
  "function_ids": ["1.1", "2"],  // 原始按鈕 ID
  "payload": {  // 完整的 payload 結構
    "session_id": "session-id",
    "effect": "automv",
    "acts": [{
      "id": 0,
      "params": {
        "musicName": "歌曲名稱",
        "useVocal": true,
        "startNum": 1,
        "endNum": 5,
        "functions": ["storyboard_initial", "actor_setting"],
        "versionIndices": {
          "storyboard_initial": 0
        },
        "character_selections": {
          "角色名稱1": 0
        },
        "paragraph_selections": {
          "1": 0
        }
      }
    }]
  },
  "message": "任務已提交"
}
```

### Query Cache
```
POST /api/query_cache
Body: {
  "function_id": "1.1",  // 也可用 function name，character_gen 不支援
  "music_name": "歌曲名稱"
}
Response: {
  "success": true,
  "job_id": "job-id",
  "session_id": "session-id",
  "query_function": "storyboard_initial",
  "message": "查詢任務已提交"
}
```

### Clear Cache
```
POST /api/clear_cache
Body: {
  "function_name": "storyboard_initial",
  "music_name": "歌曲名稱"
}
Response: {
  "success": true,
  "job_id": "job-id",
  "function_name": "storyboard_initial",
  "music_name": "歌曲名稱",
  "cache_keys": ["storyboard_initial"],
  "message": "Cache clear task submitted for storyboard_initial"
}
```

### Regenerate Character
```
POST /api/regenerate_character
Body: {
  "music_name": "歌曲名稱",
  "character_name": "角色名",
  "version_indices": { "character_gen": 0 }
}
Response: {
  "success": true,
  "job_id": "job-id",
  "task_id": "task-id",
  "session_id": "session-id",
  "character_name": "角色名",
  "message": "Character regeneration task submitted"
}
```

### Regenerate Paragraph Image
```
POST /api/regenerate_paragraph
Body: {
  "music_name": "歌曲名稱",
  "paragraph_number": 3,
  "version_indices": { "start_frame_gen": 0 },
  "character_selections": { "角色名": 0 }
}
Response: {
  "success": true,
  "job_id": "job-id",
  "task_id": "task-id",
  "session_id": "session-id",
  "paragraph_number": 3,
  "message": "Paragraph regeneration task submitted"
}
```

### Regenerate Video Paragraph
```
POST /api/regenerate_video_paragraph
Body: {
  "music_name": "歌曲名稱",
  "paragraph_number": 3,
  "version_indices": { "video_gen_unified": 0 },
  "character_selections": { "角色名": 0 },
  "paragraph_selections": { "3": 0 }
}
Response: {
  "success": true,
  "job_id": "job-id",
  "task_id": "task-id",
  "session_id": "session-id",
  "paragraph_number": 3,
  "message": "Video paragraph regeneration task submitted"
}
```

### Select Character Image (Batch)
```
POST /api/select_character_image
Body: {
  "music_name": "歌曲名稱",
  "cast": ["角色A", "角色B"],
  "idx": [0, 1]
}
Response: {
  "success": true,
  "job_id": "job-id",
  "task_id": "task-id",
  "session_id": "session-id",
  "cast": ["角色A", "角色B"],
  "idx": [0, 1],
  "message": "Batch character image selection task submitted for 2 characters"
}
```

### Select Paragraph Image (Batch)
```
POST /api/select_paragraph_image
Body: {
  "music_name": "歌曲名稱",
  "paragraphs": [1, 2, 3],
  "idx": [0, 1, 0]
}
Response: {
  "success": true,
  "job_id": "job-id",
  "task_id": "task-id",
  "session_id": "session-id",
  "paragraphs": [1, 2, 3],
  "idx": [0, 1, 0],
  "message": "Paragraph image selection task submitted"
}
```

### Select Video Paragraph (Batch)
```
POST /api/select_video_paragraph
Body: {
  "music_name": "歌曲名稱",
  "paragraphs": [1, 2, 3],
  "idx": [0, 1, 0]
}
Response: {
  "success": true,
  "job_id": "job-id",
  "task_id": "task-id",
  "session_id": "session-id",
  "paragraphs": [1, 2, 3],
  "idx": [0, 1, 0],
  "message": "Video paragraph selection task submitted"
}
```

### Task Status
```
GET /api/task_status/{job_id}
```

### Task Logs
```
GET /api/task_logs/{job_id}
Response: {
  "logs": [
    {
      "time": "HH:MM:SS",
      "message": "日誌內容",
      "type": "info"
    }
  ]
}
```

### Task Response
```
GET /api/task_response/{job_id}
Response: {
  "response_data": {
    "return_message": { ... }
  }
}
```

---

## 7. ProjectCache 資料結構

ProjectCache 是 worker 端用於存儲整個專案數據的核心結構，使用 S3 進行持久化存儲。

### ProjectCache 頂層結構

```python
@dataclass
class ProjectCache:
    project_id: str  # 專案 ID（通常是 music_name）
    songformer_result: Optional[List[Dict]] = None  # SongFormer 音樂結構分析結果
    whisper_result: Optional[List[str]] = None  # Whisper 語音識別結果
    storyboard_emo: Optional[List[InitialStoryboard]] = None  # 初始故事板列表（支援多版本）
    next_layer_ver: int = 0  # 下一層使用的版本索引
```

### InitialStoryboard 結構

```python
@dataclass
class InitialStoryboard:
    version_idx: int = 0  # 版本索引
    emotion_analysis: Optional[List[Dict]] = None  # 情感分析結果
    paragraph_arrangement: Optional[List[Dict]] = None  # 段落安排結果
    refined_storyboard: Optional[List[RefinedStoryboard]] = None  # 精煉故事板列表（支援多版本）
    next_layer_ver: int = 0  # 下一層使用的版本索引
```

### RefinedStoryboard 結構

```python
@dataclass
class RefinedStoryboard:
    version_idx: int = 0  # 版本索引
    refined_storyboard: Optional[List[Dict]] = None  # 精煉故事板資料
    env_setting: Optional[List[EnvSetting]] = None  # 環境設定列表（支援多版本）
    next_layer_ver: int = 0  # 下一層使用的版本索引
```

### EnvSetting 結構

```python
@dataclass
class EnvSetting:
    version_idx: int = 0  # 版本索引
    setting_desc: Optional[Dict] = None  # Actor Setting 描述
    all_actors: Optional[ALLActors] = None  # 所有角色資料
    next_layer_ver: int = 0  # 下一層使用的版本索引
```

### ALLActors 結構

```python
@dataclass
class ALLActors:
    version_idx: int = 0  # 版本索引
    actors: Optional[List[Actor]] = None  # 角色列表
    all_paragraphs: Optional[Dict[str, Paragraph]] = None  # 所有段落資料，key 為段落號
    next_layer_ver: int = 0  # 下一層使用的版本索引
```

### Actor 結構

```python
@dataclass
class Actor:
    name: str  # 角色名稱
    history: List[ActorHistory] = field(default_factory=list)  # 角色歷史記錄（支援多版本生成）
    next_layer_ver: int = 0  # 當前選中的版本索引
```

### ActorHistory 結構

```python
@dataclass
class ActorHistory:
    image_url: Optional[str] = None  # 角色圖片 S3 URL
    description: Optional[str] = None  # 角色描述
    timestamp: Optional[str] = None  # 生成時間戳
```

### Paragraph 結構

```python
@dataclass
class Paragraph:
    paragraph_number: str  # 段落編號
    history: List[ParagraphHistory] = field(default_factory=list)  # 段落歷史記錄（支援多版本生成）
    video_history: List[VideoHistory] = field(default_factory=list)  # 視頻歷史記錄（支援多版本生成）
    next_layer_ver: int = 0  # 當前選中的起始幀版本索引
    next_video_ver: int = 0  # 當前選中的視頻版本索引
```

### ParagraphHistory 結構

```python
@dataclass
class ParagraphHistory:
    pic: Optional[str] = None  # 起始幀圖片 S3 URL
    cam: Optional[List] = None  # 相機腳本
    timestamp: Optional[str] = None  # 生成時間戳
```

### VideoHistory 結構

```python
@dataclass
class VideoHistory:
    url: Optional[str] = None  # 視頻 S3 URL
    duration: Optional[float] = None  # 視頻時長
    start_frame_url: Optional[str] = None  # 起始幀圖片 URL
    timestamp: Optional[str] = None  # 生成時間戳
```

### 版本管理機制

ProjectCache 使用樹狀結構管理不同版本的資料：

1. **頂層版本（ProjectCache.storyboard_emo）**：支援多個 InitialStoryboard 版本
2. **第二層版本（InitialStoryboard.refined_storyboard）**：支援多個 RefinedStoryboard 版本
3. **第三層版本（RefinedStoryboard.env_setting）**：支援多個 EnvSetting 版本
4. **第四層版本（ALLActors）**：
   - 每個 Actor 支援多個 ActorHistory（圖片生成歷史）
   - 每個 Paragraph 支援多個 ParagraphHistory（起始幀歷史）和 VideoHistory（視頻歷史）

#### versionIndices 參數格式

前端透過 `versionIndices` 參數指定使用哪個版本：

```json
{
  "versionIndices": {
    "storyboard_initial": 0,  // 使用第 0 個 InitialStoryboard
    "storyboard_refiner": 1,  // 使用第 1 個 RefinedStoryboard
    "actor_setting": 0,       // 使用第 0 個 EnvSetting
    "character_gen": 0        // 使用第 0 個 ALLActors
  }
}
```

### ProjectCache 存儲位置

- **S3 路徑格式**：`s3://bucket/project_cache/{music_name}.json`
- **本地路徑格式**：`/workspace/result/{music_name}/project_cache.json`

### 資料存取 API

#### 讀取 ProjectCache
```python
from Engine.tools.cache_structure import S3ProjectCacheStore
project_cache = S3ProjectCacheStore.load(job)
```

#### 保存 ProjectCache
```python
S3ProjectCacheStore.save(job, project_cache)
```

#### 更新 ProjectCache（原子操作）
```python
def update_fn(cache: Optional[ProjectCache]) -> ProjectCache:
    if cache is None:
        cache = ProjectCache(project_id=music_name)
    # 修改 cache...
    return cache

project_cache = S3ProjectCacheStore.update(job, update_fn)
```

---

## 8. 注意事項

1. **資料格式可能變動**：實際的 key 名稱會根據後端實作而有所不同，建議執行後用瀏覽器 Console 或 View Cache JSON 功能查看實際結構。

2. **Query Cache 為非同步**：Query Cache 只回傳 job_id，需透過 `/api/task_status/{job_id}` 或 `/api/task_response/{job_id}` 取得結果。

3. **JSON 字串解析**：某些回傳資料可能以 JSON 字串形式返回，前端會自動解析。

4. **S3 URL 格式**：所有圖片/視頻 URL 都使用 S3 URI 格式（`s3://bucket/path/to/file`）。

5. **時間戳格式**：所有時間戳使用 ISO 8601 格式（`YYYY-MM-DDTHH:mm:ss`）。

6. **段落編號**：段落編號格式為 `para_N`，其中 N 為數字（從 1 開始）。

7. **History 功能**：Character Generation 和 Start Frame Generation 支援歷史記錄功能，可保存多個版本的生成結果。

8. **Video Type 判斷**：`video_gen_unified` 會根據 `video_gen_count` 和 `lip_sync_count` 自動判斷使用 video_gen 或 lip_sync 格式。

9. **版本索引機制**：
   - `version_idx`：當前版本的索引號
   - `next_layer_ver`：指向下一層應使用的版本索引
   - 前端透過 `versionIndices` 參數指定使用哪個版本

10. **ProjectCache 持久化**：所有資料都會自動保存到 S3，確保資料不會丟失。

---

## 9. 版本記錄

- **v1.1** (2026-01-28): 新增 Payload 結構說明和 ProjectCache 詳細文件
- **v1.0** (2026-01-19): 初始版本，包含所有主要功能的資料結構文件
