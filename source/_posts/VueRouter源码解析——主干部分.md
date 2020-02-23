---
title: VueRouter源码解析——主干部分
date: 2020-02-23 18:04:24
tags: [前端,Vue]
---

## Install 阶段
VueRouter的安装一般我们都是通过`Vue.use(VueRouter)`，而`Vue.use`这个方法实际上会调用插件的`install`方法，先来看看VueRouter的install方法：

```js
import View from './components/view'
import Link from './components/link'

export let _Vue

export function install (Vue) {
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```

### _Vue 的作用
`_Vue`的用于保存`install(Vue)`方法中传入的Vue构造函数，首先会通过`install.installed = true`标志已安装过，然后结合`_Vue = Vue`，只要这两个条件为`true`则表示插件已安装过，避免了重复调用`Vue.use(VueRouter)`而产生的副作用。

### Mixin
跟Vuex一样，VueRouter也是通过mixin的方式全局添加`beforeCreated`钩子。一般来说，我们都是把`router`实例放到根实例上，因此，对于根实例调用`beforeCreated`钩子时，就会执行以下逻辑：

```js
if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } 
```

把`_routerRoot`设置为根实例，然后添加一个`_router`属性，把根实例自己传入`this._router.init(this)`，这是VueRouter的一个方法，用于初始化路由，具体后面会谈到。最后把`this._route`定义为响应式元素，而`this._route`实际上是`this._router.history.current`，指向了当前路由的Route。因为`this._route`是响应式的，另外，我们可能会经常碰见通过侦听器监听`$route`的情况，是因为`$route`是`_route`的一层只读代理，它是通过这样去实现的：

```js
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })
```

可以看到，为了避免`_route`被误操作更改，Vue为它加上了一层用Object.defineProperty定义的只读属性代理，同时，因为`_route`是响应式的，因此，`$route`也是响应式的。

对于非根实例，会走这个逻辑：
```js
this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
```

对于`beforeCreated`钩子而言，执行的顺序是先父到子，因此，子组件实例通过`this.$parent._routerRoot`便可获取到根实例。

然后，就是引入两个组件，`router-view`以及`router-link`。

```js
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)
```

最后，就是引入合并策略。在Vue实例化的过程中，对于data、computed、钩子函数都会运用不同的合并策略。而对于路由的钩子，也是一样，这些说明，路由的钩子函数用的合并策略与Vue的created钩子相同，也就是把它们都合并到同一个数组中，触发钩子时依次执行。

```js
const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
```

`install`的主要脉络就完成了。

## VueRouter 实例化过程
先来看VueRouter的构造函数：

```js

export default class VueRouter {
  static install: () => void;
  static version: string;

  app: any;
  apps: Array<any>;
  ready: boolean;
  readyCbs: Array<Function>;
  options: RouterOptions;
  mode: string;
  history: HashHistory | HTML5History | AbstractHistory;
  matcher: Matcher;
  fallback: boolean;
  beforeHooks: Array<?NavigationGuard>;
  resolveHooks: Array<?NavigationGuard>;
  afterHooks: Array<?AfterNavigationHook>;

  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

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
  }

  match (
    raw: RawLocation,
    current?: Route,
    redirectedFrom?: Location
  ): Route {
    return this.matcher.match(raw, current, redirectedFrom)
  }

  get currentRoute (): ?Route {
    return this.history && this.history.current
  }

  init (app: any /* Vue component instance */) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )

    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }

  beforeEach (fn: Function): Function {
    return registerHook(this.beforeHooks, fn)
  }

  beforeResolve (fn: Function): Function {
    return registerHook(this.resolveHooks, fn)
  }

  afterEach (fn: Function): Function {
    return registerHook(this.afterHooks, fn)
  }

  onReady (cb: Function, errorCb?: Function) {
    this.history.onReady(cb, errorCb)
  }

  onError (errorCb: Function) {
    this.history.onError(errorCb)
  }

  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  }

  go (n: number) {
    this.history.go(n)
  }

  back () {
    this.go(-1)
  }

  forward () {
    this.go(1)
  }

  getMatchedComponents (to?: RawLocation | Route): Array<any> {
  }

  resolve (
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
  }

  addRoutes (routes: Array<RouteConfig>) {
  }
}
```

对于构造函数而言，最主要的操作有以下几步：
1. 初始化一些必要的属性，如根实例、所有实例、钩子数组
2. 通过 createMatcher创建匹配器 matcher
3. 根据 mode 创建合适的 history 对象

```js
  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

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
  }
```

对于`mode:history`时，还会判断浏览器是否支持，不支持的化会回退到`hash`。

## `init()`
从`install`的时候可以看到，router实例执行了init方法，并且传入了根实例，我们来看看它做了些什么：

```js
  init (app: any /* Vue component instance */) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )

    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }
```

首先，把传入的Vue实例放入`this.apps`数组中，然后在该Vue实例上添加一个监听`destroyed`钩子的侦听器，为了让该Vue实例在销毁时能够释放`this.apps`数组对它的一个引用，避免内存泄漏。

```js
    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null
    })
```

然后，如果`this.app`为空，说明Vue根实例还没传入，那么就把当前实例作为Vue根实例。从install方法知道，我们整个组件树的根App实例是第一个调用init方法的，因此，它是根实例，也就是`this.app`。

如果不会空，那么传入的实例并不是根实例，那么，直接返回。因为后续的操作是对于根实例而言的。

```js
    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }
```

上面这一坨代码，代表着路由初始化的第一次路由匹配。先获取当前的Location对象，然后通过`history.transitionTo`实现路由的跳转。

```js
    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
```

最后这个很关键，listen方法接收一个回调，这个回调在`history.transitionTo`执行完毕就会执行，目的是更新每一个Vue实例上的`_route`属性，这样，因为`_route`是响应式元素，那么当`_route`更新时，就会通知`_route`的依赖，通知`Watcher`刷新视图。