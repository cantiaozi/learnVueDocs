# vue的diff算法

## 一、patch方法

 当数据发生变化的时候，会触发渲染 `watcher` 的回调函数updateComponent，该方法中执行了_update()方法。 

```JavaScript
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

```JavaScript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  // ...
  const prevVnode = vm._vnode
  if (!prevVnode) {
     // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  // ...
}
```

在_update()方法中，很明显，更新组件必定存在老的Vnode和新的Vnode，所以prevVnode是存在的，执行的是else中的逻辑。` __patch__ `方法最终执行的是patch方法。

```JavaScript
function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (sameVnode(oldVnode, vnode)) {
      // patch existing root node
      patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
    } else {

      // replacing existing element
      const oldElm = oldVnode.elm
      const parentElm = nodeOps.parentNode(oldElm)

      // 创建新的节点
      createElm(
        vnode,
        insertedVnodeQueue,
        // extremely rare edge case: do not insert if old element is in a
        // leaving transition. Only happens when combining transition +
        // keep-alive + HOCs. (#4590)
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      )

      // 更新父的占位符节点
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

      // 删除老的节点
      if (isDef(parentElm)) {
        removeVnodes(parentElm, [oldVnode], 0, 0)
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode)
      }
    }
}
```

patch方法中会通过 `sameVNode(oldVnode, vnode)` 判断它们是否是sameVnode的来决定走不同的更新逻辑。

```JavaScript
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



#### 新旧节点不是sameVnode的

如果新旧 `vnode` 不是sameVnode的，那么更新的逻辑非常简单，它本质上是要替换已存在的节点，大致分为 3 步：1.创建新的节点；2.更新父的占位符节点；3.删除旧的节点。

#### 新旧节点是sameVnode的

调用patchVnode方法，该方法可以看成是更新两个sameVnode节点的子节点，使oldvnode的子节点和vnode的子节点保持一致。此时不像上面那样直接删除旧的节点，插入新的节点，可以提高相同节点的利用率。



## 二、patchVnode方法

patchVnode方法的主要操作逻辑如下：

```javascript
const oldCh = oldVnode.children
const ch = vnode.children
if (isUndef(vnode.text)) {
  if (isDef(oldCh) && isDef(ch)) {
    if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
  } else if (isDef(ch)) {
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
```

如果 `vnode` 是个文本节点且新旧文本不相同，则直接替换文本内容。如果不是文本节点，则判断它们的子节点，并分了几种情况处理：

1. `oldCh` 与 `ch` 都存在且不相同时，使用 `updateChildren` 函数来更新子节点，这个后面重点讲。

2.如果只有 `ch` 存在，表示旧节点不需要了。如果旧的节点是文本节点则先将节点的文本清除，然后通过 `addVnodes` 将 `ch` 批量插入到新节点 `elm` 下。

3.如果只有 `oldCh` 存在，表示更新的是空节点，则需要将旧的节点通过 `removeVnodes` 全部清除。

4.当只有旧节点是文本节点的时候，则清除其节点文本内容。



## 三、updateChildren方法

```JavaScript
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
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
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
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue)
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

 `oldCh`和`newCh`各有两个头尾的下标`StartIdx`和`EndIdx`，相应的有四个vnode，oldStartVnode、oldEndVnode、newStartVnode和newEndVnode，这四个节点相互比较，一共有5种比较方式。在比较的过程中，变量会往中间靠，进行循环，一旦`StartIdx>EndIdx`表明`oldCh`和`newCh`至少有一个已经遍历完了，就会结束比较。 

五种比较方式：

* #### oldStartVnode和newStartVnode是sameVnode的

  ```
  patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
  oldStartVnode = oldCh[++oldStartIdx]
  newStartVnode = newCh[++newStartIdx]
  ```

  比较更新oldStartVnode和newStartVnode的子节点；oldStartIdx和newStartIdx加一；

* #### oldEndVnode和newEndVnode是sameVnode的

```
patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
oldEndVnode = oldCh[--oldEndIdx]
newEndVnode = newCh[--newEndIdx]
```

比较更新oldEndVnode和newEndVnode的子节点；oldEndIdx和newEndIdx减一；

* #### oldStartVnode和newEndVnode是sameVnode

```JavaScript
patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
oldStartVnode = oldCh[++oldStartIdx]
newEndVnode = newCh[--newEndIdx]
```

比较更新oldStartVnode和newEndVnode的子节点；将oldStartVnode.elm节点移动到oldEndVnode.elm节点前面。

* #### oldEndVnode和newStartVnode是sameVnode的

```
patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
oldEndVnode = oldCh[--oldEndIdx]
newStartVnode = newCh[++newStartIdx]
```

比较更新oldEndVnode和newStartVnode的子节点；将oldEndVnode.elm节点移动到oldStartVnode.elm节点前面。

* #### 第五种情况

如果上面四种情况都不存在，会根据oldch的key值生成一张hash表，key为旧的vnode的key值，value为旧的vnode的index下标值。

1.如果newStartVnode存在key值，则找到hash表中对应的index值。如果不存在key值，则执行findIdxInOld方法在oldch中找到与newStartVnode节点sameVnode的节点，findIdxInOld方法返回的是oldch中那个与newStartVnode节点sameVnode的节点的下标index值；findIdxInOld方法如下

```JavaScript
function findIdxInOld (node, oldCh, start, end) {
    for (let i = start; i < end; i++) {
      const c = oldCh[i]
      if (isDef(c) && sameVnode(node, c)) return i
    }
}
```



2.如果上一步的index存在，拿到oldCh中该下标的vnode，将这个vnode和newStartVnode进行比较；如果是sameVnode的话，调用patchVnode方法将这两个节点的子节点进行比较更新，并且将vnode.elm移动到oldStartVnode.elm节点前面；

```JavaScript
patchVnode(vnodeToMove, newStastartrtVnode, insertedVnodeQueue)
oldCh[idxInOld] = undefined
nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
```

3.如果不是sameVnode的话或者步骤1中的index不存在，执行createElm方法将newStartVnode插入到oldStartVnode.elm节点前面。



到最后循环结束的时候，进行下面的判断：

 1.当结束时oldStartIdx > oldEndIdx，这个时候老的vnode节点已经遍历完了，但是新的还没有。说明了新的vnode节点实际上比老的vnode节点多，也就是比真实DOM多，需要调用addVnode将剩下的（也就是新增的）vnode节点插入到真实DOM节点中去。

2.当newStartIdx > newEndIdx时，新的vnode节点已经遍历完了，但是老的节点还有剩余，说明真实DOM节点多余了，需要删除，这时候调用removeVnodes将这些多余的真实DOM删除。 



## 四、实例

假如渲染这样一系列的li节点

```html
<li v-for="item in items" :key="item.id">{{item.val}}</li>
```

更新前：

```JavaScript
items: [
    {id: 0, val: 'a'},
    {id: 1, val: 'b'},
    {id: 2, val: 'c'}
]
```

更新后：

```
items: [
    {id: 1, val: 'a'},
    {id: 3, val: 'd'},
    {id: 2, val: 'e'},
    {val: 'f'}
]
```

#### 第一次循环：

![](D:\前端开发笔记\JavaScript\learnVue\others\第一步.png)

​		第一次循环第五种情况。newStartVnode和oldch中的id为1的vnode是sameVnode的，更新vnode的子节点，并且将vnode.elm移动到oldStartVnode.elm前面，即将真实DOM中第二个节点移动到第一个节点前面。更新后真实的DOM如下图，注意此时oldch中id为0的vnode对应的是第二个a。

![](D:\前端开发笔记\JavaScript\learnVue\others\第一步结果.png)



#### 第二次循环：

![](D:\前端开发笔记\JavaScript\learnVue\others\第二步.png)

​		第二次循环第五种情况。注意，此时oldch数组中第二位值是undefined。oldch中没有和newStartneVnode节点sameVnode的节点。执行createElm方法，根据newStartneVnode创建一个新的节点并插入到oldStartVnode.elm前面。更新后真实的DOM如下图。

![](D:\前端开发笔记\JavaScript\learnVue\others\第二步结果.png)



#### 第三次循环：

![](D:\前端开发笔记\JavaScript\learnVue\others\第三步.png)

​		第三次循环符合第四种情况。oldEndVnode和newStartVnode是sameVnode的，调用patchVnode方法更新子节点，然后将oldEndVnode.elm移动到oldStartVnode.elm前面。更新后真实的DOM如下图。

![](D:\前端开发笔记\JavaScript\learnVue\others\第三步结果.png)



#### 第四次循环：

![](D:\前端开发笔记\JavaScript\learnVue\others\第四步.png)

在此次循环中，oldEndVnode是undefined，因此将oldEndIdx减一。



#### 第五次循环：

![](D:\前端开发笔记\JavaScript\learnVue\others\第五步.png)

第五次循环符合第五种情况。newStartVnode没有key值，在oldch中没有找到与newStartVnode节点sameVnode的节点，直接执行createElm方法，根据newStartneVnode创建一个新的节点并插入到oldStartVnode.elm前面。更新后真实的DOM如下图。

![](D:\前端开发笔记\JavaScript\learnVue\others\第五步结果.png)



#### 循环结束后：

​		循环结束后，newStartIdx大于newEndIdx，执行removeremVnodes方法删除多余节点。此时的oldStartIdx为0，oldEndIdx为0。removeremVnodes方法删除的是oldStartIdx和oldEndIdx之间的节点，即删除oldStartVnode.elm，也就是真实DOM中最后一个节点a。



## 五、总结

1.v-for指令需要设置key值，因为设置了key值，可以更好的利用已有的节点，当更新前的节点中有和更新后的节点key值相同的节点，只需要移动位置即可。如果没有key值，需要生成新的节点再插入到DOM中，代价较大。

2.diff算法是同级比较，同级比较完了，才会去比较子节点。