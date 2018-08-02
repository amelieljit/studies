# CH 4 Algorithms and Flow Control
> 演算法和流程控制

## 1. Loops 循環

> 代碼整體結構是執行速度的決定因素之一。代碼量少不一定運行速度快，代碼量多也不一定運行速度慢。性能損失與代碼組織方式和具體問題解決辦法直接相關。

### Types of Loops

四種基本的 JS loops:
1. for loop
```
for (var i = 0; i < 10; i++) {
	// loop body
}
```

執行順序:
```
初始化體: `var i = 0` (先執行)
↓
前測條件(pretest condition): `i < 10`
↓ (結果為 `true`)
循環體: loop body
↓
後執行體(post-execute): `i++`
```

2. while loop
- 是一個簡單的預測試循環，由一個前測條件和一個循環體構成
```
var i = 0;
while(i < 10) {
	// loop body

	i++;
}
```

執行順序:
```
前測條件: `i < 10` (先對前測條件進行計算)
↓ (結果為 `true`)
循環體: loop body
```

(任何 for 循環都可以寫成 while 循環，反之亦然。)

3. do-while loop
- JS 中唯一一種後測試循環，包含循環體和後測試條件(post-test condition)兩部分
- 循環體至少執行一次
```
var i = 0;
do {
	// loop body
} while (i++ < 10);
```

執行順序:
```
循環體: loop body
↓
後測試條件: `i++ < 10`
```

4. for-in loop
- 返回的屬性包括對象的實例屬性和它從原型鏈繼承而來的屬性。
```
for (var prop in object) {
	// loop body
}
```

## Loop Performance 循環性能

> JS 的四種循環，以 for-in loop 較其他還慢。

由於每次迭代操作要搜索實例或原形的屬性，for-in 循環每次迭代都要付出更多開銷，所以比其他類型循環慢一些。在同樣的循環迭代操作中，for-in
循環比其他類型的循環慢 7 倍之多。因此，除非你需要對數目不詳的對象屬性進行操作，否則避免使用 for-in 循環。

如果要迭代遍歷一個有限的，已知的屬性列表，使用其他循環類型更快，可使用如下模式：
```
var props = ['prop1', 'prop2'];
var i = 0;

while (i < props.length) {
	process(object[props[i]]);
}
```

創建一個由成員和屬性名構成的陣列。 while 循環用於遍歷這幾個屬性並處理所對應的對象成員，而不是遍歷對象的每個屬性。只關注感興趣的屬性，節約了循環時間。

若無關性能時，選擇使用何種循環，取決於兩個因素：
1. Work done per iteration 每次的迭代做了什麼
2. Number of iterations 迭代的次數

此外，透過減少以上這兩者中的一個或者全部的執行時間，來降低性能負擔。

### Decreasing the work per iteration
> 減少迭代的工作量

限制在循環體(loop body)內進行耗時操作的數量是一個加快循環的好方法。因為若一次循環迭代需要較長時間來執行，那麼多次循環將需要更長時間。

舉個 🌰: array-processing loop
```
// for loop
for (var i = 0; i < items.length; i++) {
	process(items[i]);
}

// while loop
var j = 0;
while (j < items.length) {
	process(items[j++]);
}

// do-while loop
var k = 0;
do {
	process(items[k++]);
} while (k < items.length);
```

每次執行循環體(loop body)時都會發生以下幾個操作：
1. 在控制條件中(control condition)**讀一次屬性(`items.length`)**
2. 在控制條件中**執行一次比較(`i < items.length`)**
3. 查看控制條件的**運算結果是不是 true (`i < items.length == true`)**
4. 一次**自加操作(`i++`)**
5. 一次**陣列查找(`items[i]`)**
6. 一次**函數調用(`process(items[i])`)**

這樣簡單的循環中，雖然沒有太多的程式碼，但每次迭代時也要進行許多操作。程式碼執行速度很大程度上取決於 `process()` 對每個項目的操作所決定的。然而，減少每次迭代中操作的總數，可以大幅度提高循環的整體性能。

優化第一步，最直接有效的就是減少對象成員和陣列項目查找的次數。在大部分瀏覽器上，這些操作比訪問局部變數或字面量需要更長的時間。每次循環查找`items.length` 就是一種浪費，因為在循環體執行過程中不會改變，因此產生不必要的性能損失。
```
var len = items.length;
for (var i = 0; i < len; i++) {
	process(items[i]);
}
// 只在循環執行之前對陣列長度進行一次查詢
```

根據陣列的長度，在大多數瀏覽器上可以節省大約 25％ 的總循環時間（在 IE 可節省 50％)。

另外，若陣列元素的處理順序與任務無關，可以從最後一個開始，直到處理完第一個元素。使用倒循環：
```
var len = items.length;
for (var i = len; i--) {
	process(items[i]);
}
```

上面例子中使用了倒序循環，並在控制條件中使用了減法。每個控制條件只是簡單地與零進行比較。控制條件與 true 值進行比較，任何非零數字自動強制轉換為 true，而零等同於 false。實際上，控制條件已經從兩次比較(迭代少於總數嗎？它等於 true嗎？)減少到一次比較(它等於true嗎？)。將每個迭代中兩次比較減少到一次可以大幅度提高循環速度。通過倒序循環和最小化屬性查詢，可以使執行速度比原始版本快了約 50% - 60%。

與原始版本相比，每次迭代中只進行以下四個操作：
1. 在控制條件中**進行一次比較(`i == true`)**
2. 一次**減法操作(`i--`)**
3. 一次**陣列查詢(`items[i]`)**
4. 一次**函數調用(`process(items[i])`)**

### Decreasing the number of iterations
> 減少迭代次數

"[Duff's Device](https://zh.wikipedia.org/zh-tw/%E8%BE%BE%E5%A4%AB%E8%AE%BE%E5%A4%87)" 是一種展開迴圈體技術(a technique of unrolling loop bodies)。Jeff Greenberg(被認為是將達夫循環從原始的 C 實現移植到 JavaScript 中的第一人)
```
// credit: Jeff Greenberg

var iterations = Math.floor(items.length / 8);
var startAt = items.length % 8;
var i = 0;

do {
	switch (startAt) {
		case 0: process(items[i++]);
		case 7: process(items[i++]);
		case 6: process(items[i++]);
		case 5: process(items[i++]);
		case 4: process(items[i++]);
		case 3: process(items[i++]);
		case 2: process(items[i++]);
		case 1: process(items[i++]);
	}
	startAt = 0;
} while (--iterations);
```

每次循環中最多可 8 次調用 `process()`。循環迭代次數為元素總數除以 8。因為總數不一定是 8 的整數倍，所以 `startAt` 變數存放餘數，指出第一次循環中應當執行多少次 `process()`。比方說現在有 12 個元素，那麼第一次循環將調用 `process()` 4 次，第二次循環調用 `process()` 8次，用 2 次循環代替了 12次循環。

```
// credit: Jeff Greenberg
// 速度比上面那個稍快
// 取消了 switch 表達式，將餘數處理與主循環分開，速度更快)

var i = items.length % 8;

while (i) {
	process(items[i--]);
}

i = Math.floor(items.length / 8);

while (i) {
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
	process(items[i--]);
}
```

迭代工作量大於 1000 的時候適用 Duff's Device 這個技術。

### Function-Based Iteration
> 基於函數的迭代

**forEach()**
> [forEach()](https://goo.gl/5yPkxa)

遍歷一個陣列的所有成員，並在每個成員上執行一個函數。在每個元素上執行的函數作為 `forEach()` 的參數傳進去，並在調用時接收三個參數：陣列項目的值，陣列項目的索引，和陣列自身。
```
var arr = ['a', 'b', 'c'];

arr.forEach(function(element) {
	console.log(element);
});

// output
// a
// b
// c
```

除了各大瀏覽器幾乎都有支援外，有些 JS library 也有等價實現，比如
```
//jQuery
jQuery.each(items, function(index, value) {
	process(value);
});
```

基於函數的迭代雖然顯得更加便利，但是它還是比基於循環的迭代要慢一些。因為每個陣列項目要關聯額外的函數調用(by the overhead associated with an extra method being called on each array item.)，會是造成速度慢的原因。在所有情況下，基於函數的迭代佔用時間是基於循環的迭代的八倍，因此在關注執行時間的情況下它並不是一個合適的辦法。

## Conditionals
> 條件語句

conditionals 決定了 JS 執行流(execution flows)的走向，然而由於不同的瀏覽器針對流程控制進行了不同的優化，使用哪種技術較好就要適情況而定了。

### if-else Versus switch
> if-else 和 switch 的比較

使用 if-else 或者 switch 的大部分理論是基於測試條件的數量：條件數量較大，傾向於使用 switch 而不是 if-else。這通常歸結到代碼的易讀性。這種觀點認為，如果條件較少時，if-else 容易閱讀，而條件較多時 switch 更容易閱讀。(當條件體增加時，if-else 性能負擔增加的程度比 switch 更多)

一般來說，if-else適 用於判斷兩個離散的值或者判斷幾個不同的值域。如果判斷多於兩個離散值，switch 表達式將是更理想的選擇。

### Optimizing if-else
> if-else 優化

優化 if-else 的目標是最小化找到正確分支之前所判斷條件體的數量。最簡單的優化方法是將最常見的條件體放在首位。(把最可能出現的條件放在首位)
```
if (value < 5) {
  //do something
} else if (value > 5 && value < 10) {
  //do something
} else {
  //do something
}
```

條件應始終從最可能的順序排序，以確保最快的執行時間。

另一種是使用二分搜索法將值域分成了一系列區間，然後逐步縮小範圍
```
if (value < 6) {
	if (value < 3) {
		if (value == 0) {
			return result0;
		} else if (value == 1) {
			return result1;
		} else {
			return result2;
		}
	} else {
		if (value == 3) {
			return result3;
		} else if (value == 4) {
			return result4;
		} else {
			return result5;
		}
	}
} else {
	if (value < 8) {
		if (value == 6) {
			return result6;
		} else {
			return result7;
		}
	} else {
		if (value == 8) {
			return result8;
		} else if (value == 9) {
			return result9;
		} else {
			return result10;
		}
	}
}
```

上面的寫法是每次抵達正確分支時最多通過四個條件判斷。
(不過這個情況用 switch 更適合吧)

### Lookup Tables
> 查表法

當有大量離散值需要測試時，if-else 和 switch 都比使用查表法要慢得多。在 JS 中查表法可使用陣列或者普通對象(regular objects)實現，查表法訪問數據比 if-else 或者 switch 更快，特別當條件體的數目很大時，也有助於保持代碼的可讀性，效果更顯著。

```
// define the array of results

var results = [result0, result1, result2, result3, result4, result5, result6, result7, result8, result9, result10];

// return the correct result
return results[value];
```

使用查表法時，必須完全消除所有條件判斷。操作轉換成一個陣列項目查詢(或者一個對象成員查詢)。使用查表法的一個主要優點是：由於沒有條件判斷，當候選值數量增加時，很少(幾乎沒有)增加額外的性能開銷。

### Recursion and Call Stack Limits
> 遞迴和調用棧限制

```
function factorial(n) {
	if (n == 0) {
		return 1;
	} else {
		return n * factorial(n-1);
	}
}

factorial(5);  // 120
```

遞迴函數導致的性能問題是，一個錯誤定義或者缺少終結條件可能導致長時間運行，凍結用戶界面。此外，遞歸函數還會遇到瀏覽器調用棧大小的限制。

JS 引擎所支持的遞歸數量與 JS 調用棧大小直接相關(只有 IE 例外，它的調用棧與可用系統內存相關)。其他瀏覽器有固定的調用棧限制。大多數現代瀏覽器的調用棧尺寸比老式瀏覽器要大(例如 Safari 2 調用棧尺寸是100)
而調用棧大小限制了遞迴演算法在 JS 中的應用；棧溢位錯誤會導致其他程式碼中斷執行。

```
function foo() {
	return foo();
}

foo();  // Uncaught RangeError: Maximum call stack size exceeded
```

關於調用棧溢出錯誤，在某些瀏覽器中，他們的確是 JS 的錯誤，可以用一個 `try-catch` 表達式捕獲：
```
function foo() {
	try {
		return foo();
	} catch (ex) {
		alert('Too much recursion!');
	}
}

foo();
```

異常類型因瀏覽器而不同:
- Firefox: InternalError
- Safari 和 Chrome: RangeError
- IE: 拋出一般性的錯誤類型
- Opera 不拋出錯誤: 它終止 JS 的引擎

函式呼叫時會在記憶體形成一個 call frame ("呼叫記錄"，又稱"呼叫幀")，儲存呼叫位置和內部變數等資訊。如果在函式 A 的內部呼叫函式 B，那麼在 A 的呼叫幀上方，還會形成一個 B 的呼叫幀。等到 B 執行結束，將結果返回到 A，B 的呼叫幀才會消失。如果函式 B 內部還呼叫函式 C，那就還有一個 C 的呼叫幀，以此類推。所有的呼叫幀，就形成一個 call stack ("呼叫棧")。

遞迴非常耗費記憶體，因為需要同時儲存成千上百個呼叫幀，很容易發生"棧溢位"錯誤(stack overflow)。現在 ES6 遞迴可以使用尾遞迴。對於尾遞迴來說，由於只存在一個呼叫幀，所以永遠不會發生"棧溢位"錯誤，相對節省效能。
```
// 用尾遞迴改寫 factorial()
function factorial(n, total = 1) {
	if (n === 1) return total;
	return factorial(n - 1, n * total);
}

factorial(5) // 120
```

- 原本的 factorial() 寫法是: 計算 n 的階乘，最多需要保存 n 個調用記錄，複雜度 O(n)。
- 改寫後的是: 只保留一個調用記錄，複雜度 O(1)。

尾遞歸的實現，需要改寫遞歸函數，確保最後一步只調用自身。做到這一點的方法，就是把所有用到的內部變量改寫成函數的參數。[ES6 尾調用優化](https://goo.gl/QZKG4x)

### Memoization (Tabulation)
> 製表

另一個提升性能優化的方式是使用 Memoization。代碼所做的事情越少，它的運行速度就越快。根據這些原則，避免重複工作也很有意義。多次執行相同的任務也在浪費時間。通過製表來緩存先前計算結果為後續計算所重複使用，避免了重複工作。這使得製表成為遞歸算法中有用的技術。

```
function memfactorial(n) {
	if (!memfactorial.cache) {
		memfactorial.cache = {
			"0": 1,
			"1": 1
		};
	}
	if (!memfactorial.cache.hasOwnProperty(n)) {
		memfactorial.cache[n] = n * memfactorial (n-1);
	}

	return memfactorial.cache[n];
}

memfactorial(0) // 0
memfactorial(1) // 1
memfactorial(5) // 120

var fact6 = memfactorial(6);
var fact5 = memfactorial(5);
var fact4 = memfactorial(4);
```

上面使用製表技術的階乘函數的關鍵是建立一個緩存對象。此對象位於函數內部，並預置了兩個最簡單的階乘：0 和 1。在計算階乘之前，首先檢查緩存中是否已經存在相應的計算結果。沒有對應的緩衝值說明這是第一次進行此數值的計算，計算完成之後結果被存入緩存之中，以備今後使用。此函數與原始版本的 factorial() 函數用法相同。

不過通用製表要注意的是，這種通用製表函數與人工更新算法相比優化較少，因為 memoize() 函數緩存特定參數的函數調用結果。當代碼以同一個參數多次調用外殼函數時才能節約時間（如果外殼函數內部還存在遞歸，那麼內部的遞歸就不能享用這些中間運算結果了)。因此，當一個通用製表函數存在顯著性能問題時，最好在這些函數中人工實現製表法。

[Functional Memoization in JS](https://goo.gl/59MQym)

---

<!-- http://jstherightway.org/zh-tw/ -->
<!-- https://blog.csdn.net/situdesign/article/details/5631527 -->
<!-- https://blog.csdn.net/situdesign/article/details/5631524 -->