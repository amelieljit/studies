# CH7 Concurrency
## Outlines
- 7-61 不要阻塞 I/O 事件佇列
- 7-62 在異步序列中使用 Nested 或命名的 Callback function
- 7-63 注意錯誤的處理
- 7-64 對異步的循環使用遞迴
- 7-65 不要在計算時阻塞事件隊列
- 7-66 使用計數器來執行並行操作
- 7-67 絕不要同步地調用異步的 callback 函數
- 7-68 使用 promise 模式來清理你的異步邏輯

---


### 7-61 Don’t Block the Event Queue on I/O

> 不要阻塞 I/O 事件佇列

js 是構建在事件之上的， input 可能來自各種不同的外部來源。
```
var text = downloadSync('http://example.com/file.txt');
console.log(text);
```

像上面這樣的函數(downloadSync)稱為同步函数(或 block 函数)。以這個為例，程式會等待這個 input，也就是等它從網路上下載 file.txt 的結果完成，所以在等待期間，它不會做其他事情。

由於在等待下載完成期間，電腦可以做其他有用的工作，因此這樣的語言通常為程序員提供一種方法來創建多個線程，即並行執行子計算。它允許程序的一部分停下來等待（阻塞）一個低速的輸入，而程序的另一部分可以繼續進行獨立的工作。
在 js 中，大多的 I/O 操作都提供了異步的或非阻塞的 API。工程師提供一個 callback 函數，一旦輸入完成就可以被系統調用，而不是程序阻塞在等待結果的線程上。
```
downloadSync('http://example.com/file.txt', function(text) {
  console.log(text);
});
```

這個 API 初始化下載進行，然後在內部註冊表中存儲了 callback 函數後立即返回，而不是被網絡請求阻塞。當下載完成之後，系統會將下載完的文件的文本作為參數調用該已註冊的 callback 函數。
隨著事件的發生，事件被添加到應用程序的事件隊列的尾端。 js 系統使用一個內部循環機制來執行應用程序。該循環機制每次都拉取隊列底部的事件，以接收到這些事件的順序來調用這些已經註冊的 js 事件處理程序，並將事件的數據作為該事件處理程序的參數。

![Example event queues in a) a web client application and b) a web server](/assets/images/7-61_event_queues.png)

運行到完成機制擔保的好處是當代碼運行時，你完全掌握應用程序的狀態。根本不必擔心一些變數和對象屬性的改變由於並發執行代碼而超出你的控制。然而，機制的不足是，實際上所有你編寫的代碼支撐著餘下應用程序的繼續執行。像瀏覽器這樣的交互式應用程序中，一個阻塞的事件處理程序會阻塞任何將被處理的其他用戶輸入，甚至可能阻塞一個頁面的渲染，從而導致頁面失去響應的用戶體驗。在服務器環境中，一個阻塞的事件處理程序可能會阻塞將被處理的其他網絡請求，從而導致服務器失去響應。

js 並發的一個最重要的規則是絕不要在應用程序事件隊列中使用阻塞 I/O 的 API。在瀏覽器中，甚至幾乎沒有任何阻塞 API 是可用的，儘管有一些平台實現了。提供類似於 downloadAsync 功能的網絡 I/O 的XMLHttpRequest 庫有一個同步的版本實現，被認為是不好的。對於 web 應用程序的交互性，同步的 I/O 會導致災難性的後果，它在 I/O 操作完成之前一直會阻塞用戶與頁面的交互操作。

相比之下，異步的 API 用**在基於事件的環境中是安全的**，它們迫使應用程序邏輯在一個獨立的事件循環"輪流查詢"("turn")中繼續處理。如上面的例子，假設需要幾秒鐘來下載網絡資源，在這段時間裡，數量龐大的其他事件很可能發生。在同步的實現中，這些事件就會堆積在事件隊列中，而事件循環將停留等待該 JS 代碼執行完成，這將阻塞任何其他事件的處理。**在異步的版本中，JS 代碼註冊一個事件處理程序並立即返回，這將在下載完成之前，允許其他處理程序處理這期間的事件**。
在主應用程序事件隊列不受影響的環境中，阻塞操作很少出問題。例如，web 平台提供了 Worker 的 API，該 API 使得產生大量的並行計算成為可能。不同於傳統的線程執行，Workers 在一個完全隔離的狀態下執行，沒有獲取全局作用域或應用程序主線程 web 頁面內容的能力。因此，它們不會妨礙主事件隊列中運行的代碼的執行。在一個 Worker 中，使用 XMLHttpRequest 同步的變種很少出問題。下載操作的確會阻塞 Worker 繼續執行，但這並不會阻止頁面的渲染或事件隊列中的事件響應。在服務器端環境中，阻塞的 API 在啟動一開始是沒有問題的，也就是在服務器開始響應輸入的請求之前。然後在處理請求期間，瀏覽器事件隊列中存在阻塞的 API 就是有問題的。

- 異步 API 使用 callback 函數來延緩處理代價高昂的操作以避免阻塞主應用程序
- js 並發地接收事件，但會使用一個事件隊列按序地處理事件處理程序
- 在應用程序事件隊列中絕不要使用阻塞的 I/O

---

### 7-62 Use Nested or Named Callbacks for Asynchronous Sequencing

> 在異步序列中使用 Nested 或命名的 Callback function

上一個講述了異步 API 如何執行潛在的代價高昂的 I/O 操作，而不阻塞應用程序繼續處理其他輸入。理解異步程序的操作順序剛開始有點混亂。例如，下面的代碼會在打印 "finished" 之前打印 "starting"，即使這兩個動作的程序源文件中以相反的順序呈現。
```
downloadAsync('file.txt', function(file) {
    console.log('finished');
});
console.log('starting');
```

downloadAsync 調用會立即返回，不會等待文件完成下載。同時，js 的運行到完成機制確保下一行代碼會在處理其他事件處理程序之前被執行。也就是說 "starting" 一定會在 "finished" 之前被打印。
理解操作序列的最簡單的方式是**異步 API 是發起操作而不是執行操作(initiating rather than performing an operation)**。上面的程式碼發起了一個文件的下載然後立即印了"starting"。當下載完成後，在事件循環的某個單獨的輪次(separate turn)中，被註冊的事件處理程序才會印出"finished"。

#### 如何串聯異步操作

如果你需要在發起一個操作後做一些事情，如果只能在一行中放置好幾個宣告，那麼如何串聯已完成的異步操作呢？例如，如果我們需要在異步數據庫中查找一個 URL，然後下載這個 URL 的內容？不可能發起兩個連續的請求。
```
db.lookupAsync('url', function(url) {
    // do something...
});
downloadAsync(url, function(text) {
    // error:url is not bound
    console.log('contents of ' + url +':' + text);
});
```

以上程式碼不可能 work，因為從數據庫查詢到的 URL 結果需要作為 downloadAsync 方法的參數。但是它並不在作用域內。我們所做的這一步只是發起數據庫查找，查找的結果還不可用。

#### callback 函數處理

最簡單的處理方法使用嵌套(nesting)。借助於閉包的魔力，將第二個動作嵌套在第一個動作的 callback 函數中。
```
db.lookupAsync('url', function(url) {
    downloadAsync(url, function(text) {
       console.log('contents of ' + url + ':' + text);
    });
});
```

創建閉包能夠訪問外部 callback 函數的變數。嵌套的異步操作很容易，但當擴展到更長的序列時會很快變得麻煩。

#### callback 命名的函數

減少過多的嵌套的方法之一是將嵌套的 callback 函數作為命名的函數，並將它們需要附加數據作為額外的參數傳遞。以上代碼可以改寫為
```
db.lookupAsync('url', downloadURL);

function downloadURL(url) {
    downloadAsync(url, function(text) {
        // still nested
        showContents(url, text);
    });
}

function showContents(url, text) {
    console.log('contents of ' + url + ':'+ text);
}
```

#### 使用 bind 方法消除嵌套

為了合併外部的 url 變數和內部的 text 變量作為 showContents 方法的參數，在 downloadURL 方法中仍然使用了嵌套的 callback 函數。這裡可以使用 bind 方法消除最深層的嵌套回調函數。
```
db.lookupAsync('url', downloadURL);

function downloadURL(url) {
    downloadAsync(url, showContents.bind(null, url));
}

function showContents(url, text) {
    console.log('contents of ' + url + ':' + text);
}
```

這種做法可以使程式碼看起來很有順序性，但需要為操作序列的每個中間步驟命名，並且一步步地使用綁定。這可能在多層嵌套時導致尷尬的情況。


#### 结合兩種方法

結合這兩種方法，會使程式碼更易理解。

```
db.lookupAsync('url', function(url) {
   downloadURLAndFiles(url);
});

function downloadURLAndFiles(url) {
    downloadAsync(url, downloadFiles.bind(null, url));
}

function downloadFiles(url, file) {
    downloadAsync('a.txt', function(a) {
        downloadAsync('b.txt', function(b) {
            downloadAsync('c.txt', function(c) {
                //...
            });
        });
    });
}
```

最後一步可以使用一個額外的抽象來簡化，可以下載多個文件並將它們存儲在數組中。
```
function downloadFiles(url, file) {
    downloadAllAsync(['a.txt', 'b.txt', 'c.txt'], function(all) {
        var a = all[0], b = all[1], c = all[2];
        // ...
    });
}
```

使用 downloadAllAsync 函數允許我們同時下載多個文件。排序意味著每個操作只有等前一個操作完成後才能啟動。一些操作本質上是連續的，比如下載我們從數據庫查詢到的 URL。但如果我們有一個文件列表要下載，沒理由等每個文件完成下載後才請求接下來的一個。除了嵌套和命名回調，還可以建立更高層的抽象使異步控制流更簡單、更簡潔。

- 使用嵌套或命名的 callback 函數按順序地執行多個異步操作
- 嘗試在過多的嵌套的回調函數和尷尬的命名的非嵌套回調函數之間取得平衡
- 避免將可被並行執行的操作順序化

---

### 7-63 Be Aware of Dropped Errors

> 注意錯誤的處理

管理異步編程的其中一個是錯誤處理
(error handling)。同步的程式碼中只要使用 try 語句塊包裝一段代碼很容易一下子處理所有的錯誤。
```
try {
    f();
    g();
    h();
} catch(e) {
    // 這裡用來抓出現的錯誤
}
```

#### try 語句塊

但對於異步的程式碼，多步的處理通常會被分隔到事件隊列的單獨輪次("turn")中，因此，不可能將它們包裝在一個 try 語句塊中。事實上異步的 API 甚至根本不可能拋出異常，因為，當一個異步的錯誤發生時，沒有一個明顯的執行上下文來拋出異常！相反，異步的 API 傾向於將錯誤表示為回調函數的特定參數，或使用一個附加的錯誤處理回調函數。例如，一個涉及下載文件的異步 API 可能會有一個額外的回調函數來處理網絡錯誤。
```
downloadAsync('http://cnblogs.com/wengxuesong', function(text) {
    console.log('file contents:' + text);
}, function(error){
    console.log('error:' + error);
});
```

如果下載多個文件，可以像上面第 62 條講的，使用回調函數嵌套起來。
```
downloadAsync('a.txt', function(a) {
    downloadAsync('b.txt', function(b) {
        downloadAsync('c.txt', function(c) {
            console.log('Contents:'+ a + b + c);
        }, function(error){
            console.log('error:' + error);
        });
    }, function(error) {
        console.log('error:' + error);
    });
}, function(error){
    console.log('error:' + error);
});
```

上面程式碼上，每一步的處理都使用了相同的錯誤處理邏輯，然而我們在多個地方重複了相同的代碼。在編程領域裡，應該努力堅持避免重複代碼。通過共享作用域中定義一個錯誤處理的函數，將重複代碼抽像出來。
```
function onError(error) {
    console.log('Error:' + error);
}
downloadAsync('a.txt', function(a) {
    downloadAsync('b.txt', function(b) {
        downloadAsync('c.txt', function(c) {
            console.log('Contents:' + a + b + c);
        }, onError);
    }, onError);
}, onError);
```

如果使用如函數 downloadAllAsync 將多個步驟合併到一個複合的操作中，那麼，只需要提供一個錯誤處理的 callback 函數。
```
downloadAllAsync(['a.txt', 'b.txt', 'c.txt'], function(abc) {
    console.log('Contents:' + abc[0] + abc[1] + abc[2]);
}, function(error) {
   console.log('Error:' + error);
});
```

#### callback 函數錯誤參數

另一種錯誤處理 API 的風格是 Node.js 平台使用的。該風格只需要一個 callback 函數，該 callback 函數的第一個參數如果有錯誤發生就表示一個錯誤。否則就是一個假值，比如 null。對於這類 API，我們可以定義一個通用的錯誤處理函數，需要使用 if 語句來控制每個 callback 函數。
```
function onError(error) {
    console.log('Error:' + error);
}
downloadAsync('a.txt', function(error, a) {
    if(error) {
        onError(error);
        return;
    }
    downloadAsync('b.txt', function(error, b) {
        if(error) {
            onError(error);
            return;
        }
        downloadAsync('c.txt', function(error, c) {
            if(error) {
                onError(error);
                return;
            }
            console.log('Contents:' + a + b + c);
        });
    });
});
```

也可以使用一個抽象合併步驟來幫助避免重複
```
var filenames = ['a.txt', 'b.txt', 'c.txt'];
downloadAllAsync(filenames, function(error, abc) {
    if(error) {
        console.log('Error:' + error);
        return;
    }
    console.log('Contents:'+ abc[0] + abc[1] + abc[2]);
});
```

try...catch 語句和在異步 API 中典型的錯誤處理邏輯的一個實際差異是，try 語句使得定義一個"捕獲所有"的邏輯很容易導致工程師難以忘懷整個程式碼區的錯誤處理。而上面給出的異步 API，非常容易忘記在進程的任意一步提供錯誤處理。這將導致錯誤被丟棄。忽視錯誤處理的程序會令用戶非常沮喪：應用程序出錯時沒有任何的反饋。類似的，預設的錯誤不好調試。因為沒有提供問題來源的線索。最好是做好防禦，即使用異步 API 需要警惕，確保明確地處理所有的錯誤狀態條件。

- 通過編寫共享的錯誤處理函數來避免複製和黏貼錯誤處理代碼
- 確保明確地處理所有的錯誤條件以避免丟棄錯誤

---

### 7-64 Use Recursion for Asynchronous Loops

> 對異步的循環使用遞迴

假設需要有這樣一個函數:接收一個 URL 的陣列，並嘗試依次下載每個文件直到有一個文件被成功下載。如果 API 是同步的，使用循環很簡單實現。
```
function downloadOneSync(urls) {
    for(var i = 0, n = urls.length; i < n; i++) {
        try {
            return downloadSync(urls[i]);
        } catch(e) {}
    }
    throw new Error('all downloads failed.');
}
```

在異步情況下，上面這種方式就無法正確運作。因為不能在 callback 函數中暫停循環並恢復。如果嘗試使用循環，它將啟動所有的下載，這不是等待完成一個再進行下一個。
```
function downloadOneAsync(urls, onsucess, onerror) {
    for(var i = 0, n = urls.length; i < n; i++) {
        downloadAsync(urls[i], onsucess, function(error) {
            // ?
        });
        // loop continues
    }

    throw new Error('all downloads failed');
}
```

所以這裡我們要實現一個類似循環的東西，我們需要明顯地說繼續執行，它才會繼續執行。解決方案是將循環實現為一個函數，可以決定何時開始每次迭代。
```
function downloadOneAsync(urls, onsucess, onfailure) {
    var n = urls.length;
    function tryNextURL(i) {
        if (i >= n) {
            onfailure('all downloads failed');
            return;
        }

        downloadAsync(urls[i], onsuccess, function() {
            tryNextURL(i + 1);
        });
    }

    tryNextURL(0);
}
```

局部函數 tryNextURL 是一個遞迴函數。它的實現調用了其自身。典型的 javascript 環境中一個遞迴函數同步調用自身過多次會導致失敗。例如，下例中的遞迴函數試圖調用自身 10 萬次，在大多數的 js 環境中會產生一個運行時錯誤。
```
function countdown(n) {
    if (n === 0) {
        return 'done';
    } else {
        return countdown(n - 1);
    }
}

countdown(100000); // Uncaught RangeError: Maximum call stack size exceeded
```

當 n 太大時 countdown 函數會執行失敗，那麼如何確保 downloadOneAsync 函數是安全的呢？查看一下 countdown 函數提供的錯誤信息。
```
VM58:1 Uncaught RangeError: Maximum call stack size exceeded
```

js 環境通常在內存中保存一塊固定的區域，稱為調用棧(call stack)，用於記錄函數調用返回前下一步該做什麼。執行下面的小程序。
```
function negative(x) {
    return abs(x) * -1;
}

function abs(x) {
    return Math.abs(x);
}

console.log(negative(42));  // -42
```

當程序使用參數 42 調用 `Math.abs` 方法時，有幾個其他的函數調用也在進行，每個都在等待另一個的調用返回。在每個函數調用時，如下圖示項目符號(.)描述了在程序中已經發生的函數調用地方及這次調用完成後將返回哪裡。就像傳統的 stack 數據結構，這個信息遵循"後進先出"(last-in, first-out)協議。最新的函數調用將信息推入 stack ((表示為 call stack 最底層的一個 stack frame))，該信息也將首先從 stack 中彈出)。當 `Math.abs` 執行完畢，將會返回給 abs 函數，其將返回給 negative 函數，然後將返回到最外面的腳本。

![A call stack during the execution of a simple program](/assets/images/7-64_a_call_stack.png)

![A call stack during the execution of a recursive function](/assets/images/7-64_b_call_stack.png)

而當一個程序執行中有太多的函數調用，它會耗盡 stack 空間，最終拋出異常。這種情況被稱為 stack overflow。在此例中，調用 countdown 調用自身10萬次，每次推入一個 stack。存儲這麼多 stack 需要的空間量可能會耗盡大多數 js 環境分配空間，導致運行時錯誤。

再回頭來看看 downloadOneAsync 函數。不像 countdown 直到遞迴調用返回後才會返回，downloadOneAsync 只在異步 callback 函數中調用自身。記住異步 API 在其 callback 函數被調用前會立即返回。所以 downloadOneAsync 返回，導致其 stack 在任何遞迴調用將新的 stack frame 推入 stack 前，會從 call stack 中彈出。(事實上，callback 函數總在事件循環的單獨輪次中被調用，事件循環的每個輪次中調用其他事件處理程序的 call stack 最初是空的。)所以無論 downloadOneAsync 需要多少次迭代，都不會耗盡 stack 空間。

- 循環不能是異步的
- 使用遞迴函數在事件循環的單獨輪次中執行迭代
- 在事件循環的單獨輪次中執行遞迴，並不會導致 call stack overflow

---

### 7-65 Don’t Block the Event Queue on Computation

> 不要在計算時阻塞事件隊列

第 61 項解釋了異步 API 怎樣幫助我們防止一段程序阻塞應用程序的事件隊列。使用下面程式碼，可以很容易使一個應用程序陷入泥潭。
`while (true) { }`

而且它並不需要一個無限循環來寫一個緩慢的程序。程式碼需要時間來運行，而低效的算法或數據結構可能導致運行長時間的計算。效率不是 js 唯一關注的。基於事件的編程的確強加了一些特殊的約束。為了保持客戶端應用程序的高度交互性和確保所有傳入的請求在服務器應用程序中得到充分的服務，保持事件循環的每個輪次盡可能短是至關重要。否則，事件隊列會滯銷，其增長速度會超過分發處理事件處理程序的速度。在瀏覽器環境中，一些代價高昂的計算也會導致糟糕的用戶體驗，因為一個頁面的用戶界面無響應多數是由於在運行 js 代碼。

如果你的應用程序需要執行代價高昂的計算該怎麼辦呢？沒有一個完全正確的答案，但有一些通用的技術可用。也許最簡單的方法是使用像 Web 客戶端平台的 Worker API 這樣的並發機制(concurrency mechanism)。這對於需要搜索大量可移動距離的人工智能遊戲是一個很好的方法。遊戲可能以生成大量的專門計算移動距離的 worker 開始。例如：

```
var ai = new Worker('ai.js');
```

這將使用 ai.js 文件作為 worker 的腳本，產生一個新的 thread 獨立事件隊列的 concurrency 執行線程。該worker 運行在一個完全隔離的狀態--沒有任何應用程序對象的直接訪問。但是，應用程序與 worker 之間可以通過發送形式為字符串的 messages 來互動。所以，每當遊戲需要程序計算移動時，它會發送一個消息給 worker。
```
var userMove = /* ... */;
ai.postMessage(JSON.stringify({
    userMove: userMove
}));
```

postMessage 的參數被作為一個消息增加到 worker 的事件隊列中。為了處理 worker 的 response，遊戲會註冊一個事件處理程序。
```
ai.onmessage = function(event) {
    executeMove(JSON.parse(event.data).computerMove);
};
```

同時，ai.js 指示 worker 監聽消息並執行計算下一步移動所需的工作。
```
self.onmessage = function(event) {
    var userMove = JSON.parse(event.data).userMove;
    var computerMove = computeNextMove(userMove);
    var message = JSON.stringify({
        computerMove:computerMove
    });

    selft.postMessage(message);
};

function computeNextMove(userMove) {
    //...
}
```

不是所有的 js 平台都提供類似 Worker 這樣的API。而且有時傳遞消息的開銷可能會過於昂貴。另一種方法是將算法分解為多個步驟，每個步驟組成一個可管理的工作塊。第 48 項中搜索社交網絡圖的工作表算法：
```
Member.prototype.inNetwork = function(other) {
  var visited = {};
  var worklist = [this];

  while(worklist.length > 0) {
    var member = worklist.pop();
    if(member === other) {
        return true;
    }
  }

  return false;
};
```

如果這段程序核心的 while 循環代價太過高昂，搜索工作很可能會以不可接受的時間運行而阻塞應用程序事件隊列。即使我們可以使用 Worker API, 它也是昂貴或不方便實現的，因為它需要複製整個網絡圖的狀態或在 worker 中存儲網絡圖的狀態，並且總是使用消息傳遞來更新和查詢網絡。
幸運的是，這種算法被定義為一個步驟集的序列-- while 循環的迭代。可以通過增加一個 callback 參數將inNetwork 轉換為一個匿名函數，並像第 64 項講述的，將 while 循環替換一個匿名的遞迴函數。
```
Member.prototype.inNetwork = function(other, callback) {
    var visited = {};
    var worklist = [this];
    function next() {
        if (worklist.length === 0) {
            callback(false);
            return;
        }

        var member = worklist.pop();
        if (member === other) {
            callback(true);
            return;
        }

        // schedule the next iteration
        setTimeout(next, 0);
    }

    // schedule the first iteration
    setTimeout(next, 0);
};
```

這段程式碼的工作方式，為了替換 while 循環，這裡寫了一個局部的 next 函數，該函數執行循環中的單個迭代然後高度應用程序事件隊列來異步運行下一次迭代。這使得在些期間已經發生的其他事件被處理後才繼續下一次迭代。當搜索完成後，通過迭代的 next 來返回，從而有效地完成循環。
要調度迭代，我們使用多數 js 平台都可用的、通用的  setTimeout API 來註冊 next 函數，使 next 函數經過一段最少時間(0毫秒)後運行。這具有幾乎立刻將 callback 函數添加到事件隊列上的作用。值得注意的是，雖然 setTimeout 有相對穩定的跨平台移植性，但通常還有更好的替代方案。例如，在瀏覽器環境中，最低的超時時間被壓制為 4 毫秒，可以採用一種替代方案，使用 postMessage 立即壓入一個事件。
如果應用程序事件隊列的每個輪次中只執行算法的一個迭代。可以調整算法，自定義每個輪次中的迭代次數。這很容易實現，只須在 next 函數的主要部分的外圍使用一個循環計數器。
```
Member.prototype.inNetwork = function(other, callback) {
    // ...
    function next() {
        for(var i = 0; i < 10; i++) {
            //...
        }

        setTimeout(next, 0);
    }

    setTimeout(next, 0);
};
```

- 避免在主事件隊列中執行代價高昂的算法
- 在支持 Worker API 的平台，該 API 可以用來在一個獨立的事件隊列中運行長計算程序
- 在 Worker API 不可用或代價昂貴的環境中，考慮將計算程序分解到事件循環的多個輪次中

---

### 7-66 Use a Counter to Perform Concurrent Operations

> 使用計數器來執行並行操作

第 63 項建議使用 downloadAllAsync 函數接收一個 URL 陣列並下載所有文件，結果返回一個存儲了文件內容的陣列，每個 URL 對應一個字符串。downloadAllAsync 函數並不只有清理嵌套 callback 函數(nested callbacks)的好處，其主要好處是並行下載文件。我們可以在同一個事件循環中一次啟動所有文件的下載，而不用等待每個文件完成下載。並行邏輯是微妙的，很容易出錯。下面有實現有一個隱藏的缺陷。
```
function downloadAllAsync(urls, onsuccess, onerror) {
	var result = [], length = urls.length;
		if (length === 0) {
		setTimeout(onsuccess.bind(null, result), 0);
	}

	urls.forEach(function(url) {
		downloadAsync(url, function(text) {
			if (result) {
				reslut.push(text);
				if (result.length === urls.length) {
					onsuccess(result);
				}
			}
		}, function(error) {
			if (result) {
				result = null;
				onerror(error);
			}
		});
	});
}
```

這個函數有嚴重的錯誤，但首先讓我們看看它是如何工作的。先確保如果陣列是空的，則會使用空結果數組調用 callback 函數。如果不這樣做，這兩個 callback 將不會被調用，因為 forEach 循環是空的。接下來，遍歷整個 URL 陣列，為每個 URL 請求一個異步下載。每次下載成功，就將文件內容加入到 result 陣列中。如果所有 URL 都被成功下載，使用 result 陣列調用 onsuccess callback 函數。如果有任何失敗的下載，使用錯誤值調用 onerror callback 函數。如果有多個下載失敗，設置 result 陣列為 null，從而保證 onerror 只被調用一次，即在第一次錯誤發生時。
錯誤的例子:
```
var filenames = [
	"huge.txt", // huge file
	"tiny.txt", // tiny file
	"medium.txt" // medium-sized file
];

downloadAllAsync(filenames, function(files) {
	console.log("Huge file: " + files[0].length); // tiny
	console.log("Tiny file: " + files[1].length); // medium
	console.log("Medium file: " + files[2].length); // huge
}, function(error) {
	console.log("Error: " + error);
});
```

由於這些文件是並行下載的，事件可以以任意的順序發生(並因此被添加到應用程序事件序列)。例如，如果 tiny.txt 先下載完成，接下來是 medium.txt 文件，最後是 huge.txt 文件，則註冊到 downloadAllAsync 的 callback 函數並不會按照它們被創建的順序進行調用。但 downloadAllAsync 的實現是一旦下載完成就立即將中間結果保存在 result 陣列的末端。所以 downloadAllAsync 函數提供的保存下載文件內容的陣列順序是未知的。這個 API 幾乎不可用，因為無法確認哪個結果對應哪個文件。
程序的執行順序不能保證與事件發生的順序一致。
當一個應用程序依賴於特定的事件順序才能正常工作時，這個程序會遭受數據競爭。數據競爭是指多個並發操作可以修改共享的數據結構，這取決於它們發生的順序。數據競爭是真正棘手的錯誤。它們可能不會出現於特定的測試中，因為運行相同的程序兩次，每次可能會得到不同的結果。例如 downloadAllAsync 的使用者可能會對文件重新排序，基於的順序是哪個文件可能會最先完成下載。

```
downloadAllAsync(filenames, function(files) {
  	console.log('Huge file:' + files[2].length);
  	console.log('Tiny file:' + files[0].length);
  	console.log('Medium file:' + files[1].length);
}, function(error) {
  	console.log('Error: ' + error);
});
```

上面的這種情況大多數結果是相同的順序，但偶爾會由於改變了 server 負載均衡或網絡緩存，文件可能不是期望的順序。我們可以順序下載文件，但也失去了並發的性能優勢。
下面實現 downloadAllAsync 不依賴不可預期的事件執行順序而總能提供預期結果。我們不將每個結果放置到陣列尾端，而是儲存在其原始的索引位置。
```
function downloadAllAsync(urls, onsuccess, onerror) {
	var result = [];
	var length = urls.length;

	if (length === 0) {
		setTimeout(onsuccess.bind(null, result), 0);
		return;
	}

	urls.forEach(function(url, i) {
		downloadAsync(url, function(text) {
			if (result) {
				reslut[i] = text;  // store at fixed index

				// race condition
				if (result.length === urls.length) {
					onsuccess(result);
				}
			}
		}, function(error) {
			if (result) {
				result = null;
				onerror(error);
			}
		});
	});
}
```

這個方法利用了 forEach callback 的第二個參數。第二個參數為當前迭代提供了陣列索引。但這也不正確。第 51 項描述了陣列更新的約定，即是設置一個索引屬性，總是確保陣列的 length 屬性值大於索引。假設有如下的一個請求。
```
downloadAllAsync(["huge.txt", "medium.txt", "tiny.txt"]);
```

如果 tiny.txt 文件最先被下載，結果陣列將獲取索引為 2 的屬性，這將導致 result.length 被更新為 3。用戶的 success callback 函數將被過早地調用，結果為一個不完整的陣列。
正確的實現應該是使用一個計數器來追踪正在進行的操作數量。
```
function downloadAllAsync(urls, onsuccess, onerror) {
	var pending = urls.length;
	var result = [];

	if (pending === 0) {
		setTimeout(onsuccess.bind(null, result), 0);
		return;
	}

	urls.forEach(function(url, i) {
		downloadAsync(url, function(text) {
			if (result) {
				result[i] = text; // store at fixed index
				pending--; // register the success

				if (pending === 0) {
                    onsuccess(result);
                }
			}
		}, function(error) {
			if (result) {
				result = null;
                onerror(error);
            }
		});
	});
}
```

如此，不論事件以什麼樣的順序發生，pending 計數器都能準確地指出何時所有的事件會被完成，並以適當的順序返回完整的結果。

- js 應用程序中的事件發生是不確定的，即順序是不可預測的
- 使用計數器避免並行操作中的數據競爭(data races)

---

### 7-67 Never Call Asynchronous Callbacks Synchronously

> 絕不要同步地調用異步的 callback 函數

假設有 downloadAsync 函數的一種變種，它持有一個緩存(類似實作見第 45 項裡的 Dict)來避免多次下載同一個文件。在文件已經被緩存的情況下，立即調用 callback 函數是最優選擇。
```
var cache = new Dict();

function downloadCachingAsync(url, onsuccess, onerror) {
	if (cache.has(url)) {
		onsuccess(cache.get(url)); // synchronous call
		return;
	}

	return downloadAsync(url, function(file) {
		cache.set(url, file);
		onsuccess(file);
    }, onerror);
}
```

通常情況下，它會立即提供數據，但這種方式是違反了異步 API 客戶端的期望。首先，它改變了操作的預期順序。第 62 項顯示了下面的例子，對於循規蹈矩的異步 API 應該總是以一種可預測的順序來記錄 log messages。
```
downloadAsync("file.txt", function(file) {
	console.log("finished");
});

console.log("starting");
```

使用上面的 downloadCachingAsync 實現，這樣的客戶端代碼可能最終會以任意的順序記錄事件，這取決於文件​​是否已被緩存起來。

```
downloadCachingAsync("file.txt", function(file) {
	console.log("finished"); // might happen first
});

console.log("starting");
```

log message 的順序是一回事。更普遍的是，異步 API 的目的是維持事件循環中每輪的嚴格分離。正如第 61 項解釋的，這簡化了並發(concurrency)，通過減輕每輪事件循環的代碼量而不必擔心其他代碼並發地修改共享的數據結構。同步地調用異步的 callback 函數違反了這個分離，導致在當前輪完成之前，程式碼用於執行一輪隔離的事件循環。(An asynchronous callback that gets called synchronously violates this separation, causing code intended for a separate turn of the event loop to execute before the current turn completes.)
例如，應用程序可能會持有一個剩餘的文件隊列給用戶下載和顯示消息。
```
downloadCachingAsync(remaining[0], function(file) {
	remaining.shift();
	// ...
});

status.display("Downloading " + remaining[0] + "...");
```

如果同步地調用該 callback 函數，那麼將顯示錯誤的文件名的 message (或者更糟糕的是，如果隊列為空會顯示"undefined")。
同步的調用異步的 callback 函數甚至可能會導致一些微妙的問題。第 64 項解釋了異步的 callback 函數本質上是以空的 call stack 來調用，因此將異步的循環實作為遞迴函數是安全的，完全沒有累積超越 call stack 空間的危險。但是，同步的調用不能保障這一點，因而使得一個表面上的異步循環很可能會耗盡 call stack 的空間。另一種問題是異常。對於上面的 downloadCachingAsync 實作，如果 callback 函數拋出一個異常，它將會在每輪的事件循環中，也就是開始下載時而不是期望的一個分離的回合中拋出該異常。

為了確保總是異步地調用 callback 函數，我們可以使用已存在的異步 API。就像我們在第 65 項和第 66 項中所做的一樣，我們使用通用的 library 函數 setTimeout 在每隔一個最小的超時時間後給事件隊列增加一個 callback 函數。可能有比 setTimeout 函數更完美的替代方案來調度即時事件，這取決於特定平台。
```
var cache = new Dict();

function downloadCachingAsync(url, onsuccess, onerror) {
	if (cache.has(url)) {
		var cached = cache.get(url);
		setTimeout(onsuccess.bind(null, cached), 0);
		return;
	}

	return downloadAsync(url, function(file) {
        cache.set(url, file);
        onsuccess(file);
    }, onerror);
}
```

上面這裡使用 bind 函數將結果保存為 onsuccess callback 函數的第一個參數。

- 即使可以立即得到數據，也絕不要同步地調用異步的 callback 函數
- 同步地調用異步的 callback 函數會擾亂預期的操作序列，並可能導致意想不到的交錯的程式碼
- 同步地調用異步的 callback 函數可能導致 stack overflow(溢出)或錯誤地處理異常
- 使用異步的 API，比如 setTimeout 函數來調度異步的 callback 函數，使其運行於另一回合

---

### 7-68 Use Promises for Cleaner Asynchronous Logic

> 使用 promise 模式來清理你的異步邏輯

構建異步 API 的一種流行的替代方式是使用 promise (有時也被稱為 deferred 或 future)模式。在本章中討論過的異步 API 以 callback 為參數：
```
downloadAsync("file.txt", function(file) {
	console.log("file: " + file);
});
```

基於 promise 的 API 不接收 callback 函數作為參數。相反地，它返回一個 promise 對象，該對象通過其自身的 then 方法接收 callback 函數。
```
var p = downloadP("file.txt");

p.then(function(file) {
	console.log("file: " + file);
});
```

這裡看不出與原先的版本有什麼不同。但是 promise 的力量在於它們的組合性。傳遞給 then 方法的 callback 函數不僅產生影響，也可以產生結果。通過 callback 函數返回一個值，可以構造一個新的promise。
```
var fileP = downloadP("file.txt");

var lengthP = fileP.then(function(file) {
	return file.length;
});

lengthP.then(function(length) {
	console.log("length: " + length);
});
```

理解 promise 的一種方法是將它理解為表示最終值的對象。它封裝了一個還未完成的並發操作，但最終會產生一個結果值。then 方法允許我們提供一個代表最終值的一種類型的 promise 對象，並產生一個新的 promise 對象來代表最終值的另一種類型，而不管 callback 函數返回了什麼。
從現有的 promise 中構造新 promise 的能力帶來了很大的靈活性，並且具有一些簡單但強大的慣用法。例如，構造一個實用程序來拼接多個 promise 的結果。
```
var filesP = join(downloadP("file1.txt"),
			      downloadP("file2.txt"),
                  downloadP("file3.txt"));

filesP.then(function(files) {
	console.log("file1: " + files[0]);
	console.log("file2: " + files[1]);
	console.log("file3: " + files[2]);
});
```

promise library 也提供一個叫做 when 的工具函數，其使用方法類似。
```
var fileP1 = downloadP("file1.txt"),
	fileP2 = downloadP("file2.txt"),
	fileP3 = downloadP("file3.txt");

when([fileP1, fileP2, fileP3], function(files) {
	console.log("file1: " + files[0]);
	console.log("file2: " + files[1]);
	console.log("file3: " + files[2]);
});
```

使 promise 成為卓越抽象層級的部分原因是通過 then 方法的返回值來聯繫結果，或者通過工具函數如 join 來構成 promise，而不是在並行的 callback 函數間共享數據結構。本質上是安全的，因為避免了第 66 項中討論過的數據競爭(data racing)。即使最小心謹慎的工程師也可能會在保存異步操作的結果到共享的變數或數據結構時犯下簡單的錯誤。
```
var file1, file2;

downloadAsync("file1.txt", function(file) {
	file1 = file;
});

downloadAsync("file2.txt", function(file) {
	file1 = file;  // wrong variable
});
```

promise 避免這種 BUG，簡單風格的組合 promise 避免了修改共享數據。
要注意的是，異步邏輯的有序鏈事實上也可用有序的 promise，而不是在第 62 項中展現的笨重的嵌套(nested)模式。錯誤處理會自動地通過 promise 傳播。當你通過 promise 串聯異步操作的集合時，你可以為整個序列提供一個簡單的 error callback 函數，而不是將 error callback 函數傳遞給每一步(如第 63 項中的程式碼所示)。
儘管如此，有時故意創建某些種類的數據競爭是有用的。 promise 提供了一個很好的機制。例如，一個應用程序可能需要嘗試從多個不同的 server 上同時下載同一份文件，而選擇最先完成的那個文件。 select(或choose)工具函數接收幾個 promise 並產生一個其值是最先完成下載的文件的 promise。換句話說，如同幾個promise 彼此競爭。
```
var fileP = select(downloadP("http://example1.com/file.txt"),
				   downloadP("http://example2.com/file.txt"),
                   downloadP("http://example3.com/file.txt"));

fileP.then(function(file) {
	console.log("file: " + file);
});
```

select 函數的另一個用途是提供超時來終止長時間的操作。
```
var fileP = select(downloadP("file.txt"), timeoutErrorP(2000));

fileP.then(function(file) {
	console.log("file: " + file);
}, function(error) {
	console.log("I/O error or timeout: " + error);
});
```

上面的方法提供了 error callback 函數作為第二個參數給 promise 的 then 方法。

- promise 代表最終值，即並行操作完成時最終產生的結果
- 使用 promise 組合不同的並行操作
- 使用 promise 模式的 API 避免數據競爭(data races)
- 在要求有意義的競爭條件時使用 select(也被稱為 choose)

### 補充

[concurrency mode](https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/EventLoop)





<!-- https://juejin.im/post/5ab73dcdf265da237e09af43 -->
<!-- https://www.cnblogs.com/wengxuesong/p/5674544.html -->
<!-- https://www.ctolib.com/docs/sfile/effective-javascript/chapter-6/avoid-unnecessary-state.html -->