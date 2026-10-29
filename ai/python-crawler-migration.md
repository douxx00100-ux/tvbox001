# Python 爬蟲舊 API 遷移

本文件適用於本次移除舊 Python API 後的原始碼。這些異動尚未對應已發布的模組版本；實際使用需搭配包含此異動的 Python 模組與相容 App，建置方式見[執行環境模組](runtime-modules.md)。爬蟲仍使用 CPython 3.10，入口維持同步 `def`。

## 替換清單

| 舊寫法 | 現行寫法 | 差異 |
| --- | --- | --- |
| `self.fetch()`、`self.post()` | `self.net.req()`、`self.net.json()` | 使用宿主注入的網路，不再回傳 requests.Response |
| `self.str2json()`、`self.json2str()` | `json.loads()`、`json.dumps()` | 明確 import json；入口直接回傳 dict／list |
| `getCache()`、`setCache()`、`delCache()` | `self.local` 或 `self.net` 的 TTL 快取 | 依持久化設定或限時資料選擇；這三個舊方法已在先前共享 API 改版移除 |
| 共用 `cache/py`、`SourceFileLoader` 載入 | `self.loadModule()`、`self.loadSpider()` | 方法保留，只使用站點隔離的 Runtime |
| `Runner`、`Trigger` | 繼承 `base.spider.Spider`，由 App 呼叫 | 舊轉接檔已移除，不自行包裝宿主橋接 |
| `self.getProxyUrl(False)` | `self.getProxyUrl(local=False)` | 第一個參數是代理參數集合，不再接受布林值 |
| `self.html(content)` | `lxml.etree.HTML(content)` | 自行 import lxml，不再經基底轉呼叫 |
| `self.regStr(reg, src, group)` | `re.search()` 後取得群組 | 呼叫端保留找不到時回傳空字串等規則 |
| `self.removeHtmlTags(src)` | 解析 HTML 後擷取文字 | 原方法只用正則刪除標籤，與解析 HTML 的結果可能不同 |
| `self.cleanText(src)` | 來源自己的 `re` 過濾規則 | 原方法只刪除部分 Emoji 區段，沒有通用文字清理合約 |

`getDependence()`、`loadModule()`、`loadSpider()` 仍可使用。上述四個 HTML／文字工具已從基底移除，lxml 與標準庫 re 仍可直接 import。不要新增同名舊方法或捕捉 AttributeError 後改走另一套相容流程。

## 網路請求

`self.net` 在 `init()` 前由 App 注入。請求使用 options 字典，時間單位為毫秒。

| 原 requests 用法 | 遷移方式 |
| --- | --- |
| `response.json()` | 直接使用 `self.net.json(url, options)` |
| `response.text` | `self.net.req(url, options)["content"]`，先檢查 error 與 code |
| `response.content` | options 設 `buffer: 3`，成功回應的 content 為 bytes |
| `response.status_code` | `response["code"]`；網路錯誤先檢查 error |
| `response.headers` | `response["headers"]`；重複 header 的值可能為陣列 |
| `params={...}` | options 的 `params`；保留既有 query，字典值可用陣列傳送同名參數，None 會略過 |
| `timeout=5` | `timeout: 5000`；整次請求期限另用 `callTimeout` |
| `allow_redirects=False` | `redirect: 0`；預設跟隨重新導向 |
| `cookies={...}`、requests.Session | 使用 Cookie API 或 `self.net.session()` |
| `stream=True` | `self.net.openStream()`；以 with／finally 關閉 reader |
| `verify=False` | 沒有同名 options；TLS 行為由宿主網路層決定，不原樣搬入此參數 |

新 `timeout` 預設為 10000 毫秒，與舊 fetch／post 的 5 秒不同。需要維持舊期限時明確傳 5000；`timeout` 管理連線／讀寫，`callTimeout` 管理整次請求。

### GET 與 JSON

只需要 JSON 時直接取得解析結果。網路失敗、HTTP 非 2xx 或無效 JSON 會拋出例外，不再額外呼叫 `.json()` 或 `json.loads()`。

```python
data = self.net.json(url, {
    "params": {"keyword": key, "page": pg},
    "headers": headers,
    "timeout": 5000,
})
```

需要狀態碼、錯誤正文或文字時使用 req。HTTP 非 2xx 仍保留回應；不要把登入頁或錯誤偽裝成成功空列表。

```python
response = self.net.req(url, {"headers": headers, "timeout": 5000})
if response.get("error"):
    raise RuntimeError(response["error"])
if not 200 <= response["code"] < 300:
    raise RuntimeError(f"HTTP {response['code']}")
text = response["content"]
```

req 預設回傳文字，編碼依 HTTP 回應處理，不再由基底強制指定 UTF-8。需要自行解碼時使用 `buffer: 3`，檢查成功後對 bytes 呼叫 `decode()`。

### POST 與正文

舊 `post(data={...})` 通常送出表單，新網路 API 的 `postType` 預設為 json。遷移表單時必須明確指定 form。

```python
data = self.net.json(url, {
    "method": "POST",
    "postType": "form",
    "data": {"name": name, "page": pg},
    "headers": headers,
    "timeout": 5000,
})
```

| 正文格式 | options |
| --- | --- |
| JSON 值 | `method: "POST", postType: "json", data: value` |
| 表單 | `method: "POST", postType: "form", data: {...}` |
| multipart 文字欄位 | `method: "POST", postType: "form-data", data: {...}` |
| 已編碼文字 | `method: "POST", body: text`，並提供 Content-Type |
| 原始 bytes | `method: "POST", body: list(payload)`，並提供 Content-Type |

`data` 與 `body` 互斥；GET／HEAD 不接受正文。form／form-data 的 data 必須是物件，欄位經共用網路層轉字串並去除前後空白。需要重複表單欄位或精確保留文字時，自行編碼後以 body 傳送；form-data 此處只提供文字欄位，不對應 requests 的 files 參數。

```python
response = self.net.req(url, {
    "method": "POST",
    "headers": {"Content-Type": "application/octet-stream"},
    "body": list(payload),
    "buffer": 3,
})
```

`buffer` 只控制回應格式；原始二進位正文仍須轉成 0～255 整數陣列。req 的網路與 HTTP 錯誤檢查方式與 GET 相同。

### Cookie、Session 與生命週期

`self.net.getCookie(url)`／`setCookie(url, cookie)` 讀寫 WebView Cookie，寫入接受單筆 Set-Cookie 格式；請求設 `cookie: True` 才會自動帶入與更新。App 的 User-Agent 可由 `self.net.getUserAgent()` 取得，再按來源需要放入 headers，不會自動套用到所有請求。

需要隔離 Cookie 時建立 Session，請求會自動使用該 Session 的 Cookie，並在結束時關閉。

```python
with self.net.session() as session:
    session.json(login_url, {
        "method": "POST",
        "postType": "form",
        "data": credentials,
    })
    data = session.json(data_url)
```

Session 關閉只釋放自己的請求、Cookie 與快取，不關閉父站點。站點的 self.net 由 App 管理；子爬蟲不要自行關閉共用 self.net。串流以 `reader.read(size)` 取得 bytes，結束時回傳 None，reader 也須以 with／finally 關閉。

## Local 與 TTL 快取

小型持久化設定使用 Local，Python 直接存 JSON 值，不先序列化成字串。

```python
self.local.set("preferences", {"category": "movie", "page": 1})
preferences = self.local.get("preferences", {})
self.local.delete("preferences")
```

Local 按 Site.key 隔離；不存在時回傳 None，或 get 的第二參數。set／delete 回傳 None，呼叫時的例外會向外傳遞，不能再比較舊的 `"succeed"`／`"failed"`。App 清快取不會刪除 Local，改 Site.key 後也不會自動沿用原站點資料。

舊 `/cache` 與 Local 是不同資料空間，不會自動搬移。Local 不解析 `expiresAt`，即使欄位時間已過仍原樣回傳；不要直接搬入舊資料並假定具有過期判斷。重要設定需由來源明確處理遷移或透過配置重新提供，不加入全域舊快取 fallback。

需要期限的資料使用 cached，由 loader 產生 JSON 值。

```python
catalog = self.net.cached(
    "catalog",
    {"ttl": 60000, "stale": False},
    lambda: self.net.json(catalog_url),
)
```

- ttl 為必填毫秒數；0 不快取。stale 預設 True，可能先回過期值再背景更新；需要等待新資料時設 False。
- persist 預設 False；只為可公開持久化的資料啟用，Session 不支援 persist。Cookie、token、簽名播放網址不寫入長期 Local 或持久快取。
- `self.net.peek(key)` 可讀到過期值，不能拿來取代有效期判斷。`self.net.clearCache(key)` 清單筆，省略 key 清整個站點快取。
- cached 不是 setCache 的同參數替代；儲存設定用 Local，限時資料才交給 loader 與 TTL。

## 依賴與子爬蟲

保留 loadModule 回傳的模組物件，不再載入後用 `importlib.import_module("helper")` 找全域同名模組，也不手動修改 sys.path、sys.modules 或共用 cache/py。

```python
def init(self, extend=""):
    self.helper = self.loadModule("helper")
    self.child = self.loadSpider("child")
    self.child.init(extend)
```

loadModule／loadSpider 的名稱相對於入口爬蟲來源解析，模組與檔案快取按站點、來源及載入世代隔離。loadSpider 不會自動呼叫子爬蟲 init；在父爬蟲 init 中載入時，子爬蟲共用目前站點的 net、local、js 與 Dialog，不建立另一份站點資源。

使用 `self.js.eval()` 仍需配套的 App 與 QuickJS 模組，Python 模組本身不另建 JavaScript 引擎。

getDependence 仍可宣告依賴。若使用一般 import 且名稱可能與全域模組碰撞，在 getDependence 宣告後於 init 中 import，或直接採用 loadModule。

```python
def getDependence(self):
    return ["helper"]

def init(self, extend=""):
    import helper
    self.helper = helper
```

不要直接建構脫離 App Runtime 的爬蟲，再期待 loadModule／loadSpider 從舊共用目錄載入。Runner／Trigger 不再使用；以 `from base.spider import Spider as BaseSpider` 匯入基底並定義 `class Spider(BaseSpider)`，由 App 完成載入、注入、呼叫與釋放。

## JSON 與入口回傳

只在解析正文或建立指定格式的請求時使用標準庫 JSON。

```python
import json

data = json.loads(text)
body = json.dumps(data, ensure_ascii=False)
```

首頁、分類、搜尋、詳情與播放結果直接回傳 dict／list，由 app.py 序列化，不回傳額外 json.dumps 後的字串。這次刪除的是基底的兩個舊 helper；app.py 內部的同名橋接函式仍在使用。

```python
def detailContent(self, ids):
    return {"list": [self.net.json(self.detail_url, {"params": {"id": ids[0]}})]}
```

布林接口仍回傳 bool，localProxy 仍回傳 `[status, mime, body, headers]`；直播文字格式也維持原有合約，不套用一般資料入口的 JSON 包裝。

## HTML 與文字處理

HTML 解析直接使用 lxml，並處理空內容無法建立樹的情況。

```python
from lxml import etree

html_text = "<div>你好 &amp; <b>世界</b></div>"
root = etree.HTML(html_text)
plain_text = "" if root is None else "".join(root.itertext())
```

這也可取代需要擷取文字的 removeHtmlTags 用法。解析器會處理 `&amp;` 等實體，與原本 `re.sub('<.*?>', '', src)` 的結果不完全相同；script／style 內容與段落空白需按來源需求處理。

正則擷取由來源明確指定群組及無匹配結果，以下保留舊 regStr 找不到時回傳空字串的行為。

```python
import re

source = "ID:123"
match = re.search(r"ID:(\d+)", source)
value = match.group(1) if match else ""
```

cleanText 的過濾範圍由來源決定。確實需要原有行為時，可直接使用以下明確規則；它只涵蓋舊方法的四段 Unicode 範圍，不是完整的 Emoji 過濾。

```python
import re

title = "電影😀"
pattern = re.compile('[\U0001F600-\U0001F64F\U0001F300-\U0001F5FF\U0001F680-\U0001F6FF\U0001F1E0-\U0001F1FF]')
title = pattern.sub("", title)
```

## 代理網址

第一個參數傳入業務參數集合，是否使用本機位址以 local 指定。

```python
local_url = self.getProxyUrl({"id": "a+b &=中文"})
lan_url = self.getProxyUrl({"id": "a+b &=中文"}, local=False)
lan_base = self.getProxyUrl(local=False)
```

App 自動加入 do／siteKey 並編碼 query；呼叫端不能覆寫這兩個保留參數，也不要再手動附加站點識別或重複編碼。舊的 `getProxyUrl(False)`／`getProxyUrl(True)` 會失敗。

## 驗證

先移除來源對上述舊方法與轉接檔的引用，再核對表單／JSON 正文、逾時單位、錯誤回應、Cookie 隔離、快取期限及依賴載入。必要時以 `python -m py_compile` 檢查來源語法。

在配套 App 與 Python 模組中驗證首頁／搜尋 → 詳情 → 每集播放值 → 播放網址；使用 Session、快取、代理或子爬蟲的來源，另驗證各自的隔離與釋放。主機測試或取得播放網址不代表已在裝置呈現影片畫面。
