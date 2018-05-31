# CH6 Library and API Design
## Outlines
- 6-53 保持一致的約定
- 6-54 將 `undefined` 看做**沒有值**
- 6-55 接收關鍵字參數的選項對象
- 6-56 避免不必要的狀態
- 6-57 使用结構類型設計靈活的接口
- 6-58 區分數組對象和類數組對象
- 6-59 避免過度的強制轉換
- 6-60 支持方法鏈

### 6-53 保持一致的約定

1. 函數名稱盡量能夠表達我們的意思
就像在 css 中我們在描述矩形的四條邊的參數時，總是以上右下左的順序。因為這個和margin, padding 等屬性約定相同順序。
而對於 api 使用者來說，你所使用的命名和函數名是最能產生普遍影響的決策。這些約定很重要具有巨大的影響力。它建立了基本的辭彙和使用它們的應用程式的慣用法。library 的使用者必須學會閱讀和使用這些。一致的約定可以讓人更容易理解和記憶，除非有特殊的要求，要不然就不要修改這些約定。

2. 參數順序的約定很重要
比如用戶界面庫通常具有一些接收多個測量值（寬，高）的函數。確保這些參數總是以相同的順序出現。選擇和其它常用庫的參數順序相同，可以方便用戶使用，如下第一個參數是寬度，第二個參數是高度。

```
function Rectangle(width, height) {
    this.width = width;
    this.height = height;
}

var rect = new Rectangle(320, 240);
console.log(rect); // Rectangle { width: 320, height: 240 }
```

3. 選項對象參數
如果 api 使用的是參數是選項對象(option objects)時，則裡面對應項的順序就沒有這麼重要了。重要的是每一項鍵值的命名，及後面參數的格式。

4. 詳盡的文檔
每個優秀的 library 都需要詳盡的文檔，而一個極優秀的 librabry 將文檔作為輔助。一旦你的用户用熟了，他們可以不再依賴於文檔而自由使用。一致的約定可以幫助用户推測一個方法的行為。

---

### 6-54 將 `undefined` 看做**沒有值**

- 我們應該判斷參數是否是 `undefined` 來決定是否使用默認值

!!! 真值測試並不是總是安全的
例如函數可能允許一個元素的寬度或高度為 0 (或其它可轉化為 false 的值)時，則不應該使用真值測試

```
function Rectangle(width, height) {
    this.width = width || 320;
    this.height = height || 240;
}

var rect = new Rectangle();
console.log(rect); // Rectangle { width: 320, height: 240 }

// ↓ this is wrong ↓
var rect1 = new Rectangle(0, 0);
console.log(rect1); // Rectangle { width: 320, height: 240 }
```

在允許 0、NaN 或空字符串為有效參數的地方，絕不要透過真值測試來實現參數默認值，
這個情況就應該使用 undefined 來檢測

```
function Rectangle(width, height) {
    this.width = width === undefined? 320 : width;
    this.height = height === undefined ? 240 : height;
}

var rect = new Rectangle();
console.log(rect); // Rectangle { width: 320, height: 240 }

var rect1 = new Rectangle(0, 0);
console.log(rect1); // Rectangle { width: 0, height: 0 }
```

---

### 6-55 接收關鍵字參數的選項對象

1. 參數蔓延
一個函數起初很簡單，然而隨著功能的擴展，該函數裡面帶的參數便會得越来越多。參數過多，記憶和理解起來就不太容易。

```
var alert = new Alert(100, 75, 300, 200,
	'Error', message,
	'blue', 'white', 'black',
	'error', true);
```

js 提供了一個簡單、輕量的慣用法：選項對象。選項對象在應對較大規模的函數參數時運作良好。一個選項參數就是一個通過其命名屬性來提供額外參數數據的參數。對象字面量的形式使得讀寫變得容易。

```
var alert = new Alert({
    x:100, y:75,
    width:300, height:200,
    title:'Error', message:message,
    titleColor:'blue', bgColor:'white', textColor:'black',
    icon:'error', modal:true
});
```

舉個例子：
定義一個接受參數的選項對象的函數

```
function Alert(obj) {
    this.level = obj.level;
    this.msg = obj.msg;
}
var warnMsg = new Alert({
    level: 0,
    msg: 'hello'
});
console.log(warnMsg); // Alert { level: 0, msg: 'hello' }
```

如果一些參數是必選的話，那就把他們都單獨拿出来，而參數的選項對象上的屬性不是必選的

```
function Alert1(level, msg, options) {
    this.level = level;
    this.msg = msg;
    for(var opt in options) {
        this[opt] = options[opt]
    }
}
var warnMsg1 = new Alert1(1, 'find error', {
    count: 8,
    theme: 'default'
});
console.log(warnMsg1); // Alert1 { level: 1, msg: 'find error', count: 8, theme: 'default' }
```

許多 library 提供的 extend 函數。比如該 extend 函數接收一個 target 對象和一個 source 對象，並將後者的屬性複製到前者中。借助 extend 函數，可以使用有用的抽象(對象的擴展或合併函數)，從選項對象中提取值的邏輯來簡化工作。

舉個例子:
使用 extend 函數擴展我們的參數對象

```
function extend(target, source) {
    if(source) {
        for(var p in source) {
            var val = source[p];
            if('undefined' !== typeof val) {
                target[p] = val;
            }
        }
    }
    return target;
}
```

升級原來的構造函數

```
function Alert2(level, msg, options) {
    var opt = extend({
        level: level,
        msg: msg
    });
    opt = extend(opt, options);
    extend(this, opt);
}
var ale2 = new Alert2(2, 'bug', {
    count: 1,
    theme: 'highlight'
});
console.log(ale2); // Alert2 { level: 2, msg: 'bug', count: 1, theme: 'highlight' }
```

使用選項對象使得 API 更具可讀性，更容易記憶。
所有通過選項對象提供的參數應當被視為可選的。

---

### 6-56 避免不必要的狀態

API 有時被歸為兩類：有狀態的和無狀態的。無狀態的 API 提供的函數或方法的行為只取決於輸入，而與程序的狀態改變無關。比如，字符串的方法是無狀態的。字符串的内容不能被修改，方法只取決於字符串的内容及傳遞给方法的參數。不管程序其他部分的情况，表達式 `"foo".toUpperCase()` 總是產生 "FOO"。相反，Date 對象的方法卻是有狀態的。對於相同的 Date 對象調用 toString 方法會產生不同的結果，這取決於 Date 的各種 set 方法是否已經將 Date 的屬性修改。

無狀態的 API
```
console.log('hello'.toUpperCase()); // HELLO
```

定義一個類名和一個有狀態的方法
```
function Rect(width, height) {
    this.name = name;
    this.age = age;
}
User.prototype.setAge = function(age) {
    this.age = age;
};
```

無狀態的方法，取決於給對象的 age
```
User.prototype.sayHello = function() {
    if(this.age > 60) {
        console.log('I am old.');
    }
    else {
        console.log('I am young.');
    }
};

var u1 = new User('dream', 20);
u1.sayHello(); // I am young.
u1.setAge(80);
u1.sayHello(); // I am old.
```

盡可能使用無狀態的 API。
如果 API 是有狀態的，標示出每個操作與哪些狀態有關聯。

---

### 6-57 使用结構類型設計靈活的接口

```
function Rectangle(width, length) {
    this.width = width;
    this.length = length;
}
Rectangle.prototype.getArea = function() {
    return this.width * this.length;
};

// 使用结構類型
function rectangle1(width, length) {
    var _width = width,
        _length = length;
    return {
        getArea: function() {
            return _width * _length;
        }
    }
}

var r1 = new Rectangle(10, 20);
console.log(r1.getArea()); // 200
var r2 = rectangle1(300, 50);
console.log(r2.getArea()); // 15000
```

結構類型(aka. 鴨子類型)：任何對像只要具有預期的結構就屬於該類型(看起来像隻鴨子，或叫聲像隻鴨子)。(這個類似於強類型面向對象語言裡說的接口，也就是面向接口編程。)
在 js 中這是一種優雅、輕量的編程模式，因為不需要編寫顯示的聲明。一個調用某個對象方法的函數能夠與任何實現了相同接口的對像一起工作。也要在 API 的文檔中列出對象接口的預期結構。這樣接口實現者便會知道哪些屬性和方法是必需的以及庫和應用程序期望的行為是什麼。

- 結構類型可以有利於單元測試，可以很容易去實現一個測試的數據結構。
- 結構類型也可以使代碼各部分解耦，代碼的依賴只是通過結構類型。
- 在實現代碼時，不用去管結構類型最終的實現細節，只要提供對應的方法及屬性，那麼程序就可以正常運行。

---

### 6-58 區分數組對象和類數組對象

1. 分離數組對象

```
function testing() {
    console.log(arguments.length); // 3

    // arguments 是類數組對象
    console.log(arguments);
    // { 0: 'a', 1: 'b', 2: 'c' }
    // Arguments(3) ["a", "b", "c", callee: ƒ, Symbol(Symbol.iterator): ƒ]

    // 判斷 arguments 的類型
    console.log(typeof arguments); // object

    // 判斷是否是數組
    console.log(Array.isArray(arguments)); // false

    // 判斷 arguments 是什麼類型的對象
    console.log(Object.prototype.toString.call(arguments));
    // [object Arguments]

    // instanceof
    console.log(arguments instanceof Array);  // false
}

testing('a', 'b', 'c');
```

這樣的區分和 js 的靈活的類數組對象的概念是有爭執的。任何對像都可被視為數組，只要它遵循正確的接口。(上一條使用結構類型設計靈活的接口中提到，靈活的結構只要一個數據符合相應的接口，就可以把它視為正確的數據。)而且也沒有明確的方法來測試一個對象是否滿足一個接口。也許可以嘗試把具有 length 屬性的對象視為數組，但這也會有錯誤出現，比如碰巧一個字典對象或 `arguments` 有 length 屬性呢？

2. 重載

重載兩種類型意味著必須有一種方法來區分兩種不同情況。如果出現了兩種情況的重疊區域，則無法對 API 進行重載。API 絕不應該重載與其他類型有重疊的類型。

3. Array.isArray函數(ES5)

這個函數測試一個值是否是數組，而不管原型繼承。
在ES標準中，該函數測試對象的內部 `[[Class]]` 屬性值是否是 Array。當需要測試一個對象是否是真數組，而不僅僅是類數組對象，Array.isArray 方法比 `instanceof` 操作符更好。

```
var a = {};
var b = [];
var c = 10;

console.log(Array.isArray(a)); // false
console.log(Array.isArray(b)); // true
console.log(Array.isArray(c)); // false
```

4. 類數組轉化為數組

```
var slice = [].slice;
var obj = { 0: 10, 1:'abc', length: 2 };
slice.call(obj);  // [10, 'abc'];
```

如果對API的傳入參數有特殊指定要求的，需要在文檔中註明，API 的重載也必須要註明不同情況的參數要求。

---

### 6-59 避免過度的強制轉換

1. 強制轉換

強制轉換有時可以帶來方便性，但也會帶來相關的麻煩，一些錯誤無法顯露出來，導致程序行為的不穩定和難以調試。而當強制轉換與重載的函數一起工作的時候，結果會更難理解。

```
function square(x) {
    // 這裡會進行强制的類型轉換
    return x * x;
}

console.log(square('3')); // 9

// 一種比較好的方式是我們在函數內部判斷參數是否是一個數字
function square1(x) {
    if('number' === typeof x) {
        return x * x;
    }
    throw new Error('請傳遞正確的參數類型!');
}
console.log(square1('3')); // Error: 請傳遞正確的參數類型!
```

2. 重載和強制轉換

強制轉換使參數類型信息丟失，導致結果和預期不同。所以在使用參數類型來作為重載依據時，應該避免強制轉換。通過對參數的類型進行強制要求，來實現 API 的設計，這樣代碼更謹慎。

3. 防禦性編程

以額外的檢查來抵禦潛在的錯誤。抵御所有的錯誤是不可能的。除 js 中提供的基本檢查工具外，可以通過編寫一些簡潔的檢查工具函數來輔助開發。
額外的代碼會影響程序的性能，也可以更早地捕獲錯誤。看具體情況來使用防禦性編程。

---

### 6-60 支持方法鏈

無狀態的 API 的部分能力是將複雜操作分解為更小的操作的靈活性。
有一個很好的例子是字符串的 `replace` 方法。由於結果本身也是字符串(返回的是一個新的字符串對象)，可以對前一個 `replace` 操作重複執行替換。這種模式的一個常見用例是在將字符串插入到 HTML 前替換字符串的特殊字符字母。

```
function escapeBasicHTML(str) {
    return str.replace(/&/g,"&amp;")
              .replace(/< /g,"&lt;")
              .replace(/>/g,"&gt;")
              .replace(/"/g,"&quot;")
              .replace(/'/g,"&apos;");
}
```

對 `replace` 的第一次調用返回一個將所有特殊字符 "&" 替換為 HTML 字符串的轉義序列 "&" 的字符串；以此類推。而這種重複的方法調用風格叫做**方法鏈**。這種風格不需要保存中間結果為變量，更簡潔。

數組方法(另一個鏈式 API)

```
var users = records.map(function(record) {
    return record.username;
})
.filter(function(username){
    return !!username;
})
.map(function(username){
    return username.toLowerCase();
});
```

這裡因為數組的每一種迭代方法，返回的都是一個數組。可以方便地再使用數組方法進行處理。

無狀態的 API 中，如果 API 不修改對象，而是返回一個新對象，則鍊式得到了自然的結果。因此，API 的方法提供了更多相似方法集的對象。
有狀態的 API 中，方法在更新對象時返回 `this`, 而不是 `undefined`。這使得通過鍊式方法調用的序列來對同一個對象執行多次更新成為可能。

三要素支持方法鏈：
- 使用方法鏈來連接無狀態的操作
- 通過在無狀態的方法中返回新對象來支持方法鏈
- 通過在有狀態的方法中返回 this 來支持方法鏈

Jquery 就是一個很好的例子。它有一組(無狀態的)方法用於從用戶界面元素中查詢網頁，還有一組（有狀態的）方法用於更新這些元素。

```
$('body').html('good')
         .addClass('nice');
```

#### API 文檔很重要。
#### API 文檔很重要。
#### API 文檔很重要。
#### 有特殊要求和慣用法不同的都要文檔說明。好嗎。




<!-- https://www.ctolib.com/docs/sfile/effective-javascript/chapter-6/avoid-unnecessary-state.html -->