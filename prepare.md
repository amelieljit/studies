# CH 8 Programming Practices

> 編程過程中的性能優化

# 1. Avoid double evaluation

- 避免雙重求值帶來的效能影響

JS 有四種標準方法可以實現：允許你在程序中提取一個包含代碼的字符串，然後動態執行。

eval(), Function(), setTimeout(), setInterval()

但是像這樣在 JS 程式碼中執行另外一段 JS 程式碼就會造成雙重求值，不推薦使用，以避免效能消耗。

- eval()
```
var array = ['first','second','third'];

// Bad
console.time();
var item = eval("array[0]");
console.timeEnd();  // default: 0.041015625ms

// Good
console.time();
var item = array[0];
console.timeEnd();  // default: 0.002685546875ms
```

- setTimeout()
```
var num1 = 5, num2 = 6, sum;

// Bad
console.time();
setTimeout("sum=num1+num2", 0); // String 參數
console.timeEnd();              // default: 0.05615234375ms

// Good
console.time();
setTimeout(function() {
  var sum = num1 + num2
}, 0)                       // function 參數
console.timeEnd();          // default: 0.041015625ms
```

# 2. Use Object/Array Literals

- 使用物件/陣列直接量來創建比較快

使用直接量(Literals)在程式碼中佔用較少空間，所以整個文件尺寸可以更小。
```
// Bad
console.time();
var array = new Array('first','second','third'); // constructor 創建
console.timeEnd();  // default: 0.01708984375ms

// Good
console.time();
var array=['first','second','third']; // 用直接量創建
console.timeEnd();  // default: 0.01513671875m
```

# 3. Don't Repeat Work

- 不要重複工作

常見的重複工作類型例如：瀏覽器檢測，通過測試 `addEventListener()` 和 `removeEventListener()` 檢查 DOM Level 2 的事件支持情況，它能夠被 IE 之外的所有現代瀏覽器所支持。如果這些方法不存在於 target 中，那麼就認為當前瀏覽器是 IE 瀏覽器，並使用 IE 特有的方法。
```
function addHandler(target, eventType, handler) {
  if (target.addEventListener) {
    // DOM Level 2 Events
    target.addEventListener(eventType, handler, false);
  } else {
    // IE
    target.attachEvent("on" + eventType, handler);
  }
}

function removeHandler(target, eventType, handler) {
  if (target.removeEventListener) {
    // DOM2 Level 2 Events
    target.removeEventListener(eventType, handler, false);
  } else {
    // IE
    target.detachEvent("on" + eventType, handler);
  }
}
```

這裡隱藏的性能問題在於每次函數調用時都執行重複工作。每一次都進行同樣的檢查，看看某種方法是否存在。如果假設目標唯一的值就是 DOM 對象，而且用戶不可能在頁面加載時魔術般地改變瀏覽器，那麼這種判斷就是重複的。如果 addHandler 一上來就調用的 `addEventListener()` 那麼每個後續調用都要出現這句代碼。在每次調用中重複同樣的工作是一種浪費。

- Lazy loading 延遲載入

在信息被使用之前不做任何工作。在前面的例子中，不需要判斷使用哪種方法附加或分離事件句柄，直到有人調用此函數。
```
function addHandler(target, eventType, handler) {
  // overwrite the existing function
  if (target.addEventListener) {
    // DOM2 Events
    addHandler = function(target, eventType, handler) {
      target.addEventListener(eventType, handler, false);
    };
  } else {
    // IE
    addHandler = function(target, eventType, handler) {
      target.attachEvent("on" + eventType, handler);
    };
  }
  // call the new function
  addHandler(target, eventType, handler);
}

function removeHandler(target, eventType, handler) {
  // overwrite the existing function
  if (target.removeEventListener) {
    //DOM2 Events
    removeHandler = function(target, eventType, handler) {
      target.addEventListener(eventType, handler, false);
    };
  } else {
    // IE
    removeHandler = function(target, eventType, handler) {
      target.detachEvent("on" + eventType, handler);
    };
  }
  // call the new function
  removeHandler(target, eventType, handler);
}
```

只有第一次被調用時，檢查一次並決定使用哪種方法附加(attach)或分離(detach)事件句柄。然後，原始函數就被包含適當操作的新函數覆蓋掉了。最後調用新函數並將原始參數傳給它。以後再調用 `addHandler()` 或者 `removeHandler()` 時不會再次檢測，因為檢測代碼已經被新函數覆蓋了。

調用一個延遲載入的函數總是在第一次使用較長時間，因為它必須運行檢測然後調用另一個函數以完成任務。但是，後續調用同一函數將快很多，因為不再執行檢測邏輯了。延遲載入適用於函數不會在頁面上立即被用到的場合。

- Conditional Advance Loading
- 條件提前載入

在腳本載入之前提前進行檢查，而不等待函數調用。這樣做檢測仍只是一次，但在此過程中來得更早。

    var addHandler = document.body.addEventListener ?
      function(target, eventType, handler) {
        target.addEventListener(eventType, handler, false);
      } :
      function(target, eventType, handler) {
        target.attachEvent("on" + eventType, handler);
      };

    var removeHandler = document.body.removeEventListener ?
      function(target, eventType, handler) {
        target.removeEventListener(eventType, handler, false);
      } :
      function(target, eventType, handler) {
        target.detachEvent("on" + eventType, handler);
      };

這個例子檢查的 `addEventListener()` 和 `removeEventListener()` 是否存在，然後根據此信息指定最合適的函數。三元操作符返回 DOM Level 2 的函數，如果它們存在的話，否則返回 IE 特有的函數。(調用 `addHandler()` 和
 `removeHandler()` 同樣很快，只是檢測功能提前了。)

條件提前載入確保所有函數調用時間相同，其代價是在腳本載入時進行檢測。它適用於一個函數馬上就會被用到，而且在整個頁面生命週期中經常使用的場合。

# 4. Use the Fast Parts

- 使用 JS 速度快的方法

## Bitwise Operators

bitwise 運算符是 JS 中經常容易在布林表達式中被誤用的東西之一。JS 中的數字按照 [IEEE-754 標準 64 位格式]([https://goo.gl/hLpPaC](https://goo.gl/hLpPaC)) 儲存。在 bitwise 運算中，數字被轉換為有符號 32 位的格式。每種操作均直接操作在這個 32 位數上實現結果。儘管需要轉換，這個過程與 JS 中其他數學和布林運算相比還是非常快。 (refs: [IEEE-754]([https://segmentfault.com/a/1190000009084877](https://segmentfault.com/a/1190000009084877)))

JS 可以很容易地用 `toString()` 方法將數字轉換為字符形式的二進位表達式：

    var num1 = 25, num2 = 3;
    console.log(num1.toString(2)); // 11001
    console.log(num2.toString(2)); // 11

JS 中四種 bitwise 邏輯操作符

[MDN: Bitwise operators↗](https://goo.gl/BWrA16)

    // bitwise AND
    var result1 = 25 & 3;            // 1
    console.log(result.toString(2)); // "1"
    // bitwise OR
    var result2 = 25 | 3;            // 27
    console.log(resul2.toString(2)); // "11011"
    // bitwise XOR
    var result3 = 25 ^ 3;            // 26
    console.log(resul3.toString(2)); // "11000"
    // bitwise NOT
    var result = ~25;                // -26
    console.log(resul2.toString(2)); // "-11010"

利用 bitwise operators 提升 JS 的速度

- 使用 bitwise operators 代替純數學操作，以提升速度
```
// 通常會採用對 2 取餘數(modulus)的方式實現表格颜色交换
for (var i = 0, len = rows.length; i < len; i++) {
  if (i % 2) {
    className = "even";
  } else {
    className = "odd";
  }
}

// 32 位數字的底層(二進制)表示法，其偶數的最低位是 0，奇數的最低位是 1,
// 如果此數為偶數，那麼它和 1 進行 bitwise AND 操作的結果就是 0,
// 如果此數為奇數，那麼它和 1 進行 bitwise AND 操作的結果就是 1
for (var i = 0, len = rows.length; i < len; i++) {
  if (i & 1) {
    className = "even";
  } else {
    className = "odd";
  }
}

console.time();
console.log(25 % 3);  // 1
console.timeEnd();    // default: 0.287109375ms

console.time();
console.log(25 & 3);  // 1
console.timeEnd();    // default: 0.190185546875ms
```

<!-- [https://www.avioconsulting.com/blog/overcoming-javascript-numeric-precision-issues](https://www.avioconsulting.com/blog/overcoming-javascript-numeric-precision-issues) -->

盡量使用 JS 提供的原生方法


# CH 9 Building and Deploying High-Performance JavaScript Applications

> 創建並部署高性能 JavaScript 應用程序

## 1. 自動化工具

例如：Gulp, Grunt, Apache Ant...等。

對於需要反複執行的任務，比如壓縮、編譯、單元測試等，可以使用自動化構建工具完成，簡化工作；還可以合併多個文件，減少 HTTP 請求數。

## 2. JS Minification

- JS 壓縮

例如：JSMin, YUI Compressor

JS Minificartion 是剔除 JS 文件中一切與執行無關内容的過程。包括注釋和不必要的空格等等。該處理通常可將文件尺寸縮減到一半，使其下載速度更快。

- HTTP Encoding

當瀏覽器發送請求時，會發送一個 `Accept-Encoding` 的 HTTP Header，以告訴伺服器支持的編碼類型，值可能為 gzip, compress, deflate 等。此信息主要用於允許文檔壓縮以獲得更快下載速度，從而改善用戶體驗。伺服器根據這些值對資源進行壓縮，然後使用 HTTP Header 中的 Content-Encoding 告訴瀏覽器壓縮類型。 Gzip 是當前常用的編碼方式，但是 Gzip 主要適用於文本，比如 JavaScript 文件、CSS 等。而對於其他文件比如圖片，PDF 等不應該使用 Gzip 壓縮。因為它們本身已經被壓縮過，再壓縮只會浪費 CPU 資源。

## 3. Buildtime Versus Runtime Build Processes

- 構建時間與運行時構建過程

連接(concatenation)，預處理(preprocessing)，和 minification 既可以在編譯時發生，也可以在運行時發生。在開發過程中，執行時創建過程非常有用，但由於擴展性原因一般不建議在 production 環境中使用。開發高性能應用程序的一個普遍規則是，**只要能夠在編譯時完成的工作，就不要在執行時去做**。

## 4. Working Around Caching Issues

- 關於緩存資源

緩存 HTTP components 可以極大提高用戶體驗，緩存適用於所有靜態內容，比如圖片、JavaScript 文件等。 Web 伺服器通過 Expires HTTP Header 告訴瀏覽器資源應該被緩存多長時間，它的值是一個時間戳(timestamp)。 使用緩存的一個缺點：當應用升級時，需要確保用戶下載到最新的靜態內容。這個問題可以通過對靜態資源進行重命名解決。比如在資源 URL 後追加一個版本號或開發編號，也可以是一個時間戳。

補充：[HTML5 manifest↗](https://goo.gl/JrZVgt)

<!-- [https://goo.gl/UXVDP2](https://goo.gl/UXVDP2) -->

## 5. Using CDN

- 使用內容傳遞網

CDN 是在互聯網上按照地理位置分佈計算機網絡，負責傳遞內容給終端用戶，使用 CDN 的主要原因是增強 Web 應用的可靠性和可擴展性，也能提高性能。通過向地理位置最近的用戶傳輸內容，CDN 可以減少網絡延遲。

## 總結

在設計和開發階段做的一些優化可以更有效的提高Web應用，但是在構建和部署階段也很重要且容易被忽略。開發和部署過程對基於 JS 的應用程序可以產生巨大影響，最重要的幾個步驟如下：

1. 合併 JS 文件，減少 HTTP 請求的數量
2. 使用壓縮器(manification)處理 JS 文件
3. 在伺服器端以壓縮形式提供 JS 文件(Gzip 編碼)
4. 通過設置 HTTP Respons Header 使 JS 文件可緩存，通過向文件名附加時間戳解決緩存問題
5. 使用 CDN(內容傳遞網)提供 JS 文件，CDN 不僅可以提高性能，它還可以為你管理壓縮和緩存

所有這些步驟應當自動完成，不論是使用公開的開發工具諸如 Apache Ant，還是使用自定義的開發工具以實現特定需求。使這些開發工具可以極大改善那些大量使用 JS 程式碼的網頁應用或網站的性能。

<!-- [https://github.com/xswei/JavaScript_Faster/tree/master/Building_Deploying_JavaScript_Application](https://github.com/xswei/JavaScript_Faster/tree/master/Building_Deploying_JavaScript_Application) -->


AJAX

缺點:
1. 可能破壞瀏覽器的機制(後退鍵與加入收藏書籤功能): 在動態更新頁面的情況下，用戶無法回到前一個頁面狀態，這是因為瀏覽器僅能記下歷史記錄中的靜態頁面。(比如希望單擊後退按鈕，就能夠取消前一次操作，但是在 Ajax 應用程式中，卻無法這樣做。)
  <!--
  例如，當用戶在Google Maps中單擊後退時，它在一個隱藏的IFRAME中進行搜索，然後將搜索結果反映到Ajax元素上，以便將應用程式狀態恢復到當時的狀態。
  使用動態頁面更新使得用戶難於將某個特定的狀態保存到收藏夾中。該問題的解決方案也已出現，大部分都使用URL片斷標識符（通常被稱為錨點，即URL中#後面的部分）來保持跟蹤，允許用戶回到指定的某個應用程式狀態。（許多瀏覽器允許JavaScript動態更新錨點，這使得Ajax應用程式能夠在更新顯示內容的同時更新錨點。）這些解決方案也同時解決了許多關於不支持後退按鈕的爭論。
  -->
2. 瀏覽器不兼容: AJAX 高度依賴 JavaScript，而不同的瀏覽器對 JavaScript支持性不同。
3. 對搜尋引擎支持較弱: 如果使用不當會增大網絡數據的流量，從而降低整個系統的性能。
4. 客戶端過肥: 太多客戶端的程式碼易造成開發上的成本。
   <!--
   編寫複雜、容易出錯；冗餘代碼比較多（層層包含js文件是AJAX的通病，再加上以往的很多服務端代碼現在放到了客戶端；破壞了Web的原有標準。
   [ref](https://zh.wikipedia.org/wiki/AJAX)
   -->
<!--
abort()
[link](https://github.com/ljit-io/customer-service/pull/584)
[link_a](https://medium.com/@dashtinejad/ajax-requests-with-jquery-inside-react-4fa60255f10d)
-->



<!--

refs:

★ [https://blog.csdn.net/michael8512/article/details/79531790](https://blog.csdn.net/michael8512/article/details/79531790)

[https://blog.csdn.net/HeliumLau/article/details/73527918](https://blog.csdn.net/HeliumLau/article/details/73527918)

[https://juejin.im/post/58fdcdc31b69e60058a29444](https://juejin.im/post/58fdcdc31b69e60058a29444)

[http://obkoro1.com/2018/01/09/【读书笔记】《高性能JavaScript》/](http://obkoro1.com/2018/01/09/%E3%80%90%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0%E3%80%91%E3%80%8A%E9%AB%98%E6%80%A7%E8%83%BDJavaScript%E3%80%8B/)

[https://blog.csdn.net/situdesign/article/details/5635798](https://blog.csdn.net/situdesign/article/details/5635798)

[http://tw93.com/2015-02-16/High-Performance-JavaScript-7.html](http://tw93.com/2015-02-16/High-Performance-JavaScript-7.html)

[http://blog.sina.com.cn/s/blog_53a5865c0100ishc.html](http://blog.sina.com.cn/s/blog_53a5865c0100ishc.html)



refs:

[https://blog.csdn.net/situdesign/article/details/5635767](https://blog.csdn.net/situdesign/article/details/5635767)

[https://codertw.com/%E5%89%8D%E7%AB%AF%E9%96%8B%E7%99%BC/295498/](https://codertw.com/%E5%89%8D%E7%AB%AF%E9%96%8B%E7%99%BC/295498/)

[http://tw93.com/2015-02-16/High-Performance-JavaScript-7.html](http://tw93.com/2015-02-16/High-Performance-JavaScript-7.html)

[https://hk.saowen.com/a/a47dd4e36ff4d641404d54a0a39e52b8d00852bb305e06eccde6d79cd3b2382e](https://hk.saowen.com/a/a47dd4e36ff4d641404d54a0a39e52b8d00852bb305e06eccde6d79cd3b2382e)

-->