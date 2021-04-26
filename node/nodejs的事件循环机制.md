## Node.js 的事件循环机制

[Node.js 的官网](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)中有对事件循环机制的详解，但是看完依然会觉得费解：

**问题一：**

在 timers 的介绍中，timers 阶段会执行 setTimeout 和 setInterval 的回调，但是实际上的执行又是由 poll 决定的，具体怎么决定的，官网解释如下：

> there are no more callbacks in the queue, so the event loop will see that the threshold of the soonest timer has been reached then wrap back to the timers phase to execute the timer's callback

当 poll 阶段没有其他回调的时候，如果有到达阈值的 timers 回调，那么就会回到 timers 阶段来执行。

仔细一想：是直接从 poll 进入到 timers 阶段吗？后面的 check , close callback 还执行吗？



**问题二**

宏任务、微任务和事件循环的关系是什么？



先看问题一：

### 事件循环机制

看到网上一篇事件循环源码的文章https://huangtengxiao.gitee.io/post/Nodejs%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF.html，对照源码清晰的解释了整个过程。学习过后总结一下。


下图是官网中事件循环机制的图：

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

每个阶段依次执行，一个阶段完成后进入下一阶段。源码中也是依次调用各个阶段的函数：

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
	//...
  int r;
  while (r != 0 && loop->stop_flag == 0) {
    //更新时间
    uv__update_time(loop);
    
    //timer阶段
    uv__run_timers(loop);
    
    //pending 阶段
    ran_pending = uv__run_pending(loop);
    
    //idle prepare阶段
    uv__run_idle(loop);
    uv__run_prepare(loop);
    
		//计算timeout，timeout 的结果会作为 poll 阶段可等待的最大时间
    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);
    
	  //poll阶段
    uv__io_poll(loop, timeout);
    
    //check阶段
    uv__run_check(loop);
    
    //close阶段
    uv__run_closing_handles(loop);
    
	  //...
    r = uv__loop_alive(loop); // 检查 loop 是否还存在
  }
```

所以说在循环中**每个阶段依次执行，不会跳过某个阶段**。上面问题一中，check 和 close callback 是不会被跳过的。那么 timers 阶段和 poll  的关系是啥，为什么官网说

> Technically, the [**poll** phase](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#poll) controls when timers are executed.

要清晰的理解这个问题，需要从源码层面看 timers 和 poll 阶段具体做了啥

**先看 timers 阶段的执行过程：**

获取到时间最近的一个回调 

—> 判断一下是否已经到了执行时间 

​       —> 到了就执行

​       —>不到的话，就退出 timers 阶段

例如下面的代码

```
var timer1 = setTimeout(() => {
  console.log(1)
}, 300)
var timer2 = setTimeout(() => {
  console.log(2)
}, 200)
```

当第一次进入事件循环的时候，进入 timers 阶段，获取到 timer2 对应的回调。它设定的 timeout 时间 200ms，代码从开始执行到第一次事件循环开始还不到 200ms，所以 timers 阶段在这时没有要立即执行的回调。那么就进入下一个阶段。



经过了 pending callbacks/ idle prepare 阶段，进入到了 poll 阶段

**poll阶段**

poll阶段的循环执行回到的核心逻辑：参考文章 https://segmentfault.com/a/1190000022449328

>以 `linux` 的 `epoll` 轮询机制为例，在 [uv__io_poll](https://github.com/libuv/libuv/blob/v1.35.0/src/unix/linux-core.c#L309-L312) 函数体中调用了系统底层 [epoll_wait](http://refspecs.linux-foundation.org/LSB_4.0.0/LSB-Core-generic/LSB-Core-generic/libc-epoll-wait-1.html) 函数来实现 `libuv` 的轮询核心功能：
>
>```
>nfds = epoll_wait(loop->backend_fd,
>                        events,
>                        ARRAY_SIZE(events),
>                        timeout);
>
>```
>
>从 [epoll_wait](http://refspecs.linux-foundation.org/LSB_4.0.0/LSB-Core-generic/LSB-Core-generic/libc-epoll-wait-1.html) 文档描述，当 `timeout` 传参为 `0` 时，将立即返回，当 `timeout` 传参为 `-1` 时，将无限制阻塞，直到某个事件触发或无限阻塞状态被主动打断。

上面控制 poll 阶段结束的变量相关的有一个 timeout。它定义了 poll 阶段的可等待时间。 在 timeout 允许的时间内，会一直停在 poll 阶段。假如有一个 fs.readFile 的方法，在进入 poll 阶段时文件读取还未完成，回调还没触发。epoll_wait 会在这里等待一段时间，如果这个期间内 readFile 的回调触发了就会被加入执行的队列，然后执行。如果直到 timeout 时间到了，回调还未触发，那么 poll 阶段结束，进入下一个阶段。

timeout 是在 poll 阶段开始的时候计算得出的，方法在 `uv_backend_timeout` 中：

- 某些停止标记为 true 时，timeout = 0

- Idle,pending,close 阶段有要执行的回调 ,  那么 timeout = 0

- 上面都没有时，则根据 timers 中最小值来设定，也就是我们距离设定的 setTimeout 和 setInterval 要执行的最近的时间。如果没有timers，那么返回 -1，也就是说 timeout 为无限大

上面第三点决定了 poll 和 timers 的关系。假设下面的代码

```
setTimeout(() => {
  console.log('1')
}, 2000)
fs.readFile('path', {/*options*/}, () => {
	console.log('readFile done')
})
```

假设执行 readFile 需要 1000ms

到 timer 阶段的时候，setTimeout 的时间并没有到，timer 阶段结束，进入后面的阶段。然后在 poll 阶段开始前，计算 timeout 的时间：假定时间过去了 100ms, 距离 setTimeout 的时间还有 1900ms，那么 timeout = 1900。进入 poll 阶段，这个时候 readFile 还没结束，poll 会在这里等待，到1000ms readFile 进入回调，开始执行回调。如果 readFile 执行需要 2000 ms，也就是说到1900ms上限时间后，还未进入回调。那么 poll 阶段就结束了，进入后面的阶段，经过 check, close, 再进到 timers，然后执行 setTimeout 的回调



现在看问题二

### 微任务/宏任务

官方介绍， process.nextTick 时说

>This is because `process.nextTick()` is not technically part of the event loop. Instead, the `nextTickQueue` will be processed after the current operation is completed, regardless of the current phase of the event loop. Here, an *operation* is defined as a transition from the underlying C/C++ handler, and handling the JavaScript that needs to be executed.

在 libuv 底层并没有微任务这个概念，nextTick 是 nodejs 利用技术的方式专门实现的。那么 nodejs 是怎么在 libuv 的宏任务过程中运行微任务的呢？

下面文章中 http://acemood.github.io/2016/02/01/event-loop-in-javascript/  和[Understand-nodejs](https://github.com/theanarkh/understand-nodejs)的第十一章，都大概介绍了 nextTick 的实现原理。大概总结一下：

libuv 事件循环的阶段中，队列中有需要处理的事件回调时候，会调用 `AsyncWrap` 中的 `MakeCallback`方法，这个方法可以大概理解成我们所有写的回调，最终都会被封装成这样一个回调函数。在 MakeCallback 中，最后会有一个 `InternalCallback.Close`，这个 Close 中会检查是否有 nextTick 的回调，并执行。

> 每次执行 js 层的回调的时候，就会处理 tick 任务

(注：网上很多文章提到 nextTick 的执行是在”任务切换的间隙“，当初被这个说法误导了很久，一度以为是事件循环从一个阶段切换到下一阶段的时候才会执行 nextTick，源码中找了很久也没发现怎么控制的。最后才意识到不是找不到，是文章中描述错了，用下面例子验证：

```javascript
setImmediate(() => {
  console.log(1)
  process.nextTick(() => {
    console.log(33)
  })
})
setImmediate(() => {
  console.log(2)
})
setImmediate(() => {
  console.log(3)
})
```

setImmediate是在 check 阶段，如果按照”任务切换间隙“的说法，输出应该是 1,2,3,33。但是实际是 1,33,2,3。

)



资料：

https://github.com/theanarkh/understand-nodejs

http://acemood.github.io/2016/02/01/event-loop-in-javascript/

https://huangtengxiao.gitee.io/post/Nodejs%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF.html

https://segmentfault.com/a/1190000022449328