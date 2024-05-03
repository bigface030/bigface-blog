---
title: 從 C 語言重新認識 Javascript - 關於變數傳遞的本質
date: 2024-04-19 00:00:00
tags: 技術交流
cover: https://images.unsplash.com/photo-1515173522882-fa8de74f27f6?q=80&w=2942&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

Javascript 到底是 pass by value 還是 pass by reference？嘗試向初學者時的自己解釋 Javascript 入門階段中最不能理解的事情。

<!-- more -->

![](https://images.unsplash.com/photo-1515173522882-fa8de74f27f6?q=80&w=2942&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D)

## 發現問題

在 Javascript 中，一個 object 變數作為函式的引數傳遞時，可以在函式內修改該變數的屬性，但不能在函式內針對該變數重新賦值。

以底下程式碼為例：

```javascript=
let arr1 = [1, 2, 3];
fn(arr1);
console.log(arr1);    // [4, 2, 3]

function fn(arr2) {
    arr2[0] = 4;
    arr2 = [3, 4, 5];
}
```

執行完的結果會印出 `arr1` 為 `[4, 2, 3]`。

看似理所當然的行為，到底為什麼會這樣？我有思考過類似的問題嗎？
我不禁好奇當時那個身為 Javascript 新手的自己，是如何學習的。

回到最初的起點，我打開了塵封已久的初學者填鴨式筆記，上面寫著：
> Javascript 的變數傳遞，是 **pass by value** 和 **pass by reference** 的綜合。

這到底是什麼意思? 又代表了什麼? 初步猜測我當初也只是用行為去記憶而已。

為了進一步確認當初的認知，請教了 GPT 和 Google 大神，才發現針對 Javascript 的變數傳遞這件事，各方說法都有所不同。大部分的人說是 **pass by value + pass by reference**，但也有人說是 **pass by value**，有人說是 **pass by sharing**。仔細看內文敘述，大家討論的明明都是同一件事，但就是會有不同的詮釋方式，得出不同的結論。

顯然**不是大家對 Javascript 的認知不同，就是大家對這些術語的定義不同**。

幸好在眾多術語中，Wiki 上可以找到 **reference** 這個名詞的定義：
> In computer programming, a reference is a value that enables a program to indirectly access a particular datum, such as a variable's value or a record, in the computer's memory or in some other storage device.
> 
> 對於電腦科學來說，reference 是一個值，能夠被程式用來間接訪問某個資料對象。

好了，我知道 **reference** 是什麼了，然後呢？

## 觀察行為形成假設
讓我們將剛剛的程式對照以下這段程式：

```javascript=
let arr1 = [1, 2, 3];
let arr2 = arr1;
arr2[0] = 4;
arr2 = [3, 4, 5];
console.log(arr1);    // [4, 2, 3]
```

仔細觀察後，發現變數在函式之間的傳遞，和變數複製，兩段程式的結果是一模一樣的。

稍微寫過一點 Javascript 的人應該就會知道，第 2 行程式代表的並不是真正的複製，我們針對 `arr2` 內部屬性的修改還是會修改到 `arr1` 的資料。
若要真正創建一個新的物件並複製 value，需要使用 `Array.slice()` 或是 `spread operator` 等方法進行**淺拷貝**。

我們姑且針對第 2 行程式 `arr2` 的複製行為，視為某種對 `arr1` `[1, 2, 3]` 的間接訪問，並稱作 **reference**。也就是說以下兩句成立：
- `arr2` 複製的是 `arr1` `[1, 2, 3]` 的 **reference**。
- `arr2` 是 `arr1` `[1, 2, 3]` **reference** 的 copy。

因此我們可以針對 Javascript 對於 object 變數傳遞的行為，得到一個推測：
> object 變數作為引數在函式內的操作，等同於在操作這個變數 **reference** 的 copy。

## 用底層原理驗證假設

至於什麼是 **reference**？ **reference** 的 copy 是什麼？為什麼操作它，會根據是重新賦值或是修改，而有不同的行為？

要了解這件事我們需要從底層去看。

我們以相對於 Javascript 較為 low-level 的 C 語言為例。C 語言可以透過指標的語法，手動管理程式中記憶體的使用。

### reference 是什麼
![0503.001](https://i.imgur.com/Hw4Cdk3.jpeg)

我們知道在 C 語言中，**陣列變數的本質是指向陣列第一個元素的指標**。
也就是說 `arr1` 這個變數作為指標，在其對應記憶體位置所存的值，是陣列第一個元素的地址。

此時 `arr1` 對應位置存的 `0x125` 滿足 **reference** 的定義，是一個地址的值，能夠被程式用來間接訪問 `[1, 2, 3]` 中的 `1` 這個資料對象。

而因為在 C 語言中陣列是一塊連續的記憶體，可以透過這個第一個地址去拿到第二個，第三個，甚至整個陣列的資料。
因此我們可以說：`0x125` 是 `[1, 2, 3]` 的 **reference**。

### reference copy 是什麼
![04172](https://i.imgur.com/02fcAps.gif)
```c=
// 整數變數的複製
int n = 1;
int m = n;

// 陣列變數的複製
int arr1[] = {1, 2, 3};
int *arr2 = arr1;
```

在整數變數複製的情況，我們將存在 `n` 對應的地址 `0x123` 這個位置的 value，複製一份並存到 `m` 對應的 `0x124` 位置。

而在陣列變數複製的情況，我們一樣將存在 `arr1` 對應位置的 value，複製一份並存到 `arr2` 的對應位置。所以在 `arr2` 這個對應位置存的值同樣是第一個元素的地址，並指向陣列第一個元素。

此時 `arr2` 針對 `arr1` 的複製行為，則是將 `0x125` 作為 `[1, 2, 3]` 的 **reference** 然後複製一份並存到其對應的位置。這就是 **reference** copy 的本質。

### 重新賦值和修改
![04173](https://i.imgur.com/fKhvIA5.gif)
```c=
int arr1[] = {1, 2, 3};
int *arr2 = arr1;

// 修改陣列變數內指定元素
arr2[0] = 4;

// 將陣列變數重新賦值
int arr3[] = {3, 4, 5};
arr2 = arr3;
```

在第 5 行中，我們將 `4` 這個整數傳入 `arr2` 所指向的陣列第一個元素。因為僅是修改該元素對應的記憶體位置所存的值，並沒有修改到地址，所以 `arr1` 和 `arr2` 依舊指向同一個陣列的第一個元素，得以用 `0x125` 這個地址作為 **reference** 去間接訪問 `4` 這個最新的資料對象。

而重新賦值的情況，事實上則是把 `arr2` 指向另一個陣列的第一個元素，也就是用 `0x128` 這個地址作為 **reference** 去間接訪問 `[3, 4, 5]` 中的 `3` 這個資料對象。

至於 C 語言中之所以沒有 `arr = {3, 4, 5}` 這個語法？那是因為對於一個指標變數來說，重新賦值就等同於傳入另外一個地址。而這個錯誤語法中，等號右側的 `{3, 4, 5}` 是一個常數，還沒有被我們存進記憶體中，自然也沒有可供操作的記憶體地址。

經過一連串的操作後，此時 `arr1` 指向原陣列 `[4, 2, 3]` 的第一個元素 `4`，`arr2` 則是指向新陣列 `[3, 4, 5]` 的第一個元素 `3`。

### 回到函式
![04174.001](https://i.imgur.com/2I3YJXP.jpeg)
```c=
void fn(int *arr2);

int main(void) {
  int arr1[] = {1, 2, 3}; 
  int *arr2 = arr1;
  fn(arr1);
}
```

可以發現在 C 語言中，變數作為引數在函式之間的傳遞，就等同於變數複製，兩者的行為是一致的：都是將 `0x125` 作為 `[1, 2, 3]` 的 **reference** 然後複製一份並存到其對應的位置。

這是因為記憶體的分配設計，讓函式之間有著不同的作用域，無法直接存取不同作用域的變數資料。(stack)

所以當我們在 `fn` 函式內完成一些實作，例如針對引數的變數重新賦值或屬性修改，結果也會等同於直接在 `main` 函式進行實作，其行為是一致的。

## 只靠 Javascript 無法解釋的事情

我們已經透過 C 語言了解 **reference** 的概念以及變數傳遞的原理，並可以得到這樣的結論：
> 在 C 語言中，變數作為引數在函式之間的傳遞是透過複製來實現的。且陣列操作的本質是操作它的 reference。

可以通過 C 語言的這個行為，去呼應我們先前針對 Javascript 的推斷：

> 在 Javascript 中，object 作為引數在函式內的操作，等同於在操作這個 reference 的 copy。

到這裡你可能會想說，**難道我們用 C 語言去解釋變數傳遞的行為，就能完全代表 Javascript 嗎?**

的確不能。Javascript 的實作一定也沒有那麼簡單，簡單到只要隨便畫個圖就可以完整地解釋。

以陣列來舉例，C 語言和 Javascript 的 Array 光底層的實作就有很大的差別。

前面有提到，C 語言的陣列是一塊連續的記憶體；而 Javascript 的 Array 相較於 C 語言是更為複雜的結構，實作上可能包含某種不連續，以換來某些更為彈性的操作方法。(如 Linked list 的 non-numeric index, Stack 的 push, pop 等方法...)

這也正是高階語言和低階語言的差異：**「high-level 的語法比起 low-level 較為簡單好用且較具彈性，但這同時也表示，在 high-level 語言的設計原理中，包含 low-level 語言當中某些操作的封裝。」**

我們有時候還是需要將自己拉回 low-level 的視角，去反思 high-level 語言的設計原理。

儘管 Javascript 已經幫我們處理好記憶體的問題，不需要像寫 C 語言的時候一樣，煩惱要如何配置、什麼時候要釋放，但當我們要試著解釋 **reference** 的概念和變數傳遞的行為時，還是需要引用到 C 語言中的指標這個語法。

這同時也意味著，在 Javascript 中我們無法從 **reference** 的角度充分地解釋 object 變數傳遞的行為原理，我們甚至無法只單靠 Javascript 去解釋 **reference** 這個概念。

所以我們才會需要用 C 語言，不能說完全代表 Javascript，但只是透過 low-level 的一些語法特性，來針對 high-level 中變數傳遞的行為，試著給出一個解釋，一個更為根本的理解罷了。

## 總結：關於名詞定義這回事

可能正是因為發現到高階語言中無法完整解釋變數傳遞的概念，但每次又都要回到低階語言去從底層開始解釋很麻煩，所以前人們開始導入一些名詞如 **pass by value**, **pass by reference** 等等，嘗試透過這些名詞，不僅簡化底層知識的描述也降低討論的門檻。

但這些沒有被完整定義的名詞一旦被氾濫使用，尤其又被拿來教導新手時，就很容易造成人們的誤會。

舉例來說，作為 C 語言的擴充，C++ 有 **reference 語法**，可以確實地讓變數在函式間傳遞時，得以修改該引數的地址。

而正因為 **pass by reference** 中的 **reference** 沒有被定義好，到底指的是廣義中電腦科學的 **reference**，還是狹義中 C++ 的 **reference 語法**，

所以 C 語言作為 C++ 的原型，會了方便區分，普遍還是會被認為是一個 **pass by value** 的語言。

但我們可以看看 CS50 中的 L4 是怎麼說的：

>Notice that variables are not passed by value but by reference. That is, the addresses of a and b are provided to the function. Therefore, the swap function can know where to make changes to the actual a and b from the main function.

即使我們知道 C++ 語言的 pointer 和 reference 不一樣，一個會對參考 copy 一個不會，

但對於初學者來說，又為什麼要去爭論於名詞？**了解電腦科學中的 **reference** 這個概念以及在他入門的語言當中實作的行為才是最重要的。**（註：CS50 使用 C 語言作為入門）

**reference** 本質上就只是代表一個**間接訪問**的概念，是**一個值，能夠被程式用來間接訪問某個資料對象**。而不論是 C 語言的 pointer 還是 C++ 的 reference，都只是根據這個概念去實作的語法。

**pass by XXX** 只是一種歸納法，為了方便不同程式語言之間的溝通，用來統稱該語言針對變數傳遞這個行為的結果。

在某些語境和討論底下，這個名詞確實可以帶來幫助；但是我們**不能倒果為因，嘗試用這短短幾個字去說明特定語言針對變數傳遞的方式**。更不能以某個特定語言針對 **reference** 的實作，去囊括整個電腦科學的概念。

### 我的看法

回到最初的問題，現在的我會如何對還是初學者時的自己，解釋關於變數傳遞這件事？

確實對於用 Javascript 入門的程式新手，是很難完全理解 **reference** 這個概念的。我想我會先從宏觀電腦科學的角度描述，這件事情的本質是**為了減少值的複製**，並 Javascript 當中物件的變數傳遞就是依照這個概念實作。

等了解這個前提之後，再以「**pass by value** 和 **pass by reference** 的綜合」做為總結。只要不牽涉不同語言之間的比較的話，這樣的名詞引用我想也不會有牽涉爭議的問題。

重點還是在於明白 Javascript 中變數傳遞會有怎樣的行為，同時在結合 **mutable**, **immutable** 的概念之後，能夠分辨在哪些情況下應該分別使用哪種寫法。這也會和日後寫出來的程式的**適度抽象化**有高度的相關。

說了這麼多，最重要的一點，當然還是要因材施教。
