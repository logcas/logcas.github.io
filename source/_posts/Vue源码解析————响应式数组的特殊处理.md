---
title: Vue源码解析————响应式数组的特殊处理
date: 2019-12-27 14:55:01
tags: [前端,Vue]
---

在Vue中，数组的响应式是通过特殊处理实现的，不是说`Object.defineProperty`不能劫持数组的下标元素更新，而是，Vue作者尤大表示是性能的问题。Vue通过别的方法，转换了思路，对数组进行了特殊处理。

## Observer
如果你记得`observe(data)`之后是为`data`定义了一个`__ob__`对象，那么，很容易在`Observer`里面找到针对数组的一些逻辑：
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
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

在构造函数中，对`value`进行了判断，如果`value`是一个数组，就走以下逻辑：
```js
if (hasProto) {
  protoAugment(value, arrayMethods)
} else {
  copyAugment(value, arrayMethods, arrayKeys)
}
```

`hasProto`位于`src\core\util\env.js`中，具体实现是这样的：
```js
// can we use __proto__?
export const hasProto = '__proto__' in {}
```

`hasProto`的目的是查询是否存在`__proto__`属性，而这个属性是用来访问原型的属性。这个属性我相信很多人都很常用，所以我特意查了一下哪些浏览器是不支持`__proto__`的：

![image](http://static-cdn.lxzmww.xyz/__proto__.JPG)

果不其然，就是IE。因为Vue支持IE8+的浏览器，而对于IE8、IE9、IE10，是不支持`__proto__`的，那么就会走`copyAugment`的逻辑。

对于可以通过`__proto__`访问到原型的浏览器，就走`protoAugment(value, arrayMethods)`。

## `protoAugment(value, arrayMethods)`
```js
function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}
```

`protoAugment`实际上就是修改了`value`的原型为`arrayMethods`。

而`arrayMethods`是什么呢? 它位于`src\core\observer\array.js`中：

```js
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)
```

实际上，`arrayMethods`是通过原型继承自`Array.prototype`，所以，继承的关系没有变。只是Vue通过一些处理，改写了一些方法（或者说在这些原生方法上增加了一些劫持操作）。

整个逻辑的目标是一些可以修改原数组的方法：
```js
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
```

为什么需要这样的处理呢？因为数组是走`this.observeArray(value)`，而这个方法的目的只是遍历数组，为数组中的对象构建响应式。上述的一些修改原数组的方法，本质上也是通过索引直接修改元素的，这部分是检测不到的，如果不改写一下逻辑，那么响应式就会无效。

那Vue是怎么处理的呢？
```js
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
//  对于上面 methodsToPatch 的方法
//  逐一进行手动通知更新
//
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```

首先遍历`methodsToPatch`，缓存原生的方法到`original`。通过`def`在`arrayMethods`上定义一个同名的方法。

为什么通过`def`定义？因为`def`通过`Object.defineProperty`定义且`enumerable`为`false`，这样定义是不可枚举的。

为什么定义在`arrayMethods`上？这样可以屏蔽掉`Array.prototype`上的同名方法。

然后先执行原生的方法，获取返回值。实际上这层劫持本身执行的还是原生方法，但是对于`push`、`unshift`和`splice`这些可以原地新增数组元素，需要为新增的元素数组中的每个元素调用`observe`，使得如果元素是对象，那么构建响应式对象。

最后，手动调用`ob.dep.notify()`通知`Watcher`进行视图的更新。

## $set
虽然通过劫持原生方法，手动通知视图更新可以解决原生方法无法触发响应式更新的问题。但是，通过下标修改数组元素但无法自动派发更新的问题依然没有解决，也无法解决。因此，`Vue`提供了`$set`的API，通过这个API就可以为新增的元素添加响应式、或者修改数组的元素，也会派发更新。

其实思想的一样的，都是通过劫持一些操作然后手动调用：

```js
export function set (target: Array<any> | Object, key: any, val: any): any {
  // 如果 target 没有定义
  // 或者是原始类型
  // 那么报错
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  // 如果 target 是数组，那么直接修改就可以了
  // 因为 splice 方法已经被改写
  // 因此响应式依然有效的
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  // 如果对应的 key 已经在 target 中
  // 那么直接修改就可以
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  // 然后就是 key 处理不在 target 中的情况
  const ob = (target: any).__ob__
  //
  //
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  // 如果没有 observer 
  // 那么就说明 target 本来就没有响应式（或者不需要响应式）
  // 所以直接修改即可
  if (!ob) {
    target[key] = val
    return val
  }
  // 否则
  // 为这个新添加的属性进行响应式处理
  defineReactive(ob.value, key, val)
  // 主动触发更新
  ob.dep.notify()
  return val
}
```

从上面的逻辑可以看到，如果是调用`set(arr, 0, 100)`修改数组，那么最终的实现是通过`splice`去修改的：
```
  // 如果 target 是数组，那么直接修改就可以了
  // 因为 splice 方法已经被改写
  // 因此响应式依然有效的
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
```

此时`splice`已经被劫持，因此调用`splice`会派发更新。

如果`target`本身没有`ob`，也就是本身不是响应式的，那么直接修改。

否则，通过`defineReactive(ob.value, key, val)`为这个属性构建响应式。

最后也是手动派发更新：
```js
ob.dep.notify()
return val
```

## 总结
可以看到，无论是处理数组还是暴露`$set`的API，核心的思想还是劫持操作+手动派发更新。对数组遍历而构建响应式对象，可能会造成性能问题，是目前`Object.defineProperty`的缺陷，期待ES6的`Proxy`能解决这些问题。