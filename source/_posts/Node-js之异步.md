---
title: Node.js之异步
date: 2019-01-24 16:40:29
tags:
---

﻿这几天大致学习了Node.js的基本操作，本想记录一下操作的内容。在翻看资料书的过程中发现作者在描述Node的特点的时候，着重强调了异步I/O。
在介绍异步I/O前，有几个基本概念需要了解一下。

> 进程：一段程序的执行过程。是操作系统动态执行的基本单元，在传统的操作系统中，进程既是基本的分配单元，也是基本的执行单元。
> 进程有三个状态:就绪、运行和阻塞。

> 单线程：一个进程中只有一个线程。程序顺序执行，前面的执行完，才会执行后面的程序。我们所学的JavaScript就是单线程的，也叫做主线程。

> 多线程：一个进程中有多个线程。多线程是为了同步完成多项任务，不是为了提高运行效率，而是为了提高资源使用效率来提高系统的效率。

异步在我们学习Ajax的时候就有所接触。当客户端向服务端发送请求后，直到服务器端进行响应，这个过程中，用户可以做任何其他事情(不等待)。这与我们所知道的JS为单线程好像有点出入，这种类似“多线程”的感觉实际大有文章。

## 为何JS能实现异步

浏览器有三个进程

- 浏览器进程（Browser进程）
- GPU进程：用于3D绘制等
- 浏览器渲染进程（Renderer进程，内部是多线程的）

浏览器渲染进程

- js引擎线程
  JS内核，负责处理Javascript脚本程序。（例如V8引擎）JS引擎一直等待着任务队列中任务的到来，然后加以处理，一个Tab页（renderer进程）中无论什么时候都只有一个JS线程在运行JS程序
- 界面渲染线程
  作用：渲染页面(js可以操作dom，影响渲染，所以js引擎线程和UI线程是互斥的。js执行时会阻塞页面的渲染。)
- 浏览器事件触发线程
  作用：控制交互，响应用户。归属于浏览器而不是JS引擎，用来控制事件循环。当对应的事件符合触发条件被触发时，该线程会把事件添加到待处理队列的队尾，等待JS引擎的处理
- http请求线程
  在XMLHttpRequest在连接后是通过浏览器新开一个线程请求。将检测到状态变更时，如果设置有回调函数，异步线程就产生状态变更事件，将这个回调再放入事件队列中。再由JavaScript引擎执行。
- 定时触发器线程
  作用：setTimeout和setInteval
- 事件轮询处理线程 
  作用：轮询消息队列，event loop

 实现异步其实是js代码的执行（Event Loop）与其他线程之间的合作。JavaScript 引擎并不是独立运行的，它运行在宿主环境中，对多数开发者来说通常就是Web 浏览器。

## 异步的过程

```
	fs.readFile('node.txt' ,function(err,data){
				console.log(data)
	})
```

在这个代码片中，我们想要得到文件中的内容并且输出它，这段过程需要时间，但它不会傻等I/O语句结束，而会执行后面的语句，即node.js采用单线程异步非阻塞模式。

## 构成异步过程的细节

以上的代码可以称作异步函数的调用。但整个异步过程没有结束。

> 主线程发起一个异步请求，相应的工作线程接收请求并告知主线程已收到(异步函数返回)
> 主线程可以继续执行后面的代码，同时工作线程执行异步任务
> 工作线程完成工作后，通知主线程
> 主线程收到通知后，执行一定的动作(调用回调函数)。

工作线程：除了主线程之外的线程，比如处理AJAX请求的线程、处理DOM事件的线程、定时器线程、读写文件的线程(例如在Node.js中)等等。
要知道，在发起一个异步函数后，才能执行接来的过程。

**fun(参数1，参数2，callback())**
这里fun()就是发起函数，callback()就是回调函数

## 事件循环和消息队列

当异步函数执行完毕后，主线程需要知道异步函数执行完了并且处理它。那么，我怎么怎么判断主线程该执行哪个函数呢？
**消息队列：一个 JavaScript 运行时包含了一个待处理的消息队列。消息队列是一个先进先出的队列，它里面存放着各种消息，每一个消息都关联着一个用于处理这个消息的函数。**
那么消息队列中放的是什么东西呢？
消息就是注册异步任务时添加的回调函数。

```
$.ajax('http://segmentfault.com', function(resp) {
    console.log('我是响应：', resp);
});

// 其他代码
...
...
...
```

主线程发出了AJAX请求后，会继续执行其他代码。而AJAX线程负责请求http://segmentfault.com，当拿到相应后，它会把响应封装成一个JS对象（请求对象），然后会构造一条消息。

```
// 消息队列中的消息就长这个样子
var message = function () {
    callbackFn(response);
}
```

其中callbackFn就是前面代码中得到成功响应时的回调函数。

**消息与事件：可以说，消息队列中的每条消息实际上都对应着一个事件**

```
var button = document.getElement('#btn');
button.addEventListener('click', function(e) {
    console.log();
});
```

从事件看,代码表示：在按钮上添加了一个鼠标单击事件的监听器。当用户点击按钮的时候，鼠标单击事件被触发，事件监听函数（即这里的funtion()）被调用。
从异步看：addEventListener就是异步过程的发起函数，事件监听函数就是异步过程的回调函数。让事件触发时，表示异步任务完成了，会将事件监听器函数封装成一条消息放到消息队列中，等待主线程执行。

*注:setTimeout 并不会在计时器到期之后直接执行。如果队列中没有其它消息，在这段延迟时间过去之后，消息会被马上处理。但是，如果有其它消息，setTimeout 消息必须等待其它消息处理完。*

**事件循环：事件循环是指主线程重复从消息队列中取消息并执行的过程**

```
while (队列中有事件) {
  	取出事件及其相关的回调函数并执行
  	进入下个循环
}
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
如果当前没有任何消息，queue.waitForMessage() 会同步地等待消息到达。
```

事件循环就类似于这样的while循环。

综上所述:主线程在执行完当前循环中的所有代码后，就会到消息队列取出这条消息(也就是message函数)，并执行它。到此为止，就完成了工作线程对主线程的通知，回调函数也就得到了执行。如果一开始主线程就没有提供回调函数，AJAX线程在收到HTTP响应后，也就没必要通知主线程，从而也没必要往消息队列放消息。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190122004447938.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2d4eXFuNjI2,size_16,color_FFFFFF,t_70)

## 小结

同步可以保证顺序一致，但是容易导致阻塞；
异步可以解决阻塞问题，但是会改变顺序性。
Node.js有着非阻塞IO和事件驱动的特点
由于NodeJS是单线程运行的，首先会顺序执行js文件中的代码，此时事件循环是暂停的。setTimeout和读文件的操作都是异步操作，异步函数会在工作线程执行，当异步函数执行完成以后，将回调函数放入消息队列。当js文件执行完成以后，事件循环开始执行，并从消息队列中取出消息，开始执行回调函数。

