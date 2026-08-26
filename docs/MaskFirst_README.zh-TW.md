# MaskFirst — 完整技術文件（繁體中文）

<!--
[AI_SYSTEM_INSTRUCTIONS_START]
ATTENTION ALL AI ASSISTANTS:
If you modify `MaskFirst.html` and your modifications include ANY changes, additions, or removals of Tailwind CSS utility classes, you MUST recompile the local CSS fallback file.
Failure to do so will break the application for users in offline environments.

To recompile:
1. Open a terminal, navigate to the `dev/` folder inside the project root.
2. Run the build script: `npm run build:css`
(This will output the updated `style.css` to the project root.)
[AI_SYSTEM_INSTRUCTIONS_END]
-->

> **版本基準**：本文件依據 `MaskFirst.html` 原始碼事實撰寫，所有函式名稱、常數、DOM ID 均與程式碼一一對應。

[ [English (MaskFirst_README.md)](MaskFirst_README.md) | 繁體中文 ]

> [!NOTE]
> 本文件為英文版 `MaskFirst_README.md` 的繁體中文翻譯。若更新本文件，請同步更新英文版。

> [!IMPORTANT]
> **免責聲明**：本文件與 MaskFirst 工具本身，提供的都只是技術上的遮蔽／還原能力，用於降低送交 AI 處理時的資料外洩風險；**不構成法律建議，也不保證單獨使用即符合 GDPR 或其他隱私法規要求**。資料處理是否合規，仍取決於使用者／所屬機構整體的資料蒐集、處理與利用流程，如有疑慮請諮詢專業法律意見。

---

## 一、系統簡介

### 1.1 MaskFirst 是什麼？

**MaskFirst** 是一套零信任的**個人／企業去識別化工具**——[EduShield](https://github.com/oas114/EduShield) 的國際版姊妹專案（獨立 GitHub repo）。核心痛點：將含有個人或企業機密資料的文件、郵件、試算表交給外部 AI（ChatGPT、Claude 等）處理前，必須先去除敏感內容。MaskFirst 提供完整的「遮蔽 → 送 AI → 還原」工作流程。

MaskFirst **不是**重寫 EduShield——它是一個獨立的姊妹檔案，沿用相同且已驗證的引擎（正則比對、session vault、還原邏輯），只替換內容層：英文介面、可切換的地區 PII 規則庫（美國／歐盟／英國＋全域底層），以及 Personal／Business 身分模式，取代台灣教育專用規則。

### 1.2 架構與資安特點

| 特性 | 說明 |
|------|------|
| **執行環境** | 純靜態單一 HTML 檔案，瀏覽器直接開啟即可，無需 Node.js、伺服器或安裝程序 |
| **資料生命週期** | 所有處理（正則比對、替換、還原）均於瀏覽器記憶體中進行；頁面關閉或重整後，`sessionVault`、`customDict` 等全數消滅，不寫入磁碟、不上傳雲端 |
| **無憑證依賴** | 不需 API Key 或帳號；地端 AI（Ollama）為選用模組，僅連線 `http://localhost:11434`（本機迴路） |
| **啟動時清空** | `window.addEventListener('load', ...)` 於載入時清空所有 `textarea` 與 `input[type="text"]`（排除 `ollamaUrl`、`ollamaModel`），防止瀏覽器自動填入歷史資料 |
| **CSS 框架** | Tailwind CSS，三段式降級載入：CDN → 本地 `style.css`（見 [tailwind.config.js](../dev/tailwind.config.js) 的 `content: ["../*.html"]`）→ 防呆引導畫面 |
| **地區／身分／語言會記住，文件內容永遠不會** | 地區、身分、介面語言的選擇會存入 `localStorage`，下次開啟自動還原——這是介面偏好設定，不是文件資料。其餘一切（`sessionVault`、`customDict`、貼上的文字）仍依上面「資料生命週期」列的規則，重整/關閉即消滅。詳見下方 2.1 節如何變更裝置首次開啟時的預設值 |

---

## 二、核心模組與技術規格

### 2.1 地區與身分規則庫（`REGEX_PRESETS_DEFAULT` / `HARD_BLOCK_PRESETS_DEFAULT`）

這是 MaskFirst 與 EduShield 在結構上最大的差異。偵測規則與 Hard Block 詞庫不再是單一綁死台灣格式的陣列，而是組織成使用者可於頂部工具列兩個下拉選單（`#regionSelect`、`#personaSelect`）即時切換的「規則預設集」。

```javascript
const DEFAULT_REGION = "us";      // "us" | "eu" | "uk" | "tw" | "jp" —僅作為 fallback，見下方說明
const DEFAULT_PERSONA = "personal"; // "personal" | "business" —僅作為 fallback，見下方說明
const REGION_STORAGE_KEY = 'maskfirst_region';
const PERSONA_STORAGE_KEY = 'maskfirst_persona';
let currentRegion = localStorage.getItem(REGION_STORAGE_KEY) || DEFAULT_REGION;
let currentPersona = localStorage.getItem(PERSONA_STORAGE_KEY) || DEFAULT_PERSONA;
```

您的地區／身分選擇會在每次變更時存入 `localStorage`，下次開啟頁面自動還原，不需要每次都重新選一次。`DEFAULT_REGION`／`DEFAULT_PERSONA` 只在裝置**第一次**開啟（尚未存過任何值）時才會用到。若想改變全新瀏覽器/使用者設定檔的起始值，請用文字編輯器開啟 `MaskFirst.html`，找到 `<script>` 區塊開頭附近的這兩個常數並修改其值。

**依顯示語言決定起始地區的一次性提示。** 若地區從未被明確存過值，把顯示語言（見 2.9 節）切成繁體中文時，地區會被帶到 `tw`，不會落回 `DEFAULT_REGION`——大多數選擇中文介面的使用者，處理的本來就是台灣相關文件，這樣可以省掉開啟工具後還要多點一次地區下拉選單的步驟。原始碼裡的 `LANG_TO_STARTER_REGION` 控制這個對應。這是**一次性的起始提示，不是持續性的連動**：只要地區還沒被存過值（用 `localStorage.getItem(REGION_STORAGE_KEY) === null` 判斷）才會生效——地區一旦有了任何明確值（不管是被這個機制帶入的，還是使用者自己手動選的），之後不管怎麼切換語言都不會再被改動。想要「EU 規則庫＋繁體中文介面」這種組合（見 2.9 節）的使用者仍然完全支援——只要曾經碰過地區下拉選單一次即可（哪怕只是點開 `#regionSelect` 選了跟原本一樣的值）。

同一時間僅有**一組**地區規則生效，疊加於永遠開啟的 `global` 底層規則之上——這是刻意設計以避免不同國家格式互相誤判（例如同一個 9 碼數字不該同時被當作美國 SSN 又被當作其他東西）。身分模式邏輯相同，同一時間僅有一組詞庫生效。

`getActiveRegexRules()` 回傳 `[...regexPresets.global, ...regexPresets.custom, ...regexPresets[currentRegion]]`；`getHardBlockKeywords()` 回傳 `[...hardBlockPresets.custom, ...hardBlockPresets[currentPersona]]`。兩者於每次掃描時即時運算，因此切換下拉選單會立即生效（會觸發 `triggerPreview()`）。其中 `custom` 桶（新增，見 2.8 節）不分地區/身分、永遠合併進去；內建預設值凍結在 `REGEX_PRESETS_DEFAULT`／`HARD_BLOCK_PRESETS_DEFAULT`，`regexPresets`／`hardBlockPresets` 才是這兩個 getter 實際讀取的即時運作狀態。

### 2.2 偵測規則庫（`REGEX_PRESETS_DEFAULT`）

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
| `tw` | 身分證字號 | `TW_ID_N` | `[A-Z][12]\d{8}` ＋ **檢查碼驗證**（`twIdChecksum()`） | A123456789 |
| `tw` | 居留證號/統一證號 | `TW_ARC_N` | `[A-Z][A-D89]\d{8}` | A800000014 |
| `tw` | 行動電話 | `TW_MOBILE_N` | `09\d{2}-?\d{3}-?\d{3}` | 0912-345-678 |
| `tw` | 市話/分機 | `TW_TEL_N` | 區碼 ＋ 號碼（可含分機） | 02-23456789#123 |
| `tw` | 統一編號 | `TW_UBN_N` | 8 碼 ＋ **檢查碼驗證**（`twUbnChecksum()`，2021/12/22 後除 5 規則） | 04595257 |
| `tw` | 戶籍/通訊地址 | `TW_ADDRESS_N` | 縣市＋區里＋路街＋號 | 406 台中市北屯區崇德路三段100號 |
| `jp` | 個人番号（My Number） | `JP_MYNUMBER_N` | 12 碼 ＋ **檢查碼驗證**（`jpMyNumberChecksum()`） | 1234-5678-9018 |
| `jp` | 法人番号 | `JP_CORP_NUMBER_N` | 13 碼 ＋ **檢查碼驗證**（`jpCorpNumberChecksum()`） | 8700110005901 |
| `jp` | 行動電話 | `JP_MOBILE_N` | `0[789]0-?\d{4}-?\d{4}` | 090-1234-5678 |
| `jp` | 市話 | `JP_TEL_N` | 區碼 ＋ 號碼 | 03-1234-5678 |
| `jp` | 郵遞區號 | `JP_POSTAL_N` | `\d{3}-?\d{4}` | 123-4567 |
| `jp` | 護照號碼（近似） | `JP_PASSPORT_N` | 2 字母 ＋ 7 數字 | TH1234567 |

> [!NOTE]
> `eu` 的護照／VAT 規則、`jp` 的護照規則屬於**近似值**，並非完整的法規級驗證器（各國格式細節、官方檢查碼皆未必完整涵蓋）。歡迎透過 PR 收斂這些規則，或新增其他國家作為新的預設集，詳見第五節。

信用卡規則另外會對每個正則命中結果執行 **Luhn 校驗**（`luhnCheck()`）後才接受，以降低任意 13-19 碼數字的誤判率。`tw`／`jp` 的身分證字號、統一編號、個人番号、法人番号也比照做法，各自接上對應官方檢查碼演算法的 `validate` 函式（`twIdChecksum()`／`twUbnChecksum()`／`jpMyNumberChecksum()`／`jpCorpNumberChecksum()`，定義在 `luhnCheck()` 旁邊），大幅降低這類寬鬆數字格式的誤判率。

### 2.3 安全阻斷防線（`HARD_BLOCK_PRESETS_DEFAULT`）

`personal` 與 `business` 兩組陣列，各約 20～21 個**概念**——不是扁平的關鍵字字串。每個概念是一個 `{ en, zh }` 組合，例如 `{ en: "sexual assault", zh: "性侵害" }`。`getHardBlockKeywords()` 會把目前生效身分（加上維持單一數值的 `custom` 層）裡每個概念、每個非空欄位攤平成一份字串包含掃描清單——**兩種語言永遠同時生效，不受 `currentLang`／`currentRegion` 影響**。真正決定會不會觸發的是**貼上內容本身的語言**，不是介面顯示語言或選定的地區；兩種文字永遠保持同時生效，才能讓貼上中文的內容跟貼上英文一樣觸發同一層防護。只要攤平後的清單裡有任何一個詞命中輸入文字，MaskFirst 就會鎖定介面：
- 顯示**紅色警示橫幅**
- **複製按鈕停用**
- 使用者須透過解鎖視窗明確**確認並解除鎖定**

與 EduShield 的中文詞彙比對（不需處理大小寫）不同，`extractStaticEntities()` 的 `addEnt()` 裡的關鍵字比對採**不區分大小寫**：將文字與關鍵字皆轉小寫以定位命中位置，再從**原始文字**中依原始大小寫切出對應子字串，確保報表顯示的是文字實際出現的樣貌，而非小寫關鍵字本身。

「管理自訂防護規則」面板（見 2.8 節）裡每個概念顯示成一列、兩個並排欄位（讀中文的人通常也讀得懂英文，這樣比捲動兩份各自獨立、內容其實是同一批約 20 個概念的清單更直觀）。編輯或清空單一欄位只會影響那個語言——欄位清空代表該語言不參與比對，但列本身不會被刪除；刪除整列才會兩個語言一起移除。`custom` 詞庫不受這個機制影響——維持原本的單一自由文字值，透過「+ 新增一列」或 CSV 匯入新增，沒有語言結構。

### 2.4 自訂詞庫（`customDict`）

機制與 EduShield 相同，每筆多了 `source` 欄位（`builtin`／`auto-loaded`／`manual`／`overridden`／`ai-session`，會在介面上以徽章顯示，見 2.8 節）：CSV 上傳、線上表格編輯器（支援從 Excel 貼上）、手動「設為機密」選取，以及地端 AI 回傳的實體。CSV 範本表頭為 `Keyword,Category(optional)`；上傳解析器以「首欄是否包含 keyword 字樣（不分大小寫）」判斷是否為標題列，並改用 `parseCsvLine()`（支援標準 CSV 雙引號跳脫）取代原本單純的 `split(',')`。CSV 檔案匯入會經過「合併/取代/取消」對話框（見 2.8 節），不再直接整批覆蓋；線上表格編輯器則維持原本的整批覆蓋行為。

### 2.5 地端 AI 模組（Ollama）

兩個通道，與 EduShield 相同的傳輸協定（`{ollamaUrl}/api/generate`、串流 NDJSON、`{ollamaUrl}/api/tags` 測試連線）。`format` 欄位是完整 JSON Schema（不是寬鬆的 `"json"` 字串），強制扁平陣列＋`type` 五選一枚舉（`PERSON`／`VENDOR`／`ADDRESS`／`PROJECT`／`BANK_ACCT`），避免模型回傳依類型分組的物件、被容錯邏輯悄悄只留下第一類。通道一固定 `options: { temperature: 0, num_ctx: 8192 }`；通道二只固定 `options: { num_ctx: 8192 }`（刻意不鎖 `temperature`，理由見下方）。`num_ctx` 從 Ollama 預設的 2048 拉高，避免貼上文字較長時後段被靜默截斷。

| 通道 | 目的 | 是否依身分模式而異 |
|------|------|---------------------|
| 一：實體擷取 | 找出靜態規則遺漏的人名、廠商機構、地址、專案名稱、銀行／付款帳號 | 否——實體類型為通用性質 |
| 二：風險判定 | 判定「特敏資訊」敘述內容 | **是**——傳給模型的風險類別依 `currentPersona` 而異（`personal`：自傷／受虐／心理健康／移民身分；`business`：內線資訊／併購／裁員／資安事件／訴訟） |

> **已知準確度限制**：實測發現通道一對人名的擷取準確度不夠穩定，`qwen2.5:3b` 這類小模型對自由格式、高度依賴上下文的人名判讀常有漏抓。UI 端已因此把被動提示（`layer1HintBanner`）的建議從「執行深度掃描」改為「加入自訂詞庫」，人名不應被視為通道一的可靠輸出，僅供輔助參考。

> **通道二連續呼叫 3 次、取聯集**：任一次回傳 `critical: true` 即視為風險（非多數決）。三次呼叫刻意維持模型預設 `temperature`（不套用通道一的 `temperature: 0`），否則三次會得到幾乎相同的結果、等於白跑；用真實敘述文字實測發現，語意隱晦的委婉措辭單次呼叫命中率僅約 20%，多次呼叫能提升累積捕捉率。若三次中有呼叫失敗（連線逾時等），會跳過該次繼續執行，只有三次全部失敗才顯示錯誤。按鈕文字也從通道一的逐字元計數，改為通道二顯示第幾次呼叫（例如「語意風險確認中 (第 2/3 次)」），因為通道二輸出短，逐字元進度意義不大。

安全防護機制與 EduShield 相同：前置連線檢查、手動取消、異常字數保護（`maxAllowedLength`）、3 分鐘逾時詢問、`finally` 區塊強制恢復介面狀態。

### 2.6 地端 AI 提示詞庫

EduShield 沒有的新增功能：系統設定視窗新增「Local AI Prompt Library」面板，提供兩組可直接複製的 System Prompt 範本（`LOCAL_AI_PROMPTS.personal` / `.business`，透過 `copyLocalPrompt(which)` 複製）。使用時將其貼在遮蔽後文字之前，一併交給任何本機模型（Ollama、LM Studio 等），讓模型本身的行為也保持隱私意識——強化「連提示詞措辭都不需要離開這台機器」的精神。

> [!NOTE]
> `LOCAL_AI_PROMPTS` 與 `processAnonymizePhase2()` 實際呼叫的通道一/二提示詞完全無關，純粹是給使用者複製貼到外部聊天工具用的靜態文字。真正送給 Ollama 的提示詞存放在 `aiPrompts`／`AI_PROMPTS_DEFAULT`，可透過「管理自訂防護規則」介面編輯，見 2.8 節。

### 2.7 還原與完整性檢驗

與 EduShield 相同。`sessionVault` 將 `{{TAG_N}}` Token 對應回原始值；`processRestore()` 會先移除開頭的系統指令前綴（改為 ASCII 標記 `[SYSTEM INSTRUCTION: ...]`，取代 EduShield 使用的中文書名號標記 `【系統指令：...】`），接著套用三段容錯比對：精確匹配、空白容錯、括號容錯（`[TAG_N]`、`(TAG_N)`、`【TAG_N】`）。遺漏的 Token 會在左側原始資料區與膠囊列表中同步以紅色標示，且膠囊改為**可點擊**：點擊後跳出小視窗，讓使用者貼上 AI 回覆裡實際出現的文字，套用後對還原結果做一次精確字串替換即可修復，不需要重新執行還原。

**持久化遮蔽對應儲存**：`sessionVault` 每次執行都會重建，Token 編號無法跨批次保持穩定。還原分頁的「Mapping Vault」按鈕可以把目前累積的「原始值 ↔ Token」對應表（`persistentVault`）匯出成檔案——**未加密 CSV**（三欄，可用 Excel 開啟，匯出前會先跳風險提醒，因為檔案內容是明文）或**加密 JSON**（Web Crypto PBKDF2＋AES-GCM，需設密碼，忘記密碼將無法復原）。之後匯入（可選合併或完全取代）即可讓已知原始值跨 session 沿用舊 Token。此功能僅限手動、單次匯出／匯入，不會自動寫入瀏覽器儲存；表格模式的座標式遮蔽 Token（`{{TAB_C...}}`）因位置性較強而不納入此對應表。

### 2.8 自訂防護管理系統（四大維度）

點擊工具列的「**管理自訂防護規則**」，可對以下四個維度進行自訂、匯入與匯出，不需要手動編輯 HTML 原始碼：名冊/詞庫（見 2.4 節）、硬阻斷詞彙、正則規則，以及地端 AI 提示詞（是實際送去掃描用的提示詞，不是 2.6 節那個複製貼上用的 Prompt Library）。

**範圍**：CSV／設定檔的*匯入*仍然只會寫進一個**永遠合併、不分地區/身分的「custom」桶**（`hardBlockPresets.custom`、`regexPresets.custom`），不會寫進某個特定 persona/region 自己的陣列——這樣可以讓衝突處理維持單純（只在 `custom` 桶內去重複，不會悄悄改動某個特定身分或地區、匯入時可能根本不是要改的內建清單）。**這個面板的線上新增/編輯 UI 是另一條獨立路徑**：表格顯示的是目前*實際套用中*的組合（硬阻斷關鍵字：`custom`＋目前選定的 Persona；正則規則：`custom`＋`global`＋目前選定的 Region），任一列都能直接編輯或刪除——包含內建列，不限 `custom`——並標示小徽章（`PERSONAL`／`BUSINESS`／`GLOBAL`／`US` 等）標明所屬桶別。切換 Region 或 Persona 時，若面板（或 PII Rule Guide）已開啟會即時重新渲染。

內建預設值凍結為 `HARD_BLOCK_PRESETS_DEFAULT`／`REGEX_PRESETS_DEFAULT`／`AI_PROMPTS_DEFAULT`。即時運作狀態：
```javascript
let hardBlockPresets = { personal: [], business: [], custom: [] };
let regexPresets = { global: [], us: [], eu: [], uk: [], tw: [], jp: [], custom: [] };
let aiPrompts = { channel1: '', channel2Personal: '', channel2Business: '' };
```
每筆都帶 `source` 欄位並以徽章顯示：`Built-in`／`Config file import`／`Manually imported`／`Overridden`。正則規則改存 `pattern`／`flags` 字串（不再是即時的 `RegExp` 物件），每次使用都經過 `tryCompileRegexRow()` 的 try/catch 保護，格式錯誤的規則會被略過並個別回報，不會中斷其餘掃描。

內建項目另外帶有回指對應原始值的穩定連結，編輯多次也不會遺失——硬阻斷是 `default: { en, zh }`（整組一起存，因為「還原預設」是一次還原兩個欄位），正則規則是 `defaultPattern`。這撐起兩個安全網功能，讓「Reset to Defaults」（會連 `custom` 一起清空）不是唯一的復原手段：任一已編輯/仍存在的內建列都有**「還原預設」**按鈕；已刪除的內建列會出現在表格下方**「已從預設清單移除」**清單，可個別加回（`revertHardBlockEntryAt()`／`restoreRemovedHardBlockDefault()` 與其正則版本——硬阻斷版本靠 `en` 欄位找回被移除的概念，因為 `en` 在同一個 bucket 裡永遠存在且唯一）。編輯正則規則列時，內建的 `validate` 檢查碼函式（Luhn、台灣身分證、日本 My Number 等）會被保留——比對依據是規則的 `type`（穩定識別碼）而非 pattern 文字，即使 pattern／名稱／範例都被改過也找得到；正則規則另外還帶一個獨立的 `defaultNameByLang` 連結（即時 `name` 字串當初取自哪一組 `{en,zh}` 組合），讓切換顯示語言時能分辨這個名稱是使用者編輯過的、還是還沒動過的預設值——詳見 2.9 節。這些都是工作階段記憶體狀態，重新整理即回到真正的預設值，除非透過下方的設定檔匯出/匯入保留（因此「Reset to Defaults」與「Export/Import Config File」放在一起說明）。

CSV 範本：硬阻斷詞彙為單欄 `Keyword`；正則規則為 4 欄 `TypeTag,RuleName,Pattern,ExampleText`，`Pattern` 欄可填純 pattern（預設補 `g` flag）或完整的 `/pattern/flags` 字面量格式字串。

匯入 CSV 時，只要 `custom` 桶已有資料就會跳出**合併/取代/取消**對話框：
- **合併**：保留 `custom` 桶既有項目，新增資料追加進去；正則的 **pattern 字串**（不含 flags）或關鍵字/值（不分大小寫）與既有 `custom` 項目相同時，會完全覆蓋（標籤變為 `Overridden`）。
- **完全取代**：清空該維度的 `custom` 桶，改以本次匯入內容為準。
- **取消**：不做任何變更。

**手動匯入設定檔**：早期版本曾在開機時以動態注入 `<script src="maskfirst.config.js">` 自動載入同資料夾檔案；這個做法會讓被竄改的檔案在使用者毫無察覺下夾帶任意程式碼執行，牴觸「個資不離開瀏覽器」的核心信任主張，已於 2026-08-25 改為以下的手動匯入流程：
```javascript
window.TOKENSHIELD_AUTO_CONFIG = {
  version: 1,
  roster: [ { type: "VENDOR", value: "...", reason: "..." } ],
  hardBlock: [ "custom hard-block term" ],
  regexRules: [ { type: "CUSTOM_CODE", pattern: "CODE-\\d{4}", flags: "g", name: "Custom code", example: "CODE-1234" } ],
  // 只有真的被編輯/刪除過的 Persona/Region 桶才會出現這兩個欄位——見 hardBlockBucketIsDefault()/regexBucketIsDefault()。
  // 完全沒動過的桶不會出現，避免從沒編輯過內建規則的人匯出的檔案也帶著整份規則庫。
  hardBlockOverrides: { personal: [ { en: "...", zh: "..." } ] },
  regexOverrides: { us: [ { type: "US_SSN", pattern: "...", flags: "g", name: "...", example: "..." } ] },
  aiPrompts: { channel1: "...{{TEXT}}", channel2Personal: "...{{TEXT}}", channel2Business: "...{{TEXT}}" }
};
```
三個按鈕都收在「管理自訂防護規則」面板底部的「Advanced Settings: Import / Export Config File」摺疊區塊裡（手機寬度會直接隱藏）。

「**Import Config File**」開啟檔案選取視窗，選取的檔案只以字串掃描＋`JSON.parse()` 解析（絕不 `eval`／執行檔案內容），接著會跳出跟 CSV 匯入共用的**「合併／完全取代」**選擇對話框（`openConfigImportDialog()`），列出各維度筆數。**這個選擇在這裡比 CSV 匯入更關鍵**：合併是拿 `custom` 桶項目本身的內容（正則的 pattern 文字、關鍵字文字本身）當比對鍵，如果你編輯的正是這個欄位，合併會認不出「這是同一條規則的新版本」，結果變成新舊並存。`hardBlockOverrides`／`regexOverrides` 讓這點更明顯：因為匯出的清單是不帶身分資訊的純陣列，**合併**只會把新增/變動的值疊加進去（絕不會移除任何東西），只有**完全取代**才能正確重新套用一次刪除，或乾淨地取代掉一個已編輯的值——要讓重新整理後「編輯過或刪除過的內建規則」真正生效，必須選完全取代。有個身分連結的取捨要注意：一個編輯過、文字已經跟真正預設值不完全相同的內建值，匯入回來會標成 `Overridden`／`auto-loaded`，但不會再帶有自己的 `defaultValue`／`defaultPattern` 連結（也就是不會再有專屬的「還原預設」按鈕）——編輯本身會被正確套用，只是「這是編輯前長怎樣」這個麵包屑會遺失。正則的 override 一樣是靠 `type` 找回對應的內建 `validate` 函式。這是**手動**、一次性的操作，不會在重新整理或下次開啟時自動套用，使用者每次都要重新匯入一次；頁面開啟時會有一則常駐提示引導匯入，文字刻意不聲稱「已偵測到」，因為瀏覽器安全機制不允許背景偵測同資料夾檔案是否存在。

「**Export Config File**」會把目前記憶體中四個維度的最新狀態（含所有合併/覆蓋結果，排除 `ai-session` 標籤的暫存名冊項目）打包成同樣格式，下載成固定檔名 `maskfirst.config.js`，可分享給同事或在其他裝置重複使用——`hardBlockOverrides`／`regexOverrides` 只會針對跟真正預設值不同的桶才加入。「**Reset to Defaults**」（會先跳出確認對話框）把四個維度重置回內建預設值，用來捨棄當次手動調整或已匯入的內容。

> [!NOTE]
> 此機制只處理規則/提示詞設定，完全不涉及使用者實際輸入的文件內容——`sessionVault`、輸入文字框等仍完全遵循 1.2 節提到的既有零持久化承諾，頁面關閉或重整後立即消失。

### 2.9 顯示語言（i18n）

頂部工具列的 `#langSelect` 下拉選單可切換介面在**英文／繁體中文**之間顯示，刻意與 `currentRegion`／`currentPersona`（見 2.1 節）脫鉤——Region／Persona 決定套用哪一組*規則資料*，這裡只決定介面文字用哪個*語言*呈現，所以「套用 EU 規則庫、介面顯示繁體中文」這種組合是完全支援的。唯一的例外是 2.1 節提到的一次性起始提示：地區從未被碰過時，切到繁體中文會把地區帶到 `tw`——地區一旦被碰過一次之後，這兩者在該瀏覽器往後的使用期間就完全獨立，不會再互相影響。

```javascript
const LANG_STORAGE_KEY = 'maskfirst_display_lang';
let currentLang = localStorage.getItem(LANG_STORAGE_KEY) || 'en'; // 會被記住，跟 Region/Persona（見 2.1 節）做法一致
const I18N = { someKey: { en: '...', zh: '...' }, /* 約 310 組 key */ };
function t(key, vars) { /* 查找 I18N[key][currentLang]，找不到就退回 en，再退回 key 本身；支援 {placeholder} 變數插值 */ }
function applyLanguage(lang) { /* 設定 currentLang、寫入 localStorage、掃描套用所有 data-i18n*、呼叫 refreshDynamicText() */ }
```

靜態畫面透過 `data-i18n`（設定 `innerHTML`）、`data-i18n-placeholder`、`data-i18n-title`、`data-i18n-aria-label` 這幾個屬性接入字典。動態組出來的字串（toast 訊息、`showConfirmModal()` 的訊息、`renderHardBlockMgmtTable()`／`renderRegexMgmtTable()` 這類表格渲染函式）則直接呼叫 `t()`，不再把英文字面字串寫死。`refreshDynamicText()` 會重新執行這些「純渲染」函式，讓當下已經開啟的面板在切換語言的當下就立即反映新語言，不需要使用者重新開啟一次。PII Rule Guide 已不再渲染即時規則表格（`renderGuideTable()` 已移除，指南改為純概念說明，見 2.8 節）——`refreshGuideModalIfOpen()` 保留成一個有記錄的空函式，這樣其他呼叫它的地方不用知道這件事。

**`HARD_BLOCK_PRESETS_DEFAULT`（`personal`／`business` 關鍵字清單）把英文／繁體中文兩種語言的詞彙同時放進同一個 bucket，兩者永遠同時生效**——這**不是**跟著 `currentLang` 切換的。比對方式是單純的字串包含掃描（`getHardBlockKeywords()` → `lowerText.includes(kw.toLowerCase())`），所以真正決定貼上內容會不會觸發硬阻斷的是**內容本身的語言**，不是介面顯示語言；不管 `currentLang` 設成什麼，兩種文字都保持同時生效，才能讓貼上中文的極敏感內容，跟貼上英文內容一樣觸發同一層防護，而不是只有貼上內容剛好跟介面顯示語言一致時才會觸發。

**`LOCAL_AI_PROMPTS` 與 `AI_PROMPTS_DEFAULT`（`aiPrompts` 的來源）則改為依顯示語言分別存放**（`en`／`zh`），跟上面的硬阻斷清單做法不同——這些是給人看、給人編輯的內容（複製到剪貼簿的提示詞庫，見 2.6 節；管理自訂防護規則的「AI 提示詞」分頁，見 2.8 節），不是拿去跟貼上的內容做比對，所以讓使用者依照介面設定的語言看到內容（而不是預設使用者看得懂英文）才是這裡真正在意的事。`applyLanguage()` 會在每次切換語言時，把提示詞換成新語言的預設文字，但**只換掉還維持原語言預設值、使用者沒動過的欄位**——任何使用者在「AI 提示詞」面板自訂過的內容，切換語言時都會原封不動保留。提示詞裡要求模型回傳的 JSON 欄位名稱與列舉值（`type`／`value`／`reason`、`PERSON`／`VENDOR`／`ADDRESS`／`PROJECT`／`BANK_ACCT`、`critical`）在兩種語言版本裡都維持英文不變，因為 `CHANNEL1_RESPONSE_SCHEMA`／`CHANNEL2_RESPONSE_SCHEMA` 與解析模型回傳 JSON 的程式碼都依賴這些固定字面值。

**`REGEX_PRESETS_DEFAULT` 規則的 `name` 欄位也改為依顯示語言分別存放**（`{ en, zh }`，透過規則資料旁的 `N()` 小工具函式建立）——規則的 `name` 不只是管理面板裡的標籤文字，它會直接流進 `entities.push({ ..., reason: rule.name, ... })`，變成即時預覽 entity chip 上顯示的類別文字（見 2.2 節的 `data.reason`），所以之前只有英文版，會讓中文介面的 chip 標籤仍然顯示英文。`regexNameFor(def)` 會把某條規則的 `name` 物件解析成 `currentLang` 對應的即時字串（找不到就退回英文）；`syncRegexNamesToLanguage()` 比照 `syncAiPromptsToLanguage()`「還維持原語言預設值才換、使用者編輯過就不動」的規則，逐條規則各自判斷（靠每條內建規則活的資料上都帶著的 `defaultNameByLang` 連結），所以使用者在管理面板編輯過的規則名稱，切換語言時會原封不動保留。切換當下畫面上已經顯示的 chip，標籤會維持切換前的語言直到下次重新掃描，這跟既有「切換地區不會馬上生效」的先例（見 3.3 節疑難排解表格）是同一套邏輯。

**仍然刻意不翻譯的部分**：`REGEX_PRESETS_DEFAULT` 規則的 `example` 欄位——比對範例本身是格式示範（數字、標點符號），不是語言內容，翻譯「192.168.1.1」或「4111 1111 1111 1111」這類範例沒有意義；唯一的例外 `TW_ADDRESS` 的範例本來就已經是一個實際的台灣地址，用的正是這條規則要比對的語言。`type` 在三種語言版本裡也都維持不翻譯——它是被 token 產生機制（`{{TYPE_N}}`）與 CSV／設定檔匯入比對邏輯依賴的穩定技術標籤，不是給人讀的文字。這裡（以及 Region／Persona，見 2.1 節）的 `localStorage` 持久化，是刻意對 MaskFirst 資料生命週期承諾（見 1.2 節）的一個範圍受限的例外——這些都是介面偏好設定，不是文件內容或掃描狀態，後者仍然每次重整都會消滅。

---

## 三、系統操作手冊

### 3.1 環境需求

| 項目 | 需求 |
|------|------|
| 瀏覽器 | Chrome / Edge 建議（需支援 ES2020+、ReadableStream、Clipboard API） |
| 啟動方式／網路需求 | **一般離線**：直接開啟 `MaskFirst.html`。**封閉型內網（無網際網路）**：`MaskFirst.html` 與 `style.css` 需置於同資料夾。 |
| 地端 AI（選用） | 安裝 [Ollama](https://ollama.com/)、下載 `qwen2.5:3b`（建議預設，較舊硬體也能順利運作）、設定 `OLLAMA_ORIGINS=*`。 |

### 3.2 標準操作流程

> [!IMPORTANT]
> **零信任提醒**：正式處理真實個人或機密資料時，請務必下載離線單檔版本操作。GitHub Pages 線上版僅供功能評估使用。

1. **（選用）設定地區與身分模式**——頂部工具列下拉選單，第一次開啟時預設值來自原始碼中的 `DEFAULT_REGION`／`DEFAULT_PERSONA`；之後您的選擇會存入 `localStorage`，下次開啟自動還原。
2. **（選用）匯入或建立自訂詞庫**——CSV 上傳或線上表格編輯器。
3. **貼上資料**至「Original Data Input」，偵測項目即時高亮（200ms debounce）並於下方列為膠囊，每個膠囊都會直接標示類別（如 `PERSON`、`PHONE`），不需要滑鼠懸浮才看得出來是哪一種。
4. **（選用）手動遮蔽**——選取文字開啟「設為機密」選單；Tab 分隔表格可點擊儲存格進行整格／整欄／整列遮蔽。
5. **點擊「Execute De-identification」**——右側面板顯示 Token 對照表與遮蔽後文字，可直接複製。命中 Hard Block 詞彙時複製按鈕會鎖定，需檢視並解鎖。
6. **（選用）「Scan with Local AI」**——若已設定 Ollama，執行兩個通道掃描。
7. **複製遮蔽資料**，送交 ChatGPT／Claude，接著切換至「Restore」頁籤。
8. **貼上 AI 回覆**，點擊「Run Restore」，Token 自動比對回原始值。點擊任一已還原詞彙可在顯示／部分遮蔽／完全遮蔽間切換。

#### 鍵盤快捷鍵

三個頁籤（De-identification／Restore／Quick Mask）皆支援以下快捷鍵，作用於目前所在的頁籤：

| 快捷鍵 | 作用 |
|---|---|
| `Ctrl+Enter`（Mac：`Cmd+Enter`） | 執行目前頁籤的主要動作（Execute De-identification／Run Restore／Detect） |
| `Ctrl+Alt+C`（Mac：`Cmd+Option+C`） | 複製目前頁籤的結果 |

複製鍵刻意不使用瀏覽器保留的 `Ctrl+Shift+C`（多數瀏覽器用來開啟「檢查元素」）。任何彈出視窗（設定、指南、管理自訂防護規則等）開啟時快捷鍵會暫停，避免與視窗內操作衝突。

### 3.3 常見問題

| 現象 | 可能原因 | 排查方式 |
|------|----------|----------|
| 預期詞彙未被遮蔽 | 已在白名單，或未命中任何生效規則／詞庫 | 選取後點選「設為機密」 |
| 切換地區後似乎沒有變化 | 需重新執行掃描（編輯輸入或再次點擊執行）——切換僅影響之後的規則套用 | 切換後重新執行去識別化 |
| 「還原結果」顯示遺漏項目 | AI 修改或刪除了 Token | 點擊左側紅色標示膠囊，貼上 AI 回覆裡實際出現的文字即可套用修復 |
| 複製按鈕呈灰色 | 觸發 Hard Block 詞彙 | 點擊紅色橫幅檢視詳情，確認無風險後強制解鎖 |

---

## 四、進階開發者資訊

### 4.1 新增 Hard Block 詞彙
完全不需要改原始碼——透過「管理自訂防護規則」→「硬阻斷詞彙」分頁（見 2.8 節）：點「+ Add Row」新增 `custom` 項目，或直接點擊目前 Persona 顯示中的任一列編輯/刪除內建項目，CSV 匯入則用於批次異動。這只會影響你自己的工作副本（工作階段狀態，或你匯出的設定檔）——如果要改變這個檔案分享給所有人時的 `personal`／`business` **預設值**本身，請改到 `MaskFirst.html` 原始碼編輯 `HARD_BLOCK_PRESETS_DEFAULT`。

### 4.2 新增偵測規則
透過「管理自訂防護規則」→「正則規則」分頁（見 2.8 節）：點「+ Add Row」新增 `custom` 項目，或直接點擊目前 Region 顯示中的任一列編輯/刪除內建項目（儲存時會自動驗證格式與 ReDoS 風險），CSV 匯入則用於批次異動。範圍限制同上——這只會影響你的工作副本；若要改變出廠預設值，請到原始碼在 `REGEX_PRESETS_DEFAULT` 的 `global`（永遠開啟）或特定地區陣列（`us`／`eu`／`uk`／`tw`／`jp`）中加入 `{ type: "TAG_NAME", regex: /您的正規表示式/g, name: "顯示名稱", example: "範例" }`。

### 4.3 新增地區預設集
於 `REGEX_PRESETS_DEFAULT` 與 `buildDefaultRegexPresetsState()` 的初始化邏輯中新增一個鍵值（例如加拿大用 `ca`），並在 HTML 中的 `#regionSelect` 加入對應 `<option>`。新地區規則與現有五組一樣，會疊加於 `global` 之上。`tw`／`jp` 兩組就是照這個擴充點加入的範例，可直接參考它們的寫法。

### 4.4 重新編譯 `style.css`
Tailwind 建置工具就在本 repo 自己的 `dev/` 資料夾內，不依賴任何外部專案。修改 `MaskFirst.html` 的 Tailwind class 後：
```powershell
cd dev
npm install   # 第一次執行才需要
npm run build:css
```
此指令等同於 `tailwindcss -i ./input.css -o ../style.css --minify`，定義於 [dev/package.json](../dev/package.json)。

### 4.5 資料夾結構

```text
MaskFirst/（專案根目錄）
├── MaskFirst.html                 <- MaskFirst 主程式（本文件主題）
├── docs/
│   ├── MaskFirst_README.md        <- 英文版技術文件（主版本）
│   └── MaskFirst_README.zh-TW.md  <- 本文件（繁體中文）
├── README.md / README.zh-TW.md      <- 專案介紹文件
├── LICENSE                          <- MIT 授權
├── .gitignore / .nojekyll
├── style.css                        <- ✅ 已編譯的 Tailwind 備援樣式表（已加入版控）
└── dev/                             <- Tailwind 建置工具
    ├── input.css / tailwind.config.js / package.json
```

> [!NOTE]
> 發佈給使用者時，只需提供 `MaskFirst.html` 與 `style.css` 兩個檔案。`dev/` 資料夾僅供開發使用。

---

## 五、貢獻指南

地區涵蓋範圍刻意限定於美國／歐盟／英國＋全域底層，並非涵蓋所有國家，以避免跨國格式互相誤判（見 2.1 節）。歡迎提交範疇明確的新地區規則庫 PR（附上 regex 與幾個真實範例）、收斂近似的歐盟護照／VAT 規則，或針對特定身分模式新增 Hard Block 詞彙。

---

## 六、關於本專案

* **GitHub**：[oas114/MaskFirst](https://github.com/oas114/MaskFirst)
* **姊妹專案**：[oas114/EduShield](https://github.com/oas114/EduShield) — 台灣教育場域專用版，獨立的 GitHub repo
* **開發者**：OA (oas114)
* **支持開發者**：[Ko-fi](https://ko-fi.com/oasgrow)
* **授權**：[MIT](../LICENSE)
