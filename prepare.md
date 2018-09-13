# CH 7 AJAX

> AJAX 異步 JavaScript 和 XML

[MDN↗](https://developer.mozilla.org/zh-TW/docs/Web/Guide/AJAX/Getting_Started)

AJAX 代表 Asynchronous JavaScript And XML，即非同步 JavaScript 及 XML。簡單地說，AJAX 使用 XMLHttpRequest 物件來與伺服器進行通訊。它可以傳送並接收多種格式的資訊，包括 JSON、XML、HTML、以及文字檔案。AJAX 最吸引人的特點是「非同步」的本質，這代表它可以與伺服器溝通、交換資料、以及更新頁面，且無須重整網頁。本章考察從伺服器收發數據最快的技術，以及最有效的數據編碼格式。

AJAX 的兩個特點：

- 無須重新載入整個頁面，便能對伺服器發送請求
- 分析並運用 XML 文件

## 1. Data Transmission 數據傳輸

AJAX 與伺服器通訊而不重載當前頁面的方法，數據可從伺服器獲得或發送給伺服器。有多種不同的方法構造這種通訊通道，每種方法都有自己的優勢和限制。

### 1.1 Requesting Data 請求數據

有五種常用技術用於向伺服器請求數據：

- XMLHttpRequest (XHR)
- Dynamic script tag insertion (動態腳本標籤等入)
- iframes
- Comet
- Multipart XHR

在現代高性能 JavaScript 中使用的三種技術是 XHR，動態腳本標籤插入和多部分的 XHR。使用 Comet 和iframe (作為數據傳輸技術)往往是極限情況，不在這裡討論。

### 1.2 XMLHttpRequest

[WIKI↗](https://zh.wikipedia.org/wiki/XMLHttpRequest)

JavaScript 著名的 古老 API，頁面能透過它操作 HTTP 請求進行網路作業，最大的好處在於可以動態地更新網頁，它無需重新從伺服器讀取整個網頁，是 AJAX 的重要組成部分。

[example](https://codepen.io/amelieljit/pen/wEyLWy)

當使用 XHR 請求數據時，可以選擇 POST 或 GET。如果請求不改變伺服器狀態只是取回數據(又稱作冪等動作)則使用 GET。 GET 請求被 cache 起來，如果你多次提取相同的數據可提高性能。只有當 URL 和參數的長度超過了 2048 個字符時才使用 POST 提取數據。因為 IE 限制 URL 的長度，過長將導致請求(參數)被截斷。

### 1.3 Dynamic script tag insertion 動態脚本標籤插入

<!-- [https://github.com/letiantian/how-to-load-dynamic-script](https://github.com/letiantian/how-to-load-dynamic-script) -->

克服了 XHR 的最大限制：它可以從不同域的伺服器上獲取數據，而不是實例化一個專用對象，用 JS 創建了一個新腳本標籤，並將它的源屬性設置為一個指向不同域的 URL。

    var scriptElement = document.createElement('script')
    scriptElement.src = 'http://any-domain.com/javascript/lib.js';
    document.getElementsByTagName('head')[0].appendChild(scriptElement);

與 XHR 相比只提供更少的控制。你不能通過請求發送 headers。參數只能通過 GET方 法傳遞，不能用 POST。也不能設置 timeouts 或 retry；實際上也不會知道它是否失敗了。必須等待所有數據返回之後才可以訪問它們。你不能訪問 response headers 或者像訪問整個 response string。

- 動態脚本標籤插入允許跨域請求和本地執行 javascript 和 JSON 但是它的接口不那麼安全。

### 1.4 Multipart XHR 多部分 XHR

MXHR 允許客戶端只使用一個 HTTP 請求就可以從伺服器端向客戶端傳送多個資源，它通過在伺服器端將資源(CSS文件，HTML 片段，JavaScript 代碼，或 base64 編碼的圖片)打包成一個雙方約定的字符串並發送到客戶端，然後用 JavaScript 代碼處理這個長字符串，並根據它的 mime-type 類型和傳入其他 header 解析出每個資源。但是以這種技術獲得的資源不能夠被瀏覽器緩存，不過在某些情況下 MXHR 依然能顯著提高頁面的整體性能：

- 網頁包含許多其他地方不會用到的資源(所以不需要緩存)，尤其是圖片。
- 網站為每個頁面使用了獨一無二的打包的 JS 或 CSS 文件以減少 HTTP 請求，因為它們對每個頁面來說是獨一的，所以不需要從緩存中讀取，除非重新載入特定頁面。

由於 HTTP Request 是 Ajax 中最極端的瓶頸之一，減少其需求數量對整個頁面性能有很大影響。尤其是當你將 100 個圖片請求轉化為一個 MXHR 請求時。

## 2. Sending Data 發送數據

有時不用關心接收數據，而只要將數據發送給伺服器(比如可以發送用戶的非私有信息以備日後分析，或者捕獲所有腳本錯誤然後將有關細節發送給伺服器進行記錄和提示)。當數據只需發送給伺服器時，有兩種廣泛應用的技術：XHR 和 Beacons。

### 2.1 XMLHttpRequest

數據可以使用 GET 或 POST 的方式傳回來，包括任意數量的 HTTP headers，當使用 XHR 發送數據給伺服器時候，使用 GET 會更快，只需要發送一個數據包，POST 至少兩個數據包，一個裝載 header，一個裝載 POST 正文。

    var url = '/data.php';
    var params = [
      'id=934875',
      'limit=20'
    ];
    var req = new XMLHttpRequest();
    req.onerror = function() {
      // Error.
    };
    req.onreadystatechange = function() {
      if (req.readyState == 4) {
        // Success.
      }
    };
    req.open('POST', url, true);
    req.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    req.setRequestHeader('Content-Length', params.length);
    req.send(params.join('&'));

POST 更適合發送大量數據到伺服器，因為它不關心額外數據包的數量，又因為 IE 的 URL 長度限制(2048 個字符)，它不能使用過長的 GET 請求。

### 2.2 Beacons

<!-- [https://juejin.im/post/5b694b5de51d4519700fa56a](https://juejin.im/post/5b694b5de51d4519700fa56a) -->

是一種從網頁把信息傳遞給伺服器而無需等待響應的輕量並且高效的方法，這項技術非常類似動態腳本標籤插入。使用 JS 創建一個新的 Image 對象，並把 src 屬性設置為伺服器上傳腳本的 URL。該 URL 包含我們通過 GET 傳回的鍵值對數據。請注意並沒有創建 img 元素或把它插入 DOM。

    var url='/status_tracker.php';
    var params=[
    	'step=2',
    	'time=123241223'
    ];

    (new Image()).src=url+'?'+params.join('&');

伺服器會接收數據並保存下來，它無需向客戶端發送任何回饋信息，因為沒有圖片會實際顯示出來，這是往伺服器回傳信息最有效的方式。它的性能消耗很小，而且伺服器的錯誤完全不會影響客戶端。但如果你需要返回大量數據給客戶端，那麼請使用 XHR。

[mdn-beacon↗](https://developer.mozilla.org/en-US/docs/Web/API/Beacon_API)

[caniuse-beacon↗](https://caniuse.com/#feat=beacon)

## 3. Data Formats 數據格式

### 3.1 XML

早期 XML 極端的互通性(伺服器端和客戶端都能夠良好支持)，格式嚴格，易於驗證。(那時)JSON 還沒有正式作為交換格式，幾乎所有的伺服器端語言都有操作 XML 的庫。

    <?xml version="1.0" encoding='UTF-8'?>
    <users total="4">
      <user id="1">
        <username>alice</username>
        <realname>Alice Smith</realname>
        <email>alice@alicesmith.com</email>
      </user>
      <user id="2">
        <username>bob</username>
        <realname>Bob Jones</realname>
        <email>bob@bobjones.com</email>
      </user>
    </users>

與其他格式相比，XML

- 極其冗長，每個離散的數據片斷需要大量結構
- 語法解析模糊耗時，必須先知道 XML response 的 layout，然後才能搞清楚它的含義。(解析 XML 必須確切地知道如何解開這個結構然後再將它們寫入 JS 對像中)

### 3.2 JSON

由 Douglas Crockford 的發明與推廣，JSON 是一個輕量級並易於解析的數據格式，它按照 JavaScript 對象和陣列字面語法所編寫。下例是用 JSON 書寫的用戶列表：

    [
      { "id":1, "username":"alice", "realname": "Alice Smith" },
      { "id":2, "username":"bob", "realname": "Bob Jones" },
      { "id":3, "username":"carol", "realname": "Carol Williams" }
    ]

### 3.3 JSON-P

[wiki-jsonp↗](https://zh.wikipedia.org/wiki/JSONP)

事實上 JSON 可被本地執行有幾個重要的性能影響。當使用 XHR 時 JSON 數據作為一個字符串返回。該字符串使用的 eval() 轉換為一個本地對象。然而，當使用動態腳本標籤插入時，JSON 數據被視為另一個的 JavaScript文件並作為本地碼執行。為做到這一點，數據必須被包裝在回調函數之中。這就是所謂的 "JSON Padding" 或JSON-P。下面是用 JSON-P 格式書寫的用戶列表：

    parseJSON([
      { "id":1, "username":"alice", "realname":"Alice Smith" },
      { "id":2, "username":"bob", "realname":"Bob Jones" },
      { "id":3, "username":"carol", "realname":"Carol Williams" }
    ]);

JSON-P 因為回調包裝的原因略微增加了文件尺寸，但與其解析性能的改進相比這點增加微不足道。由於數據作為本地 JavaScript 的處理，它的解析速度像本地的 JavaScript 一樣快。

最快的 JSON 格式是使用陣列的 JSON-P 格式。雖然這只比使用 XHR 的 JSON 略快，但是這種差異隨著列表尺寸的增大而增大。如果你所從事的項目需要一個 10,000 或 100,000 個單元構成的列表，那麼 JSON-P 比 JSON
 好很多。

與 XML 相比 JSON 的優點：

1. 格式小得多，結構佔用的空間更小，有效數據佔用的更多。特別是數據包含陣列而不是對象時以 `.json` 與大多數伺服器端語言的編解碼庫之間有著很好的互操作性。它在客戶端的解析工作微不足道，使你可以將更多寫程式碼的時間放在其他數據處理上。
2. 在線傳輸相對較小，也因為解析十分之快以 `.json` 是高性能的 Ajax 的基石，特別是使用動態腳本標籤插入時。

### 3.4 HTML

通常你所請求的數據以 HTML 返回並顯示在頁面上，JS 能夠比較快地將一個大數據結構轉化為簡單的 HTML，但是伺服器完成同樣工作的速度更快。有一種技術考慮是在伺服器端構建整個 HTML 然後傳遞給客戶端，JS 只是簡單地下載它然後放入 innerHTML 之中。

    <ul class="users">
      <li class="user" id="1-id002">
        <a href="http://www.site.com/alice/" class="username">alice</a>
        <span class="realname">Alice Smith</span>
      </li>
      <li class="user" id="2-id002">
        <a href="http://www.site.com/bob/" class="username">bob</a>
        <span class="realname">Bob Jones</span>
      </li>
      <li class="user" id="3-id002">
        <a href="http://www.site.com/carol/" class="username">carol</a>
        <span class="realname">Carol Williams</span>
      </li>
    </ul>

此技術的缺點：

1. HTML 是一種詳細的數據格式，比 XML 更加冗長。在數據本身的最外層，可有嵌套的 HTML 標籤，每個都具有 ID，Class，和其他屬性，HTML 格式可能比實際數據佔用更多的空間。
2. HTML 傳輸數據量明顯偏大時，也就需要較長時間來"解析"(將 HTML DOM 插入的操作)。因為將 HTML 插入到 DOM 的單一操作看似簡單(儘管它只有一行代碼)，卻仍需要時間向頁面加載很多數據。
3. HTML 不能像本地的 JS 陣列那樣輕易迅速地進行迭代操作。

作為數據格式，它緩慢而且臃腫。

### 3.5 Custom Formatting 自定義格式

最理想的數據格式只包含必要的結構，使你能夠分解出每個字段。你可以自定義一種格式只是簡單地用一個分隔符將數據連結起來：

    Jacob;Michael;Joshua;Matthew;Andrew;Christopher;Joseph;Daniel;Nicholas

這些分隔符基本上創建了一個數據陣列，類似於一個逗號分隔的列表。通過使用不同的分隔符，你可以創建多維陣列。比如：

    1:alice:Alice Smith:alice@alicesmith.com;
    2:bob:Bob Jones:bob@bobjones.com;
    3:carol:Carol Williams:carol@carolwilliams.com

這種格式非常簡潔，與其他格式相比(不包括純文本)，其數據 / 結構比例明顯提高。自定義格式下載迅速，易於解析，只需調用字符串的 `split()` 將分隔符作為參數傳入即可。更複雜的自定義格式具有多種分隔符，需要在循環中分解所有數據。`split()` 是最快的字符串操作之一，通常可以在數毫秒內處理具有超過 10,000 個元素的"分隔符分割"列表。

    function parseCustomFormat(responseText) {
      var users = [];
      var usersEncoded = responseText.split(';');
      var userArray;
      for (var i = 0, len = usersEncoded.length; i < len; i++) {
        userArray = usersEncoded[i].split(':');
        users[i] = {
          id: userArray[0],
          username: userArray[1],
          realname: userArray[2],
          email: userArray[3]
        };
      }
      return users;
    }

[example](https://codepen.io/amelieljit/pen/Pdewvv)

通常數據格式越輕量越好，JSON 和字符串分割的自定義格式是最好的。如果數據集很大並且對解析時間有要求，那麼請從如下兩種格式中做出選擇。

1. JSON-P 格式：使用動態腳本插入獲取它把數據當作可以執行的 JS 而不是字符串，解析速度極快。它能夠跨域使用，但涉及敏感數據的時候不應該使用它。
2. 字符分割的自定義格式：使用 XHR 或動態腳本插入來獲取，用 `split()` 解析。這種技術解析大數據集比JSON-P 略快，而且通常文件尺寸更小。

## 4. Ajax performance guidelines Ajax 性能指南

### 4.1 Cache Data 緩存數據

1. 兩種主要方法避免發出一個不必要的請求：
    - 在伺服器端，設置 HTTP header，確保返回的 response 將被緩存在瀏覽器中。(最容易設置和維護)
    - 在客戶端，於本地緩存已獲取的數據，不要多次請求同一個數據。(能有最大程度的控制)
2. Setting HTTP headers 設定 HTTP headers

<!-- [https://blog.techbridge.cc/2017/06/17/cache-introduction/](https://blog.techbridge.cc/2017/06/17/cache-introduction/) -->

在 HTTP Response Header 裡面加上一個 Expires 的字段，裡面就是這個 Cache 到期的時間

- Expires header
  - 設置 Expires header 告訴瀏覽器應當緩存 response 多長時間。
  - 其值是一個日期，當過期之後任何對該 URL 發起的請求都不再從緩存中獲得，而要重新訪問伺服器。

```
Expires: Mon, 28 Jul 2050 23:30:00 GMT

// 用於那些永不改變的內容，例如圖片和靜態數據集。
```

[mdn-expires↗](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Expires)

3. Storing data locally 本地儲存數據

  直接存儲那些從伺服器收到的 response。可將 response 存放在一個對象中，以 URL 為鍵值索引它。例如，下面這是一個 XHR 封裝，它首先檢查一個 URL 此前是否被取用過：

    var localCache = {};

    function xhrRequest(url, callback) {
      // Check the local cache for this URL.
      if (localCache[url]) {
        callback.success(localCache[url]);
        return;
      }

      // If this URL wasn't found in the cache, make the request.
      var req = createXhrObject();
      req.onerror = function() {
        callback.error();
      };

    	req.onreadystatechange = function() {
        if (req.readyState == 4) {
          if (req.responseText === '' || req.status == '404') {
            callback.error();
            return;
          }

    			// Store the response on the local cache.
          localCache[url] = req.responseText;
          callback.success(req.responseText);
        }
      };

      req.open("GET", url, true);
      req.send(null);
    }

  本地緩存也可很好地作用於移動設備上。此類設備上的瀏覽器緩存小或者根本不存在，手工緩存成為避免不必要請求的最佳選擇。

### 4.2 Summary 總結

- 減少請求數，可以通過合併 JavaScript 和 CSS 文件，或者使用 MXHR。
- 縮小頁面的加載時間，頁面主要內容加載完成後，用 Ajax 獲取那些次要的文件。
- 確保你的代碼錯誤不會輸出給用戶，並在服務端處理錯誤。
- 知道何時使用成熟的 Ajax library 庫，以及編寫自己的底層 Ajax 代碼。

<!--

refs:

[https://blog.csdn.net/situdesign/article/details/5635767](https://blog.csdn.net/situdesign/article/details/5635767)

[https://codertw.com/%E5%89%8D%E7%AB%AF%E9%96%8B%E7%99%BC/295498/](https://codertw.com/%E5%89%8D%E7%AB%AF%E9%96%8B%E7%99%BC/295498/)

[http://tw93.com/2015-02-14/High-Performance-JavaScript-6.html](http://tw93.com/2015-02-14/High-Performance-JavaScript-6.html)

[https://hk.saowen.com/a/a47dd4e36ff4d641404d54a0a39e52b8d00852bb305e06eccde6d79cd3b2382e](https://hk.saowen.com/a/a47dd4e36ff4d641404d54a0a39e52b8d00852bb305e06eccde6d79cd3b2382e)

XMLHttpRequest

[https://notfalse.net/29/xmlhttprequest](https://notfalse.net/29/xmlhttprequest)

[https://www.html5rocks.com/zh/tutorials/file/xhr2/](https://www.html5rocks.com/zh/tutorials/file/xhr2/)

-->