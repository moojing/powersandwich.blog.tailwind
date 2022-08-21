---
title: React 元件庫製造紀事錄 (ㄧ) – 問題說明與第一個麻煩：JavaScript 的模組系統
date: '2021-08-18'
tags: [frontend, javascript]
summary: '這陣子為了解決了一個讓我蠻頭痛的問題，第一次接觸前端 module 的打包，也了解到原來平常我們很方便能夠用 Npm 或 Yarn 這類套件管理器裝一裝就能夠直接使用的一些第三方工具在功能開發完之後，為了要讓別人能夠使用所做的處理上並不是這麼的單純，隨著使用方式的不同，要處理的部份也會有所不一樣。'
layout: PostLayout
---

這陣子為了解決了一個讓我蠻頭痛的問題，第一次接觸前端 module 的打包，也了解到原來平常我們很方便能夠用 Npm 或 Yarn 這類套件管理器裝一裝就能夠直接使用的一些第三方工具在功能開發完之後，為了要讓別人能夠使用所做的處理上並不是這麼的單純，隨著使用方式的不同，要處理的部份也會有所不一樣。

舉例來說，如果是一般純粹都是 JavaScript 函式庫，可能只需要透過 Babel 這類工具來處理 JavaScript 版本與瀏覽器的支援性問題就好。不過若你的專案需要處理 React 元件或是圖片 Icon 的打包，而且這些元件還必須要能夠應用在 SSR 的架構內，那可就要多花一點心思了。除了必須考慮 JavaScript 在 Node Server 以及在 Browser 兩種環境是否都能夠正常被使用，還必須著手進行圖片、 SVG 等等靜態資源的處理。

本系列文預計會以三篇來呈現：

- 第一篇會介紹問題發生的原因，和 JS 模組系統的差異
- 第二篇則是會介紹 Webpack 的使用過程
- 第三篇則是 Rollup 的解說，以及為何最後會選擇 Rollup
  這系列文章也許會有比較進階的內容，初學者若有興趣，但覺得太難也沒關係，可以直接跳過或是先了解問題發生的原因即可。

## 我所遭遇的問題

我目前所參與的團隊在很久以前有把嘗試做過設計系統，並把一些需要共用的元件從原本各個專案整理出來成為一個第三方的元件專案。當時所有專案的架構都是同一個前人手工組裝的 Server Side Rendering ( SSR ) 架構 （ 以下稱為 A 專案 ）， 為何是手工組裝？因為那時候在 React 的世界裡面，還沒有像是 Next.js 這麽方便又流行的 SSR 框架，想必是一個厲害的前輩手把手打造出來的吧。

而針對當時所整理出來的第三方元件，都是沒有經過處理的 React code ，也就是說這些元件雖然都有拆分並作成 Npm Module ，但是在使用上也與專案內其他資料夾下的 React 元件無異。

這代表了幾件事情：

- 這些 React Code 一定要先經過處理才能在 Server Side / Client Side 被使用，也就是說這麼一來**使用這個第三方元件 module 的專案就會被迫需要進行這個 module 原始碼的處理** ，隨著專案打包流程的不同，也有可能會影響專案的正常運行。
- 沒辦法去除沒有在使用的元件程式碼 ( 這個動作稱為 Tree Shaking )，代表這個 module 內的所有 React 元件都要統一在專案內被處理過，間接增加了專案的負擔。
  接著，因為某些原因，在原來的專案架構下後來又出現了另外一個新專案（ 以下稱為 B 專案 ），它的專案架構與上述 A 專案在 Application 內 SSR 的處理完全不一樣，**所以毫不意外的沒辦法直接使用我們的共用 module**。

![problem description](/static/images/blog/frontend/react-module-note-1/problem-1.png)

一份原始碼，要同時在兩個不同架構的專案之下執行，最直接的方式當然是兩個專案各自去處理，這個方式可以透過 Monorepo 或是 Submodule 來達成，只是這樣或許不是最有效率的方式？在這個問題上我曾和團隊成員們有了多次的討論。

後來，我們認為這種另包成 module 的元件或函式庫，在被其他專案引用的時候都應該已經要是能夠直接被使用的 JavaScript Code ，不需要再另外經過處理，會是一個相對比較好的方式。

所以這次的挑戰就是要在這個第三方 module 的發布流程中間**加入一個處理原始碼的環節，把原來的 React Code 轉成可以直接被 Node Server / Browser 看懂的 JavaScript 程式碼**，為了後面方便理解，我在接下來的部分將會用打包來代稱這個處理的動作。

## 打包？是要包什麼？怎麼做？

如上所述，這邊說的打包**就是把原來的 React Code 這類瀏覽器看不懂的程式碼想辦法先轉成一般的 JavaScript 程式碼**，而不是到了專案內才由其他專案來處理這個動作。至於打包具體要怎麼做呢？目前你可以選擇的可能有 Webpack 、 Rollup 以及 Parcel 這幾個比較常引起討論的工具。不過到底選擇哪一個工具才是最好的呢？我認為沒有絕對的好或壞，只要能夠達到你的目的就好，所以在選擇工具前不妨再次確定你想要達成什麼樣的效果。

關於工具上的差異，如果你去 google 「Webpack v.s. Rollup」，應該很容易會找到「Webpack 比較適合用在應用程式，而 Rollup 比較適合用在函式庫上」這類的說法，但是我依舊是那句話「只要能達成目的就好」，工具沒有絕對的好或壞，更何況在一開始考慮太多很容易讓你躊躇不前，另外，也建議不要盲目相信這類別人歸納出的簡短結論，自己了解看看來由會比較好。不過若你真的有興趣的話，有關上述的說法可以[參考這裡](https://powersandwich.com.tw/2021/08/18/react-%e5%85%83%e4%bb%b6%e5%ba%ab%e8%a3%bd%e9%80%a0%e7%b4%80%e4%ba%8b%e9%8c%84-%e3%84%a7-%e5%95%8f%e9%a1%8c%e8%aa%aa%e6%98%8e%e8%88%87%e7%ac%ac%e4%b8%80%e5%80%8b%e9%ba%bb%e7%85%a9%ef%bc%9ajavascr/#:~:text=%E7%9A%84%E8%AA%AA%E6%B3%95%E5%8F%AF%E4%BB%A5-,%E5%8F%83%E8%80%83%E9%80%99%E8%A3%A1,-%E3%80%82)。

![webpack rollup bundle flow](/static/images/blog/frontend/react-module-note-1/webpack-rollup-bundle.jpg)

上述提及工具的前兩者，也就是 Webpack / Rollup 可能比較適合我所面臨的情境，因為 Parcel 雖然也是一套打包工具，不過其主張的是讓開發者不用調整任何設定就能夠直接達成目的，但是若我們未來想要針對某個特定部分做優化，可能就會比較麻煩，所以先不考慮。

那麼直接講工具選擇上的結論的話就是：在這次的處理上，我各自嘗試了 Webpack 版本的處理方式，以及 Rollup 版本的處理方式，最後決定使用 Rollup ，原因**是我需要把原始碼打包之後是 ES module ，而不是 CommonJS 模組系統（module system）的程式碼**。 但是 Webpack 對 ESM 的支援性，在撰寫這篇文章的當下，似乎還停留在[**實驗階段**](https://webpack.js.org/configuration/output/#type-module)，這部分相對來講 Rollup 就成熟許多。

那麼為什麼會需要 ESM ? ESM 又是什麼？在繼續往下解釋打包的做法之前，我想我們必須先來了解 JS 的模組系統（ Module System ) 。

## JavaScript Module System

上一段的最後面所提到的 ESM 以及 CommonJS 又是什麼東西呢？它們在 JavaScript 裏面被稱作模組系統，是用來讓不同邏輯可以被妥善切分到不同檔案來管理的方式。

最早 JavaScript 是被設計來用在瀏覽器的互動操作上的，在當時我們可以使用 HTML 的 `<script>` 標籤來載入一段 JavaScript 自己設計的程式碼。若想使用 JQuery 之類的函式庫呢？同樣也可以透過這個 `<script>` 標籤

當然，這個方法到今天依然還是可以使用的，而且還不算少見，但是使用這個方法可能會有幾項缺點，以及需要注意的地方：

- `<script>`被載入的順序很重要，如果想要定義一些全域共用的變數，就要特別注意這一點，否則可能在使用上會出錯。
- 須注意全域變數意外被覆寫的問題，跟上一點很像，使用這個方法就沒辦法自由的切分變數的 作用域。既然會因為少給一個全域變數而出錯，就有可能會因為多給了一次而導致意外。
- 檔案切分困難，如果想把一段一千行的邏輯拆成 N 個檔案，但其中有某些邏輯必須讓拆分後的檔案共用，那麼只能透過把這些共用邏輯放到全域來達成，長久下來增加了管理上的風險跟麻煩。
- 使用 `<script>` 標籤無法去除多餘沒有在使用的程式碼，也就是說沒辦法做到網站載入速度上的優化。

我們需要一個更好切分程式碼的方式，讓我們在使用 JavaScript 開發及維護規模較大的產品時不至於因為上述問題而感到太過痛苦。於是就有人跳出來提出了一些模組的概念，想要解決這些問題。這中間經歷了許多不同模組系統的百家爭鳴時期，而目前最常見的模組系統是 **CommonJS （ 這邊簡稱 CJS ）** 與前端這幾年常見的 **ECMAScript Module ( ESM )**，以下我們一個一個來看看。

### CommonJS

CommonJS 可以說是最早被創造出來的模組系統，主要於 Node.js 內被使用。在 Node.js 裏面想要引入一個獨立的檔案內容或是一個 lib 的話，用的就是 “require” 這個方法，而想要讓一個程式內容可以被用在其他地方，則是使用 `exports` 或 `module.exports`

```js
const someOtherFunction = require('./utils/otherFunction')
const getSomeValue = () => someOtherFunction('33')
// exports.getSomeValue = getSomeValue
module.exports = getSomeValue
```

### AMD

Node 使用的 CommonJS 主要是為了後端開發而生的模組系統，並且在使用 “require” 來載入其他模組的時候，是非同步的，而在瀏覽器端如果載入所有任一模組都是使用非同步的方式來進行的話，是很容易會造成嚴重的體驗問題的。

因此有一群開發者就跳出來以 CommonJS 為原型，提出了比較適合在瀏覽器端使用，並且能夠以非同步方式被載入的模組系統，它就叫做 AMD （Asynchronous Module Definition）。在 AMD 裏面會是使用`”define`來定義一個模組，讓一個模組可以被其他模組所用：

```js
  define(id?, dependencies?, factory);
```

上述 define 函式的參數裏面，其中 id 為模組名稱，dependencies 為此模組相依的其他模組，而第三個 factory 可以是物件或是函式，若是物件，那麼此物件會被指派至引用它的模組，若是一個函式，則會以回傳值來指派給引用該模組的位置。以下是一個 AMD 模組與 jquery 一起搭配的簡單範例：

```js
define('myModule', ['jquery'], function ($) {
  $(document).ready(function () {
    // do some thing.
  })
})
```

### UMD

隨著 AMD 與 CommonJS 模組越來越流行，接下來就有人想到另外一個問題，那就是如果要讓同一個模組可以同時被用在前端，也能夠被用在後端 Node.js 裡面的話該怎麼辦？於是就催生了另一種能夠在前後端通用的通用模組： **Universal Module Definition，簡稱 UMD**。

```js
;(function (root, factory) {
  // 判斷其他模組是用什麼方式來引用的，並給予對應的輸出方式

  if (typeof define === 'function' && define.amd) {
    // AMD
    define(['jquery'], factory)
  } else if (typeof exports === 'object') {
    // CommonJS
    module.exports = factory(require('jquery'))
  } else {
    // Browser global "window"  (root is window)
    root.returnExports = factory(root.jQuery)
  }
})(this, function ($) {
  //    methods
  function myFunc() {}

  //    exposed public method
  return myFunc
})
```

上述就是一個 UMD 模組的例子，UMD 其實就是透過判斷其他模組的引入方式來給予對應的輸出方式來達到所謂「通用」的目的，**可以看成是 CommonJS 與 AMD 模組的組合**。

### ESM

ESM 是目前已經存在 ECMAScript 裡面的官方模組規範。而 ESM 系統則是在 JavaScript 史詩級大改版的 ES6 規範中ㄧ起問世的。若你是前端工程師，那麼在 ESM 的 module 用法中，你會看到非常熟悉的 `import` 以及 `export` 語法。

```js
import someOtherFunction from './utils/otherFunction'
const getSomeValue = () => someOtherFunction('33')
export default getSomeValue
```

這是目前對前端工程師來說最常見的一種模組化方式，並且在 Node.js 裏面也已經[逐漸支援](https://powersandwich.com.tw/2021/08/18/react-%e5%85%83%e4%bb%b6%e5%ba%ab%e8%a3%bd%e9%80%a0%e7%b4%80%e4%ba%8b%e9%8c%84-%e3%84%a7-%e5%95%8f%e9%a1%8c%e8%aa%aa%e6%98%8e%e8%88%87%e7%ac%ac%e4%b8%80%e5%80%8b%e9%ba%bb%e7%85%a9%ef%bc%9ajavascr/#:~:text=%E9%80%99%E6%98%AF%E7%9B%AE%E5%89%8D%E5%B0%8D,%E5%A5%BD%E5%81%9A%E5%8D%80%E5%88%A5%E3%80%82)，它對我們接下來要進行的模組打包有著非常重要的地位，這點我們後面會再來提。

最後我們把各種模組系統列出來看看他們各自的用法和差別在哪邊，比較好做區別。

### 各種 Module System 比較

說明 使用環境

|                         | 說明                                     | 使用環境                      |
| ----------------------- | ---------------------------------------- | ----------------------------- |
| CommonJS ( CJS )        | 用 module.exports 、require 來定義       | Node.js                       |
| AMD                     | 用 define 、 require、export 來定義模組  | Browser                       |
| UMD                     | 判斷引用方式來決定要用 AMD 還是 commonJS | Both                          |
| ECMAScript Module (ESM) | 目前前端常見的 import 、export           | Browser (Node.js 也逐漸支援） |

### 為什麼要了解模組系統？

模組系統跟打包元件庫有什麼關係？我們為什麼在這篇文章裏面要花這麼大的篇幅來解釋模組系統呢？這個問題的答案並不是那麼明顯：原因是這個模組必須被使用在 Server Side Rendering （SSR）的架構底下。

這個 SSR 有什麼關係嗎？當然有，關係可大了。請先想想，後端 Node Server 所使用的是 commonJS 模組，而在瀏覽器端目前則通常是使用 ESM 模組系統。

先不考慮我們要打包的內容是不是 React 元件，就算是一個純 JS 的函式庫好了，同一個**JavaScript 模組系統要同時被後端 （ Node.js ) 與前端 ( Browser ）同時使用的話，不管單獨透過 ESM 、AMD、CommonJS 都是不可能的，除非透過工具處理**過。

所以如果用 CommonJS 的語法在 Node.js 裏面去 require 另一個 ESM 模組， 那麼 JavaScript 肯定會毫不留情的回傳錯誤給你。所以我們在接下來的元件模組打包有兩種可能的做法：

1. 第一種方法很單純，直接打包成上面提過可以相容兩種系統的 UMD 模組，但這樣的話在前端必須另外處理（ 需另外安裝 Module loader，如 [RequireJS](https://requirejs.org/) ) ，**不然的話就必須確定引用它的專案有用 Webpack 作處理**。
2. 第二個方法是，我們**針對前後端打包兩個不同模組版本的 JS code，並透過調整模組內 package.json 的設定**，讓該模組在被引入的時候可以辨別並根據不同的模組系統，來回傳對應的程式碼內容。

上述第二個調整 package.json 的方式具體來說怎麼做，原理又是什麼？這一點我們在下一章 Webpack 篇來說明。

### 參考文章

- [A 10 minute primer to JavaScript modules, module formats, module loaders and module bundlers (jvandemo.com)](https://www.jvandemo.com/a-10-minute-primer-to-javascript-modules-module-formats-module-loaders-and-module-bundlers/)
- [JavaScript Module Systems. Even an old topic as it is today, it is… | by E.Y. | Medium](https://elfi-y.medium.com/javascript-module-systems-e34515f1585d)
- [What is AMD, CommonJS, and UMD? – David Calhoun’s blog (davidbcalhoun.com)](https://www.davidbcalhoun.com/2014/what-is-amd-commonjs-and-umd/)
