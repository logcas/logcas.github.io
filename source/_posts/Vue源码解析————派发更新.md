---
title: Vue源码解析————派发更新
date: 2019-12-25 13:25:19
tags: [前端,Vue]
---
因为在构建响应式元素时，`data`上的属性已经被劫持了`getter`和`setter`。当我们修改`data`的时候，通过`this.a = 123`的方式，会触发`a`的`setter`。

所以，先来回顾一下构建响应式的核心函数`defineReactive`：
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

这次解析派发更新的过程，我们就把目光放在`setter`开始的逻辑上：
```js
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
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
```

首先，如果之前缓存了`getter`，那么就通过`getter`求原来的值，如果没有，就从`val`获取。然后比较`value`和`newValue`是否相等，如果相等就返回了，不需要触发视图的更新。

但是这里需要注意值相等的逻辑：

```
newVal === value || (newVal !== newVal && value !== value)
````

前半部分`newVal === value`很容易理解，就是一个严格比较。后半部分`(newVal !== newVal && value !== value)`看似有点啰嗦，但实际上它是为了比较`NaN`元素而设立的。如果`NaN`之间直接比较，会返回`false`，任意两个NaN都是不等的。这里通过新旧值的比较和旧值与自己比较，确定如果新旧值都是`NaN`，那么也直接返回，因为值并没有变。

如果这个元素原本有`getter`但是没有`setter`，那么说明它是一个只读元素，那么劫持后也不能修改。

```
if (getter && !setter) return
```

接下来，如果原来有缓存的`setter`，那么就通过`setter`改变值，否则就直接赋值。
```
if (setter) {
  setter.call(obj, newVal)
} else {
  val = newVal
}
```

因为对某个元素赋值后，如果是对象，那么需要重新遍历这个对象的所有属性，使得它的属性也成为响应式元素：

```
childOb = !shallow && observe(newVal)
```

最后就是通过`dep`通知`Watcher`更新：
```
dep.notify()
```

## `dep.notify`
```
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
```

`dep.notify`实际上是遍历实例上的`subs`，之前我们了解到`subs`是一个`Watcher`数组。逐一执行`Watcher`实例上的`update`方法。

## `Watcher.prototype.update`
因为`subs`逐一执行`Watcher`实例的`update`方法，所以我们把目光转移到`Watcher`上：
```js
  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
```

`update`方法做了一层判断：
1. 如果是`lazy`，就设置`this.dirty = true`，这个是一般是计算属性的`computed watcher`使用的，对于渲染`Watcher`的`this.lazy`为`false`。
2. 如果是`sync`，同步执行，一般来说，Vue的默认都是异步的，因此`this.sync = false`。
3. 最后，就执行`queueWatcher(this)`，把`this`作为参数传入。

## queueWatcher
`queueWatcher`函数在`src\core\observer\scheduler.js`：
```js
const queue: Array<Watcher> = []
let has: { [key: number]: ?true } = {}
let waiting = false
let flushing = false
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

这里先获取`Watcher`的`id`，通过`has[id]`判断时候存在。这个的作用是确保每一个`Watcher`在每一轮的渲染中只会被添加一次，即使修改了多个数据，对某一个Watcher进行了多次通知更新，最终`Watcher`只会添加一次。

然后就是判断是否正在执行更新：
```
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
```

如果更新还没开始，那么就直接加入到`queue`中，否则就在`queue`中寻找合适的位置，把当前的`Watcher`添加到第一个`id`大于它的`Watcher`的前面：

```
如果目前`queue`是这样的：
[1,3,6]

现在`queue`中的Watcher正在依次执行更新，但此时有一个id=4的`Watcher`添加进来：
[1,3,4,6]
```

接下来就是判断`!waiting`，`waiting`代表着`flushSchedulerQueue`是否已经加入到`nextTick`中，如果是，则代表`flushSchedulerQueue`正在执行，不需要做任何操作；如果否，则说明`flushSchedulerQueue`还未执行，那么现在把它加入到`nextTick`中执行。

```js
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
```

`nextTick`代表什么呢？我们用`$nextTick`的时候，通过文档可以知道，`$nextTick`的执行时机就是每次视图渲染完毕的下一个`Tick`的时候。现在，我们只需要知道，Vue是异步渲染的，因此，根据浏览器的`Event loop`机制，`nextTick`就是把`flushSchedulerQueue`放到宏任务或微任务队列中，等待当前的同步逻辑执行完，再执行任务队列中的任务。

为什么是`宏任务或微任务`？`nextTick`的具体实现以后会说到，它会根据浏览器的兼容性不同做不同的降级处理。

那么到这里就很显然，`flushSchedulerQueue`承载着调用`Watcher`更新视图的任务。

## `flushSchedulerQueue`
```js
/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

从上面的逻辑可以看到，首先是对`queue`中的`Watcher`进行排序，根据`Watcher`ID进行升序排序：
1. 因为组件的更新是先父后子的，创建也是，因此父组件的渲染Watcher是先定义于子组件的渲染Watcher的，因此，父的Watcher比子Watcher在前。
2. 组件的`User Watcher`必须在渲染`Watcher`之前，因为，数据初始化的时候，对于`watch`属性，会有一个对应的`User Watcher`。而渲染`Watcher`是在数据初始化之后定义的。
3. 如果一个组件在父组件的`Watcher`执行过程中销毁，那么这个组件的`Watcher`可以跳过。

总的来说，可以归纳为这样：
1. 对于父子之间的渲染`Watcher`而言，父先于子。
2. 对于同一个Vue实例的渲染Watcher和UserWatcher而言，UserWatcher先于渲染Watcher。

排序的目的实际上就是为了确保**父的渲染先于子的渲染，处理数据的Watcher先于渲染视图的Watcher。**

排序完毕后，就是遍历`queue`逐一执行`before`和`run`方法。

我们把目光回到Watcher的`run`方法：
```js
  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
```

对于渲染`Watcher`而言，实际上就是为了这一句：
```
const value = this.get()
```

Watcher的`get`函数之前在渲染Watcher初始化的时候在构造函数中执行过，如果你记得的话，它实际上就是进行压栈、执行`updateComponent`方法、出栈。

```js
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

```js
    updateComponent = () => {
      // vm._update 是把 vnode 渲染成真实的DOM结点
      // vm_render 方法通过 render函数生成 vnode
      vm._update(vm._render(), hydrating)
    }
```

然后再执行`vm._render`的时候，又会触发**依赖收集**的过程，会更新渲染`Watcher`的`deps`和`newDeps`，也就是重新回到了依赖收集的过程，收集完毕之后，通过`vm._update`方法更新视图。而整个视图，经过修改属性到派发更新到Watcher执行`updateComponent`方法的过程也执行完毕了，通过`vm._update`方法后整个视图也重新进行了渲染。

执行完`queue`里面的所有Watcher之后，回到`flushSchedulerQueue`逻辑，还有几行代码：

```js
  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)
```

首先是获取`activatedQueue`和`updateQueue`，前者是对应`keep-alive`包裹的组件，后者是一般的组件。

然后就是调用`resetSchedulerState`把一些变量设置初始值，清除`queue`。
```js
/**
 * Reset the scheduler's state.
 */
function resetSchedulerState () {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}
```

最后就是对于`keep-alive`的组件，触发`activated`钩子，对于一般的组件，触发`updated`钩子。

