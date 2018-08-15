# CH 5 Strings and Regular **Expressions**
> 字串和正規表達式

## 1. String Concatenation 字串連接

### 1.1 Plus (+) and Plus-Equals (+=) Operators

`str += "one" + "two";`

上面程式碼執行時，會發生四個步驟：

1. A temporary string is created in memory. (內存中創建一個臨時字串)

2. The concatenated value "onetwo" is assigned to the temporary string. (臨時字串的值被賦予 `onetwo`)

3. The temporary string is concatenated with the current value of str. (臨時字串與 `str` 的值進行連接)

4. The result is assigned to `str`. (最後將結果賦予 `str`)

下面程式碼可以避免產生臨時字串(避免產生上面的第一、二步)，在大多數瀏覽器中會提高速度 10% - 40%。
```
str += "one";
str += "two";
```

也可以改寫成一行以達到同樣的效果：
```
str = str + "one" + "two";
// 等價於 str = ((str + "one") + "two")
```

![](/assets/images/f5-1.gif)

(圖: s1 複製到 s2 的尾部形成 s3；基本字符串 s2 沒有被複製)

這樣避免了使用臨時字符串，因為賦值表達式開頭以 `str` 為基礎，一次追加一個字符串，從左至右依次連接。但如果改變連接順序(例如，`str ="one" + str + "two"` )，就會失去這種優化。這與瀏覽器合併字符串時分配內存的方法有關。除 IE 以外，瀏覽器嘗試擴展表達式左端字符串的內存，然後簡單地將第二個字符串拷貝到它的尾部（如上圖）。如果在一個循環中，基本字符串位於最左端，就可以避免多次複製一個越來越大的基本字符串。

上面的技術並不適用於 IE。在 IE8 中，連接字符串只是記錄下構成新字符串的各部分字符串的引用(references)。在最後時刻(當你真正使用連接後的字符串時)，各部分字符串才被逐個拷貝到一個新的"真正的"字符串中，然後用它取代先前的字符串引用，所以並非每次使用字符串時都發生合併操作。

在 IE7 以及更早的版本中，每連接一對字符串，都要把它複製到一塊新分配的內存中，而不是拷貝到第一個的尾部，這樣上面的優化只會更慢，因為這樣會多次複製大字符串(large strings)(例如: `largeStr = largeStr + s1 + s2` -> `longstr + s1`、`longstr + s2` )。

### 1.2 Array Joining and String.prototype.concat

#### Array Joining

`Array.prototype.join` 方法將陣列的所有元素合併成一個字串，並在每個元素之間插入一個分隔符字串。如果傳遞一個空字串作為分隔符，則可以簡單地將陣列的所有元素連接起來。
```
var strs = [];
var str1 = 'abc';
var str2 = '456';

strs.push(str1);
strs.push(str2.);

var newStr = strs.join(''));
newStr; // 'abc456'
```

在大多數瀏覽器上，在大部分的現代瀏覽器中 `join` 方法比 `+/+=` 方法更慢。但較適用於 IE7 及更早版本，避免了 `+/+=` 帶來的不斷增大的字符串的重複拷貝。

#### String.prototype.concat

`String.prototype.concat` 要避免使用，`concat` 比 `+/+=` 稍慢，而且和 IE7 的 `+/+=` 操作一样，存在重複拷貝大字符串(large strings)的性能問题。

## 2. Regular Expression Optimization 正規表達式優化

### 2.1 How Regular Expressions Work 正規表達式工作原理

正規表達式處理的基本步驟：

Step 1: Compilation 編譯

- 當創建了一個正規表達式對象後，瀏覽器對它進行驗證，之後再把它轉換成原生代碼程序，用於執行匹配工作；如果把正規對象賦值給一個變量，可以避免重複這個步驟。

Step 2: Setting the starting position 設置起始位置

- 目標字符串的起始搜索位置，一般是字符串的起始字符、或者正則的 `lastIndex` 屬性指定位置、或者從第四步驟返回時的最後一次匹配字符的下一個字符。 瀏覽器優化正規表達式引擎的辦法是，在這一階段中通過早期預測跳過一些不必要的工作。例如，如果一個正則表達式以
 `^` 開頭，IE 和 Chrome 通常判斷在字符串起始位置上是否能夠匹配，然後可避免愚蠢地搜索後續位置。另一個例子是匹配第三個字母是 x 的字符串，一個聰明的辦法是先找到 x，然後再將起始位置回溯兩個字符。

Step 3: Matching each regex token 匹配每個正規表達式的字元

- 一旦找好起始位置，它將一個一個地掃描目標文本和正規表達式 pattern。當一個特定字元匹配失敗時，正則表達式將試圖回溯到掃描之前的位置上，然後進入正則表達式其他可能的路徑上。

Step 4: Success or failure 匹配成功或失敗

- 如果在字符串的當前位置上發現一個完全匹配，那麼正規表達式宣布成功。如果正規表達式的所有可能路徑都嘗試過了，但是沒有成功地匹配，那麼正則表達式引擎會回到第二步驟，從字符串的下一個字符重新嘗試。只有字符串中的每個字符(以及最後一個字符後面的位置)都經歷了這樣的過程之後，還沒有成功匹配，那麼正規表達式就宣布徹底失敗。

### 2.2 Understanding Backtrack 理解回溯

回溯是匹配过程的基本组成部分，也是正規表達式的性能消耗所在。因此如何减少回溯是提高其效能的關鍵。

當一個正規表達式掃描目標字符串時，它**從左到右**逐個掃描正規表達式的組成部分，在每個位置上測試能不能找到一個匹配。對於每一個量詞(quantifier)和分支(alternation)，都必須決定如何繼續進行。如果是一個量詞(例如 `*，+?`，或者 `{2,}`)，正規表達式必須決定何時嘗試匹配更多的字符；如果遇到分支(例如通過 `|` 操作符)，它必須從這些選項中選擇一個進行嘗試。

每當正規表達式做出一個的決定，如果有必要的話，它會記住另一個選項，以備將來返回後使用。如果所選方案匹配成功，它將繼續掃描 pattern，如果其餘部分匹配也成功了，那麼匹配就結束了。但是如果所選擇的方案未能發現相應匹配，或者後來的匹配也失敗了，正則表達式將回溯到最後一個決策點，然後在剩餘的選項中選擇一個。它繼續這樣下去，直到找到一個匹配，或者量詞和分支選項所有可能的排列組合都嘗試失敗了，那麼它將放棄這一過程，然後移動到此過程開始位置的下一個字符上，重複此過程。

#### Alternation and backtracking 分支和回溯

`/h(ello|appy) hippo/.test("hello there, happy hippo");`

[https://regex101.com/r/ofS5jj/2](https://regex101.com/r/ofS5jj/2)

1. regexp 開始的 `h` 與字符串起始位置的 `h` 匹配，接下來的分支，按從左到右的原則，`(ello|appy)` 中的 `ello` 先嘗試匹配，字符串`h` 後面也是`ello`，匹配成功，於是繼續匹配
 regexp 中 `(ello|appy)` 之後的空格，仍然匹配成功，繼續匹配 regexp 中空格之後的 `h`，字符串空格之後是 `t`，匹配失敗。
2. 回到 regexp 的分支`(ello|appy)`(這就是回溯)，嘗試用 `appy` 對字符串第一位個字符 `h` 之後的字符進行匹配，失敗，這裡沒有更多的選項，不再回溯。
3. 第一個起始位置匹配失敗，起始位置後延一位，重新匹配 h… 直到字符串起始位置為 14 時，匹配到 `h`。
4. 於是開啟新一輪的字元匹配，進入分支 `(ello|appy)` 中的 `ello`，匹配失敗。
5. 回到 regexp 的分支`(ello|appy)`(再次回溯)，`appy` 匹配成功，退出分支，匹配後續的 `hippo`，匹配字符串 `happy hippo`，匹配成功，結束匹配。

![](/assets/images/example_of_backtracking_with_alternation.gif)

(圖: Example of backtracking with alternation 分支回溯的例子)

#### Repetition and backtracking 重複與回溯
```
var str = "<p>Para 1.</p>" +
          "<img src='smiley.jpg'>" +
          "<p>Para 2.</p>" +
          "<div>Div.</div>";

/<p>.*<\/p>/i.test(str);
```

1. regexp 開始的 `<p>` 與字符串起始位置的 `<p>` 匹配，接下來是 `.*`  (`.` 匹配換行以外任意字符，`*` 是貪婪量詞，表示重複 0 次或多次，匹配盡可能多的次數)， `.*` 匹配後續一直到字符串尾部的所有字符。
2. 嘗試匹配 regexp 中 `.*` 後面的 `<`，在字符串最後匹配失敗，然後每次向前回溯一個字符嘗試匹配…，直到 `</div>` 的第一個字符匹配成功，接下來 regexp 中的 `\/` 也與字符串中的 `/` 匹配成功，繼續匹配 regexp 中的 `p`，匹配失敗，返回 `</div>`，繼續向前回溯，直到第二段的 `</p>`，匹配成功，返回 `<p>Para 1. </p><img src='smiley.jpg'><p>Para 2.</p>`，裡面有 2 個段落和一張圖片，結束匹配。但這個可能並不是我們真正需要的結果，我們需要的可能是一個單一的段落。

#### The greedy and the lazy quantifier

可以通過把"貪婪量詞 `*`"改成"惰性量詞 `*?` "來匹配單個段落 `/<p>.*?<\/p>/i.test(str)`。惰性量詞回溯的方式與貪婪連詞的相反，當匹配到 `.*?` 時，它會先嘗試匹配盡可能少的次數(0 次)，直接進入 regexp 接下來的匹配部分 `<\/p>`，但是字符串的 `<p>` 後面並沒有 `<`，於是進行回溯，嘗試對 `.*?` 進行一次重複匹配，仍然不行，繼續回溯，兩次重複匹配…直到找到最近的
 `</p>`，完成匹配，返回 `<p >Para 1.</p>`。

ref: the greedy `*` quantifier with the lazy (aka nongreedy) `*?` quantifier(貪婪量詞 `*` 與惰性量詞 `*?`)

- [http://blog.stevenlevithan.com/archives/greedy-lazy-performance](http://blog.stevenlevithan.com/archives/greedy-lazy-performance)
- `/<p>.*<\/p>/i.test('<p>Para 1.</p>');` [https://regex101.com/r/giY4b1/4](https://regex101.com/r/giY4b1/4)

![](/assets/images/example_of_backtracking_with_greedy_and_lazy_quantifiers.gif)

(圖: Example of backtracking with greedy and lazy quantifiers)

#### Runaway Backtracking 回溯失控

回溯失控的時候，可能導致瀏覽器假死數秒、數分鐘或更長時間。
```
/<html>[\s\S]*?<head>[\s\S]*?<\/head>[\s\S]*?<body>[\s\S]*?<\/body>[\s\S]*?<\/html>/
```

<!-- 最後一個[\s\S]?重複會擴展到字符串末尾，匹配</html>失敗，正則會依次向前搜索可以繼續回溯的位置，在這裡是<\/body>之前的倒數第二個[\s\S]?，用它匹配到第一個</body>之後，繼續向後擴展(惰性量詞不是重複最少次數)，查找第二個</body>，直到末尾仍然沒有匹配，於是繼續回溯到倒數第三個[\s\S]*?，依次類推下去…沒有意義的回溯不斷擴展開來，會消耗掉很多的資源。 -->

回溯失控發生在 regexp 本應該快速匹配的地方，但因為某些特殊的字符串匹配動作導致執行緩慢甚至瀏覽器崩潰。避免這個問題的辦法是：使相鄰的字元互斥，避免嵌套量詞對同一字符串的相同部分多次匹配，透過重複利用 emulating atomic groups using lookahead and backreferences (向前查看和後向引用的模擬原子组) 的技術來去除不必要的回溯。

`((?=([\s\S]*?<head>))\1`
```
/<html>(?=([\s\S]*?<head>))\1(?=([\s\S]*?<\/head>))\2(?=([\s\S]*?<body>))\3(?=([\s\S]*?<\/body>))\4[\s\S]*?<\/html>/
```

[https://regex101.com/r/GZOMdZ/1](https://regex101.com/r/GZOMdZ/1)

原子組(向前查看)的任何回溯位置都會被丟棄，從根源上避免了回溯失控，但是向前查看不會消耗任何字符作為全局匹配的一部分，"捕獲組+反向引用"在這裡可以用來解決這個問題，需要注意的是這的反向引用次數，即上面的\1、\2、\3、\4對應的位置。

#### Nested quantifiers and runaway backtracking 嵌套量詞和回溯失控

嵌套量詞（例如 `(x+)*`）在匹配時，內部量詞與外部量詞的排列組合，會產生數量巨大分支路徑，這在匹配失敗之前會嘗試所有的路徑，這時候的消耗是巨大的。

例如：最糟糕的情況是用 `/(A+A+)+B/` 來匹配 10 個 A 的字符串：

https://regex101.com/r/c9xLlL/1

- 第一個 `A+` 匹配到 10 個 A，回溯一個字符，第二個 `A+` 匹配到最後一個 A，然後開始查找
 B，沒有匹配
- 嘗試所有的路徑，第一個 `A+` 匹配到 8 個 A，第二個 `A+` 匹配 2 個 A…；或者分組 `(A+A+)+` 的重複中，第一個 `A+` 匹配到 2 個 A，第二個 `A+` 匹配 3 個 A…一共嘗試 2 的 10（字符串長度）次方 1024 次回溯
- 顯示這個 B 不可能匹配到，但是 regexp 會把所有可能的路徑都嘗試一遍，最後才宣布匹配失敗。

要預防這種情況，需要確保表達式的 2 部分不能對字符串的相同部分進行匹配，`/(A+A+)+B/` 可以優化為 `/AA+B/`，或者用模擬原子組(`/((?=(A+A+))\2)+B/`)來徹底消除回溯問題。

因為正規表達式性能因應用文本不同而產生很大差異，沒有簡單明瞭的方法可以測試正規表達式之間的性能差別。為了得到最好的結果，你需要在各種字符串上測試你的正規表達式，包括不同長度，能夠匹配的，不能匹配的，和近似匹配的。

## 3. 更多提高正規表達式效率的方法

1. 讓匹配更快失敗，尤其是匹配很長的字符串時，匹配失敗的位置要比成功的位置多得多。
2. 以簡單、必須的字元開始，排除明顯不匹配的位置，如錨點(^或$)，特殊字符(`x` 或 `\u263A`)字符類(`[az]` 或 `\d` 之類的速記符)，和單詞邊界(`\b`)；盡量避免使用分組、選擇、重複量詞開頭，如 `/one|two/`、`\s`、`\s{1,}` 等。
3. 使用量詞模式時，盡量讓重複部分具體化，讓字元互斥，如用 `[^"\r\n]`代替 `.*?`(這個依賴回溯)。
4. 減少分支數量、縮小分支範圍，用字符集和選項組件來減少分支的出現，或把分支在 regexp 上出現的位置推後，把分支中最常出現的情況放在分支的最前面。
    ```
    cat|bat -> [cb]at;
    red|read -> rea?d;
    red|raw -> r(?:ed|aw);
    (.|\r|\n) -> [\s\S]
    ```

5. 精確匹配需要的文本以減少後續的處理，如果需要引用匹配的一部分，可使用捕獲，然後通過反向引用來處理
6. 暴露必需的字元，用 `/^(ab|cd)/` 而不是 `/(^ab|^cd)/`。
7. 使用合適的量詞，基於預期的回溯數量，使用合適的量詞類型。
8. 使用非捕獲組，因為捕獲組需要消耗時間和內存來記錄反向引用，並不斷更新，如果不需要反向引用，可用非捕獲組 `(?:…)` 代替捕獲組 `(…)`；當需要全文匹配的反向引用時，可用
 `regex.exec()` 返回的結果或者在替換字符串時使用  `$&` (此優化在 Firefox 中效果較小，但其他瀏覽器中處理長字符串時有較大影響)。
9. 把正規表達式賦值給變數以便重複使用和提升效能，這樣可以讓它減少不必要的編譯過程。
    ```
    var regex1 = /regex1/,regex2 = /regex2/;
    while (regex1.test(str1)) {
        regex2.exec(str2);
      	…
    }
    ```

10. 將復雜的正規表達式拆分成簡單的​​片段，每個 regepx 只在上一個成功的匹配中查找，更高效，而且可以減少回溯。

## 4. 何時不使用正規表達式

如果僅僅是搜索字符串，而且事先知道字符串的哪部分需要被測試時，正規表達式並不是最佳的解決方案。比如，檢查一個字符串是否以分號結尾：`/;$/.test(str);`

正則會從第一個字符開始，逐個測試整個字符串，看她是否是分號，在判斷是否在字符串的最後，當字符串很長時，需要的時間越多。

`str.charAt(str.length – 1) == ";";`

這個直接跳到最後一個字符，檢查是否為分號，字符串很小是可能只是快一點點，但是對於長字符串，長度不會影響所需的時間。

字符串的原生方法都是很快的，比如 slice、substr、substring、indexOf、lastIndexOf 等，他們可以避免 regexp 帶來的性能開銷。

## 5. 去除字符串首尾空白

```
function trim(str) {
    var	str = str.replace(/^\s\s*/, ''),
        ws = /\s/,
        i = str.length;
    while (ws.test(str.charAt(--i)));
    return str.slice(0, i + 1);
}

var str = " abcdefg 123 ";
trim(str);  // "abcdefg 123"
```

儘管有許多方法可以去除字符串的首尾空白，但像上面使用兩個簡單的正規表達式(一個用來去除頭部空白，另一個用於去除尾部空白)來處理大量字符串內容能提供一個簡潔而跨瀏覽器的方法。從字符串末尾開始循環向前搜索第一個非空白字符，或者將此技術同正規表達式結合起來，會提供一個更好的替代方案，它很少受到字符串長度的影響。

[http://blog.stevenlevithan.com/archives/faster-trim-javascript](http://blog.stevenlevithan.com/archives/faster-trim-javascript)

<!-- [http://programmermagazine.github.io/201307/htm/article2.html](http://programmermagazine.github.io/201307/htm/article2.html) -->

---

<!-- http://jstherightway.org/zh-tw/ -->

<!-- https://blog.csdn.net/situdesign/article/details/5631527 -->

<!-- https://blog.csdn.net/situdesign/article/details/5631524 -->

<!-- 各章小結

https://blog.csdn.net/hello_world_20/article/details/46793317

https://segmentfault.com/a/1190000008364597

https://github.com/jawil/blog/issues/2 -->