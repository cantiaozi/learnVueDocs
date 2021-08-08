```
import Vue from 'vue'
let A = {
    template: '<div class="a">'  
        + '<p>A comp</p>'
        + '</div>',
    name: 'A',
    mounted() {
        console.log("A mounted")
    },
    activated() {
        console.log("A activated")
    },
    deactivated() {
        console.log("A deactivated")
    }
}

let B = {
    template: '<div class="b">'  
        + '<p>B comp</p>'
        + '</div>',
    name: 'B',
    mounted() {
        console.log("B mounted")
    },
    activated() {
        console.log("B activated")
    },
    deactivated() {
        console.log("B deactivated")
    }
}

new Vue({
    el: '#app',
    template: '<div>' + '<keep-alive>' + '<component :is="currentComp"></component>'
        + '</keep-alive>'
        + '<button @click="change">change</button>'
        + '</div>',
    data() {
        return {
            currentComp: 'A'
        }
    },
    components: {
        A,
        B
    },
    methods: {
        change() {
            this.currentComp = this.currentComp === 'A' ? 'B' : 'A'
        }
    }
})
```

本文以上面的这个例子来说明keep-alive的源码。

## 一、keep-alive生成vnode

内置组件在使用前，也和普通组件一样是需要注册的，不过内置组件，vue已经帮我们注册了。内置组件keep-alive是在core/global-api/index.js中initGlobalAPI函数内注册的。

```JavaScript
//builtInComponents指向的就是keep-alive的定义
extend(Vue.options.components, builtInComponents)
```

keep-alive的定义如下。render函数是重点，返回了实际要渲染的vnode。keep-alive实例中的keys数组作用是什么呢？其实是为了实现一种缓存的策略，LRU思想。

LRU 最近最久未使用

 LRU的设计原理就是，当数据在最近一段时间经常被访问，那么它在以后也会经常被访问。这就意味着，如果经常访问的数据，我们需要然其能够快速命中，而不常访问的数据，我们在容量超出限制内，要将其淘汰。  每次访问的数据都会放在栈顶，当访问的数据不在内存中，且栈内数据存储满了，我们就要选择移除栈底的元素，因为在栈底部的数据访问的频率是比较低的。所以要将其淘汰。 

当keep-alive实例中缓存的vnode数量大于max的时候，需要清理缓存，这是就将keys数组中的第一个值代表的vnode清除掉，因为这个vnode最久没有使用到。当keep-alive渲染某个组件vnode的时候，需要将这个vnode的key给push到数组中，并删除原来的数组中的相同的key，这样就代表这个vnode是最近使用的。

```JavaScript
export default {
  name: 'keep-alive',
  abstract: true,

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]//缓存的组件vnode的最大值
  },

  created () {
    //缓存所有的vnode
    this.cache = Object.create(null)
    //keys保存的是cache中的所有key
    this.keys = []
  },

  destroyed () {
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  mounted () {
    this.$watch('include', val => {
      pruneCache(this, name => matches(val, name))
    })
    this.$watch('exclude', val => {
      pruneCache(this, name => !matches(val, name))
    })
  },

  //render函数是重点，render会生成vnode
  render () {
    //拿到keep-alive组件中的没有名字的插槽，即默认插槽。在本文的例子中，就是组件A
    //的占位符vnode，因为动态组件再_createElement生成vnode的时候，生成的是
    //A组件的占位符vnode
    const slot = this.$slots.default
    //拿到插槽中第一个组件节点的vnode
    //keep-alive只会缓存组件节点，对于普通的节点是没有作用的。
    const vnode: VNode = getFirstComponentChild(slot)
    //拿到缓存的组件节点vnode的componentOptions
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) {
      // check pattern
      //拿到缓存的组件的名称
      const name: ?string = getComponentName(componentOptions)
      const { include, exclude } = this
      //没有匹配上include或者匹配到exclude，不需要做缓存，直接返回子组件中的第一个组件节点
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      const { cache, keys } = this
      //给组件vnode取定一个key
      const key: ?string = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (#3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        //如果该组件vnode在缓存cache中存在，就将vnode.componentInstance指向缓存中的
        //那个vnode的componentInstance
        vnode.componentInstance = cache[key].componentInstance
        // make current key freshest
        remove(keys, key)
        keys.push(key)
      } else {
        //如果该组件vnode在缓存cache中不存在
        cache[key] = vnode
        keys.push(key)
        // prune oldest entry
        //当缓存的组件vnode的数量大于max，需要清理缓存
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }
	  //注意这个vnode是keep-alive的第一个组件子节点占位符vnode，
      //而不是keep-alive的占位符vnode
      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
}

export function getFirstComponentChild (children: ?Array<VNode>): ?VNode {
  if (Array.isArray(children)) {
    for (let i = 0; i < children.length; i++) {
      const c = children[i]
      if (isDef(c) && (isDef(c.componentOptions) || isAsyncPlaceholder(c))) {
        return c
      }
    }
  }
}

function getComponentName (opts: ?VNodeComponentOptions): ?string {
  return opts && (opts.Ctor.options.name || opts.tag)
}

//清理缓存
function pruneCacheEntry (
  cache: VNodeCache,
  key: string,
  keys: Array<string>,
  current?: VNode
) {
  const cached = cache[key]
  //当要清理的缓存的vnode不是当前正在渲染的vnode，就执行组件的destory方法销毁组件。
  if (cached && (!current || cached.tag !== current.tag)) {
    cached.componentInstance.$destroy()
  }
  //将需要清理的的组件vnode从chche对象中去掉
  cache[key] = null
  remove(keys, key)
}
```

mounted函数中监听了include和exclude两个prop的变化，当变化时，判断cache中缓存的所有vnode是否满足匹配。不能满足匹配的vnode会被清除出cache缓存中。

destroyed函数中清空cache对象中的所有缓存。

keep-alive组件在render生成vnode的过程中，执行了上面的render方法，返回了

## 二、keep-alive渲染到dom

组件被渲染到页面上，是在patch过程中实现的。在patch函数中，会调用createElm函数。该函数中会调用createComponent函数。

```JavaScript
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
      //执行init这个钩子
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        i(vnode, false /* hydrating */)
      }
      // after calling the init hook, if the vnode is a child component
      // it should've created a child instance and mounted it. the child
      // component also has set the placeholder vnode's elm.
      // in that case we can just return the element and be done.
      if (isDef(vnode.componentInstance)) {
        initComponent(vnode, insertedVnodeQueue)
        insert(parentElm, vnode.elm, refElm)
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
        }
        return true
      }
    }
  }
```

当一个组件在patch过程中遇到keepalive组件的时候，调用init钩子方法。因为此时keep-alive组件的占位符vnode的componentInstance（即组件实例）还不存在，因此执行else中的逻辑，调用createComponentInstanceForVnode方法创建keep-alive组件的实例。然后调用$mount方法，$mount方法中会执行keep-alive的render函数生成vnode，即上一小节中的render函数，生成的是keep-alive的第一个组件子节点的占位符vnode。

```JavaScript
init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
```

在本文例子中，注意这个占位符vnode不是component组件的占位符vnode，而是A组件的占位符vnode。因为动态组件在调用 _createElement函数生成占位符vnode的时候，有下面这样的逻辑。因此动态组件实际生成的是A组件的占位符vnode。当点击change按钮切换时，同样动态组件生成的也是B组件的占位符vnode。所以在keep-alive的实例中的cache对象中有两个vnode值，一个是A的占位符vnode，一个是B的占位符vnode。

```javascript
function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
) {
	......
	if (isDef(data) && isDef(data.is)) {
        tag = data.is
    }
    ......
    else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } 
}
```

紧接着上一步调用keep-alive中的定义的render生成渲染vnode（该渲染vnode就是第一个子组件的占位符vnode）后，又会执行patch，渲染到页面上。在patch的过程中，会调用createElm函数。该函数中会调用createComponent函数。因为此时的vnode是第一个子组件的占位符vnode，因此又会调用init等函数将第一个子组件给render成渲染vnode，再patch到页面上。

当点击change按钮后再次点击change按钮，切换回A组件的时候。针对keep-alive这个组件的vnode而言，会执行patchVnode这个函数。该函数中会执行prepatch钩子。prepatch钩子中执行updateChildComponent函数。updateChildComponent函数中判断keep-alive组件下面有子节点，因此触发keep-alive的重新渲染。

```javascript
function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    ......
  	let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }
    ......
  }
    
prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
}
        
export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
    ......
    //因为keep-alive组件下面有子节点，故renderChildren为true
    const needsForceUpdate = !!(
        renderChildren ||               // has new static slots
        vm.$options._renderChildren ||  // has old static slots
        hasDynamicScopedSlot
    )
    ......
    if (needsForceUpdate) {
        //重新更新$slots对象，并且强制keep-alive更新
        //keep-alive组件的更新是在这里触发的，patchVnode并不能触发组件的更新，
        //patchVnode只能触发html标签的更新
        vm.$slots = resolveSlots(renderChildren, parentVnode.context)
        vm.$forceUpdate()
    }
}
```

keep-alive重新渲染时，执行keep-alive的render函数时，返回的是A组件的占位符vnode，但是这个vnode的componentInstance属性是第一次生成的vnode的componentInstance。

在上面生成了keep-alive的渲染vnode之后就进入patch流程，因为是更新过程，oldVnode和vnode分别是组件B和组件A的占位符vnode，因此不会执行patchVnode。而是执行createElm函数，createElm函数中执行createComponent函数。此次执行createComponent函数时，vnode是A组件的占位符vnode，vnode的data的keepAlive为true，且vnode的componentInstance存在，因此isReactivated为true。接着执行init钩子函数，再init钩子函数中，满足if的逻辑，又去执行prepatch钩子（A组件的prepatch钩子和keep-alive没什么关系），而不是走else逻辑中vnode的componentInstance的创建的逻辑，也就不会走vnode的render生成渲染vnode，patch到页面成为真实dom等步骤。

回到createComponent中，接着去执行initComponent方法，将vnode.componentInstance上的$el赋给vnode的elm，这个也就是组件A缓存的vnode的实例上的dom节点。initComponent方法后执行insert插入。最后执行reactivateComponent函数。

```JavaScript
function initComponent (vnode, insertedVnodeQueue) {
    if (isDef(vnode.data.pendingInsert)) {
      insertedVnodeQueue.push.apply(insertedVnodeQueue, vnode.data.pendingInsert)
      vnode.data.pendingInsert = null
    }
    vnode.elm = vnode.componentInstance.$el
    if (isPatchable(vnode)) {
      invokeCreateHooks(vnode, insertedVnodeQueue)
      setScope(vnode)
    } else {
      // empty component root.
      // skip all element-related modules except for ref (#3455)
      registerRef(vnode)
      // make sure to invoke the insert hook
      insertedVnodeQueue.push(vnode)
    }
}

function reactivateComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i
    // hack for #4339: a reactivated component with inner transition
    // does not trigger because the inner node's created hooks are not called
    // again. It's not ideal to involve module-specific logic in here but
    // there doesn't seem to be a better way to do it.
    let innerNode = vnode
    while (innerNode.componentInstance) {
      innerNode = innerNode.componentInstance._vnode
      if (isDef(i = innerNode.data) && isDef(i = i.transition)) {
        for (i = 0; i < cbs.activate.length; ++i) {
          cbs.activate[i](emptyNode, innerNode)
        }
        insertedVnodeQueue.push(innerNode)
        break
      }
    }
    // unlike a newly created component,
    // a reactivated keep-alive component doesn't insert itself
    //再次执行插入，其实没必要，因为上面已经插入过了。
    insert(parentElm, vnode.elm, refElm)
  }
```

## 三、keep-alive的生命周期

在patch函数的最后会执行invokeInsertHook函数。invokeInsertHook函数，该函数遍历queue数组（数组中的每个值都是一个vnode），执行每个vnode的insert钩子函数。

```JavaScript
function invokeInsertHook (vnode, queue, initial) {
    // delay insert hooks for component root nodes, invoke them after the
    // element is really inserted
    if (isTrue(initial) && isDef(vnode.parent)) {
      vnode.parent.data.pendingInsert = queue
    } else {
      for (let i = 0; i < queue.length; ++i) {
        queue[i].data.hook.insert(queue[i])
      }
    }
}

insert (vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    //在组件初次渲染而非更新的渲染，走到此步的时候，_isMounted为false
    //调用mounted钩子函数
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true
      callHook(componentInstance, 'mounted')
    }
    if (vnode.data.keepAlive) {
      //context在本文的实例中是new vue的实例
      if (context._isMounted) {
        // vue-router#1212
        // During updates, a kept-alive component's child components may
        // change, so directly walking the tree here may call activated hooks
        // on incorrect children. Instead we push them into a queue which will
        // be processed after the whole patch process ended.
        queueActivatedComponent(componentInstance)
      } else {
        activateChildComponent(componentInstance, true /* direct */)
      }
    }
}
```

insert钩子函数中，对于keep-alive组件包裹的组件（本文实例中的A组件、B组件等），vnode.data.keepAlive为true，当vnode的环境context已经挂载完通过queueActivatedComponent函数去执行组件的activated钩子函数。当未挂载完会执行activateChildComponent方法，该函数主要是执行组件的activated钩子函数，并且遍历所有子组件，执行所有子组件的activated钩子函数。

activateChildComponent函数中的 _directInactive和 _inactive等变量确保一个组件的activated钩子函数在patch的过程中执行过之后就不再执行了。这两个变量是在core/instance/lifecycle.js中的initLifecycle函数中定义的。initLifecycle会在vue实例或者组件初始化的时候执行。

queueActivatedComponent函数将vnode对应的实例vm给push到activatedChildren数组中。然后再flushSchedulerQueue函数中，即下一个tick中遍历activatedChildren数组中的所有vm实例，对vm实例执行activateChildComponent函数，即执行了activated钩子函数。

```JavaScript
export function activateChildComponent (vm: Component, direct?: boolean) {
  if (direct) {
    vm._directInactive = false
    if (isInInactiveTree(vm)) {
      return
    }
  } else if (vm._directInactive) {
    return
  }
  if (vm._inactive || vm._inactive === null) {
    vm._inactive = false
    for (let i = 0; i < vm.$children.length; i++) {
      activateChildComponent(vm.$children[i])
    }
    callHook(vm, 'activated')
  }
}

export function queueActivatedComponent (vm: Component) {
  // setting _inactive to false here so that a render function can
  // rely on checking whether it's in an inactive tree (e.g. router-view)
  vm._inactive = false
  activatedChildren.push(vm)
}
```

总结一下， _isMounted为true，说明是组件已经挂载，此patch过程是组件的更新；当为false的时候，说明此patch过程是组件的初始渲染过程。当更新的时候，是在下一个tick去执行的组件的activated钩子函数。



在patch的函数中，当一个组件更新的过程中，如果更新前的vnode和更新后的vnode不是sameVnode的，不能复用，那么需要对oldVnode进行销毁，即调用invokeDestroyHook函数。该函数中会去执行组件的destory钩子函数。

```javascript
// destroy old node
if (isDef(parentElm)) {
	removeVnodes([oldVnode], 0, 0)
} else if (isDef(oldVnode.tag)) {
	invokeDestroyHook(oldVnode)
}

function invokeDestroyHook (vnode) {
    let i, j
    const data = vnode.data
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
      for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode)
    }
    if (isDef(i = vnode.children)) {
      for (j = 0; j < vnode.children.length; ++j) {
        invokeDestroyHook(vnode.children[j])
      }
    }
}
```

destroy钩子中，针对被keep-alive包裹的组件，执行deactivateChildComponent方法。deactivateChildComponent函数的核心是执行vnode的deactivated钩子函数，并且保证在patch过程中之执行一次deactivated。为什么要保证呢，因为在父组件调用deactivated钩子函数的时候，会遍历子组件，调用子组件的deactivated钩子函数，所以当子组件在patch的时候，已经不需要再调用deactivated钩子函数了。

```JavaScript
destroy (vnode: MountedComponentVNode) {
    const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
      //对于普通的组件，会调用组件实例的$destroy方法
      if (!vnode.data.keepAlive) {
        componentInstance.$destroy()
      //对于被keep-alive包裹的组件，执行deactivateChildComponent方法
      } else {
        deactivateChildComponent(componentInstance, true /* direct */)
      }
    }
}

export function deactivateChildComponent (vm: Component, direct?: boolean) {
  if (direct) {
    vm._directInactive = true
    if (isInInactiveTree(vm)) {
      return
    }
  }
  if (!vm._inactive) {
    vm._inactive = true
    //遍历子组件，执行所有子组件的deactivated钩子函数
    for (let i = 0; i < vm.$children.length; i++) {
      deactivateChildComponent(vm.$children[i])
    }
    callHook(vm, 'deactivated')
  }
}
```



总结：经过keep-alive包裹的组件在更新渲染的时候，不会经过组件的实例化，render生成vnode和patch到dom等流程。而是从缓存的vnode的实例中拿到dom直接插入到页面上。

keep-alive组件的渲染分为首次渲染和缓存渲染，当命中缓存，则不会执行created和mounted钩子函数，而是执行activated钩子函数。销毁时也不会执行destroyed钩子函数，而是执行deactivated钩子函数。