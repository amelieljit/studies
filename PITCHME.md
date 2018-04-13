# Effective JavaScript

### CH 3 bind, call, apply, arguments

---

## 大綱
1. .bind()
2. .call() & .apply()
3. arguments

---

## bind

普通的函數調用隱式傳入 this，但 call, apply, bind 可以顯式地指定它。

```
var obj = {
  x: 123
};

var func = function() {
  console.log(this.x);
};

func();            // undefined
func.bind(obj)();  // 123
```

Note:
可以想像成把某個 function 在執行的時候，「暫時」把它掛在某個物件下，以便透過 this 去取得該物件的 Context。

bind() 讓這個 function 在呼叫前先綁定某個物件，使它不管怎麼被呼叫都能有固定的 this。 尤其常用在像是 callback function 這種類型的場景，可以想像成是先綁定 this，然後讓 function 在需要時才被呼叫的類型。

加上了 `bind` 之後的 `func.bind(obj)()` 執行的結果，會替我們將 `func` 的 `this` 暫時指向我們所設定的 `obj`。 於是，`console.log(this.x)` 的結果自然就是 `obj.x` 也就是 `123` 了。

bind 會 copy 這個函數，並且設定這個新函數物件、新的 copy, 所以當他執行時，執行環境被創造，JS 看到這個被 bind 創造，它會把 this 指向被傳入給 bind 的東西(bind 的第一個參數)

---

## bind

bind allows us to borrow methods

```
var cars = {
  data: [
    {name:"Honda Accord", age:14},
    {name:"Tesla Model S", age:2}
  ]
};
​var user = {
	showData: function(event) {
    // random number between 0 and 1
    var randomNum = ((Math.random() * 2 | 0) + 1) - 1;

    console.log(this.data[randomNum].name + " " + this.data[randomNum].age);
	}
};
cars.showData = user.showData.bind(cars);
cars.showData(); // Honda Accord 14​
```

Note:
// Here we have a cars object that does not have a method to print its data to the console​
// We can borrow the showData() method from the user object
// Here we bind the user.showData method to the cars object we just created.​

One problem with this example is that we are adding a new method (showData) on the cars object and we might not want to do that just to borrow a method because the cars object might already have a property or method name showData. We don’t want to overwrite it accidentally. As we will see in our discussion of Apply and Call below, **it is best to borrow a method using either the Apply or Call method**.

---

## bind

bind allows us to curry a function

```
function multiply(a,b) {
  return a*b;
}

var multiplyByTwo = multiply.bind(this, 2);
console.log(multiplyByTwo(2));  // 4
console.log(multiplyByTwo(3));  // 6

var multiplyByThree = multiply.bind(this, 3);
console.log(multiplyByThree(2));  // 6
console.log(multiplyByThree(3));  // 9
```

Note:
Function Currying, also known as partial function application, is the use of a function (that accept one or more arguments) that returns a new function with some of the arguments already set. The function that is returned has access to the stored arguments and variables of the outer function.

---

## call & apply

透過 `.call()` 或是 `.apply()` 去指定當下的 `this` 是誰， 而差別只在傳入參數的方式有所不同。

Note:
.call() 與 .apply()是使用在 context 較常變動的場景，依照呼叫時的需要帶入不同的物件作為該 function 的 this，在呼叫的當下就立即執行。

`.call()` 或是 `.apply()` 都是去執行這個 function ，並將這個 function 的 context 替換成第一個參數帶入的物件，換句話說，就是**強制指定某個物件作為該 function 的 this**。而兩者的作用完全一樣，差別只在傳入參數的方式有所不同

---

## call

傳入參數的方式是由「,」隔開，一個一個去指定參數

```
var obj = { name: "Doraemon" };

var greeting = function(a,b) {
  return this.name + " hates " + a + " and loves " + b;
};

console.log(greeting.call(obj, "mouse", "cat"));
// Doraemon hates mouse and loves cat
```

---

## apply

以 array 方式做為要傳入的參數

```
var obj = { name: "Doraemon" };

var greeting = function(a,b) {
  return this.name + " hates " + a + " and loves " + b;
};

var answers = ["mouse", "cat"];
console.log(greeting.apply(obj, answers));
// Doraemon hates mouse and loves cat
```

---

## Borrow functions

borrow some array methods to operate on the array-like object

```
var anArrayLikeObj = {
  0: "Martin",
  1: 78,
  2: 67,
  3: ["Letta", "Marieta", "Pauline"], length:4
}

var newArray = Array.prototype.slice.call(anArrayLikeObj, 0);
​console.log(newArray);  // ["Martin", 78, 67, Array[3]]​

console.log(Array.prototype.indexOf.call (anArrayLikeObj, "Martin") === -1 ? false : true); // true

console.log(anArrayLikeObj.indexOf("Martin") === -1 ? false : true); // Uncaught TypeError: anArrayLikeObj.indexOf is not a function(Object has no method 'indexOf')
```

Note:
// Search for "Martin" in the array-like object​
// Try using an Array method without the call () or apply ()​

---

## bind, call, apply 的差異

```
var obj = {
  x: 42,
};

var foo = {
  getX: function() {
    return this.x;
  }
}

console.log(foo.getX.bind(obj)());  //42
console.log(foo.getX.call(obj));    //42
console.log(foo.getX.apply(obj));   //42
```

Note:
三個輸出的都是42，但是要注意的是 bind() 方法後面多了對括號。

也就是說，區別是當你希望改變上下文環境之後並非立即執行，而是回調執行的時候，使用 bind() 方法。而 apply/call 則會立即執行函數。

---

## bind, call, apply 的差異

1. apply 、 call 、bind 三者都是用來改變函數的 this 對象的指向
2. apply 、 call 、bind 三者第一個参數都是 this 要指向的對象(也就是想指定的上下文)
3. apply 、 call 、bind 三者都可以利用後續參數按照顺序作為原函數運行時的参數
4. bind 是返回對應函数，便於稍後調用；apply 、call 則是立即調用

---

### arguments

可用來取得函數傳入的實際變數，適合用在可變參數的函數，包含在 arguments 物件中的個別引數都可以透過與存取陣列元素相同的方式進行存取。

```
function sum() {
  var total = 0;
  for(var i=0; i < arguments.length; i++ ) {
    total += arguments[i];
  }
  return total;
}

console.log(sum());      // 0
console.log(sum(1, 2));  // 3
console.log(sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));  // 55
```

---

### arguments

```
function values() {
    var i = 0, n = arguments.length;
    return {
        hasNext: function() {
            return i < n;
        },
        next: function() {
            if (i >= n) {
                throw new Error("end of iteration");
            }
            return arguments[i++]; // wrong arguments
        }
    };
}

var it = values(1, 4, 1, 4, 2, 1, 3, 5, 6);
it.next(); // undefined
it.next(); // undefined
```

Note:
next 方法含有自己的 arguments，所以 return arguments[i++], 我們訪問的是 it.next 的參數，而不是 values 的

---

### arguments

解決方法：在 arguments 對象作用域內把 arguments 綁到一個新的局部變數裡
```
function values() {
    var i = 0, n = arguments.length, a = arguments;
    return {
        hasNext: function() { return i < n;
        },
        next: function() {
            if (i >= n) {
                throw new Error("end of iteration");
            }
            return a[i++];
        }
    };
}
var it = values(1, 4, 1, 4, 2, 1, 3, 5, 6);
it.next(); // 1
it.next(); // 4
```

---

### arguments

arguments is an "Array-like" object, a reference, but not the real array. so, for the sake of saving your life...

1. Never modify the arguments
2. Use `[].slice.call(arguments)`
3. Use a variable to save a reference to arguments (when it's in nested functions)