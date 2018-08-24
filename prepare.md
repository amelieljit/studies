# CH 6 Responsive Interfaces

> 快速嚮應的用戶介面

由於瀏覽器是單線程運作的，在處理 UI 事件的時候無法處理 JS 事件，反之亦然，當
 JS 運行時，用戶界面就被"鎖定"了。所以管理好 JS 運行時間對網頁應用的性能很重要。

## 1. The Browser UI Thread 瀏覽器 UI 線程

JS 和 UI 更新共享的進程(process)通常被稱作瀏覽器 UI 線程(雖然對所有瀏覽器來說"線程"一詞不一定準確)。此 UI 線程圍繞著一個簡單的隊列系統工作，任務被保存到隊列中直至進程空閒。一旦空閒，隊列中的下一個任務將被檢索和運行。這些任務不是運行 JS 碼，就是執行 UI 更新，包括重繪和重排版。此進程中最令人感興趣的部分是每次輸入均導致一個或多個任務被加入隊列。

舉個🌰 : 按下一個按鈕，然後屏幕上顯示出一個消息：

    <html>
      <head>
        <title>Browser UI Thread Example</title>
      </head>
      <body>
        <button onclick="handleClick()">Click Me</button>
        <script type="text/javascript">
          function handleClick() {
            var div = document.createElement("div");
            div.innerHTML = "Clicked!";
            document.body.appendChild(div);
          }
        </script>
      </body>
    </html>

![](/assets/images/figure_6-1.gif)

(圖: UI thread tasks get added as the user interacts with a page)

當按鈕被點擊時，它觸發 UI 線程創建兩個任務並添加到隊列中。第一個任務是按鈕的 UI 更新，它需要改變外觀以指示出它被按下了，第二個任務是 JS 執行任務，包含 `handleClick()` 的程式碼，被執行的唯一程式碼就是這個方法和所有被它調用的方法。假設 UI 線程空閒，第一個任務被檢查並運行以更新按鈕外觀，然後 JS 任務被檢查和運行。在執行過程中，`handleClick()` 創建了一個新的 `<div>` 元素，並追加在 `<body>` 元素上，其效果是引發另一次 UI 改變。也就是說在
 JS 執行過程中，一個新的 UI 更新任務被添加在隊列中，當 JS 執行完之後，UI 還會再更新一次(如上圖)。

當所有 UI 線程任務執行之後，進程進入空閒狀態，並等待更多任務被添加到隊列中。空閒狀態是理想的，因為所有用戶操作立刻引發一次 UI 更新。如果用戶企圖在任務運行時與頁面交互，不僅沒有即時的 UI 更新，而且不會有新的 UI 更新任務被創建和加入隊列。事實上，大多數瀏覽器在JavaScript 運行時停止 UI 線程隊列中的任務，也就是說 JavaScript 任務必須盡快結束，以免對用戶體驗造成不良影響。

### 1.1 Browser Limits 瀏覽器限制

瀏覽器在 JS 運行時間上採取了一些必要限制，確保惡意代碼編寫者不能通過無盡的密集操作鎖定用戶瀏覽器或計算機。此類限制有兩個：調用棧尺寸限制(the call stack size limit，第四章討論過)和長運行腳本限制(long-running script limit)。長運行腳本限制有時被稱作"長運行腳本定時器"或者"失控腳本定時器"；瀏覽器記錄一個腳本的運行時間，一旦到達一定限度時就終止它。當此限製到達時，瀏覽器會向用戶顯示一個對話框。

如果你的腳本在瀏覽器上觸發了此對話框，意味著腳本只是用太長的時間來完成任務。它還表明用戶瀏覽器在 JavaScript 代碼繼續運行狀態下無法響應輸入。從開發者觀點看，沒有辦法改變長運行腳本對話框的外觀，你不能檢測到它，因此不能用它來提示可能出現的問題。顯然，長運行腳本最好的處理辦法首先是避免它們。

### 1.2 How Long Is Too Long ? 多久才算太久？

> 100 毫秒。

一個單一的 JS 操作應當使用的總時間(最大)是 100 毫秒。這個數字根據 Robert Miller 在 1968 年的研究。有趣的是，可用性專家 Jakob Nielsen 在他的"可用性工程"(Morgan Kaufmann，1944)上註釋說這一數字並沒有因時間的推移而改變。

Nielsen 指出如果該界面在 100 毫秒內響應用戶輸入，用戶會認為自己是"直接操作用戶界面中的對象"。而超過 100 毫秒意味著用戶認為自己與界面斷開了。由於 UI 在 JS 執行時無法更新，如果運行時間長於 100 毫秒，用戶就不能感受到對介面的控制。

每種瀏覽器的行為大致相同。當腳本執行時，UI 不隨用戶交互而更新。此時 JS 任務作為用戶交互的結果被創建被放入隊列，然後當原始 JS 任務完成時隊列中的任務被執行。用戶交互導致的 UI 更新被自動跳過，因為優先考慮的是頁面上的動態部分。因此，當一個腳本運行時點擊一個按鈕，將看不到它被按下的樣子，即使它的 `onclick` 被執行了。因此最好的方法是，通過限制任何 JS 任務在 100 毫秒或更少時間內完成，避免此類情況出現。(這種測量應當在你要支持的最慢的瀏覽器上執行。)

## 2. Yielding with Timers  用定時器讓出時間片段

有一些 JS 任務因為複雜性原因不能在 100 毫秒或更少時間內完成。這種情況下，理想方法是讓出對 UI 線程的控制，使 UI 更新可以進行。讓出控制意味著停止 JS 執行，給 UI 線程機會進行更新，然後再繼續執行 JS。

### 2.1 Timer Basics 定時器

在 JS 的中使用的 `setTimeout()` 或 `setInterval()` 創建定時器，兩個函數都接收一樣的參數：一個要執行的函數，和一個運行它之前的等待時間(單位毫秒) `.setTimeout()` 函數創建一個定時器只運行一次，而的 `setInterval()` 函數創建一個週期性重複運行的定時器。

使用定時器與 UI 線程交互的方式有助於分解長運行腳本成為較短的片段。調用的 `setTimeout()` 或 `setInterval()` 告訴 JS 引擎等待一定時間然後將 JS 的任務添加到 UI 隊列中。例如：

    function greeting() {
      alert("Hello world!");
    }
    setTimeout(greeting, 250);

程式碼將在 250 毫秒之後，向 UI 隊列插入一個 JS 任務執行 `greeting()` 函數。在那個點之前，所有其他 UI 更新和
 JS 任務都在執行。第二個參數指出什麼時候應當將任務添加到 UI 隊列之中，並不是說那時程式碼將被執行。這個任務必須等到隊列中的其他任務都執行之後才能被執行。再看一個例子：

    var button = document.getElementById("my-button");
    button.onclick = function() {
      oneMethod();
      setTimeout(function() {
        document.getElementById("notice").style.color = "red";
      }, 250);
    };

當按鈕被點擊時，它調用一個方法然後設置一個定時器。用於修改 `notice` 元素顏色的程式碼被包含在一個定時器中，將在 250 毫秒之後添加到隊列。 250 毫秒**從調用 `setTimeout()` 時開始計算，而不是從整個函數運行結束時開始計算**。如果 `setTimeout()` 在時間點 `n` 上被調用，那麼執行定時器程式碼的 JS 任務將在 `n+250` 的時刻加入 UI 隊列。

![](/assets/images/figure_6-2.gif)

(圖: `setTimeout()` 的第二個參數指出何時將新的 JS 任務插入到 UI 隊列中)

定時器程式碼只有等創建它的函數執行完成之後，才有可能被執行。再看一個🌰：如果前面的程式碼中定時器延時變得更小，然後在創建定時器之後又調用了另一個函數，定時器程式碼有可能在 `onclick` 事件處理完成之前加入隊列：

    var button = document.getElementById("my-button");
    button.onclick = function(){
      oneMethod();
      setTimeout(function(){
        document.getElementById("notice").style.color = "red";
      }, 50);
      anotherMethod();  // 執行時間超過 50 毫秒
    };

上面如果 `anotherMethod()` 執行時間超過 50 毫秒，那麼定時器程式碼將在 `onclick` 處理完成之前加入到隊列中。其結果是等 `onclick` 處理執行完畢，定時器程式碼立即執行，而察覺不出其間的延遲。

![](/assets/images/figure_6-3.gif)

(圖: 如果調用 `setTimeout()` 的函數又調用了其他任務，耗時超過定時器延時，定時器程式碼將立即被執行，它與主調函數之間沒有可察覺的延遲)

在任何一種情況下，創建一個定時器造成 UI 線程暫停，如同它從一個任務切換到下一個任務。因此，定時器程式碼重置(reset)所有相關的瀏覽器限制，包括長運行腳本時間。此外，調用棧也在定時器程式碼中重置為零。這一特性使得定時器成為長運行 JS 代碼理想的跨瀏覽器解決方案。

### 2.2 Array Processing with Timers 使用定時器來處理陣列

對每一個陣列項調用的函數，處理完成後執行的回調函數：

    // processArray 接收了三個參數：待處理的陣列，對每個陣列項目調用的處理函數，處理結束時執行的回調函數
    function processArray(items, process, callback) {
        var todo = items.concat();

        setTimeout(function() {
            process(todo.shift());

            if (todo.length > 0) {
                setTimeout(arguments.callee, 25);
            } else {
                callback(items);
            }
        }, 25);
    }

    var items = [1,2,3,4,5];

    function output(value) {
        console.log(value);
    }

    processArray(items, output, function() {
        console.log("Done!");
    })

    // 1
    // 2
    // 3
    // 4
    // 5
    // Done!

像上面這樣使用定時器處理陣列的副作用是處理陣列的總時長增加了，這是因為在每一個條目處理完成之後 UI 線程會空閒出來，並且在下一條目開始處理之前會有一段延時，儘管如此，為避免鎖定瀏覽器給用戶帶來的體驗，這種取捨是有必要的。

### 2.3 Splitting Up Tasks 分割任務

由待執行函數組成的陣列，為每一個函數運行時提供參數的陣列，以及處理結束時調用的回調函數

    function multistep(steps, args, callback) {
        var tasks = steps.concat();

        setTimeout(function() {
            // 執行下一个任務
            var task = tasks.shift();
            task.apply(null, args || []);

            // 檢查是否還有其他任務
            if (tasks.length > 0) {
                setTimeout(arguments.callee, 25);
            } else {
                callback();
            }
        }, 25);
    }

    function saveDocument(id) {
        var tasks = [openDocument, writeText, closeDocument, updateUI];

        multistep(tasks, [id], function() {
            alert("Done")
        });
    }

注意 `multistep()` 的第二個參數必須為陣列, 它創建時只包含一個 `id`。正如陣列處理那樣，使用此函數的前提條件：任務可以異步處理而不影響用戶體驗或造成相關代碼錯誤。

### 2.4 Timers and Performance 定時器與效能

定時器使你的 JS 程式碼整體性能表現出巨大差異，但過度使用它們會對性能產生負面影響。前面講的程式碼使用定時器序列，同一時間只有一個定時器存在，只有當這個定時器結束時才創建一個新的定時器。以這種方式使用定時器不會帶來性能問題。

而當多個重複的定時器被同時創建會產生性能問題。因為只有一個 UI 線程，所有定時器會競爭執行時間。所以要注意在你的網頁應用中限制高頻率重複定時器的數量。同時，建議創建一個單獨的重複定時器，每次執行多個操作。

## 3. Web Workers

自 JS 誕生以來，還沒有辦法在瀏覽器 UI 線程之外執行程式碼。 web worker 線程 API 改變了這種狀況，它引入一個接口，使程式碼執行而不佔用瀏覽器 UI 線程的時間。作為最初的 HTML 5 的一部分，web worker 線程 API 已經分離出去成為獨立的規範([http://www.w3.org/TR/workers/](https://www.w3.org/TR/workers/))。web worker 已經被 Firefox 3.5，Chrome 3，和 Safari 4 原生實現。

![](/assets/images/figure_6-4-worker.png)

JavaScript 通常在作業系統的 Main Thread 執行，但若把程式碼放在 Web Workers 就可另外開一條 Worker Thread，兩條線互不影響，讓 JavaScript 在背景執行，並且兩線可由訊息溝通(使用 postMessage 發送訊息、onmessage 接收訊息)。通常我們會將需要長時間運算且不含 Window 或 DOM Element 操作的程式碼放在 Web Workers，好處是不阻塞 Main Thread 而讓速度變快。(web worker 是運行在後台的 JavaScript，不會影響頁面的性能。例如處理信息量比較龐大的東西時有可能會造成頁面阻塞，因此這種時候就可以通過 Worker 創建一個線程在後台處理信息，當處理完成時會把信息返回回來。)

![](/assets/images/figure_6-5-support.png)

(圖: Internet Explorer 11, Firefox, Chrome, Safari 和 Opera 都支持 Web workers，(not `Shared Web Workers`))

### 3.1 Worker Environment

由於 web worker 不綁定 UI 線程，這也意味著它們將不能訪問許多瀏覽器資源。JS 和 UI 更新共享同一個進程的部分原因是它們之間互訪頻繁，如果這些任務失控將導致糟糕的用戶體驗。web worker 修改 DOM 將導致用戶界面出錯，但每個
 web worker 都有自己的全局執行環境，只有 JS 特性的一個子集可用。它的環境組成為：

- 一個 navigator 對象， 只包含四個属性: `appName` 、 `appVersion` 、 `userAgent` 和 `platform`
- 一個 location 對象(與 `window.location` 相同)， 不過所有屬性都是 `read-only`
- 一個 self 對象， 指向全局 worker 對象
- 一個 importScripts( ) 方法，用來加載 Worker 所用到的外部 javascript 文件
- 所有的 ECMAScript 對象，諸如: Object, Array, Date 等
- XMLHTTPRequest constructor
- `setTimeout()` 和 `setInterval()` 方法
- 一個 close( ) 方法，它能立刻停止 worker 運行

由於 Web Worker 有著不同的全局環境，因此無法從 Javascript 程式碼中創建它。需要創建一個完全獨立的 JavaScript 文件，其中包含了需要在 Worker 中運行的程式碼。要 Web Worke，必須傳入這個 Javascript 文件的 URL:

`var worker = new Worker("code.js");`

程式碼一旦執行，將為這個文件創建一個新的線程和一個新的 Worker 運行環境。該文件會被異步下載，直到文件下載並執行完成後才會啟動此 Worker。

### 3.2 Worker Communication

worker 與網頁程式碼通過事件接口進行溝通。網頁程式碼可以通過 `postMessage()` 方法給 worker 傳遞數據，它接收一個參數，即需要傳遞給 Worker 的數據。此外，Worker 還有一個用來接收信息的 `onmessage` 事件處理器。如下：

    var worker = new Worker("code.js");
    worker.onmessage = function(event) {
        alert(event.data);
    }
    worker.postMessage("Hi");

Worker 通過觸發 `message` 事件來接收數據。定義 `onmessage` 事件處理器之後，該事件對象具有一個 data 屬性用於存放傳入的數據。Worker 可通過它自己的 `postMessage()` 方法把信息傳給頁面：

    // code.js 内部程式碼
    // self 是全局 worker 對象
    self.onmessage = function(event) {
        self.postMessage("Hello!");
    }

### 3.3 Loading External Files

`importScripts()` 的調用過程是阻塞的，直到所有文件加載並執行完成之後，腳本才會繼續運行。由於 Worker 在 UI 線程之外執行，所以這種阻塞並不會影響 UI 響應。

    // code.js 内部程式碼
    importScripts("file1.js", "file2.js");

    self.onmessage = function( ) {
    	...
    }

### 3.4 Practical Uses

Web Worker 適用於那些處理純數據，或者與瀏覽器 UI 無關的長時間運行腳本。例如：解析一個很大的 JSON 字符串，正常執行需大概 500 毫秒，因此這時可以使用 Web Worker：

    var worker = new Worker("code.js");

    // 數據就位時，調用事件處理器
    worker.onmessage = function(event) {
        // JSON 结構被回傳回来
        var jsonData = event.data;

        // 使用 JSON
        execute(jsonData);
    }

    // 傳入要解析的大段 JSON 字符串
    worker.postMessage(jsonText);


    // code.js 文件内部程式碼，worker 解析
    self.onmessage = function(event) {
        // JSON 字符串由 event.data 傳入

        // 解析
        var jsonData = JSON.parse(jsonText);

        // 回傳结果
        self.postMessage(jsonData);
    }

要注意即使 JSON.parse() 可能需要 500 毫秒或更多時間，也沒有必要添加更多程式碼來分解處理過程。此處理過程發生在一個獨立的線程中，所以你可以讓它一直運行完解析過程而不會干擾用戶體驗。

<!--
refs:
[https://medium.com/@francesco_rizzi/javascript-main-thread-dissected-43c85fce7e23](https://medium.com/@francesco_rizzi/javascript-main-thread-dissected-43c85fce7e23)

[https://codertw.com/前端開發/46140/](https://codertw.com/%E5%89%8D%E7%AB%AF%E9%96%8B%E7%99%BC/46140/)

[https://juejin.im/post/5a05b4576fb9a04519690d42](https://juejin.im/post/5a05b4576fb9a04519690d42)

web worker

[https://cythilya.github.io/2018/07/18/web-workers/](https://cythilya.github.io/2018/07/18/web-workers/)

[https://ithelp.ithome.com.tw/articles/10118851](https://ithelp.ithome.com.tw/articles/10118851)

[https://developer.mozilla.org/zh-TW/docs/Web/API/Web_Workers_API/Using_web_workers](https://developer.mozilla.org/zh-TW/docs/Web/API/Web_Workers_API/Using_web_workers)

[https://www.pixelstech.net/article/1354451079-HTML5-Web-Worker](https://www.pixelstech.net/article/1354451079-HTML5-Web-Worker)

[http://jsfiddle.net/greenrhino/zjvGD/](http://jsfiddle.net/greenrhino/zjvGD/)

-->