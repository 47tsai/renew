# Python 自動化專案群組 — 寫法層面推薦清單

- 審查基準：2026-11-09，唯讀審查，未改任何程式碼、未 commit、未 push、未執行測試、未發任何對外請求。
- 審查範圍：8 個 Python repo（free-game、free-game-old、chi-source、chi-source-personal、status、bahasign、apktwsign、oci-arm-retry），以 `git clone --depth 1` 取得當日 HEAD 逐檔閱讀（入口、核心邏輯、workflow yml）。
- 與既有健檢報告的關係：本文件補充 `~/workspace/github-audit/research/report.md`（2026-11-08）沒有的東西——程式寫法層面的推薦做法。報告已列的 P0/P1 錯誤與結構問題不在此重寫，只在某條建議與它相關時引用一句。
- 原則：只提小型、可逐步做的改進，絕不提大重寫；不提新依賴（除非理由很充分）；不提設計系統統一的事。free-game-old 是舊版、將封存：只寫值得搬到新版的東西，不提在舊版上修東西。
- 行號說明：行號以 2026-11-09 的 clone 為準；之後若有新 commit，行號可能偏移（例如 `status/scripts/update_status.py` 健檢時 916 行，審查時已是 991 行），請以函式名／關鍵字複核。

## 條目總覽

| Repo | 條數 | P1 | P2 |
| --- | --- | --- | --- |
| free-game | 6 | 2 | 4 |
| free-game-old（只列可搬到新版者） | 6 | 3 | 3 |
| chi-source | 6 | 5 | 1 |
| chi-source-personal | 6 | 3 | 3 |
| status | 8 | 3 | 5 |
| oci-arm-retry | 6 | 3 | 3 |
| bahasign | 6 | 5 | 1 |
| apktwsign | 6 | 5 | 1 |
| **合計** | **50** | **29** | **21** |

每條格式：檔案:行號／現在的寫法／推薦的寫法／為什麼（一行）／優先級（P1 值得做／P2 有空再做）／工作量（S 小／M 中／L 大）。

---

## 1. free-game（新版，6 條）

### 新-1. 魔法字串／數字與共用小函式散在各檔，收進常數／工具模組
- **檔案:行號**：`src/free_game/runner.py:23,33`、`src/free_game/writer.py:45,61,322,343-344`、`src/free_game/notifications.py:28,40,46,48,51,54,65,96,110,144`、`src/free_game/fgc.py:68-69`、`src/free_game/article_analysis.py:290`
- **現在的寫法**：同一個值在 4 個檔各寫一次：`User-Agent: FreeGameAggregator/0.1`（runner:33、writer:61、notifications:65、fgc:69，三處還缺 `(+https://github.com/47tsai/free-game)` 後綴，格式已不一致）；`timeout_seconds: float = 20.0` 在 runner:23、writer:45、notifications:28、fgc:68 各寫一次；台灣時區 `timezone(timedelta(hours=8))` 在 notifications:46 與 article_analysis:290 各寫一次；Discord 色碼 `0x2ECC71／0x5865F2／0xF1C40F` 與截斷 `[:256]` 寫死在 notifications:40,48,51,54；`writer.py:343-344` 已有 `_now()`（`.isoformat().replace("+00:00", "Z")`），但 notifications:96,110,144 仍各寫一次同樣的 `.isoformat().replace("+00:00", "Z")`；`writer.py:322` 在函式內才 `import unicodedata`。
- **推薦的寫法**：新增 `src/free_game/constants.py`（或放進現有共用模組），並把函式內 import 移到檔頭：
```python
# constants.py
USER_AGENT = "FreeGameAggregator/0.1 (+https://github.com/47tsai/free-game)"
DEFAULT_TIMEOUT_SECONDS = 20.0
TAIPEI_TZ = timezone(timedelta(hours=8))
DISCORD_GREEN, DISCORD_BLURPLE, DISCORD_YELLOW = 0x2ECC71, 0x5865F2, 0xF1C40F
EMBED_TITLE_MAX_LEN = 256  # Discord embed title 上限
# writer.py 的 _now() 改為公開的 now_iso()，notifications.py 直接 import 使用，不再各寫一次 replace
```
註：`notifications.py:13` 已有跨模組 import 私有函式 `from free_game.writer import _stable_id` 的先例，順手把 `_now`、`_parse_time` 也改為公開共用函式即可，不必新開依賴。
- **為什麼**：同一數值改一處漏一處就會行為分叉（User-Agent 已分叉），且 Discord 截斷長度改版時只改常數即可。
- **優先級**：P2／**工作量**：S

### 新-2. 例外只留類別名、訊息被丟掉，除錯時看不到原因
- **檔案:行號**：`src/free_game/cli.py:47-52`、`src/free_game/cli.py:94-96`、`src/free_game/notifications.py:102-104`
- **現在的寫法**：`collect_candidates` 抓文章失敗時 failures 只記 `"reason": type(exc).__name__`（cli:51），連 `str(exc)` 都沒有，例如 `SourceFetchError("http_status_403")` 的 403 會被簡化成類別名；`main` 最外層 except（cli:94）同樣只印類別名＋訊息、無 traceback；`notifications.py:102-104` 更進一步把非 `DiscordDeliveryError` 的例外一律改寫成 `"discord_delivery_outcome_unknown"`，原始錯誤（例如 `KeyError`、JSON 問題）直接消失。
- **推薦的寫法**：失敗紀錄保留訊息（截斷即可），並在 CLI 印出 traceback 到 stderr；非預期例外不要改寫成通用字串：
```python
# cli.py:47-52
except Exception as exc:
    failures.append({
        "article_id": observation.external_id,
        "error": "article_fetch_or_analysis_failed",
        "reason": f"{type(exc).__name__}: {str(exc)[:200]}",
    })
# cli.py:94 最外層：print(..., file=sys.stderr) 後加 traceback.print_exc()
# notifications.py:102-104
record["last_error"] = str(exc)[:200]  # 不分種類都留原文；只有在要對外顯示時才映射成通用字串
```
- **為什麼**：現在失敗只剩類別名，線上出問題時無法從 Actions log 判斷是 403、逾時還是程式 bug，只能重現猜測。
- **優先級**：P1／**工作量**：S

### 新-3. 所有對外請求都只有一次機會，只對 GET 加上小重試
- **檔案:行號**：`src/free_game/runner.py:35-48`（`fetch` 只有一次 `urlopen`）、`src/free_game/fgc.py:67-77`（`fetch_fgc_claims` 同樣一次）、`src/free_game/writer.py:63-77`（`_request` 無重試，只有 409 另有重試）、`src/free_game/notifications.py:66-76`（`send_game` 無重試）
- **現在的寫法**：四處各寫一次 `urlopen`＋`except HTTPError/URLError`，皆無迴圈、無退避；短暫的 429／502／網路抖動就整輪失敗。（報告已列「無重試：RSS／文章／ntfy／Discord／GitHub Contents 皆單次嘗試」為 P1，此條是寫法層面的具體做法。）
- **推薦的寫法**：只在一個地方加 GET 重試（例如 `HTTPFetcher.fetch` 與 `GitHubContentsStore._request` 的 GET），POST／Discord 不自動重試（避免重複發通知；429 才可重試一次並遵守 `Retry-After`）。舊版 `free-game-old/src/free_game/http.py:17-100` 已有現成可搬的寫法（見舊-1），不用新寫也不用新依賴：
```python
# 概念（搬舊版 HttpClient.get 的結構）
RETRY_STATUS = {408, 429, 500, 502, 503, 504}
for attempt in range(3 + 1):
    try:
        return urlopen(request, timeout=...)
    except HTTPError as exc:
        if exc.code not in RETRY_STATUS or attempt == 3:
            raise SourceFetchError(f"http_status_{exc.code}") from exc
        time.sleep(min(float(exc.headers.get("Retry-After") or 2 ** attempt), 30))
```
- **為什麼**：排程每 6 小時跑一次，單次抖動就丟一輪文章或通知，而重試邏輯已有舊版實作可直接搬，風險低。
- **優先級**：P1／**工作量**：M（四個呼叫點共用一個 helper，另加測試）

### 新-4. `validate_collection` 每次都重讀 schema 檔並重跑 `check_schema`
- **檔案:行號**：`src/free_game/validation.py:27-36`、`src/free_game/writer.py:273`
- **現在的寫法**：`validate_collection(document)` 在 `schema is None` 時每次呼叫 `load_schema()`（開檔＋`json.load`＋`Draft202012Validator.check_schema`），而 `writer.py:273` 的 `_upsert` 每次寫入、每次 409 重試都會再驗一次；一次 `process_games` 產生多則通知時，`write_notification` 每次都重驗整份 schema。
- **推薦的寫法**：用模組級快取，schema 只載入驗證一次：
```python
# validation.py
from functools import lru_cache
@lru_cache(maxsize=1)
def load_schema(path: Path = SCHEMA_PATH) -> dict[str, Any]:
    ...  # 原內容不變
# 測試要換 schema 時用 load_schema.cache_clear()
```
- **為什麼**：schema 在執行期間不會變，每次重讀重驗是純浪費，且寫入量長大後每次驗證成本跟著長。
- **優先級**：P2／**工作量**：S

### 新-5. `diagnostics()` 把失敗文章的 HTML 又完整解析一次
- **檔案:行號**：`src/free_game/article_analysis.py:140-152`（`analyze` 內 `parser.feed(text)`，行 149）、`src/free_game/article_analysis.py:207-214`（`diagnostics` 又建一個 `_ArticleHTMLParser` 再 `feed` 一次，行 213）；呼叫點 `src/free_game/cli.py:42-47`（`analyze` 回空才調 `diagnostics`）
- **現在的寫法**：同一份 HTML 在失敗時被 parse 兩次（一次在 `analyze`，一次在 `diagnostics`），兩次都跑完整個 `_ArticleHTMLParser`，且兩邊的解碼 `html.decode("utf-8", errors="replace")` 也重複。
- **推薦的寫法**：二選一，小改即可——（a）`diagnostics` 接受已 parse 好的 parser／body_text 當參數，（b）或 `analyze` 直接回傳 `(candidates, diagnostics)`，失敗時 cli 不用再調一次：
```python
# (b) 示意
def analyze(self, observation, html):
    ...
    return tuple(results), self._diagnostics_from(parser, text)
# cli.py 改為：extracted, diag = analyzer.analyze(...); 失敗時直接用 diag
```
- **為什麼**：失敗文章通常正是最大、最怪的那幾篇，重複解析等於把最貴的成本付兩次，且兩份解析邏輯日後改一邊漏一邊會對不上。
- **優先級**：P2／**工作量**：S

### 新-6. 兩個排程 workflow 的安裝步驟幾乎逐行相同，且 validate／worker／pages 三個 workflow 沒有 `timeout-minutes`
- **檔案:行號**：`.github/workflows/ingest.yml:17-27` 與 `.github/workflows/fgc-sync.yml:17-27`（checkout → setup-python 3.12 → `pip install .` 完全相同，只有最後一步指令不同）；`.github/workflows/validate.yml`、`.github/workflows/worker.yml`、`.github/workflows/pages.yml` 全文皆無 `timeout-minutes`（只有 ingest:19 設 15 分鐘、fgc-sync:19 設 10 分鐘有設）
- **現在的寫法**：兩個排程 workflow 各複製一份 Python 安裝步驟；其餘三個 workflow 的 job 沒有逾時上限，卡住時預設可跑 6 小時。
- **推薦的寫法**：先做最小的兩步，不必抽 reusable workflow：（a）把共用的三步抽成同一 repo 的 composite action（例如 `.github/actions/setup-python-project/action.yml`），ingest 與 fgc-sync 改為 `uses: ./.github/actions/setup-python-project`；（b）其餘三個 job 各加一行 `timeout-minutes`（validate 15、worker 15、pages 10，照現有 ingest／fgc-sync 的量級估）：
```yaml
jobs:
  python-validation:
    runs-on: ubuntu-latest
    timeout-minutes: 15
```
- **為什麼**：安裝步驟改版（例如 Python 升版）現在要改兩個檔，遲早漏改；無逾時的 job 卡住時會靜默燒 Actions 分鐘且無失敗訊號。
- **優先級**：P2／**工作量**：S

### 新版次要觀察（未列入上面 6 條，僅供參考）
- `src/free_game/writer.py:228-237` 的 `update_game` 先 `store.read` 一次找遊戲，進 `_upsert`（行 250）又 `store.read` 一次，同一輪讀兩次整份 games.json；且 `merged.update(changes)`（行 235）未過濾 key，呼叫方傳錯 key 會直接寫進紀錄，建議 `_upsert` 接受已合併好的 dict 或加 key 白名單。P2／S。
- `src/free_game/fgc.py:51-58` 的 `claims[(item_platform, external_id)]` 以 dict 去重，同一遊戲出現兩次時後者靜默覆蓋前者，建議去重時若標題不同記一筆 warning。P2／S。
- `src/free_game/notifications.py:87-115` 的 `deliver()` 是帶 `nonlocal sent/skipped/failed` 的巢狀閉包，且同一 `record` dict 被原地改寫後連寫兩次（pending → sent／failed），建議拆成 `def _deliver_one(game, kind, key, ...)` 獨立函式並回傳新 dict 而非原地改。P2／S。

---

## 2. free-game-old（舊版，6 條——只寫值得搬到新版的東西）

### 舊-1. `HttpClient` 的重試＋`Retry-After`＋`AmbiguousDeliveryError` 區分（新版最值得搬的一段）
- **舊版檔案:行號**：`src/free_game/http.py:12-16`（`AmbiguousDeliveryError`）、`:21-39`（`get` 重試迴圈）、`:40-92`（`post_json`）、`:93-100`（`_retry_delay` 讀 `Retry-After`，上限 30 秒）
- **現在的寫法（舊版）**：`get`／`post_json` 在 `RETRY_STATUS = {408, 425, 429, 500, 502, 503, 504}` 時重試最多 3 次、指數退避 `min(2 ** attempt, 30)`，且 408／5xx 失敗時視情況拋 `AmbiguousDeliveryError`（「POST 可能已被接受、結果未知」，呼叫方不得直接重發），與確定失敗的 `HttpError` 區分開；舊版測試 `tests/test_http_retry.py` 已覆蓋此行為，可一併參考寫法。
- **推薦的搬法**：把 `http.py` 的 `RETRY_STATUS`、`_retry_delay`、`AmbiguousDeliveryError` 搬進新版 `runner.py` 的 `HTTPFetcher`（GET 重試，見新-3），並把「模糊投遞」概念搬進新版 `notifications.py`：Discord 回 408／5xx 或傳輸中斷時記為 `uncertain`（不重發、等下一輪人工／狀態比對），只有確定失敗（4xx 如 400／401）才記 `failed`。
- **為什麼**：新版通知失敗永不重試是刻意設計（報告已述），但「不知道有沒有送出」和「確定沒送出」混在一起會導致重發造成重複通知或漏通知，舊版已把這個區分寫好。
- **優先級**：P1／**工作量**：M（搬重試邏輯＋uncertain 狀態＋測試）

### 舊-2. ntfy 用持久化 cursor 取代寫死的 `since=24h`
- **舊版檔案:行號**：`src/free_game/claims.py:166-181`（`load_cursor`／`save_cursor`，讀寫 `data/ntfy_cursor.json`）、`:187-213`（`poll_ntfy`：有 cursor 時 `query["since"] = cursor`，行 191；`x-messages-truncated == "1"` 時拒絕推進 cursor，行 196）
- **現在的寫法（舊版）**：每次成功同步後把最新 ntfy 訊息 id 存檔，下次從該 id 繼續拉；若伺服器回覆被截斷（`x-messages-truncated`），拋錯且不更新 cursor，下一輪重拉，不會跳過訊息。（報告已列新版 `fgc.py:13` 寫死 `since=24h`、排程中斷超 24h 會永久遺失為 P1，此條是舊版已有的解法。）
- **推薦的搬法**：新版 `fgc.py` 的 `_TOPIC_URL`（行 13）去掉 `since=24h`，改為讀 `data/operations/ntfy_cursor.json`（或沿用現有 `notifications.json` 的存放模式）取得上次 cursor，成功後再寫回；`fetch_fgc_claims` 回傳時一併帶回最新訊息 id，`cli.py` 的 `fgc-sync` 分支負責存檔；截斷時同舊版做法：不更新 cursor、讓下一輪重拉。
- **為什麼**：排程中斷超過 24 小時是遲早會發生的事（Actions 故障、停電），cursor 制是唯一能回補的做法，且舊版邏輯與測試都已存在。
- **優先級**：P1／**工作量**：M（cursor 讀寫＋cli 接線＋測試）

### 舊-3. `format_deadline` 的「星期＋相對時間」呈現（新版通知只有絕對日期）
- **舊版檔案:行號**：`src/free_game/discord.py:8`（`WEEKDAYS_ZH`）、`:11-56`（`format_deadline`：ISO 或中文日期轉台灣時間，輸出如 `2026 年 11 月 20 日（週四） 23:59（約 3 天後）`，已截止／不足一天時顯示「已截止」／「約 X 小時後」）
- **現在的寫法（新版對照）**：`free-game/src/free_game/notifications.py:44-48` 只拼 `local_deadline.strftime("%Y/%m/%d %H:%M") + "（台灣時間）"`，沒有星期、沒有「還剩幾天」。
- **推薦的搬法**：把舊版 `format_deadline` 整段搬進新版（可直接取代 notifications:44-48 的期限字串），相對時間的門檻沿用舊版（`>= 86400` 秒顯示天數，否則顯示小時數、不足一小時進位為 1 小時）。
- **為什麼**：同一行字多給「週四」「約 3 天後」，使用者不用自己算日子是否來得及領，是純呈現改進、零風險。
- **優先級**：P2／**工作量**：S

### 舊-4. Steam `/widget/` 連結正規化（新版的 `_external_product_id` 認不得 widget 連結）
- **舊版檔案:行號**：`src/free_game/article.py:87-93`（`normalize_store_url`：`store.steampowered.com` 且路徑含 `/widget/` 時，取末段數字 appid 轉成 `https://store.steampowered.com/app/{appid}/`）
- **現在的寫法（新版對照）**：`free-game/src/free_game/article_analysis.py:326-330` 的 `_external_product_id` 只認 `/app/(\d+)`；4Gamers 文章若嵌的是 Steam widget 連結（`/widget/123456/`），新版會取不到 external_id，game_id 退化成 URL hash，之後同一遊戲用正常連結進來時會被當成另一款（報告已列 FGC 缺 platform 標頭時 external_id 退化的同類問題 P1，此處是文章端的同類缺口）。
- **推薦的搬法**：在新版 `article_analysis.py` 的連結正規化處（`handle_starttag` 內 `urlunparse` 之前，約行 100-110）先調一次等價的 `normalize_store_url`，或在 `_external_product_id` 的 steam 分支加一條：
```python
# _external_product_id，steam 分支
match = re.search(r"/app/(\d+)(?:/|$)", path) or re.search(r"/widget/(\d+)(?:/|$)", path)
```
- **為什麼**：一行正則就補上一個連結形態，避免同一遊戲因連結寫法不同而分裂成兩筆。
- **優先級**：P1／**工作量**：S

### 舊-5. 時間／標題正規化小函式應先收斂成一檔再搬，不要整批複製
- **舊版檔案:行號**：`src/free_game/notifications.py:12-26`（`_iso` 要求 naive 時間拋錯，行 15；`_parse` 行 19）、`src/free_game/active.py:9-16`（`_parse_deadline`，同樣的 `fromisoformat(...replace("Z", "+00:00"))`）、`src/free_game/claims.py:18-21`（`normalize_title`：NFKC＋casefold＋空白合併）、`src/free_game/state.py:48-51` 與 `src/free_game/stores.py:133-135`（兩個長得幾乎一樣的 `_normalize`）
- **現在的寫法（新版對照）**：新版同樣散在三處：`free-game/src/free_game/writer.py:321-336`（`_normalize_title`）、`:343-344`（`_now`）、`free-game/src/free_game/notifications.py:148-155`（`_parse_time`，naive 時間靜默回 `None`，與舊版 `_iso` 的拋錯做法相反）。
- **推薦的搬法**：不要把舊版五份各搬一份；只搬兩個新版沒有的行為，收進一個新檔（例如 `free_game/timeutil.py`＋`textutil.py`，或併入新-1 的共用模組）：（a）舊版 `_iso` 的做法——naive 時間視為程式錯誤直接拋 `ValueError`，而不是靜默回 `None`（新版 `_parse_time` 對 naive 回 `None` 會讓提醒計時無聲跳過）；（b）一份共用的 `normalize_title`（NFKC＋casefold），取代新版 `writer._normalize_title` 與未來 FGC 標題比對時的各寫一份。
- **為什麼**：時間解析靜默回 `None` 會讓錯誤資料無聲消失，比拋錯更難查；正規化寫兩份遲早出現「一邊 NFKC、一邊沒有」的比對不一致。
- **優先級**：P2／**工作量**：S

### 舊-6. Epic 促銷 feed 的「只取當期活動截止日＋原價」（需齊先確認是否要，暫列 P2）
- **舊版檔案:行號**：`src/free_game/stores.py:229-290`（`EpicPromotionResolver._metadata`：`original_price` 只在原價 >0 且折扣價 ==0 時取用，行 236-247；`promotions.promotionalOffers` 有值時只取當期、不取 `upcomingPromotionalOffers`，行 254-259；截止日取當期各 offer 的最小 `endDate`，行 260-290）；另 `src/free_game/stores.py:40-70`（`SteamPriceResolver.resolve_deadline`：用 `price_overview.discount_expiration` 取 Steam 折扣截止日）
- **現在的寫法（新版對照）**：新版 `offer_end_at` 只從文章內文用正則抽（`article_analysis.py:260-324`），沒有任何商店 feed 來源；報告已述「Steam／Epic 商店驗證是新版有意的產品取捨」，因此這條不是無條件搬。
- **推薦的搬法**：若齊確認想要「Epic 截止日以後援來源」才搬，且只搬 `_metadata` 的兩個判斷（當期優先於即將開始；原價僅在免費當期才取），不要整顆 `EpicPromotionResolver` 搬過去；Steam 的 `resolve_deadline` 同理，只取 `discount_expiration` 一欄即可。搬之前先確認：Epic feed 每款遊戲重打一次整份 feed 的做法不要搬（報告已列為舊版問題），應改為每輪抓一次 feed、再比對多款 slug。
- **為什麼**：文章沒寫截止日時，Epic feed 是唯一可靠的截止日來源；但這是產品取捨問題，不是純寫法問題，須齊點頭。
- **優先級**：P2（待齊確認是否要）／**工作量**：M

---

## 3. chi-source（6 條）

### C-1. 模組 import 時就建 Session、讀設定檔，還用相對路徑找檔案
- **檔案:行號**：`update_source.py:30-32`、`update_source.py:76-84`；連帶 `validate_source.py:8,11`
- **現在的寫法**：模組頂層直接執行 `SESSION = create_session()`、`GITHUB_PROVIDER = GitHubProvider(SESSION)`，以及 `GITHUB_APPS = load_app_config()`（讀 `app_config.json`）。檔案路徑全是工作目錄相對路徑（`FILENAME = "apps.json"`，L13；`Path(APP_CONFIG_FILENAME)`，L77）。結果是 `import validate_source` 也會觸發建 Session＋讀設定檔，`app_config.json` 壞掉或不在工作目錄時，連跑測試／跑驗證都會在 import 階段就炸掉；從 repo 外目錄執行 `python /path/to/update_source.py` 還會讀寫到錯誤位置的 `apps.json`。
- **推薦的寫法**：把 Session／Provider／`GITHUB_APPS` 的建立移到 `main()` 內（或包成 `load_context()` 一次建立後當參數傳進各函式）；路徑改抄 personal 已有的做法，以檔案位置為基準：
```python
# before（模組頂層）
SESSION = create_session()
GITHUB_APPS = load_app_config()
FILENAME = "apps.json"

# after
ROOT = Path(__file__).resolve().parent
APP_CONFIG_PATH = ROOT / "app_config.json"

def main():
    session = create_session()
    apps_config = load_app_config(APP_CONFIG_PATH)
    ...
```
- **為什麼**：import 應無副作用，測試與驗證工具才能在不碰網路、不依賴工作目錄的情況下被載入。
- **優先級**：P1／**工作量**：S

### C-2. `apps.json` 與 `README.md` 都是直接 `write_text`，寫到一半中斷會留壞檔
- **檔案:行號**：`update_source.py:458`（README）、`update_source.py:569`（apps.json）；另 `main()` 先寫 apps.json（L569）才寫 README，若後者出錯，兩檔狀態不一致
- **現在的寫法**：`Path(FILENAME).write_text(json.dumps(...) + "\n")`、`path.write_text(readme)`，都是原地覆寫；程序在寫入中途被 kill 或磁碟出錯時，`apps.json` 會剩一半 JSON，下次執行連舊檔都讀不到。
- **推薦的寫法**：抽一個小 helper，先寫同目錄暫存檔再 `os.replace` 原子置換，兩處共用：
```python
# before
Path(FILENAME).write_text(json.dumps(source, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")

# after
def write_text_atomic(path, text):
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_text(text, encoding="utf-8")
    os.replace(tmp, path)

write_text_atomic(Path(FILENAME), json.dumps(source, ensure_ascii=False, indent=2) + "\n")
```
- **為什麼**：每小時自動跑的程式最怕寫壞唯一一份輸出，原子置換讓任何時刻磁碟上的檔案要不是舊的完整版、就是新的完整版。
- **優先級**：P1／**工作量**：S

### C-3. `keep_latest_only` 用 `except Exception` 靜默吞掉版本解析失敗
- **檔案:行號**：`update_source.py:141-153`，其中 `except Exception: latest[bid] = app` 在 L151
- **現在的寫法**：
```python
try:
    if pkg_version.parse(ver) > pkg_version.parse(get_version(latest[bid])): latest[bid] = app
except Exception: latest[bid] = app
```
版本字串非法時無聲地直接採用後來的 app，log 裡完全看不出來是哪個版本字串壞了；且 `Exception` 還會順手吞掉 `parse` 以外不該吞的錯（例如 `get_version` 內部的 `TypeError`）。
- **推薦的寫法**：只抓 `packaging.version.InvalidVersion`，並印出壞掉的字串與 bundle ID 再決定 fallback：
```python
from packaging.version import InvalidVersion
try:
    if pkg_version.parse(ver) > pkg_version.parse(get_version(latest[bid])):
        latest[bid] = app
except InvalidVersion:
    print(f"⚠️ Invalid version for {bid}: {ver!r} vs {get_version(latest[bid])!r}, keeping latest seen")
    latest[bid] = app
```
- **為什麼**：例外處理應只抓預期中的錯誤種類，否則壞資料會無聲地變成正式輸出。
- **優先級**：P1／**工作量**：S

### C-4. 四個 `build_from_*` 各自手寫一份 12 鍵 App dict，連單行長度都失控
- **檔案:行號**：`update_source.py:225-247`（`build_from_github`，dict 在 L245，486 字元一行）、`update_source.py:250-308`（`build_from_eeveeipa`）、`update_source.py:311-317`（`build_from_apptesters`，dict 在 L316，608 字元一行）、`update_source.py:319-350`（`build_from_ipavault`）
- **現在的寫法**：四個函式各自拼出 `{"name", "bundleIdentifier", "developerName", "subtitle", "localizedDescription", "iconURL", "tintColor", "category", "screenshots", "versions": [...]}` 的 dict，其中三處還是整包塞在一行；將來 AltStore 欄位一改（例如補 `appPermissions`）要改四處，極易漏改。
- **推薦的寫法**：抽一個 `build_altstore_app(name, bundle_id, developer, subtitle, description, icon_url, tint_color, category, version_entry)`，四個 builder 只負責「找版本」，組裝交給它；順便把 dict 拆成多行寫：
```python
# before（L245，整行 486 字元）
return {"name": app["name"], "bundleIdentifier": app["bundleID"], ... "versions": [{...}]}

# after
return build_altstore_app(
    name=app["name"],
    bundle_id=app["bundleID"],
    developer=app["author"],
    ...
    version=version_entry,
)
```
- **為什麼**：輸出結構是四個來源的共同契約，集中一處才能保證四路輸出欄位永遠一致。
- **優先級**：P1／**工作量**：M

### C-5. 兩 repo 共用邏輯應先收斂成一份「原樣複製」的共用小模組（不合併 repo、不新增依賴）
- **檔案:行號**：chi-source `update_source.py:359`（`now_utc8`）、`:396-402`（`format_display_date`）、`:367-376`（`get_status_updated_at`）；`validate_source.py:37-55`（`valid_https_url`、`valid_date`）、`:25`（`HEX_COLOR_RE`）；personal 對應位置為 `update_source.py:59-67, 383-408`、`validate_source.py:13, 29-45`
- **現在的寫法**：`is_https_url`／`valid_https_url`、`valid_date`、`format_display_date`、`now_utc8`、HEX 色正則、`VALID_CATEGORIES`、README 標記區塊解析正則，兩 repo 各寫一份，寫法還不完全相同（例如 chi-source `validate_source.py:43` 用 `except Exception`，personal `validate_source.py:34` 正確地只抓 `ValueError`；兩邊 `screenshots` 格式也不同：chi-source 驗字串，personal 驗 `{"imageURL": ...}` dict）。報告已指出兩 repo「應收斂為同一代架構」，但那是架構層面；寫法層面可先做更小的一步。
- **推薦的寫法**：只挑「純函式、無狀態、無網路」的 helper，抽成一個約 80 行的 `chi_common.py`，以原樣複製（vendoring）方式同時放進兩 repo，檔頭註明「以 chi-source 版為準，改動後同步複製到另一 repo」；第一批只放 `is_https_url`、`valid_date`、`format_display_date`、`now_utc8`、`HEX_COLOR_RE`、`replace_marked_section`（見 C-6 的邏輯）。不建新 repo、不發套件、不加依賴。
```python
# chi_common.py（兩 repo 各放一份相同內容）
def is_https_url(value): ...
def valid_date(value): ...
def format_display_date(value): ...
def now_utc8(): ...
```
- **為什麼**：純函式複製成本最低、零部署風險，卻能先消滅「同一邏輯兩種寫法、修一邊漏一邊」的漂移來源。
- **優先級**：P1／**工作量**：M

### C-6. `update_readme` 內三段標記區塊置換是同一段邏輯抄三次
- **檔案:行號**：`update_source.py:413-459`，三段置換在 L437-441（app 表格）、L443-447（刪除舊 STATUS 區塊）、L452-457（status times 區塊）；另 `update_readme` 本身 47 行、`update_source_apps` L508-554、`main` L556-590 也都偏長
- **現在的寫法**：每段都是 `left = readme.index(START)` → `right = readme.index(END, left) + len(END)` → `readme[:left] + new + readme[right:]`，差別只在起止標記與新內容；中間還夾一段「找到 STATUS 區塊就刪掉它」的特殊邏輯（L443-447），與 L443 上方 STATUS_START／END 已定義卻「永不寫入新區塊」呼應（報告已列為死碼，此處只談重複寫法）。
- **推薦的寫法**：抽 `replace_marked_section(readme, start, end, new_block)`（找不到標記時回傳原字串或在尾端附加，由參數決定），三段都呼叫它；`update_readme` 則只留「算表格、算狀態、組三段區塊」的協調邏輯：
```python
# before（三段各 5 行，L437-457）
left = readme.index(APP_TABLE_START)
right = readme.index(APP_TABLE_END, left) + len(APP_TABLE_END)
readme = readme[:left] + app_table + readme[right:]

# after
def replace_marked_section(readme, start, end, block):
    if start in readme and end in readme:
        left = readme.index(start)
        right = readme.index(end, left) + len(end)
        return readme[:left] + block + readme[right:]
    return readme.rstrip() + "\n\n" + block + "\n"

readme = replace_marked_section(readme, APP_TABLE_START, APP_TABLE_END, app_table)
```
- **為什麼**：字串切片的邊界條件（`index` 找不到會拋 `ValueError`）只需在一處處理正確，三處就不會各壞各的。
- **優先級**：P2／**工作量**：S

---

## 4. chi-source-personal（6 條）

### P-1. `main()` 用 `except Exception` 包住全部邏輯，程式 bug 也被當成「同步失敗」
- **檔案:行號**：`update_source.py:486-576`，其中 `except Exception as exc: ... raise SystemExit(1)` 在 L573-575；另 `main()` 內 `--app` 合併分支佔 L543-565
- **現在的寫法**：整個 `main()`（讀設定、抓來源、合併、寫檔、寫 README）全包在一個 `try/except Exception` 裡，任何 `KeyError`／`TypeError`／`AttributeError`（例如 `app_config.json` 某欄位缺失導致的程式錯誤）都只印一行 `❌ Source sync failed: ...` 就 exit 1，traceback 被吃掉，CI log 看不出是哪一行出的錯；且 `main()` 長達 90 行，`--app` 分支內又重複讀了兩次 `OUTPUT_FILE`（見 P-2）。
- **推薦的寫法**：分兩步做。先把 `--app` 合併抽成 `merge_single_app(old_apps, match)` 純函式；再把 broad except 縮小到「預期中的外部失敗」，讓程式錯誤帶 traceback 自然拋出：
```python
# before（L573-575）
except Exception as exc:
    print(f"❌ Source sync failed: {exc}", file=sys.stderr)
    raise SystemExit(1)

# after
except (OSError, ValueError, requests.RequestException) as exc:
    print(f"❌ Source sync failed: {exc}", file=sys.stderr)
    raise SystemExit(1)
# KeyError / TypeError 等程式錯誤不抓，讓 traceback 留在 CI log
```
- **為什麼**：外部失敗（網路、壞 JSON）與程式 bug 應有不同能見度，前者一句話就夠，後者沒有 traceback 根本無法修。
- **優先級**：P1／**工作量**：M

### P-2. `main()` 內 `apps.json` 被讀了兩次，且 `validate_source.py` 仍用工作目錄相對路徑
- **檔案:行號**：`update_source.py:527-531`（讀舊檔求 `old_apps_for_status`）與 `:536-541`（`--app` 時又讀一次求 `existing`）；`validate_source.py:8`（`FILENAME = "apps.json"`）、`:179`（`Path(FILENAME).read_text(...)`）
- **現在的寫法**：`main()` 在 L527 讀一次 `OUTPUT_FILE`，`--app` 分支 L536 又讀一次，兩次都包各自的 `try/except JSONDecodeError`；而 `update_source.py` 已正確用 `ROOT = Path(__file__).resolve().parent`（L14-16），`validate_source.py` 卻仍用 CWD 相對路徑，同一 repo 內兩種路徑慣例並存。
- **推薦的寫法**：抽 `read_old_source()`（讀不到或壞掉回 `{}`）供兩處共用，並把 `validate_source.py` 的路徑改成與 `update_source.py` 相同的 `ROOT` 寫法：
```python
# before（L527-531 與 L536-541 各一段 try/except）
old_source = {}
if OUTPUT_FILE.exists():
    try:
        old_source = json.loads(OUTPUT_FILE.read_text(encoding="utf-8"))
    except json.JSONDecodeError:
        old_source = {}

# after
def read_old_source(path):
    try:
        return json.loads(path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError):
        return {}
```
- **為什麼**：同一份檔案的讀取邏輯寫兩次，改一次容錯行為必漏一次；路徑慣例全 repo 統一才能保證從任何目錄執行結果相同。
- **優先級**：P1／**工作量**：S

### P-3. `apps.json`／`README.md` 直接覆寫，應與 chi-source 共用同一份原子寫入 helper
- **檔案:行號**：`update_source.py:483`（README `write_text`）、`:567-570`（`OUTPUT_FILE.write_text`）
- **現在的寫法**：與 chi-source C-2 完全相同的問題：`OUTPUT_FILE.write_text(json.dumps(...) + "\n")` 原地覆寫，寫入中途被中斷會留半份 JSON；且寫檔在 `update_readme` 之前，若 README 寫入失敗，`apps.json` 已是新版、README 仍是舊版。
- **推薦的寫法**：與 C-2、C-5 一起做：把 `write_text_atomic()` 放進共用模組 `chi_common.py`（見 C-5），兩 repo 各自呼叫；順序上可先把新 README 內容算好、兩個寫入都成功才算完成，或至少讓寫入失敗時 `apps.json` 不會先被換掉。
```python
# before（L567-570）
OUTPUT_FILE.write_text(
    json.dumps(source, ensure_ascii=False, indent=2) + "\n",
    encoding="utf-8",
)

# after
write_text_atomic(OUTPUT_FILE, json.dumps(source, ensure_ascii=False, indent=2) + "\n")
```
- **為什麼**：與 C-2 同一理由，且這是兩 repo 可共用的第一個「有狀態」helper，適合驗證 C-5 的 vendoring 做法是否跑得通。
- **優先級**：P1／**工作量**：S

### P-4. 每個 App 的版本特殊規則散落在函式內的 `if name == ...`
- **檔案:行號**：`update_source.py:321-322`（`EnsWilde` 的 `re.sub(r"^EnsWilde", ...)` 藏在 `load_github_release_catalog` 內）；對照 chi-source `update_source.py:130-139`（`normalize_version` 內 `if name == "MaxMusic"`／`if name == "YTKACE"` 兩分支）
- **現在的寫法**：版本字串的抽取規則是「某個 App 名 → 一段正則」的對應關係，卻寫成函式體內的 `if/elif` 分支；新增一個特殊 App 就要改函式本體，且規則散在兩個 repo 的不同函式裡，沒人能一眼看全。
- **推薦的寫法**：把規則收成一張模組頂層的表，函式只做「查表、套用」：
```python
# before（L321-322）
if app_config.get("name") == "EnsWilde":
    display_version = re.sub(r"^EnsWilde", "", display_version, flags=re.IGNORECASE)

# after
VERSION_RULES = {
    "EnsWilde": (re.compile(r"^EnsWilde", re.IGNORECASE), ""),
    # 未來新增特殊 App 時只加一行，不動函式本體
}
strip_re, repl = VERSION_RULES.get(app_config.get("name"), (None, None))
if strip_re:
    display_version = strip_re.sub(repl, display_version)
```
- **為什麼**：資料（哪個 App 有何規則）與邏輯（如何套用規則）分開後，加 App 不必讀懂整個抓取函式，出錯時也只需看一張表。
- **優先級**：P2／**工作量**：S

### P-5. `update_readme` 內標記區塊置換抄兩次，24 小時判斷也 inline 寫了一遍
- **檔案:行號**：`update_source.py:410-484`，兩段置換在 L466-470（app 表格）與 L478-482（status times）；24 小時判斷在 L435-442（`datetime.strptime` 寫兩次、`86400` 寫死）
- **現在的寫法**：與 chi-source C-6 相同的三行切片邏輯抄了兩份；「更新時間是否在 24 小時內」的判斷 inline 寫在迴圈內，`datetime.strptime(checked_at, ...)` 與 `datetime.strptime(previous_time, ...)` 各出現一次，格式字串 `"%Y-%m-%d %H:%M:%S"` 重複；chi-source 同功能已抽成 `is_updated_within_day()`（`update_source.py:378-386`），personal 卻沒用。
- **推薦的寫法**：直接沿用 C-5／C-6 的共用 helper：`replace_marked_section()` 處理兩段置換，`is_updated_within_day(updated_at, checked_at)`（連同 `86400` 常數一起收進共用模組）取代 L435-442 的 inline 計算：
```python
# before（L435-442）
try:
    recent = previous_time and 0 <= (
        datetime.strptime(checked_at, "%Y-%m-%d %H:%M:%S")
        - datetime.strptime(previous_time, "%Y-%m-%d %H:%M:%S")
    ).total_seconds() < 86400
except (TypeError, ValueError):
    recent = False

# after
recent = bool(previous_time) and is_updated_within_day(previous_time, checked_at)
```
- **為什麼**：chi-source 已有現成正確實作，personal 重寫一次等於多一份需要同步維護的日期邏輯。
- **優先級**：P2／**工作量**：S

### P-6. 測試用 `try/except` 手寫「預期拋錯」，且 fake 物件寫在測試檔內
- **檔案:行號**：`test_update_source.py:93-101`（`test_invalid_source_entry_fails_loudly` 用 try/except + `raise AssertionError`）；同檔 L103-121（`FakeResponse`／`FakeSession` 手寫假物件）；對照 chi-source `test_update_source.py:155-170` 同樣問題
- **現在的寫法**：
```python
try:
    updater.merge_catalogs([("BrokenSource", bad)])
except ValueError as exc:
    assert "bundleIdentifier" in str(exc)
else:
    raise AssertionError("Expected malformed source entry to fail")
```
這種寫法在「沒拋錯」與「拋錯 wrong type」時訊息都不如 pytest 內建機制清楚；且 `FakeSession.get(self, url, timeout)` 只斷言 URL 後綴與 timeout，換個呼叫方式就失效。
- **推薦的寫法**：改用 `pytest.raises`（附 `match` 直接斷言訊息），fake session 改用 `monkeypatch` 配 `unittest.mock.Mock` 或沿用現有 Fake 但把斷言收進 helper：
```python
# before（L93-101，7 行）
try:
    updater.merge_catalogs([("BrokenSource", bad)])
except ValueError as exc:
    assert "bundleIdentifier" in str(exc)
else:
    raise AssertionError("Expected malformed source entry to fail")

# after（3 行，失敗訊息由 pytest 產生）
with pytest.raises(ValueError, match="bundleIdentifier"):
    updater.merge_catalogs([("BrokenSource", bad)])
```
- **為什麼**：測試的失敗訊息本身就是除錯資訊，用標準寫法能讓 CI 失敗時一眼看出預期與實際的落差。
- **優先級**：P2／**工作量**：S

### chi-source 兩 repo 跨 repo 附註（供參考，非主表條目）
- **Workflow 幾乎逐行相同，僅 4 處不同**：兩份 `.github/workflows/main.yml`（各 66 行）差異只有 name、concurrency group、`py_compile` 檔案清單、repo 名；但已出現漂移——personal 版 commit 訊息仍寫 `"auto: sync Chi Source"`（`chi-source-personal/.github/workflows/main.yml:64-65`，應為 Personal），且 chi-source 版 `py_compile` 漏列 `test_special_versions.py`（`chi-source/.github/workflows/main.yml:40`，personal 版 L40 有列）。建議：不建共用 workflow（跨 repo reusable workflow 需額外 repo，現階段過度設計），先把「改一邊 workflow 就對照改另一邊」列入檢查清單，並把 `py_compile` 的手寫清單改成 `python -m compileall -q .`，清單就不會再漂移。兩邊 `timeout-minutes: 10`（L23）與 `fetch-depth: 0`（L28-29）本身沒問題，`fetch-depth: 0` 若只是為了 auto-commit 可再確認是否真需要全量歷史。
- **requirements.txt 兩邊逐字相同**（`requests>=2.32.0,<3`、`packaging>=25.0,<27`、`pytest>=8.0,<9`）：`pytest` 是測試依賴卻裝在每小時執行的正式環境裡，可拆成 `requirements-dev.txt`（workflow 的測試步驟才裝），正式執行只裝 `requests`＋`packaging`。另 `packaging` 在 personal 其實沒被用到（grep 無 `pkg_version` 引用），可確認後移除。
- **其餘小觀察**：chi-source `providers.py:11-22` 的 `latest_release`／`releases` 除了 URL 外完全相同，可抽 `_get(path)`；兩 repo 全檔用 `print` 記 log（chi-source 約 15 處、personal 約 8 處），可逐步改 `logging`（先從 `fetch_json` 失敗與 fallback 處開始，不必一次全改）；兩 repo 逾時寫法不一致（chi-source `timeout=15`，personal `timeout=25`；Retry 的 `allowed_methods` chi-source 含 HEAD、personal 只有 GET），建議收進 C-5 共用模組的具名常數（`REQUEST_TIMEOUT`、`GITHUB_TIMEOUT`）統一；`[:500]` 截斷 release body（chi-source L245、L300）、`86400`（兩邊各 1 處）、`per_page=30/100` 都是魔法數字，改常數時順手處理即可。以上皆為 P2、S。

---

## 5. status（8 條）

### A1. `status.json`／`alert-state.json` 直接覆寫寫入，寫到一半被中斷會毀檔
- **檔案:行號**：`scripts/update_status.py:985-987`（寫入）、`:791`（讀取）；`scripts/notify_discord.py:202, 228, 239`（同一段 `json.dumps(...)` 寫入重複三次）
- **現在的寫法**：
```python
# update_status.py:985
with open("status.json", "w", encoding="utf-8") as f:
    json.dump(payload, f, ensure_ascii=False, indent=2)
# notify_discord.py 三處完全相同的寫入
STATE_PATH.write_text(json.dumps({"active_failures": next_active}, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")
```
另外 `:791` 用 `open("status.json", ...).read()`，沒用 `with`，且用相對路徑（依賴 cwd 剛好是 repo 根目錄）。
- **推薦的寫法**：抽一個小 helper，用臨時檔＋`os.replace` 做原子寫入（報告提到 free-game-old 已有四份原子 JSON 寫入實作，可直接抄最短的一份）；路徑改以 `Path(__file__).resolve().parents[1]` 為基準：
```python
def atomic_write_json(path, payload):
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_text(json.dumps(payload, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")
    os.replace(tmp, path)   # 同一檔案系統內是原子操作

# notify_discord.py 三處寫入改成：
def save_alert_state(state):
    atomic_write_json(STATE_PATH, {"active_failures": state})
```
- **為什麼**：寫入中途被 workflow timeout 砍掉時 `status.json` 會殘留半份 JSON，下輪 `parse_json` 回 `None`、`stale_fallback` 失去上次資料，整張儀表板從「保留上次資料」退化成空白。
- **優先級**：P1／**工作量**：S

### A2. `get_json`／`get_text`／`health_check` 是三套各寫一次的 urllib 包裝，逾時值還不一致，且 GitHub API 失敗無重試
- **檔案:行號**：`scripts/update_status.py:81-89`（`get_json` timeout=20、`get_text` timeout=12）、`:288-313`（`health_check` 又寫一套 `Request`＋`urlopen`，timeout=12）
- **現在的寫法**：三個函式各自組 `Request`、各自 `urlopen`、各自處理例外；GitHub API 回 429／5xx 時 `get_json` 直接拋出，`main()` 的 `except`（`:858-860`）就把整個 repo 標成 unknown——明明只是短暫限流。
- **推薦的寫法**：抽一個共用的小 helper，逾時與重試集中一處（只重試 429／5xx，固定退避即可，不用指數退避套件）：
```python
HTTP_TIMEOUT = 20  # 秒，三處共用

def _fetch(url, headers=HEADERS, timeout=HTTP_TIMEOUT, retries=2):
    last_exc = None
    for attempt in range(retries + 1):
        try:
            req = urllib.request.Request(url, headers=headers)
            with urllib.request.urlopen(req, timeout=timeout) as resp:
                return resp
        except urllib.error.HTTPError as exc:
            if exc.code not in (429, 500, 502, 503, 504) or attempt == retries:
                raise
            last_exc = exc
            time.sleep(2 * (attempt + 1))
    raise last_exc
```
`get_json`／`get_text`／`health_check` 再包它，只留各自的解析邏輯。
- **為什麼**：GitHub API 短暫限流或 5xx 不該讓整張卡片變 unknown，而現在三處寫法各異，改逾時或加 retry 要改三個地方，一定會漏。
- **優先級**：P1／**工作量**：M（helper 小，但三處呼叫點都要換，需跑一次測試）

### A3. `main()` 尾段六個「服務卡片」是近乎逐字複製的字典，每個約 17 行
- **檔案:行號**：`scripts/update_status.py:864-960`（`host_item`、`vault_item`、`auth_item`、`sub_item`、`home_item`、`sms_item`，結構完全相同，連 `status` 三元判斷都複製六次）
- **現在的寫法**：六段字典只有 `name`／`full_name`／`html_url`／`branch`／`commit_message`／`runs_url` 不同，其餘欄位與 `business_data(...)`＋`status` 對映邏輯逐字重複。
- **推薦的寫法**：抽一個小函式，六個卡片變成六行資料表（不動 `business_data` 本身，報告已列它過大的問題，這裡只收斂 `main()` 的重複）：
```python
SERVICE_CARDS = [
    ("vps-01", "VPS-01 · Oracle Cloud Infrastructure", "https://alist.chi.qzz.io/", "OCI", "每 10 分鐘自動更新檢查三個公開服務端點", f"https://github.com/{OWNER}/status/actions"),
    ("vaultwarden-service", "Vaultwarden · 密碼管理服務", "https://vault.chi.qzz.io/", "VPS-01", "每 10 分鐘自動更新檢查 Vaultwarden /alive 存活端點", ...),
    # ...其餘四個同格式
]

def make_service_item(name, full_name, html_url, branch, commit_message, runs_url):
    item = {"name": name, "full_name": full_name, "html_url": html_url,
            "workflow": "外部服務探測", "status": "unknown", "branch": branch,
            "updated_at": datetime.now(timezone.utc).isoformat(),
            "created_at": datetime.now(timezone.utc).isoformat(),
            "event": "healthcheck", "commit_message": commit_message,
            "duration_seconds": None, "runs_url": runs_url, "run_url": runs_url}
    item["business"] = business_data(name, item)
    item["status"] = {"success": "success", "failure": "failure"}.get(
        item["business"]["business_status"], "unknown")
    return item

for args in SERVICE_CARDS:
    results.append(make_service_item(*args))
```
- **為什麼**：現在加第七個服務要複製 17 行，六份裡改錯一份（例如漏改 `run_url`）就會出現卡片連到錯誤 repo 的靜默錯誤。
- **優先級**：P1／**工作量**：S

### A4. 多處 `except Exception` 把「程式 bug」和「預期中的網路失敗」混在一起吞掉
- **檔案:行號**：`scripts/update_status.py:91-96`（`raw()`）、`:106-114`（`taipei_time()`）、`:483-484`（freegame diff）、`:858-860`（`main()` 每 repo 的 catch-all）；`scripts/notify_discord.py:232-235`（`main()` 的 catch-all）
- **現在的寫法**：
```python
# :91-96 raw()：連 KeyError、TypeError 這類程式 bug 也吞掉，只印一行就回 None
try:
    return get_text(RAW_BASE.format(OWNER, repo, path))
except Exception as exc:
    print(f"Optional business record unavailable: {repo}/{path}: {exc}")
    return None
```
`taipei_time()` 的 `except Exception: return str(value)` 同理：時間格式真的錯了會靜默回傳原字串，卡片上看不出是解析失敗。
- **推薦的寫法**：只抓「預期會發生的」例外，程式 bug 讓它炸出來（在 CI 測試裡被發現，而不是上線後靜默錯）：
```python
# raw() 改成：
except (urllib.error.URLError, urllib.error.HTTPError, TimeoutError, UnicodeDecodeError) as exc:
    print(f"Optional business record unavailable: {repo}/{path}: {exc}")
    return None
# taipei_time() 改成：
except (ValueError, TypeError, AttributeError):
    return f"時間格式錯誤：{value}"
```
`main()` `:858` 的 catch-all 可保留（它是每 repo 的隔離層），但應把 `except Exception` 縮小為 `(urllib.error.URLError, ValueError, KeyError)` 之外再加 `traceback` 印出，或至少在訊息裡註明是未預期例外。
- **為什麼**：`raw()` 回 `None` 時上層只會顯示「來源紀錄讀取失敗」，無法分辨是網路斷線還是程式把路徑拼錯，後者會被當成前者無限期忽略。
- **優先級**：P2／**工作量**：S

### A5. `health_check()` 內同一行耗時計算寫了三次；`business_data()` 的 claim 分支用 `__import__` 動態 import
- **檔案:行號**：`scripts/update_status.py:300, 307, 311`（三處 `elapsed_ms = int((datetime.now(timezone.utc) - started).total_seconds() * 1000)`）；`:395-396`（`taipei_tz = __import__("zoneinfo").ZoneInfo("Asia/Taipei")`）
- **現在的寫法**：`health_check` 的成功、HTTPError、一般例外三條路徑各寫一次耗時計算；claim 分支明明檔案頂部已有 `from zoneinfo import ZoneInfo`（`:11`）和現成的 `TAIPEI_TZ` 常數（`:69`），卻在函式內用 `__import__("zoneinfo")` 再取一次。
- **推薦的寫法**：
```python
# health_check：把計時包成內部小函式，或在 finally 算一次
def _elapsed_ms(started):
    return int((datetime.now(timezone.utc) - started).total_seconds() * 1000)
# 三處改成 elapsed_ms = _elapsed_ms(started)

# :395-396 改成：
now_taipei = datetime.now(TAIPEI_TZ)
```
- **為什麼**：三處複製的算式將來改單位（例如改秒）一定漏改一路徑；`__import__` 是動態 import 的特殊寫法，讀者會誤以為這裡有什麼條件載入的理由，其實只是重複造輪子。
- **優先級**：P2／**工作量**：S

### A6. 魔法數字／字串散落：Discord 限制、顏色、截斷長度、重試退避上下限不一致
- **檔案:行號**：`scripts/notify_discord.py:92-99`（429 分支退避 `min(30, max(1, ...))`，URLError 分支卻是無上限的 `2 ** attempt`）、`:107-109, 120-122, 133`（`[:256]`、`[:4096]`、顏色 `15158332`／`3066993`、`embeds[:10]`）；`scripts/update_status.py:299`（`read(65536)`）、`:313, 779, 846`（`[:160]`／`[:180]` 截斷）
- **現在的寫法**：Discord embed 上限 256／4096、顏色十進位整數、每則訊息 10 個 embed，這些數字直接寫在程式裡；同一檔案內兩種重試退避的上限還不一樣（429 有 cap 30 秒，URLError 沒有）。
- **推薦的寫法**：在檔頭集中成常數，顏色用 `0x...` 十六進位寫法一眼可讀：
```python
# notify_discord.py 檔頭
DISCORD_EMBED_TITLE_MAX = 256
DISCORD_EMBED_DESC_MAX = 4096
DISCORD_EMBEDS_PER_MESSAGE = 10
DISCORD_COLOR_FAILURE = 0xED4245   # 15158332
DISCORD_COLOR_RECOVERED = 0x57F287  # 3066993
RETRY_MAX_DELAY = 30  # 秒，429 與 URLError 兩分支共用
```
- **為什麼**：Discord 改限制或要換顏色時，現在得全文搜尋裸數字，漏改一個就會出現「標題被截斷但描述沒截」這類不一致；URLError 無上限退避在多次重試後會睡過久。
- **優先級**：P2／**工作量**：S

### A7. 兩個測試檔的 import 風格不一致；VPS 測試用「順序 list」餵 mock，換順序就壞
- **檔案:行號**：`tests/test_notify_discord.py:8`（`sys.path.insert(0, .../scripts)` 再 `import update_status`）vs `tests/test_reset_cycles.py:5`（`from scripts.update_status import ...`）；`tests/test_notify_discord.py` 約 L300-330 的 `VpsAggregationTests` 用 `mock_health_check.side_effect = [{...alist...}, {...bili...}, {...twitch...}]`
- **現在的寫法**：同一 repo 兩種 import 路徑寫法並存；VPS 測試依賴 mock 被呼叫的順序剛好是 alist→bili→twitch，一旦 `business_data` 內服務順序調整，測試就假性失敗。
- **推薦的寫法**：統一成 `test_reset_cycles.py` 的 `from scripts.update_status import ...`（或兩個都改 `sys.path`，選一種即可）；mock 改用「依 URL 分派」的函式：
```python
def fake_health(url):
    return {"https://alist.chi.qzz.io/": {...}, "https://bili.chi.qzz.io/": {...}, ...}[url]
mock_health_check.side_effect = fake_health
```
另：`test_notify_discord.py` 檔名與內容不符（內含 Vaultwarden、VPS、health-check 三大類測試），改名或拆檔是 P2、可日後再做。
- **為什麼**：順序敏感的 mock 讓測試失敗時無法分辨是程式壞了還是測試寫法壞了；兩種 import 風格並存會讓新測試不知道抄哪一份。
- **優先級**：P2／**工作量**：S

### A8. `update-status.yml` 的 Discord 通知步驟設 `continue-on-error: true`，通知失敗會被靜默吃掉
- **檔案:行號**：`.github/workflows/update-status.yml`（`Notify Discord about new project failures` 步驟）；另 `deploy` job 無 `timeout-minutes`（`update` job 有 10 分鐘），`deploy-trigger-worker.yml` 用 `checkout@v4`，同一 repo 內與 `update-status.yml` 的 `checkout@v7`／`setup-python@v7` 版本不一致
- **現在的寫法**：`notify_discord.py` 失敗時 workflow 仍顯示綠燈；唯一能發現的途徑是事後人工對照 `alert-state.json`，而報告已指出這類「失敗被遮住」是此生態系最普遍的模式。
- **推薦的寫法**：拿掉 `continue-on-error: true`，讓通知失敗使 job 變紅（`notify-workflow-failure` job 會再發一則 Discord 通知兜底）；或至少在該步驟加 `if: failure()` 的後續檢查步驟。另把兩個 yml 的 `checkout`／`setup-python` 版本統一，並給 `deploy` job 補 `timeout-minutes: 10`。
- **為什麼**：通知腳本本身就是「讓失敗可見」的機制，它自己失敗時絕不能靜默，否則異常與「通知掛了」同時發生卻只看到綠燈。
- **優先級**：P2／**工作量**：S

---

## 6. oci-arm-retry（6 條）

### B1. 兩檔約 70% 相同，先抽「共用模組＋設定 dataclass」而不是維持兩份複製（最小收斂步驟）
- **檔案:行號**：`oci_retry.py` 全檔（349 行）vs `oci_retry_micro.py` 全檔（201 行）；幾乎逐字相同的函式：`get_availability_domain()`（ARM `:39-42`／Micro `:33-36`）、`create_vcn_and_subnet()`（ARM `:60-173`／Micro `:54-140`，含完全相同的開埠 `[22, 80, 443, 8501]` 與 `protocol="6"`）、`try_create_instance()`（ARM `:176-200`／Micro `:143-162`）、`main()` 的重試迴圈骨架（ARM `:271-345`／Micro `:164-197`）
- **現在的寫法**：兩檔各自複製整套邏輯，只差形狀字串、OCPU／記憶體／磁碟、VCN 名稱、SSH key、instance 名稱。結果就是兩檔已經漂移：ARM 有 `verify_instance()`、Discord 成功通知、429／5xx／401／quota 分類，Micro 全部沒有（報告 P1 已列 Micro 無 fail-fast 等後果，這裡只談寫法層面：修 ARM 的 bug 不會自動修到 Micro）。
- **推薦的寫法**：最小步，不動行為，只抽共用部分（不提大重寫、不引新依賴）：
```python
# 新增 oci_common.py：放兩檔共用的東西
@dataclass
class InstanceSpec:
    shape: str            # "VM.Standard.A1.Flex" / "VM.Standard.E2.1.Micro"
    ocpus: int | None     # Micro 沒有 shape_config，給 None
    memory_in_gbs: int | None
    boot_volume_size_in_gbs: int
    instance_name: str
    vcn_name: str
    subnet_name: str
    cidr_block: str       # "10.0.0.0/16" / "10.1.0.0/16"
    ssh_public_key: str

OPEN_PORTS = (22, 80, 443, 8501)
TCP_PROTOCOL = "6"

def get_availability_domain(config, compartment_id): ...
def create_vcn_and_subnet(config, compartment_id, spec): ...
def try_create_instance(config, compartment_id, spec, subnet_id, ad_name, image_id): ...
def get_ubuntu_image(config, compartment_id, spec): ...  # 取代 get_ubuntu_arm_image / get_ubuntu_x86_image

# oci_retry.py / oci_retry_micro.py 各剩約 30 行：定義自己的 InstanceSpec + main() 重試迴圈
```
先只搬「完全相同」的函式與開埠邏輯；`main()` 的例外分類差異（B3）第二步再對齊，不要一次全改。
- **為什麼**：現在任何共用邏輯的修正都要改兩份，改一份忘一份是已經發生的事實（兩檔功能落差就是證據），抽模組後修一次就兩邊生效。
- **優先級**：P1／**工作量**：M（搬函式＋改參數傳遞，需兩檔各跑一次乾跑驗證；建議配合 B3 分兩次做）

### B2. `config = oci.config.from_file()` 寫在模組頂層 import 時就執行，且十幾個設定值是寫死的字串常數
- **檔案:行號**：`oci_retry.py:18-36`（`COMPARTMENT_ID` 寫死、`SSH_PUBLIC_KEY` 寫死、`config = oci.config.from_file()`）；`oci_retry_micro.py:14-30`（同樣結構）
- **現在的寫法**：
```python
# :36 模組載入時就讀 ~/.oci/config，import 測試或被別的腳本 import 時也會立刻讀檔
config = oci.config.from_file()
```
另 `COMPARTMENT_ID` 是直接寫在程式碼裡的 tenancy OCID（報告 P0 已列兩檔 OCID 不一致須核對，這裡只談寫法：它根本不該寫在程式碼裡）。
- **推薦的寫法**：把認證延遲到 `main()` 內建立，並把會變動的設定改從環境變數讀（workflow 已有 `OCI_TENANCY` secret，程式卻不用它）：
```python
def load_config():
    return oci.config.from_file()   # 只在 main() 呼叫

COMPARTMENT_ID = os.environ.get("OCI_TENANCY", "")  # 無 secret 時早點報錯
if not COMPARTMENT_ID:
    raise SystemExit("OCI_TENANCY 未設定")
```
`SSH_PUBLIC_KEY` 同理可改 `os.environ.get("OCI_SSH_PUBLIC_KEY", "<預設值>")`，或至少集中到 B1 的 `InstanceSpec` 裡，不要散在模組頂層。
- **為什麼**：import 時就讀設定檔讓這兩個檔案無法被測試 import（沒 `~/.oci/config` 就直接炸掉），也讓「換 tenancy 要改程式碼」變成必要流程，而 workflow 明明已經有現成的 secret 可以傳。
- **優先級**：P1／**工作量**：S（`load_config()` 一行搬移＋`main()` 開頭呼叫；環境變數部分需同步改 workflow，算 M，若只搬 `load_config()` 則為 S）

### B3. 重試迴圈的 `except Exception` 把所有未知錯誤都當「網路逾時」無限重試
- **檔案:行號**：`oci_retry.py:334-335`；`oci_retry_micro.py:194-195`（Micro 更嚴重：連 `ServiceError` 的非 capacity 分支都只印一句就重試，報告 P1 已列 Micro 無 fail-fast，這裡只談 `except Exception` 寫法）
- **現在的寫法**：
```python
# oci_retry.py:334-335
except Exception as e:
    print(f"⚠️ Network timeout or other error, retrying... ({type(e).__name__})")
# 之後無條件 time.sleep(RETRY_INTERVAL) 繼續迴圈
```
`launch_instance` 內部的程式 bug（如 `LaunchInstanceDetails` 參數拼錯拋 `TypeError`、`AttributeError`）也會被當成暫時性錯誤，配合 `while True`（`:284`）與 workflow 330 分鐘接力，會燒完整輪 Actions 分鐘才停。
- **推薦的寫法**：把 catch 收窄到真正「暫時性」的例外，未知的一律拋出（讓 workflow 變紅、讓 B1 收斂後的共用迴圈只有一種行為）：
```python
except (urllib.error.URLError, TimeoutError, oci.exceptions.RequestException) as e:
    print(f"⚠️ 暫時性連線錯誤，{RETRY_INTERVAL}s 後重試... ({type(e).__name__}: {e})")
# 沒有 except Exception：其他錯誤直接拋出，workflow 失敗、接力停止
```
若擔心 OCI SDK 還有其他暫時性例外，可先加 `except oci.exceptions.ServiceError` 的 5xx 分支（ARM `:313` 已有），而不是用 `Exception` 兜底。
- **為什麼**：重試迴圈的兜底 catch 會把「永遠不會好的 bug」偽裝成「等一下就好」，而這支腳本的設計就是無限跑，兩者相乘等於用 Actions 分鐘燒一個打不開的錯誤。
- **優先級**：P1／**工作量**：S

### B4. `ads[0]`／`images[0]` 直接取第一筆，空清單時拋 `IndexError`；`get_ubuntu_*_image` 用 `raise Exception`
- **檔案:行號**：`oci_retry.py:39-42`（`ads[0].name` 無空檢查）、`:45-58`（`if not images: raise Exception(...)`）、`:56`；`oci_retry_micro.py:33-36`、`:48-51`（相同寫法）
- **現在的寫法**：
```python
def get_availability_domain():
    ads = identity.list_availability_domains(COMPARTMENT_ID).data
    return ads[0].name          # 空清單 → IndexError，訊息完全看不出是「查不到 AD」
# ...
if not images:
    raise Exception("Ubuntu 22.04 ARM image not found")   # 裸 Exception
```
注意 `images` 有檢查、`ads` 沒有，同一檔案內標準不一。（報告已列「只試第一個 AD」的容量問題，那是另一回事；這裡只談空清單會炸出無意義的 `IndexError`。）
- **推薦的寫法**：
```python
def get_availability_domain():
    ads = identity.list_availability_domains(COMPARTMENT_ID).data
    if not ads:
        raise RuntimeError(f"Compartment {COMPARTMENT_ID} 查不到可用區，請檢查 tenancy／權限")
    return ads[0].name

if not images:
    raise RuntimeError("Ubuntu 22.04 ARM image not found")  # Exception → RuntimeError
```
- **為什麼**：`IndexError: list index out of range` 在 Actions log 裡完全無法定位原因，而這支腳本的失敗排查全靠 log 文字；`raise Exception` 則讓呼叫方無法用 `except RuntimeError` 精確接住。
- **優先級**：P2／**工作量**：S

### B5. 時間處理：log 時間戳用無時區的 `datetime.now()`，且 `verify_instance` 的輪詢間隔與重試間隔是裸數字
- **檔案:行號**：`oci_retry.py:286`（`now = datetime.datetime.now().strftime(...)`）、`:233`（`time.sleep(15)`）、`:26`（`RETRY_INTERVAL = 90` 已是常數，是好的對照）；`oci_retry_micro.py:179`、`:197`（相同寫法）
- **現在的寫法**：log 的 `[2026-11-09 11:11:45]` 是 runner 本地時間（GitHub runner 為 UTC），與齊看 log 時習慣的台北時間差 8 小時，對照 Discord 通知或 OCI Console 時間時容易誤判「幾點搶的」；`verify_instance` 的 `time.sleep(15)` 則是裸魔法數字。
- **推薦的寫法**：
```python
from datetime import datetime, timezone
VERIFY_POLL_INTERVAL = 15  # 秒，與 RETRY_INTERVAL 並列放檔頭常數區

# main() 迴圈內：
now = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")
# 或直接用台北時間，與 status repo 的 TAIPEI_TZ 做法一致：
now = datetime.now(ZoneInfo("Asia/Taipei")).strftime("%Y-%m-%d %H:%M:%S 台北")
```
- **為什麼**：無時區的時間戳在跨時區對照時一定會被誤讀一次，而這支腳本的成敗排查高度依賴「幾點幾分發生什麼事」；`sleep(15)` 散在函式內則是下次調校時找不到的數字。
- **優先級**：P2／**工作量**：S

### B6. 兩個 workflow yml 幾乎逐字相同，卻各自維護；Micro 的 relay 步驟與 ARM 的寫法落差大
- **檔案:行號**：`.github/workflows/oci_retry.yml` vs `.github/workflows/oci_retry_micro.yml`（checkout／setup-python／裝 OCI SDK／寫 `~/.oci/config` 四個步驟完全相同；`pip install oci` 未釘版，兩檔皆然）
- **現在的寫法**：兩份 yml 只有腳本檔名與 workflow 名稱不同，其餘逐行相同；Micro 的 relay 用 `if: steps.run_script.outcome != 'success'` 而 ARM 用 `exit_code` 輸出判斷（報告 P1 已列 Micro relay `curl -s` 無 `--fail` 會靜默死亡，這裡只談寫法：兩個 relay 的「成功／逾時／異常」三態判斷邏輯寫法不同，改一邊不會同步到另一邊）。
- **推薦的寫法**：最小步，先把兩份 yml 的 relay／disable 邏輯統一成 ARM 的寫法（`exit_code` 輸出＋`continue-on-error: true`＋`--fail-with-body`，三個步驟逐字相同），再把「寫 OCI config」抽成同一段可複製的步驟區塊；更大步的 reusable workflow 可日後再做，不在此次建議內。另 `pip install oci` 改 `pip install "oci==<目前驗證過的版號>"`，版號以 `pip index versions oci` 或現有執行環境查到的為準，不要憑空猜版號。
- **為什麼**：workflow 是這支腳本的「接力生命線」，兩份寫法不同等於接力邏輯有兩個版本，修好 ARM 的 relay 問題不會自動修到 Micro。
- **優先級**：P2／**工作量**：S（統一 relay 寫法）／M（若連同釘版與驗證一起做）

### oci-arm-retry 補充說明
- `print` 取代 `logging`（兩檔全檔皆是）未單獨列為一條：對這類「跑完即丟」的腳本，`print` 並非錯誤；若要列，建議只在重試迴圈改用 `logging`（分級＋時間戳），工作量 S、P2。但 `print` 本身不影響正確性，故不列入正式條目。

---

## 7. bahasign（6 條）

### B1. 用 `requests.Session` 取代手動拼 `Cookie` 字串
- **檔案:行號**：`bahasign.py:76-113`（登入 `POST` L76-88、取 `BAHARUNE` L95、簽到 `POST` L104-113）
- **現在的寫法**：兩次 `requests.post()` 各自手動拼 header，`L80 "Cookie": "ckAPP_VCODE=7045"`、`L109 f"BAHARUNE={rune}; ckBahamutCsrfToken={csrf_token}"`；登入回傳的 cookie 需手動 `login_response.cookies.get("BAHARUNE")` 再拼回去。
- **推薦的寫法**：一個 `Session` 管 cookie 與共用 header，登入後 cookie 由 Session 自動帶著走：
```python
# Before
login_response = requests.post(LOGIN_URL, headers={"User-Agent": ..., "Cookie": "ckAPP_VCODE=7045"}, ...)
rune = login_response.cookies.get("BAHARUNE")
response = requests.post(SIGNIN_URL, headers={..., "Cookie": f"BAHARUNE={rune}; ckBahamutCsrfToken={csrf_token}"}, ...)

# After
session = requests.Session()
session.headers.update({"User-Agent": LOGIN_USER_AGENT})
session.cookies.set("ckAPP_VCODE", VCODE)  # VCODE 常數化見報告已列項，此處只改寫法
login_response = session.post(LOGIN_URL, data={"uid": account, "passwd": password, "vcode": VCODE}, timeout=TIMEOUT)
login_response.raise_for_status()
if not session.cookies.get("BAHARUNE"):
    raise SignInError("登入失敗，未取得 BAHARUNE Cookie")
session.cookies.set("ckBahamutCsrfToken", csrf_token)
session.headers.update({"X-Bahamut-Csrf-Token": csrf_token})
response = session.post(SIGNIN_URL, data={"action": SIGNIN_ACTION}, timeout=TIMEOUT)
```
- **為什麼**：手動拼 `Cookie` 字串容易漏分號、錯字或漏帶某個 cookie，且每次請求都新建連線。
- **優先級**：P1／**工作量**：S

### B2. 成功判斷要有正面確認，不能只靠「沒有 `error`」
- **檔案:行號**：`bahasign.py:129-148`
- **現在的寫法**：`L129-130` 只檢查 payload 是 dict，`L132 if "error" in payload` 沒命中就一路掉到 `L148 return "success", "✅ **巴哈姆特 簽到成功**"`；回傳 `{}`、空 dict 或結構完全走樣的回應也會被當成功。
- **推薦的寫法**：先向實際 API 回應確認成功欄位名稱（不要猜），只有命中該欄位才回成功，否則視為未預期回應：
```python
# Before
if "error" in payload:
    ...  # 已簽到則 return "already_signed"，否則 raise
return "success", "✅ **巴哈姆特 簽到成功**"

# After（<成功欄位> 須以真實回傳為準，例如 data 內某個簽到成功旗標）
if "error" in payload:
    ...  # 同上
if not <成功欄位為真>:
    raise SignInError(f"簽到回傳未預期：{str(payload)[:200]}")
return "success", "✅ **巴哈姆特 簽到成功**"
```
- **為什麼**：沒有 `error` 不等於成功，空回傳或改版後的回應也會誤報成功。
- **優先級**：P1／**工作量**：S

### B3. 兩段幾乎相同的 `POST`＋包錯邏輯抽成一個小函式
- **檔案:行號**：`bahasign.py:76-93`（登入）與 `bahasign.py:104-118`（簽到）
- **現在的寫法**：兩段都是 `requests.post(..., timeout=TIMEOUT)` → `raise_for_status()` → `except requests.RequestException` → 包成 `SignInError(f"...請求失敗（{type(exc).__name__}: {exc}）")`，只差 URL、header、data 與錯誤字首。
- **推薦的寫法**：
```python
# After
def _post(url, *, headers, data, label):
    try:
        response = requests.post(url, headers=headers, data=data, timeout=TIMEOUT)
        response.raise_for_status()
        return response
    except requests.RequestException as exc:
        raise SignInError(f"{label}請求失敗（{type(exc).__name__}: {exc}）") from exc

login_response = _post(LOGIN_URL, headers={...}, data={...}, label="登入")
response = _post(SIGNIN_URL, headers={...}, data={"action": SIGNIN_ACTION}, label="簽到 API")
```
- **為什麼**：同樣的錯誤包裝寫兩次，日後改 timeout 或錯誤格式時很容易只改一處、漏另一處。
- **優先級**：P1／**工作量**：S

### B4. `main()` 的 `except Exception` 太寬，且成功狀態檔寫入完全沒保護
- **檔案:行號**：`bahasign.py:151-165`（`main()`）、`bahasign.py:182`（`write_text`）、`bahasign.py:30-34`（`success_already_notified`）
- **現在的寫法**：`L154 except Exception` 把 `SignInError`、程式 bug、`ZoneInfo` 失敗、狀態檔讀寫的 `OSError` 全包成同一句「簽到失敗」；`L182 NOTIFICATION_STATE.write_text(...)` 在 `try` 之外，若寫入失敗（權限、磁碟），Discord 明明已送達卻拋 traceback、exit 非零，下次執行會重發通知；`L30-34` 也只捕 `FileNotFoundError`，其他 `OSError` 同樣會一路冒泡被誤標為簽到失敗。
- **推薦的寫法**：
```python
# After
def main():
    try:
        status, message = sign_in()
    except SignInError as exc:
        ...  # 原有失敗通知流程不變
        return 1
    except Exception:
        logging.exception("簽到流程發生未預期錯誤")  # 保留 traceback，不要只印一句話
        return 1
    ...
    try:
        NOTIFICATION_STATE.write_text(taipei_today() + "\n", encoding="utf-8")
    except OSError as exc:
        print(f"❌ 成功通知已送達，但狀態檔寫入失敗：{exc}", file=sys.stderr)
        return 2  # 同「通知未送達」語意：保留下次重試機會，不要當成功結束
```
- **為什麼**：寬捕捉會把程式錯誤偽裝成簽到失敗，誤導除錯方向；狀態檔寫入失敗不該推翻已送達的通知。
- **優先級**：P1／**工作量**：S

### B5. workflow job 沒有 `timeout-minutes`
- **檔案:行號**：`.github/workflows/actions.yml:13-15`（`jobs.bahasign.runs-on: ubuntu-latest`，全檔無 `timeout-minutes`）
- **現在的寫法**：job 沒有任何超時設定；`bahasign.py:18 TIMEOUT = (10, 25)` 只管單次 HTTP 請求，不管 `pip install`、`git push` 或其他步驟卡住。
- **推薦的寫法**：
```yaml
# After
jobs:
  bahasign:
    runs-on: ubuntu-latest
    timeout-minutes: 10
```
- **為什麼**：沒設上限時單一 job 卡住最多可燒 6 小時 Actions 時間，且失敗不會自動浮現。
- **優先級**：P1／**工作量**：S

### B6. 抽模組常數（時區、Discord 顏色、`action`）並補型別提示
- **檔案:行號**：`bahasign.py:26-27`、`bahasign.py:37`、`bahasign.py:51-53`、`bahasign.py:68`、`bahasign.py:111`、`bahasign.py:151`
- **現在的寫法**：`ZoneInfo("Asia/Taipei")`（L27）與 `ZoneInfo("UTC")`（L53）每次呼叫都重建；Discord 顏色 `0x00FF99`／`0xFF5555`（L51）是裸數字；`data={"action": 1}`（L111）的 `1` 沒說明是什麼動作；全檔函式（`taipei_today` L26、`send_discord` L37、`sign_in` L68、`main` L151）皆無型別提示，`sign_in()` 回傳 `tuple[str, str]` 只能靠讀程式才知道。
- **推薦的寫法**：
```python
# After（檔頭常數區，與現有 LOGIN_URL、TIMEOUT 放在一起）
TAIPEI_TZ = ZoneInfo("Asia/Taipei")
UTC_TZ = ZoneInfo("UTC")
DISCORD_COLOR_SUCCESS = 0x00FF99
DISCORD_COLOR_ERROR = 0xFF5555
SIGNIN_ACTION = 1  # 1 = 執行簽到（依巴哈 API 文件註明）

def taipei_today() -> str:
    return datetime.now(TAIPEI_TZ).date().isoformat()

def send_discord(message: str, success: bool) -> bool: ...
def sign_in() -> tuple[str, str]: ...
def main() -> int: ...
```
- **為什麼**：裸數字與重複建 `ZoneInfo` 讓人讀程式時要逐行猜含義，型別提示則讓 `sign_in()` 的回傳結構一目了然。
- **優先級**：P2／**工作量**：S

---

## 8. apktwsign（6 條）

### A1. 用 `requests.Session` 統一帶 header，不要三次 `requests.get` 各傳一次
- **檔案:行號**：`main.py:61-68`（`headers` dict）、`main.py:72-76`、`main.py:106-110`、`main.py:113-117`（三次 `requests.get`）
- **現在的寫法**：`headers` 在 `main()` 內建一次，卻在三次 `requests.get` 各傳 `headers=headers`；每次請求都是新連線，`Cookie` 也是原始字串直接放 header（L62）。
- **推薦的寫法**：
```python
# Before
headers = {"Cookie": APK_COOKIE, "User-Agent": "...Chrome/120..."}
home = requests.get("https://apk.tw/home.php?mod=spacecp", headers=headers, timeout=30)
requests.get(sign_url, headers=headers, timeout=30)
check = requests.get("https://apk.tw/plugin.php?id=dsu_amupper:pper", headers=headers, timeout=30).text

# After
session = requests.Session()
session.headers.update({"Cookie": APK_COOKIE, "User-Agent": "...Chrome/120..."})
home = session.get(HOME_URL, timeout=TIMEOUT)
session.get(sign_url, timeout=TIMEOUT)
check = session.get(CHECK_URL, timeout=TIMEOUT).text
```
- **為什麼**：header 集中在一處設定，日後改 UA 或加 header 不會漏改某次請求，且 Session 可重用連線。
- **優先級**：P1／**工作量**：S

### A2. 四處重複的「失敗結果＋Discord 訊息＋回傳」抽成小函式，URL／timeout／正則一併抽常數
- **檔案:行號**：`main.py:80-96`（Cookie 失效、取不到 formhash）、`main.py:137-145`（Cookie 失效、未知錯誤）、`main.py:73`、`main.py:101-114`（URL 重複寫三次）、`main.py:75,109,116`（`timeout=30` 寫三次）、`main.py:33`（Discord `timeout=10`）、`main.py:87-90`（`re.search` 內聯正則）
- **現在的寫法**：每個失敗分支都重複 `result = "❌ ..."` → `discord_msg = "❌ **APK.TW 簽到失敗：...**"` → `send_discord_msg(discord_msg)` → `return result, False`；`"https://apk.tw/..."` 出現三次，`timeout=30` 出現三次，formhash 正則寫在流程中間。
- **推薦的寫法**：
```python
# After（檔頭）
BASE_URL = "https://apk.tw"
HOME_URL = f"{BASE_URL}/home.php?mod=spacecp"
CHECK_URL = f"{BASE_URL}/plugin.php?id=dsu_amupper:pper"
TIMEOUT = 30
DISCORD_TIMEOUT = 10
FORMHASH_RE = re.compile(r'name="formhash" value="([^"]+)"')

def _fail(reason: str) -> tuple[str, bool]:
    send_discord_msg(f"❌ **APK.TW 簽到失敗：{reason}**")
    return f"❌ {reason}", False

# 呼叫端變成一行
if "退出" not in html:
    return _fail("Cookie 已失效")
if not match:
    return _fail("無法取得 formhash")
```
- **為什麼**：同樣的失敗格式寫四次，改一句文案或加一個失敗原因時很容易寫出不一致的版本。
- **優先級**：P1／**工作量**：S

### A3. 兩處 `except Exception` 都只印 `str(e)`，錯誤型別與 traceback 全丟掉
- **檔案:行號**：`main.py:37-38`（`send_discord_msg`）、`main.py:155-159`（`main()`）
- **現在的寫法**：`L37 except Exception as e: print(f"Discord 發送失敗: {e}")`；`L155 except Exception as e: result = f"❌ {str(e)}"`，timeout、連線錯誤、檔案讀寫錯誤、程式 bug 全變同一句話，有些例外 `str(e)` 甚至是空字串；對照 bahasign 同位置的寫法是 `f"{type(exc).__name__}: {exc}"`，兩 repo 標準不一。
- **推薦的寫法**：
```python
# After
except requests.RequestException as exc:
    print(f"Discord 發送失敗：{type(exc).__name__}: {exc}", file=sys.stderr)
    return False
# main() 尾端
except requests.RequestException as exc:
    return _fail(f"請求失敗（{type(exc).__name__}: {exc}）")
except OSError as exc:
    return _fail(f"狀態檔讀寫失敗（{type(exc).__name__}: {exc}）")
```
- **為什麼**：只剩 `str(e)` 時無法分辨是逾時、斷線還是程式錯誤，收到失敗通知也無從下手。
- **優先級**：P1／**工作量**：S

### A4. `main()` 開頭先驗證 secret 是否缺失，不要讓缺 secret 被誤報成「Cookie 已失效」
- **檔案:行號**：`main.py:10-11`（import 時讀 env）、`main.py:62`（`"Cookie": APK_COOKIE`）、`main.py:80-84`（`"退出" not in html` → Cookie 失效）
- **現在的寫法**：`APK_COOKIE` 為 `None` 時（secret 未設或名字打錯），`headers` 仍照常發請求，首頁自然沒有「退出」，於是回報「❌ Cookie 已失效」，使用者會白忙去重拿 Cookie；`DISCORD_WEBHOOK` 為 `None` 時（L15、L39）也只是靜默 `return False`，失敗通知根本沒地方去。
- **推薦的寫法**：（此條與報告已列的「import 時讀取 secret」相關：驗證邏輯放進 `main()` 開頭，順帶把讀取時機一併收斂到 `main()` 內最乾淨）
```python
# After（main() 第一行）
if not APK_COOKIE:
    return "❌ 未設定 APK_COOKIE", False
if not DISCORD_WEBHOOK:
    print("⚠️ 未設定 DISCORD_WEBHOOK，失敗通知無法送達", file=sys.stderr)
```
- **為什麼**：缺 secret 與 Cookie 過期是兩種完全不同的問題，誤報會讓人修錯方向。
- **優先級**：P1／**工作量**：S

### A5. 成功通知狀態檔的讀寫太脆弱：只捕 `FileNotFoundError`、非原子寫入、寫失敗會觸發第二則失敗通知
- **檔案:行號**：`main.py:8`（`STATE_FILE` 是裸字串）、`main.py:42-57`（`send_success_once`）
- **現在的寫法**：`L45-49` 只捕 `FileNotFoundError`，權限錯誤等其他 `OSError` 會直接炸掉；`L55-57` 先發 Discord、再 `open(STATE_FILE, "w")` 寫入，若寫入失敗，例外冒泡到 `main()` 的 `except Exception`（L155）又發一則失敗 Discord，等於成功通知發了、又補一則失敗通知；對照 bahasign 是用 `Path.read_text`／`write_text`，兩 repo 寫法不一。
- **推薦的寫法**：
```python
# After
from pathlib import Path
STATE_FILE = Path("last_success_notification_date.txt")

def send_success_once(content):
    try:
        last_notified = STATE_FILE.read_text(encoding="utf-8").strip()
    except FileNotFoundError:
        last_notified = ""
    except OSError as exc:
        print(f"⚠️ 狀態檔讀取失敗：{exc}，仍發送通知", file=sys.stderr)
        last_notified = ""
    if last_notified == today:
        ...
        return
    if send_discord_msg(content):
        try:
            STATE_FILE.write_text(today + "\n", encoding="utf-8")
        except OSError as exc:
            print(f"⚠️ 成功通知已送達，但狀態檔寫入失敗：{exc}（下次將重發通知）", file=sys.stderr)
```
- **為什麼**：寫入失敗不該把一次成功簽到翻成失敗通知，狀態檔只是去重手段，不該反過來決定成敗。
- **優先級**：P2／**工作量**：S

### A6. workflow job 沒有 `timeout-minutes`，且「更新 activity」步驟沒有 `if: always()`、grep 失敗會讓整輪狀態停擺
- **檔案:行號**：`.github/workflows/checkin.yml:13-15`（job 無 `timeout-minutes`）、`.github/workflows/checkin.yml:41-51`（Update activity，無 `if: always()`）、`.github/workflows/checkin.yml:43`（`grep "RESULT=" result.txt`）
- **現在的寫法**：job 無超時（`timeout=30` 只管單次請求，`pip install`、`git push` 卡住時無上限）；Update activity 沒有 `if: always()`，且 `L43 grep "RESULT=" result.txt` 在 `main.py` 連 `RESULT=` 都印不出來時（例如 import 失敗）會以非零結束、整個步驟失敗，後面的 Commit 步驟也不跑，README／activity.md 就停在舊狀態。
- **推薦的寫法**：
```yaml
# After
jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      ...
      - name: Update activity
        if: always()
        run: |
          RESULT=$(grep "RESULT=" result.txt 2>/dev/null | sed 's/RESULT=//' || true)
          RESULT=${RESULT:-"⚠️ 執行異常，請查看 Actions 日誌"}
          ...
```
- **為什麼**：超時是防卡死的底線，而 grep 一失敗就整輪狀態不更新，會讓 README 在最需要看的時候停在舊的成功畫面。
- **優先級**：P1／**工作量**：S

### 簽到兩 repo 補充說明
- 兩 repo 的 Discord 顏色值相同（`0x00FF99`／`0xFF5555`）、「當日成功通知只發一次」狀態檔機制相同，但各寫一份；報告「跨 Repository 整體檢查」已提過 Discord 通知格式可共用，故此處不另立條目。
- bahasign 的 `TIMEOUT = (10, 25)`（connect／read 分開）寫法比 apktwsign 三處 `timeout=30` 好，A2 建議 apktwsign 跟進常數化即可，未建議照抄 tuple 形式。
- 未驗證事項：B2 所需的巴哈簽到成功欄位名稱未向真實 API 確認，條目中已標明「須以真實回傳為準，不要猜」；其餘建議皆為靜態寫法層面，無需執行請求即可判斷。

---

## 總結

### 最重要的 3 條建議

1. **舊-1＋新-3：把舊版 `HttpClient` 的重試邏輯搬到新版 free-game**（free-game，P1／M）。新版所有對外請求都只有一次機會，短暫抖動就丟一整輪；舊版已有寫好且有測試覆蓋的重試＋`Retry-After`＋`AmbiguousDeliveryError` 區分，直接搬，不用新寫、不用新依賴。
2. **舊-2：ntfy 用持久化 cursor 取代寫死的 `since=24h`**（free-game，P1／M）。排程中斷超過 24 小時是遲早會發生的事，現在的寫法會讓那段期間的 claimed 訊息永久遺失；舊版的 cursor 讀寫＋截斷不推進邏輯已存在，直接搬。
3. **A1：`status.json`／`alert-state.json` 改原子寫入**（status，P1／S）。每 10 分鐘跑一次的程式用原地覆寫，寫到一半被中斷會留半份 JSON，整張儀表板從「保留上次資料」退化成空白；一個小 helper 就能修掉，而且同一寫法可順手套到 chi-source 兩 repo（C-2、P-3）。

### 整組最值得先做的 1 件事

**先做 free-game 的重試搬運（舊-1＋新-3）。**理由：free-game 是齊最優先的專案；重試邏輯在舊版已經寫好、有測試覆蓋，搬運是「複製已知正確的程式碼」而不是「從零設計」；它直接修掉健檢報告列為 P1 的「無重試」問題（RSS／文章／ntfy／Discord／GitHub Contents 皆單次嘗試），一次改動同時提升新版四個對外呼叫點的穩健性。做完這一件，再做舊-2（ntfy cursor），free-game 的「漏資料」兩大缺口就都補上了。

### 全組共通的寫法壞習慣（前三名）

1. **`except Exception` 兜底**：幾乎每個 repo 都有（free-game cli／notifications、personal `main()`、status `raw()`／`main()`、bahasign `main()`、apktwsign 兩處、oci 重試迴圈）。程式 bug 被偽裝成外部失敗，log 只剩一句話，無法除錯。修正方向一致：只抓預期中的例外種類，程式錯誤留 traceback。
2. **輸出檔案原地覆寫**：status、chi-source、chi-source-personal、bahasign／apktwsign 的狀態檔都是直接 `write_text`。修正方向一致：臨時檔＋`os.replace` 原子寫入，一個 helper 全組通用。
3. **同一份邏輯寫兩份以上**：oci 兩檔 70% 相同、chi-source 兩 repo 共用 helper 各寫一份、free-game 新版四處各寫一次 `urlopen`、status 六張服務卡片逐字複製、bahasign／apktwsign 各寫一份 Discord 通知。修正方向一致：先抽最小的共用小函式／常數，不要一次做大重構。
