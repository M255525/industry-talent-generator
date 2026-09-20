# CLAUDE.md — industry-talent-generator

單檔前端工具（型態仿照 `icap-generator/`）：填寫表單 → 即時規則檢查 → 產生勞動部勞動力發展署「產業人才投資方案（產業人才投資計畫）」開課申請的核心文件**附表二「訓練班別計畫表」**預覽 → 可列印/匯出 PDF、匯出/匯入 JSON，並可選用串接 LLM API 優化敘述段落。無建置步驟、無框架、無 package.json，直接開啟 `index.html`（`file://`）或以靜態伺服器託管即可。

`manual.html` 是操作手冊，創作者（Mark Tsai）證照與經歷內容與 `icap-generator/manual.html`、`sbir-generator/manual.html`、`phoenix-loan-generator/manual.html` 為同一份，更新其中一邊時同步其餘各處。

## 範圍界定（重要，決定了本工具「做什麼、不做什麼」）

依官方《產業人才投資計畫作業手冊》（112年10月19日修正版）：訓練單位在「在職訓練網」資訊系統完成師資設定、場地設定、班級申請等作業後，系統會**自動產出**附表一（單位基本資料）、附表三～七（總表、場地資料表、師資名冊等）——這些是行政報表，不是需要人工撰寫的敘述文件。真正需要撰寫、且會被審查委員逐項評分的核心文件是**附表二「訓練班別計畫表」**（一個班級填一張）的「訓練計畫內容」大段落。**本工具只做附表二**，不複刻附表一、三～七。

`課程平台/ojt-scraper/` 的 `coursedata.json`（產業人才投資方案課程查詢資料）**不是**附表二的內容來源——那是課程核准開班後、對外公開招生查詢頁的營運數字（瀏覽人次、報名人數、學員負擔/政府負擔總額等），完全沒有訓練需求調查、訓練目標、課程大綱等審查用的敘述欄位。本工具內建範例的訓練時數／人數／政府負擔／學員負擔數字經 `課程平台/ojt-scraper/coursedata.json` 的真實分布校準（分職類抓中位數），但範例的敘述內容（訓練需求調查、訓練目標等）皆為原創虛構，非取自 ojt-scraper。

官方審查規則來源：《產業人才投資計畫作業手冊》「七、規劃訓練班次之標準」與「附錄三、訓練經費編列/支用標準」。手冊若有更新，`CATEGORY_TABLE`（業別材料費占比上限）與 `refreshDerived()` 內的數字門檻需要一併更新，並以 ojt.wda.gov.tw 與所屬分署最新公告版本為準。

v1 只涵蓋「新提報開課」情境，不含核准後的訓練計畫變更（附表八）——那是另一種維護性表單，範圍不同，之後如需要可另外擴充。不套用序號授權、不打包 exe、未部署（比照 icap-generator 根目錄版本；符合「實驗性新工具先問過再上線」的既有慣例）。

## 架構

單一 `index.html`：內嵌 CSS/JS、無外部資源、無 fetch（AI 功能與 Word 匯出的 docx CDN 除外）。`data-path` 屬性綁定巢狀 `state`（含布林值的 `<select>` 特殊處理：讀寫時字串"true"/"false"轉真布林）；`localStorage`（key: `industryTalentCourseState`）自動儲存。

- 七個分頁：案件與課程 → 分析 Analyze → 設計 Design → 發展 Develop → 實施 Implement → 評估 Evaluate → 預覽與列印。分頁外殼比照 icap-generator 的 ADDIE 排列，內容依附表二官方段落分組（規劃與執行能力／裝備與設施／訓練模式／訓練績效評估／促進學習機制／訓練費）。
- 動態清單透過 `LIST_CONFIGS` + `renderListEditor()` 通用渲染，共三組：師資（`design.instructors`）、助教（`design.assistants`）、課程大綱（`develop.outline`，逐時段填寫，11 個欄位含日期/時段/時數/學術科/內容/地點/遠距/室外/教師/助教）。
- **規則檢查（`refreshDerived()`）是本工具的核心價值**，比 icap-generator 的語意涵蓋檢查更量化，貼近經費編列規則審查：
  - 受理期別結訓期限（上半年度當年8月底前／下半年度翌年2月底前）
  - 課程大綱時數加總＝訓練總時數
  - 單一時段≤4小時、同日總時數≤8小時（依 `develop.outline` 逐列與依日期分組加總核算）
  - 室外教學時數≤總時數1/5
  - 同一講師授課時數≤54小時（原則性提醒，非硬性擋修）
  - **單一人時成本≤165元／人時**（訓練時數<48小時可提高15%）：`develop.fixed_cost_total / (人數×時數)`
  - **材料費占固定費用比率上限**：依 `meta.category_code` 查 `CATEGORY_TABLE`（19類業別，5%～40%）
  - 每人費用個位數應調整為零、政府80%/學員20%補助比例提示（僅供參考，全額補助對象不適用）
- `CATEGORY_TABLE`：業別分類代碼與材料費占比上限對照表（依附錄三整理，05類拆分05-01美容紓壓40%／05-02傳統整復5%）。
- 預覽（`renderPreview()`）依附表二章節順序輸出（規劃與執行能力／裝備與設施／訓練模式／訓練績效評估／促進學習機制／訓練費／其他），未填欄位顯示紅字 placeholder。列印用 `window.print()` + `@media print` 只印 `#preview`。
- Word (.docx) 匯出：直接走訪已渲染的 `#preview` DOM 轉成 docx 段落/表格（`docx@9.7.1` CDN 懶載入），與 icap-generator 手法完全相同（整段複用，通用 DOM-walker，非另維護樣板）。
- `SAMPLE_LIBRARY` 內建 **5 組完全虛構**的範例，橫跨 5 種業別材料費占比（5%／10%／25%／35%／40%），課程大綱（`OUTLINE_LANG`／`OUTLINE_AI`／`OUTLINE_FOOD`／`OUTLINE_CNC`／`OUTLINE_SPA`）與費用金額均已用 Python 腳本核算，確保：時數加總正確、單一時段與同日時數不超標、同一講師時數不超過54小時、材料費占比在上限內、每人費用個位數為零、政府/學員80/20分攤：
  - `lang`（預設）越南語商務溝通實務班——語文類、45hr、單一講師、無術科場地
  - `ai` AI行銷數據分析與自動化工具應用班——電腦資訊技術類、42hr、雙講師＋助教、含遠距教學列、政策性產業
  - `food` 地方特色伴手禮烘焙實務班——餐飲及食品加工類、54hr、示範目的事業主管機關規定欄位
  - `cnc` CNC銑床智慧製造技術班——工業製造類、48hr、示範室外教學（參觀見習）
  - `spa` 芳療保健與紓壓技法實務班——美容紓壓類（材料費上限最高40%）、60hr、雙講師分授示範54小時上限
- 匯入 JSON 經 `deepMerge(EMPTY_STATE, data)` 以空白骨架補齊缺欄位，無 icap 那種依 case_type 裁剪欄位的匯出轉換（v1 無案件類型分支，匯出即全量 state）。
- **AI 優化（串接外部 LLM API，選用）**：位於「預覽與列印」分頁，實作與 `icap-generator`/`sbir-generator` 相同模式：
  - Claude／OpenAI／Gemini／OpenRouter 瀏覽器直連 `fetch()`；設定存 localStorage（key: `industryTalentApiConfig`），優化前備份到 `industryTalentCourseBackup`。逾時 180 秒；遇暫時性錯誤（429/500/503/529）自動重試最多 2 次（間隔 8、16 秒）。
  - `AI_FIELDS` 定義 15 個可改寫的敘述欄位（訓練需求調查、學員資格條件、訓練目標四段、招訓/遴選/激勵三段、四層次評估說明）；**時數、費用、材料品項、日期、師資助教資格與場地資料一律不交給模型**——這些是官方逐項核算的規則性欄位，改功能時不可放寬。

## 指令

無建置/測試指令。修改 `index.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server 8818 --directory industry-talent-generator` 測完關閉（8818 為工作區目前最大已用埠號 8817 之後第一個空號）。
