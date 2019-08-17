### 单线程的 Javascript

接触过 JS 的同学都知道，JS 是一门单线程语言，它不像 Java 一样可以提供多线程来执行任务，所有任务只能在单线程上一个一个执行。

打个比方，假设我们去营业厅办理业务，这个营业厅只有一个窗口，那所有客户都只能在这个窗口排队，即使你前面的人需要耗费很久的时间，你依旧要等前面所有人办理完了才轮到你。

1. 为什么 Javascript 是单线程的？

举个例子，假如现在有两个线程同时执行，对同一个 Dom 节点进行操作，那浏览器应该以哪个线程为准呢？

2. Javascript 为什么需要异步？

当我们打开一个网站时，很多任务开始执行，比如页面骨架加载、图片音乐加载、服务端数据请求等等。假设所有的任务都是同步执行，就很容易出现页面加载时间很长的情况，甚至当资源加载报错阻塞了代码的执行，会出现页面加载不出来的情况。

3. Javascript 如何实现异步？

通过事件循环（Event Loop），理解了事件循环，就理解了 JS 的执行机制

### Javascript 事件循环
![image](https://user-gold-cdn.xitu.io/2017/11/21/15fdd88994142347?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

从上图可以看到
- 同步任务直接进入主线程，异步任务进入 Event Table 并注册函数
- Event Table 将注册的函数移入 Event Queue
- 当主线程所有任务执行完毕后，会到 Event Queue 中读取对应的函数，放到主线程中执行
- 上述步骤是不断重复的，也就是我们说的事件循环（Event Loop）

再举个例子简单说明

```javascript
console.log(1);
$.ajax({
    url: xxx,
    data: {},
    success: () => {
        console.log(2);
    }
});
console.log(3);         // 输出顺序 1 3 2
```
因为 JS 是单线程的，上述代码我们从一行往下执行。
- 第一行代码是同步任务，进入主线程执行，输出 1
- 往下执行第二行代码，ajax 是个异步任务，这个 ajax 任务进入 Event Table 并注册回调函数 success
- 到了最后一行代码，也是同步任务，在主线程中执行，输出 3
- 此时主线程的任务全部执行完毕，主线程从 Event Queue 中读取回调函数 success 并执行，输出 2

### 异步任务
除了广义的同步任务和异步任务，任务还有更精细的定义：
- macro-task(宏任务)： 整体的 script 代码、setTimeout、setInterval
- micro-task(微任务)： Promise、process.nextTick

#### setTimeout
通常我们会使用 setTimeout 来让函数延时执行

```js
setTimeout(() => {
    console.log('3秒后输出');
}, 3000);
```

正常我们的理解是，这个函数会在 3 秒后执行并打印信息。但事实是这样吗？他一定会在 3 秒后准时准点打印出来吗？其实并不是！

更严谨的说法是，当执行到 setTimeout 这个异步任务是，会将它放入 Event Table 中，3 秒后这个异步任务将对应的函数注册到 Event Queue 中，这时候只有当主线程是空闲的，这个函数才会执行，如果主线程有个任务的执行超过了 3 秒，那这个函数依旧需要等到主线程全部执行完，才轮到它。

```js
setTimeout(() => {
    console.log('3秒了，还没轮到我')
}, 3000);

sleep(10000);
```

sleep 函数在主线程中，执行需要 10s 的时间，所以 setTimeout 里的函数只能等主线程中的 sleep 执行完毕后才会执行。

通过上述例子我们也不难理解，`setTimeout(fn(), 0)` 并不是立即执行的意思，而是当主线程空闲时最早执行。不过即使主线程为空，0ms 实际上也是做不到的，HTML 的标准最低为 4ms。

#### setInterval
setInterval 跟 setTimeout 的执行原理差不多，只不过 setInterval 是循环执行的。对于 `setInterval(fn, ms)` 来说，我们知道不是每过 ms 秒会执行一次 fn，而是每过 ms 秒，会有 fn 进入 Event Queue 中。

#### Promise
从这里开始我们将涉及宏任务和微任务的概念，不同类型的任务会进入对应的 Event Queue

事件循环的顺序决定了 js 代码的执行顺序。进入整体的 script 代码（宏任务）后，开始第一次循环，执行完主线程任务后接着执行所有的微任务。然后再次从宏任务考试，从宏任务 Event Queue 中读取一个宏任务执行完毕，再执行微任务 Event Queue 里所有任务。一直循环下去。

![image](https://user-gold-cdn.xitu.io/2017/11/21/15fdcea13361a1ec?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

先看完这张图，然后我们从一段代码来理解上述的顺序

```js
setTimeout(function() {
    console.log('setTimeout');
})

new Promise(function(resolve) {
    console.log('promise');
}).then(function() {
    console.log('promise-then');
})

console.log('console');
```

- 上述代码作为宏任务进入主线程
- 执行 setTimeout，进入 Event Table 将其回调函数注册后分发到宏任务的 Event Queue 中
- 执行 Promise，`new Promise` 立即执行，控制台输出 promise，`then` 函数被注册分发到微任务的 Event Queue 中
- 执行最后一行代码，立即执行，控制台输出 console
- 至此，第一轮的宏任务执行完毕，此时我们看看有哪些微任务可以执行，`then` 函数在微任务的 Event Queue 中，执行，控制台输入 promise-then
- 第一轮事件循环结束了，紧接着开始第二轮，我们从宏任务的 Event Queue 找到了 `setTimeout` 注册的回调函数，执行，控制台输出 setTimeout
- 事件循环结束

#### process.nextTick(callback)
process.nextTick(callback) 类似 node.js 版的 setTimeout，在事件循环的下一次循环中调用 callback 回调函数。笔者对 nodejs 没有太多了解，暂不做过多描述

### 总结
- Javascript 是一门单线程语言
- Event Loop 是 Javascript 的执行机制


### 参考
- [这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/59e85eebf265da430d571f89)