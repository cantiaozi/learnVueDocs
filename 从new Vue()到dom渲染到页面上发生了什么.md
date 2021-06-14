## 从new Vue()到dom渲染到页面上发生了什么

```JavaScript
new Vue({
  render(h) {
    return h('div', {
      attrs: {
        id: 'app1'
      }
    }, 
      [
        this.msg,
        [h('div', {attrs: {id: 'app2'}}, [h('div', {attrs: {id: 'app3'}}, 'gg')]), 'tt']
      ]
    )
  },
  el: '#app',
  data: {
    msg: 'hello world'
  }
})
```

上面的代码最总会在页面上渲染成下面的内容。

![](E:\开发文档\新建文件夹\learnVueDocs\others\页面渲染.png)

那么这其中发生了什么呢？vue做的事情主要如下。

### 1.1 Vue构造方法

Vue函数定义在src\core\instance目录下的index.js文件中。

```JavaScript
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

_init方法src\core\instance目录下的init.js文件中。挂载在Vue的原型对象上。

```javascript
Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    
    vm._uid = uid++

    let startTag, endTag
    
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    vm._isVue = true
    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }

    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }

    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) 
    initState(vm)
    initProvide(vm) 
    callHook(vm, 'created')

    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
}
```

_init方法中进行了一些列初始化操作，将vue实例的配置放到实例的$options属性上。这里我们重点分析3个地方，initProxy(vm)、initState(vm)和$mount页面挂载方法。

### 1.2 initProxy(vm)

当我们在vue的template模板中使用了一个未定义的变量时，会报错提示。这是因为做了Proxy代理，当访问vue实例上的相应属性时，会通过这层代理。

![](E:\开发文档\新建文件夹\learnVueDocs\others\image-20210607191609302.png)

initProxy是这样定义的。

```JavaScript
function initProxy (vm) {
    if (hasProxy) {
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
    } else {
      vm._renderProxy = vm
    }
}
```

其中hasProxy判断浏览器是否支持原生的Proxy对象。hasHander的定义如下。在hasHander对象中拦截了`propKey in proxy`的操作。方法中定义了一个Proxy对象，并挂载vue实例的_renderProxy属性上。warnNonPresent方法就是报错提示的方法。

```JavaScript
const hasHandler = {
    has (target, key) {
      const has = key in target
      const isAllowed = allowedGlobals(key) || (typeof key === 'string' && key.charAt(0) === '_')
      if (!has && !isAllowed) {
        warnNonPresent(target, key)
      }
      return has || !isAllowed
    }
}
```

```JavaScript
const warnNonPresent = (target, key) => {
    warn(
      `Property or method "${key}" is not defined on the instance but ` +
      'referenced during render. Make sure that this property is reactive, ' +
      'either in the data option, or for class-based components, by ' +
      'initializing the property. ' +
      'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
      target
    )
}
```

### 1.3 initState(vm)

我们在vue的某些生命周期中可以通过this.xxx的方式来访问data、props和methods中的属性。这是通过代理来实现的。initState函数的代码如下：

```JavaScript
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

initState函数中主要是对props、methods、data、computed和watch属性进行初始化。initData方法的代码如下：

```JavaScript
function initData (vm: Component) {
  let data = vm.$options.data
  //data属性通常是一个函数，当是函数时，执行getData方法。
  //getData方法主要时执行了data函数，并将其函数内部的this绑定为vue实例。方法返回一个对象
  //之所以设计为函数，是因为组件可以复用，这样可以为每个组件实例生成一个data对象
  //避免修改一个组建的data属性，影响到另一个组件。
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
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  //while循环中判断data中的属性是否会和props和methods中的属性重复
  //因为props、data和methods中的属性都可以通过组件实例访问，需要避免重名。
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
      //data属性放到了组件实例的_data属性上。  
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

proxy方法中通过Object.defineProperty可以在组件实例上访问data中的属性。proxy方法如下：

```JavaScript
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

proxy传入的第二个参数是'_data'，当访问或者设置组件实例上的属性时，会去访问或者设置组件实例__data属性上的相应属性。

### 1.4 $mount方法

```JavaScript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  //query方法就是返回el代表的dom对象。
  //如果传入的是string，通过document.querySelector找到页面上的dom对象并放回该对象
  //如果是Element，就直接返回
  el = el && query(el)

  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // vue中template或者el都会最终转换为render函数。如果render不存在，需要将
  // template或者el转换为render
  if (!options.render) {
    let template = options.template
    if (template) {
      //当vue实例中有template属性，且是字符串
      if (typeof template === 'string') {
        //字符串以#开头，代表id,template中也可以el属性那样写id
        if (template.charAt(0) === '#') {
          //idToTemplate方法实际上是调用query方法，获取dom对象，并且该方法中做了缓存。
          template = idToTemplate(template)
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      //template.nodeType存在，说明template是一个虚拟dom对象
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    //如果template不存在，el存在，就拿到el这个dom对象的outerHTML属性
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }
	  //将template转换为render函数，最终vue实例挂载是依赖render函数
      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}
```

$mount方法的最后调用mount方法。mount方法如下：

```JavaScript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

mount方法中调用mountComponent方法并返回结果。mountComponent方法如下：

```JavaScript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      //当实例中template或者el存在时，说明没有编译为render函数
      //提示当前版本为runtime-only版本
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  //性能埋点相关，可以忽略，走else里的逻辑，定义updateComponent函数
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }

  //这里定义了一个watcher对象，watcher分为两类：一是vue实例中的自定义watch
  //二是render watcher即渲染watcher，这里就是渲染watcher。
  //noop是一个空函数，true代表这是一个渲染watcher
  //第四个参数时watcher对象的配置
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

Watcher的构造函数中与页面首次加载渲染相关的逻辑主要如下，首次渲染的时候就是执行updateComponent方法，该方法就是渲染整个页面。

```JavaScript
export default class Watcher {
	constructor(
    	vm: Component,
        expOrFn: string | Function,
        cb: Function,
        options?: ?Object,
        isRenderWatcher?: boolean
    ) {
		//如果是渲染watcher，将watcher实例挂到vue实例的_watcher属性上
		if (isRenderWatcher) {
          vm._watcher = this
        }
        this.getter = expOrFn
        //执行expOrFn函数
        this.value = this.get()
	}
	
	get () {
        let value
        const vm = this.vm
        value = this.getter.call(vm, vm)
        return value
  	}

}
```

### 1.5 _render方法

上面的updateComponent方法中调用了vue实例上的_render方法。 _render方法是在src\core\instance目录下的index.js文件中的renderMixin中定义的（详情请看1.1）。

```JavaScript
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    .....
	//执行vm.$options中的render函数，传入两个参数
    //第一个参数vm._renderProxy是一个Proxy对象，详情查看1.2
    //vm.$createElement方法定义在initRender函数中
    //initRender函数在_init方法中被执行了
    let vnode
    try {
      //调用render函数,把render函数中的this绑定为Proxy对象。
      //当对不存在的属性进行in操作符时，或者在template模板中使用不存在的属性都会报错
      //执行render函数会生成vnode虚拟dom
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      ...
    }
    // 判断当vnode是一个数组时，进行报警。根节点只能有一个dom
    if (!(vnode instanceof VNode)) {
      if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
        warn(
          'Multiple root nodes returned from render function. Render function ' +
          'should return a single root node.',
          vm
        )
      }
      vnode = createEmptyVNode()
    }
    vnode.parent = _parentVnode
    return vnode
  }
```

render函数在执行时，传入了$createElement方法，我们在vue中使用render一般如下所示，render函数中的h就是$createElement的形参。

```JavaScript
render(h) {
    return h('div', {
      attrs: {
        id: 'app1'
      }
    }, this.msg)
},
```





```JavaScript
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```

$createElement方法中执行createElement方法。createElement方法返回的是vnode，即虚拟dom。

```JavaScript
const SIMPLE_NORMALIZE = 1
const ALWAYS_NORMALIZE = 2
//normalizationType一般是布尔值，当为true时，表示手写的render函数
//为false时，表示编译生成的render函数
export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  //当使用render函数时，data属性是可以不传的，这时候需要把后面的参数向前面移动一位
  //当data是数组或者基本类型的数据时，判断成data没传
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}

export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  ...
  //这里走的是if里面的逻辑，将children转换为vnode数组  
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```



normalizeChildren方法如下，当children为基本数据类型时，创建一个文本节点并返回。当children是一个数组时，调用normalizeArrayChildren方法。当是其它类型时，就直接返回undefined了，例如下面这种情况，id为'app1'的div的子节点是一个vnode，结果normalizeChildren直接返回了undefined，体现在页面上就是页面为空，不会展示'gg'字段。解决手段就是将其包装成一个数组。

```JavaScript
new Vue({
  render(h) {
    return h('div', {
      attrs: {
        id: 'app1'
      }
    }, 
      h('div', {attrs: {id: 'app3'}}, 'gg')
    )
  },
  el: '#app',
  data: {
    msg: 'hello world'
  }
})
```



```javascript 
export function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}

function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
  const res = []
  let i, c, lastIndex, last
  for (i = 0; i < children.length; i++) {
    c = children[i]
    if (isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    //  nested
    if (Array.isArray(c)) {
      if (c.length > 0) {
        c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
        // merge adjacent text nodes
        if (isTextNode(c[0]) && isTextNode(last)) {
          res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
          c.shift()
        }
        res.push.apply(res, c)
      }
    } else if (isPrimitive(c)) {
      if (isTextNode(last)) {
        // merge adjacent text nodes
        // this is necessary for SSR hydration because text nodes are
        // essentially merged when rendered to HTML strings
        res[lastIndex] = createTextVNode(last.text + c)
      } else if (c !== '') {
        // convert primitive to vnode
        res.push(createTextVNode(c))
      }
    } else {
      if (isTextNode(c) && isTextNode(last)) {
        // merge adjacent text nodes
        res[lastIndex] = createTextVNode(last.text + c.text)
      } else {
        // default key for nested array children (likely generated by v-for)
        if (isTrue(children._isVList) &&
          isDef(c.tag) &&
          isUndef(c.key) &&
          isDef(nestedIndex)) {
          c.key = `__vlist${nestedIndex}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```

normalizeArrayChildren方法中对children数组进行for循环处理，结果返回一个数组res，res数组中的每个值都是一个vnode对象。当children[i]是基础数据类型时，生成一个文本类型的vnode，并且push到结果数组res中，如果连续两个vnode对象都是文本节点，则合并成一个文本类型的vnode；当children[i]是一个数组时，递归调用normalizeArrayChildren方法生成一个vnode节点数组c，并将该数组c合并到res数组中，如果c[0]和res的最后一项都是文本vnode，则合并成一个vnode;其它情况直接对应的children[i]就是一个vnode，直接push到res数组中。最终生成的数组是一个一维数组，该方法有将数组扁平化的功能。

回到_createElement方法，再将children转换为vnode数组后，剩下的逻辑主要是判断tag，如果tag是一个表示普通dom节点的字符串，直接调用VNode构造函数生成vnode对象；如果tag是一个表示vue组件的字符串，则调用createComponent方法生成vnode对象。最后返回vnode对象。createComponent方法不在本文讨论范围内。

### 1.6 _update

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
  }
```

_update方法的调用时间有两点：一是初始化页面渲染的时候会调用，二是响应式数据改变，页面更新的时候会调用。 _update方法中的prevVnode表示老的虚拟dom，在响应式数据改变、页面更新的时候才有，因此!prevVnode是true。 --patch--方法定义在runtime/index.js文件中，实际上就是对patch方法的引用。

```javascript
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

patch方法是调用createPatchFunction返回的函数，nodeOps是一个对象，封装了对于真实dom元素的各种操作，modules中封装了对于dom元素属性、类、样式属性、事件等的操作方法。createPatchFunction函数最后返回了一个名为patch的函数。patch函数执行时，传递的四个参数中，oldVnode是真实的dom对象，vnode是虚拟dom对象，hydrating和removeOnly都是false。其中会执行createElm方法将vnode转换为真实dom并插入到页面上，然后删除原来的dom节点。

```javascript
return function patch (oldVnode, vnode, hydrating, removeOnly) {

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      ......
    } else {
      //oldVnode是真实dom对象，oldVnode.nodeType存在
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
      } else {
        if (isRealElement) {
          ......
          //emptyNodeAt方法将真实的dom转换为vnode虚拟dom
          oldVnode = emptyNodeAt(oldVnode)
        }

        // 虚拟dom的elm属性是其表示的真实的dom对象
        const oldElm = oldVnode.elm
        //找到真实dom对象的父节点
        const parentElm = nodeOps.parentNode(oldElm)

        // 执行createElm方法将vnode转换为真实dom并插入到页面上。
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // 删除原来的dom节点，即id为app的节点
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

createElm方法如下，在函数执行过程中，首先会生成真实的dom元素，此时不会立即插入到页面上。然后会调用createChildren方法将子节点转换为真实dom并插入到父节点上，createChildren方法的逻辑比较简单，判断子节点的类型，如果是一个数组，就递归调用createElm，将子节点转换为真实dom并且插入到父节点中，如果是一个基础类型数据，则直接生成一个文本节点插入到父节点中。最后在调用insert方法将父节点插入到页面上。因此节点插入的顺序是先子后父。

```javascript
function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      //生成真实的dom对象，并且赋给vnode的elm属性
      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      
      if (__WEEX__) {
        ......
      } else {
        //将子节点插入到vnode节点的真实dom节点上
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        //将父节点的真实dom插入到页面上
        insert(parentElm, vnode.elm, refElm)
      }

      if (process.env.NODE_ENV !== 'production' && data && data.pre) {
        creatingElmInVPre--
      }
    } else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
}
        
function createChildren (vnode, children, insertedVnodeQueue) {
    if (Array.isArray(children)) {
      ......
      for (let i = 0; i < children.length; ++i) {
        createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
      }
    } else if (isPrimitive(vnode.text)) {
      nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
    }
  }
```

### 1.7 总结

用一张流程图总结

![](E:\开发文档\新建文件夹\learnVueDocs\others\newVue到页面渲染的主要流程图.png)

## 一、可复用的方法

1、判断浏览器是否支持某个原生类（函数）

```JavaScript
function isNative (Ctor) {
  return typeof Ctor === 'function' && /native code/.test(Ctor.toString())
}
```

2、数组扁平化，二维数组转换为一维数组

```javascript
var arr = [[1,2], 3]
var resArray = Array.prototype.concat.apply([], arr)// [1, 2, 3]
```

参考

```javascript
var resArray1 = [].apply([], [1,2], 3)//[1, 2, 3]
```

