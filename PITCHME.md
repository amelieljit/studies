# Effective JavaScript

### CH 1 ~ 2  定義、變數作用域

---

## 大綱
1. Numbers
2. Coercions
3. IIFE Function Declarations
4. Restricted productions
5. Variable hoisting
6. Scope
7. Closure

---

## Number

```
0.1 + 0.2           // 0.30000000000000004
(0.1 + 0.2) + 0.3;  // 0.6000000000000001 
0.1 + (0.2 + 0.3);  // 0.6

(1 + 2) + 3  // 6
1 + (2 + 3)  // 6
```

---

## Coercions

=== is going to save your life.

```
2 + 3          // 5
"2" + 3        // "23"
2 + "3"        // "23"
(1 + 2) + "3"  // "33"
(1 + "2") + 3  // "123"
"10" * 8       // 80
```

```
1 < 2 < 3  // true
3 < 2 < 1  // true
// 3 < 2 => false, false < 1 => false was coercioned as 0, 0 < 1 => true

0 == false   // true, only compares value
0 === false  // false, compares both value and type
```

---

## IIFE

```
// using a function expression
var greetFunc = function(name) {
  console.log('Hello ' + name);
}
greetFunc('Joe');
```

```
// using an IIFE, 在創造函數時立刻傳入參數並執行
var greeting = function(name) {
  return 'Hello ' + name;
}('Joe');

console.log(greeting);    // Hello Joe
console.log(greeting());  // Uncaught TypeError: greeting is not a function, 因為 greeting 回傳是字串
```

---

## IIFE

![IIFE](/assets/images/iife.png)

---

## Restricted productions

```
function getPerson() {
  return
  {
    firstname: 'Joe'
  }
}

console.log(getPerson());  // undefined
```
```
function getPerson() {
  return {
    firstname: 'Joe'
  }
}

console.log(getPerson());  // {firstname: "Joe"}
```

---

## Variable declaration and Hoisting

```
function f() {
  console.log(i);
  if(true) {
    var i = 123;
  }
}

f(); // undefined
```

via Hoisting

```
function f() {
  var i; 
  console.log(i);
  if(true) {
    i = 123;
  }
}
```

---

## Variable declaration and Hoisting

![var variables lifecycle](/assets/images/var_lifecycle.png)

---

## Scope

![scope](/assets/images/scope.png)

---

## Scope

```
function a() {
  function b() {
    console.log(myVar);
  }
    
  var myVar = 2;
  b();
}
var myVar = 1;
a();
b();

// out put
2
Uncaught ReferenceError: b is not defined
```

---

## Scope

![let variables lifecycle](/assets/images/let_lifecycle.png)

---

## Scope

```
var a = 1;
function one() {
  if (true) {
    let a = 4;
    console.log(a);    // 4
  }

  console.log(a);     // 1
}
one();
```

---

## Closure

1.JS 允許你引用在當前函數以外定義的變數

```
function makeAdder() {
  var x = 1;
  function make(y) {
    return x + y;
  };
  return make(3);
}

makeAdder();  // 4
```

---

## Closure

2.函數可以引用在其 scope 內的任何變數，包括參數和外部函數變量。

```
function makeAdder(x) {
  function make(y) {
    return x + y;
  };
  return make;
}

var add1 = makeAdder(3);
console.log(add1(1));  // 4
```

---

## Closure

3.closure 可以更新外部變數的值; 它儲存的是外部變數的 reference(參照引用)，而不是 value 的副本 

```
function cal() {
  var a = undefined;
  return {
    setNum: (newVal) => { a = newVal },
    getNum: () => { return a },
    addNum: (n) => { return a + n; },
    updateNum: (n) => { a = n; return a; },
    getNumType: () => { return typeof a; },
    checkNumType: () => { return typeof a === 'number' ? 'number' : 'You should pass a number!!'; }
  };
}

var b = cal();
console.log(b.getNumType());   // undefined
console.log(b.setNum(10));
console.log(b.getNum());       // 10
console.log(b.getNumType());   // number
console.log(b.addNum(20));     // 30
console.log(b.setNum("2"));
console.log(b.getNumType());   // string
console.log(b.checkNumType()); // You should pass a number!!
```
