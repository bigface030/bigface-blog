---
title: Wait！Await 和你想的不一樣！淺談非同步的 Parallel & Seqential
date: 2022-07-17 23:49:23
tags: 技術交流
cover: https://images.unsplash.com/photo-1432753759888-b30b2bdac995?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2953&q=80
---

在處理多個非同步操作的時候，大致上會分為並行 (Parallel) 和序列 (Sequential) 執行兩種情況。
本文將會從常見的應用邏輯切入，先釐清 Javascript 裡面 Async Function 的概念，進而探究序列非同步的核心，最後輔以 Event Loop 作為圖像化的解釋。

<!-- more -->

![](https://images.unsplash.com/photo-1432753759888-b30b2bdac995?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2953&q=80)

## Parallel 最常見的應用
先來看一段程式碼：
```javascript=
const arr = [1, 2, 3];
const printAfterOneSecond = (value) =>
  new Promise(() =>
    setTimeout(() => {
      console.log(value);
    }, 1000)
  );
arr.forEach((a) => printAfterOneSecond(a));
```
可以看到一秒後會同時印出陣列裡所有的元素。

在實務上通常會搭配簡潔的 Array Method 來操作，以達到同時執行多個非同步的效果。這種情況我們稱作並行 (Parallel)。

---

如果要談到非同步，API 類型的 AJAX 操作大概是最常見的應用。

以下是一個簡單的範例，模擬同時執行多個 AJAX Request 的結果。為了方便講解，僅將回傳的結果印出而不儲存。
```javascript=
// 利用瀏覽器的 setTimeout 模擬 response
const getData = (value) =>
  new Promise((res) =>
    setTimeout(() => {
      res(value);
    }, 1000 * Math.random())
  );

const arr = [1, 2, 3];

const getAllData = async () => {
  await Promise.all(
    arr.map((a) =>
      getData(a).then((result) => {
        console.log(result);
      })
    )
  );
  console.log('done!');
};

/* getAllData(); */
```

<iframe width="50%" height="300" src="//jsfiddle.net/dalian_bigface/by6wh1m0/21/embedded/result,js/dark/" allowfullscreen="allowfullscreen" allowpaymentrequest frameborder="0"></iframe>

因為 Response 回傳的時間是介於 0~1000ms 的一個隨機數，在同時發送 Request 的情況下，先完成的 Promise 就會先輸出 Resolve 值。

也就是說，利用 Parallel 的方式同時執行多支 API 時，是無法保證完成順序 = 陣列元素順序的。

但如果我今天想要一個一個照順序來呢？

## 不要忘記 Async Function 的本質

你可能會想說，在非同步動作 `getData` 的前面加上 `await`，應該就可以解決這個問題了吧？

很不幸的是，結果會出現這個錯誤：
![](https://i.imgur.com/Wg6HmPr.png)

之所以會出錯，是因為 `.map` 裡面的參數是一個 Callback Function。

也就是說剛剛加進去的 <b>`await` 並不屬於 `getAllData` 這個 Async Function 的一部份，而是在 Callback Function 的作用域裡</b>。

因為原本的 Callback Function 並不是非同步的，所以在裡面使用 `await` 這個語法，自然而然就會出現錯誤。

---

如果你認為把 callback 改成非同步就能修復錯誤並讓 APIs 變成 sequentially 執行，讓我們把 API 的範例再簡化一下：

1. 所有 Promise 都完成後不要印出「done!」。

2. 因為所有 Promise 都完成後不需要再做其他事，加上我們只需要單純地將結果輸出，我們可以把外層的 `getAllData` 轉變成一般的同步函式，並把 `.map` 改成 `.forEach`，如 sol A 所示。

3. 同時，我們可以利用 Async Function 拆解 Promise Chain，將 sol A 改寫成 sol B。

```javascript=
// sol A
const getAllData = () => {
  arr.forEach((a) => {
    getData(a).then((result) => {
      console.log(result);
    })
  });
};

// sol B
const getAllData = () => {
  arr.forEach(async (a) => {
    const result = await getData(a);
    console.log(result);
  });
};
```

如果你清楚 Async Function 的概念，就會知道這兩段程式碼是代表同一件事情。

也就是說把 callback 改成非同步，並不會影響原本 parallel 的輸出結果，只是改變寫法，利用 `async`/`await` 讓 callback 變得更簡潔而已。

又或是說，使用 `.forEach`/`.map` 這類的陣列處理方法執行非同步動作，必定會是 parallelly 執行。

---

所以我們可以用非同步的方式改寫原本的範例：

```javascript=
const getAllData = async () => {
  await Promise.all(
    arr.map(async (a) => {
      const result = await getData(a);
      console.log(result);
    })
  );
  console.log('done!');
};
```

因為 <b> `getData` 前面的 `await` 並不屬於 `getAllData` 這個 Async Function 的一部份，而是在 Callback Function 的作用域裡。</b>所以改寫過後依舊是 parallelly 地執行。


## 真正的 Sequential

這邊的三種寫法，後者其實就是前者寫法的原型。
1. async/await
2. promise chaining
3. nested callbacks

### 讓 await 發揮作用
因為 `.forEach` 裡面會帶一個 Callback Function 做為參數，既然我們不能在 Async Function 裡面使用 `await` 去搭配 `.forEach`，那我們就不要使用 `.forEach` 去迭代陣列。

回歸最原始的 `for` Loop 寫法，可以順利地和外層的 Async Function 一起運作，達到 sequential 執行的效果。反之，如果不搭配 `async`/`await` 使用，則是會變回 parallel 執行的效果。

```javascript=
// for
const getAllData = async () => {
  for (let i = 0; i < arr.length; i++) {
    const result = await getData(arr[i]);
    console.log(result);
  }
  console.log("done!");
};

// for-in
const getAllData = async () => {
  for (const i in arr) {
    const result = await getData(arr[i]);
    console.log(result);
  }
  console.log("done!");
};

// for-of
const getAllData = async () => {
  for (const a of arr) {
    const result = await getData(a);
    console.log(result);
  }
  console.log("done!");
};

// for-await-of
const getAllData = async () => {
  const promises = arr.map((a) => getData(a));
  for await (const result of promises) {
    console.log(result);
  }
  console.log("done!");
};
```

### Promise Chaining

第一個方法主要是透過在 Async Function 裡面使用多個 `await` 來達成依序執行的效果。然而這個寫法的原型其實是來自於 Promise Chain。

前面有說過用 `.forEach`/`.map` 等陣列處理方法來執行非同步操作必定會是 Parallel，不過其實有一個例外：`.reduce`。

我們可以使用 `.reduce` 來實作 Promise Chain 的概念，來讓程式碼不會過於冗長。

```javascript=
const getAllData = async () => {
  await arr.reduce(
    (acc, cur) =>
      acc.then(() =>
        getData(cur).then((res) => {
          console.log(res);
        })
      ),
    Promise.resolve()
  );
  console.log("done!");
};
```

將 callback 改寫成 Async Function（你終究要用 async 的，為什麼不現在就用）

```javascript=
const getAllData = async () => {
  await arr.reduce(async (acc, cur) => {
    await acc;
    const res = await getData(cur);
    console.log(res);
    return res;
  }, Promise.resolve());
  console.log("done!");
};
```
### 巢狀 Callback

在第二種解法我們使用到了 Promise Chain 的概念。然而這種寫法之所以存在，則是為了解決多個非同步的巢狀結構呼叫容易造成 Callback Hell 的問題。

我們可以利用遞迴 (Recursive) 的概念，讓非同步的程式碼不要包那麼多層。

```javascript=
const getRecursiveData = (array, index) => {
  return getData(arr[index]).then((res) => {
    console.log(res);
    if (index + 1 === array.length) return;
    return getRecursiveData(array, index + 1);
  });
};
const getAllData = () => {
  getRecursiveData(arr, 0).then(() => {
    console.log("done!");
  });
};
```

其中的 `.then` 一樣能改寫成 `Async Function`，這邊就不再重複。

## 用 Event Loop 的角度看

有人可能會想說 Javascript 不是單一執行緒的語言嗎？parallel 執行又是如何做到的？
這個問題我們可以用 Event Loop 的概念來解釋。（詳情請見 Philip Roberts 的[這個影片](https://youtu.be/8aGhZQkoFbQ)）

簡單來說就是：非同步操作並不是在 main thread 上被 Javascript 執行的。

Javascript 在執行時若遇到非同步動作的情況，為了不要造成阻塞就會將他從 call stack 移開，從而得以繼續在 main thread 上執行其他操作。

![event_loop](/images/event_loop.gif)

這邊有兩個值得注意的地方：

1. 可以看到 JS 的確會依序執行陣列裡的元素，不過因為 main thread 僅負責將 AJAX requests 放入 Web APIs，並沒有直接執行，當然也不需要花時間等待，所以我們可以視為同時 trigger 多個 AJAX requests。
2. 假設我需要在拿到 response 後將結果印出，如上方範例，這個非同步動作會帶一個 callback。以 XMLHttpRequest 來說就是 `onload`，Fetch API 則是 Promise 的 `.then` 或 `.catch`。另外，若今天不是 parallel 而是 sequential 的情況，那麼下一個即將執行的非同步動作也會被包含在這個 callback 裡面。


## 結語

礙於篇幅的緣故，很多觀念只能點到為止，甚至需要省略一些部分。（例如本文沒提到的 `generator`，也是常常應用於非同步操作的一種語法）

另外就是，本文的 API 範例為了方便講解，簡化了非常多步驟。實務上針對非同步處理這個問題則會有更多需要列入考量的項目，如條件判斷、response 如何儲存、error handling 等等。這部分就會因為業務邏輯的不同而產生非常多寫法上的變化。

總而言之就是，非同步的世界還有很多東西正在等著大家去探索！

## 後記

其實會寫這篇一開始只是因為工作上碰巧有遇到 sequential 的情況，過程中意外地發現了自己過往的一些盲點，所以想記錄一下解決這個問題的思路，單純地讓自己之後能夠再次複習。

誰知道在堆砌文字的過程中，竟不自覺地將自己帶入一個分享者的角色。有時候甚至還會想：「如果也能恰巧地幫助到和我一樣有相同盲點的朋友，那就太好了！」

因此若內容有需要調整的地方，或認為怎麼寫會更容易讓人理解，任何的意見回饋都歡迎和我聯絡，一同交流讓彼此更進步！
