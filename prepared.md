# CH 3 DOM Scripting
> DOM Scripting

## 1. 瀏覽器中的 DOM

> DOM 的操作天生就慢，儘量減少 DOM 操作，及訪問 DOM 的次數。

DOM (獨立語言，使用 XML 和 HTML 操作的 API)在瀏覽器中的接口是以 JS 實現的。而瀏覽器通常要求 DOM implement 和 JS implement 保持相互獨立，例如，Chrome 用 WebKit 的 WebCore library 來渲染頁面，另外有自己的 JS 引擊(V8)。如此，兩個獨立的部分以功能連接就會帶來性能損耗。

把 DOM 和 JS(ECMAScript) 看成兩個獨立的島嶼，兩者之間以一座收費橋連接。每次 JS 需要訪問 DOM 時，需要交一次"過橋費"，所以操作 DOM 次數越多，費用就越高。

<!-- [DOM mdn] (https://developer.mozilla.org/zh-CN/docs/Web/API/Document_Object_Model/Introduction)[https://stackoverflow.com/questions/2726554/what-is-the-difference-between-javascript-and-dom](https://stackoverflow.com/questions/2726554/what-is-the-difference-between-javascript-and-dom)
-->

## 2. DOM 的訪問與修改

訪問一次 DOM 就會增加一個性能成本，修改 DOM 元素的代價可能更高。因為可能會導致瀏覽器需要重新計算頁面的變化。最糟糕的情況是用 loop 來操作 DOM 的修改：
```
function innerHTMLLoop() {
	for (var count = 0; count < 15000; count++) {
		document.getElementById('here').innerHTML += 'a';
	}
}
```

上面這個是在每次循環中都對 DOM 元素訪問了兩次: 一次讀取 `innerHTML` 屬性內容，另一次寫入它。更有效率的版本是使用區域變數儲存更新後的内容，在循環結束時一次寫入：
```
function innerHTMLLoop2() {
	var content = '';
	for (var count = 0; count < 15000; count++) {
		content += 'a';
	}

	document.getElementById('here').innerHTML += content;
}
```

需要多次訪問某個 DOM 節點時，使用區域變數儲存它的引用。

### 2.1 HTML Collections HTML 集合

HTML 集合是 **array-like objects**，存放了 DOM 節點的引用(references)，比如：
```
document.getElementsByName()
document.getElementsByClassName()
document.getElementsByTagName_r()
document.links
...
```

它們都會 return 一個 array-like lists(不是陣列，但帶有 `length` 屬性)，而每次在作查詢、更新時，都要重複執行使用 index 來訪問列表中的元素，這也是導致性能效率低的原因。例如：
```
// an accidentally infinite loop
var alldivs = document.getElementsByTagName_r('div');
for (var i = 0; i < alldivs.length; i++) {
	document.body.appendChild(document.createElement('div'))
}
```

上面的目的是，增加頁面中 div 的數量。它遍歷現有的 div，並且每次遍歷時創建一個新 div 並附加到 body 上面。但實際上這是個死循環，因為循環終止條件 `alldivs.length` 在每次迭代中都會增加，它反映出底層文檔的當前狀態。這樣的遍歷容易導致邏輯錯誤，而且因為每次迭代都要查詢而效能極慢。

如果需要操作集合，建議是把它複製到一個陣列中，把集合的長度存到一個變數中，並在迭代時使用它：
```
// function that copies an HTML collection into a regular array

function toArray(coll) {
	for (var i = 0, a = [], len = coll.length; i < len; i++) {
		a[i] = coll[i];
	}

	return a;
}

// setting up a collection and a copy of it into an array

var coll = document.getElementsByTagName_r('div');
var ar = toArray(coll);

//  simply cache the length of the collection into a variable and use this variable to compare in the loop's exit condition

function loopCacheLengthCollection() {
	var coll = document.getElementsByTagName_r('div'),
		len = coll.length;

	for (var count = 0; count < len; count++) {}
}
```

### 2.2 The Selectors API

使用速度更快的 API，比如 `querySelectorAll()` 和 `firstElementChild()`，例如：
```
var elements = document.querySelectorAll('#menu a');
```

函數 querySelectorAll() 返回一個 **NodeList** ー由符合條件的節點構成的 array-like object。此函數不返回 HTML 集合，所以返回的節點不呈現文檔的"存在性結構"。這就避免了前面提到的 HTML 集合所固有的性能問題(以及潛在的邏輯問題)。

如果不用 `querySelectorAll()` 的話：
```
var elements = document.getElementById('menu').getElementsByTagName_r('a');
```

拿到的會是一個 HTML 集合，所以還需要將它複製到一個陣列中，需要另作更多工作(如果想得到與 `querySelectorAll()` 同樣的返回值類型的話)。

## 3. Repaints and Reflows

![browser repaints and reflows](/assets/images/repaints_and_reflows.png)

1. 瀏覽器引擎首先開始解析 HTML 文檔，將各標籤逐個轉化成 DOM 節點，生成 DOM Tree。
2. 在解析 HTML 文檔的同時也會解析外部 CSS 文件以及 HTML 中 Style 標籤裡的樣式數據（也包括 JS 動態生成的樣式），這些樣式數據將用於創建另一個樹結構：Render Tree。它包含多個帶有視覺屬性（如顏色和尺寸）的矩形(boxes with paddings, margings, borders, and position)。這些矩形的排列順序將決定它們在屏幕上顯示的順序。
3. DOM Tree 和 Render Tree 構造完畢後，進入 layout 處理階段，也就是為每個節點分配一個應該出現在屏幕上的確切坐標。
4. 進入到繪製階段，瀏覽器渲染引擎會遍歷 Render Tree，將每個節點繪製出來。

整個過程是漸進完成的，渲染引擎為了盡快將內容顯示在屏幕上，不等整個 HTML 文檔解析完畢，就開始構建 Render Tree 和 layout，部分內容開始解析並顯示出來，直到整個網頁呈現在我們面前。而 repaint 和 reflow 就發生在第 3 和 第 4 階段。

- repaint —— 重繪，當一個 DOM 元素的外觀發生改變時(例如：outline, visibility, color, background color 等改變)， 瀏覽器會根據元素的新樣式屬性重新繪製，使元素呈現新的外觀，它不會影響到佈局。重繪的性能代價是高昂的，因為瀏覽器必須驗證 DOM 樹上其他節點元素的可見性。

- reflow —— 回流，可以理解為 Render Tree 需要重新計算，並根據計算的結果將元素渲染到它所在屏幕上的確切坐標，回流影響到部分（或整個）頁面的 layout，一個元素回流會導致它所有的子元素、祖先元素、兄弟元素發生回流。

repaint 和 reflow 是負擔很重的操作，可能導致網頁應用的用戶界面失去相應。所以，十分有必要盡可能減少這類事情的發生。

### 3.1 Repaints 和 Reflows 何時發生

- 哪些情況只發生 repaint

只修改 DOM 元素的顏色屬性(color, background-color, border-color...)或 visibility、outline 屬性，因為不會改變 layout 所以只發生 repaint。

- 哪些情況會發生 reflow

	1. 使用 JS 增加、删除、或修改 DOM 節點(包括移除或增加樣式表)
	2. 設置元素的 style 屬性
	3. 更改字體
	4. CSS 中修改 display 屬性
	5. 元素位置改變
	6. 元素尺寸改變(margin, padding,border thickness, width, height, etc.)
	7. 網頁上內容發生變化，例如在 input 或 textarea 裡输入文字
	8. 內容改變(例如文本改變或圖片被另一個不同尺寸的所替代)
	9. 最初的頁面渲染
	10. 瀏覽器視窗改變尺寸
	...

- 一些減少 reflows 的優化建議：

1. 操作元素樣式：
	- 避免同時設置多個元素的內聯樣式(style 屬性)
	- 實現動畫時，最好應用在 position 為 `fixed` 或 `absolute` 的元素上
		- 避免對大部分頁面進行重排版
			- 頁面上可以"折疊/展開"的元素稱作"動畫元素"，用絕對坐標對它進行定位，當它的尺寸改變時，就不會推移頁面中其他元素的位置，而只是覆蓋其他元素。
			- 展開動作只在"動畫元素"上進行。這時其他元素的坐標並沒有改變，換句話說，其他元素並沒有因為它的擴大而隨之下移，而是被它覆蓋。
			- "動畫元素"的動畫結束時，將其他元素的位置下移到動畫元素下方，讓界面只"跳"一下。
	- 避免使用 table 佈局，因為可能很小的一個改動會造成整個 table 重新佈局。
	- 不要使用 CSS JavaScript 表達式
		- 例如：`color: expression((new Date()).getHours() % 2 ? '#0091ea' : '#00b8d4')`
		- 問題在於它的計算頻率要比我們想像的多。不僅僅是在頁面顯示和縮放時，就是在頁面滾動、移動鼠標時都會要重新計算一次，非常容易損耗性能。

2. 修改 DOM：
	- 如果想用 JS 修改元素的樣式，最好通過改變元素的 class 名稱，並儘可能在 DOM Tree 最末端的節點上修改（例如可以想辦法只修改元素子節點上的 class）
	- 不要多次修改 DOM，可以使用 `document.createDocumentFragment()` 把要改的 DOM 節點緩存起來 在內部修改，再一次性添加進 HTML 中
	- 將要修改的 DOM 節點設置 `display:none`，會有一次 repaint，接著可以多次修改，修改完後再設置為 `display:block`

例子 1 - 修改 class：
```
// bad ((原則上)會導致瀏覽器 reflows 三次)
var el = document.getElementById('mydiv');
el.style.borderLeft = '1px';
el.style.borderRight = '2px';
el.style.padding = '5px';

// good
var el = document.getElementById('mydiv');
el.className = 'active';
```

例子 2 - documentFragment
```
// 刷 100 次
for ( var i = 0; i < 100; i++ ) {
	var item = document.createElement('div');

	$(item).text('Element-' + i);
	$(item).css({
		background: 'gold',
		padding: 5,
		margin: 5,
		float: 'left'
	});

	$('body').append(item);
}
// 改成用 documentFragment
// 刷 1 次
var fragment = document.createDocumentFragment();

for ( var i = 0; i < 100; i++ ) {
	var item = document.createElement('div');

	$(item).text('Element-' + i);
	$(item).css({
		background: 'gold',
		padding: 5,
		margin: 5,
		float: 'left'
	});

	fragment.appendChild(item);
}

$('body').append(fragment);
```

documentFragment 一個便利的語法特性是當你向節點附加一個片斷(fragment)時，實際添加的是它的子節點群，而不是自己。但是瀏覽器支援程度不高。

### 3.2 Minimize the number of reflows 最小化 relows

> 查詢並刷新 Render Tree 的改變
>
> 緩存 layout 信息

在修改 style 的過程中，盡量不要使用下列這類的屬性，因為每次查詢時瀏覽器就會刷新一次 render queue 中待改變(pendding))的項目並重新排版(reflow)以返回正確的值。
	- offsetTop, offsetLeft, offsetWidth, offsetHeight, scrollTop...
	(layout 相關的信息都是由這類屬性或方法來返回最新的值)

最好還是盡量減少對 layout 信息的查詢次數，查詢時將它 assign 給區域變數，並用區域變數參與計算，舉個例子：
```
// inefficient
var myElement.style.left = 1 + myElement.offsetLeft + 'px';
var myElement.style.top = 1 + myElement.offsetTop + 'px';
if (myElement.offsetLeft >= 500) {
  stopAnimation();
}

// better
var current = myElement.offsetLeft;

current++
myElement.style.left = current + 'px';
if (current >= 500) {
  stopAnimation();
}
```

### 3.3 其他

- Use event delegation to minimize the number of event handlers.

> 使用事件托管來最小化 event handler 的數量。

event bubble up 總能被父元素捕獲。採用事件託管技術之後，只需要在一個包裝元素上掛接一個 event handler，用於處理子元素發生的所有事件。
```
<div class="parent">
	<div class="child" data-name="a"></div>
	<div class="child" data-name="b"></div>
	<div class="child" data-name="c"></div>
	<div class="subitem" data-name="d"></div>
</div>

$('.parent').on('click', '.child', function(){
    console.log($(this).data('name'));
});

// a
// b
// c
// (不會有 d)
```

將 click 事件綁在 parent 上，藉由 Event Bubbling 來傳遞給 child，而非直接將事件綁定在 child 上。優點是可以減少監聽器的數目，缺點是由於需要判斷哪些 child node 是我們有興趣的項目，而必須多寫一些程式碼做判斷。

- IE and :hover

自從 IE7 之後，IE 可以在任何元素(嚴格模式)上使用 `:hover` 這個 CSS 偽選擇器。然而，如果大量的元素使用了 `:hover` 會降低反應速度。此問題在 IE8 中更顯著。



<!-- http://cythilya.blogspot.com/2015/07/javascript-event-delegation.html -->


<!--
repaints and reflows
https://mobidev.biz/blog/how_to_optimize_the_performance_of_phonegap_apps
http://www.mrfront.com/2016/10/09/repaint-reflow/
https://juejin.im/post/5a9372895188257a6b06132e
https://www.jianshu.com/p/b8f7e6c32a1b
https://blog.kaolafed.com/2017/03/30/repaint%E4%B8%8Ereflow/
https://leohxj.gitbooks.io/front-end-database/theory/repaint-and-reflow.html

DocumentFragment
https://developer.mozilla.org/zh-TW/docs/Web/API/DocumentFragment
-->





<!-- http://jstherightway.org/zh-tw/ -->
<!-- https://blog.csdn.net/situdesign/article/details/5631527 -->
<!-- https://blog.csdn.net/situdesign/article/details/5631524 -->