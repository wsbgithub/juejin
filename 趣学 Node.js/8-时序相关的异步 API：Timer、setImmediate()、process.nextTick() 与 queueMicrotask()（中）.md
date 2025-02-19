Node.js 中的 `setTimeout()`
-------------------------

上一章中，我们讲了在 Node.js 中，`setTimeout()` 函数本质是创建一个 `Timeout` 实例，并将其搞成链表推入一个 `Map` 和一个优先队列中。在背后有且仅有一个定时器来推动 libuv 定期消费这些链表。

但是讲到最后，也就讲了这个流程，那么所谓的 `Timeout` 实例到底是什么呢？在消费链表的时候，实际上发生了什么事呢？我们上一章讲到了所谓的薅羊毛算法，或是珠宝店算法，那么进到某个羊圈或者珠宝店后，到底发生了什么呢？进入羊圈的逻辑就是在上一章中未详解的 `listOnTimeout()` 函数中。

### `listOnTimeout()`

我从[该函数的源码](https://github.com/nodejs/node/blob/v18.14.1/lib/internal/timers.js#L517-L588 "https://github.com/nodejs/node/blob/v18.14.1/lib/internal/timers.js#L517-L588")中提取逻辑，保留主要逻辑以便分析。

      function listOnTimeout(list, now) {
        const msecs = list.msecs;
    
        let ranAtLeastOneTimer = false;
        let timer;
        while ((timer = L.peek(list)) != null) {
          const diff = now - timer._idleStart;
    
          // Check if this loop iteration is too early for the next timer.
          // This happens if there are more timers scheduled for later in the list.
          if (diff < msecs) {
            list.expiry = MathMax(timer._idleStart + msecs, now + 1);
            list.id = timerListId++;
            timerListQueue.percolateDown(1);
            debug('%d list wait because diff is %d', msecs, diff);
            return;
          }
    
          if (ranAtLeastOneTimer)
            runNextTicks();
          else
            ranAtLeastOneTimer = true;
    
          // The actual logic for when a timeout happens.
          L.remove(timer);
    
          ...
    
          let start;
          if (timer._repeat)
            start = getLibuvNow();
    
          try {
            const args = timer._timerArgs;
            if (args === undefined)
              timer._onTimeout();
            else
              ReflectApply(timer._onTimeout, timer, args);
          } finally {
            if (timer._repeat && timer._idleTimeout !== -1) {
              timer._idleTimeout = timer._repeat;
              insert(timer, timer._idleTimeout, start);
            } else if (...) {
              ...
            }
          }
        }
    

上面就是所谓的“进入一个羊圈，把能薅的羊全薅一遍”的过程。`list` 就是羊圈，`now` 就是当前 Tick 的时间。首先就是一个 `while` 去不断从 `list` 中获取首个元素。上一章我们也思考过了，由于时间一往无前的特性，任意一个 `Timeout` 的链表元素都是按时序插入的，所以肯定是越前面的元素越早过期——即这是一个有序链表。所以“不断从 `list` 中获取首个元素”意味着按时间早晚顺序从链表中拿到对应的 `Timeout` 实例。

> 上面代码中，`L.peek()` 就是从一个 `list` 中获取首个元素。

在 `while` 每次循环中，先判断一下拿到的 `Timeout` 实例是否应被触发，即是否过期。如果没有过期，则进入 `if` 分支。将该 `Timeout` 实例对应的过期时间作为当前链表整体的过期时间，并重排优先队列。

> #### IEEE754 的坑
> 
> 我们上一章中讲过，链表的 `expiry` 为这条链表 `Timeout` 最早过期时间。所以当下一个 `Timeout` 没过期的时候，自然就会更新该值为对应 `Timeout` 的值，即：
> 
>     list.expiry = MathMax(timer._idleStart + msecs, now + 1);
>     
> 
> 可能大家会好奇，为什么 `list.expiry` 不直接是 `timer._idleStart + msecs`，而是还要跟 `now + 1` 进行一次最大值比较。**其实之前的** **Node.js** **的确是直接这么写的，但这么写有问题。**
> 
> 不知道大家有没有注意到，我之前给大家介绍这整段逻辑的时候，全程没有提过对 `setTimeout()` 的第二个时间参数做过处理。也就是说，它可以是个浮点数。举个例子，`1.1 * 100` 在 JavaScript 并不是 `110`，而是 `110.00000000000001`。就是这么点细微的差别，有可能导致问题。
> 
> 比如下面这段代码：
> 
>     const time = 1.1 * 100;
>     function exec(i) {
>       console.log(i);
>       setTimeout(exec, time, ++i);
>     }
>     exec(0);
>     
> 
> 我们能看到这是一个一直 `setTimeout(110.00000000000001)` 的例子。如果当前时间是 `18`，那么超时时间理论上是 `18 + 110.00000000000001` ，即 `128.00000000000001`。实际上呢？大家不妨可以打开 Chrome 的控制台或者 Node.js 测试一下：
> 
> ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3677d226db543baa94cb0e702795201~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=896&h=504&s=38782&e=png&a=1&b=ffffff)
> 
> 看吧！变成整数了！显然 JavaScript 的 IEEE754 浮点数并不靠谱。但是作为 Node.js 运行时，显然不能被这种不靠谱坑了。我们继续往下看，如果还是 Node.js 之前的逻辑，那么 `list.expiry` 会被设置为 `128`。如果当前时间恰好是 `128`，在 `processTimer()` 的时候会循环进该链表认为它“过期”了。但是进了链表 `listOnTimer()` 后，`diff = now - timer._idleStart` 为 `128 - 18`，即 `diff` 为 `110`。那这个时候 `diff < mses` 显然是成立的（毕竟 `110 < 110.00000000000001`），会被认为“没过期”。
> 
> 外层认为“过期了”，内层认为“没过期”。那么将 `list.expiry` 原封不动地设置为该 `Timeout` 的过期时间的话会发生什么呢？退出该层 `listOnTimer` 后，在 `processTimer` 的下一次循环仍然会拿到这个 `list`，拿到后再判断仍然会认为其“过期了”。然后再进去，然后……🤡 死循环了！
> 
> 这就相当于薅羊毛时候，在外面看羊毛长好了，进去一看发现是脏东西。再出去看，好像羊毛长好了，进去一看又是脏东西……
> 
> 于是，为了避免这种情况，Node.js 就手动往这个超时时间上加个一，反正一毫秒左右的误差很正常。即使是 Node.js 正常在跑，`setTimeout()` 本身也就没有那个精确度。

上面代码中 `timerListQueue.percolateDown(1)` 的意思是，对优先队列第一个元素进行下滤操作。毕竟这个时候它的 `expiry` 被修改了，不一定是最早过期的链表了，需要下滤以得到新的最早链表。下滤过后，退出该函数，回到之前的 `processTimers()`，进入下一个循环，即再拿出新的最早过期链表，并判断有没有过期，然后做后续逻辑……这就是薅羊毛薅到没有可薅之后的流程。

但若链表中当前 `Timeout` 已过期，则是后面的逻辑了。先还是模拟一次 Tick，就如 `processTimers()` 里面那样：

    if (ranAtLeastOneTimer)
      runNextTicks();
    else
      ranAtLeastOneTimer = true;
    

然后将该 `Timeout` 从链表中移除。接下去的整体逻辑就是去执行 `Timeout` 的 `_onTimeout()` 方法，里面就是去执行用户侧传入 `setTimeout()` 的第一个参数 `callback`，这就是 `setTimeout()` 逻辑的最终归宿了，毕竟它的用处就是在多少时间后去执行第一个参数 `callback`。再之后就是 `setInterval()` 逻辑（`timer._repeat`）了，如果是 Repeat 的，则将 `Timeout` 改期并通过 `insert()` 重新插入链表、更新 `Map` 和优先队列。

将 `processTimers()` 和 `listOnTimeout` 联立起来，去掉一些边角料逻辑，它的流程如下所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ddf478d4d764a72ba6f7788dd827528~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1473&h=1073&s=133411&e=png&a=1&b=ffffff)

### `Timeout`

上一章，以及本章前文中，我们提了太多次的 `Timeout` 了。凡流程中，必出现 `Timeout`。这到底是个什么东西？它是 `setTimeout()`、`setInterval()` 的最终归宿，存储了定时器的一些元数据，如起始时间、过期间隔、过期回调函数等。

    class Timeout {
      constructor(callback, after, args, isRepeat, isRefed) {
        after *= 1; // Coalesce to number or NaN
        if (!(after >= 1 && after <= TIMEOUT_MAX)) {
          if (after > TIMEOUT_MAX) {
            process.emitWarning(`${after} does not fit into` +
                                ' a 32-bit signed integer.' +
                                '\nTimeout duration was set to 1.',
                                'TimeoutOverflowWarning');
          }
          after = 1; // Schedule on next tick, follows browser behavior
        }
    
        this._idleTimeout = after;
        this._idlePrev = this;
        this._idleNext = this;
        this._idleStart = null;
        // This must be set to null first to avoid function tracking
        // on the hidden class, revisit in V8 versions after 6.2
        this._onTimeout = null;
        this._onTimeout = callback;
        this._timerArgs = args;
        this._repeat = isRepeat ? after : null;
        this._destroyed = false;
    
        if (isRefed)
          incRefCount();
        this[kRefed] = isRefed;
    
        ...
      }
    }
    

最开始是判断过期间隔的合法性，如果时间不合法，则强行将时间改为 `1`。然后就是将各种元数据信息放到成员变量中。这个 `_onTimeout`，我们可以看到，就是 `setTimeout()` 时候传进来的回调函数。是不是就能跟上面的 `listOnTimeout()` 串联起来了？

上面的 `isRepeat` 即该 `Timeout` 是否重复执行。在上一章中，我们可以看到 `setTimeout()` 将该值设置为 `false`，即不重复。而 `setInterval()` 的代码与 `setTimeout()` 几乎一样，唯一的区别就是该参数位传的是 `true`。

### `timeout.ref()` 与 `timeout.unref()`

还记得我们在上一章留了一手没有解释的内容吗？`uv_ref` 和 `uv_unref`。有兴趣的读者可返回去搜一下这个关键字，以及 `timeoutInfo[0]`。

先来看文档：

> When called, requests that the Node.js event loop _not_ exit so long as the `Timeout` is active. Calling `timeout.ref()` multiple times will have no effect.
> 
> By default, all `Timeout` objects are "ref'ed", making it normally unnecessary to call `timeout.ref()` unless `timeout.unref()` had been called previously.

就是说，如果 `Timeout` 有被“引用”，则在没有其他让事件循环“长存”的条件下（如文件 I/O 等待、网络事件等待等），Node.js 的执行生命周期会被 `Timeout` 撑着。否则，若没有其他“长存”条件，Node.js 会执行完当前 Tick 后，马上退出，并不会等 `Timeout` 执行。

举个最简单例子：

    console.log('start');
    const timer = setTimeout(() => {
      console.log('done');
    }, 100);
    

这段代码用 Node.js 跑，会先输出 `start`，然后在 100 毫秒之后，输出 `done`，然后 Node.js 退出。如果在第四行插入：`timer.unref()`，则 Node.js 会在输出 `start` 后，创建 `Timeout`，然后直接退出。

在 libuv 中，就有 `uv_ref()` 和 `uv_unref()` 的概念。如果 libuv 中，不存在任何被 Reference 的句柄，就会退出事件循环。所以 `uv_ref()` 是为了撑起其生命周期用的。

Node.js 中，用了一种非常规的方式，来做那个唯一的 C++ 侧定时器的 Reference 与 Unreference。首先，在 Node.js 的 `Environment` 中，有一个 `timeout_info_` 的 `Int32Array` 的 JavaScript `TypedArray`。说是数组，实际上它就一个元素，`timeout_info_[0]`。如果我们直接用一个 `Number` 类型，那么在获取它、更改它的时候，要在 JavaScript 与 C++ 侧反复横跳，这是有开销的。而根据 V8 的特性，C++ 侧是可以直接访问 `Int32Array` 指定下标的内容，获取的是原生 4 字节的整型内存，而 JavaScript 侧也可以直接操作该数组。这么一来，双方对其第 `0` 个元素读取和操作都可直接进行，而不用切换上下文，少了些开销。

所以，虽然它是个 `Int32Array`，但 Node.js 却 Trick 地将其当作横跳 C++ 侧与 JavaScript 侧的一个 `Number` 值。简而言之，你就简单粗暴地当 `timeoutInfo[0]` 是一个 `Number` 值用就好了。

这个值的用处是，记录所有 `Timeout` 实例的 Reference 与 Unreference 累加的值。当该值为 `0` 的时候，说明不再有 JavaScript 侧的 `Timeout` 需要被 Reference，那么 `Environment` 中那个唯一的定时器可被 `uv_unref()`；否则就说明还有至少一个 `Timeout` 被 Reference，生命周期需继续苦撑着。

在 `Timeout` 的构造函数中，最后一个参数是 `isRefed`。`setTimeout()` 与 `setInterval()` 中传的都是 `true`。

我们看上面的构造函数代码，如果 `isRefed` 为 `true`，会增加 Reference 的值（`incRefCount`）。

    function incRefCount() {
      if (timeoutInfo[0]++ === 0)
        toggleTimerRef(true);
    }
    

这里面做的事情就是，将 `timeoutInfo[0]` 加一。若加一前其值为 `0`，意味着之前没有一个被 Reference 的 `Timeout`，所以从无到有，我们需要去 C++ 侧将唯一的定时器给 Reference（`toggleTimerRef(true)`）。

根据上面的代码，易得 `decRefCount()` 的代码：

    function decRefCount() {
      if (--timeoutInfo[0] === 0)
        toggleTimerRef(false);
    }
    

对于一个 `Timeout` 的 `ref` 和 `unref` 函数，实际上就是[调用上面这两个函数](https://github.com/nodejs/node/blob/v18.14.1/lib/internal/timers.js#L221-L237 "https://github.com/nodejs/node/blob/v18.14.1/lib/internal/timers.js#L221-L237")，只不过多了一些额外的判断和逻辑而已。

而在 C++ 侧的 `toggleTimerRef()` 函数，它是调用 `uv_ref()` 和 `uv_unref()` 来达到最终目的。

    void Environment::ToggleTimerRef(bool ref) {
      if (started_cleanup_) return;
    
      if (ref) {
        uv_ref(reinterpret_cast<uv_handle_t*>(timer_handle()));
      } else {
        uv_unref(reinterpret_cast<uv_handle_t*>(timer_handle()));
      }
    }
    

这是对于新建 `Timeout`、对 `Timeout` 进行 `ref()` 和 `unref()` 操作的流程，这都是自然增删过程。而在定时器自然消耗（逐个过期）的过程来说，一样也是需要对其进行一定的 `unref()` 操作的。

我们回到上一章讲的 `processTimers()` 函数，我们曾经讲过 `nextExpiry` 的正负问题：**这个事情不在主干上，不重要，暂且不提**。不过也还是提了一嘴，实际上在 C++ 侧的 `RunTimers()` 中始终用的是该值的绝对值，所以正负只是一个简易的判断标识，用于判断是否需要对 libuv 进行 Reference 或者 Unreference 操作。

    if (list.expiry > now) {
      nextExpiry = list.expiry;
      return timeoutInfo[0] > 0 ? nextExpiry : -nextExpiry;
    }
    

什么意思呢？当我们链表跑完了，剩下的都是没过期的 `Timeout`，那么我们要告知 C++ 侧下一次的过期时间，以便其重设定时器。由于我们跑完了一堆的 `Timeout`，跑完的 `Timeout` 自然将 `timeoutInfo[0]` 的值排除自身，这个时候要看剩下的 `timeoutInfo[0]` 的值是否还存在至少一个被 Reference 的 `Timeout`。若存在，则返回正的 `nextExpiry`；否则返回负值。

看看 C++ 侧怎么用这个返回值吧。

      if (expiry_ms != 0) {
        int64_t duration_ms =
            llabs(expiry_ms) - (uv_now(env->event_loop()) - env->timer_base());
    
        env->ScheduleTimer(duration_ms > 0 ? duration_ms : 1);
    
        if (expiry_ms > 0)
          uv_ref(h);
        else
          uv_unref(h);
      } else {
        uv_unref(h);
      }
    

在用的时候，通过 `llabs()` 对这个返回值取绝对值。也就是说用过期值的时候，总用其绝对值（即真实过期时间）。而正负符号则用于判断是否需要 `uv_ref()` 或 `uv_unref()`。如果返回值是 `0`，则意味着没有更多 `Timeout` 了，那么直接 `uv_unref()`。

在前面的 `listOnTimeout()` 函数中，我省略了执行 `timer._onTimeout()` 之后，判断其是否被 Reference 的逻辑。若其被 Reference，则执行完后，[会将 timeoutInfo\[0\] 减一](https://github.com/nodejs/node/blob/v18.14.1/lib/internal/timers.js#L579-L580 "https://github.com/nodejs/node/blob/v18.14.1/lib/internal/timers.js#L579-L580")。

### 小插曲：时序问题可以怎么解？

我们在上一章中还留了个问题代码。

    'use strict';
    
    setTimeout(() => {
      console.log(1);
    }, 10);
    setTimeout(() => {
      console.log(2);
    }, 15);
    let now = Date.now();
    while (Date.now() - now < 100) {
      //
    }
    setTimeout(() => {
      console.log(3);
    }, 10);
    now = Date.now();
    while (Date.now() - now < 100) {
      //
    }
    

在当前 Node.js 版本（如 v18.14.1）中，上面这个问题代码的结果是 `1`、`3`、`2`。原因也解释过了，薅羊毛薅到秃噜皮。用顺序箭头图展示，如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f3cf575f9b54e1fbca1ba6355746c03~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=653&h=493&s=31813&e=png&a=1&b=ffffff)

也就是说，过期时刻为 `205` 的 `Timeout` 居然比 `35` 的早执行。这显然不合理。合理的顺序箭头图应该如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89040989b3cc4e16a1b155deb847c426~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=663&h=473&s=32632&e=png&a=1&b=fefefe)

前者是**薅羊毛薅到死**，后者是**反复横跳**。假设你是一个 Node.js 的贡献者，想想我们之前讲解的 `processTimers()` 和 `listOnTimeout()`，先不论性能如何，我们可以怎么改让其恢复逻辑正常？

![9150e4e5gy1g63ilh1p83g203c03naa2.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5ec571a016b4495b9f73779bd80da50~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=120&h=131&s=10526&e=gif&f=4&b=fcfafa)

> 毕竟由薅到死变成反复横跳，切换和性能肯定是有额外开销的，性能肯定会劣化。

其实做这个改造，在逻辑上的方案很简单：

1.  在每次 `listOnTimeout()` 最开始，都先获取第二超时的链表；
2.  在 `listOnTimeout()` 循环中，每次 `Timeout` 除了判断是否没过期，再额外判断一下与第二超时链表谁会先到期，若是第二链表先到期，则重排优先队列后退出 `listOnTimeout()`。

就这两步，就可以将上面第一张图变成第二张图了。比如第二张图中，`10ms` 的 `21` 执行完了，接下去是 `205`，这个时候 `205` 与第二链表过期时间 `35` 比较。发现比不过，就更新链表过期时间为 `205`，重排优先队列，这个时候 `20ms` 就跑第一位来了，接下去就执行 `20ms` 的第一个元素 `35`，然后 `67` 时候发现比不过 `30ms` 的 `36`，以此类推……最终达成了 `10ms`→`20ms`→`30ms`→`20ms`→`10ms` 的反复横跳。

现在的问题在于，如何获取第二超时的链表呢？优先队列的本质是一个二叉堆，我们只能保证堆顶元素是最小的。那第二小的怎么获取？

我们看看以最小值为优先的优先队列的二叉堆是按什么规则排列的：父元素总比子元素小。比如这就是一个优先队列可能的内部结构：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c26691243fe4fd485d60ca1de01b4fe~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=251&h=251&s=13418&e=png&b=fefefe)

堆顶是 `3`，子节点 `4`、`5` 均比 `3` 小；`4` 的子节点 `6`、`8` 也比 `4` 小。当然，把 `6` 挪到 `5` 的子节点，也是一个合法的优先队列。只要满足上面那个条件，怎么排列无所谓。如果我们在堆尾插入一个 `2`，就会是这样的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4968317be844414da71e477ddb6f0414~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=351&h=255&s=17011&e=png&b=ffffff)

然后对堆尾进行滤上操作，`2` 比 `5` 小，要把 `5` 换下来。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0b357a5410c4f6b94dce1d34ac574a9~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=351&h=255&s=16975&e=png&b=ffffff)

然后 `2` 比 `3` 小，再换。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59c58c50ceb74c76bacdf280445c3d0e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=351&h=255&s=15905&e=png&b=ffffff)

这个时候，`2` 就处于堆顶了。`3` 变成了这个优先队列的第二小元素。由于这是一本《趣学 Node.js》小册，而非《趣学算法》《趣学数据结构》，更多关于优先队列的信息，大家可自行去网上学习，这里点到为止。

根据优先队列的二叉堆性质，我们很容易判断出堆顶元素是最小元素，即优先队列每次都是从这儿取。而其两个子节点都是比堆顶元素大的，但是它们各自又比各自的子节点们小。虽然可能会有右子节点比左子节点大，也有可能比左子节点的所有子节点都大，但至少它比自己的子节点小。所以光判断第二小元素还是很容易的，只要判断堆顶节点的两个子节点谁小，谁就是第二小了。**它们俩其中一个肯定是第二小的，但是另一个则不一定第三小。**

这就相当于，堆顶肯定是龙头。然后左右子节点中肯定是有一个凤头，带着一帮凤小弟。另一个虽然也带了一帮比它菜的小弟，但它不一定比凤小弟厉害——人才梯队问题，他们只是各自人才梯队里面顶尖的，但是有可能只是个鸡头，比不上凤尾。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/892b0c4e53b44a2bbb00713249974001~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=800&h=536&s=575044&e=png&b=1f1e19)

解决了第二超时链表问题，一切就简单了。先打开 Node.js 的[优先队列源码](https://github.com/nodejs/node/blob/v18.14.1/lib/internal/priority_queue.js "https://github.com/nodejs/node/blob/v18.14.1/lib/internal/priority_queue.js")。可以看到它内部用一个数组模拟了堆。

    class PriorityQueue {
      #compare = (a, b) => a - b;
      #heap = new Array(64);
      ...
    }
    

堆初始大小为 `64`。与平常使用数组不一样，优先队列通常用堆的 `1` 下标作为根节点，在此处即为 `#heap[1]`。然后每个节点左右子节点下标分别为父节点乘二，以及乘二加一。也就是说，根节点子节点的下标分别为 `2`、`3`。要取第二元素的值，相当于判断 `2`、`3` 下标元素哪个小。

我们可以为其补一个 `secondary()` 方法：

      secondary() {
        // As the priority queue is a binary heap, the secondary element is
        // always the second or third element in the heap.
        switch (this.#size) {
          case 0:
          case 1:
            return undefined;
    
          case 2:
            return this.#heap[2] === undefined ? this.#heap[3] : this.#heap[2];
    
          default:
            return this.#compare(this.#heap[2], this.#heap[3]) < 0 ?
              this.#heap[2] :
              this.#heap[3];
        }
      }
    

这里 `#compare` 函数为比较两个元素大小的函数，默认情况下为直接判断数值大小。然而在 `Timeout` 中，是不能直接这么判断的。所以针对 `Timeout` 的 `#compare` 函数是判断其过期时间这些的。

可以看到，上面这个 `secondary()` 函数做了一些边界判断，如没有任何元素、只有一个元素以及只有两个元素的时候，应该如何返回。然后剩下的情况就按我之前讲的这种方式判断。

有了 `secondary()` 函数后，我们只需要在 `listOnTimeout()` 中做点手脚就好了。假设你是 Node.js 贡献者，可以在 `diff < msecs` 的判断上做手脚，如：

    ...
    const secondaryList = timerListQueue.secondary();
    while ((timer = L.peek(list)) != null) {
      ...
      if (diff < msecs || secondaryList?.expiry < timer._idleStart + msecs) {
        ...原逻辑
      }
      ...
    }
    

如上，我们只需要在判断 `diff < msecs` 这里多加一个或的条件判断即可，判断它是否大于第二链表的超时时间，如果大于了，就重排优先队列并开始横跳。

> #### [PR #46644](https://github.com/nodejs/node/pull/46644 "https://github.com/nodejs/node/pull/46644")
> 
> 实际上，自我在上一章发现了这个问题后，就提交了这个小插曲的 PR 了。的确性能有些许劣化，在我的 MacBook M1 上，Benchmark 粗略跑了一下如下：
> 
>     // 原逻辑
>     timers/timers-timeout-unpooled.js n=1000000: 19,307,131.535027202
>     
>     // PR 逻辑
>     timers/timers-timeout-unpooled.js n=1000000: 19,102,850.664916743
>     
> 
> 不过，对于正确性来说，我个人认为这个劣化在可接受范围。 最终 PR 合并不合并还得看其他 TSC 或者 Collaborator 的 Review。若没通过也无所谓，为你们讲解的目的也达到了；若最终通过了，那么上一章关于这一块的薅羊毛算法就会成为历史，从而变成新的反复横跳法。但是大家多了解了解历史问题也挺好的。

小结
--

上一章中，我们讲了 `Timeout` 优先队列是如何访问并触发的，并提出薅羊毛算法和珠宝店算法。而本章中，我们介绍了真进到羊圈后，里面发生的逻辑，如何逐一将羊毛薅秃噜皮的。在这里面为大家避开了一个 IEEE754 的坑。很多人总以为这是 JavaScript 的数字体系不好，实际上这要怪 IEEE754，并非 ECMAScript。更多关于这块浮点数的内容，大家也可以去看看网上的资料，或者阅读一下《[JavaScript 悟道](https://book.douban.com/subject/35469273/ "https://book.douban.com/subject/35469273/")》。

在上一章，还遗留了一个“不重要、暂且不提”的 `uv_ref()` 与 `uv_unref()`，在本章中也为大家描述清楚了其用途，以及它与 `timer` 中的 `timer.ref()`、`timer.unref()` 的关系。

最后，来了一个小插曲，以身试法，以贡献者的思路，一步步解析我们可以如何分析问题，并且通过怎么样的修改，能解决上一章中遗留的时序问题。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2873eac1c4964dbc8be829120d0f17fb~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=658&h=370&s=54312&e=png&b=faf8f8)