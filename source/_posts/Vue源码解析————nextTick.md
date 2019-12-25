---
title: Vue源码解析————nextTick
date: 2019-12-25 15:41:04
tags: [前端,Vue]
---

`nextTick`是我们日常开发也许很经常用的一个功能。整个`nextTick`的模块封装到了`src\core\util\next-tick.js`中。

## 前菜
首先可以思考一下下面两种情况的输出：
```vue
<template>
  <div id="app2">
    <div ref="root">Hello {{ a }}</div>
    <button @click="run">Run</button>
  </div>
</template>

<script>
export default {
  name: "app",
  data() {
    return {
      a: 1,
      b: 2
    };
  },
  methods: {
    run() {
      this.$nextTick(() => {
        console.log(this.$refs.root.innerHTML);
      });
      this.a = 5;
      this.$nextTick(() => {
        console.log(this.$refs.root.innerHTML);
      });
    }
  }
};
</script>
```

```js
export default {
  name: "app",
  data() {
    return {
      a: 1,
      b: 2
    };
  },
  methods: {
    run() {
      this.$nextTick(() => {
        console.log(this.a);
        console.log(this.b);
      });
      this.a = 5;
      this.b = 10;
      this.$nextTick(() => {
        console.log(this.a);
        console.log(this.b);
      });
    }
  }
};
```

## Event Loop
在了解nextTick之前，需要先理清一下浏览器事件循环的模型：

![image](http://static-cdn.lxzmww.xyz/eventloop.jpg)

主线程的执行过程就是一个Tick。异步任务的回调都存放在任务队列中，每一个Tick的过程主线程都会执行一个宏任务，然后清空所有微任务。

在浏览器环境中，常见的 macro task 有 setTimeout、MessageChannel、postMessage、setImmediate；常见的 micro task 有 MutationObsever 和 Promise.then。

## nextTick
`nextTick`的代码除去注释后大概在70行左右，主要为了适应不同浏览器的兼容状况。

```js
/* @flow */
/* globals MutationObserver */

import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

export let isUsingMicroTask = false

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

// Here we have async deferring wrappers using microtasks.
// In 2.5 we used (macro) tasks (in combination with microtasks).
// However, it has subtle problems when state is changed right before repaint
// (e.g. #6813, out-in transitions).
// Also, using (macro) tasks in event handler would cause some weird behaviors
// that cannot be circumvented (e.g. #7109, #7153, #7546, #7834, #8109).
// So we now use microtasks everywhere, again.
// A major drawback of this tradeoff is that there are some scenarios
// where microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690, which have workarounds)
// or even between bubbling of the same event (#6566).
let timerFunc

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}

```

`timerFunc`是Vue根据浏览器的兼容状况最终返回的异步调用。可以看到，对于原生支持Promise的浏览器，会优先使用Promise.then，对于支持MutationObserver的非IE浏览器，则使用MutationObserver。因为浏览器中的微任务主要是Promise.then以及MutationObserver，可以看到，Vue选择了降级到setImmediate，但由于setImmedate目前只有IE11和Edge支持，因此，对于低版本的IE浏览器，最终会使用setTimeout。

最后导出`nextTick`函数。`nextTick`是支持Promise调用的（如果浏览器支持的话），因此，最后会做一个判断，对于支持Promise的浏览器，如果没有传回调函数，则会返回一个Promise。
因此，下面的使用是相等的：
```js
this.$nextTick(() => {});
this.$nextTick.then(() => {});
```

最后把回调函数加入到`callbacks`数组中，然后执行`timerFunc()`。如果是Promise，那么将会执行下面的逻辑：
```js
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
```

通过Promise.then将flushCallbacks加入到微任务的队列中，待下一个Tick时执行：
```js
const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

`flushCallbacks`复制了一个`callbacks`的副本，然后清空原来的`callback`，遍历副本执行回调。


回顾之前，当我们派发更新时，通过`nextTick(flushSchedulerQueue)`调用`flushSchedulerQueue`遍历Watcher进行相应的更新。

这时候，我们就把`flushSchedulerQueue`添加到了`callbacks`中。然后等待浏览器主线程执行完这个Tick后，就会去执行任务队列中的回调`callbacks`，然后就会执行`flushSchedulerQueue`，这就达到了异步渲染的目的。

## 前菜解读
#### 对于第一种情况：
```vue
<template>
  <div id="app2">
    <div ref="root">Hello {{ a }}</div>
    <button @click="run">Run</button>
  </div>
</template>

<script>
export default {
  name: "app",
  data() {
    return {
      a: 1,
      b: 2
    };
  },
  methods: {
    run() {
      this.$nextTick(() => {
        console.log(this.$refs.root.innerHTML);
      });
      this.a = 5;
      this.$nextTick(() => {
        console.log(this.$refs.root.innerHTML);
      });
    }
  }
};
</script>
```

执行`run`方法前，情况如下：
```
// nextTick 的 callbacks
callbacks = [];
// 浏览器的微任务队列
microTaskQueue = [];
```

执行`run`方法时，主线程执行`run`的逻辑，遇到第一个`$nextTick`，我们设回调函数为`a`，这时候情况如下，此时`timerFunc`已经放入了浏览器微任务队列中：
```
// nextTick 的 callbacks
callbacks = [ a ];
// 浏览器的微任务队列
microTaskQueue = [ timerFunc ];
```

然后遇到了`this.a = 5`，从派发更新的源码可以看到，此时`flushSchedulerQueue`会加入到`callbacks`中：
```
// nextTick 的 callbacks
callbacks = [ a, flushSchedulerQueue ];
// 浏览器的微任务队列
microTaskQueue = [ timerFunc ];
```

然后遇到第二个`$nextTick`，设为`b`，然后加入到`callback`中：
```
// nextTick 的 callbacks
callbacks = [ a, flushSchedulerQueue, b ];
// 浏览器的微任务队列
microTaskQueue = [ timerFunc ];
```

`run`方法执行完毕，然后从微任务队列中取`timerFunc`执行。`timerFunc`依次清空`callbacks`。因为渲染Watcher的更新是在`flushSchedulerQueue`中执行了，那么，当我们执行回调函数`a`的时候，视图还没有更新，所以回调函数`a`输出的是`1`。然后再执行`flushSchedulerQueue`更新视图。所以回调函数`b`则是输出`5`。

#### 针对第二种情况：
```js
export default {
  name: "app",
  data() {
    return {
      a: 1,
      b: 2
    };
  },
  methods: {
    run() {
      this.$nextTick(() => {
        console.log(this.a);
        console.log(this.b);
      });
      this.a = 5;
      this.b = 10;
      this.$nextTick(() => {
        console.log(this.a);
        console.log(this.b);
      });
    }
  }
};
```

第一个回调和第二个回调输出的都是一样的：
```
5
10
```

与第一种情况不同的是，nextTick是把回调添加入`callbacks`，再把`timerFunc`加入到任务队列，这个过程是异步的。但是，在`run`方法中同步修改了`this.a`和`this.b`的值，并且，修改响应式元素的值会触发`setter`，然后在`setter`中先修改值，再通过`dep`通知Watcher更新视图。数据修改的逻辑是同步的。因此，再回调函数中获取数据，就是新的数据，因为数据更新同步执行，再异步回调之前。
