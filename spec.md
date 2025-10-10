# 短網址服務規格（GitHub Pages + 404 轉向）

## 概述

此專案為架設在 GitHub Pages 的純靜態短網址服務。它利用 GitHub Pages 的 404 回退行為，將不存在的路徑導向 `404.html`，由前端 JavaScript 讀取 `routes.json` 將短網址鍵（key）對應到目標網址並執行轉址；同時可依 `meta.json` 動態注入 Open Graph 描述。

- 託管環境：GitHub Pages（靜態）
- 進入點：`index.html`（落地頁）、`404.html`（路由與轉址）
- 資料來源：`routes.json`（key → 目標 URL）、`meta.json`（key → Open Graph 描述）
- 自訂網域：由 `CNAME` 設定

## 目標

- 提供可讀且好記的短網址，統一在站台網域下導向完整目標 URL。
- 新增／修改短網址只需調整內容檔（`routes.json`），提交即自動發佈。
- 全程純前端、無伺服器或建置流程依賴。

## 非目標

- 伺服器端 301/302 轉址（本服務為前端 JavaScript 轉向）。
- 保證社群分享預覽（OG 標籤為執行時注入，許多爬蟲不會執行 JavaScript）。
- 提供後台管理 UI（以 Git 編輯檔案、透過 GitHub Pages 發佈）。

## 名詞定義

- 短網址鍵（key）：請求路徑最後一段（例：`/github` → `github`）。
- 目標網址（value）：鍵對應的絕對 URL。
- 路由頁：`404.html`，負責解析鍵並轉向。

## 高階行為流程

1. 使用者造訪 `https://<domain>/<key>`。
2. 由於該路徑並無靜態檔案，GitHub Pages 回傳 `404.html`。
3. `404.html` 之路由腳本：
   - 以 `location.pathname.split('/')` 取最後一段為鍵 `<key>`。
   - 嘗試讀取 `meta.json`，若有 `<key>` 的設定，動態插入 `<meta property="og:description">`。
   - 加上時間戳避免快取地擷取 `routes.json`，查找 `<key>` 對應的目標網址。
   - 若找到，設定 `location.href` 為目標網址；若找不到，導回 `./index.html`。

## 資料契約（Data Contracts）

### routes.json

- 型別：JSON 物件（object）
- Key：短網址鍵（字串）
- Value：目標網址（字串，建議使用 https）

約束：

- Value 應為絕對 URL（建議 `https://`）。
- Key 對大小寫敏感（例如 `fb` 與 `FB` 可同時存在且各自對應）。

範例（節錄自此庫）：

- `"github": "https://github.com/poychang"`
- `"FB": "https://www.facebook.com/poychang.tech"`

版本與部署：不在檔內做版本管理；變更以整個檔案提交發佈。

快取：路由腳本以 `?t=<timestamp>` 查詢字串避免瀏覽器快取。

### meta.json

- 型別：JSON 物件（object）
- Key：短網址鍵（字串）
- Value：物件，支援欄位：
  - `og-description`（字串）：提供 Open Graph 描述文字。

約束：

- 此檔與各鍵皆為可選；未設定時不注入任何 meta。
- 由於為執行時注入，未執行 JavaScript 的爬蟲可能無法讀取。

範例（節錄自此庫）：

- `"build-this-app": { "og-description": "how to use github pages build a short url app" }`

## JSON Schema 與說明（可作為驗證依據）

本節提供可用於驗證 `routes.json` 與 `meta.json` 的 JSON Schema。建議在 CI 中加入驗證步驟，以確保提交的短網址資料符合規格。

說明：

- 兩份 Schema 皆設定 `$schema` 與 `$id` 以利工具引用。
- `routes.json` 的值使用 `format: "uri"` 並搭配 `pattern: "^https?://"` 以鼓勵使用 https（允許 http，但請審慎使用）。
- `meta.json` 中每個鍵對應一個物件。預設 Schema 僅允許 `og-description` 欄位，並設 `additionalProperties: false` 以避免不小心加入未定義欄位；若需擴充，請在提出 spec 變更後將 Schema 同步更新或放寬限制。

routes.json.schema.json：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://s.poychang.net/schemas/routes.json",
  "title": "Routes Mapping",
  "type": "object",
  "description": "短網址鍵（key）與目標網址（value）的對應表。鍵為大小寫敏感字串，值為絕對 URL。",
  "patternProperties": {
    ".+": {
      "type": "string",
      "format": "uri",
      "pattern": "^https?://"
    }
  },
  "additionalProperties": false
}
```

meta.json.schema.json：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://s.poychang.net/schemas/meta.json",
  "title": "Meta Mapping",
  "type": "object",
  "description": "各短網址鍵的 Open Graph 描述設定。",
  "patternProperties": {
    ".+": {
      "type": "object",
      "properties": {
        "og-description": { "type": "string" }
      },
      "required": [],
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

使用建議：

- 若將來需要在 meta 中支援更多欄位（例如 `og-title`、`og-image`），請先在本規格提出變更，更新 Schema 的 `properties` 與 `required`，並視需要將 `additionalProperties` 放寬或維持嚴格。

## 前端路由（404.html）

### 演算法（摘要）

- 取鍵：
  - `const fragment = location.pathname.split('/').pop();`
- 載入中繼資料（盡力而為）：
  - 取得 `meta.json`；若 `meta[fragment]` 存在，建立 `<meta property="og:description">` 並附加至 `<head>`。
- 解析並轉址：
  - 以 `?t=Date.now()` 載入 `routes.json` 避免快取。
  - 若 `routes[fragment]` 存在 → `location.href = routes[fragment]`。
  - 否則 → `location.href = './index.html'`。

### 錯誤處理

- 例外記錄於主控台（`console.log`）。
- 若 `routes.json` 載入失敗（網路或解析錯誤），目前實作不會自動導向 `index.html`，使用者可能停留在空白的 404 文件。營運上請確保檔案可被成功讀取。

### 大小寫敏感

- 鍵查找對大小寫敏感，可同時存在 `fb` 與 `FB`。請於文件與測試予以說明並覆蓋。

### 尾斜線行為

- 當網址以尾斜線結尾（如 `/github/`），`split('/').pop()` 會取得空字串，結果為不匹配 → 轉往 `index.html`。此為既有限制，見「未來改進」。

## 首頁（index.html）

- 極簡落地頁，無額外互動；作為未匹配鍵時的 fallback。

## SEO 與社群分享

- 由於 OG 描述為執行時動態注入，許多社群爬蟲不會執行 JavaScript，因此不保證能生成預覽卡片。
- 必要時請改採伺服器端永久轉址或預先渲染之分享頁面。

## 安全性考量

- 開放轉址風險：僅庫維護者能修改 `routes.json`，請妥善管理權限；惡意變更可能導向釣魚或不安全網站。
- XSS：`og-description` 作為 meta 的 `content` 純文字插入（非 HTML），風險低；不建議新增會插入 HTML 的欄位。
- 混合內容：站台以 HTTPS 服務，建議目標網址亦使用 `https://`。

## 效能考量

- 每次路由會擷取兩個小型 JSON（`meta.json`、`routes.json`）。
- `routes.json` 每次加上時間戳避免快取，換取即時性與網路開銷。
- 若流量很高，建議整合檔案或改為伺服器／CDN 層處理永久轉址。

## 觀測性

- 僅主控台輸出；預設不含分析與錯誤回報。
- 可視需求導入隱私友善的分析工具或錯誤追蹤服務（需額外第三方或後端）。

## 營運操作

### 新增／更新短網址

1. 編輯 `routes.json`：
   - 加入或更新一組鍵值：`"<key>": "https://destination"`。
2. 視需要編輯 `meta.json`：
   - `{ "<key>": { "og-description": "描述文字" } }`
3. 提交到預設分支，GitHub Pages 將自動發佈（依庫設定）。

### 移除短網址

- 從 `routes.json` 刪除該鍵，並同步移除 `meta.json` 對應項目（若有）。

### 自訂網域

- `CNAME` 檔案記載自訂網域；請依 GitHub Pages 文件正確設定 DNS。

## 驗收標準（Acceptance Criteria）

- 當 `routes.json` 中存在某鍵，使用者造訪 `/<key>` 時，瀏覽器應導向其目標網址。
- 當 `routes.json` 中不存在某鍵，使用者造訪 `/<key>` 時，瀏覽器應導向 `./index.html`。
- 對大小寫不同的鍵（如 `/fb` 與 `/FB`），應能各自正確解析。
- 不帶尾斜線的有效鍵應轉址；帶尾斜線時（現況）應回落至 `index.html`。
- 若 `meta.json` 為該鍵提供資訊，文件 `<head>` 中應在執行時新增 `<meta property="og:description">`。

## 邊界情況（Edge Cases）

- 尾斜線（`/key/`）→ 不匹配 → 落地頁。
- 空鍵（根路徑 `/`）→ GitHub Pages 直接回 `index.html`，不經 404 路由。
- 多段路徑（`/foo/bar`）→ 只取最後一段 `bar` 作為鍵。
- URL 編碼片段（如 `%2F`）目前未解碼後再查找；建議鍵使用 ASCII 安全字元。
- `routes.json` 取用失敗 → 不會自動轉向；使用者可能停留在空白頁面。

## 重要檔案清單（與執行時相關）

- `404.html`：前端路由與轉址、動態注入 meta。
- `routes.json`：短網址鍵 → 目標 URL 對應表。
- `meta.json`：每鍵的 Open Graph 描述設定（可選）。
- `index.html`：未匹配時的落地頁。
- `CNAME`：自訂網域設定。

## 未來改進（不破壞相容的增強）

- 尾斜線相容：將 `/key/` 視為 `/key`（可在取鍵時過濾空片段）。
- 對鍵先行 `decodeURIComponent`，允許更廣泛的鍵字元集合（仍需風險評估）。
- 規範大小寫（選用）：改為不分大小寫並定義唯一寫法。
- 改善社群預覽：改為預渲染頁面或伺服器端轉址與 meta；執行時注入通常不足。
- 快取策略：合併設定檔、引入內容雜湊，或遷移至可控回應標頭之託管平台。
- 錯誤體驗：當設定檔載入失敗時提供友善提示，而非空白頁。

## 相容性與限制

- 純前端：所有邏輯在瀏覽器執行；不使用 serverless 或動態後端。
- 瀏覽器需求：需支援 `fetch` 與 ES6 功能之現代瀏覽器。若要支援舊版瀏覽器，需導入轉譯或 polyfill（不在本規格範圍）。

---

本文件為後續 Spec-Driven Development 的行為契約與資料規格。若有行為變更（例如尾斜線相容、大小寫規範化等），請先在此更新規格，再依規格實作並驗收。
