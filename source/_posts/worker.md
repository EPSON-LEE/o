---
title: JavaScript 中的线程与 Event Loop
date: 2019-11-28 11:56:39
tags: [JavaScript, 线程]
---

## 为什么 JavaScript 是单线程

众所周知 JavaScript 最开始是被设计运行在浏览器中的脚本语言，从浏览器的功能和特性角度来讲，单线程是最合适的。多个线程操作 DOM 的时候容易出现此类问题：这个 DOM 应该被哪个线程控制的问题？ 因此为了避免问题的复杂性，从诞生开始 JavaScript 就是单线程，这已经成为这门语言的核心特征。

随着硬件资源（具体指的是 CPU 核数）不断提升和业务场景的不断的扩大（密集型计算、游戏），需要充分利用多核能力。 HTML 5 提出 Web Worker 标准， 允许 JavaScript 脚本创建多个线程，但主线程完全控制子线程且子线程没有操作 DOM 的权限, 所以说并没有改变 JavaScript 单线程的本质。

## 执行栈与任务队列

单线程意味着排队，必须按照书写代码的顺序执行下去 A -> B -> C -> D，然而一旦哪一步执行的时间非常长就不得不进行长时间的等待，那么这个时候任务就不得不等着前面的执行完。
如果是计算量过大 CPU 忙不过来了倒也好说， 然而这时候 CPU 是闲着的，因为许多任务是比较缓慢的，例如： IO、网络，比如等到有结果才能继续执行。


那么可以把任务分为两种：同步任务（synchronous）、异步任务（asynchronous）。同步任务是指：一个任务执行完后再进行下一个任务，异步任务是说：不进入主线程而进入“任务队列（task queue）“的任务，只有“任务队列“通知主线程某个异步任务可以执行了，该任务才可以执行。

具体来说，异步执行的运行机制如下。（同步执行也是如此，因为它可以被视为没有异步任务的异步执行。）

```
（1）所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。

（2）主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。

（3）一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。

（4）主线程不断重复上面的第三步。
```
下图就是主线程和任务队列的示意图。

![](https://raw.githubusercontent.com/EPSON-LEE/image-hosting/master/20191128151057.png)

只要主线程空了，就会去读取"任务队列"，这就是JavaScript的运行机制。这个过程会不断重复。

## 事件和回调函数

"任务队列"是事件的队列，IO 设备每完成一项任务，就在“任务队列”中添加一个事件，表示相关的异步任务可以进入“执行栈”了。主线程读取“任务队列”，就是读取里面有哪些事件。

"任务队列"中的事件，除了IO设备的事件以外，还包括一些用户产生的事件（比如鼠标点击、页面滚动等等）。只要指定过回调函数，这些事件发生时就会进入"任务队列"，等待主线程读取。

所谓"回调函数"（callback），就是那些会被主线程挂起来的代码。异步任务必须指定回调函数，当主线程开始执行异步任务，就是执行对应的回调函数。

"任务队列"是一个先进先出的数据结构，排在前面的事件，优先被主线程读取。主线程的读取过程基本上是自动的，只要执行栈一清空，"任务队列"上第一位的事件就自动进入主线程。但是，由于存在后文提到的"定时器"功能，主线程首先要检查一下执行时间，某些事件只有到了规定的时间，才能返回主线程。

## Event Loop

主线程从"任务队列"中读取事件，这个过程是循环不断的，所以整个的这种运行机制又称为Event Loop（事件循环）。

![](https://raw.githubusercontent.com/EPSON-LEE/image-hosting/master/20191128152551.png)


上图中，主线程运行的时候，产生堆（heap）和栈（stack），栈中的代码调用各种外部API，它们在"任务队列"中加入各种事件（click，load，done）。只要栈中的代码执行完毕，主线程就会去读取"任务队列"，依次执行那些事件所对应的回调函数。

## 定时器

除了放置异步任务的事件，"任务队列"还可以放置定时事件，即指定某些代码在多少时间之后执行。这叫做"定时器"（timer）功能，也就是定时执行的代码。

总之，setTimeout(fn,0)的含义是，指定某个任务在主线程最早可得的空闲时间执行，也就是说，尽可能早得执行。它在"任务队列"的尾部添加一个事件，因此要等到同步任务和"任务队列"现有的事件都处理完，才会得到执行。

HTML5标准规定了setTimeout()的第二个参数的最小值（最短间隔），不得低于4毫秒，如果低于这个值，就会自动增加。在此之前，老版本的浏览器都将最短间隔设为10毫秒。另外，对于那些DOM的变动（尤其是涉及页面重新渲染的部分），通常不会立即执行，而是每16毫秒执行一次。这时使用requestAnimationFrame()的效果要好于setTimeout()。

需要注意的是，setTimeout()只是将事件插入了"任务队列"，必须等到当前代码（执行栈）执行完，主线程才会去执行它指定的回调函数。要是当前代码耗时很长，有可能要等很久，所以并没有办法保证，回调函数一定会在setTimeout()指定的时间执行。

### 定时器运行机制

setTimeout和setInterval的运行机制，是将指定的代码移出本轮事件循环，等到下一轮事件循环，再检查是否到了指定时间。如果到了，就执行对应的代码；如果不到，就继续等待。

这意味着，setTimeout和setInterval指定的回调函数，必须等到本轮事件循环的所有同步任务都执行完，才会开始执行。由于前面的任务到底需要多少时间执行完，是不确定的，所以没有办法保证，setTimeout和setInterval指定的任务，一定会按照预定时间执行。

### 特殊的定时器语句

我们经常会看到这么一些语句 setTimeout(fn, 0)，那么他的意思真的是等待 0 秒后立即执行么？ 当然不会，正如上文所说需要等到上文全部执行完, 总之，setTimeout(f, 0)这种写法的目的是，尽可能早地执行f，但是并不能保证立刻就执行f。

### 应用

## Node.js 中的 Event Loop

![](https://raw.githubusercontent.com/EPSON-LEE/image-hosting/master/20191128155205.png)

根据上图， Node.js 的运行机制

```
（1）V8引擎解析JavaScript脚本。

（2）解析后的代码，调用Node API。

（3）libuv库负责Node API的执行。它将不同的任务分配给不同的线程，形成一个Event Loop（事件循环），以异步的方式将任务的执行结果返回给V8引擎。

（4）V8引擎再将结果返回给用户。
```

## 问题遗留

process.nextTick process.setImmediate setTimeout 的区别