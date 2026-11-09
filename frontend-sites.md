# 前端網站群組：推薦寫法審查

- 審查日：2026-11-09；方式：唯讀審查，`git clone --depth 1` 到 `/tmp/review-frontend/`，未改任何程式碼。
- 範圍：merch、47tsai.github.io（個人站）、nav、h，共 4 個 repo。
- 本文件補充 2026-11-08 健檢報告（`~/workspace/github-audit/research/report.md`）沒有的東西：**程式寫法層面的推薦**。報告已列的問題不重複寫，只在相關時引用一句。
- 邊界：設計系統統一（主題、圖示、header、跨站導覽、共用 CSS）正在 chi-system 另一條線進行，不提；不提換框架等大重寫；不提新依賴（除非理由很充分）；只提小型、可逐步做的改進。
- 每條格式：檔案:行號、現在寫法、推薦寫法（附 before/after 示意）、為什麼（一行）、優先級（P1 值得做／P2 有空再做）、工作量（S 小／M 中／L 大）。

## 對健檢報告的一處更正

報告寫 nav「搜尋無 ranking」，實際上有：`nav/js/search.js:4` 的 `searchLinks` 有加權評分（標題完全相符 1000、開頭 400、單字開頭 260、包含 180、描述 55、分類 45、關鍵字 35、網域 20）。以下 N1 是針對它的效能寫法，不是補 ranking。

---

## merch（TypeScript + Vite + Cloudflare Worker）— 6 條

### M1. `src/main.ts` 是 225 行的 god module，路由、渲染、搜尋、modal、分頁、統計全混在一起

- **現在的寫法**：`src/main.ts`（225 行，但每行極長）包辦全站行為；內部自定 `$`／`$$`、`escapeHtml`、`money()`，與 `src/utils/dom.ts`、`src/utils/format.ts` 已有的工具重複。
- **推薦的寫法**：先不動行為，只把 `main.ts` 按「頁面」拆檔：`home.ts`（首頁＋統計）、`collection.ts`（搜尋＋篩選＋分頁）、`detail-modal.ts`（modal），`main.ts` 只留路由分派（判斷目前是哪頁、呼叫對應的 `init`）。拆一頁、測一頁，不用一次拆完。

```ts
// before：main.ts 內混雜
setupCollection(); renderHome(); renderStatistics(); /* …全在同一檔 */

// after：main.ts 只做分派
import { initCollectionPage } from './pages/collection';
if (document.querySelector('[data-page="collection"]')) initCollectionPage(store);
```

- **為什麼**：現在改一個功能要先讀完整檔才敢動，拆開後每次只讀一頁的邏輯。
- **優先級**：P2；**工作量**：M（可一頁一頁拆，不用一次做完）。

### M2. `escapeHtml` 寫了 8 份，`Intl.NumberFormat('zh-TW')` 寫了 6 處

- **現在的寫法**：`escapeHtml` 各寫一份在 `src/utils/dom.ts`、`src/main.ts`、`src/statistics.ts`、`src/shipping.ts`、`src/shipping-detail-modal.ts`、`src/item-detail-shipping.ts`、`src/home-enhancements.ts`、`src/works-management.ts`；`money()`／`formatMoney` 每次呼叫都 `new Intl.NumberFormat('zh-TW')`（`src/utils/format.ts:2`、`src/shipping.ts:23`、`src/shipping-detail-modal.ts:9`、`src/main.ts:24`、`src/item-detail-shipping.ts:14`、`src/home-enhancements.ts:6`）。
- **推薦的寫法**：以 `src/utils/dom.ts` 的 `escapeHtml` 和 `src/utils/format.ts` 的 `formatMoney` 為唯一版本，其他 7 處刪掉改 import；formatter 在模組頂層建一次共用。

```ts
// before：src/main.ts:24 每次呼叫都新建
function money(n: number) { return new Intl.NumberFormat('zh-TW').format(n); }

// after：src/utils/format.ts 頂層建一次
const twd = new Intl.NumberFormat('zh-TW');
export const formatMoney = (n: number) => twd.format(n);
```

- **為什麼**：同一邏輯 8 份拷貝，修一個 bug 要改 8 個地方，遲早漏改。
- **優先級**：P1；**工作量**：S（純機械式取代，行為不變）。

### M3. `src/store.ts:69` 每次開頁對 148 件收藏做 150+ 次循序請求，且全部 `no-store`

- **現在的寫法**：`loadNewStaticData` 把 `categories.json` 的 fetch 寫在 `for (const entry of index.works)` 迴圈內（同一檔案每件作品重抓一次）；之後每類別 `index.json`、再每件 `data.json` 全部 `await` 循序抓；`src/` 內 `cache: 'no-store'` 共 7 處（`store.ts`、`api.ts`、`version.ts:10`），每次開頁全部重抓。
- **推薦的寫法**：三步，各自獨立可做：(1) `categories.json` 移到迴圈外抓一次；(2) 同層級的 `data.json` 改 `Promise.all` 平行抓（148 件循序 → 一批平行，失敗的個別處理，不要整批失敗）；(3) `no-store` 改 `no-cache`（可用 ETag revalidate，沒變就回 304）或預設快取，搭配 `version.json` 判斷是否真的需要重抓。

```ts
// before：迴圈內重抓 + 循序
for (const entry of index.works) {
  const categories = await fetch('categories.json'); // 每件都抓一次
  const data = await fetch(`works/${entry.id}/data.json`);
}

// after：抓一次 + 平行
const categories = await fetch('categories.json');
const results = await Promise.allSettled(
  index.works.map((entry) => fetch(`works/${entry.id}/data.json`).then((r) => r.json())),
);
```

- **為什麼**：這是四個站裡對使用者體感影響最大的效能問題，開頁速度直接差一個量級。
- **優先級**：P1；**工作量**：M。

### M4. `src/store.ts` 的 `loadStore` 用空 `catch` 靜默退回靜態資料

- **現在的寫法**：`try { remote } catch { /* API 尚未切換時使用靜態來源。 */ }`；`src/api.ts:67` 的 `getRemoteData` 也有 `catch {}`；Worker 回 500 也會被當成「尚未切換」默默吃舊資料，使用者看到舊資料卻不知道同步已壞。
- **推薦的寫法**：catch 裡至少 `console.warn` 並區分錯誤種類：網路失敗／404（API 真的還沒切換）才退回靜態；500 或回應格式錯誤則顯示「同步失敗，顯示的是舊資料」提示，不要無聲。

```ts
// before
try { data = await getRemoteData(); } catch { /* API 尚未切換時使用靜態來源。 */ }

// after
try { data = await getRemoteData(); }
catch (error) {
  console.warn('[store] 遠端資料失敗，改用靜態資料', error);
  if (error instanceof ResponseError && error.status >= 500) showNotice('同步失敗，顯示舊資料');
  data = await getStaticData();
}
```

- **為什麼**：靜默退回會讓壞掉的同步長期沒人發現，出問題時也無從查起。
- **優先級**：P1；**工作量**：S。

### M5. 驗證邏輯有三套，`fetch` 超時樣板寫了兩份

- **現在的寫法**：`src/api.ts` 的 `request()` 與 `getAsset()` 各寫一份近乎相同的 AbortController＋12 秒 timeout 樣板；驗證有三套並存：`api.ts` 的 `validateItem`／`validateWork`／`validateShippingList`、`src/validation.ts` 的 `isValidItem`／`isValidShipping`（本身是一整行超長布林表達式）、`src/store.ts` 的 `validateItem`（手寫逐欄位檢查＋重建物件）。
- **推薦的寫法**：(1) 超時抽成一個 `fetchWithTimeout(url, ms)`，兩處共用；(2) 驗證只留 `validation.ts` 一套，其他兩處改呼叫它（`store.ts` 的「重建物件」若需要，拆成獨立的 `normalizeItem`，不要和驗證綁在一起）；(3) `isValidItem` 的超長單行布林拆成逐欄位 `if` 提早回傳，錯誤時才知道是哪個欄位不過。

```ts
// before：validation.ts 一整行
export const isValidItem = (x: unknown): boolean => isObj(x) && typeof x.id === 'string' && /* …十幾個條件同一行 */;

// after：逐欄位提早回傳
export function isValidItem(x: unknown): x is Item {
  if (!isObj(x)) return false;
  if (typeof x.id !== 'string' || !x.id) return false;
  if (typeof x.price !== 'number' || x.price < 0) return false;
  return true;
}
```

- **為什麼**：三套驗證遲早互相矛盾（一個說合法、一個說不合法），到時不知道該信哪個。
- **優先級**：P2；**工作量**：M。

### M6. 搜尋沒有 debounce，每按一鍵全站重渲染一次

- **現在的寫法**：`main.ts` 的 `setupCollection` 在 `input` 事件直接 `store.setUi({ collectionQuery })`；每次按鍵觸發 subscribe，全部重跑 `renderHome`＋`renderCollection`＋`renderStatistics`＋`syncDynamicCollectionOptions`（重建全部篩選 `<select>` 的 innerHTML）。
- **推薦的寫法**：(1) 輸入加 150–200ms debounce（等使用者停下來再搜尋）；(2) 渲染拆開：只有搜尋結果區跟著 `collectionQuery` 變，統計數字和篩選 `<select>` 只有在資料或篩選條件變時才重建。

```ts
// before
input.addEventListener('input', () => store.setUi({ collectionQuery: input.value }));

// after
let timer: number | undefined;
input.addEventListener('input', () => {
  clearTimeout(timer);
  timer = window.setTimeout(() => store.setUi({ collectionQuery: input.value }), 150);
});
```

- **為什麼**：打字時每鍵都重建整頁是最容易感知的卡頓，修起來只要幾行。
- **優先級**：P1；**工作量**：S。

> 其他次要觀察：27 支 `scripts/verify-*.mjs`＋27 個 `package.json` 的 `verify:*` 腳本無共用 helper、無總入口，`npm run verify` 只跑其中一支（`verify-data-layout.mjs`），`.github/workflows/verify.yml` 逐條列出約 30 個步驟——建議抽一個 `verify-all.mjs` 當唯一入口（或把共用讀檔／比對邏輯抽到 `scripts/lib/`），CI 與本機都只跑它。P2／S。

---

## nav（原生 HTML/CSS/JS + Worker）— 5 條

### N1. 搜尋每次按鍵重算全部連結的小寫 haystack＋`new URL()`，方向鍵還會重算兩次

- **現在的寫法**：`js/search.js:4` 的 `searchLinks` 每次呼叫都對全部連結重做 `toLowerCase()`、組 haystack、`new URL(link.url)` 取 domain，再排序；`js/search.js:11` 的搜尋 `input` 事件無 debounce，每鍵全量重算＋重建 `main.innerHTML`；`js/search.js:10` 的 command palette 方向鍵處理先呼叫一次 `commandItems(input.value)`（只用來取長度），接著 `renderCommandResults()`（`search.js:8`）又呼叫一次，等於每按一次方向鍵全量搜尋跑兩遍。
- **推薦的寫法**：(1) 載入資料後預先算好每條連結的 `haystack` 和 `domain` 存回連結物件上（`link._haystack`／`link._domain`），`searchLinks` 只做比對和評分；(2) 搜尋輸入加 150ms debounce；(3) 方向鍵只移動 `commandIndex` 後重繪，不要重算 `commandItems`——把上次算好的結果存在變數裡，輸入變了才重算。

```js
// before：search.js:10 方向鍵
input.addEventListener('keydown', e => {
  const items = commandItems(input.value);          // 第 1 次全量搜尋
  if (e.key === 'ArrowDown') { commandIndex = Math.min(commandIndex + 1, items.length - 1); renderCommandResults(); } // 內部第 2 次
});

// after：只移動索引、重繪
let cachedItems = [];
input.addEventListener('input', () => { cachedItems = commandItems(input.value); renderCommandResults(cachedItems); });
input.addEventListener('keydown', e => {
  if (e.key === 'ArrowDown') { e.preventDefault(); commandIndex = Math.min(commandIndex + 1, cachedItems.length - 1); renderCommandResults(cachedItems); }
});
```

- **為什麼**：連結數一多，打字和按方向鍵都會明顯卡；預先算好 haystack 是一次性成本。
- **優先級**：P1；**工作量**：M。

### N2. `js/app.js:7` 的 `collectData` 是 O(n²)，而且檢查本身就是多餘的

- **現在的寫法**：`flat(s).forEach(l => { const k = key(l); if (!map.has(k)) map.set(k, l); if (!all.some(x => key(x) === k)) all.push(l) })`——每條連結都對整個 `all` 跑一次 `some`＋`key()`（`key()` 內含 `new URL()` 與參數排序），但同一行前面已經有 `map.has(k)` 在做去重判斷。
- **推薦的寫法**：直接用 `map` 的判斷結果決定要不要 push，刪掉 `all.some` 那段。

```js
// before
flat(s).forEach(l => { const k = key(l); if (!map.has(k)) map.set(k, l); if (!all.some(x => key(x) === k)) all.push(l); });

// after
flat(s).forEach(l => { const k = key(l); if (!map.has(k)) { map.set(k, l); all.push(l); } });
```

- **為什麼**：同一行裡 `map` 已經知道這條去重過了，`some` 是白跑的 O(n²)。
- **優先級**：P1；**工作量**：S。

### N3. `js/app.js:14` 的 `handleDragOver` 每個 dragover 事件都全頁掃描

- **現在的寫法**：每次 `dragover` 事件都跑 `document.querySelectorAll('.drag-over').forEach(x => x.classList.remove('drag-over'))` 再加到新卡片上；拖曳時 dragover 每秒觸發幾十次。
- **推薦的寫法**：用一個變數記住目前帶 `.drag-over` 的卡片，只有換卡片時才移除舊的、加上新的。

```js
// before
function handleDragOver(e) { const card = e.target.closest('.sortable-card'); if (!card) return; e.preventDefault(); document.querySelectorAll('.drag-over').forEach(x => x.classList.remove('drag-over')); card.classList.add('drag-over'); }

// after
let dragOverCard = null;
function handleDragOver(e) { const card = e.target.closest('.sortable-card'); if (!card || card === dragOverCard) return; e.preventDefault(); dragOverCard?.classList.remove('drag-over'); dragOverCard = card; card.classList.add('drag-over'); }
```

- **為什麼**：拖曳是最頻繁的事件之一，全頁掃描每次都做是白白耗效能。
- **優先級**：P2；**工作量**：S。

### N4. `css/redesign.css`（26KB）沒有任何頁面引用，loading 動畫樣式散在 4 個檔案

- **現在的寫法**：`redesign.css`（26,330 位元組）沒有任何 HTML／JS 引用，只有 `changelog.json` 裡的舊 commit 訊息提到它；`app-loading` 相關樣式同時出現在 `css/enhancements.css`、`css/loading.css`、`css/nav.css`、`css/style.css` 四處（`loading.css` 有完整一套 orbit／ring 動畫，`style.css` 另一套 breathe／spin，`nav.css` 只有 1 條，`enhancements.css` 只有 spinner）。
- **推薦的寫法**：確認 `redesign.css` 無引用後直接刪除（git 歷史還留著）；loading 動畫只留一份 canonical（`loading.css` 那套最完整），其他檔案裡重複的 `app-loading*` 規則刪掉，註明「loading 樣式唯一來源是 `loading.css`」。
- **為什麼**：26KB 死檔白佔 repo，loading 樣式四份拷貝改一處漏三處，遲早長得不一樣。
- **優先級**：P2；**工作量**：S。

### N5. `data.json` 每次開頁都 `no-store` 重抓，`version.json` 還加時間戳 cache-buster

- **現在的寫法**：`js/app.js` 的 `loadNavPayload` 依序試 `data/data.json` 與 `../data/data.json`，都用 `cache: 'no-store'`；`js/update.js` 的 `checkVersion` 用 `version.json?t=${Date.now()}`＋`no-store`。
- **推薦的寫法**：`data.json` 改 `cache: 'no-cache'`（瀏覽器可用 ETag revalidate，沒變回 304，不必每次下載全文）；`version.json` 拿掉 `?t=` 時間戳——`no-cache` 已保證每次都向伺服器確認，時間戳只會讓瀏覽器連 304 的機會都沒有。`sw.js` 的 network-first 策略已會在背景更新快取，不需要 fetch 層再繞過快取。
- **為什麼**：`no-store` 讓每次開頁都全量下載資料，手機網路下最有感。
- **優先級**：P2；**工作量**：S。

> 其他次要觀察：`js/card-tooltip.js` 的網址 tooltip 只在 `mouseover` 時出現，鍵盤 focus 的使用者看不到——補一個 `focusin` 監聽共用同一段邏輯（P2／S）。`index.html` 底部有一段 inline script 處理手機選單與 command trigger，與 `js/navigation.js` 的 `initSidebarToggle` 分散兩處——建議搬進 `navigation.js`，讓行為集中在 js 檔、HTML 只留結構（P2／S）。`key(l)`（URL 正規化）在渲染每張卡時重複計算，可在 `collectData` 時算一次存成 `l._key`（P2／S）。

---

## h（Astro 7 + Svelte 5 + Worker，H Tools）— 5 條

### H1. `src/components/AppShell.svelte` 858 行，5 個功能區＋全部狀態擠在同一元件

- **現在的寫法**：`AppShell.svelte` 包含編號轉換、歷史紀錄、我的收藏、碼錶、設定 5 個區，`let` 狀態約 25 個（`converterInput`、`records`、`favorites`、`favoriteQuery`、`recordSort`……），所有事件處理函式都在同一檔。
- **推薦的寫法**：按區拆成子元件，一次拆一區：`ConverterPanel.svelte`、`HistoryPanel.svelte`、`FavoritesPanel.svelte`、`StopwatchPanel.svelte`、`SettingsPanel.svelte`，`AppShell` 只留導覽列、主題切換、同步狀態與各區共用的 `records`／`favorites`／`resultHistory` 狀態，用 props 傳下去、事件回呼傳上來。Svelte 5 的寫法不變（`let` 即 state，`$derived` 照用），只是檔案變小。

```svelte
<!-- before：AppShell.svelte 內 -->
let converterInput = ''; let favoriteQuery = ''; let records: StopwatchRecord[] = [];
<!-- …5 個區的模板與函式全在同一檔 -->

<!-- after：AppShell.svelte 只留外殼 -->
<script>import ConverterPanel from './ConverterPanel.svelte'; /* … */</script>
{#if activeSection === 'converter'}<ConverterPanel bind:converterInput oncopy={copyConverterOutput} />{/if}
```

- **為什麼**：858 行的元件沒人敢重構，拆成 5 個後每次改一個功能只讀一個檔。
- **優先級**：P1；**工作量**：M（一次拆一區，每區獨立驗證；全拆完是 L）。

### H2. 寫得比較好的轉換器是死碼，上線用的反而是寫死長度對照表的那個

- **現在的寫法**：`src/lib/converter.ts` 有完整的 `SourceAdapter` 註冊表＋`convertInput`（逐站驗證 id、錯誤碼 `empty-input`／`unknown-source`／`ambiguous-source`／`invalid-id`／`invalid-generated-url`，並擋不安全網址），但全站只有 `converter.test.ts` 在用它；`AppShell.svelte` 實際用的是 `src/lib/converter-sources.ts` 的 `buildConversionResults`，用寫死的長度對照表決定來源（`{ 5: ['W'], 6: ['N', 'W'], 7: ['JM'], 8: ['P'] }`）。後者有兩個洞：使用者手選文字來源（M／J）卻輸入數字時，`filter` 會靜默過濾掉，結果是空清單、沒有任何錯誤提示；新增來源要同時改長度表和來源表。
- **推薦的寫法**：把 `converterSources` 的 6 個來源改寫成 `SourceAdapter`（每站實作 `matchesInput`／`extractId`／`isValidId`／`buildUrl`），`AppShell` 改呼叫 `convertInput`；`buildConversionResults` 的「一次顯示多站結果」需求，用「對全部 adapter 跑 `matchesInput`，收集所有通過的」來實現，而不是查長度表。`converter-sources.test.ts` 的既有測試先留著當對照，確認輸出網址不變再刪舊函式。

```ts
// before：converter-sources.ts 用長度猜來源
const numericSources = ({ 5: ['W'], 6: ['N', 'W'], 7: ['JM'], 8: ['P'] })[id.length] ?? [];

// after：converter.ts 的 adapter 自己判斷
const adapters: SourceAdapter[] = [nhentaiAdapter, wnacgAdapter /* … */];
const result = convertInput(input, adapters, selectedSourceId); // 失敗時回傳錯誤碼 + 中文訊息
```

- **為什麼**：測試覆蓋最完整的轉換邏輯沒上線，上線的卻用長度猜來源、錯了還不報錯。
- **優先級**：P1；**工作量**：M。

### H3. 前端和 Worker 各寫一份同步文件驗證，`isSafeHttpUrl` 寫了 4 份，遲早漂移

- **現在的寫法**：`src/lib/sync.ts` 的 `isValidSyncDocument` 與 `worker/src/index.js` 的 `validateDocument` 是同一份 schema 的兩份實作（TS 一份、JS 一份，連 `hasOnlyKeys` 都各寫一次）；`isSafeHttpUrl` 出現在 `src/lib/sync.ts:47`、`src/lib/storage.ts:340`、`src/lib/converter.ts:54`（名為 `safeHttpUrl`）、`worker/src/index.js:84`，共 4 份。
- **推薦的寫法**：(1) 前端三份 `isSafeHttpUrl` 合併成 `src/lib/urls.ts` 一個匯出，三處改 import（Worker 那份因部署獨立先留著）；(2) 前後端驗證邏輯的漂移用測試鎖住：建一個共用的 fixture 檔（例如 `test/fixtures/sync-documents.json`，含一份合法文件＋每種非法變體），`src/lib/sync.test.ts` 與 `worker/src/index.test.js` 都讀同一份 fixture 斷言，任一邊改嚴或改鬆都會立刻失敗。

```ts
// before：兩邊各寫一次，改一邊不會有人提醒另一邊
// sync.ts: isValidSyncDocument(...)   worker/index.js: validateDocument(...)

// after：共用 fixture 鎖住行為
// test/fixtures/sync-documents.json：{ valid: {...}, invalid: [{ name: '重複 id', doc: {...} }, ...] }
// sync.test.ts 與 worker/index.test.js 各自：for (const c of fixture.invalid) expect(validate(c.doc)).toBe(false)
```

- **為什麼**：同步是 h 最關鍵的功能，前後端驗證一旦不一致，合法資料會被一邊拒收、非法資料被另一邊放行。
- **優先級**：P1；**工作量**：S（抽共用函式＋fixture，邏輯本身不動）。

### H4. `worker/src/index.js` 的 `mergeDocuments` 裡，favorites 和 resultHistory 的「按 URL 去重」是兩份近乎相同的程式碼

- **現在的寫法**：`mergeDocuments` 內 favorites 與 resultHistory 各有一段 `byUrl` 去重迴圈，只差時間戳欄位（`updatedAt` vs `lastActionAt`）與合併時保留哪個 id 的細節，約 15 行重複兩次。
- **推薦的寫法**：抽成 `dedupeByUrl(items, getTimestamp)` helper，兩處傳入不同的取時間函式；合併後「保留較新者、id 取較舊者、`createdAt` 取較小者」的規則寫一次。

```js
// before：兩段 byUrl 迴圈，邏輯相同、欄位不同
// favorites: byUrl … item.updatedAt … ; resultHistory: byUrl … item.lastActionAt …

// after
function dedupeByUrl(items, getTimestamp) {
  const byUrl = new Map();
  for (const item of items) {
    const current = byUrl.get(item.url);
    if (!current || getTimestamp(item) > getTimestamp(current)) {
      byUrl.set(item.url, current ? { ...item, id: current.id, createdAt: Math.min(item.createdAt, current.createdAt) } : item);
    } else {
      byUrl.set(item.url, { ...current, createdAt: Math.min(item.createdAt, current.createdAt) });
    }
  }
  return [...byUrl.values()];
}
```

- **為什麼**：去重規則改一邊漏另一邊，同步後會出現同一 URL 兩筆資料。
- **優先級**：P2；**工作量**：S。

### H5. `worker/src/index.js` 的 `githubRequest` 沒有超時，重試也沒有退避

- **現在的寫法**：`githubRequest` 的 `fetch` 沒帶 timeout，GitHub 掛住時 Worker 會等到平台時限；`MAX_RETRIES = 3` 的衝突重試是立即重跑，沒有任何延遲。
- **推薦的寫法**：fetch 加 `signal: AbortSignal.timeout(10_000)`，超時當成 `GITHUB_UNAVAILABLE` 回 502；重試間隔加指數退避（例如 200ms → 500ms → 1s），避免 GitHub 剛衝突完又立刻被打。

```js
// before
const response = await fetch(url, { method, headers: { ...headers } });

// after
const response = await fetch(url, { method, headers: { ...headers }, signal: AbortSignal.timeout(10_000) });
// 重試迴圈內：await new Promise(r => setTimeout(r, 200 * 2 ** attempt));
```

- **為什麼**：無超時的對外請求是 Worker 被拖垮最常見的原因；立即重試則會把短暫衝突打成持續衝突。
- **優先級**：P2；**工作量**：S。

> 其他次要觀察：`AppShell.svelte` 的 `copyConverterOutput` 用 `for...await` 逐筆寫歷史紀錄，改 `Promise.allSettled` 平行寫、再統一檢查失敗（P2／S）。另提醒：h Worker「無驗證」是健檢報告已列的 P0，不在此重複。

---

## 47tsai.github.io（Astro 7 + Svelte 5 + Tailwind 4，個人站）— 5 條

### C1. `removeFileExtension` 兩個檔案寫得一字不差，`toAbsoluteUrl` 也有兩份

- **現在的寫法**：`src/utils/url-utils.ts:12` 與 `src/utils/permalink-utils.ts:4` 的 `removeFileExtension` 連正則都完全相同（`id.replace(/\.(md|mdx|markdown)$/i, "")`）；`toAbsoluteUrl` 在 `src/utils/url-utils.ts:149`（完整版：處理 `data:`、相對路徑、`baseOrigin`、try/catch）與 `src/utils/llms-utils.ts:107`（簡化版：字串拼接）各一份。
- **推薦的寫法**：`permalink-utils.ts` 刪掉自己的 `removeFileExtension`，改 `import { removeFileExtension } from './url-utils'`；`llms-utils.ts` 的 `toAbsoluteUrl` 也改 import `url-utils` 的版本，改完後比對產出的 `llms.txt` 前後一致再提交（`url-utils` 版行為是前者的超集，差異只在邊界情況）。

```ts
// before：permalink-utils.ts:4 自帶一份
export function removeFileExtension(id: string): string {
  return id.replace(/\.(md|mdx|markdown)$/i, "");
}

// after：permalink-utils.ts 頂部
import { removeFileExtension } from "./url-utils.ts";
export { removeFileExtension };
```

- **為什麼**：`src/utils/` 已有 70+ 個檔案，再放任同名同邏輯的 helper 複製，以後沒人知道該改哪一份。
- **優先級**：P1；**工作量**：S。

### C2. `package.json` 裡 `dev`／`start`／`build` 把同一串三段前置步驟各抄一次

- **現在的寫法**：`dev`、`start`、`build` 都以 `node scripts/content/sync.mjs && node scripts/icons/generate-local-icons.mjs && node scripts/images/generate-moment-thumbnails.mjs` 開頭（`dev` 與 `start` 整行完全相同），之後才接各自的命令。
- **推薦的寫法**：抽成一個具名腳本 `prepare-content`，三個命令改呼叫它（或用 npm 的 `predev`／`prebuild`／`prestart` hook 自動執行它），以後前置步驟增減只改一處。

```json
// before
"dev": "node scripts/content/sync.mjs && node scripts/icons/generate-local-icons.mjs && node scripts/images/generate-moment-thumbnails.mjs && astro dev",
"start": "node scripts/content/sync.mjs && node scripts/icons/generate-local-icons.mjs && node scripts/images/generate-moment-thumbnails.mjs && astro dev",
"build": "node scripts/content/sync.mjs && node scripts/icons/generate-local-icons.mjs && node scripts/images/generate-moment-thumbnails.mjs && astro build && npx pagefind --site dist && node scripts/fonts/check-fonts.mjs"

// after
"prepare-content": "node scripts/content/sync.mjs && node scripts/icons/generate-local-icons.mjs && node scripts/images/generate-moment-thumbnails.mjs",
"dev": "npm run prepare-content && astro dev",
"build": "npm run prepare-content && astro build && npx pagefind --site dist && node scripts/fonts/check-fonts.mjs"
```

- **為什麼**：前置步驟改一處漏兩處，遲早出現「本機跑得起來、build 卻缺圖」的落差。
- **優先級**：P2；**工作量**：S。

### C3. `src/components/organisms/Search.svelte:118,124,130` 的除錯 `console.log` 會在正式站每次開頁都印

- **現在的寫法**：`console.log("Pagefind status on init:", pagefindLoaded)`（:118）與 `console.log("Pagefind ready event received.")`（:130）在正式環境也會執行（只有 :124 那句包在 `import.meta.env.DEV` 裡）；`src/integration/index.ts:380` 明明已用 esbuild `pure: ["console.log", "console.debug"]` 幫使用者程式碼去掉 console.log，自己的元件卻還在印。
- **推薦的寫法**：三句全刪，或統一包進 `if (import.meta.env.DEV)`；`console.warn`（pagefind 載入失敗那句）留著，那是真的需要被看到的錯誤。

```svelte
// before：Search.svelte:118 正式環境也印
console.log("Pagefind status on init:", pagefindLoaded);

// after：刪掉；真的需要除錯時再用 DEV 包起來
if (import.meta.env.DEV) console.log("Pagefind status on init:", pagefindLoaded);
```

- **為什麼**：正式站 console 殘留除錯訊息會蓋掉真正需要看的警告，也顯得沒收尾。
- **優先級**：P2；**工作量**：S。

### C4. `src/components/organisms/BannerStage.astro`（1164 行）的前 matter 在做圖片管線的工作

- **現在的寫法**：`BannerStage.astro:38-160` 的 frontmatter 內含 `optimizeBannerImage`、`normalizeBannerSource`、以 `Map` 做的最佳化快取、四組 `Promise.all`（desktop／mobile × light／dark）與一堆衍生變數；模板本身只在後半段。
- **推薦的寫法**：把圖片正規化＋最佳化＋快取整段搬到 `src/utils/banner-images.ts`（例如 `getBannerImages(siteConfig.banner)` 一次回傳四組響應式圖片），元件 frontmatter 只剩「拿設定、呼叫 util、把結果交給模板」。

```astro
---
// before：BannerStage.astro:38-160，optimizeBannerImage、normalizeBannerSource、optimizationCache、四組 Promise.all 全在元件內
// after：BannerStage.astro 前 matter 只剩
import { getBannerImages } from "@utils/banner-images";
const { desktopImages, mobileImages, initialImage } = await getBannerImages(siteConfig.banner);
---
```

- **為什麼**：圖片管線邏輯被鎖在單一元件裡，其他頁面想用同一套最佳化時只能再抄一份。
- **優先級**：P2；**工作量**：M。

### C5. `src/integration/index.ts:227-304` 連續載入 5 個站點設定模組，每個都是同形狀的載入＋轉型

- **現在的寫法**：`shirones()` 的 `astro:config:setup` hook 內，site、sidebar、music、umami、markdown 五個設定模組各寫一次 `await loadConfigModule(...)`＋`as { … }` 轉型（:227、:237、:247、:257、:300），中間穿插各自的解析函式呼叫；`index.ts` 本身 608 行。
- **推薦的寫法**：在 `src/integration/load-config.ts`（已存在）補一個泛型 helper `loadConfig<T>(paths, name)`，把「載入＋轉型＋缺檔報錯」包起來，`index.ts` 每個設定只剩一行；hook 內的各設定解析（`resolveMusicOptions`、`resolveUmamiOptions` 等）也各自抽成小函式，讓 `index.ts` 只剩 wiring。

```ts
// before：index.ts:227-232
const siteModule = await loadConfigModule(paths, "siteConfig");
const siteConfig = siteModule.siteConfig as { /* … */ };

// after
const siteConfig = await loadConfig<SiteConfig>(paths, "siteConfig");
```

- **為什麼**：五次重複的載入樣板是未來加第六個設定時最容易抄錯的地方。
- **優先級**：P2；**工作量**：M。

> 其他次要觀察：`src/components/atoms/feedback/loadingShapes.ts`（3280 行）是標頭已註明「Generated from androidx.graphics.shapes」的純資料表，卻放在元件目錄下以 TS 寫死——建議移到 `src/data/` 改為 JSON 檔 import，標明是產出物、不要手改，也省下 `tsc` 解析 3280 行的成本（P2／S）。`src/components/atoms/feedback/ProgressIndicator.svelte`（918 行）雖大，但它是 M3E 規範的忠實移植、內部已用 `$derived` 分層清楚，不建議硬拆。另提醒：CI 不跑測試、依賴浮動、字型／mp3 進 git、無 README 是健檢報告已列項目，不在此重複。

---

## 摘要

| repo | 條數 | P1 | P2 |
| --- | --- | --- | --- |
| merch | 6 | 4（M2、M3、M4、M6） | 2（M1、M5） |
| nav | 5 | 2（N1、N2） | 3（N3、N4、N5） |
| h | 5 | 3（H1、H2、H3） | 2（H4、H5） |
| 47tsai.github.io | 5 | 1（C1） | 4（C2–C5） |
| **合計** | **21** | **10** | **11** |

### 最重要的 3 條

1. **merch M3**（`src/store.ts:69`）：148 件收藏每次開頁 150+ 次循序請求＋全部 `no-store`——四站裡對使用者體感影響最大的效能問題，修法是迴圈外抓一次＋`Promise.all` 平行＋快取。
2. **h H2**（`src/lib/converter.ts` vs `src/lib/converter-sources.ts`）：測試最完整的轉換器是死碼，上線的用寫死長度表猜來源，選錯來源時靜默回空結果。
3. **nav N1**（`js/search.js:4,10,11`）：搜尋每鍵重算全部連結的 haystack，方向鍵還重算兩次；預先算好 haystack＋debounce＋快取結果即可解決。

### 整組最值得先做的 1 件事

**先做 merch M3（資料抓取平行化＋快取）**。理由：它是 21 條裡唯一「每次開頁都直接拖慢使用者」的問題，修法集中在 `store.ts` 一個檔案、不用動 UI 與資料結構；而且 M3 做完後，M6（搜尋 debounce）的效果才會完整顯現——否則輸入再順，資料載入還是瓶頸。若只能再多做一件，順手做 M2（`escapeHtml`／`formatMoney` 唯一化），因為它是四站共通的毛病（h 有 4 份 `isSafeHttpUrl`、個人站有 2 份 `removeFileExtension`），同一習慣修一次、四站都受惠。
