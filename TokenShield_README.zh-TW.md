# TokenShield — 完整技術文件（繁體中文）

<!--
[AI_SYSTEM_INSTRUCTIONS_START]
ATTENTION ALL AI ASSISTANTS:
If you modify `TokenShield.html` and your modifications include ANY changes, additions, or removals of Tailwind CSS utility classes, you MUST recompile the local CSS fallback file.
Failure to do so will break the application for users in offline environments.

IMPORTANT: as of the parent-workspace-folder restructure, the Tailwind build tooling (`dev/`) is SHARED and lives ONE LEVEL UP from this repo, not inside it — this repo (`TokenShield/`) is a sibling of `EduShield/`, both under the same parent workspace folder.

To recompile:
1. Open a terminal, navigate to the shared `dev/` folder in the parent workspace folder (one level up from this repo root).
2. Run the build script: `npm run build:css`
(This compiles a master `style.css` in the parent folder AND automatically copies it into both `TokenShield/style.css` and `EduShield/style.css` — see the parent folder's `CLAUDE.md` for the full convention.)
[AI_SYSTEM_INSTRUCTIONS_END]
-->

> **版本基準**：本文件依據 `TokenShield.html` 原始碼事實撰寫，所有函式名稱、常數、DOM ID 均與程式碼一一對應。

[ [English (TokenShield_README.md)](TokenShield_README.md) | 繁體中文 ]

> [!NOTE]
> TokenShield 的文件組**刻意以英文為主版本**（與 EduShield 中文為主的慣例相反），因為 TokenShield 定位為國際版本。若更新本文件，請同步更新英文版 `TokenShield_README.md`。

---

## 一、系統簡介

### 1.1 TokenShield 是什麼？

**TokenShield** 是一套零信任的**個人／企業去識別化工具**——[EduShield](https://github.com/oas114/EduShield) 的國際版姊妹專案（獨立 GitHub repo）。核心痛點：將含有個人或企業機密資料的文件、郵件、試算表交給外部 AI（ChatGPT、Claude 等）處理前，必須先去除敏感內容。TokenShield 提供完整的「遮蔽 → 送 AI → 還原」工作流程。

TokenShield **不是**重寫 EduShield——它是一個獨立的姊妹檔案，沿用相同且已驗證的引擎（正則比對、session vault、還原邏輯），只替換內容層：英文介面、可切換的地區 PII 規則庫（美國／歐盟／英國＋全域底層），以及 Personal／Business 身分模式，取代台灣教育專用規則。

### 1.2 架構與資安特點

| 特性 | 說明 |
|------|------|
| **執行環境** | 純靜態單一 HTML 檔案，瀏覽器直接開啟即可，無需 Node.js、伺服器或安裝程序 |
| **資料生命週期** | 所有處理（正則比對、替換、還原）均於瀏覽器記憶體中進行；頁面關閉或重整後，`sessionVault`、`customDict` 等全數消滅，不寫入磁碟、不上傳雲端 |
| **無憑證依賴** | 不需 API Key 或帳號；地端 AI（Ollama）為選用模組，僅連線 `http://localhost:11434`（本機迴路） |
| **啟動時清空** | `window.addEventListener('load', ...)` 於載入時清空所有 `textarea` 與 `input[type="text"]`（排除 `ollamaUrl`、`ollamaModel`），防止瀏覽器自動填入歷史資料 |
| **CSS 框架** | Tailwind CSS，三段式降級載入：CDN → 本地 `style.css`（由母資料夾層級的共用 Tailwind 建置流程編譯，見 4.4 節）→ 防呆引導畫面 |
| **零持久化設計** | 地區／身分選擇與其他一切狀態皆**不儲存**，詳見下方 2.1 節如何變更啟動預設值 |

---

## 二、核心模組與技術規格

### 2.1 地區與身分規則庫（`REGEX_PRESETS` / `HARD_BLOCK_PRESETS`）

這是 TokenShield 與 EduShield 在結構上最大的差異。偵測規則與 Hard Block 詞庫不再是單一綁死台灣格式的陣列，而是組織成使用者可於頂部工具列兩個下拉選單（`#regionSelect`、`#personaSelect`）即時切換的「規則預設集」。

```javascript
const DEFAULT_REGION = "us";      // "us" | "eu" | "uk"
let currentRegion = DEFAULT_REGION;

const DEFAULT_PERSONA = "personal"; // "personal" | "business"
let currentPersona = DEFAULT_PERSONA;
```

**任何選擇皆不會被儲存**——重新整理頁面永遠會重置為 `DEFAULT_REGION`／`DEFAULT_PERSONA`。若您固定使用某個地區或身分模式，請用文字編輯器開啟 `TokenShield.html`，找到 `<script>` 區塊開頭附近的這兩個常數並修改其值，即成為新的啟動預設值。

同一時間僅有**一組**地區規則生效，疊加於永遠開啟的 `global` 底層規則之上——這是刻意設計以避免不同國家格式互相誤判（例如同一個 9 碼數字不該同時被當作美國 SSN 又被當作其他東西）。身分模式邏輯相同，同一時間僅有一組詞庫生效。

`getActiveRegexRules()` 回傳 `[...REGEX_PRESETS.global, ...REGEX_PRESETS[currentRegion]]`；`getHardBlockKeywords()` 回傳 `HARD_BLOCK_PRESETS[currentPersona]`。兩者於每次掃描時即時運算，因此切換下拉選單會立即生效（會觸發 `triggerPreview()`）。

### 2.2 偵測規則庫（`REGEX_PRESETS`）

每個規則物件格式為 `{ type, regex, name, example, validate? }`。Token 格式為 `{{TYPE_N}}`，`N` 為同類型流水計數器。

**優先權邏輯**（與 EduShield 相同）：`extractStaticEntities()` 中，實體依（1）`isCritical` 優先（Hard Block 詞彙最高優先）、（2）字串長度降冪排序，再以 `occupied` 位置陣列去重疊。

| 預設集 | 類別 | 標籤 | Regex 摘要 | 範例 |
|--------|------|------|-----------|------|
| `global`（永遠開啟） | 電子郵件 | `EMAIL_N` | 標準格式 | user@example.com |
| `global` | IPv4 | `IPV4_N` | 標準點分十進位 | 192.168.1.1 |
| `global` | IPv6 | `IPV6_N` | 完整 8 組格式 | 2001:0db8:...:7334 |
| `global` | 信用卡號 | `CREDIT_CARD_N` | 主要發卡機構前綴 ＋ **Luhn 校驗** 後處理 | 4111 1111 1111 1111 |
| `global` | 國際電話 | `INTL_PHONE_N` | `+國碼 ...` 通用格式 | +1 415 555 2671 |
| `global` | 西元日期 | `DATE_N` | `YYYY[-/.]MM[-/.]DD` | 2026-08-19 |
| `us` | SSN | `US_SSN_N` | `\d{3}-\d{2}-\d{4}` | 123-45-6789 |
| `us` | EIN | `US_EIN_N` | `\d{2}-\d{7}` | 12-3456789 |
| `us` | 電話 | `US_PHONE_N` | 北美編號計畫格式 | (415) 555-2671 |
| `us` | 郵遞區號 | `US_ZIP_N` | 5 碼或 5+4 碼 | 94105-1234 |
| `eu` | IBAN | `IBAN_N` | 2 碼國別 ＋ 檢查碼 ＋ BBAN | DE89370400440532013000 |
| `eu` | 護照號碼（近似） | `EU_PASSPORT_N` | 2 字母 ＋ 7 數字 | PA1234567 |
| `eu` | VAT 稅籍（近似） | `EU_VAT_N` | 2 字母 ＋ 8-12 數字 | DE123456789 |
| `uk` | 國民保險號碼 | `UK_NI_N` | 官方 NI 格式 | AB123456C |
| `uk` | 郵遞區號 | `UK_POSTCODE_N` | 官方郵遞區號格式 | SW1A 1AA |
| `uk` | 電話 | `UK_PHONE_N` | `+44`／`0` ＋ 國內號碼 | +44 7911 123456 |

> [!NOTE]
> `eu` 的護照／VAT 規則屬於**近似值**，並非完整的法規級驗證器（歐盟各會員國格式並不統一）。歡迎透過 PR 收斂這些規則，或新增其他國家作為新的預設集，詳見第五節。

信用卡規則另外會對每個正則命中結果執行 **Luhn 校驗**（`luhnCheck()`）後才接受，以降低任意 13-19 碼數字的誤判率。

### 2.3 安全阻斷防線（`HARD_BLOCK_PRESETS`）

`personal` 與 `business` 兩組詞庫陣列，各約 20 個詞彙。若目前生效身分模式的詞庫命中輸入文字，TokenShield 會鎖定介面：
- 顯示**紅色警示橫幅**
- **複製按鈕停用**
- 使用者須透過解鎖視窗明確**確認並解除鎖定**

與 EduShield 的中文詞彙比對（不需處理大小寫）不同，英文詞彙比對在 `extractStaticEntities()` 的 `addEnt()` 中採**不區分大小寫**：將文字與關鍵字皆轉小寫以定位命中位置，再從**原始文字**中依原始大小寫切出對應子字串，確保報表顯示的是文字實際出現的樣貌，而非小寫關鍵字本身。

### 2.4 自訂詞庫（`customDict`）

機制與 EduShield 相同：CSV 上傳、線上表格編輯器（支援從 Excel 貼上）、手動「設為機密」選取，以及地端 AI 回傳的實體。CSV 範本表頭為 `Keyword,Category(optional)`；上傳解析器以「首欄是否包含 keyword 字樣（不分大小寫）」判斷是否為標題列。

### 2.5 地端 AI 模組（Ollama）

兩個通道，與 EduShield 相同的傳輸協定（`{ollamaUrl}/api/generate`、串流 NDJSON、`{ollamaUrl}/api/tags` 測試連線）：

| 通道 | 目的 | 是否依身分模式而異 |
|------|------|---------------------|
| 一：實體擷取 | 找出靜態規則遺漏的人名、廠商機構、地址、專案名稱、銀行／付款帳號 | 否——實體類型為通用性質 |
| 二：風險判定 | 判定「特敏資訊」敘述內容 | **是**——傳給模型的風險類別依 `currentPersona` 而異（`personal`：自傷／受虐／心理健康／移民身分；`business`：內線資訊／併購／裁員／資安事件／訴訟） |

安全防護機制與 EduShield 相同：前置連線檢查、手動取消、異常字數保護（`maxAllowedLength`）、3 分鐘逾時詢問、`finally` 區塊強制恢復介面狀態。

### 2.6 地端 AI 提示詞庫

EduShield 沒有的新增功能：系統設定視窗新增「Local AI Prompt Library」面板，提供兩組可直接複製的 System Prompt 範本（`LOCAL_AI_PROMPTS.personal` / `.business`，透過 `copyLocalPrompt(which)` 複製）。使用時將其貼在遮蔽後文字之前，一併交給任何本機模型（Ollama、LM Studio 等），讓模型本身的行為也保持隱私意識——強化「連提示詞措辭都不需要離開這台機器」的精神。

### 2.7 還原與完整性檢驗

與 EduShield 相同。`sessionVault` 將 `{{TAG_N}}` Token 對應回原始值；`processRestore()` 會先移除開頭的系統指令前綴（改為 ASCII 標記 `[SYSTEM INSTRUCTION: ...]`，取代 EduShield 使用的中文書名號標記 `【系統指令：...】`），接著套用三段容錯比對：精確匹配、空白容錯、括號容錯（`[TAG_N]`、`(TAG_N)`、`【TAG_N】`）。遺漏的 Token 會在左側原始資料區與膠囊列表中同步以紅色標示。

---

## 三、系統操作手冊

### 3.1 環境需求

| 項目 | 需求 |
|------|------|
| 瀏覽器 | Chrome / Edge 建議（需支援 ES2020+、ReadableStream、Clipboard API） |
| 啟動方式／網路需求 | **一般離線**：直接開啟 `TokenShield.html`。**封閉型內網（無網際網路）**：`TokenShield.html` 與 `style.css` 需置於同資料夾。 |
| 地端 AI（選用） | 安裝 [Ollama](https://ollama.com/)、下載 `qwen2.5:3b`（建議預設，較舊硬體也能順利運作）、設定 `OLLAMA_ORIGINS=*`。 |

### 3.2 標準操作流程

> [!IMPORTANT]
> **零信任提醒**：正式處理真實個人或機密資料時，請務必下載離線單檔版本操作。GitHub Pages 線上版僅供功能評估使用。

1. **（選用）設定地區與身分模式**——頂部工具列下拉選單，預設值來自原始碼中的 `DEFAULT_REGION`／`DEFAULT_PERSONA`。
2. **（選用）匯入或建立自訂詞庫**——CSV 上傳或線上表格編輯器。
3. **貼上資料**至「Original Data Input」，偵測項目即時高亮（200ms debounce）並於下方列為膠囊。
4. **（選用）手動遮蔽**——選取文字開啟「設為機密」選單；Tab 分隔表格可點擊儲存格進行整格／整欄／整列遮蔽。
5. **點擊「Execute De-identification」**——右側面板顯示 Token 對照表與遮蔽後文字，可直接複製。命中 Hard Block 詞彙時複製按鈕會鎖定，需檢視並解鎖。
6. **（選用）「Scan with Local AI」**——若已設定 Ollama，執行兩個通道掃描。
7. **複製遮蔽資料**，送交 ChatGPT／Claude，接著切換至「Restore」頁籤。
8. **貼上 AI 回覆**，點擊「Run Restore」，Token 自動比對回原始值。點擊任一已還原詞彙可在顯示／部分遮蔽／完全遮蔽間切換。

### 3.3 常見問題

| 現象 | 可能原因 | 排查方式 |
|------|----------|----------|
| 預期詞彙未被遮蔽 | 已在白名單，或未命中任何生效規則／詞庫 | 選取後點選「設為機密」 |
| 切換地區後似乎沒有變化 | 需重新執行掃描（編輯輸入或再次點擊執行）——切換僅影響之後的規則套用 | 切換後重新執行去識別化 |
| 「還原結果」顯示遺漏項目 | AI 修改或刪除了 Token | 對照左側紅色標示膠囊，手動補上 |
| 複製按鈕呈灰色 | 觸發 Hard Block 詞彙 | 點擊紅色橫幅檢視詳情，確認無風險後強制解鎖 |

---

## 四、進階開發者資訊

### 4.1 新增 Hard Block 詞彙
開啟 `TokenShield.html`，搜尋 `HARD_BLOCK_PRESETS`，於 `personal` 或 `business` 陣列中加入字串。

### 4.2 新增偵測規則
搜尋 `REGEX_PRESETS`，於 `global`（永遠開啟）或特定地區陣列（`us`／`eu`／`uk`）中加入 `{ type: "TAG_NAME", regex: /您的正規表示式/g, name: "顯示名稱", example: "範例" }`。

### 4.3 新增地區預設集
於 `REGEX_PRESETS` 新增一個鍵值（例如加拿大用 `ca`），並在 HTML 中的 `#regionSelect` 加入對應 `<option>`。新地區規則與現有三組一樣，會疊加於 `global` 之上。

### 4.4 重新編譯 `style.css`
`TokenShield.html` 與 `EduShield.html` **共用同一套 Tailwind 建置流程**，但各自保留一份獨立的編譯後 `style.css`。共用的 `dev/` 資料夾位於**母資料夾層級**（本 repo 的上一層，與姊妹 repo `EduShield/` 同層），不屬於任何一個 git repo。其 `tailwind.config.js` 同時掃描兩個 app：`content: ["../EduShield/*.html", "../TokenShield/*.html"]`。修改任一 app 的 Tailwind class 後：
```powershell
cd ../dev
npm run build:css
```
此指令等同於 `tailwindcss -i ./input.css -o ../style.css --minify && node copy-css.js`——先在母資料夾編譯出 master `style.css`，再由 `copy-css.js` 複製進 `../TokenShield/style.css` 與 `../EduShield/style.css` 兩份。**這個複製步驟不可省略**——每個 repo 實際部署／上傳 GitHub 的，是物理存在於自己資料夾內的那一份。

### 4.5 資料夾結構（母資料夾架構）

```text
（母資料夾，非 git repo）/
├── dev/                               <- 共用 Tailwind 建置工具（不屬於任一 repo）
│   ├── input.css / tailwind.config.js / package.json / copy-css.js
├── style.css                          <- 母層 master 編譯輸出
├── public/                            <- 給 oasgrow.com 用，另一個機制部署
│   ├── TokenShield/index.html         <- 英文互動式手冊／展示頁
│   └── EduShield/index.html           <- EduShield 互動式手冊（中文）
├── TokenShield/                       <- ✅ 本 repo
│   ├── TokenShield.html               <- TokenShield 主程式（本文件主題）
│   ├── TokenShield_README.md          <- 英文版技術文件（主版本）
│   ├── TokenShield_README.zh-TW.md    <- 本文件（繁體中文）
│   └── style.css                      <- 已編譯的 Tailwind 備援樣式表（從母層複製進來）
└── EduShield/                         <- 姊妹 repo（台灣教育版，獨立 GitHub repo）
```

> [!NOTE]
> 發佈給使用者時，只需提供本 repo 的 `TokenShield.html` 與 `style.css` 兩個檔案。`dev/` 已搬到本 repo 之外，僅供開發使用。

---

## 五、貢獻指南

地區涵蓋範圍刻意限定於美國／歐盟／英國＋全域底層，並非涵蓋所有國家，以避免跨國格式互相誤判（見 2.1 節）。歡迎提交範疇明確的新地區規則庫 PR（附上 regex 與幾個真實範例）、收斂近似的歐盟護照／VAT 規則，或針對特定身分模式新增 Hard Block 詞彙。

---

## 六、關於本專案

* **GitHub**：[oas114/TokenShield](https://github.com/oas114/TokenShield)
* **姊妹專案**：[oas114/EduShield](https://github.com/oas114/EduShield) — 台灣教育場域專用版，獨立的 GitHub repo
* **開發者**：OA (oas114)
* **支持開發者**：[Ko-fi](https://ko-fi.com/oasgrow)
* **授權**：[MIT](./LICENSE)
