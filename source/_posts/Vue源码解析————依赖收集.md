---
title: Vue源码解析————依赖收集
date: 2019-12-24 16:30:44
tags: [前端,Vue]
---

先回顾以下`defineReactive`的逻辑：
```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  // 缓存原来的 getter 和 setter
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 如果 val = obj[key] 任然是一个对象
  // 那么 childOb 就等于这个对象经过observe后返回的一个 Observer 
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        // 如果 obj[key] 为对象类型
        // 那么前面通过 observe(val) 变成可观察对象
        // 收集依赖
        // 当更新 obj[key][xxx] =  yyy 的时候
        // 会调用诸如 {{ obj[key] }} 的回调函数
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      // 处理NaN
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 如果 newVal 是一个引用类型
      // 那么 就要重新 observe(newVal)
      // 然后再返回 childOb
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}

```

`defineReactive`为每一个元素通过闭包维持了一个`dep`，那`dep`就是有什么作用呢？

## Dep

`Dep`的位置在`src\core\observer\dep.js`中。

```js
/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

```

可以看到，每一个`Dep`实例都有一个属性`subs`，它是一个`Watcher`数组。通过一些方法添加和删除Watcher，通过`notify`遍历`subs`调用`Watcher`的`update`方法达到通知`Watcher`更新的目的。

## 依赖收集
从前面我们可以知道一些基本事实：
`Dep`用于管理依赖元素的`Watcher`，当元素进行更新时，通过`dep.notify()`通知`Watcher`更新。那我们现在来分析整个依赖收集过程，来看看`Dep`和`Watcher`之间是如何建立有效且高效的联系的。

我们都知道，每一个Vue实例都有一个渲染`Watcher`用于渲染视图：
```js
updateComponent = () => {
  // vm._update 是把 vnode 渲染成真实的DOM结点
  // vm_render 方法通过 render函数生成 vnode
  vm._update(vm._render(), hydrating)
}

// we set this to vm._watcher inside the watcher's constructor
// since the watcher's initial patch may call $forceUpdate (e.g. inside child
// component's mounted hook), which relies on vm._watcher being already defined
new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

而`Watcher`在实例化的时候的逻辑是这样的：
```js
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  lazy: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      // 如果是 watch 侦听器的话
      // 假如侦听 a.b.c
      // 调用parsePath时会解析 a.b.c
      // 最终返回 () => c 的函数
      // 执行 this.getter 时会返回 c 的值
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }
}
```

现在把关注点放到构造函数的组后两行：
```js
this.value = this.lazy
    ? undefined
    : this.get()
```

因为渲染Watcher的`lazy`属性为`false`，因此在实例化时会触发`this.get()`。

```js
  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
```

这里需要先了解一下`pushTarget`和`popTarget`这两个函数：

它们位于：`src\core\observer\dep.js`
```js
// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}

```

从这里可以发现`Dep`有一个静态属性`target`，从`Dep`的定义可知，它是`Watcher`类型的。因为JavaScript是单线程的，所以在渲染时每次只有一个`Watcher`在执行，而`pushTarget`就是把当前渲染的`Watcher`压栈，然后链接到`Dep.target`上；当这个`Watcher`执行完毕后，就通过`popTarget`出栈。

当我们初次渲染，实例化渲染`Watcher`的时候，就通过`pushTarget`把当前的渲染`Watcher`压栈，并且通过`Dep.target`使它成为全局目前执行的`Watcher`，便于访问。

然后执行
```js
this.getter.call(vm, vm)
```

通过构造函数的过程可以知道， `this.getter`实际上就是`updateComponent`方法。所以，这里实际上是在执行`updateComponent`方法：
```js
updateComponent = () => {
  // vm._update 是把 vnode 渲染成真实的DOM结点
  // vm_render 方法通过 render函数生成 vnode
  vm._update(vm._render(), hydrating)
}
```

首先执行`vm._render()`生成VNode。在生成VNode的过程中，会通过`this.xxx`获取实例上的数据，这时候，就触发了响应式元素的`getter`：

```js
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        // 如果 obj[key] 为对象类型
        // 那么前面通过 observe(val) 变成可观察对象
        // 收集依赖
        // 当更新 obj[key][xxx] =  yyy 的时候
        // 会调用诸如 {{ obj[key] }} 的回调函数
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
```

因为`getter`最终也是要返回值，因此先获取`value`。

接下来就是依赖收集的过程。

先判断`Dep.target`是否为空，因为之前说了现在的`Dep.target`是渲染Watcher，所以肯定不为空。然后执行`dep.depend()`。

```js
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
```

`dep.depend`方法执行了渲染`Watcher`的`addDep`方法，并且把自身当作参数传入。

```js
  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```

每个Dep实例都有一个ID，渲染`Watcher`通过判断是否在`newDepId`中，如果没有，就加入到`newDeps`和`newDepsId`中，如果`depIds`也没有，调用`dep`的`addSub`方法，参数为渲染`Watcher`自身。

```js
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }
```

`addSub`方法把渲染`Watcher`添加到`subs`中。这样，依赖收集的过程就完成了。

当`this.xxx = 1`执行时，数据发生改变，就会通过`setter`调用`dep.notify()`通知`subs`中的`Watcher`进行更新。

## 清理依赖
实际上刚刚`Watcher`的`addDep`方法是非常绕的：
```js
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```

它们是在`Watcher`初始化的时候这样定义的：
```js
// 本轮的deps
this.deps = []
// 下一轮的deps
this.newDeps = []
// 本轮的depsId
this.depIds = new Set()
// 下一轮的depsId
this.newDepIds = new Set()
```

我们通过一个例子说明为什么要这样做：
```vue
<html>
  <div>
    <div v-if="flag">{{ a }}</div>
    <div v-else>{{ b }}</div>
  </div>
</html>
<script>
export default {
    data() {
        return {
            a: 11,
            b: 22,
            flag: true
        }
    }
}
</script>
```

当初次渲染时，`flag`为`true`，因此`<div v-else>{{ b }}</div>`是不会渲染的，所以渲染`Watcher`不会与`this.b`建立联系，`this.b`的更新也不会触发渲染`Watcher`的更新。

当我们设置`this.flag = false;`时，我们就渲染了下面的`div`，而上面的`div`就不渲染了。因此，这时候只需要建立`this.b`的联系就可以了。但除了建立`this.b`的联系之外，我们还需要断开`this.a`与渲染`Watcher`的关联。

#### flag 更新前
```
// 本轮的deps
this.deps = [ a's dep]
// 下一轮的deps
this.newDeps = []
// 本轮的depsId
this.depIds = [ a's depId ]
// 下一轮的depsId
this.newDepIds = []
```

#### 设置 flag = false
```
// 本轮的deps
this.deps = [ a's dep]
// 下一轮的deps
this.newDeps = [ b's dep ]
// 本轮的depsId
this.depIds = [ a's depId ]
// 下一轮的depsId
this.newDepIds = [ b's depId ]
```

当我们收集完依赖的时候，前面是还有一个方法调用没有说的，就是：
```js
popTarget()
this.cleanupDeps()
```

`popTarget`非常好理解，就是当前渲染Watcher出栈。

然后就是执行`this.cleanupDeps`方法，它的目的是为了清除依赖的关联：
```js
  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
```

首先把下一轮不存在的依赖清除，也就是通过调用元素的`dep.removeSub`：
```js
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
```

```js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;
  
  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }
}
```

然后把`depIds`与`newDepsIds`交换，`deps`与`newDeps`交换。然后清除`newDepsIds`与`newDeps`（因为变量是保存对象的引用，所以`newDepsIds`的清除实际上是清除`depsId`，`newDeps`同理。）

#### 依赖收集后，调用 cleanupDeps
```
// 本轮的deps
this.deps = [ b's dep ]
// 下一轮的deps
this.newDeps = [ ]
// 本轮的depsId
this.depIds = [ b's depId ]
// 下一轮的depsId
this.newDepIds = [  ]
```

这样，就移除了`this.a`的关联，当`this.a`更新时，就不会再触发视图的更新了。