---
title: Vue源码解析————响应式对象
date: 2019-12-24 16:30:02
tags: [前端,Vue]
---

众所周知，Vue的核心思想是数据驱动，当数据发生变化时自动更新到视图上。也有很多人也了解到Vue的响应式原理是通过`Object.defineProperty`实现的，但除了`Object.defineProperty`以外，实际上还需要配合`Dep`、`Observer`、`Watcher`，它们共同构成了Vue的响应式系统。

## initState
因为数据的初始化是发生在`created`钩子之前，因此构建响应式对象也是在这个时候。从`src\core\instance\init.js`可以看到以下代码：

```js
    // expose real self
    vm._self = vm
    // 初始化生命周期
    initLifecycle(vm)
    // 初始化事件处理
    initEvents(vm)
    // 初始化渲染函数
    initRender(vm)
    // 调用 beforeCreate 钩子
    callHook(vm, 'beforeCreate')
    // 初始化 inject
    initInjections(vm) // resolve injections before data/props
    // 初始化 data\props\methods
    initState(vm)
    // 初始化 provide
    initProvide(vm) // resolve provide after data/props
    // 数据初始化完成之后，调用 created 钩子
    callHook(vm, 'created')
```

而其中`initState`就是用来初始化`data`、`props`、`computed`、`methods`的，并且把`data`、`props`等构建为响应式对象。

`initState`位于`src\core\instance\state.js`中，为了保留主线，这里选取`data`作为分析。

```js
// 这个函数在 this._init 调用时调用
// 初始化 data / props / methods / computed / watch
// 并为它们代理到 this 上
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

对于为`data`构建响应式对象，我们关注点在于：
```js
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
```

可以看到，如果我们没有传入`data`对象，`Vue`依然会构建一个空的响应式对象。这是为什么呢？因为，虽然Vue鼓励大家在开发的时候把程序运营的所有`data`都先声明，哪怕是个空值，但它依然为开发者暴露了`$set`的API，用于添加新的`data`属性，并且手动构建为响应式对象。如果没有这个空对象，也就是根`data`属性也没有`__ob__`，那么`$set`并没有效果。具体`__ob__`是什么，下面会说到。

## initData
```js
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

可以看到，`initData`首先判断`data`是不是一个函数，如果是一个函数，就通过`getData`执行，否则就直接使用这个`data`对象。

`getData`的逻辑很简单，就是执行这个函数，返回函数返回的对象。
```js
export function getData (data: Function, vm: Component): any {
  // #7573 disable dep collection when invoking data getters
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}
```

然后把`data`中的`key`值遍历，逐一对比是否已经在`props`和`methods`中定义过了。如果已经存在，并且在开发环境，就报错提醒。如果不是就通过`proxy(vm, '_data', key)`代理到`vm`对象上。

### `proxy(vm, '_data', key)`

我们之所以能够通过`vm.xxx`可以访问到`props`、`data`、`methods`、`computed`、`watch`，都是`proxy`起的作用。

```js
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}
//
// 这里就是利用 Object.defineProperty 进行数据的代理
// this.$data.somekey => this.somekey
// this.$methods.somefn => this.somefn
// 
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

Vue通过`Object.defineProperty`把`vm.xxx`代理到`vm.$data.xxx`上，因为我们不单单可以访问`data`，也可以访问`props`，所以`key`是不能够重复的。

## observe
紧接着，就调用`observe(data, true /* asRootData */)`把`data`构建为响应式对象。

`observe`位于`src\core\observer\index.js`中：

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

`obserse`函数先判断传入的`value`是不是对象，如果不是一个对象或者是一个`VNode`，就直接返回了。如果是一个普通对象，就判断有没有`__ob__`属性，如果有，就通过赋值`ob`返回`__ob__`。然后就是做一层判断，实例化一个`Observer`，`ob = new Observer(value)`。最后返回这个`Observer`实例。

在Vue中，一旦一个对象它拥有了`__ob__`属性，那么它就是一个响应式对象。因此，整个响应式的初始化过程肯定存在于`Observer`的构造函数之中。

## Observer
```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    // 对于数组
    // 如果可以访问 __proto__ ，就 hasProto 直接把
    // 对应数组的原型链修改为： [xx,xx].__proto__ = arrayMethods
    // 否则逐个复制到 数组实例 上
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

从`Observer`的构造函数可以看到，它初始化了`value`和`dep`，而其中的`dep`是一个`Watcher`的集中管理器，具体后面会说到。然后通过`def(value, '__ob__', this)`，把`__ob__`赋值到`value`上，而`__ob__`代表的就是`this`，也就是当前`Observer`实例。

为什么不通过`value[__ob__] = this`赋值而是通过包裹一个`def()`函数呢？原因很简单，当我们通过`value[__ob__] = this`赋值的话，`__ob__`是可枚举的，也就是说，如果我们通过`Object.keys`等函数对某个对象进行`key`的遍历，那么它就会出现在遍历的结果中。因为我们在开发中当然不希望有外部代码的入侵造成一些问题，因此，`def`函数通过`Object.defineProperty`，把`__ob__`赋值在`value`上并且设置`enumerable = false`。所以`__ob__`不会出现在枚举的过程中，因此一般也不会影响到我们日常开发。

```js
export function def (obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}
```

然后就到了响应式的核心逻辑：
```js
    // 对于数组
    // 如果可以访问 __proto__ ，就 hasProto 直接把
    // 对应数组的原型链修改为： [xx,xx].__proto__ = arrayMethods
    // 否则逐个复制到 数组实例 上
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
```

这里有两个逻辑：
1. 如果`value`是数组，就调用`this.observeArray`。
2. 否则，就是一个对象，调用`this.walk`。

我们先来看逻辑1，其实也很简单，就是遍历数组中的元素，为数组中的对象构建响应式对象。
```js
  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
```

如果是对象，那么就走`this.walk`。
```js
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
```

`this.walk`的核心也是遍历`key`，然后调用`defineReactive(obj, keys[i])`，为每一个元素建立响应式。

## defineReactive
`defineReactive`是建立元素的响应式的核心函数。
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

`defineReactive`的作用是为每一个元素通过闭包的方式维护一个`dep`，`dep`用于收集`Watcher`。然后通过`Object.defineProperty`劫持该元素的`getter`和`setter`。

因为数据初始化在模板渲染之前（好像是废话），因此，当Vue通过`render`函数渲染`VNode`时，就会通过`this.xxx`获取数据，这时候就触发到了这个元素的`getter`。这时候就通过`getter`，把对应的`Watcher`添加到这个元素的`dep`中。当我们通过`this.xxx = 132`修改响应式元素时，就会触发`setter`，如果修改的值和旧值不同，那么就通过`dep.notify()`通知`Watcher`刷新视图，从而做到数据驱动。

通过`observe`的层层遍历，把`data`数据中的每一个对象都实例化了一个`Observer`，然后通过`this.walk`函数逐一遍历元素，最后调用`defineReactive`劫持`getter`和`setter`。这时候，`data`的响应式就构建完毕了。