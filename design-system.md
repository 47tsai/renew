# 設計系統與模板群組 — 更推薦的寫法審查

- 審查日：2026-11-09
- 範圍：`chi-system`、`chi-design-system`、`example-site`、`47tsai`（各以 `git clone --depth 1` 複製到 `/tmp/review-design-system/`，唯讀審查，未改任何程式碼）
- 定位：補充 2026-11-08 健檢報告（`~/workspace/github-audit/research/report.md`）沒有的東西；報告已列的問題（舊系統無 package.json／測試、圖示四套混用、主題邏輯六次重複、example-site 與舊 example 分叉＋Google Fonts＋熱連結、採用率低等）不重複寫。
- 驗證方式：對比數值以 Python 按 WCAG 相對亮度公式驗算（不四捨五入）；行號皆實讀當下 main 的檔案；`chi-system` 以 repo 根目錄的 SYSTEM.md v1.19／DESIGN.md v2.12 為對照基準。
- 邊界：不提改規範方向或改名（已凍結）、不提大重寫、不提新依賴。

---

## chi-system（6 條主建議）

### 1. 深色模式：危險按鈕與勾選勾記的對比未達標

- **檔案：行號**：`components.css:120-122`、`:302`；`tokens.css:41`、`:58`、`:180`、`:183`、`:192`
- **現在的寫法**：
  ```css
  /* components.css:120-122 */
  .chi-btn--danger {
    background-color: var(--chi-color-danger);
    color: #ffffff;   /* 寫死白字 */
  }
  /* components.css:302（勾選狀態背景圖） */
  background-image: url("data:image/svg+xml,... stroke='white' ...");
  ```
  `tokens.css` 有 `--chi-color-on-accent`（淺色 `:41` `#ffffff`、深色 `:183` `#131519`），但**沒有 `--chi-color-on-danger`**。實測對比（WCAG 公式驗算）：深色 danger `#fc432e`（`:192`）對白字只有 **3.51:1**，低於 AA 文字門檻 4.5:1（淺色 `#d73220` 對白字 4.81:1，通過）；深色 accent `#6b9bff`（`:180`）對白勾記只有 **2.71:1**，低於非文字門檻 3:1（淺色 5.17:1，通過）。
- **推薦的寫法**：新增 `--chi-color-on-danger`（淺色 `#ffffff`、深色 `#131519`，仿 `on-accent` 模式），兩處改引用它：
  ```css
  /* tokens.css：淺色區加 --chi-color-on-danger: #ffffff;
     深色區加 --chi-color-on-danger: #131519; */
  .chi-btn--danger {
    background-color: var(--chi-color-danger);
    color: var(--chi-color-on-danger);
  }
  /* 勾選勾記：深色改用 on-accent 的深色字作 stroke（或同理新增 on-accent 已有，
     直接以 var(--chi-color-on-accent) 帶入 data-URI 的 stroke） */
  ```
  註：舊 `chi-design-system/tokens.css` 本來就有 `--chi-color-on-danger`（淺 `#fff`、深 `#17191f`）——搬的是「token 概念」，色值不搬，沿用新系統已驗算過的 on-accent 色值。Switch 滑塊白底對深色軌道同為 2.71:1，但滑塊靠位置辨認，問題較輕，不列入本條。
- **為什麼**：深色模式下危險操作與勾選狀態的辨識都低於 WCAG 門檻。
- **優先級**：P1　**工作量**：S

### 2. 全站缺 skip link，`<main>` 也沒有 id

- **檔案：行號**：`header.html:100`、`template/index.html:81`、`example.html:84`、`preview.html:96`；`components.css`（無此 class）
- **現在的寫法**：四個頁面的 `<main class="chi-main">` 皆無 `id`；全 repo（HTML、CSS、SYSTEM.md、DESIGN.md）沒有任何 skip link。header 每頁重複四段導覽（品牌、本站導覽、跨站導覽、主題切換），鍵盤使用者每開一頁都要逐一 Tab 過全部導覽才能到內容。
- **推薦的寫法**（舊 repo 有現成寫法可搬，token 名需改新版）：
  ```html
  <!-- header.html / template/index.html：放在 <body> 開頭、header 之前 -->
  <a class="chi-skip-link" href="#main">跳至主要內容</a>
  <main class="chi-main" id="main">
  ```
  ```css
  /* components.css：平時移出畫面，:focus 時出現
     （舊 chi-design-system/components.css:193-194 同款做法） */
  .chi-skip-link { position: fixed; z-index: 200; top: .75rem; left: .75rem;
    transform: translateY(calc(-100% - 1rem)); padding: .7rem 1rem; ... }
  .chi-skip-link:focus { transform: translateY(0); }
  ```
- **為什麼**：WCAG 2.4.1 Bypass Blocks 是 A 級要求，每頁重複導覽卻無跳過機制。
- **優先級**：P1　**工作量**：S

### 3. 表格選取列的 `box-shadow` 在 Firefox 不渲染

- **檔案：行號**：`components.css:539-541`
- **現在的寫法**：
  ```css
  .chi-table tr[aria-selected="true"] {
    background-color: var(--chi-color-surface-sunken);
    box-shadow: inset 3px 0 0 var(--chi-color-accent);
  }
  ```
- **推薦的寫法**：標記改到第一個 `td` 上（Firefox 不渲染 `tr` 上的 box-shadow，是已知行為；清單列是 flex div，不受影響）：
  ```css
  .chi-table tr[aria-selected="true"] td:first-child {
    box-shadow: inset 3px 0 0 var(--chi-color-accent);
  }
  ```
- **為什麼**：同一規則在 Firefox 下選取狀態完全不可見。
- **優先級**：P1　**工作量**：S

### 4. `.chi-link` 預設無底線，違反 SYSTEM.md §11

- **檔案：行號**：`components.css:145-150`；SYSTEM.md §11
- **現在的寫法**：
  ```css
  .chi-link { color: var(--chi-color-accent); text-decoration: none; }
  .chi-link:hover { text-decoration: underline; }
  ```
  SYSTEM.md §11 規定「內文中的連結除顏色外須有底線或其他非顏色區隔」——實作只在 hover 才給底線，靜態下連結只靠顏色區分。
- **推薦的寫法**：
  ```css
  .chi-link { color: var(--chi-color-accent); text-decoration: underline; }
  .chi-link:hover { text-decoration-thickness: 2px; } /* 或維持 hover 加粗／改色 */
  ```
- **為什麼**：規範有寫、實作沒做到；只靠顏色區分連結違反 WCAG 1.4.1。
- **優先級**：P1　**工作量**：S

### 5. 跨站導覽五處手寫清單已漂移，且檢查腳本用硬編碼清單

- **檔案：行號**：`header.html`、`example.html`、`preview.html`、`template/index.html`、`template/states.html`（各六站清單）；`scripts/check-integrity.mjs:36-37`；`scripts/check-breakpoints.mjs:19`
- **現在的寫法**：五個 HTML 各手寫一份六站清單，與「由 `sites.json` 產生，不得手寫站點 URL 清單」的規定矛盾，且已漂移——站名 5/6 與 `sites.json` 不同（HTML：Chi／Free Game／收藏／導覽／工具／狀態；`sites.json`：Chi 主站／Free Game／Chi Merch／Chi Nav／Chi Tools／Chi Status），描述文字亦不同（目前 href 與 icon 還一致）。同時 `check-integrity.mjs:36-37` 把 `HTML_FILES`／`CSS_FILES` 寫死，`check-breakpoints.mjs:19` 把 `cssFiles` 寫死——新增頁面或 CSS 檔會靜默逃過全部引用與斷點檢查。
- **推薦的寫法**：`check-integrity.mjs` 加一項檢查：解析各 HTML 的 `.chi-sites__item`，比對 href／icon／站名是否與 `sites.json` 一致；檔案清單改掃描目錄自動發現（舊 `chi-design-system/scripts/check-integrity.mjs` 用 `git ls-files` 列舉追蹤檔，就是此做法）：
  ```js
  // 取代寫死的 HTML_FILES / CSS_FILES
  import { readdirSync } from 'node:fs';
  const walk = (dir) => readdirSync(dir, { withFileTypes: true })
    .flatMap((e) => e.isDirectory() ? walk(join(dir, e.name)) : [join(dir, e.name)]);
  const HTML_FILES = walk(root).filter((f) => f.endsWith('.html'));
  ```
- **為什麼**：手寫清單已證實會漂移，硬編碼清單讓檢查本身有盲區。
- **優先級**：P1　**工作量**：M

### 6. 對話框：死選擇器＋Focus Management 只有註解、無共享實作

- **檔案：行號**：`components.css:17`、`:604-605`；`DESIGN.md:279`；`example.html:259`、`preview.html:467`
- **現在的寫法**：`:where(...)` 的 focus-visible 清單（`:17`）含 `.chi-dialog__close`，但全 repo 無此 class 定義、無任何 HTML 使用，對話框也無關閉（X）按鈕模式。DESIGN.md:279 與 components.css:604-605 要求「開啟時焦點移入、焦點限制在內、關閉後焦點返回」，但 chi-system 無任何共享 helper——`example.html:259`、`preview.html:467` 的演示用 `showModal()`（原生提供三項行為），正式站若用 `<dialog open>` 或自製對話框即無保障。
- **推薦的寫法**：(a) 定義 `.chi-dialog__close` 或從 `:where` 清單移除；(b) 二選一：文件化「正式站一律用 `showModal()`」，或加小型共享 helper（焦點儲存／還原＋Tab 限制＋Escape），參考實作是 `example-site/app.js` 的 `setDrawer`（previousFocus／Tab trap／Escape 三段式，見下方 example-site 第 2 條）。
- **為什麼**：規範要求的無障礙行為目前只靠註解，無程式保障。
- **優先級**：P2　**工作量**：M

---

## chi-system 次要／順手修（附錄，一行一條，不計入主建議數）

- **表單錯誤範例缺程式關聯**：`example.html:109-112` 的錯誤欄位（`.chi-field--error`＋`<p class="chi-field__error">`）無 `aria-describedby`／`aria-invalid`，全 repo grep 零命中；SYSTEM.md §11 要求錯誤文字以 `aria-describedby` 與欄位關聯。建議：演示中為錯誤 `<p>` 加 id、輸入加 `aria-invalid="true"`＋`aria-describedby`。P2，S。
- **focus-visible 清單漏列 header／連結**：`components.css:15-17` 的 `:where` 清單含 12 個選擇器，漏了 `.chi-header__theme`、`.chi-sites__button`、`.chi-sites__item`、`.chi-link`、頁尾連結；這些改用瀏覽器預設 focus 樣式（§11 允許沿用預設，故非違規，僅一致性問題）。P2，S。
- **check-breakpoints.mjs 穩健性缺口**：`:14` 的 `readdirSync` import 後未用（死 import）；`:42` 的 `@media` 正則 `.` 不跨行，多行條件會被靜默漏檢；未先剔除 CSS 註解，註解內的 `@media` 會誤判；`:19` 只掃三份 CSS，不掃 HTML 內嵌 `<style>`（目前各 HTML 皆無 `@media`，現況無漏洞，屬未來防護）。P2，S。
- **check-integrity.mjs 穩健性缺口**：`:92` 的 `JSON.parse` 無 try/catch，`hashes.json` 損毀時直接拋 stack trace；`:34` 的 `logo/` 列表若含子目錄，後續 `readFileSync` 會拋錯；`:71` 的檔案引用正則只認 `./` 與 `../` 開頭，裸相對路徑（如 `href="RELEASE.md"`）漏檢，且不剔除 `?query`。P2，S。
- **sites.json、breakpoints.json 是單行 JSON**：任何改動都是整行 diff，難以審閱；改 pretty-print（2 空格縮排）。P2，S。
- **載入 spinner 置中 inline style 重複三處**：`template/states.html:85`、`preview.html:369`、`example.html:198` 皆為 `style="margin:0 auto 16px"`；components.css 加 `.chi-state .chi-spinner { margin: 0 auto 16px; }` 收掉。P2，S。
- **CI**：`.github/workflows/check.yml:13-14` 用 checkout@v4／setup-node@v4（Node.js 20 棄用警告，首次 CI 執行已出現，不影響通過）；無 `permissions:` 區塊。建議：升 action 大版、加 `permissions: contents: read`。P2，S。
- **theme.js 兩個相同陣列**：`:31` 的 `MODES` 與 `:34` 的 `ORDER` 內容完全相同（`['system','light','dark']`），可合併為一（`ORDER` 僅 `:128` 用於循環）。P2，S。
- **`.chi-btn--sm` 寫死 32px**：`components.css:81-82` 的 `min-height: 32px`；`tokens.css:141` 有 `--chi-size-control-compact: 36px`（目前只在緊湊密度區塊用）。是否改引用由規範釐清（SYSTEM.md 第 4 節約束對象主要是各站，元件檔內幾何常數是否全數 token 化未明文；顏色類已有 token 模式可循，幾何類列為次要）。P2，S。

**已確認無問題、不列建議**：`preview.html` 對比表 4 列「未達標」是預期設計（正是需要 `-strong` 的依據）；`icons.svg` 87 個 symbol 皆 `viewBox="0 0 24 24"`、無重複 id；六站 icon 與 colorId 皆與 COLOR-REGISTRY 一致；`chi-breakpoint-lg`（1024）已定義但全 CSS 未用，保留觀察；`theme.js` 核心邏輯（循環、無效值回落、儲存失敗、系統變更、按鈕綁定、事件）穩健。

---

## chi-design-system（只寫「值得搬到 chi-system、現在還沒搬」，不在舊 repo 上修）

### 1. 搬檢查：舊檢查腳本有四項新腳本沒有的檢查

- **來源**：`chi-design-system/scripts/check-integrity.mjs`
- **現在的寫法（新腳本缺的）**：舊腳本檢查 (a) HTML `id` 重複、(b) `getElementById('...')` 目標不存在、(c) Emoji 掃描（`\p{Extended_Pictographic}`，呼應 SYSTEM.md §11 禁 Emoji；目前全 repo 零違規，但新 CI 沒有強制檢查）、(d) `icons.svg` symbol id 重複。另舊腳本用 `git ls-files` 列舉追蹤檔（見 chi-system 主建議 5）。
- **推薦的寫法**：四項直接抄進新 `check-integrity.mjs`。
- **為什麼**：四項都是靜態可驗證的既有規則，搬過來即補上新 CI 的盲區。
- **優先級**：P2　**工作量**：S

### 2. 搬文件：圖示清單表的格式

- **來源**：`chi-design-system/ICON-GUIDELINES.md`「現有圖示清單與語意」表
- **現在的寫法**：新系統 87 個 symbol 完全沒有用途清單——要挑圖示只能去翻 `icons.svg` 原始檔或猜 Lucide 名；舊文件有「Symbol ID → 標準用途」表＋「先查再新增／不造別名」規則，舊檢查腳本還曾強制清單與 sprite 逐一對應。
- **推薦的寫法**：在 chi-system 立 `icons.md`（或同等文件），列 87 個 symbol 的用途表（先以 Lucide 原名自動生成骨架，再補常用者的用途）；檢查腳本核對清單與 `icons.svg` 一致。
- **為什麼**：沒有清單，新增圖示前「先查再新增」做不到，遲早收錄重複語意的圖示。
- **優先級**：P2　**工作量**：M

### 3. 搬元件概念（只搬概念與原則，不搬色值與舊 class 實作）

- **來源**：`chi-design-system/design-system.yml` 的 `components` 節＋`components.css`
- **現在的寫法**：新系統 DESIGN.md 7.7 未列以下三類，舊系統已有現成定義與原則：`.chi-alert`（四語意變體，info／success／warning／danger；原則：語意色搭配文字訊息，不只靠顏色）、`.chi-toast`＋`.chi-toast-region`（短暫、非阻斷性回饋；重要錯誤留在頁面中；舊 `example/framework.js` 有 `role="status"`＋2.8 秒自動移除的實作參考）、`.chi-setting-row`／`.chi-account-row`（設定頁：標題與描述在一側、狀態或操作在另一側，適合帳號中心與設定頁）。
- **推薦的寫法**：先補 DESIGN.md 規格（變體、色值、無障礙行為）再實作，不直接複製舊 CSS。
- **為什麼**：三類都是各站遲早需要的回饋／設定型元件，舊系統已驗證過需求與原則。
- **優先級**：P2　**工作量**：M

**已列入 chi-system 主建議、不在此重複的舊 repo 來源**：`--chi-color-on-danger`（舊 `tokens.css` 已有此 token）→ 主建議 1；`.chi-skip-link`（舊 `components.css:193-194`）→ 主建議 2。

**明確不搬**：舊色值（已由新 `tokens.css` 重新定案）、`--chi-color-accent #4a88fc` 作主色（新系統已證實對比不足）、`760px` 斷點、`data-theme` 命名、sidebar 版面、漸層／玻璃擬態風格。

---

## example-site（只寫「值得搬到 chi-system、現在還沒搬」）

### 1. `theme-color` meta＋JS 同步更新

- **來源**：`example-site/index.html:7`、`app.js:33`
- **現在的寫法**：新系統全 repo 無 `theme-color`，手機瀏覽器網址列顏色不會跟主題走。舊寫法：
  ```html
  <!-- index.html:7 -->
  <meta name="theme-color" content="#f4f7fc">
  ```
  ```js
  // app.js:33（renderTheme 內）
  document.querySelector('meta[name="theme-color"]').content = dark ? "#11151c" : "#f4f7fc";
  ```
- **推薦的寫法**：`header.html`／`template/index.html` 加 `<meta name="theme-color">`，`theme.js` 的 apply() 內同步改 `content`（淺色取 canvas `#f7f8fa`、深色 `#131519`）。
- **為什麼**：一行 meta＋三行 JS，手機瀏覽器 chrome 顏色才會跟主題一致。
- **優先級**：P2　**工作量**：S

### 2. 抽屜／對話框焦點管理 pattern（現成且正確的參考實作）

- **來源**：`example-site/app.js` 的 `setDrawer`（約 `:45-75`）
- **現在的寫法**：新系統對話框 Focus Management 只有註解、無共享實作（見 chi-system 主建議 6）。舊寫法一次做齊五件事：開啟時儲存 `previousFocus` 並把焦點移入關閉鈕、關閉後焦點還原、Tab trap（首尾循環）、Escape 關閉、scrim 點擊關閉、`aria-hidden`／`aria-expanded` 同步。
- **推薦的寫法**：改寫為新系統對話框 Focus Management 的小型共享 helper（不搬 sidebar／scrim 本身）。
- **為什麼**：不用從零設計焦點陷阱，舊程式已是驗證過的正確形狀。
- **優先級**：P2　**工作量**：M

**明確不搬**：sidebar 版面、收合按鈕、`760px` 斷點（與凍結的 768 衝突）、`data-theme`／`data-theme-mode` 命名、手寫 inline SVG 圖示、raw.githubusercontent.com 熱連結 logo。

---

## 47tsai（profile repo，空 repo）

一句話建議：目前是空 repo、無 README，補一頁簡短 profile README（姓名、身份、主要連結）即可；不補也不影響任何功能。

---

## 最重要的 3 條建議（跨 repo）

1. **chi-system 主建議 1**：深色 danger 按鈕 3.51:1、checkbox 勾記 2.71:1 皆未達 WCAG 門檻，補 `--chi-color-on-danger`（舊 repo 已有此 token 概念可參考）。
2. **chi-system 主建議 2**：全站缺 skip link（WCAG 2.4.1 A 級），舊 repo 有現成寫法可搬。
3. **chi-system 主建議 5**：五處手寫跨站導覽已漂移（站名 5/6 與 `sites.json` 不同），用檢查腳本鎖住一致性，否則漂移只會擴大。

## 整組最值得先做的 1 件事

**chi-system 主建議 1（`--chi-color-on-danger`）**：最小改動、一次修兩個未達標的對比、且舊 repo 已證明此 token 模式可行；做完後深色模式的危險操作與勾選狀態才真正可辨識。
