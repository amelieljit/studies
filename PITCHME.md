# Effective JavaScript
## CH 4 Objects and Prototypes

---

# 大綱

## prototype, __proto__, and object

---

# Prototype

```
function Person() { }

Person.prototype.name = 'Kevin';

var person1 = new Person();
var person2 = new Person();

console.log(person1.name) // Kevin
console.log(person2.name) // Kevin
```

Person (構造函數) ===prototye==> Person.prototype (實例原型)

Note:
每個函數都有一個 prototype 屬性, 而函數的 prototype 屬性指向了一個對象，這個對象正是調用該構造函數而創建的實例的原型，也就是這個例子中的 person1 和 person2 的原型。
每一個JavaScript對象(null除外)在創建的時候就會與之關聯另一個對象，這個對象就是我們所說的原型，每一個對像都會從原型"繼承"屬性。
(source from: https://goo.gl/gq8ATV)

---

# __proto__

```
function Person() {}

var person = new Person();

console.log(person.__proto__ === Person.prototype); // true
```

這個属性會指向該對象的原型。

Note:
非標準的方法訪問原型，然而它並不存在於Person.prototype 中，實際上，它是來自於 Object.prototype，與其說是一個屬性，不如說是一個 getter/setter ，當使用obj.__proto__ 時，可以理解成返回了 Object.getPrototypeOf(obj)。

並不是所有的 js 環境都支持通過 __proto__ 屬性來獲取對象的原型，例如，對於有 null 原型的對象來說，不同的環境處理得結果會不一樣。在一些環境中，__proto__屬性繼承自 Object.prototype, 因此，有 null 原型的對象沒有這個特殊的__proto__屬性。(它可能會污染所有的 objects，易導致 bug)，推薦使用符合標準的 Object.getPrototypeOf。
(source from: https://goo.gl/gq8ATV)

要避免修改 __proto__，最大的原因是為了保持行為的可預測性。修改對象的原型鏈會改變了它的整個繼承層次結構。而保持繼承層次結構的相對穩定是一個基本準則。

---

# constructor
```
function Person() {}

var person = new Person();

console.log(Person === Person.prototype.constructor); // true
console.log(person.__proto__ == Person.prototype) // true
console.log(Person.prototype.constructor == Person) // true

console.log(person.constructor === Person.prototype.constructor) // true
console.log(Object.getPrototypeOf(person) === Person.prototype) // true (ES5,可以獲得對象的原型)
```

Note:
既然實例對象和構造函數都可以指向原型，那麼原型是否有屬性指向構造函數或者實例呢？一個構造函數可以生成多個實例，但是原型指向構造函數倒是有的，這就要講到第三個屬性：constructor，每個原型都有一個 constructor 屬性指向關聯的構造函數。

當獲取 person.constructor 時，其實 person 中並沒有 constructor 屬性,當不能讀取到constructor 屬性時，會從 person 的原型也就是 Person.prototype 中讀取，正好原型中有該屬性，所以
`console.log(person.constructor === Person.prototype.constructor) // true`
(source from: https://goo.gl/gq8ATV)

---

![構造函数、實例原型、和實例之間的關系](/assets/images/prototype1.png)

---

# constructor

var u = new User  vs.  var u = User

```
function User(name, passwordHash) {
    this.name = name;
    this.passwordHash = passwordHash;
}
var u = User("Thor", "d8b74df39352");
u;                 // undefined
this.name;         // "Thor"
this.passwordHash; // "d8b74df39352"
var u2 = new User("Ken", "dd87sd89s8d9");
u2;                // User {name: "Ken", passwordHash: "dd87sd89s8d9"}
```

Note:
使用這個方式創建一個構造函數時，此時依賴於調用者是否記得使用 `new` 操作符來調用它。上面的例子假設接收者是一個全新的對象。

如果調用者忘記使用 `new` 關鍵字，那麼函數的接收者會是全局對象。結果回傳 undefined，還會災難性地創建(如果這些全局變量已經存在則會被修改)全局變量的 name 和 passwordHash。

如果將 User 函數定義為 ES5 的 strict mode, 則它的接收者預設為 undefined，如此一來錯誤的調用會導致明確的錯誤`error: this is undefined`, 至少可以即早發現錯誤並修復。

---

# instance and prototype
```
function Person() {}

Person.prototype.name = 'Kevin';

var person = new Person();

person.name = 'Daisy';
console.log(person.name) // Daisy

delete person.name;
console.log(person.name) // Kevin
```

Note:
當讀取實例的屬性時，如果找不到，就會查找與對象關聯的原型中的屬性，如果還查不到，就去找原型的原型，一直找到最頂層為止。
當我們給實例對象 person 添加了 name 屬性，console.log person.name 的結果自然為 Daisy。

但是當我們刪除了person 的 name 屬性時，讀取 person.name，從person 對像中找不到 name 屬性就會從 person 的原型也就是 person.__proto__ ，也就是Person.prototype 中查找，找到了name 屬性，結果為 Kevin。
(source from: https://goo.gl/gq8ATV)

---

# instance and prototype
```
var obj = new Object();
obj.name = 'Kevin'

console.log(obj.name) // Kevin

console.log(Object.prototype.__proto__ === null) // true
```
原型也是一個對象，既然是對象，我們就可以用最原始的方式創建它

Note:
所以原型對像是通過 Object 構造函數生成的，結合之前所講，實例的 __proto__ 指向構造函數的 prototype
而 Object.prototype 的原型是 null, null 表示“沒有對象”，即該處不應該有值。
Object.prototype.__proto__ 的值為 null 跟 Object.prototype 沒有原型，其實表達了一個意思。也就是說查找屬性的時候查到 Object.prototype 就可以停止查找了。
(source from: https://goo.gl/gq8ATV)

---

![原型鏈](/assets/images/prototype2.png)

Note:
圖中由相互關聯的原型組成的鏈狀結構就是原型鏈，也就是藍色的這條線。

[!!!]關於繼承，繼承意味著複製操作，然而 JavaScript 默認並不會復制對象的屬性，相反，JavaScript 只是**在兩個對象之間創建一個關聯，這樣，一個對象就可以通過委託訪問另一個對象的屬性和函數，所以與其叫繼承，委託的說法反而更準確些**。
(source from: https://goo.gl/gq8ATV)

---

# instance and prototype

```
function User(name, passwordHash) {
    this.name = name; this.passwordHash = passwordHash;
    this.toString = function() {
        return "[User " + this.name + "]";
    };
    this.checkPassword = function(password) {
        return hash(password) === this.passwordHash;
    };
}
var u1 = new User(/* ... */);
var u2 = new User(/* ... */);
var u3 = new User(/* ... */);
```

將方法存在原型對象裡

Note:
將方法存在 User.prototype 讓大家去共享，不需要每個都有一樣的方法副本，另外也能避免 instance 占用更多的內存。

而相反的情況是，有時為了讓構造函數中的變數(name)在使用它們的方法的作用域內，將方法存在實例對象中。可以利用閉包在裡面存 private data (如下例子)，以變量而不是以 this 來引用 name，
```
this.toString = function() {
    return "[User " + name + "]";
};
```
如此一來現在 User 的 instance 根本不包含任何屬性，所以外部的不能直接訪問 User instance 的 name。某種程度上此方法可以實現訊息隱藏，只是代價是可能會導致方法副本的擴散。

---

# 父子 Class 的繼承

```
function C(){
    this.name = "C";
}

C.prototype.showName = function() {
    console.log(this.name);
}

function C2() {
    this.name = "C2";
    this.father = 'C';
}

C2.prototype = Object.create(C.prototype);

var b = new C2();
b.name;    // "C2"
b.father;  // "C"
b.showName();  // C2
```

Note:
可以透過 Object.create 的方法來擴展，讓子類正確地繼承自父類的 prototype 對象，以避免調用父類的構造函數。

(source from: https://blog.csdn.net/liveinjs/article/details/25182525)

---

# 不要繼承標準類

```
function Dir(path, entries) {
    this.path = path
    for (var i = 0, n = entries.length; i < n; i++) {
        this[i] = entries[i];
    }
}
Dir.prototype = Object.create(Array.prototype);  // extends Array

var dir = new Dir("/tmp/mysite", ["index.html", "script.js"]);
dir.length;  // 0
```

Note:
length 屬性只對在內部被標記為**真正的** Array 的特殊對象起作用，也就是[[class]]。
[[class]]是一個不可見的內部屬性。length 的行為只被定義在內部屬性[[class]]的值為 "Array"的特殊對象中。

因為創在子類實例時並不是透過 new Array() 或 [] 語法創建的，所以 Dir 的實例的[[class]]屬性值為 "Object"。