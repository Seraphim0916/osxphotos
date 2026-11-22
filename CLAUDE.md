# CLAUDE.md — osxphotos（Vincent 的 fork）

這是 RhetTbull/osxphotos 的 fork（`Seraphim0916/osxphotos`，上游遠端 `upstream`）。用途是整理 Vincent 自己 MacStudio 上的 iCloud 照片圖庫。這份檔只給代理看，不送回上游（同目錄 `AGENTS.md` 連向本檔）。

## 一、這個工具是什麼（兩篇文章的重點）

來源：
- 「osxphotos Python API 深度教程：構建你自己的照片管理工具」，agent.csdn.net/6a4f197e662f9a54cb8c8695.html（2026-04-02）
- 「OSXPhotos：macOS 照片庫的全能管理工具」，blog.csdn.net/iotavault252/article/details/162307604（2026-06-25）

重點：
- 用命令列與 Python API 操作 macOS「照片」App 的圖庫：直接讀圖庫資料庫，不用經過照片 App 的畫面，所以查詢、批次匯出、整理相簿、讀中繼資料都能自動化。
- 可讀的中繼資料：EXIF、位置、人物（人臉）、關鍵字、描述、Live Photo、RAW＋JPEG 配對、編輯版本。
- 安裝：建議用 uv 裝成獨立工具（`uv tool install --python 3.13 osxphotos`）；也有 pip、Homebrew。這台目前裝的是 0.77.2（`~/.local/share/uv/tools/osxphotos`）。
- 常用命令列：
  - 查詢：`osxphotos query`，可篩檔名、標籤、人物、日期、格式（`--uti`）、只照片／只影片、截圖、可能重複（`--duplicate`），可輸出 `--json`／`--count`。
  - 依查詢結果加進相簿：`--add-to-album 相簿名`（例：`--only-movies --min-size 200MB --add-to-album 大影片`）。
  - 匯出：`osxphotos export 目的地 --export-by-date`（依年月日分資料夾），可用模板自訂路徑與檔名。
  - 批次改中繼資料：`osxphotos batch-edit`（關鍵字、描述等）。
- Python API 核心：
  - `PhotosDB()`：打開圖庫（預設系統圖庫；本機要指定 `PhotosDB("~/Pictures/Photos Library.photoslibrary")`）。`db.photos()` 取全部照片，`db.albums`、`db.persons` 等取相簿、人物。
  - `PhotoInfo`：每張照片一個物件，`path`、`date`、`location`、`keywords`、`exif_info`、`uti_original`、`ismovie`、`fingerprint`、`albums` 等屬性。
  - `PhotoExporter`：匯出照片（文章範例用 `use_photos_export=True` 讓照片 App 匯出）。
  - 文章示範的 `photo.keywords = ...; photo.update()` 屬於寫入圖庫；寫入一律走下面「二」的規則。
- 延伸用途：自動備份、依人臉／場景分類、批次整理中繼資料。官方文件在 `docsrc/source/`、範例在 `examples/`、API 說明在 `API_README.md`。

## 二、在 Vincent 圖庫上的硬規則

- **不准刪除照片**，也不清「最近刪除」。要丟的只加進相簿，讓 Vincent 自己看過再刪。
- 只動主圖庫 `~/Pictures/Photos Library.photoslibrary`；`Photos Library2.photoslibrary` 不碰。
- 加相簿前先產出名單（UUID 清單＋可讀的 TSV），試跑只讀查詢，給數字再執行；執行後用 `--album 相簿 --json` 核對相簿內容與名單完全一致（數量、UUID、名單外 0）。
- 共享圖庫（`shared` 為 True）的那一份不動，可能是別人分享的。
- 拿不準的（只有畫面相似、檔名不同；或要移的那份有編輯、標了最愛）不放，列給 Vincent 看。

## 三、這台實際踩過的坑（2026-10-27）

- **加相簿會被系統擋**：`--add-to-album` 透過 AppleScript 控制照片 App。從代理的 shell 執行會失敗（`User canceled (-128)`），因為沒有「自動化」權限。做法：名單產好後，請 Vincent 在「終端機」跑加相簿那一行（終端機已允許控制照片）；照片 App 可能會被重開（log 會出現 `killall`）。
- **沒有本機原檔的照片會被用檔名找**：只在 iCloud、`path` 為 None 的照片，加相簿時會依檔名比對，結果把**所有同名副本**（含要保留的那份）一起加進去。實例：`IMG_0044`、`IMG_0046`、`IMG_0242` 各 5 份被誤加。→ 名單只放 `path` 不是 None 的照片，或執行後逐一核對 UUID。
- **`--duplicate` 很寬鬆**：主圖庫標了 2,296 項「可能重複」，大多是同一張匯入兩次、一份在共享圖庫。真重複要用「拍攝時間＋原檔大小＋長寬＋是否影片」分組，再加照片 App 自己的相似比對（`Photos.sqlite` 的 `ZASSET.ZDUPLICATEPERCEPTUALMATCHINGALBUM`，相同值同一組）。
- **只篩「重複＋JPEG」會兩份都選到**：同組兩份都是 JPEG 很常見，要逐組決定留哪份。
- **文字辨識（OCR）預設不是中文**：`osxphotos.text_detection.detect_text()` 沒有語言參數，中文截圖會辨識成亂碼。要辨識中文，自己用 Vision 的 `VNRecognizeTextRequest` 並設 `recognitionLanguages=["zh-Hant","en-US"]`。
- 唯讀查資料庫可直接 `sqlite3 -readonly ".../database/Photos.sqlite"`，但欄位隨 macOS 版本變（這台 Photos 資料庫版本 5001），以 osxphotos 的結果為準。

## 四、目前的整理工作

（新批次暫存與名單一律放 `~/.ssot/local/staging/<YYYY-MM-DD-主題>/`，因檔案放置政策已停寫舊 `~/.openclaw/staging/`；下方舊路徑僅為 2026-10-27 歷史紀錄。）
1. **重複照片 → 相簿「重複待刪除」**：每組留一份。留的優先序：Apple 原生格式（HEIC／HEIF、HEVC、ProRAW、DNG、原況照片）> 有編輯或最愛 > 所在相簿多 > 原檔大；非 Apple 格式（JPEG、PNG 轉檔等）優先放進相簿。2026-10-27 第一輪已放 33 份（歷史名單 `~/.openclaw/staging/2026-10-27-photos-dup/`）。
2. **三年前的繳款紀錄、匯款紀錄 → 相簿「過期匯款紀錄」**：拍攝日期早於執行日往前三年。候選：截圖、照片 App 標籤 Document／Receipt／Handwriting；再用中文 OCR 找關鍵字（匯款、轉帳、繳款、繳費、轉出、交易成功、付款成功、收款人、ATM 等）判定（2026-10-27 歷史目錄 `~/.openclaw/staging/2026-10-27-photos-remit/`）。相簿裡已有 Dot 放的 615 份（含三年內的），不要移出，有疑慮列給 Vincent。
