---
title: 為了寫文章，我開了一間飲料店！淺談 web worker 針對併發請求的實作
date: 2023-06-29 23:49:23
tags: 技術交流
cover: https://images.unsplash.com/photo-1516216628859-9bccecab13ca?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1769&q=80
---

本文將從 web worker 最基本的用法切入。除了程式面的語法以外，同時針對概念以及 design pattern 進行解釋，並在這個過程中逐漸模擬出真實世界的使用情境。

本文的範例程式碼：https://github.com/bigface030/web-worker-with-parallel-requests

<!-- more -->

![](https://images.unsplash.com/photo-1516216628859-9bccecab13ca?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1769&q=80)

## 什麼是 web worker? 為什麼要使用?

因為 Javascript 是 single thread 的程式語言，一次只能做一件事。所以如果在 main thread 上進行耗時的計算 (eg. 跑一個 2^53 的迴圈)，就會阻斷其他網站的基礎工作 (eg. UI render、parsing)。

使用 web worker，可以讓我們在瀏覽器上開出額外的 thread。在利用他們協助做耗時運算的同時，也讓 main thread 得以繼續執行 call stack 裡的任務。

舉例來說，來到一間飲料店，這間飲料店在那個時段只有老闆自己顧店。那麼一旦老闆收到顧客的訂單去製作飲料，就沒辦法同時應付等待點餐的客人、出飲料給客人、收貨、外送、煮珍珠等等任務。

然而，如果老闆願意多雇用一個員工，專門在工作區製作飲料，那麼在飲料製作的同時，老闆就能夠繼續執行其他的工作。

於是我們迎來了第一位員工 - 小帥。

## One Request to One Worker

web worker 最基本的用法如下：
```javascript=
const worker = new Worker('worker.js');

worker.onmessage = (e) => {
  console.log(`${e.data} 好了`);
};

// 使用時呼叫
worker.postmessage('阿娘喂翡翠檸檬綠茶');
```


假設這是一間乏人問津的飲料店，小帥一個時間點只會接到一張單，且做完每張單中間都有時間休息，那麼會長這樣：

![](https://i.imgur.com/eAP3Qrb.png)

然而，基於以下兩個理由
- 正在實作 library，想要增加 client 端使用的簡易性，不需要由 client 自行 new worker
- 正在開發大型專案，想要針對現有的程式碼做改善，做更好的抽象化和封裝

可以利用 javascript 的非同步特性，透過 promise 做一層抽象化，達到封裝 worker 的目的。

```javascript=
const worker = new Worker('worker.js');

const viaWorker = (drink) => {
  worker.postMessage(drink);
  return new Promise(resolve => {
    worker.onmessage = (e) => {
      resolve(e.data);
    };
  });
};

// 使用時呼叫
viaWorker('阿娘喂翡翠檸檬綠茶').then(resolvedValue => {
  console.log(resolvedValue);
});
```

如此一來就得到了一個更方便使用的 interface，乍看之下是一個不會有問題的寫法。

## Multiple Requests to One Worker

很遺憾的，對於台灣的天氣來說，除非真的很難喝或開在鳥不生蛋的地方，否則午茶時刻，點飲料的人總是大排長龍。

訂單一直來，製作飲料絕對也是一杯接著一杯，小帥根本沒時間可以休息。

因為每杯飲料的製程有的簡單有的繁瑣，又或者是每個訂單的製作優先順序不同，
這時如果同時有多位客人在等飲料，飲料店通常會發給每個等待中的客人一張號碼牌。

顧客只需要正確地依照店員的叫號取餐，不需要親自查看飲料杯上的標籤或內容物，就能夠確保每一位客人在取餐時都能拿到和點餐過程中相符的品項。

同樣地，如果有多個 requests 在短時間內被送到這個 worker，
在沒有經過封裝，原生的 web worker 的寫法中，我們可以加上識別碼 (identification code) 去辨識：

```javascript=
const worker = new Worker('worker.js');

worker.onmessage = ({ data }) => {
  console.log(`顧客 ${data.id} 的 ${data.drink} 做好了`);
};

// 使用時呼叫
const id = crypto.randomUUID();
worker.postMessage({ id, drink: '咪咕嚕嚕' });
```

在封裝的版本，你可能會想說可以透過一樣的方式去處理，但會先發現一個奇怪的現象：

![](https://i.imgur.com/v7gYmFo.png)

只有最後一個點餐的人會收到來自 worker 的訊息，而且不能保證拿到自己的飲料！

![](https://i.imgur.com/itnBA8C.png)

原因是因為我們每次呼叫的時候，都會 assign 一個 callback 給 message 這個 event，
也就是說 onmessage 事件在每次呼叫的時候都會被重新註冊，並且和當前 promise 的 resolve 值綁定在一起，導致互相覆蓋的情況。

在多個 request 的情況下，等待 resolve 的值會有很多個，而 onmessage 事件僅僅只能被註冊一次。

所以我們可以建立一個 resolve 值的 cache，並且在產生 worker 的時候就預先註冊 message 事件。

```javascript=
const worker = new Worker('worker.js');

const resolvers = {};

worker.onmessage = ({ data }) => {
  resolvers[data.id](data);
  delete resolvers[data.id];
};

const viaWorker = (drink) => {
  const id = crypto.randomUUID();
  worker.postMessage({ id, drink });
  return new Promise((resolve) => {
    resolvers[id] = resolve;
  });
};

// 使用時呼叫
viaWorker('咪咕嚕嚕').then(({ id, drink }) => {
  console.log(`顧客 ${id} 的 ${drink} 做好了`);
});
```

如此一來就能取消 onmessage 和各個 resolve 值的綁定。
同時一樣也可以將回傳的訊息加上識別碼，讓 main thread 得以進行辨識。

## Multiple Requests to Multiple Workers
隨著飲料店生意越來越好，加上小帥已經快要變成富貴手，老闆不得不再多雇用一名員工小美，來分擔小帥的工作量。

起初，老闆都還是會先將訂單送到小帥那邊，當小帥忙不過來的時候才會 pass 給小美，小美只要做完飲料就會 pass 回小帥，最後才回到老闆手上。

![](https://i.imgur.com/24b9gPD.png)


然而這麼做會有幾個問題：
1. 飲料在人員之間傳遞花費更多溝通時間
=> postmessage 傳遞過程中的資料運輸需要成本
2. 老闆收到一杯有問題的飲料很難確認是小美或小帥的問題
=> 從 dev tool 難以針對 sub-worker 進行 debug
3. 老闆難以管理小美的工作狀態：
=> main thread 無法及時掌握 sub-worker 執行 task 的狀態

於是老闆想了一個新的方法：

由老闆依照小帥和小美手上的工作量分配工作讓他們各自處理，做完飲料後各自直接交付給老闆。

![](https://i.imgur.com/GST6kek.png)

這個機制被稱作 thread pool，是一個針對 multi-thread 的設計模式，可以實作在大部分主流的程式語言上。

同時可以進一步優化，定義以下兩個變數：
- maximum: 短時間內多個 request 時，產生的最大 worker 數量
(若當前沒有空閒 worker 且 worker 數量少於這個值，呼叫時會產生新的 worker)
- minimum: 處理完短時間內多個 request 後，常駐的 worker 數量
(當前 worker 數量超過這個值且有空閒 worker 時，該空閒的 worker 將會被移除)

也就是說，沒有客人或人少的時候我可以只用一個人力去做飲料，尖峰時刻再增加人力來上班，
如此一來就可以增加更多的彈性，最小化人事成本。

不過在瀏覽器中並不是想開多少 thread 就有多少，數量上限通常會和 CPU 的核心數有關，這部分會需要多多注意。

## 結語
為了方便解釋，本文的範例是非常簡化的版本，有些面向礙於篇幅並沒有提及，包含但不限於
- 送到 web worker 的格式該如何驗證？
- web worker 處理過程發生錯誤怎麼辦？無法回傳訊息怎麼辦？
- 不同 worker 之間的合作關係該怎麼處理？

這些東西可能比較沒有固定的處理方式或是 best practice，比較像是根據當下的需求做評估，並參照過往經驗去實踐，而這部分筆者也還正在練功中。

本文若有誤再麻煩大大們指正，有任何想法也歡迎和我交流。
