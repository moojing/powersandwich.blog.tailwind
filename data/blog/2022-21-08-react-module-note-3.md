---
title: React 元件庫製造紀事錄 (三) – Rollup Bundler 實作，並探究 Webpack 的 Tree Shaking
date: '2021-10-12'
tags: [frontend, javascript]
layout: PostLayout
summary: '在上一篇模組化系列文章的第二篇，我們說明了使用 webpack 來打包模組程式碼、並說明了 package.json 在我們想要打包自己的 Npm 模組時扮演了什麼樣的角色。在今天這一章節，我們會繼續看到怎麼用同樣是打包工具（ Bundler ) Rollup 來進行打包，並解釋為什麼最後會選用它來進行實作的最大來由 —— Tree Shaking。'
---

## Outline

- [Rollup 打包實作](#rollup-打包實作)

- [Tree Shaking](#tree-shaking)
- [Webpack 真的有 Tree Shaking?](#webpack-真的有-tree-shaking)
- [Webpack 如何做到 Tree Shaking?](#webpack-如何做到-tree-shaking)
- [Webpack Tree Shaking 的進一步優化](#webpack-tree-shaking-的進一步優化)
- [寫在最後](#寫在最後)
- [參考資源](#參考資源)

## Rollup 打包實作

就像我們前面在 webpack 版本做過的一樣，我們必須在專案根目錄新增一個 rollup.config.js ，寫入想要的打包設定之後執行對應的 cli 指令，就能夠完成打包的基本動作。

而一個最簡單的 rollup 設定檔其實只需要輸入和輸出的設定，所以最複雜的部分通常就是弄懂你所需要的打包流程中需要用到哪些 Rollup 的 Plugin，以及他們的目的。

```js
// rollup.config.js
export default {
  input: 'components/index.js',
  output: { file: pkg.module, format: 'esm', sourcemap: true },
}
```

接著在專案根目錄底下執行

```bash
rollup -c rollup.config.js
```

就能夠進行打包的動作了。但是因為我們這個專案必須要處理 React 的程式碼，所以就需要另外經過其他工作例如 Babel 的處理 ，這時就要借助其他 Rollup Plugin 的幫忙啦，我們來看看會需要哪些 Plugin。

Rollup Plugin 說明
我們大致上會需要下列幾種 plugin :

- @rollup/plugin-commonjs
- @rollup/plugin-node-resolve
- @rollup/plugin-babel
- @svgr/rollup
- @rollup/plugin-image
- rollup-plugin-postcss
- rollup-plugin-peer-deps-external
- rollup-plugin-delete

看起來有很多個 plugin，我們一個一個來看看他們在做什麼。

### @rollup/plugin-commonjs

將 commonJS 模組系統的程式碼轉為 ESM 系統。

雖然我們在撰寫元進內容的時候所使用的都是 ESM 模組系統。但是因為還是有很多 npm 模組內的程式碼使用的是 commonJS，所以我們會需要多處理這個部分。

在 Rollup 裡面想要使用 plugin 的方式非常簡單，只要在 plugin 的屬性裡面放入對應需要的套件並已函式的方式呼叫就好：

```js
// rollup.config.js
import commonjs from "@rollup/plugin-commonjs";
...
export default {
...
  plugin:[
    commonjs({
      include: /node_modules/,
    }),
    ...
  ]
}
```

### @rollup/plugin-node-resolve

跟上一點做搭配，這個 plugin 是為了讓 Rollup 能夠順利整合用到的 npm module。

### @rollup/plugin-babel

這個 plugin 是為了讓 rollup 能夠更好的整合 babel 這個工具。如果不透過這個 plugin ，我們可能就必須要在使用 rollup 處理之前先用 babel 處理過，或是先用 rollup 處理過我們的程式碼之後再用 babel 處理。

但是這麼一來這兩個工具只能分開來進行使用，透過這個 `@rollup/plugin-babel` 才有辦法更好地和其他 plugin 之間進行組合和搭配。
`
這邊需要稍微注意一下的是，根據套件的[專案文件說明](https://www.npmjs.com/package/@rollup/plugin-babel)如果這個 plugin 和 @rollup/plugin-commonjs 同時做使用的話，必須將它放在 commonJS plugin 之後，讓他們兩個可以順利整合。

```js
// rollup.config.js
import babel from "@rollup/plugin-babel";
import commonjs from "@rollup/plugin-commonjs";
...
export default {
...
  plugin:[
    commonjs({
      include: /node_modules/,
    }),
    ...
   babel({
      exclude: ["node_modules/**"],
      babelHelpers: "runtime",
    }),
  ]
}
```

### @svgr/rollup

這是為了處理專案裡面用到的 svg 檔案，如果你的專案不需要處理 svg 的話其實不太需要這個 plugin 也沒關係。

### @rollup/plugin-image

將圖片檔案轉成 base64 的工具，通常會在這種專案裡面用到的圖片會是 icon 之類的小尺寸圖片，所以用 base64 也不會造成太大的負擔，如果你的套件模組裡面會用到大圖可能要注意一下，否則載入該模組的專案可能會在速度上受到影響。

### rollup-plugin-postcss

跟 `@rollup/plugin-babel` 的目的類似，這個套件是為了讓 rollup 更好整合 postcss 。另外，透過它我也可以將 css 輸出成另外一個單獨的檔案。

### rollup-plugin-peer-deps-external

這個 plugin 跟我們所要打包套件的程式碼內容處理比較沒關係。在一般的第三方模組專案裡面的 package.json 裡面很常看到會有一個 `peerDependencies` 的設定，這個設定的意思是「想要使用這個套件，你的專案必須安裝以下這些套件...」，也就是說預設你必須安裝 `peerDependencies` 裡面的相依模組才有辦法使用該套件模組。

因此如果你也有一些套件想要放在 `peerDependencies` 作為相依模組的其他套件，在使用 Rollup 處理的時候就可以不用打包這些模組的程式碼了，而這個 rollup plugin 就是為了讓我們可以避開這些模組不處理，它會去依照你 package.json 裡面的 peerDependencies 來避開處理。

### rollup-plugin-delete

Rollup 其實有 watch mode，當你的模組還正在進行開發的期間，可以在 rollup 指令後面加上 –watch 或是 -w ，這麼一來當 rollup config 檔案內容有所變化的時候，Rollup 就會自動為你重新打包一份新的程式碼。

```bash
rollup -c -w rollup.config.js
```

而 `rollup-plugin-delete` 會在你每次要打包程式碼之前，先去刪除你指定的檔案，如此一來就能夠確保每次打包的程式碼都是最新版本的。

以上就是每個所用到的 Rollup Plugin 的簡易說明，你可以在各個套件的 npm 或是 github 頁面找到更詳細的說明。

在 rollup config 完成後，基本上後面安裝套件的流程和動作都跟上一篇 webpack 篇差不多，就不再重複描述。想要看實際範例的人一樣可以參考我為這系列文章所做的的 [Github repo](https://github.com/moojing/react-module-bundle-demo/tree/master/packages/rollup)。

## Tree Shaking

前一篇有提到， 使用 Rollup 來進行模組的打包的話，才能將我們的程式碼打包成 ESM 模組系統，讓使用我們模組的專案裡面的 webpack 能夠順利做到 tree shaking 的效果，那麼有沒有 Tree Shaking 到底差在哪裡？又是透過什麼方式來做到的呢？下面就讓我們一起來瞭解看看。

首先，我們要先弄清楚什麼是 tree shaking ? 如果你去翻 [MDN](https://developer.mozilla.org/en-US/docs/Glossary/Tree_shaking) 上的說明，會是：

> Tree Shaking 是 JavaScript 裡面一個常用的術語，用來描述刪除 「Dead Code」這件事。

那麼，什麼是 Dead code ？ Dead code 指的就是雖然寫在專案程式碼裡面，但是實際上執行時卻不會用到的程式碼，這些程式碼對我們的應用程式、或是產品來說實際上是多餘的，其實可以去除掉。 **Tree Shaking 指的其實就是「去除沒有在使用的、多餘的程式碼」 這件事**，所以你也會聽到有人用 Dead Code Elimination （DCE） 來稱呼 Tree Shaking。

這可以說是前端效能優化的一環，透過 Tree Shaking ，我們可以幫專案進行一輪的減肥，如此一來前端在載入 JavaScript 的時候，就可以**降低拉取原始碼檔案的時間**，增快頁面啟動的速度。當然，想要做的速度上的優化也還有其他的方式，像是透過[壓縮程式碼](https://github.com/terser/terser)來減少檔案的大小、或是利用 CDN 加快資源載入速度，Tree Shaking 只是其中一種而已。

關於 Tree Shaking ，在我們繼續往下之前，可能要先弄清楚兩件事情：

- 如果是在你的專案有使用其他 module 的狀況下，通常 Tree Shaking 是專案內的 webpack 或其他 bundler 會做處理，而不是第三方的 module。
- 另外製作的第三方 module 要負責的事情主要是確保該 module 是有能力被 Tree Shaking 的，這通常代表這個 [module 必須是 ESM 系統](https://webpack.js.org/guides/tree-shaking/)。

![tree shaking doc](/static/images/blog/frontend/react-module-note-3/tree-shaking.png)

## Webpack 真的有 Tree Shaking?

這邊就以一個簡單的 React 範例來做說明。

![tree shaking example](/static/images/blog/frontend/react-module-note-3/tree-shaking-example.png)

這個專案在首頁頁面的元件引用了某個 utils 函式，這個函式又相依了另外一個 constants 檔案內的變數。它們之間的引用關係如下：

![tree shaking relation](/static/images/blog/frontend/react-module-note-3/tree-shaking-relation.png)

接著，仔細一看你就會發現有一些程式碼雖然放在專案裡面，但是其實是沒有真正被使用到的。

![tree shaking relation1](/static/images/blog/frontend/react-module-note-3/tree-shaking-relation1.png)

接著我們來實作看看，由於 Webpack 在打包 prod 程式碼的時候才會進行 Tree Shaking ，我們可以試著用 production mode 先打包試試看是不是真的會這樣。

```bash
webpack --mode=production
```

在這個專案裡面我們會將處理後的程式碼輸出到 dist 資料夾底下，接著將 Webpack 的 `mode` 調整成 production 之後並打包後，我們就可以在處理後的程式碼搜尋看看 `"Hi, This is Mujing."`這個實際上沒用到的字串。

接著你就會發現搜尋不到，因為這段程式碼已經被過濾掉了，或是說被「搖」下樹上了。

## Webpack 如何做到 Tree Shaking?

Webpack 在做 Tree Shaking 的順序大致上是這樣子的：

以入口檔案 ( entry file ) 為起點開始解析，我們在前面的篇幅有提過。
接著 Webpack 會開始依照你所有檔案之間的相依性，產生出一份相依關聯 （ 原文為 [dependency graph](https://webpack.js.org/concepts/dependency-graph/) ）。
這時候 Webpack 也會一起檢查解析的檔案中有哪些其實是沒有與其他檔案相依的，也就是沒有被其他檔案 import ，或 require。
去掉這些沒有被使用的檔案。

題外話，上述的 dependency graph ，雖然只是一種概念，並不是真的圖像，不過有興趣的人也可以照著這個 [issue](https://github.com/webpack/webpack/issues/690) 裡面提到的方法來產出自己模組圖像化後的相依關聯圖。以前一段提到的程式碼範例來看的話就會是這樣：

![dependency graph](/static/images/blog/frontend/react-module-note-3/dependency-graph.png)

上述的解析步驟在 Webpack 文件裡面並沒有那麼清楚地被寫出來，如果想要知道確切詳細流程可能必須要去翻閱原始碼才能知道，不過我們可以從 Webpack 打包完後的程式碼來看出一些端倪。

就像我們前面提到的，由於 Webpack 在 production 會做 Tree shaking ，在 development 就不會，我們可以藉由這個行為來比較一下，讓我們把 webpack config 的 mode 調整成 development 並且把 Tree Shaking 相關的設定 `usedExports` 打開。

[usedExports](https://webpack.js.org/configuration/optimization/#optimizationusedexports) 是用來讓 Webpack 判斷哪些檔案有被 import 哪些沒有的設定。在 Webpack 的 development mode 打包雖然不會發生 Tree Shaking ，但是透過設定 `usedExports` 我們可以看到 Webpack 下的標記。

```js
  // webpack.config.js
   optimization: {
      usedExports: true,
   }
```

首先我們先不設定 `usedExports` ，來打包看看，接著我們搜尋某段剛剛沒有用到的程式碼 `"Hi, This is Mujing."`。

![webpack with no usedExports](/static/images/blog/frontend/react-module-note-3/tree-shaking-result-code.png)

這邊我們可以發現 Webpack 標記了這個 `utils/index.js` 有哪些 function 被 export 。接下來讓我們把 `usedExports` 設定為 true，再來打包一版。

![webpack with usedExports set to true](/static/images/blog/frontend/react-module-note-3/tree-shaking-result-code1.png)

然後我們可以發現同樣位置的註記不一樣了！多了個 used 的標記：

> unused harmony export greeting

從這邊我們可以看到 Webpack 做 Tree Shaking 的一些蛛絲馬跡。接著如同上面提到的，如果你接著把 Webpack 的 mode 改為 production ，就會發現 greeting 這個 function 實際上不會被打包到 production 的程式碼裡面。

## Webpack Tree Shaking 的進一步優化

雖然前面有提到要確保某個第三方模組是能夠被 Tree Shaking 的話，只要使用 ESM 這個模組系統就可以，但是除了使用模組系統，我們還可以做得更好。

### SideEffects

雖然 Webpack 的 `usedExports` 搭配 ESM 的 import / export 可以透過判斷哪些 export 的檔案實際上「有沒有被使用 ( import ) 到」來做 Tree Shaking，但是單純用 import / export 來做這件事可能是不夠的。

這邊我們要再來看看另一個 Webpack 裏面 Tree Shaking 相關的設定 —— **sideEffects** 。這個設定就如其名是跟副作用有關的設定，「Side Effect（副作用）」這個詞我們平常可能會在看診的時候聽到，例如「這個藥方可能有… 跟 … 的副作用喔」。而如果再講得抽象一點的話可能就是：

> 預期目的之外的效果或動作

在我們平常使用的或是自己製作的模組裡面也有可能出現副作用，副作用有可能以各種方式出現。如果我們以一個一個檔案作為模組裡面的最小單位的話，具體來說可能就會是：

- 某段會去更改全域 window 物件的程式碼
- 如果是 React 模組，並且使用 css module ，可能會有對 DOM 插入 style tag 的程式碼
- 其他影響到自己檔案外的狀態的程式碼

Webpack 裡面的 sideEffects 設定則是為了讓 Tree Shaking 進行時能夠做得更全面，如果我們看看官網的相關說明：

> Tells webpack to recognise the [sideEffects](https://github.com/webpack/webpack/blob/master/examples/side-effects/README.md) flag in package.json or rules to skip over modules which are flagged to contain no side effects when exports are not used.

> 用來告訴 Webpack 依照 package.json 裡面的 sideEffect 欄位，在當某些 export 的程式碼沒有被使用到時，跳過那些被標記為沒有副作用的部分。

先讓我們再回頭看看前面的範例，為了方便辨識，我直接另外加入一個叫做 `sideEffect` 的檔案，並且這個檔案裡面只以 console.log 來代表一個副作用。

![Side Effect Relation](/static/images/blog/frontend/react-module-note-3/side-effect-relation.png)

所以事實上這段 sideEffect.js 裡面的程式碼是沒有在其他地方被使用到的。如果我們再不改任何設定的情況下直接打包，會看到這段程式碼還是有被打包進來的，因為我們目前還是只用 `usedExports` 來做 Tree Shaking 的判斷：

![Side Effect Code](/static/images/blog/frontend/react-module-note-3/side-effect-result-code.png)

接著我們再試著把 Webpack 裡面的 `sideEffects` 設定打開。

```js
  // webpack.config.js
...
    optimization: {
      usedExports: true,
      sideEffects: true,
    },
...
```

除了 Webpack 的設定，在專案內 **package.json 裡面也要把 sideEffects 設定為 false** ，不然這個 sideEffects 的預設值原本會是 true，意思是「專案內所有的檔案都具有副作用」，所以只要有被 import 的程式碼都會被包含在打包的程式碼內。

```json
// package.json
{
  "name": "webpack-tree-shaking-example",
  "version": "1.1.0",
  "description": "An Example for Tree Shaking.",
  "sideEffects": false,
  ...
}
```

接著在打包看看，就會發現剛剛那段 console.log 的程式碼現在不會被打包進來了，因為當我們把專案 pacakge.json 裡面的 `sideEffects` 設定為 false 後，就等於告訴了 Webpack **這個專案內「不會有 sideEffect 發生」**，所以那些被 import 但是實際上卻沒用到的程式碼就不會被打包了。

如果你的模組裡面有特定某幾隻檔案真的會有副作用發生，在 pacakge.json 裡面的 `sideEffects` 除了布林值，也可以[用陣列來表示](https://webpack.js.org/guides/tree-shaking/#mark-the-file-as-side-effect-free)。

```js
// package.json
{
...
  "sideEffects": [
    "src/utils/sideEffect.js"
  ],
...
}
```

這樣也能夠告訴 Webpack 「以下這些檔案是具有副作用的喔」。

## 寫在最後

截至目前為止是這個系列的第三篇文章了。以上這些內容就是我最近在製作第三方模組時的實作和學習紀錄。

我們先是看到了如何用最基本的方式打包模組，接著我們了解到 Webpack 做 Tree Shaking 的流程，最後了解了 `usedExports` 和 `sideEffects` 這兩個相關設定的目的和效果，讓我們有能力進一步幫專案做優化。

## 參考資源

- [How To Make Tree Shakeable Libraries | Theodo](https://blog.theodo.com/2021/04/library-tree-shaking/)
- [Deep Dive Into Tree-Shaking. In the JavaScript world, the term… | by Naveen DA | JavaScript in Plain English](https://javascript.plainenglish.io/deep-dive-into-tree-shaking-ba2e648b8dcb)
- [Webpack 中的 sideEffects 到底该怎么用？](https://segmentfault.com/a/1190000015689240)
