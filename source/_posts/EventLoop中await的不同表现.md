---
title: Event Loop 中 await 的不同表现
date: 2019-08-13 10:00:32
tags: 
- EventLoop
categories: 前端
copyright: true

---

![](https://imgs-1253349937.cos.ap-guangzhou.myqcloud.com/blog-imgs/computer-1245714_1920.jpg)

这篇文章源于一道烂大街的头条面试题，题目主要考了 Event Loop 中 async/await 对原本的事件循环造成的影响。本来看了一些解析也没啥，结果上浏览器一跑，结果错了=-=。

<!-- more -->

很奇怪，网上这道题的解析还有一些讲解  Event Loop + async/await 的文章基本都错了，为啥呢？？

咱来好好探究一下（原因很简单，规范变了，具体看最底下，不过本文主要讨论新规范下的具体表现）。

## 事先准备

讲道理，关于 js 的异步执行顺序、宏任务、微任务这些，或者 async/await 这些慨念已经有非常多的文章写了。

这里就不多讨论这些东西。

不过看这篇文章前最好还是搞清楚这些基本概念，不懂的先看看这里:

- [图解搞懂JavaScript引擎Event Loop](<https://juejin.im/post/5a6309f76fb9a01cab2858b1>) 讲的还比较清楚，不过没涉及到 async/await
- [async](<http://es6.ruanyifeng.com/#docs/async>) 再看看阮老师的 es6 了解一下 async/await

## 主要内容

### 先来看看题目

只讨论浏览器环境，node 环境事件循环机制不同。

题目如下:

```javascript

    async function async1() {
        console.log( 'async1 start' )
        await async2()
        console.log( 'async1 end' )
    }
    
    async function async2() {
        console.log( 'async2' )
    }
    
    console.log( 'script start' )
    
    setTimeout( function () {
        console.log( 'setTimeout' )
    }, 0 )
    
    async1();
    
    new Promise( function ( resolve ) {
        console.log( 'promise1' )
        resolve();
    } ).then( function () {
        console.log( 'promise2' )
    } )
    
    console.log( 'script end' )

```

如果你的结果如下:

```javascript
    script start
    async1 start
    async2
    promise1
    script end
    promise2
    async1 end
    setTimeout
```

那恭喜你，**现在**是错的（意思就是以前是这个结果没错=-=）。

在 76.0.3809.100（正式版本）的 chrome 中跑一下，正确结果如下:

```javascript
    script start
    async1 start
    async2
    promise1
    script end
    async1 end
    promise2
    setTimeout
```

图片在这里

![](https://imgs-1253349937.cos.ap-guangzhou.myqcloud.com/blog-imgs/20190813190015.png)

### 新旧结果的对比

大家都知道 await 会阻塞语句的执行，在打印 `async1 start` 之后，开始执行函数 ***async2***，此时进入函数内部打印出 `async2`，在阻塞的函数执行完之后，js 引擎会跳过这个执行上下文中后面的语句，先执行完所有的宏任务，此时相继打印出 `promise1`、`script end`。

然后呢？？分歧就来了！

后两步旧的结果跟新的结果打印的完全相反！

旧的结果打印 `promise2`、`async1 end`，意思就是说执行完宏任务之后先执行微任务，然后再执行被阻塞的语句。

这不就完事了呗！新的规范无非就是把这种情况的执行顺序调换了一下，所以才出现相反的结果。

no! no! no!

没有那么简单=-=

### 新规范下不同的表现

在网上看了那么多关于 Event Loop + async/await 的文章，发现一个问题，同一套代码（细微差别）执行结果不一样。那些文章基本都是几年前的，我试了很多次，终于摸索到了问题在哪里（就是那些细微差别）。

现在我们将函数 async2 改成这样:

```javascript
    async function async2() {
        console.log( 'async2' )
		    return new Promise((resolve, reject)=> {
			      resolve()
		    })
    }
```

再放到浏览器里面执行一下，打印啥？

![](https://imgs-1253349937.cos.ap-guangzhou.myqcloud.com/blog-imgs/20190813185728.png)

咦？咋结果跟旧规范的结果一样呢？

注意我们改的两行代码，有啥差别？

好像就多了一个返回 promise。嗯，那原因就在这里了。

其实在我们添加返回之前，函数默认返回的是 undefined，现在是返回 promise。对于被 async 修饰的函数来说，最终一定是会返回一个 promise，但是结果也有点不同。

- 返回非 promise: 此时返回结果会被包装成一个 promise，注意，这是个新的 promise，也会直接进入微任务列表。而 await 之后的语句会被加入到这个 promise 的 resolve 也就是回调中执行。这个 promise 先于后面 `promise2` 所在的 promise 进入微任务，所以会先执行，结果就是打印顺序是 `async1 end`、`promise2`。
- 返回 promise: 此时由于本身就是返回的 promise（假设叫 p1），p1 会立即加入微任务列表，新的 promise 还是会被生成（叫 p2），await 之后的语句会被加入到 p2 的 resolve 也就是回调中。而 p2 会在 p1 的回调执行时被加入微任务，假设外面那个 promise 叫 p3，那么执行顺序为 p1、p3、p2。

所以现在就有一个简单好记的结论：**被 await 阻塞的函数返回值只有为 promise 的时候，先执行微任务，然后才执行被阻塞的语句。**

## 写在最后

chrome 提交了优化 ECMAScript 的变更，就是[针对 await 规范的变动](<https://github.com/tc39/ecma262/issues/694>)。可以总结如下:

1. await 后面不一定会创建新的微任务，取决于await 后面是立即返回还是promise。
2. await 执行之后不会强制创建新的微任务，而是继续执行。

以上两种修改会导致同步调用两个async函数时，执行权交换顺序会发生改变。
但是会提高性能。

经过对问题的质疑、求解，还是能收获很多东西的，毕竟实践是检验真理的唯一标准=-=。

参考资料:

- [8张图帮你一步步看清 async/await 和 promise 的执行顺序](https://segmentfault.com/a/1190000017224799)
- [JS task到底是怎么运行的](https://github.com/rhinel/blog-word/issues/4#)
- [async await 和 promise微任务执行顺序问题](https://segmentfault.com/q/1010000016147496)