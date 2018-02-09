话不多说，先上主角。

```js
new Promise((resolve) => {
  console.log(1)
    
  process.nextTick(() => {
    console.log(2)
  })
  
  setTimeout(() => {
    console.log(3)
  }, 0)

  resolve()

  process.nextTick(() => {
    console.log(4)
  })

  setImmediate(() => {
    console.log(5)
  })
  
  console.log(6)
}).then(() => {
  console.log(7)
})

setTimeout(() => {
  console.log(8)
}, 0)

console.log(9)
```

问：上面这段代码的输出结果是多少？

其实这个例子也是从别人的文章中看到的，但是原文作者的理解和我的理解略有出入，故在此表达下自己的看法，若有错误的地方，请尽情指出~

第一眼看到这个例子，我也是一脸懵逼，这™什么玩意😠

然后就是各种查资料，各种看。其中帮助最大的就是阮老师的那篇文章和朴灵老师对此文章的批注，经过这两篇文章的洗礼，逐渐有了自己的思路。然后又经过了官方文档的净化才成就了这篇文章（其实主要是官方文档🌹），不过其中应该还是有理解不到或者理解有误之处，请多多谅解❤

首先来介绍一下nodejs中的event loop。

<img src="./img/node event-loop.png" style="display:block;margin:0 auto;" />
<p style="margin-top:10px;text-align:center;font-size:12px;">node官方文档关于event loop的流程图</p>

event loop按照执行顺序包括六个阶段（实际更多，只是node和我们说：你们不需要关心那些😊）：

> 1. timers，这个阶段执行`setTimeout`和`setInterval`的回调函数
> 2. I/O callbacks，执行几乎所有的回调，除了close回调，timer的回调，和setImmediate()的回调
> 3. idle,prepare，node内部使用，不用管
> 4. poll，后面会详细说明
> 5. check，这个阶段执行`setImmediate`的回调函数
> 6. close callbacks，关闭回调的执行阶段，例如socket的close事件

每个阶段都有一个先进先出（FIFO）的队列用来放置回调函数，每个阶段队列中的回调函数执行完或者到达最大执行数（node内部定义了每个阶段可执行的最大回调函数的数量，为了避免当前程序一直卡在某个阶段太久导致其他阶段的回调函数得不到执行的饥饿现象）之后，会进入下个阶段，继续执行下个阶段队列中的回调函数，这样循环往复就形成了nodejs的event loop。

在这些阶段中最特殊的就是poll这个阶段，也是整个event loop的核心（个人认为）。

在这个阶段node主要会做两件事：

> 1. 执行到达时间的定时器脚本（此为官方说法，个人理解仅仅检查是否到达时间，具体执行还是在timer或者check阶段做）
> 2. 执行本队列的回调函数

如果在进入poll阶段的时候，没有定时器到达规定时间，那么以下两种情况之一将会发生：

> 1. 如果poll队列中有回调函数，那么这些回调函数将会按顺序同步执行直到执行完所有函数或者到达指定的最大数量（之前有提到），然后会进入下一个check阶段。
> 2. 如果poll队列中没有回调函数，那么又会有一下两种情况之一会发生：
>> 1. 如果此时有被`setImmediate`的脚本，那么event loop将会结束当前阶段然后进入check阶段来执行被`setImmediate`的脚本。
>> 2. 如果没有被`setImmediate`的脚本，那么event loop会在此阶段等待回调函数的到来（比如在异步读取文件成功后会将回调函数插入到poll的队列中）并立即执行。

一旦poll队列空了，event loop便会检查是否有定时器到达时间，如果有，event loop会顺序进入下一阶段直到进入timers阶段将到达时间的回调函数依次执行。

其实，基于整个event loop，本文所有的解释都有一个重要的前提条件：`所有主程序代码（就是除了异步代码之外的代码）都是在event loop初始化之前跑完，然后才开始初始化event loop`。这个前提条件目前只是假设，但是个人经过不断试验，认为该假设是成立的。其实要验证这个假设成立与否很简单，去看node源码就好啦，然而我并没有去看。

介绍完了基础知识，然后说说`setTimeout`、`setImmediate`和`process.nextTick()`的区别。

> * `setTimeout`，作用是将一个匿名函数（即回调函数）推迟一定时间执行。设置一个定时器计时，在到达指定时间后，会将被`setTimeout`的回调函数放入timers队列中等待执行。有一点需要注意的是，node端`setTimeout`设定的最小时间是`1`，如果设为`0`，node会把它当做`1`。
> * `setImmediate`，作用是异步立即执行一个匿名函数（即回调函数）。此函数不会设置定时器，回调函数会被立即放入check队列中，在poll阶段结束后会进入check队列依次执行当中的回调函数。
> * `process.nextTick()`，作用也是异步立即执行一个匿名函数（即回调函数）。但是这个方法和`setImmediate`完全不一样，`process.nextTick()`不属于event loop中的任何一个阶段，也可以说任何一个阶段中都有可能出现它的身影。node中有`nextTickQueue`这样一个队列专门存放被`process.nextTick()`处理的回调函数，在当前操作完成之后（个人理解并证实的结果是在当前阶段结束之后，下个阶段进入之前）就会将`nextTickQueue`中的回调函数依次执行完并清空`nextTickQueue`。

其实这是一个危险的设定，比如说下面这个例子：

```js
function tick() {
  process.nextTick(() => {
    tick()
    console.log('n')
  })
}

tick()

setImmediate(() => {
  console.log('immediate')
})
```

`immediate`永远得不到输出，event loop会被永远的堵塞在`tick`函数执行的时候event loop所处的阶段，然后无限输出`n`，并且不会出现`Maximum call stack size exceeded`，因为每次都是执行完一个tick中的匿名函数再执行新的，不会出现溢出。

关于网上常说的`宏任务`、`微任务`的概念，个人认为这个概念仅仅是对`promise`、`setTimeout`、`setImmediate`、`process.nextTick()`执行效果的总结，因为`setTimeout`、`setImmediate`的回调都是在之后的event loop周期中执行，`promise`、`process.nextTick()`的回调是可以在当前event loop周期中执行的。官方没有任何关于这个概念的提及，只能说这个概念可以帮助理解或记住这四种操作的执行方式，但是实际上并不是依据这个概念来实现操作的，故本文不是基于这个概念来展开说明，而是更贴合真实机制。

大致原理了解了之后咱们来看几个小例子（其实难度并不小）

example1
```js
setTimeout(() => {
  console.log(1)
}, 0)

setImmediate(() => {
  console.log(2)
})
```

复制-粘贴-执行-执行-执行...经过反复执行，你会发现会有`1-2`，`2-1`两种结果。
来尝试分析一下原因：在代码执行的时候，由于机器性能原因和`setTimeout`的最小时间值，不确定event loop进入`timers`阶段之前定时器有没有到达指定时间，如果已经到达，那么进入`timers`阶段的时候会输出`1`，然后进入`check`阶段输出`2`；反之，则会出现`2-1`，所以造成了这种不确定的结果。

example2
```js
const fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log(1)
  }, 0)
  setImmediate(() => {
    console.log(2)
  })
})
```

如果是这样子呢？由于`setTimeout`，`setImmediate`这两条命令执行的阶段固定在了`poll`阶段，处于`check`阶段之前，所以这次的输出结果会固定为`2-1`。

example3
```js
setTimeout(() => {
  console.log(1)
  process.nextTick(() => {
    console.log(3)
  })
}, 0)

setTimeout(() => {
  console.log(2)
}, 0)
```

执行这段代码的时候，由于都是`setTimeout`操作，所以在初始化event loop之前肯定是将两个回调函数都放入`timers`的队列中。正常流程下，按照之前介绍的event loop机制，应该是`1-2-3`这样一个输出顺序，但是实际执行还会有`1-3-2`的情况出现。这是因为之前说的`setTimeout`的最小时间是`1`，在第一个`setTimeout`到达事件并将回调插入`timers`队列后，可能第二个`setTimeout`的定时器还没有到时间，此时event loop进入`timers`这个event loop的起始阶段，开始执行第一个回调函数，此后即使第二个`setTimeout`到了时间，回调函数也会推到下一个event loop周期中执行，于是`nextTick`的回调会优先于第二个`setTimeout`的回调函数执行，这就导致了`1-3-2`的结果。

example3
```js
setImmediate(() => {
  console.log(1)
  process.nextTick(() => {
    console.log(3)
  })
})

setImmediate(() => {
  console.log(2)
})
```

这个例子将上个例子中的`setTimeout`改为了`setImmediate`，就不会出现上述问题了，因为`setImmediate`并不会使用定时器，而是立即将回调函数插入`check`阶段的队列中，于是总是得到`1-2-3`的结果。

example4
```js
setImmediate(() => {  // a
  setImmediate(() => {  // b
    console.log(1)
    setImmediate(() =>{  // c
      console.log(2)
    })
  })

  setTimeout(() => {  // d
    console.log(3)
  }, 0)
})
```

可以看到，这个例子会出现`1-3-2`和`3-1-2`两种结果，也从侧面佐证了上述结论。首先，`a`的执行是在`check`阶段，`b`会在下一个event loop周期的`check`阶段执行，`c`会在下下个event loop周期的`check`阶段执行，而`d`由于上面描述的原因，可能会在下一个event loop周期的`timers`阶段执行，也可能会在下下个周期中执行，所以会造成`1-3-2`和`3-1-2`这两种结果。（你可能会问，`d`为什么不会在下下下个周期执行呢？理论上是可能的，但是实际上event loop一个周期的时间还没短到那个程度。。。）

example5
```js
setImmediate(() => {
  setImmediate(() => {
    console.log(1)
    setImmediate(() =>{
      console.log(2)
    })
  })

  setTimeout(() => {
    console.log(3)
  }, 0)

  process.nextTick(() => {
    console.log(4)
  })
})
```

这个例子，用`process.nextTick()`操作拖延了当前所在`check`阶段的时间，从而延长了当前event loop周期的时间，于是使得在进入下一个event loop周期的时候`setTimeout`的定时器已经到达了指定时间，同时`process.nextTick()`的回调是在当前阶段（即`check`阶段）结束之后立刻执行的，所以这个例子的结果始终的`4-3-1-2`。

目前为止，`setTimeout`、`setImmediate`和`process.nextTick()`这三个操作已经解释的很清楚了，现在来说说最后一个`promise`。

`promise`是es6新增的api，在这之前jQuery实现了一个类似的对象，在这里就不做讨论了。这里讨论的重点是`promise`中的`resolve`和`reject`方法导致回调函数执行的时间点。其实之前关于宏任务与微任务的观点中也说过了，`promise`和`process.nextTick()`一样，是可以在当前event loop周期中执行的。实例化`promise`对象的回调函数是同步执行的，promise.then中的回调函数是异步执行的，一旦经过`resolve`或`reject`方法后，相应的回调函数会在当前阶段结束后、在`nextTickQueue`队列执行完成后、下个阶段开始前执行。

来看一下下面这个例子。

example6
```js
const fs = require('fs')

fs.readFile(__filename, () => {
  new Promise((resolve) => {
    console.log(1)
      
    process.nextTick(() => {
      console.log(2)
    })
    
    setTimeout(() => {
      console.log(3)
    }, 0)

    resolve()

    process.nextTick(() => {
      console.log(4)
    })

    setImmediate(() => {
      console.log(5)
    })
    
    console.log(6)
  }).then(() => {
    console.log(7)
  })

  setTimeout(() => {
    console.log(8)
  }, 0)

  console.log(9)
})
```

这个例子将本文的主角所做的所有操作锁定在了`poll`阶段，是不是现在觉得思路就很清晰了。先来分析一波。

实例化`promise`的函数是同步执行，所以首先输出`1-6-9`；然后`poll`阶段结束，执行`nextTickQueue`中的函数，于是`2-4`跟着输出；然后执行`resolve`对应的回调，即`7`；再然后进入`check`阶段，输出`5`，最后进入下一个event loop周期，进入`timers`阶段，输出`3-8`。最终结果为`1-6-9-2-4-7-5-3-8`，执行可以发现，没有任何问题。

再来看一看本文的主角，可能有迫不及待的同学已经执行过了，可以发现结果是`1-6-9-2-4-7-3-8-5`，区别在`setImmediate`的回调执行时间延后了。

原因就留给各位读者自行分析啦~~~

官方文档：
> * [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

参考文章: 
> * [使用Vue的nextTick引发的执行顺序之争](https://juejin.im/post/5a72df6cf265da3e2c3870b9?utm_source=gold_browser_extension)
> * [What you should know to really understand the Node.js Event Loop](https://medium.com/the-node-js-collection/what-you-should-know-to-really-understand-the-node-js-event-loop-and-its-metrics-c4907b19da4c)