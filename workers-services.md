# Worker／服務群組寫法建議

- 審查日：2026-11-09（唯讀審查，未改任何程式碼）
- 範圍：SMS、SubsTracker、sc-auto-claim（各以 `git clone --depth 1` 複製到 `/tmp/review-workers/` 閱讀）
- 對照：`~/workspace/github-audit/research/report.md`（2026-11-08 全 repo 健檢）。報告已列的問題（例如 SMS 長簡訊／限流、SubsTracker 續訂無上限迴圈與金額未驗證、sc 流程永遠 exit 0）本文不再重複，只在建議與其相關時引用一句。
- 原則：只提小型、可逐步做的寫法改進，不提大重寫、不新增依賴。SubsTracker 是 fork 翻譯專案，只提安全的小範圍修正。

## SMS（`src/index.js`，952 行，JavaScript，Cloudflare Worker）

現況：Phase 1 P0 加固已上線（速率限制、timing-safe API key 比對、冪等鍵、訊息分段、輸入驗證、CSP），程式品質比 11/08 健檢報告描述時好很多。以下是在此基礎上的寫法建議。

報告已列、不再重複：單檔前後端混雜；前後端兩份 OTP 邏輯（`extractVerificationCode` 與前端 `extractCode`）與兩份 CSV escape——建議方向是抽成共用模組，前端改由 Worker 同源提供同一份 JS，而不是兩邊各自維護。

### SMS-1：路由的 if-chain 先拆成具名 handler，再考慮路由表

- 位置：`src/index.js:738`（`fetch` handler，約 200 行）
- 現在的寫法：一個 `fetch` 函式內用連續 `if (method && pathname)` 分派 7 條路由，每條路由的完整邏輯（驗證、D1、Discord）都寫在 if 區塊裡。
- 推薦的寫法：先做第一步就好——每條路由抽成具名函式，if-chain 只留分派：

```js
// before：if (request.method === "GET" && url.pathname === "/api/messages") {
//   const authError = await requireChiAuth(request); ...
//   try { ... 20 行 D1 查詢 ... } catch ...
// }

// after：
async function handleListMessages(request, env) {
  const authError = await requireChiAuth(request);
  if (authError) return authError;
  // ...原本的 D1 查詢
}
// fetch 內只剩：
if (request.method === "GET" && url.pathname === "/api/messages")
  return handleListMessages(request, env);
```

等 handler 都獨立後，若路由繼續增加，再把分派改成一張小路由表（`{ method, pattern, handler }` 陣列 + 一個帶 `:id` 參數的 match）；不需要為了路由表引入任何路由庫。

- 為什麼：路由邏輯各自獨立後才能單獨看、單獨測，`fetch` 只剩分派意圖。
- 優先級：P1；工作量：M（逐條抽，抽一條測一條）

### SMS-2：`page()` 的整頁 HTML 移到 `src/page.js` 模組

- 位置：`src/index.js:243`（`page()`，約 490 行 HTML/CSS/JS template literal，佔主檔一半以上）
- 現在的寫法：整頁前端（HTML + 內嵌 `<style>` + 內嵌 `<script>`）以 template literal 塞在 Worker 主檔裡，主檔因此佔去三分之一是前端。
- 推薦的寫法：不動前端內容，只搬家——新增 `src/page.js`，匯出 `renderPage(clientId)` 回傳同一段 template literal，`index.js` 改為 `import { renderPage } from './page.js'`；`${SMS_CHI_AUTH_CLIENT_ID}` 改由參數傳入。純 ES 模組 import，不需改 `wrangler.toml`、不需建置步驟。

```js
// src/page.js
export function renderPage(clientId) {
  return String.raw`<!doctype html>... const CHI_AUTH_CLIENT_ID = "${clientId}"; ...`;
}
// src/index.js
import { renderPage } from './page.js';
if (request.method === "GET" && url.pathname === "/") return renderPage(SMS_CHI_AUTH_CLIENT_ID);
```

- 為什麼：後端邏輯與前端頁面分檔後，兩邊各自的修改不再互相干擾，也符合本檔已有「純函式 export 供測試」的慣例。
- 優先級：P2；工作量：M（搬移為主，驗收是頁面照常開啟）

### SMS-3：搜尋輸入加 debounce，不要每次按鍵都全量重繪

- 位置：`src/index.js:644`（`$("search").addEventListener("input", render)`）
- 現在的寫法：每敲一個字就執行 `render()`，用 `innerHTML` 重建全部簡訊卡片（上限 500 則）。
- 推薦的寫法：

```js
// before：
$("search").addEventListener("input", render);

// after：
let searchTimer;
$("search").addEventListener("input", () => {
  clearTimeout(searchTimer);
  searchTimer = setTimeout(render, 150);
});
```

- 為什麼：輸入法組字與快速打字時不會反覆重建幾百個 DOM 節點，手機上尤其有感。
- 優先級：P1；工作量：S

### SMS-4：不要用 `throw` 做預期的流程控制

- 位置：`src/index.js:474`（`load()` 的 401 分支；同樣寫法另見清除處理 `628` 行、刪除處理 `665` 行）
- 現在的寫法：401 時先處理好狀態（清 token、回登入頁、寫提示），再 `throw new Error("授權已失效…")`，只為了跳出函式，靠同一個 `catch` 把訊息寫進狀態列。
- 推薦的寫法：處理完直接 `return`，錯誤訊息留在原地寫，不要繞去 `catch`：

```js
// before：
if (response.status === 401) {
  setAccessToken("");
  showInbox(false);
  $("loginStatus").textContent = "Chi Auth 授權已過期或失效，請重新登入。";
  throw new Error("授權已失效，請重新登入。");
}
// after：
if (response.status === 401) {
  setAccessToken("");
  showInbox(false);
  $("loginStatus").textContent = "Chi Auth 授權已過期或失效，請重新登入。";
  return;
}
```

- 為什麼：`throw` 應保留給真正的例外；用 `return` 讀者一眼就知道「這條路到此為止」。
- 優先級：P2；工作量：S

### SMS-5：`if (!env.SMS_DB)` 的 503 檢查抽成 `requireDb()`，比照現有的 `requireChiAuth()`

- 位置：`src/index.js:756`、`775`、`794`（三處逐字相同的兩行）
- 現在的寫法：三條路由各自重複 `if (!env.SMS_DB) return json({ error: "Database is not configured" }, 503);`。
- 推薦的寫法：仿照本檔已有的 `requireChiAuth()` 慣例：

```js
function requireDb(env) {
  if (env.SMS_DB) return null;
  return json({ error: "Database is not configured" }, 503);
}
// 各路由改為：
const dbError = requireDb(env);
if (dbError) return dbError;
```

- 為什麼：同一種前置檢查只寫一次，之後改訊息或狀態碼不用改三處，且寫法與檔內既有慣例一致。
- 優先級：P2；工作量：S

## SubsTracker（JavaScript + Hono，fork 上游 wangwangit/SubsTracker）

現況：本地只有 1 個 squashed commit，無上游歷史；業務層已有 repo 拆分（`subscriptions.repo.js`）與通知 `ALL_CHANNELS` 註冊表等不錯的寫法。以下只提小範圍寫法修正，不動架構。

報告已列、不再重複：續訂 while 無上限（P0）、金額未驗證（P0）、`Date.now().toString()` 當 ID、通知假成功、`mergeConfig` 未知 key 污染、JWT 自製問題。

### ST-1：四處重複的「到期日推進到今天」抽成一個 helper

- 位置：`src/data/subscriptions.js:174`（建立・農曆）、`:182`（建立・一般）、`:299`（更新・農曆 `do…while`）、`:311`（更新・一般）
- 現在的寫法：同一段 `while (到期日 < 今天) { 到期日 = 加一個週期 }` 寫了四次，農曆與一般各兩次，且更新的農曆版用 `do…while`、其他三處用 `while`，語意相同寫法卻不一致。
- 推薦的寫法：抽一個 helper，四處都呼叫它（報告 P0 要求的迴圈上限正好落在這一個地方加，不用改四處）：

```js
// before（四處之一）：
while (getTimezoneMidnightTimestamp(expiryDate, timezone) < todayMidnight) {
  expiryDate = addCalendarPeriodInTimezone(expiryDate, periodValue, periodUnit, timezone, { endOfMonth });
}
// after（subscriptions.js 頂部附近新增一次）：
function advanceExpiryToPresent(expiryDate, periodValue, periodUnit, timezone, opts = {}) {
  let advanced = expiryDate;
  let steps = 0;
  while (getTimezoneMidnightTimestamp(advanced, timezone) < opts.todayMidnight && steps < 120) {
    advanced = addCalendarPeriodInTimezone(advanced, periodValue, periodUnit, timezone, opts);
    steps++;
  }
  return advanced;
}
// 四處改為：expiryDate = advanceExpiryToPresent(expiryDate, subscription.periodValue, subscription.periodUnit, timezone, { endOfMonth, todayMidnight });
```

農曆版在 helper 內以一個 `advance` 回呼參數帶入即可，不必為農曆另寫第二個 helper。

- 為什麼：續訂推進是全站最危險的一段邏輯，只留一份才不會四處改漏。
- 優先級：P1；工作量：M

### ST-2：`updateSubscription` 的合併三元抽成 `applyDefined()`

- 位置：`src/data/subscriptions.js:363`（`merged` 物件，約 30 行）
- 現在的寫法：`merged` 內十幾個欄位各自寫 `subscription.X !== undefined ? … : existing.X`（另有 `|| existing.X || 預設` 的變體），讀起來全是重複的判斷。
- 推薦的寫法：

```js
// before：
const merged = {
  ...existing,
  name: subscription.name,
  subscriptionMode: subscription.subscriptionMode || existing.subscriptionMode || 'cycle',
  customType: subscription.customType || existing.customType || '',
  category: subscription.category !== undefined ? subscription.category.trim() : existing.category || '',
  // ...十幾個類似欄位
};
// after（檔案頂部加一個小 helper）：
function applyDefined(existing, updates, fields) {
  const out = { ...existing };
  for (const f of fields) if (updates[f] !== undefined) out[f] = updates[f];
  return out;
}
const merged = applyDefined(existing, subscription, ['name', 'subscriptionMode', 'customType', 'notes', 'amount', 'currency']);
// 只有 category（要 trim）、startDate（要轉格式）等少數需要特殊處理的欄位留在 merged 裡個別寫。
```

- 為什麼：合併規則只寫一次，新增欄位時加一個名字就好，不必再複製一條三元。
- 優先級：P2；工作量：M（逐欄位對照，小心 `||` 與 `!== undefined` 的語意差異——空字串與 `0` 在 `||` 下會被當成未提供）

### ST-3：`config.TIMEZONE || 'Asia/Taipei'` 這類預設值字串集中到一處

- 位置：`src/data/config.js:45`（`getConfig`）；散落點如 `src/data/subscriptions.js:154`、`273`、`460`、`611`，`config.TIMEZONE || 'Asia/Taipei'` 全 repo 共 15 處；`|| 'TWD'` 共 12 處
- 現在的寫法：每個用時區的地方都重複 `config.TIMEZONE || 'Asia/Taipei'`；每個預設幣別都重複 `|| 'TWD'`。
- 推薦的寫法：`getConfig` 本來就回傳 `{ ...DEFAULT_CONFIG, ...config }`，而 `DEFAULT_CONFIG.TIMEZONE` 已是 `'Asia/Taipei'`，所以加一個取用點就好：

```js
// config.js 增加：
export function getTimezone(config) {
  return config.TIMEZONE || DEFAULT_CONFIG.TIMEZONE;
}
// 各處改為：const timezone = getTimezone(config);
// 幣別同理，在 subscriptions.js 頂部加：const DEFAULT_CURRENCY = 'TWD';
// 各處改為：subscription.currency || DEFAULT_CURRENCY
```

- 為什麼：預設值改一個地方就好，且萬一哪天預設時區要改，不必全 repo 搜尋取代 27 處。
- 優先級：P1；工作量：S（純機械取代，行為不變）

### ST-4：`test-notification.js` 的 11 個 `else if` 分支改成表驅動

- 位置：`src/api/handlers/test-notification.js:39`–`196`
- 現在的寫法：11 個通知管道各寫一個 `else if (type === '…')` 分支，每支內重複三件事：組 `testConfig`（把 body 欄位覆蓋進 config）、組測試內容字串、呼叫 `sendXxxNotification`；檔案 212 行，近 160 行是重複結構。
- 推薦的寫法：先做 S 級的第一步——把每支都在重複的覆蓋判斷抽成 helper：

```js
// before（每支各寫 2–3 次）：
TG_BOT_TOKEN: typeof body.TG_BOT_TOKEN === 'string' && body.TG_BOT_TOKEN.trim().length > 0 ? body.TG_BOT_TOKEN.trim() : config.TG_BOT_TOKEN,
// after（檔案頂部一次）：
function pickOverride(body, config, key) {
  const v = body[key];
  return typeof v === 'string' && v.trim().length > 0 ? v.trim() : config[key];
}
// 各支改為：TG_BOT_TOKEN: pickOverride(body, config, 'TG_BOT_TOKEN'),
```

第二步（M，有空再做）才是表驅動：把 11 支收成一張 `{ type: { fields: [...], buildContent, send } }` 對照表——`dispatch.js` 的 `ALL_CHANNELS` 註冊表就是站內現成的範例寫法，直接照抄那個模式。

- 為什麼：新增第 12 個管道時加一列就好，不必再複製一整支分支；且測試內容的時間格式目前 11 處各自寫，改格式要改 11 次。
- 優先級：P2；工作量：S（第一步）／M（第二步）

### ST-5：三處函式內 `await import()` 改回頂部靜態 import

- 位置：`src/data/subscriptions.js:83`、`113`、`426`
- 現在的寫法：`getAllSubscriptions`、`syncLegacyReminderFields`、`deleteSubscription` 各自在函式內 `await import('./reminders.repo.js')`。
- 推薦的寫法：改成檔案頂部 `import { listForSubscription, formatRulesSummary, legacyFieldToRule, deriveLegacyFromRules, clearForSubscription } from './reminders.repo.js';`，三處直接呼叫。已確認 `reminders.repo.js` 本身零 import、`subscriptions.repo.js` 亦無 import，不存在循環依賴，不必用動態 import 迴避。
- 為什麼：靜態 import 讓依賴關係在檔頭一目瞭然，少了每次呼叫都重跑模組解析的成本，也少了三處重複的解構。
- 優先級：P2；工作量：S

## sc-auto-claim（JavaScript + Playwright，`claim.js` 432 行）

現況：掃描邏輯已有上限保護（`MAX_SCAN_STEPS`、`MAX_RUNTIME_MS`、兩輪上限）與備援點擊，寫得比一般自動領取腳本嚴謹。以下針對穩健寫法。

報告已列、不再重複：流程永遠 exit 0（外層 catch 只 log 不設結束碼、`tee` 無 pipefail）、成功判定只看點擊數、登入過期無偵測、無 `timeout-minutes`、用 `npm install` 而非 `npm ci`——以下建議是這些問題的具體修法。

### SC-1：讓失敗真的有非零結束碼（整組最值得先做，見文末）

- 位置：`claim.js:415`–`432`（外層 `catch`）；`.github/workflows/daily.yml`（`node claim.js | tee result_A.txt`）
- 現在的寫法：外層 `catch` 只 `console.log("❌ error", err)`，然後照常 `saveState`、`notify`，行程以 exit 0 結束；workflow 又用 `| tee`，即使 `node` 回非零，pipeline 的結束碼也是 `tee` 的 0。
- 推薦的寫法：兩處各加一行：

```js
// claim.js 外層 catch 尾端：
} catch (err) {
  // ...現有的 saveState / notify 不變
  process.exitCode = 1; // 讓 GitHub Actions 判紅，不要只靠 Discord 通知
}
// daily.yml 兩個 Run claim 步驟改為：
run: set -o pipefail; node claim.js | tee result_A.txt
```

- 為什麼：現在每天 3 次、兩個帳號的失敗全部顯示綠燈，問題只能靠人去看 Discord 才會發現；這兩行讓失敗自動變紅。
- 優先級：P1；工作量：S

### SC-2：成功判定改成三態：成功／無獎勵可領／失敗

- 位置：`claim.js:399`–`413`（`isSuccess = claimed >= 1`；報告已列成功判定只看點擊數）
- 現在的寫法：`claimed >= 1` 就是成功，否則失敗——無獎勵可領的正常日子也被判失敗、每天發失敗訊息。
- 推薦的寫法：`anyCandidateThisRound` 已經算好了，直接拿來用：

```js
// before：
const isSuccess = claimed >= 1;
// after：
const outcome = claimed >= 1 ? 'success'
  : anyCandidateThisRound ? 'failure'   // 有候選但一個都沒點到
  : 'no-rewards';                        // 連候選都沒有：今天沒有可領的
// 'no-rewards' 時 state.success 不寫 true 也不寫 false（或另記 lastResult='無獎勵'），
// notify 改發中性訊息或不發失敗訊息；RESULT= 亦區分三種。
```

- 為什麼：「沒東西可領」與「領取失敗」是兩種完全不同的狀態，混成失敗只會讓人久而久之忽略真正的失敗通知。
- 優先級：P1；工作量：M（`notify()` 與 `RESULT=` 解析、`daily.yml` 的 README 更新都要跟著區分三態）

### SC-3：成功路徑與 catch 路徑的收尾抽成一個 `finishRun()`

- 位置：`claim.js:399`–`413`（成功路徑）與 `:415`–`432`（外層 `catch`）
- 現在的寫法：兩段各寫一次 `state.lastClaimAt / lastResult / success / logicVersion / saveState / console.log(RESULT=) / notify`，只有 `lastError` 有無不同。
- 推薦的寫法：

```js
// before：兩段各約 12 行重複的收尾
// after：
async function finishRun(state, { claimed, isSuccess, error }) {
  state.lastClaimAt = new Date().toISOString();
  state.lastResult = isSuccess ? '成功' : '失敗';
  state.success = isSuccess;
  state.logicVersion = 2;
  if (error) state.lastError = String(error.message || error).slice(0, 500);
  else delete state.lastError;
  saveState(state);
  console.log(`RESULT=${isSuccess ? 'success' : 'failure'}`);
  await notify(claimed, ACCOUNT_NAME, isSuccess);
}
// 成功路徑與 catch 各剩一行：await finishRun(state, { claimed, isSuccess });
```

- 為什麼：收尾規則（要存哪些欄位、RESULT 格式、通知時機）只留一份，改規則不會只改到其中一條路。
- 優先級：P1；工作量：M

### SC-4：固定等待改成條件等待

- 位置：`claim.js:119`（`waitForLoadState('networkidle').catch(() => {})` 後又 `waitForTimeout(2000)`）；`login.js:25`（`waitForTimeout(30000)`）
- 現在的寫法：用固定秒數等頁面：`networkidle` 逾時被靜默吞掉後再硬等 2 秒；`login.js` 硬等 30 秒讓使用者手動登入。
- 推薦的寫法：等「代表頁面就緒的具體條件」，而不是等秒數：

```js
// claim.js：等商店頁真正出現內容再掃描，例如
await page.locator('main, [data-testid], article').first().waitFor({ timeout: 15000 });
// 或等候選按鈕首次出現：await page.locator('button').first().waitFor({ timeout: 15000 });
// login.js：等登入完成的訊號，而不是 30 秒
await page.waitForURL(/store\.supercell\.com\/brawlstars/, { timeout: 120000 });
// 或等登入後才有的元素：await page.locator('text=登出').waitFor({ timeout: 120000 });
```

捲動後的 `waitForTimeout(600/700)`（懶加載等待）可改為 `waitForFunction` 監看 `scrollHeight` 變化，但那是第二步，先改上面兩處即可。

- 為什麼：固定等待不是太短（頁面慢時掃到一半）就是太長（每次白白燒掉 CI 分鐘）；條件等待只等到剛好夠。
- 優先級：P2；工作量：M（要先確認哪個元素能代表「已登入／已載入」）

### SC-5：三處靜默吞錯改成至少留一句 log

- 位置：`claim.js:119`（`.catch(() => {})`）、`:304`（`catch {}`）、`:70`–`78`（`notify()` 的 fetch）；另見 `claim.js:10`–`16`（`loadState()`）
- 現在的寫法：
  - `waitForLoadState('networkidle', …).catch(() => {})`——不知道是逾時還是頁面根本沒載入；
  - 備援點擊 `page.evaluate(…).catch {}`——空 catch，連失敗原因都沒有；
  - `notify()` 的 `fetch` 只包 `try/catch`，不檢查 `response.ok`，Discord 回 404/429 也當作通知成功；
  - `loadState()` 解析失敗時靜默回 `{}`，等於把「今天已領過」的紀錄丟掉。
- 推薦的寫法：

```js
// before：
await page.waitForLoadState('networkidle', { timeout: 10_000 }).catch(() => {});
// after：
await page.waitForLoadState('networkidle', { timeout: 10_000 })
  .catch((e) => console.log('networkidle 未達成，繼續掃描：' + (e?.message || e)));

// before：
} catch {}
// after：
} catch (e) { console.log('備援點擊 evaluate 失敗：' + (e?.message || e)); }

// before（notify 內）：
await fetch(DISCORD_WEBHOOK, { method: "POST", ... });
// after：
const res = await fetch(DISCORD_WEBHOOK, { method: "POST", ... });
if (!res.ok) console.log(`Discord notify failed: HTTP ${res.status}`);

// loadState：檔案存在但解析失敗時警告，不要靜默當作全新：
} catch (e) {
  console.log(`狀態檔 ${STATE_FILE} 解析失敗，視為無紀錄：` + (e?.message || e));
  return {};
}
```

- 為什麼：自動化腳本最怕「失敗了但看起來沒事」；留一句 log 是成本最低的除錯線索。
- 優先級：P2；工作量：S

### SC-6：`getGameDay()` 的手動時區算術改用 `Intl.DateTimeFormat`

- 位置：`claim.js:22`–`36`
- 現在的寫法：`now.getTime() + 8*60*60*1000` 再用 `getUTC*` 取值、手動 `setDate(-1)`、手動 `padStart` 組字串。
- 推薦的寫法：

```js
// before：
const taiwanOffset = 8 * 60 * 60 * 1000;
const local = new Date(now.getTime() + taiwanOffset);
const hour = local.getUTCHours();
if (hour < 16) local.setDate(local.getDate() - 1);
return `${local.getUTCFullYear()}-${...}_16`;
// after：
const parts = new Intl.DateTimeFormat('zh-TW', {
  timeZone: 'Asia/Taipei', year: 'numeric', month: '2-digit', day: '2-digit', hour: '2-digit', hour12: false
}).formatToParts(now);
const get = (t) => parts.find((p) => p.type === t).value;
const day = `${get('year')}-${get('month')}-${get('day')}`;
// 16:00 切換日的邏輯不變：hour < 16 時取前一日（以前一日字串為準）
```

（附註：同檔案的 `notify()` 已用 `toISOString()`；`daily.yml` 的 README 更新已用 `toLocaleString('zh-TW', { timeZone: 'Asia/Taipei' })`——站內已有 `Intl` 寫法可照抄。）

- 為什麼：手動加 8 小時再取 UTC 分量是經典的時區 bug 溫床，`Intl` 把時區規則交給平台，程式只剩「16 點換日」這一條業務規則。
- 優先級：P2；工作量：S

## 統計

| Repo | 建議條數 | P1 | P2 |
| --- | --- | --- | --- |
| SMS | 5 | 2（SMS-1、SMS-3） | 3 |
| SubsTracker | 5 | 2（ST-1、ST-3） | 3 |
| sc-auto-claim | 6 | 3（SC-1、SC-2、SC-3） | 3 |
| 合計 | 16 | 7 | 9 |

## 最重要的 3 條建議

1. **SC-1（sc-auto-claim 結束碼 + `pipefail`）**：兩行改動，讓每天 3 次 × 2 帳號的失敗不再顯示綠燈——整組最值得先做的事（見下）。
2. **SC-2（sc-auto-claim 三態結果判定）**：「無獎勵可領」不再被判失敗，失敗通知才重新值得相信。
3. **ST-1（SubsTracker 續訂推進抽 helper）**：全站最危險的邏輯只留一份，且是報告 P0 迴圈上限的唯一正確落點。

## 整組最值得先做的 1 件事

**SC-1：`claim.js` 外層 catch 加 `process.exitCode = 1`，`daily.yml` 兩個 `node claim.js | tee` 步驟前加 `set -o pipefail`。**

理由：這是三個 repo 裡唯一「每天都在發生、但永遠看不出來」的問題——workflow 綠燈不代表領取成功，失敗只能靠人去翻 Discord 才會發現；修法兩行、零風險、立即讓 CI 恢復它本來該有的示警功能。做完之後再做 SC-2（三態判定），否則即使變紅，也會因為「無獎勵可領的日子天天紅」而很快又被忽略。
