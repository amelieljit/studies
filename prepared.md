# CH6 Library and API Design
## Outlines
- 7-61 不要阻塞 I/O 事件佇列
- 7-62 在異步序列中使用 Nested 或命名的 Callback function
- 7-63 注意錯誤的處理
- 7-64 對異步的循環使用遞迴
- 7-65
- 7-66
- 7-67
- 7-68

---


### 7-61 Don’t Block the Event Queue on I/O
<!-- https://juejin.im/post/5ab73dcdf265da237e09af43 -->
<!-- https://www.cnblogs.com/wengxuesong/p/5674544.html -->
> 不要阻塞 I/O 事件佇列

js 是構建在事件之上的， input 可能來自各種不同的外部來源。
```
var text = downloadSync('http://example.com/file.txt');
console.log(text);
```

像上面這樣的函數(downloadSync)稱為同步函数(或 block 函数)。以這個為例，程式會等待這個 input，也就是等它從網路上下載 file.txt 的結果完成，所以在等待期間，它不會做其他事情。

由于在等待下载完成的期间，计算机可以做其他有用的工作，因此这样的语言通常为程序员提供一种方法来创建多个线程，即并行执行子计算。它允许程序的一部分停下来等待（阻塞）一个低速的输入，而程序的另一部分可以继续进行独立的工作。
在js中，大多的I/O操作都提供了异步的或非阻塞的API。程序员提供一个回调函数，一旦输入完成就可以被系统调用，而不是程序阻塞在等待结果的线程上。

---

### 7-62 Use Nested or Named Callbacks for Asynchronous Sequencing

> 在異步序列中使用 Nested 或命名的 Callback function
(已經不推薦使用)

---

### 7-63 Be Aware of Dropped Errors

> 注意錯誤的處理

---

### 7-64 Use Recursion for Asynchronous Loops

> 對異步的循環使用遞迴

---




<!-- https://www.ctolib.com/docs/sfile/effective-javascript/chapter-6/avoid-unnecessary-state.html -->