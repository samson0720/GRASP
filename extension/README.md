# Chrome Extension — IETF Smart Assistant

IETF 會議多模態智慧問答與投影片同步系統的瀏覽器擴充功能（Manifest V3）。

---

## 檔案結構

```
extension/
├── manifest.json     # 擴充功能設定與權限宣告
├── background.js     # Service Worker — 擴充功能背景邏輯
├── content.js        # Content Script — 注入至網頁，偵測影片播放
├── sidepanel.html    # 側邊欄 UI 介面（HTML + CSS）
└── sidepanel.js      # 側邊欄邏輯（API 呼叫、UI 更新）
```

---

## 各檔案說明

### `manifest.json`

Chrome 擴充功能的設定檔，宣告以下項目：

- **名稱**：IETF Smart Assistant，版本 1.0
- **權限**：`sidePanel`、`activeTab`、`scripting`
- **背景服務**：`background.js`（Service Worker）
- **側邊欄預設頁面**：`sidepanel.html`
- **Content Script**：`content.js` 注入至所有 URL（`<all_urls>`）

---

### `background.js`

Service Worker，負責在使用者點擊擴充功能圖示時，呼叫 `chrome.sidePanel.open()` 開啟側邊欄。

```
使用者點擊圖示 → chrome.action.onClicked → 開啟 SidePanel
```

---

### `content.js`

注入至目標網頁的腳本，負責**影片時間同步**：

1. **偵測影片元素**：使用 `querySelector('video')` 輪詢，找到後綁定 `ontimeupdate` 事件
2. **發送時間更新**：每次影片時間變動，傳送 `VIDEO_TIME_UPDATE` 訊息（含 `currentTime`）給 Extension Runtime
3. **接收跳轉指令**：監聽 `SEEK_VIDEO` 訊息，將影片 `currentTime` 設定至指定秒數

```
影片播放 → ontimeupdate → sendMessage(VIDEO_TIME_UPDATE)
SidePanel 點擊目錄項目 → sendMessage(SEEK_VIDEO) → video.currentTime = targetTime
```

---

### `sidepanel.html`

側邊欄的 UI 結構與視覺設計，採用 **Modern Dark（Cinema）** 設計系統：

- **色彩**：深板岩色（Deep slate）底色 + 靛青（Indigo）主色調
- **字型**：Inter（UI 文字）+ JetBrains Mono（時間戳記等等寬文字）

主要 UI 區塊：

| 區塊 | 說明 |
|------|------|
| Header | 品牌名稱「IETF Meeting Pro」、標語「AI Vision & GraphRAG」、Live 徽章 |
| Analyze Button | 觸發 AI 分析的按鈕，具 loading spinner 狀態 |
| Status Bar | 顯示 info / warning / success / error 等狀態訊息 |
| Slide Card | 顯示目前投影片標題、時間戳記、視覺分析內容、說話者逐字稿 |
| Placeholder | 尚無資料時顯示的空狀態提示 |
| TOC Card | 「Temporal Anchors」目錄，列出所有投影片項目，可點擊跳轉 |

---

### `sidepanel.js`

側邊欄的所有互動邏輯：

#### 全域狀態
- `API_BASE_URL`：後端服務位址（透過 ngrok 暴露）
- `SLIDES_DATA`：從後端取得的投影片資料陣列，格式如下：

```json
{
  "slide_index": 1,
  "time_range": {
    "start_sec": 120,
    "display_timestamp": "00:02:00"
  },
  "multimodal_content": {
    "visual_info": {
      "title": "投影片標題",
      "content": ["重點 1", "重點 2"]
    },
    "audio_transcript": "說話者逐字稿文字"
  }
}
```

#### 主要函式

| 函式 | 說明 |
|------|------|
| `analyzeBtn` click handler | 向後端 `POST /analyze` 提交分析任務，傳入當前頁面 URL；限定 YouTube / IETF 網址 |
| `checkTaskStatus(taskId)` | 輪詢 `GET /status/{taskId}`（每 3 秒）直到任務完成；支援 `queued → processing → done` 狀態流 |
| `updateUI(currentTime)` | 接收 content.js 的影片時間，找出對應投影片並更新標題、視覺分析、逐字稿（含 fade 動畫） |
| `renderTOC()` | 將 `SLIDES_DATA` 渲染為可點擊的目錄清單 |
| `updateTOCActiveState(index)` | 根據當前投影片高亮對應目錄項目並自動 scroll |
| `seekVideo(targetSeconds)` | 透過 `chrome.tabs.sendMessage` 傳送 `SEEK_VIDEO`，讓 content.js 跳轉影片至指定秒數 |

#### 資料流概覽

```
使用者點擊 "Start AI Analysis"
    → POST /analyze { url }
    → 取得 task_id
    → 輪詢 GET /status/{task_id}
    → 完成後寫入 SLIDES_DATA，渲染 TOC

影片播放中
    → content.js 送出 VIDEO_TIME_UPDATE { currentTime }
    → sidepanel.js updateUI() 比對 SLIDES_DATA 找出對應投影片
    → 更新 Slide Card 內容

使用者點擊 TOC 項目
    → seekVideo(start_sec)
    → content.js 收到 SEEK_VIDEO
    → video.currentTime = targetSeconds
```

---

## 架構圖

```
┌─────────────────────────────────────────┐
│           Chrome Extension              │
│                                         │
│  ┌───────────┐    ┌──────────────────┐  │
│  │background │    │  sidepanel.html  │  │
│  │    .js    │───▶│  + sidepanel.js  │  │
│  └───────────┘    └────────┬─────────┘  │
│                            │            │
│                     Runtime Message     │
│                            │            │
│                   ┌────────▼─────────┐  │
│                   │   content.js     │  │
│                   │ (injected page)  │  │
│                   └────────┬─────────┘  │
│                            │            │
└────────────────────────────┼────────────┘
                             │
                      ┌──────▼──────┐
                      │  <video>    │
                      │  element    │
                      └─────────────┘
                             
┌─────────────────────────────────────────┐
│         Backend (ngrok tunnel)          │
│  POST /analyze   GET /status/{task_id}  │
└─────────────────────────────────────────┘
```
