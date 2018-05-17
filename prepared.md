# CH5 Array and Distionary
## Outlines
- 5-43 使用 Object 的直接實例構造輕量級的字典
- 5-44 使用 null 原型以防止原型汙染
- 5-45 使用 hasOwnProperty 方法以避免原型汙染
- 5-46 使用 array 而不要使用字典來儲存有序集合
- 5-47 絕對不要在 Object.prototype 中增加可枚舉的屬性
- 5-48 避免在枚舉期間修改對象
- 5-49 數組迭代要優先使用 for 循環而不是 for...in 循環
- 5-50 迭代方法優於循環
- 5-51 在類數組對象上複用通用的數組方法
- 5-52 數組字面量優於數組構造函數

### 5-43 使用 Object 的直接實例構造輕量級的字典

```
function NaiveDict() {}
NaiveDict.prototype.count = function(){
  var i=0;
  for(var name in this) {  // counts every property
    i++;
  }
  return i;
};
NaiveDict.prototype.toString = function(){
  return '[Object NaiveDict]';
};
var dict = new NaiveDict();
dict.alice = 24;
dict.bob = 34;
dict.chris = 62;
dict.count(); //5
```
上面實例化一個dict對象，並添加了私有屬性(alice, bob, chris)並繼承了原型對象的方法(toString,count)，所以當調用count方法時，這個時候的for...in循環，不僅枚舉了了私有屬性也枚舉了原型對象的方法屬性(alice,bob,chris,toString,count)。
再看一個使用 array 的例子:
```
var dict = new Array();
dict.alice = 24;
dict.bob = 34;
dict.chris = 62;
dict.alice; //24

Array.prototype.first = function() {
  return this[0];
};
Array.prototype.last = function() {
  return this[this.length-1];
};
var names=[];
for(var name in dict) {
  names.push(name);
}
names; //["alice", "bob", "chris","first","last"]
```
看起來好像可以 work，當我們給數組的原型對象添加一些原型方法，這時候再枚舉上面的 dict 對象的屬性時，錯誤就出現了。

---
應該僅僅將Object的直接實例作為字典，而不是子類，或其它對象。
上面 array 的例子可以改成:
```
var dict = {};
dict.alice = 24;
dict.bob = 34;
dict.chris = 62;
dict.alice;  // 24
var names = [];
for(var name in dict) {
  names.push(name);
}
names; //["alice", "bob", "chris"]
```
雖然這樣還是無法避免對於 `Object.prototype` 對象的修改，但可以將風險僅僅局限在 `Object.prototype`(如果有人增加屬性到 `Object.prototype` 對象中，會對 `for...in` 循環造成影響)。相比之下，增加屬性到 `Array.prototype` 中是合理的。例如給不支持 array 標準方法的環境中將這些方法增加到`Array.prototype` 中。

---

### 5-44 使用 null 原型以防止原型汙染
防止原型汙染的最簡單方法之一就是一開始就不使用原型。
嘗試定義一個構造函數的原型屬性為 null 或 undefined，實例化仍然得到的是 Object 的實例。
```
function C(){}
C.prototype = null;
var o = new C();
Object.getPrototypeOf(o) === null;  // false
Object.getPrototypeOf(o) === Object.prototype;  // true
```
ES5 首先提供了標準方法來創建一個沒有原型的對象: `Object.create()`
```
var x = Object.create(null);
Object.getPrototypeOf(x) === null;  // true
```
如此一來，原型汙染就不太容易影響這個對象的行為。

### 5-45 使用 hasOwnProperty 方法以避免原型汙染

前面討論了屬性的枚舉，但都沒有徹底地解決屬性查找中原型汙染的問題。看下面關於字典的一些操作，js 的 object 操作總是以繼承的方式工作的。即使是一個空的對象字面量也是繼承了 `Object.protoype` 屬性。
```
var dict = {};
dict.alice = 24;
dict.bob = 34;
dict.chris = 62;
"alice" in dict;  // true, membership test
dict.alice;       // retrieval
dict.alice = 22;  // update
"toString" in dict;  // true
"valueOf" in dict;  // true

// empty object
var dict2 = {};
"alice" in dict2;  // false
"chris" in dict2;  // false
"toString" in dict2;  // true
"valueOf" in dict2;  // true
```
`Object.prototype` 提供了 `hasOwnProperty` 方法。當測試字典條目時，它可以避免原型汙染，這正好可以解決之前的問題
```
dict2.hasOwnProperty("alice");  // false
dict2.hasOwnProperty("chris");  // false
dict2.hasOwnProperty("toString");  // false

dict2.hasOwnProperty("alice") ? dict.alice : undefined;
```
要注意我們是使用 dict2 對象的 `hasOwnProperty` 方法，但其實它自身並沒有這個方法，而是繼承自`Object.prototype` 對象。如果 dict2 字典對象有一個同為 "hasOwnProperty" 名稱的屬性，那麽原型中的`hasOwnProperty`方法不會被訪問到。這裏**會優先讀取自身包含的屬性，找不到才會從原型鏈中查找**。
```
// 假設有 "hasOwnProperty" 名稱的屬性
dict2.hasOwnProperty = 10;
dict2.hasOwnProperty("alice"); // Uncaught TypeError: dict2.hasOwnProperty is not a function
```
為了避免上述問題，改成直接使用 `Object.prototype` 中的 `hasOwnProperty`方法，然後使用函數的`call` 方法，把函數的接收者綁定到字典對象。
```
var hasOwn = Object.prototype.hasOwnProperty;
// var hasOwn = {}.hasOwnProperty;

hasOwn.call(dict2, "alice");  // true
```
這樣就可以安全地調用 `Object.prototype.hasOwnProperty` 方法來對字典對象的屬性名進行檢測，不管其接收者的 `hasOwnProperty` 方法是否被覆蓋，該方法都能 work。
```
dict2.alice = 14;
hasOwn.call(dict2, "hasOwnProperty");  // false
hasOwn.call(dict2, "alice");           // true
dict2.hasOwnProperty = 10;
hasOwn.call(dict2, "hasOwnProperty");  // true
hasOwn.call(dict2, "alice");           // true
```
另外為了避免所有查找屬性的地方都要插入上面的檢測代碼，可以把該模式抽象為 Dict 的方法(Dict 的構造函數中)。

Dict 構造函數封裝了所有在單一數據類型定義中編寫健壯字典的技術細節。這樣比使用 js 默認的對象語法更健壯，而且也同樣方便使用。
```
function Dict(elements) {
  // allow an optional initial table
  this.elements = elements || {};  // simple Object
}
Dict.prototype.has = function(key) {
  // own property only
  return {}.hasOwnProperty.call(this.elements,key);
};
Dict.prototype.get = function(key) {
  // own property only
  return this.has(key) ? this.elements[key] : undefined;
};
Dict.prototype.set = function(key, val) {
  this.elements[key]=val;
};
Dict.prototype.remove = function(key) {
  delete this.elements[key];
};

var dict = new Dict({
  bob: 23,
  chris: 40
});
dict.has('alice');  // true
dict.has('bob');  // true
dict.has('toString');  // false
```

要注意在一些特殊 js 環境中，特殊的屬性名 `__proto__` 可能導致自身的汙染問題。在某些環境中, `__proto__` 只是簡單地繼承自 `Object.prototype`，因此空對象是真正的空對象。
```
var empty = Object.create(null);
'__proto__' in empty;  // false/true 在其它環境下可能會是 true
var hasOwn = {}.hasOwnProperty;
hasOwn.call(empty,'__proto__');  // false/true 有些環境因為存在一個實例屬性 __proto__ 而永久汙染所有的對象

// 也有可能的情況是，在不同環境中， __proto__ 的值無法確定
var dict = new Dict();
dict.has('__proto__');  //無法確定
```

為了達到代碼的可移植性和安全性，只能對 `__proto__` 關鍵字增加一些操作。
```
function Dict(elements){
	// allow an optional initial table
	this.elements = elements || {};  // simple object
	this.hasSpecialProto = false;    // has "__proto__" key?
	this.specialProto = undefined;   // "__proto__" element
}
Dict.prototype.has = function(key){
	if(key === '__proto__') {
		return this.hasSpecialProto;
	}
	// own property only
	return {}.hasOwnProperty.call(this.elements, key);
};
Dict.prototype.get = function(key){
	if(key === '__proto__') {
		return this.specialProto;
	}
	// own property only
	return this.has(key) ? this.elements[key] : undefined;
};
Dict.prototype.set = function(key, val) {
	if(key === '__proto__') {
		this.hasSpecialProto = true;
		this.specialProto = val;
	} else {
		this.elements[key] = val;
	}
};
Dict.prototype.remove = function(key) {
	if(key === '__proto__') {
		this.hasSpecialProto = false;
		this.specialProto = undefined;
	} else {
		delete this.elements[key];
	}
};

var dict = new Dict();
dict.has('__proto__');  // false
```
不管環境是否處理 `__proto__` 屬性，以上代碼都能 work，因為它避免了到處處理該名稱的屬性。

---

### 5-46 使用 array 而不要使用字典來儲存有序集合
js 對象是一個無序屬性集合。`for...in`循環，先輸出哪個屬性都有可能。獲取和設置不同的屬性與順序無關，都會以大致相同的效率產生相同的結果。ES標準並未規定屬性存儲的任何特定順序，甚至於枚舉對象也未涉及。`for...in` 循環會挑選一定的順序來枚舉對象的屬性，標準允許 js 引擎自由選擇一個順序，它們的選擇會微妙地改變程序行為。
```
function report(highScores){
	var res = '', i = 1;
	for(var name in highScores) {
		res += i + '. ' + highScores[name].name + ':' + highScores[name].points + '\n';
		i++;
	}
	return res;
}
report([{name:'Hank',points:1110100},
	{name:'Steve',points:1064500},
	{name:'Billy',points:1050200}]);
// output
1. Hank:1110100
2. Steve:1064500
3. Billy:1050200
```
所以如果對於數據結構中的條目順序有強烈依賴，那麽就優先考慮 array 而不是字典對象。用 for 來循環，可以保證在所有環境順序都是一致正確的。(而在執行 `for...in` 循環時要小心，確保操作的行為與順序無關。)

---

另外，浮點型算術運算的四捨五入會導致對計算順序依賴。當組合未定義順序的枚舉時，可能會導致循環不可預知。
```
var ratings = {
	'Good Will Hunting': 0.8,
	'Mystic River': 0.7,
	'21': 0.6,
	'Doubt': 0.9
};

var total = 0, count = 0;
for(var key in ratings) {
	total += ratings[key];
	count++;
}
total /= count;
total; // 0.7499999999999999(chrome)
```
在流行的 js 環境實際上使用不同的順序執行這個循環。一些環境按照下面的順序來枚舉對象的 key，得到下面這個值
```
(0.8+0.7+0.6+0.9)/4 //0.75
```
有些環境總是先枚舉潛在的數組索引，然後才是其他key。電影21是可以作為數組的索引的整數值，它首先被枚舉，得到下面的結果。
```
(0.6+0.8+0.7+0.9)/4 //0.7499999999999999
// chrome 就是先枚舉潛在的數組索引。
```
對於浮點數的計算，可以把浮點數轉化為整數，然後再轉化回浮點數。**整數的計算順序可以是任意順序**(因此對象屬性值的列舉順序並不重要)。
```
(8+7+6+9)/4/10 //0.75
(6+8+7+9)/4/10 //0.75
```

---

### 5-47 絕對不要在 Object.prototype 中增加可枚舉的屬性
`for...in` 它便利好用，但又容易被原型污染。它最常見的用法是枚舉字典中的元素。所以想要用 `for...in` 的話，那就不要在共享的 `Object.prototype` 中增加可枚舉的属性。
```
Object.prototype.allKeys = function() {
  var result = [];
  for(var key in this) {
    result.push(key);
  }
  return result;
};

({a:1,b:2,c:3}).allKeys();  // ["a", "b", "c", "allKeys"]
```
這樣的方法也汙染了其自身。

可以改進 allKeys 方法忽略掉 `Object.prototype` 中的属性。將 allKeys 定義為獨立的函數而不是方法。
```
function allKeys(obj) {
  var result = [];
  for(var key in obj) {
    result.push(key);
  }
  return result;
}
```
也可以用 ES5 提供的機制 `Object.defineProperty` 來將 allKeys 設置為**不可枚舉**
```
Object.defineProperty(Object.prototype, "allKeys", {
  value: function( ) {
    var result = [];
    for(var key in this) {
      result.push(key);
    }
    return result;
  },
  writable: true,
  enumerable: false,
  configurable: true
});

({a: 1, b: 2, c: 3}).allKeys();  // ["a", "b", "c"]
```

---

### 5-48 避免在枚舉期間修改對象
以一個社交網路註冊列表為例子：
```
function Member(name) {
	this.name = name;
	this.friends = [];
}
var a = new Member("Alice"),
    b = new Member("Bob"),
    c = new Member("Carol"),
    d = new Member("Dieter"),
    e = new Member("Eli"),
    f = new Member("Fatima");
a.friends.push(b);
b.friends.push(c);
c.friends.push(e);
d.friends.push(b);
e.friends.push(d, f);
```
通常通過工作集(work-set)來實現。工作集以單個根節點開始，然後添加發現的節點，移除訪問過的節點。

![social](assets/images/social.png)

使用 `for...in` 來遍歷很方便：
```
Member.prototype.inNetwork =function(other) {
	var visited = {};
	var workset = {};
	workset[this.name] = this;
	for(var name in workset) {
		var member = workset[name];
		delete workset[name];  // modified while enumerating
		if(name in visited) {  // don't revisit members
			continue;
		}
		visited[name] = member;
		if(member === other) {
			return true;
		}
		member.friends.forEach(function(friend) {
			workset[friend.name] = friend;
		});
	}
	return false;
}
```
然而在有些環境下不會 work
```
a.inNetwork(f);  // false
```
`for...in` 循環並沒有要求枚舉對象的修改與當前保持一致。事實上，ES對並發修改在不同 js 環境下的行為的規範留有餘地。標準規定：
> 如果被枚舉的對象在枚舉期間添加了新的屬性，那麽在枚舉期間並不能保證新添加的屬性能被訪問。

上面的實際後果：如果我們修改了被枚舉的對象，則不能保證`for...in` 循環的行為是可預見的。

另一種遍歷方式：自己管理循環控制。當使用循環時，應該使用自己的字典抽象以避免原型汙染。我們可以將字典放置在 WorkSet 類中來追蹤當前集合中的元素數量。
```
function WorkSet() {
	this.entries = new Dict();
	this.count = 0;
}
WorkSet.prototype.isEmpty = function() {
	return this.count === 0;
};
WorkSet.prototype.add = function(key,val) {
	if(this.entries.has(key)) {
		return;
	}
	this.entries.set(key, val);
	this.count++;
};
WorkSet.prototype.get = function(key) {
	return this.entries.get(key);
};
WorkSet.prototype.remove = function(key) {
	if(!this.entries.has(key)) {
		return;
	}
	this.entries.remove(key);
	this.count--;
};
```
為了提取集合的任意一個元素，我們需要給 Dict 類添加一個新方法：
```
Dict.prototype.pick = function() {
	for(var key in this.elements) {
		if(this.has(key)) {
			return key;
		}
	}
	throw new Error("empty dictionary");
};
WorkSet.prototype.pick = function() {
	return this.entries.pick();
};
```
這次我們可以使用簡單的 while loop 來實現 inNetwork 方法。每次選擇任意一個元素並從工作集中刪除：
```
Member.prototype.inNetwork = function(other) {
	var visited = {};
	var workset = new WorkSet();
	workset.add(this.name,this);
	while(!workset.isEmpty()) {
		var name = workset.pick();
		var member = workset.get(name);
		workset.remove(name);
		if(name in visited) {
			continue;
		}
		visited[name] = member;
		if(member === other) {
			return true;
		}
		member.friends.forEach(function(friend) {
			workset.add(friend.name,friend);
		})
	}
	return false;
};
```
其中 pick 方法是一個不確定性的例子，指的是一個操作並不能保證使用語言的語義產生一個單一的可預見的結果。這個不確定性是因為 `for...in` 循環可能在不同的 js 環境中選擇不同的枚舉順序。使用不確定性可能會使你的程序引入一個不可預測的元素。測試可能在某個平臺通過，某些平臺不通過，或同一平臺不同時候，結果也可能不同。

不確定性的來源是難以避免的，基於這些原因，考慮使用一個確定的工作集算法替代方案。即工作列表算法(work-list)。將工作條目存儲到數組中而不是集合中，則 inNetwork 方法，將總是以完全相同的順序遍歷圖。
```
Member.prototype.inNetwork = function(other) {
	var visited = {};
	var worklist = [this];
	while(worklist.length > 0) {
		var member = worklist.pop();
		if(member.name in visited) {  // don't revisit
			continue;
		}
		visited[memeber.name] = member;
		if(member === other) {        // found?
			return true;
		}
		member.friends.forEach(function(friend) {
			worklist.push(friend);     // add to work-list
		})
	}
	return false;
};
```
這一版本的 inNetwork 方法會確定性地添加和刪除工作條目。無論發現什麽路徑，該方法對於連接的成員總是返回 true，所以最終結果是一樣的。

---

### 5-49 數組迭代要優先使用 for 循環而不是 for...in 循環
使用 for...in loop
```
var scores = [98,74,85,77,93,100,89];
var total = 0;
for(var score in scores) {
	total += score;
}
var mean = total / scores.length;
mean;  // 17636.571428571428 (預期要是 88 才對)
```
因為 `for...in` 循環會枚舉所有 key, 包括原型中的。也就是說上面的代碼實際應該是 (0+1+2+...+6)/7=21，但也不對。這裡的 key 即使是數组的索引，對象屬性也始终是字符串。因此，"+=" 操作符將執行字符串的連接操作。结果就是 total 的值是"00123456"。mean 最终结果是 17636.571428571428。

使用 for loop
```
var scores = [98,74,85,77,93,100,89];
var total = 0;
for(var i = 0, n = scores.length; i < n; i++) {
	total += scores[i];
}
var mean = total / scores.length;
mean;  // 88
```
這個方法確保你需要整數索引和數组元素時就能獲取到它們，並且絕不會混淆它們或引發字符串的强制轉换。此外，它還可以確保正確的迭代數组，並且不會意外地包括存儲在數组對象或其原型鏈中的非整數属性。
上面的循環中對於變數 n 的使用，這可以在循環的时候，不用每次都獲取一次數组的長度，因為即使是優化的 js 編譯器可能有時也很難保證避免重新計算 scores.length 是安全的。更重要的是，如此给閱讀該代碼的程序員傳遞了一個信息：循環的终止修件是簡單且確定的。

---

### 5-50 迭代方法優於循環
複製和貼上樣板代碼會重複錯誤，使程序很難更改。更糟糕的是，重複的代碼使人閱讀代碼時太容易忽略一個模式實例與另一個的細微差別。

js 中的 for 循環在進行一些細微變化時，可以引入不同的行為。編程的時候對於邊界條件的判斷往往會導致一些簡單的錯誤。下面的一些 for 循環的細微變化導致邊界條件的變化：
```
for(var i = 0; i <= n; i++) {...}
// extra end iteration (包括最後的)
for(var i = 1; i < n; i++) {...}
// missing first iteration (忽略第一次的)
for(var i = n; i >= 0; i--) {...}
// extra start iteration (包括第一次的)
for(var i = n - 1; i > 0; i++) {...}
// missing last iteration (忽略最後的)
```
以上都是對終止條件的一個設置。這裏可以有很多的方式，可以使終止條件發生錯誤。
js 的閉包是一種為這些模式建立叠代抽象方便的、富有表現力的手法，可以避免重復代碼。ES5 為最常用的一些模式提供了便利的方法。 `Array.prototype.forEach` 是其中最簡單的一個：
```
for(var i = 0, n = players.length; i < n; i++) {
  players[i].score++;
}
// 可以將上面改寫為：
players.forEach(function(p) {
  p.score++;
});
```
上面的代碼把循環的方式進行抽象，把要執行的具體代碼通過函數傳遞閉包，完成對數組元素的操作及訪問。這裡消除了終止條件和任何數組索引。其他還有：
`Array.prototype.map`
完成對數組的每個元素進行一些操作後建立一個新的數組。
```
// 使用循環
var trimmed = [];
for(var i = 0, n = input.length; i < n; i++) {
	trimmed.push(input[i].trim());
}
// 使用 forEach
var trimmed = [];
input.forEach(function(s) {
	trimmed.push(s.trim());
});
// 使用 map
var trimmed = input.map(function(s) {
	return s.trim();
});
```
`Array.prototype.filter`
計算一個數組建立一個新數組，該數組只包含有數組的一些元素
filter 方法需要一個謂詞函數，如果元素應該存在於新數組中則返回真值，如果元素應該被剔除則返回假值。例如，可以從價格表中提取出一個特定價格區間的列表。
```
listings.filter(function(listing) {
	return listing.price >= min && listing.price <= max;
});
```
以上的 `forEach`, `map`, `filter` 三個方法都是 ES5 中，數組的默認方法。下面實現一個自己的叠代抽象。
例：提取出滿足謂詞函數的數組的前幾個元素，直到不滿足的元素終止，不管後面是否有元素滿足條件。
```
function takeWhile(a, pred) {
	var result= [];
	for(var i = 0, n = a.length; i < n; i++) {
		if(!pred(a[i], i)) {
			break;
		}
		result[i] = a[i];
	}
	return result;
}
var prefix = takeWhile([1,2,3,4,26,18,9], function(n) {
	return n < 10;
});
prefix;  // [1, 2, 3, 4]
```
上面 pred 函數有兩個參數，而下面的回調我們只傳入了一個參數。對第二個參數沒有進行處理，這裡是無所謂的。

猴子補丁:把 takeWhile 函數添加到 `Array.prototype` 中使其成為一個方法。
```
Array.prototype.takeWhile = function(pred) {
	var result=[];
	for(var i = 0, n = this.length; i < n; i++) {
		if(!pred(this[i],i)) {
	break;
		}
		result[i] = this[i];
	}
	return result;
};
var prefix = [1,2,3,4,26,18,9].takeWhile(function(n){
	return n <10;
});
prefix;  //[1, 2, 3, 4]
```

循環只有一點優於疊代函數，那就是前者有控制流操作，如 break 和 continue。舉例來說，使用forEach 方法來實現 takeWhile 函數將是一個尷尬的嘗試。
```
function takeWhile(a, pred) {
	var result = [];
	a.forEach(function(x, i) {
		if(!pred(x)) {
			//？ 如何終止循環的當次執行？
		}
		result[i] = x;
	});
	return result;
}
```
或許可以使用一個內部異常來提前終止循環，如下，但效率不好，因為這樣的處理方法，把原本簡單的處理變得更加複雜了，不可取。
```
function takeWhile(a, pred) {
	var result = [];
	var earlyExit = {};
	try {
		a.forEach(function(x, i) {
			if(!pred(x)) {
				throw earlyExit;
			}
			result[i] = x;
		});
	} catch(e) {
		if(e !== earlyExit) {
			throw e;
		}
	}
	return result;
}
```
利用 ES5 的 `some`, `every` 可以用於提前終止循環：
```
// some 方法返回一個布林值表示其回調對數組的任何一個元素是否返回了一個真值
// 所有元素，對於傳入的函數的判斷，有一真則為真。全為假才為假。相當於所有元素執行函數後取或。
[1,10,100].some(function(x) {return x > 5;});  // true
[1,10,100].some(function(x) {return x < 0;});  // false

// every 方法返回一個布林值表示其回調函數是否對所有元素返回一個真值
// 所有元素，對於傳入函數的判斷，有一假則為假。全為真才為真。相當於所有元素執行函數後取且。
[1,10,100].every(function(x) {return x > 5;});  // false
[1,10,100].every(function(x) {return x > 0;});  // true
```
這兩個方法都是短路循環(short-circuiting)。只要產生的結果可以決定最後結果後，就不再執行後面的循環。即 some 一旦產生一個真值，則立即返回。every 一旦產生一個假值，也立即返回。
利用它們的特點來改寫 takeWhile 函數：
```
function takeWhile(a, pred) {
	var result = [];
	a.every(function(x, i) {
		if(!pred(x)) {
			return false;  // break
		}
		result[i] = x;
		return true;
	});
	return result;
}
```

---

### 5-51 在類數組對象上複用通用的數組方法
關於 `Array.prototype` 中的標準方法被設計成其他對象可複用的方法，即使這些對象並没有繼承 Array。

函數 arguments 對象。它是一個類數组對象，並不是一個標準的數组，所以無法使用數组原型中的方法，因此也無法使用 `arguments.forEach` 這樣的形式来遍歷每一個参數。這裡我們必須使用 call 方法来對其使用 forEach 方法。
```
function highlight() {
  [].forEach.call(arguments, function(widget) {
		widget.setBackground("yellow");
  });
}
```
forEach 是一個函數對象，所以它繼承了 `Function.prototype` 對象中的 call 方法。這裡就可以使用一個指定的對象作為函數内部 this 的綁定對象来調用它，並緊隨任意數量的参数。

另外，在 Web 平台，DOM 的 NodeList 類是另一个類數组對象。類似的 `document.getElementsByTagName` 會返回一個 NodeList 類數组對象。這個對象也没有繼承自 `Array.prototype`。

如何構建一個類數组對象？
- 具有一個範圍在 0 到 2^32-1 的整型 length 属性
- length 屬性大於該對象的最大索引。索引是一個範圍在 0 到 2^32-1 的整数，它的字符串表示是該對象中的一個 key

實現上面這兩點，就可以使一個對象與 `Array.prototype` 中任一方法兼容。一個簡單的對象字面量也可以用來創建一個類數组對象。
```
var arrayLike = {0:'a',1:'b',2:'c',length:3};
var result = Array.prototype.map.call(arrayLike, function(s){
	return s.toUpperCase();
});
result;  // ['A','B','C']
```

字符串也可以表現為數组，因為它們是可索引，並且其長度也可以通過 length 属性獲取。因此，`Array.protoype` 中的方法操作字符串時並不會修改原始字符串。
```
var result = Array.prototype.map.call('abc', function(s) {return s.toUpperCase();});
result;  // ['A','B','C']
```

這些模擬數组的所有行為，歸功於數组行為的兩方面：
- 將 length 屬性值設為小於 n 的值會自動地删除索引值大於或等於 n 的所有屬性
- 增加一個索引值為 n (大於或等於 length 屬性值)的屬性會自動地設置 length 屬性為 n+1

其中第 2 條規則不好實現，因靈需要監控索引屬性的增加以自動地更新 length 属性。

對於 `Array.prototype` 中的方法，這兩條都不是必須的，因為在增加或删除索引屬性的时候它們都會强制地更新 length 屬性。
Array 方法中只有一個不是通用的，即數组連接方法 `concat`。該方法可以由任意的類數组接收者調用，但它會檢查其参數 [[Class]] 屬性。如果参數是一個真實的數组，那麼 concat 會將該數组的内容連接起来作為结果；否則，参數將以一個單一的元素来連接。例如，不能簡單地連接一個以 arguments 對象作為内容的數组。
```
function namesColumn() {
  return ['Names'].concat(arguments);
}
namesColumn('Alice','Bob','Chris');  // ["Names", { 0: "Alice", 1: "Bob", 2: "Chris" }]
```
為了使 concat 方法將一個類數组對象視為真實數组，需要把類數组轉換為真正的數组。使用 slice 對類數组對象進行轉換：
```
function namesColumn() {
  return ['Names'].concat([].slice.call(arguments));
}
namesColumn('Alice','Bob','Chris');  // ["Names", "Alice", "Bob", "Chris"]
```

### 5-52 數組字面量優於數組構造函數
js 的優雅很大程度上要歸功於程序中常見的構造塊（Object,Function 及 Array）的簡明的字面量語法。字面量是一種表示數组的優雅方法：
`var a = [1,2,3,5,7,8];`
也可以使用構造函數来替代
`var a = new Array(1,2,3,5,7,8);`
由於 Array 構造函數存在一些微妙的問题。當你使用時，確保别人没有重新包装過 Array 變量：
```
function f(Array) {
	return new Array(1,2,3,4,5);
}
f(String);  // new Sring(1)
```
還必須確保没有修改過全局的 Array 變量：
```
Array = String;
new Array(1,2,3,4,5);  // new Sring(1)
```
而且還得擔心一種特殊的情况。如果使用單個數字参數來調用 Array 構造函數，效果完全不同。它試圖創建一個長度為给定参數的空數组。這意味著 `['hello']` 和 `new Array('hello')`的行為相同，但 `[17]` 和 `new Array(17)` 的行為完全不同。但字面量更清晰，更優雅，更不易出错，更規範，更一致的語義。