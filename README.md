# 全部專案推薦寫法審查 — 總結

- 審查日：2026-11-09
- 範圍：19 個公開 repo（test 已刪除，chi-system 已加入），全部以 `git clone --depth 1` 唯讀審查，未改任何程式碼
- 對照：2026-11-08 健檢報告（`~/workspace/github-audit/research/report.md`）。報告已列的 P0/P1 錯誤與結構問題本文不再重複，只補「程式寫法層面」的推薦
- 原則：只提小型、可逐步做的改進，不提大重寫、不新增依賴

## 四份分報告

| 分組 | 檔案 | repo 數 | 條數 | P1 | P2 |
| --- | --- | --- | --- | --- | --- |
| Python 自動化 | [python-automation.md](python-automation.md) | 8 | 50 | 29 | 21 |
| 前端網站 | [frontend-sites.md](frontend-sites.md) | 4 | 21 | 10 | 11 |
| Worker／服務 | [workers-services.md](workers-services.md) | 3 | 16 | 7 | 9 |
| 設計系統 | [design-system.md](design-system.md) | 4 | 12 | — | — |
| 合計 | | 19 | **99** | | |

設計系統組另有 9 條次要／順手修列為附錄，不計入主數。設計系統組未按 P1/P2 拆分，見該文件各條標示。

每條格式：檔案:行號、現在寫法、推薦寫法（附 before/after 示意）、為什麼（一行）、優先級、工作量（S/M/L）。行號以 2026-11-09 的 clone 為準，之後有新 commit 可能偏移。

## 各組最值得先做的 1 件事

1. **Python 組 — 把舊版 free-game 的重試邏輯搬到新版**（P1/M）：新版所有對外請求都只有一次機會，舊版已有寫好且有測試覆蓋的重試＋`Retry-After`＋錯誤區分，直接搬，一次補上 RSS／文章／ntfy／Discord／GitHub Contents 四個呼叫點的無重試缺口。
2. **前端組 — merch 資料抓取平行化＋快取**（M3）：148 件收藏每次開頁 150+ 次循序請求＋全部 `no-store`，四站裡對體感影響最大的效能問題，修法集中在 `store.ts` 一檔。
3. **Worker 組 — sc-auto-claim 讓失敗真的有非零結束碼**（SC-1，兩行）：`claim.js` 外層 catch 加 `process.exitCode = 1`＋workflow 加 `set -o pipefail`，結束每天 3 次 × 2 帳號失敗卻顯示綠燈的狀況。
4. **設計系統組 — chi-system 新增 `--chi-color-on-danger`**（P1/S）：深色 danger 按鈕對白字只有 3.51:1、checkbox 勾記對深色 accent 只有 2.71:1，皆未達標；舊 repo 已有此 token 模式可搬概念。

## 整體最划算的一類：已經寫好的東西沒被用上

- 舊版 free-game 的重試邏輯（有測試）→ 新版沒用
- h 的 `SourceAdapter` 轉換器（測試最完整）→ 上線程式沒用，改用寫死長度表猜來源
- 舊 chi-design-system 的 `--chi-color-on-danger` token、skip link 寫法、檢查腳本四項檢查 → 新 chi-system 沒搬
- 舊版 free-game 的 ntfy 持久化 cursor → 新版用寫死的 `since=24h`

## 跨 repo 共通壞習慣

1. **`except Exception` 兜底**：幾乎每一個 Python repo 都有，錯誤被吞掉、失敗不可見。
2. **輸出檔案原地覆寫**：status、chi-source 等每輪直接覆寫 JSON，寫到一半中斷會留半份檔案；應改原子寫入（寫暫存檔再 rename）。
3. **同一份邏輯寫多份**：`isSafeHttpUrl` 寫了 4 份、前端後端各寫一份同步文件驗證、`removeFileExtension` 兩份一字不差、主題邏輯寫 6 次（後者已由 chi-system 統一處理中）。
4. **死碼**：h 的 `SourceAdapter`、nav 無人引用的 `redesign.css`（26KB）、chi-source 已死的 AppTesters 路徑、SubsTracker 雙 API／雙軌提醒。
5. **單檔過大**：SMS `page()` 單一函式約 490 行、status `update_status.py` 991 行、h `AppShell.svelte` 859 行、SubsTracker `adminPage.html` 4,024 行。

## 審查中對 11/08 健檢報告的更正

- nav 搜尋其實**有**加權 ranking（`js/search.js:4`，標題完全相符 1000、開頭 400 等），報告寫「無 ranking」是錯的。
- nav「0 位元組 CSS 空檔」是誤判：`style.css` 等是單行壓縮檔（12,717 位元組）；實際問題是 `redesign.css`（26KB）無任何引用。
- SMS 已比報告時進步很多：主檔現為 952 行（報告寫 677 行），Phase 1 P0 加固（限流、timing-safe key、冪等鍵、訊息分段）已上線。
- status 的 `update_status.py` 審查時已是 991 行（報告基準 916 行）。

## 建議的使用順序

先做各組「最值得先做的 1 件事」（上面 4 條），其餘按 P1、再按工作量 S 挑著做。free-game-old／example-site／chi-design-system 三個將封存的 repo 不修，只搬值得搬的東西到新版。
