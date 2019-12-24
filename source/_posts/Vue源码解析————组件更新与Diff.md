---
title: Vue源码解析————组件更新与Diff
date: 2019-12-24 16:31:26
tags: [前端,Vue]
---

在阅读本文之前，请确保你已经知道`渲染Watcher`究竟是做什么的。

Vue 组件的渲染的核心在于`渲染Watcher`，以及每个组件对应的`updateComponent`函数。

```js
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

当组件更新时，会调用`渲染Watcher`的`update`方法，而`update`方法在`渲染Watcher`中实际上调用了`get`方法。

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

渲染Watcher中的`getter`方法就是`updateComponent`，在初始化渲染Watcher的时候传入的。因此，除了常规的Watcher压栈出栈操作，`updateComponent`的调用显然是组件更新的核心内容。

```js
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

`vm._render`函数的执行返回`VNode`，传入到`vm._update`方法中。

`_update`方法位置在`src\core\instance\lifecycle.js`。

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    // 缓存旧VNode
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    // 保存新Vnode
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```

`_update`的第一个参数是新的Vnode，第二个参数是SSR相关的，对于SPA，第二个参数为`false`就行。

首先缓存`vm._vnode`到`prevVnode`，这是之前的VNode，并且为vm实例赋值一个新的VNode。然后再判断是否拥有旧VNode。对于初次渲染，是没有旧VNode的，所以初次渲染走`true`的逻辑；而对于组件更新，下面的逻辑，通过`patch`更新DOM：
```js
// updates
vm.$el = vm.__patch__(prevVnode, vnode)
```

`vm.__patch__`是在哪里定义上去的呢？因为VNode的最大好处在于使得虚拟DOM->真实DOM的逻辑跨平台化，因此`patch`的过程实际上是平台相关的。对于`web`平台，`vm.__patch__`是在这里添加上去的：`src\platforms\web\runtime\index.js`。

```js
import { patch } from './patch'
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

然后找到`src\platforms\web\runtime\patch.js`：

```js
import * as nodeOps from 'web/runtime/node-ops'
import { createPatchFunction } from 'core/vdom/patch'
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

// the directive module should be applied last, after all
// built-in modules have been applied.
const modules = platformModules.concat(baseModules)

export const patch: Function = createPatchFunction({ nodeOps, modules })
```

真正的`patch`函数通过对`createPatchFunction`的柯里化返回的，这里使用柯里化的好处在于，`createPatchFunction`的过程只在首次初始化的使用执行，并且，由于Vue是跨平台的，因此，对于一些直接操作DOM的API，不同平台会有不同的方法，这里通过`nodeOps`进行了封装，使得即使在不同平台下进行`patch`，`patch`中使用的API都是一样的，屏蔽了不同平台的差异。柯里化可以做到对`patch`的一些配置进行固化，以后使用`patch`就不用在判断平台了。

因此，真正的`patch`操作位于：`src\core\vdom\patch.js`文件中：
```js
return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
```

因为组件更新时会存在`oldVnode`并且`oldVnode`只是VNode类型而不是真实的DOM，因此会走到这个逻辑：
```js
if (!isRealElement && sameVnode(oldVnode, vnode)) {
    // patch existing root node
    patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
}
```

然后我们看看`patchVnode`函数，为了保留最原始组件更新的逻辑，下面删掉了异步组件的逻辑：
```js
function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    let i
    const data = vnode.data
    // 组件Vnode调用 prepatch 钩子
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    // update children
    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    // 如果新Vnode不是文本节点：
    //   1. 如果 新Vnode的children不等于旧Vnode的children（引用不同），那么调用 updateChildren
    //   2. 如果 旧Vnode是文本节点，那么删除文本内容然后将新Vnode的Children添加上去
    //   3. 如果 新Vnode 没有 children，那么直接删除 children
    //   4. 如果 新Vnode 没有 chidren 并且旧Vnode是文本节点，那么直接删除text内容
    //  如果新Vnode是文本节点且文本内容不等于旧Vnode，那么更新文本内容
    //
    //  最后调用 postpatch 钩子（组件VNode)
    //
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

新旧VNode的`children`比对可以看到有以下几个逻辑：
- 如果新VNode不是文本节点
  1. 如果`oldCh`与`ch`不相等，因为`children`为一个VNode数组，因此这里的不相等是引用不同。然后调用`updateChildren`进行`children`的比对和更新。
  2. 如果`oldCh`不存在，说明原来该节点的没有孩子节点的，因此把新VNode的`children`添加到`DOM`即可。
  3. 如果`ch`不存在，说明新VNode没有孩子节点，所以删除原来DOM上的孩子节点。
  4. 如果旧VNode是文本节点，并且`ch`不存在。因为新VNode不是文本节点，因此把DOM的文本内容设置为空即可。
- 如果新Vnode为文本节点并且文本内容与旧VNode不相同，则更新文本内容。

`addVnodes`的逻辑实际上就是为`ch`中每一个`VNode`调用`creatElm`逻辑，这个函数是把VNode渲染真实DOM并插入到父元素上的。

```js
function addVnodes (parentElm, refElm, vnodes, startIdx, endIdx, insertedVnodeQueue) {
    for (; startIdx <= endIdx; ++startIdx) {
      createElm(vnodes[startIdx], insertedVnodeQueue, parentElm, refElm, false, vnodes, startIdx)
  }
}
```

这里的重点在于调用`updateChildren`，通过`updateChildren`进行子元素的比对以及DOM更新，这里是Diff算法的核心内容。

```js
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

这里需要注意的是`sameNode`的逻辑，它判断的不是两个VNode的引用，而是对于同步的Vnode，比对`key`、`tag`、`isComment`、`inputType`等是否相等，如果都相等则为同一个VNode；对于异步Vnode，比对异步函数是否相同。

```js
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

整个比对的逻辑是这样的：
1. 如果`sameVnode(oldStartVnode, newStartVnode)`，两个`children`的起始Vnode相等，那么就进入`patchVnode`更新，DOM的顺序并不改变，新旧的DOM节点依然在起始位置。更新起始位置为下一个。
2. 如果`sameVnode(oldEndVnode, newEndVnode)`，两个`children`的末尾Vnode相等，那么就进入`patchVnode`更新，DOM的顺序并不改变。新旧的DOM节点依然在末尾位置。更新起始位置为上一个。
3. 如果`sameVnode(oldStartVnode, newEndVnode)`，旧起始节点跟新末尾节点是同一个VNode，说明更新后原来的节点位置移到的后方。那么把该节点从DOM的位置向右移动，然后进入`patchVnode`逻辑。更新指针。
4. 如果`sameVnode(oldEndVnode, newStartVnode)`，旧末尾节点和新起始节点是同一个VNode，说明更新后原来的节点位置移动到了前面，那么把该节点从DOM的位置向左移动，然后进入`patchVnode`逻辑。更新指针。
5. 如果都不满足上面的逻辑，那么就会通过`newStartVnode`的key获取到旧VNode的引用，记为`vnodeToMove`，如果`vnodeToMove`不存在，那么直接创建一个新元素。否则，比对这两个节点，如果`sameVnode`为真，则移动DOM的位置，然后`patchVnode`进行复用。否则，虽然Key相等，但已经是完全不同的元素节点，则创建一个新的元素插入。

结束的条件是`oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx`为假，所以当`oldCh`或`ch`的双指针任意一个相互超越的时候，上面的逻辑就走完了。但是这时候还有一个问题是，`oldCh`或者`ch`其中一方还有`vnode`还没有更新完全。所以就走下面的逻辑：

```js
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
```

1. `oldStartIdx > oldEndIdx`说明旧的VNode节点已经遍历完了，新VNode还没有遍历完。所以新VNode节点是原来并没有的，是新增加了，所以就直接调用`addVnodes`添加到DOM上。
2. `newStartIdx > newEndIdx`，说明新的VNode节点已经遍历完了，旧的VNode还没遍历完。所以剩下的旧VNode在更新后会被删掉，因此调用`removeVnodes`把`oldCh`的VNode对应的真实DOM节点从父元素上删除。


为了更好地说明情况，我们假定以下情景：
```vue
<html>
  <div>
    <ul>
      <li v-for="ch in items" :key="ch">{{ ch }}</li>
    </ul>
  </div>
</html>
<script>
export default {
    data() {
        return {
            items: ['A', 'B', 'C', 'D']
        }
    },
    methods: {
        onChange() {
            this.items.push('E').reverse();
        }
    }
}
</script>
```

假如执行了`onChange`函数，这里为`items`添加了新元素`E`然后反转了原来的数组。

1. `patch`

![image](http://static-cdn.lxzmww.xyz/diff/diff1.jpg)

2. `sameVnode(oldStartVnode, newEndVnode)`

![image](http://static-cdn.lxzmww.xyz/diff/diff2.jpg)

3. `sameVnode(oldStartVnode, newEndVnode)`

![image](http://static-cdn.lxzmww.xyz/diff/diff3.jpg)

4. `sameVnode(oldStartVnode, newEndVnode)`

![image](http://static-cdn.lxzmww.xyz/diff/diff4.jpg)

5. `sameVnode(oldStartVnode, newEndVnode)`

![image](http://static-cdn.lxzmww.xyz/diff/diff5.jpg)

6. 此时`oldCh`已经遍历完了，剩下一个新Vnode。因此调用`addVnode`添加到DOM上。

![image](http://static-cdn.lxzmww.xyz/diff/diff6.jpg)