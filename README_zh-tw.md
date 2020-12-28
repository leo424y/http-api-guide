# HTTP 介面設計指北

文件主要目的是為大家在設計介面時提供建議，給大家參考 HTTP 或者其他協議/指南已經設計過的內容

**只是建議，不是必須遵從的要求**

大家有什麼問題想法或者建議歡迎 [建立 Issue](https://github.com/bolasblack/http-api-guide/issues/new) 或者 [提交 Pull Request](https://github.com/bolasblack/http-api-guide/compare/)

* [README.md](.) 主要是簡單介紹和列出對設計可能會有幫助的資料，少放一些私貨
* [SUPPLEMENT.md](./SUPPLEMENT.md) 有一些更細節的介面設計方面的我自己的想法，全是私貨

## 目錄

* [HTTP 協議](#user-content-http-協議)
* [URL](#user-content-url)
* [空欄位](#user-content-空欄位)
* [國際化](#user-content-國際化)
* [請求方法](#user-content-請求方法)
* [狀態碼](#user-content-狀態碼)
* [身份驗證](#user-content-身份驗證)
* [超文字驅動和資源發現](#user-content-超文字驅動和資源發現)
* [資料快取](#user-content-資料快取)
* [併發控制](#user-content-併發控制)
* [跨域](#user-content-跨域)
* [其他資料](#user-content-其他資料)
* [其他介面設計指南](#user-content-其他介面設計指南)

## HTTP 協議

### HTTP/1.1

2014 年 6 月的時候 IETF 已經正式的廢棄了 [RFC 2616](http://tools.ietf.org/html/rfc2616) ，將它拆分為六個單獨的協議說明，並重點對原來語義模糊的部分進行了解釋：

* RFC 7230 - HTTP/1.1: [Message Syntax and Routing](http://tools.ietf.org/html/rfc7230) - low-level message parsing and connection management
* RFC 7231 - HTTP/1.1: [Semantics and Content](http://tools.ietf.org/html/rfc7231) - methods, status codes and headers
* RFC 7232 - HTTP/1.1: [Conditional Requests](http://tools.ietf.org/html/rfc7232) - e.g., If-Modified-Since
* RFC 7233 - HTTP/1.1: [Range Requests](http://tools.ietf.org/html/rfc7233) - getting partial content
* RFC 7234 - HTTP/1.1: [Caching](http://tools.ietf.org/html/rfc7234) - browser and intermediary caches
* RFC 7235 - HTTP/1.1: [Authentication](http://tools.ietf.org/html/rfc7235) - a framework for HTTP authentication

相關資料：

* [RFC2616 is Dead](https://www.mnot.net/blog/2014/06/07/rfc2616_is_dead) （[中文版](http://www.infoq.com/cn/news/2014/06/http-11-updated)）

### HTTP/2

HTTP 協議的 2.0 版本還沒有正式釋出，但目前已經基本穩定下來了。

[2.0 版本的設計目標是儘量在使用層面上保持與 1.1 版本的相容，所以，雖然資料交換的格式發生了變化，但語義基本全部被保留下來了](http://http2.github.io/http2-spec/index.html#rfc.section.8)。

因此，作為使用者而言，我們並不需要為了支援 2.0 而大幅修改程式碼。

* [HTTP/2 latest draft](http://http2.github.io/http2-spec/index.html)
* [HTTP/2 草案的中文版](https://github.com/fex-team/http2-spec/blob/master/HTTP2%E4%B8%AD%E8%8B%B1%E5%AF%B9%E7%85%A7%E7%89%88(06-29).md)
* [HTTP/1.1 和 HTTP/2 資料格式的對比](http://http2.github.io/http2-spec/index.html#rfc.section.8.1.3)

## URL

URL 的設計都需要遵守 [RFC 3986](http://tools.ietf.org/html/rfc3986) 的的規範。

URL 的長度，在 HTTP/1.1: Message Syntax and Routing([RFC 7230](https://tools.ietf.org/html/rfc7230)) 的 [3.1.1](https://tools.ietf.org/html/rfc7230#section-3.1.1) 小節中有說明，本身不限制長度。但是在實踐中，伺服器和客戶端本身會施加限制*，因此需要根據自己的場景和需求做對應的調整

* 比如 IE8 的 URL 最大長度是 2083 個字元；nginx 的 [`large_client_header_buffers`](http://nginx.org/en/docs/http/ngx_http_core_module.html#large_client_header_buffers) 預設值是 8k ，整個 [request-line](https://tools.ietf.org/html/rfc7230#section-3.1.1) 超過 8k 時就會返回 414 (Request-URI Too Large)
* [Microsoft REST API Guidelines - 7.2. URL length](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#72-url-length)

**強烈建議 API 部署 SSL 證書**，這樣介面傳遞的資料的安全性才能獲得一定的保障。

## 空欄位

介面遵循“輸入寬容，輸出嚴格”原則，輸出的資料結構中空欄位的值一律為 `null`

## 國際化

### 語言標籤

[RFC 5646](http://tools.ietf.org/html/rfc5646) ([BCP 47](http://tools.ietf.org/html/bcp47)) 規定的語言標籤的格式如下：

```
language-script-region-variant-extension-privateuse
```

1. `language`：這部分使用的是 ISO 639-1, ISO 639-2, ISO 639-3, ISO 639-5 中定義的語言程式碼，必填
    * 這個部分由 `primary-extlang` 兩個部分構成
    * `primary` 部分使用 ISO 639-1, ISO 639-2, ISO 639-3, ISO 639-5 中定義的語言程式碼，優先使用 ISO 639-1 中定義的條目，比如漢語 `zh`
    * `extlang` 部分是在某些歷史性的相容性的原因，在需要非常細緻地區別 `primary` 語言的時候使用，使用 ISO 639-3 中定義的三個字母的程式碼，比如普通話 `cmn`
    * 雖然 `language` 可以只寫 `extlang` 省略 `primary` 部分，但出於相容性的考慮，還是**建議**加上 `primary` 部分
2. `script`: 這部分使用的是 [ISO 15924](http://www.unicode.org/iso15924/codelists.html) ([Wikipedia](http://zh.wikipedia.org/wiki/ISO_15924)) 中定義的語言程式碼，比如簡體漢字是 `zh-Hans` ，繁體漢字是 `zh-Hant` 。
3. `region`: 這部分使用的是 [ISO 3166-1][iso3166-1] ([Wikipedia][iso3166-1_wiki]) 中定義的地理區域程式碼，比如 `zh-Hans-CN` 就是中國大陸使用的簡體中文。
4. `variant`: 用來表示 `extlang` 的定義裡沒有包含的方言，具體的使用方法可以參考 [RFC 5646](http://tools.ietf.org/html/rfc5646#section-2.2.5) 。
5. `extension`: 用來為自己的應用做一些語言上的額外的擴充套件，具體的使用方法可以參考 [RFC 5646](http://tools.ietf.org/html/rfc5646#section-2.2.6) 。
6. `privateuse`: 用來表示私有協議中約定的一些語言上的區別，具體的使用方法可以參考 [RFC 5646](http://tools.ietf.org/html/rfc5646#section-2.2.7) 。

其中只有 `language` 部分是必須的，其他部分都是可選的；不過為了便於編寫程式，建議設計介面時約定語言標籤的結構，比如統一使用 `language-script-region` 的形式（ `zh-Hans-CN`, `zh-Hant-HK` 等等）。

語言標籤是大小寫不敏感的，但按照慣例，建議 `script` 部分首字母大寫， `region` 部分全部大寫，其餘部分全部小寫。

**有一點需要注意，任何合法的標籤都必須經過 IANA 的認證，已通過認證的標籤可以在[這個網頁](http://www.iana.org/assignments/language-subtag-registry)查到。此外，網上還有一個非官方的[標籤搜尋引擎](http://people.w3.org/rishida/utils/subtags/)。**

相關資料：

* ISO 639-1 Code List ([Wikipedia](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes))
* [ISO 639-2 Code List](http://www.loc.gov/standards/iso639-2/php/code_list.php) ([Wikipedia](https://en.wikipedia.org/wiki/List_of_ISO_639-2_codes))
* [ISO 639-3 Code List](http://www-01.sil.org/iso639-3/codes.asp?order=639_3&letter=%25)
* [ISO 639-5 Code List](http://www.loc.gov/standards/iso639-5/id.php) ([Wikipedia](https://en.wikipedia.org/wiki/List_of_ISO_639-5_codes))
* [ISO 639-3 Macrolanguage Mappings](http://www-01.sil.org/iso639-3/macrolanguages.asp) ([Wikipedia](https://en.wikipedia.org/wiki/ISO_639_macrolanguage))
* [Relationship between ISO 639-3 and the other parts of ISO 639](http://www-01.sil.org/iso639-3/relationship.asp)
* [網頁頭部的聲明應該是用 lang="zh" 還是 lang="zh-cn"？ - 知乎](http://www.zhihu.com/question/20797118)
* [IETF language tag - Wikipedia](https://en.wikipedia.org/wiki/IETF_language_tag)
* [語種名稱程式碼](http://www.ruanyifeng.com/blog/2008/02/codes_for_language_names.html) ：文中對帶有方言（ `extlang` ）部分的標籤介紹有誤
* [Language tags in HTML and XML](http://www.w3.org/International/articles/language-tags/)
* [Choosing a Language Tag](http://www.w3.org/International/questions/qa-choosing-language-tags)

### 時區

客戶端請求伺服器時，如果對時間有特殊要求（如某段時間每天的統計資訊），則可以參考 [IETF 相關草案](http://tools.ietf.org/html/draft-sharhalakis-httptz-05) 增加請求頭 `Timezone` 。

```
Timezone: 2016-11-06 23:55:52+08:00;;Asia/Shanghai
```

具體格式說明：

```
Timezone: RFC3339 約定的時間格式;POSIX 1003.1 約定的時區字元串;tz datebase 裡的時區名稱
```

客戶端最好提供所有欄位，如果沒有辦法提供，則應該使用空字元串

如果客戶端請求時沒有指定相應的時區，則服務端預設使用最後一次已知時區或者 [UTC](http://zh.wikipedia.org/wiki/%E5%8D%8F%E8%B0%83%E4%B8%96%E7%95%8C%E6%97%B6) 時間返回相應資料。

PS 考慮到存在[夏時制](https://en.wikipedia.org/wiki/Daylight_saving_time)這種東西，所以不推薦客戶端在請求時使用 Offset 。

相關資料：

* [RFC3339](https://tools.ietf.org/html/rfc3339)
* [tz datebase](http://www.iana.org/time-zones) ([Wikipedia](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones))
* POSIX 1003.1 時區字元串的說明文件
    * [GNU 的文件](https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html)
    * [IBM 的文章](https://www.ibm.com/developerworks/aix/library/au-aix-posix/)

### 時間格式

時間格式遵循 [ISO 8601](https://www.iso.org/obp/ui/#iso:std:iso:8601:ed-3:v1:en)([Wikipedia](https://en.wikipedia.org/wiki/ISO_8601)) 建議的格式：

* 日期 `2014-07-09`
* 時間 `14:31:22+0800`
* 具體時間 `2007-11-06T16:34:41Z`
* 持續時間 `P1Y3M5DT6H7M30S` （表示在一年三個月五天六小時七分三十秒內）
* 時間區間 `2007-03-01T13:00:00Z/2008-05-11T15:30:00Z` 、 `2007-03-01T13:00:00Z/P1Y2M10DT2H30M` 、 `P1Y2M10DT2H30M/2008-05-11T15:30:00Z`
* 重複時間 `R3/2004-05-06T13:00:00+08/P0Y6M5DT3H0M0S` （表示從2004年5月6日北京時間下午1點起，在半年零5天3小時內，重複3次）

相關資料：

* [What's the difference between ISO 8601 and RFC 3339 Date Formats?](http://stackoverflow.com/questions/522251/whats-the-difference-between-iso-8601-and-rfc-3339-date-formats)
* [JSON風格指南 - Google 風格指南（中文版）](https://github.com/darcyliu/google-styleguide/blob/master/JSONStyleGuide.md#%E5%B1%9E%E6%80%A7%E5%80%BC%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B)

### 貨幣名稱

貨幣名稱可以參考 [ISO 4217](javascript:;)([Wikipedia](http://en.wikipedia.org/wiki/ISO_4217)) 中的約定，標準為貨幣名稱規定了三個字母的貨幣程式碼，其中的前兩個字母是 [ISO 3166-1][iso3166-1]([Wikipedia][iso3166-1_wiki]) 中定義的雙字母國家程式碼，第三個字母通常是貨幣的首字母。在貨幣上使用這些程式碼消除了貨幣名稱（比如 dollar ）或符號（比如 $ ）的歧義。

相關資料：

* 《RESTful Web Services Cookbook 中文版》 3.9 節《如何在表述中使用可移植的資料格式》

## 請求方法

* 如果請求頭中存在 `X-HTTP-Method-Override` 或參數中存在 `_method`（擁有更高權重），且值為 `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS`, `HEAD` 之一，則視作相應的請求方式進行處理
* `GET`, `DELETE`, `HEAD` 方法，參數風格為標準的 `GET` 風格的參數，如 `url?a=1&b=2`
* `POST`, `PUT`, `PATCH`, `OPTIONS` 方法
    * 預設情況下請求實體會被視作標準 json 字元串進行處理，當然，依舊推薦設定頭資訊的 `Content-Type` 為 `application/json`
    * 在一些特殊介面中（會在文件中說明），可能允許 `Content-Type` 為 `application/x-www-form-urlencoded` 或者 `multipart/form-data` ，此時請求實體會被視作標準 `POST` 風格的參數進行處理

關於方法語義的說明：

* `OPTIONS` 用於獲取資源支援的所有 HTTP 方法
* `HEAD` 用於只獲取請求某個資源返回的頭資訊
* `GET` 用於從伺服器獲取某個資源的資訊
    * 完成請求後返回狀態碼 `200 OK`
    * 完成請求後需要返回被請求的資源詳細資訊
* `POST` 用於建立新資源
    * 建立完成後返回狀態碼 `201 Created`
    * 完成請求後需要返回被建立的資源詳細資訊
* `PUT` 用於完整的替換資源或者建立指定身份的資源，比如建立 id 為 123 的某個資源
    * 如果是建立了資源，則返回 `201 Created`
    * 如果是替換了資源，則返回 `200 OK`
    * 完成請求後需要返回被修改的資源詳細資訊
* `PATCH` 用於局部更新資源
    * 完成請求後返回狀態碼 `200 OK`
    * 完成請求後需要返回被修改的資源詳細資訊
* `DELETE` 用於刪除某個資源
    * 完成請求後返回狀態碼 `204 No Content`

相關資料：

* [RFC 7231 中對請求方法的定義](http://tools.ietf.org/html/rfc7231#section-4.3)
* [RFC 5789](http://tools.ietf.org/html/rfc5789) - PATCH 方法的定義
* [維基百科](http://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE#.E8.AF.B7.E6.B1.82.E6.96.B9.E6.B3.95)

## 狀態碼

### 請求成功

* 200 **OK** : 請求執行成功並返回相應資料，如 `GET` 成功
* 201 **Created** : 物件建立成功並返回相應資源資料，如 `POST` 成功；建立完成後響應頭中應該攜帶頭標 `Location` ，指向新建資源的地址
* 202 **Accepted** : 接受請求，但無法立即完成建立行為，比如其中涉及到一個需要花費若干小時才能完成的任務。返回的實體中應該包含當前狀態的資訊，以及指向處理狀態監視器或狀態預測的指針，以便客戶端能夠獲取最新狀態。
* 204 **No Content** : 請求執行成功，不返回相應資源資料，如 `PATCH` ， `DELETE` 成功

### 重定向

**重定向的新地址都需要在響應頭 `Location` 中返回**

* 301 **Moved Permanently** : 被請求的資源已永久移動到新位置
* 302 **Found** : 請求的資源現在臨時從不同的 URI 響應請求
* 303 **See Other** : 對應當前請求的響應可以在另一個 URI 上被找到，客戶端應該使用 `GET` 方法進行請求。比如在建立已經被建立的資源時，可以返回 `303`
* 307 **Temporary Redirect** : 對應當前請求的響應可以在另一個 URI 上被找到，客戶端應該保持原有的請求方法進行請求

### 條件請求

* 304 **Not Modified** : 資源自從上次請求後沒有再次發生變化，主要使用場景在於實現[資料快取](#user-content-資料快取)
* 409 **Conflict** : 請求操作和資源的當前狀態存在衝突。主要使用場景在於實現[併發控制](#user-content-併發控制)
* 412 **Precondition Failed** : 伺服器在驗證在請求的頭欄位中給出先決條件時，沒能滿足其中的一個或多個。主要使用場景在於實現[併發控制](#user-content-併發控制)

### 客戶端錯誤

* 400 **Bad Request** : 請求體包含語法錯誤
* 401 **Unauthorized** : 需要驗證使用者身份，如果伺服器就算是身份驗證後也不允許客戶訪問資源，應該響應 `403 Forbidden` 。如果請求裡有 `Authorization` 頭，那麼必須返回一個 [`WWW-Authenticate`](https://tools.ietf.org/html/rfc7235#section-4.1) 頭
* 403 **Forbidden** : 伺服器拒絕執行
* 404 **Not Found** : 找不到目標資源
* 405 **Method Not Allowed** : 不允許執行目標方法，響應中應該帶有 `Allow` 頭，內容為對該資源有效的 HTTP 方法
* 406 **Not Acceptable** : 伺服器不支援客戶端請求的內容格式，但響應裡會包含服務端能夠給出的格式的資料，並在 `Content-Type` 中聲明格式名稱
* 410 **Gone** : 被請求的資源已被刪除，只有在確定了這種情況是永久性的時候才可以使用，否則建議使用 `404 Not Found`
* 413 **Payload Too Large** : `POST` 或者 `PUT` 請求的訊息實體過大
* 415 **Unsupported Media Type** : 伺服器不支援請求中提交的資料的格式
* 422 **Unprocessable Entity** : 請求格式正確，但是由於含有語義錯誤，無法響應
* 428 **Precondition Required** : 要求先決條件，如果想要請求能成功必須滿足一些預設的條件

### 服務端錯誤

* 500 **Internal Server Error** : 伺服器遇到了一個未曾預料的狀況，導致了它無法完成對請求的處理。
* 501 **Not Implemented** : 伺服器不支援當前請求所需要的某個功能。
* 502 **Bad Gateway** : 作為閘道器或者代理工作的伺服器嘗試執行請求時，從上游伺服器接收到無效的響應。
* 503 **Service Unavailable** : 由於臨時的伺服器維護或者過載，伺服器當前無法處理請求。這個狀況是臨時的，並且將在一段時間以後恢復。如果能夠預計延遲時間，那麼響應中可以包含一個 `Retry-After` 頭用以標明這個延遲時間（內容可以為數字，單位為秒；或者是一個 [HTTP 協議指定的時間格式](http://tools.ietf.org/html/rfc2616#section-3.3)）。如果沒有給出這個 `Retry-After` 資訊，那麼客戶端應當以處理 500 響應的方式處理它。

`501` 與 `405` 的區別是：`405` 是表示服務端不允許客戶端這麼做，`501` 是表示客戶端或許可以這麼做，但服務端還沒有實現這個功能

相關資料：

* [RFC 裡的狀態碼列表](http://tools.ietf.org/html/rfc7231#page-49)
* [RFC 4918](http://tools.ietf.org/html/rfc4918) - 422 狀態碼的定義
* [RFC 6585](http://tools.ietf.org/html/rfc6585) - 新增的四個 HTTP 狀態碼，[中文版](http://www.oschina.net/news/28660/new-http-status-codes)
* [維基百科上的《 HTTP 狀態碼》詞條](http://zh.wikipedia.org/wiki/HTTP%E7%8A%B6%E6%80%81%E7%A0%81)
* [Do I need to use http redirect code 302 or 307? - Stack Overflow](http://stackoverflow.com/questions/2467664/do-i-need-to-use-http-redirect-code-302-or-307)
* [400 vs 422 response to POST of data](http://stackoverflow.com/questions/16133923/400-vs-422-response-to-post-of-data)
* [HTTP Status Codes Decision Diagram – Infographic](https://www.loggly.com/blog/http-status-code-diagram/)
* [HTTP Status Codes](https://httpstatuses.com/)

## 身份驗證

部分介面需要通過某種身份驗證方式才能請求成功（這些介面**應該**在文件中標註出來），合適的身份驗證解決方案目前有兩種：

* [HTTP 基本認證](http://zh.wikipedia.org/wiki/HTTP%E5%9F%BA%E6%9C%AC%E8%AE%A4%E8%AF%81)，**最好只在部署了 SSL 證書的情況下才可以使用，否則使用者密碼會有暴露的風險**
* [OAuth 2.0](https://tools.ietf.org/html/rfc6749)
    * [官網](http://oauth.net/2/)
    * [理解OAuth 2.0 - 阮一峰](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html) 以及對[文中 `state` 參數的介紹的修正](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html#comment-323002)
    * [JSON Web Token](https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-25) ，一種 Token 的生成標準
        * [Json Web Tokens: Introduction](http://angular-tips.com/blog/2014/05/json-web-tokens-introduction/)
        * [Json Web Tokens: Examples](http://angular-tips.com/blog/2014/05/json-web-tokens-examples/)

## 超文字驅動和資源發現

REST 服務的要求之一就是[超文字驅動](http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)，客戶端不再需要將某些介面的 URI 硬編碼在程式碼中，唯一需要儲存的只是 API 的 HOST 地址，能夠非常有效的降低客戶端與服務端之間的耦合，服務端對 URI 的任何改動都不會影響到客戶端的穩定。

目前有幾種方案試圖實現這個效果：

* [JSON HAL](http://tools.ietf.org/html/draft-kelly-json-hal-07) ，示例可以參考 [JSON HAL 作者自己的介紹](http://stateless.co/hal_specification.html)
* [GitHub API 使用的方案](https://developer.github.com/v3/#hypermedia) ，應該是一種 JSON HAL 的變體
* [JSON API](http://jsonapi.org/) ，（這裡有 [@迷渡](https://github.com/justjavac) 發起的 [中文版](http://jsonapi.org.cn/) ），另外一種類似 JSON HAL 的方案
* [Micro API](http://micro-api.org/) ，一種試圖與 [JSON-LD](http://json-ld.org/) 相容的方案

目前所知的方案都實現了發現資源的功能，服務端同時需要實現 `OPTIONS` 方法，並在響應中攜帶 `Allow` 頭來告知客戶端當前擁有的操作許可權。

## 資料快取

大部分介面應該在響應頭中攜帶 `Last-Modified`, `ETag`, `Vary`, `Date` 資訊，客戶端可以在隨後請求這些資源的時候，在請求頭中使用 `If-Modified-Since`, `If-None-Match` 等請求頭來確認資源是否經過修改。

如果資源沒有進行過修改，那麼就可以響應 `304 Not Modified` 並且不在響應實體中返回任何內容。

```bash
$ curl -i http://api.example.com/#{RESOURCE_URI}
HTTP/1.1 200 OK
Cache-Control: public, max-age=60
Date: Thu, 05 Jul 2012 15:31:30 GMT
Vary: Accept, Authorization
ETag: "644b5b0155e6404a9cc4bd9d8b1ae730"
Last-Modified: Thu, 05 Jul 2012 15:31:30 GMT

Content
```

```bash
$ curl -i http://api.example.com/#{RESOURCE_URI} -H "If-Modified-Since: Thu, 05 Jul 2012 15:31:30 GMT"
HTTP/1.1 304 Not Modified
Cache-Control: public, max-age=60
Date: Thu, 05 Jul 2012 15:31:45 GMT
Vary: Accept, Authorization
Last-Modified: Thu, 05 Jul 2012 15:31:30 GMT
```

```bash
$ curl -i http://api.example.com/#{RESOURCE_URI} -H 'If-None-Match: "644b5b0155e6404a9cc4bd9d8b1ae730"'
HTTP/1.1 304 Not Modified
Cache-Control: public, max-age=60
Date: Thu, 05 Jul 2012 15:31:55 GMT
Vary: Accept, Authorization
ETag: "644b5b0155e6404a9cc4bd9d8b1ae730"
Last-Modified: Thu, 05 Jul 2012 15:31:30 GMT
```

相關資料：

* [RFC 7232](http://tools.ietf.org/html/rfc7232)
* [HTTP 快取 - Google Developers](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn)
* [RFC 2616 中快取過期時間的演算法](http://tools.ietf.org/html/rfc2616#section-13.2.3), [MDN 版](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching_FAQ), [中文版](http://blog.csdn.net/woxueliuyun/article/details/41077671)
* [HTTP 協議中 Vary 的一些研究](https://www.imququ.com/post/vary-header-in-http.html)
* [Cache Control 與 ETag](https://blog.othree.net/log/2012/12/22/cache-control-and-etag/)

## 併發控制

不嚴謹的實現，或者缺少併發控制的 `PUT` 和 `PATCH` 請求可能導致 “更新丟失”。這個時候可以使用 `Last-Modified` 和/或 `ETag` 頭來實現條件請求，支援樂觀併發控制。

下文只考慮使用 `PUT` 和 `PATCH` 方法更新資源的情況。

* 客戶端發起的請求如果沒有包含 `If-Unmodified-Since` 或者 `If-Match` 頭，那就返回狀態碼 `403 Forbidden` ，在響應正文中解釋為何返回該狀態碼
* 客戶端發起的請求提供的 `If-Unmodified-Since` 或者 `If-Match` 頭與伺服器記錄的實際修改時間或 `ETag` 值不匹配的時候，返回狀態碼 `412 Precondition Failed`
* 客戶端發起的請求提供的 `If-Unmodified-Since` 或者 `If-Match` 頭與伺服器記錄的實際修改時間或 `ETag` 的歷史值匹配，但資源已經被修改過的時候，返回狀態碼 `409 Conflict`
* 客戶端發起的請求提供的條件符合實際值，那就更新資源，響應 `200 OK` 或者 `204 No Content` ，並且包含更新過的 `Last-Modified` 和/或 `ETag` 頭，同時包含 `Content-Location` 頭，其值為更新後的資源 URI

相關資料：

* 《RESTful Web Services Cookbook 中文版》 10.4 節 《如何在伺服器端實現條件 PUT 請求》
* [RFC 7232 "Conditional Requests"](https://tools.ietf.org/html/rfc7232)
* [Location vs. Content-Location](https://www.subbu.org/blog/2008/10/location-vs-content-location)

## 跨域

### CORS

介面支援[“跨域資源共享”（Cross Origin Resource Sharing, CORS）](http://www.w3.org/TR/cors)，[這裡](http://enable-cors.org/)和[這裡](http://code.google.com/p/html5security/wiki/CrossOriginRequestSecurity)和[這份中文資料](http://newhtml.net/using-cors/)有一些指導性的資料。

簡單示例：

```bash
$ curl -i https://api.example.com -H "Origin: http://example.com"
HTTP/1.1 302 Found
```

```bash
$ curl -i https://api.example.com -H "Origin: http://example.com"
HTTP/1.1 302 Found
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: ETag, Link, X-Total-Count
Access-Control-Allow-Credentials: true
```

預檢請求的響應示例：

```bash
$ curl -i https://api.example.com -H "Origin: http://example.com" -X OPTIONS
HTTP/1.1 302 Found
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization, Content-Type, If-Match, If-Modified-Since, If-None-Match, If-Unmodified-Since, X-Requested-With
Access-Control-Allow-Methods: GET, POST, PATCH, PUT, DELETE
Access-Control-Expose-Headers: ETag, Link, X-Total-Count
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: true
```

### JSON-P

如果在任何 `GET` 請求中帶有參數 `callback` ，且值為非空字元串，那麼介面將返回如下格式的資料

```bash
$ curl http://api.example.com/#{RESOURCE_URI}?callback=foo
```

```javascript
foo({
  "meta": {
    "status": 200,
    "X-Total-Count": 542,
    "Link": [
      {"href": "http://api.example.com/#{RESOURCE_URI}?cursor=0&count=100", "rel": "first"},
      {"href": "http://api.example.com/#{RESOURCE_URI}?cursor=90&count=100", "rel": "prev"},
      {"href": "http://api.example.com/#{RESOURCE_URI}?cursor=120&count=100", "rel": "next"},
      {"href": "http://api.example.com/#{RESOURCE_URI}?cursor=200&count=100", "rel": "last"}
    ]
  },
  "data": // data
})
```

## 其他資料

* [Httpbis Status Pages](https://tools.ietf.org/wg/httpbis/)
* [所有在 IANA 註冊的訊息頭和相關標準的列表](http://www.iana.org/assignments/message-headers/message-headers.xhtml)
* [Standards.REST](https://standards.rest/) 裡面收集了不少對 REST API 設計有借鑑意義的標準和規範

## 其他介面設計指南

這裡還有一些其他參考資料：

* [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md) ，很多設計都很有意思，比如：
    * [7.10.2. Error condition responses](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#7102-error-condition-responses)
    * [9.8. Pagination](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#98-pagination)
    * [10. Delta queries](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#10-delta-queries)
    * [13. Long running operations](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#13-long-running-operations)
* [GitHub Developer - REST API v3](https://developer.github.com/v3/)
* [HTTP API Design Guide](https://github.com/interagent/http-api-design/) ，有以下兩點我個人並不建議參考：
    * [Use consistent path formats](https://github.com/interagent/http-api-design/#use-consistent-path-formats)
        還是不建議將動作寫在 URL 中，像文件中的情況，可以將這個行為抽象成一個事務資源 `POST /runs/:run_id/stop-logs` 或者 `POST /runs/:run_id/stoppers` 來解決
    * [Paginate with Ranges](https://github.com/interagent/http-api-design/#paginate-with-ranges)
        確實是一個巧妙的設計，但似乎並不符合 `Content-Range` 的設計意圖，而且有可能和需要使用到 `Content-Range` 的正常場景衝突（雖然幾乎不可能），所以不推薦
* [Best Practices for Designing a Pragmatic RESTful API](http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api)
* [Thoughts on RESTful API Design](http://restful-api-design.readthedocs.org/en/latest/)
* [The RESTful CookBook](http://restcookbook.com/)

[iso3166-1]: javascript:;
[iso3166-1_wiki]: http://en.wikipedia.org/wiki/ISO_3166-1_alpha-2
