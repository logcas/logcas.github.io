---
title: VueRouter源码解析之History对象
date: 2020-02-26 18:04:41
tags: [前端,Vue]
---

## History 对象
VueRouter中共有三种的模式，分别是`hash（默认）`、`html5`以及`abstract（用于SSR）`。

默认的模式就是通过URL上的hash变化达到路由的目的，hash的更新为触发浏览器`onhashchange`事件，通过监听此事件可以触发组件的更新。同时，hash的变化是可以记录在浏览器history对象上的，因此是可以配合浏览器前进和后退按钮的。

HTML5模式是通过`pushState`和`replaceState`两个API实现的，当调用这两个API时，浏览器并不会前进到对应的URL，但能更新地址栏上的URL，并且，对于push能够加入到history记录中，因此也可以配合浏览器的前进和后退按钮。但是，因为它不是通过hash的，因此，当用户在地址栏上手动输入URL，浏览器就会向URL发起请求，这时候，HTML5模式需要后台配合，在任何时候都返回`index.html`。这样才能正确使用。

抽象模式用于SSR，这里先不赘述。

但是不管是hash还是html5还是abstract，它都继承自Vue定义的History基类，共享着一些路由跳转的核心方法。在VueRouter中，History掌控着路由跳转时生命周期的流程，也就是路由守卫是在History对象上执行的。

## VueRouter
回到VueRouter的构造函数，有这样一段代码：
```js
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
```

不同模式实例化不同的History对象，挂载到VueRouter实例的history属性上。

让我们进行编程式导航链接时，一般会通过`this.$router.push()`方法实现跳转，它的实现如下：
```js
  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    // $flow-disable-line
    if (!onComplete && !onAbort && typeof Promise !== 'undefined') {
      return new Promise((resolve, reject) => {
        this.history.push(location, resolve, reject)
      })
    } else {
      this.history.push(location, onComplete, onAbort)
    }
  }
```

可以看到，`this.$router.push`只是调用了`this.history.push`方法，并且包裹了一层Promise，表示着我们可以对它进行链式调用。同时说明，路由跳转的核心在History对象上。

为了方便，这里选择HTML5模式的History对象进行分析。

## HTML5History
```js
export class HTML5History extends History {
  constructor (router: Router, base: ?string) {
    super(router, base)

    const expectScroll = router.options.scrollBehavior
    const supportsScroll = supportsPushState && expectScroll

    if (supportsScroll) {
      setupScroll()
    }

    const initLocation = getLocation(this.base)
    window.addEventListener('popstate', e => {
      const current = this.current

      // Avoiding first `popstate` event dispatched in some browsers but first
      // history route not updated since async guard at the same time.
      const location = getLocation(this.base)
      if (this.current === START && location === initLocation) {
        return
      }

      this.transitionTo(location, route => {
        if (supportsScroll) {
          handleScroll(router, route, current, true)
        }
      })
    })
  }
}
```

首先来看构造函数，关键在于`window.addEventListener`，监听了`popstate`方法，当URL改变时会触发路由的跳转（执行`transitionTo`）。

我们来看看`push`方法究竟做了什么：

```js
  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this
    this.transitionTo(location, route => {
      pushState(cleanPath(this.base + route.fullPath))
      handleScroll(this.router, route, fromRoute, false)
      onComplete && onComplete(route)
    }, onAbort)
  }
```

可以看到，不管是事件监听还是`push`方法，都调用了`transitionTo`方法，可以看出，`transitionTo`跟路由跳转有关。但`transitionTo`并不在`HTML5History`中声明，这个方法是继承自`BaseHistory`的。

```js
  transitionTo (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    const route = this.router.match(location, this.current)
    this.confirmTransition(
      route,
      () => {
        this.updateRoute(route)
        onComplete && onComplete(route)
        this.ensureURL()

        // fire ready cbs once
        if (!this.ready) {
          this.ready = true
          this.readyCbs.forEach(cb => {
            cb(route)
          })
        }
      },
      err => {
        if (onAbort) {
          onAbort(err)
        }
        if (err && !this.ready) {
          this.ready = true
          this.readyErrorCbs.forEach(cb => {
            cb(err)
          })
        }
      }
    )
  }
```

从`Matcher`的分析可知，`const route = this.router.match(location, this.current)`获取的是下一个路由匹配到的`RouteRecord`，把它当作`confirmTransition`的第一个参数传入，后两个参数依次是完成时回调以及失败时回调。

`confirmTransition`方法如下：
```js
confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
    const current = this.current
    const abort = err => {
      // after merging https://github.com/vuejs/vue-router/pull/2771 we
      // When the user navigates through history through back/forward buttons
      // we do not want to throw the error. We only throw it if directly calling
      // push/replace. That's why it's not included in isError
      if (!isExtendedError(NavigationDuplicated, err) && isError(err)) {
        if (this.errorCbs.length) {
          this.errorCbs.forEach(cb => {
            cb(err)
          })
        } else {
          warn(false, 'uncaught error during route navigation:')
          console.error(err)
        }
      }
      onAbort && onAbort(err)
    }
    if (
      isSameRoute(route, current) &&
      // in the case the route map has been dynamically appended to
      route.matched.length === current.matched.length
    ) {
      this.ensureURL()
      return abort(new NavigationDuplicated(route))
    }

    // 依次为执行 beforeRouteUpdate、beforeRouteLeave、beforeRouteEnter 组件的数组
    const { updated, deactivated, activated } = resolveQueue(
      this.current.matched,
      route.matched
    )

    // extractXXX 是为了提取对应组件中的钩子函数，并且拍平为一维数组
    // 收集便于依次调用不同顺序的钩子（路由守卫）
    const queue: Array<?NavigationGuard> = [].concat(
      // in-component leave guards
      extractLeaveGuards(deactivated),
      // global before hooks
      this.router.beforeHooks,
      // in-component update hooks
      extractUpdateHooks(updated),
      // in-config enter guards
      activated.map(m => m.beforeEnter),
      // async components
      resolveAsyncComponents(activated)
    )

    this.pending = route
    // 典型的异步函数执行器
    const iterator = (hook: NavigationGuard, next) => {
      if (this.pending !== route) {
        return abort()
      }
      try {
        hook(route, current, (to: any) => {
          if (to === false || isError(to)) {
            // next(false) -> abort navigation, ensure current URL
            this.ensureURL(true)
            abort(to)
          } else if (
            typeof to === 'string' ||
            (typeof to === 'object' &&
              (typeof to.path === 'string' || typeof to.name === 'string'))
          ) {
            // next('/') or next({ path: '/' }) -> redirect
            abort()
            if (typeof to === 'object' && to.replace) {
              this.replace(to)
            } else {
              this.push(to)
            }
          } else {
            // confirm transition and pass on the value
            next(to)
          }
        })
      } catch (e) {
        abort(e)
      }
    }

    // 执行异步函数队列
    runQueue(queue, iterator, () => {
      // 这是在 beforeRouteEnter 钩子中通过 next(function) 添加的一个回调
      const postEnterCbs = []
      const isValid = () => this.current === route
      // wait until async components are resolved before
      // extracting in-component enter guards
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queue, iterator, () => {
        if (this.pending !== route) {
          return abort()
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          this.router.app.$nextTick(() => {
            // 在 $nextTick 队列中调用
            // 因为 渲染 Watcher 是排在前面的
            // 因此 渲染Watcher执行完后当然可以获取 this（组件实例）
            postEnterCbs.forEach(cb => {
              cb()
            })
          })
        }
      })
    })
  }
```

非常长，现在来一步一步分析。

首先是，如果当前的路由为你需要跳转的路由，并且是通过push/replace方法的，会报一个错误：
```js
    const abort = err => {
      // after merging https://github.com/vuejs/vue-router/pull/2771 we
      // When the user navigates through history through back/forward buttons
      // we do not want to throw the error. We only throw it if directly calling
      // push/replace. That's why it's not included in isError
      if (!isExtendedError(NavigationDuplicated, err) && isError(err)) {
        if (this.errorCbs.length) {
          this.errorCbs.forEach(cb => {
            cb(err)
          })
        } else {
          warn(false, 'uncaught error during route navigation:')
          console.error(err)
        }
      }
      onAbort && onAbort(err)
    }
    if (
      isSameRoute(route, current) &&
      // in the case the route map has been dynamically appended to
      route.matched.length === current.matched.length
    ) {
      this.ensureURL()
      return abort(new NavigationDuplicated(route))
    }
```

然后就是处理钩子的部分，首先是获取beforeRouteUpdate、beforeRouteLeave、beforeRouteEnter这几个钩子对应的组件：
```js
    const { updated, deactivated, activated } = resolveQueue(
      this.current.matched,
      route.matched
    )
```

`reolveQueue`有点巧妙和隐晦，如果你理解了RouteRecord中的matched的生成过程，那么将会很容易理解，解析已写到注释上：
```js
function resolveQueue (
  current: Array<RouteRecord>,
  next: Array<RouteRecord>
): {
  updated: Array<RouteRecord>,
  activated: Array<RouteRecord>,
  deactivated: Array<RouteRecord>
} {
  //
  // 这里有点隐晦
  // 从 matcher 中匹配到的 RouteRecord 中有一个属性是 matched
  // matched 存放当前路由匹配的组件，是一个数组，存放顺序为 父到子
  // 如果页面切换引起的是多层嵌套在内的router-view，那么该组件的父组件在路由更改后也是存在的，
  // 例如： current.matched = [A, B, C, D]
  //       next.matched = [A, B, E, F]
  // 那么说明当前路由是从 B 组件中的 router-view 进行切换的
  // 因此， B 组件以及 B组件的父组件 都进行 beforeRouteUpdate 钩子
  // cucrent 中 C D 组件执行 beforeRouteLeave 钩子
  // next 中 E F 组件执行 beforeRouteEnter 钩子
  let i
  const max = Math.max(current.length, next.length)
  for (i = 0; i < max; i++) {
    if (current[i] !== next[i]) {
      break
    }
  }
  return {
    updated: next.slice(0, i),
    activated: next.slice(i),
    deactivated: current.slice(i)
  }
}
```

所以，updated, deactivated, activated分别为执行 beforeRouteUpdate、beforeRouteLeave、beforeRouteEnter 组件的数组。

然后，接下来把所有before钩子整合到一个队列中：
```js
    // extractXXX 是为了提取对应组件中的钩子函数，并且拍平为一维数组
    // 收集便于依次调用不同顺序的钩子（路由守卫）
    const queue: Array<?NavigationGuard> = [].concat(
      // in-component leave guards
      extractLeaveGuards(deactivated),
      // global before hooks
      this.router.beforeHooks,
      // in-component update hooks
      extractUpdateHooks(updated),
      // in-config enter guards
      activated.map(m => m.beforeEnter),
      // async components
      resolveAsyncComponents(activated)
    )
```

extractLeaveGuards、extractUpdateHooks是为了把钩子函数拍平为一个一维数组。（因此如果有Mixin等操作，组件中的钩子函数都会合并为一个数组，而这时候需要将数组拍平）

从中可以清晰地看到组件钩子（路由守卫）执行的顺序。

对于异步函数，处理的方式是通过`resolveAsyncComponents`方法：
```js
export function resolveAsyncComponents (matched: Array<RouteRecord>): Function {
  return (to, from, next) => {
    let hasAsync = false
    let pending = 0
    let error = null

    flatMapComponents(matched, (def, _, match, key) => {
      // if it's a function and doesn't have cid attached,
      // assume it's an async component resolve function.
      // we are not using Vue's default async resolving mechanism because
      // we want to halt the navigation until the incoming component has been
      // resolved.
      if (typeof def === 'function' && def.cid === undefined) {
        hasAsync = true
        pending++

        // pending 用来记录异步组件正在执行的个数
        // 当 pending = 0，就执行 next
        // 每一个异步组件都返回一个Promise
        // 因此这里通过 resolve 和 reject 订阅了 Promise 的变化

        const resolve = once(resolvedDef => {
          if (isESModule(resolvedDef)) {
            resolvedDef = resolvedDef.default
          }
          // save resolved on async factory in case it's used elsewhere
          def.resolved = typeof resolvedDef === 'function'
            ? resolvedDef
            : _Vue.extend(resolvedDef)
          match.components[key] = resolvedDef
          pending--
          if (pending <= 0) {
            next()
          }
        })

        const reject = once(reason => {
          const msg = `Failed to resolve async component ${key}: ${reason}`
          process.env.NODE_ENV !== 'production' && warn(false, msg)
          if (!error) {
            error = isError(reason)
              ? reason
              : new Error(msg)
            next(error)
          }
        })

        let res
        try {
          res = def(resolve, reject)
        } catch (e) {
          reject(e)
        }
        if (res) {
          if (typeof res.then === 'function') {
            res.then(resolve, reject)
          } else {
            // new syntax in Vue 2.3
            const comp = res.component
            if (comp && typeof comp.then === 'function') {
              comp.then(resolve, reject)
            }
          }
        }
      }
    })

    if (!hasAsync) next()
  }
}
```

核心的思想是：比如有5个异步组件，那么，`pending`就为5。在Vue的异步组件实例化的过程中，会返回一个Promise，这个方法通过`resolve`、`reject`监听这个Promise的变化，每执行依次`resolve`，`pending`就减1，当`pending`为0时，说明所有异步组件都实例化成功，那么就调用`next`方法。

然后就是定义一个适合异步函数顺序执行的执行器：
```js
    // 典型的异步函数执行器
    const iterator = (hook: NavigationGuard, next) => {
      if (this.pending !== route) {
        return abort()
      }
      try {
        hook(route, current, (to: any) => {
          if (to === false || isError(to)) {
            // next(false) -> abort navigation, ensure current URL
            this.ensureURL(true)
            abort(to)
          } else if (
            typeof to === 'string' ||
            (typeof to === 'object' &&
              (typeof to.path === 'string' || typeof to.name === 'string'))
          ) {
            // next('/') or next({ path: '/' }) -> redirect
            abort()
            if (typeof to === 'object' && to.replace) {
              this.replace(to)
            } else {
              this.push(to)
            }
          } else {
            // confirm transition and pass on the value
            next(to)
          }
        })
      } catch (e) {
        abort(e)
      }
    }
```

`hook`为对应的钩子函数，参数分别是`to`、`from`和`next`。可以看到，只有通过`next`方法，才能执行下一个钩子函数。同样，我们可以通过`next`方法传参的方式，去决定路由的变化，比如取消跳转、重定向等。

有了异步函数的顺序执行器，那么需要执行它：
```js
    // 执行异步函数队列
    runQueue(queue, iterator, () => {
      // 这是在 beforeRouteEnter 钩子中通过 next(function) 添加的一个回调
      const postEnterCbs = []
      const isValid = () => this.current === route
      // wait until async components are resolved before
      // extracting in-component enter guards
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queue, iterator, () => {
        if (this.pending !== route) {
          return abort()
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          this.router.app.$nextTick(() => {
            // 在 $nextTick 队列中调用
            // 因为 渲染 Watcher 是排在前面的
            // 因此 渲染Watcher执行完后当然可以获取 this（组件实例）
            postEnterCbs.forEach(cb => {
              cb()
            })
          })
        }
      })
    })
```

`queue`就是离开的钩子的队列，iterator就是执行器函数，第三个参数是执行完毕的回调。可以看到，首先是通过执行器顺序执行所有beforeXXX钩子，在成功后执行回调。然后再把beforeRouteEnter钩子放入到`queue`中，再执行依次钩子函数队列。

在第二次调用`runQueue`后的回调中，执行`onComplete`方法，同时将`postEnterCbs`依次翻入`$nextTick`队列中执行。

`OnComplete`实际上就是这一坨：
```js
      () => {
        this.updateRoute(route)
        onComplete && onComplete(route)
        this.ensureURL()

        // fire ready cbs once
        if (!this.ready) {
          this.ready = true
          this.readyCbs.forEach(cb => {
            cb(route)
          })
        }
      },
```

先执行`updateRoute`：
```js
  updateRoute (route: Route) {
    const prev = this.current
    this.current = route
    // history.listen(route => {
    //   this.apps.forEach((app) => {
    //     app._route = route
    //   })
    // })
    // 这里更新 app._route 时会发生触发响应式数据的 setter
    // 从而引发视图的更新
    this.cb && this.cb(route)
    this.router.afterHooks.forEach(hook => {
      hook && hook(route, prev)
    })
  }
```

需要关注的是`this.cb`的执行，`this.cb`是我们在VueRouter实例化后，调用init方法的时候通过`history.listen`挂载上去的：

```js
    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
```

这个回调执行了什么呢？如果你记得`_route`是一个响应式元素的话，那么，当`_route`发生改变时，就会通知`setter`，从而通知依赖进行刷新。那么，组件的渲染Watcher就会加入到异步刷新的队列中，进行异步刷新视图。

还有就是`this.ensureURL()`的执行，这个方法是子类原型上各自实现的：
```js
  ensureURL (push?: boolean) {
    if (getLocation(this.base) !== this.current.fullPath) {
      const current = cleanPath(this.base + this.current.fullPath)
      push ? pushState(current) : replaceState(current)
    }
  }
```

如果我们通过`push/replace`等编程时的方式进行路由跳转，URL需要History模式去更改，就是ensureURL执行的结果。