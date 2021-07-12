
## 一、规范化

在初始化props之前，会对props进行一次规范化。在组件的初始化过程中，会调用mergeOptinons方法合并配置。在该方法中，调用了normalizeProps将props进行了规范化。normalizeProps传入了两个参数，组件上自定义的options配置和vm实例。因为在组件中定义props的时候，vue支持数组和对象的形式。规范化就是将props转化为对象的形式。

```JavaScript
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
//parent是父的option配置对象
//child是子的option配置对象
//在vue实例或者组件实例化过程中，调用该方法，child是
//组件上自定义的options配置
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  ......

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)
  ......
  
  return options
}


/**
 * Ensure all props option syntax are normalized into the
 * Object-based format.
 */
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  //最总返回的props对象
  const res = {}
  let i, val, name
  //将props数组转化为对象
  if (Array.isArray(props)) {
    i = props.length
    //拿到props数组中的每个值做处理
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        //将prop的key值驼峰化
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      //当props是对象时，每个prop的key后面可以是一个普通对象
      //也可以是一个js内置的对象，如果是一个内置对象的话，表示的是prop的类型
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```

## 二、初始化

props的初始化过程发生在initState方法中。调用initProps函数进行初始化。传入的参数为规范化后的props对象和vm实例。

```JavaScript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  ......
}
```

初始化可以分为3个阶段：1.对prop进行校验和求值；2.对prop实现响应式；3.对prop进行代理。

```JavaScript
function initProps (vm: Component, propsOptions: Object) {
  //propsData是组件vm的父组件传给vm的prop数据
  const propsData = vm.$options.propsData || {}
  //将初始化计算后的prop保存在vm的_props属性上
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  //保存一份prop的key值
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (vm.$parent && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

### 2.1校验和求值

在initProps方法中，遍历拿到propsOptions中所有prop的key，调用validateProp进行对prop进行校验并赋值。

validateProp方法定义在core/util/props.js中，该方法接受4个参数，vm实例，规范化的props对象、父组件的传来的prop数据。在validateProp方法中，如果发现父组件没有给某个prop传值，那么就会调用getPropDefaultValue获取默认值。然后，调用assertProp方法对prop进行校验。

```javascript
export function validateProp (
  key: string,
  propOptions: Object,
  propsData: Object,
  vm?: Component
): any {
  const prop = propOptions[key]
  const absent = !hasOwn(propsData, key)
  let value = propsData[key]
  // boolean casting
  const booleanIndex = getTypeIndex(Boolean, prop.type)
  if (booleanIndex > -1) {
    //如果prop的类型中有Boolean类型，且prop没有默认值，且
    //父组件没有在组件上定义prop，那么给该prop赋值为false
    if (absent && !hasOwn(prop, 'default')) {
      value = false
        
    //如果prop的类型中有Boolean类型，且父组件有在组件上
    //定义prop但没有传值，且prop的类型中不包含String或者
    //String在Boolean后面，那么给该prop赋值为true
    } else if (value === '' || value === hyphenate(key)) {
      // only cast empty string / same name to boolean if
      // boolean has higher priority
      const stringIndex = getTypeIndex(String, prop.type)
      if (stringIndex < 0 || booleanIndex < stringIndex) {
        value = true
      }
    }
  }
  // check default value
  if (value === undefined) {
    value = getPropDefaultValue(vm, prop, key)
    // since the default value is a fresh copy,
    // make sure to observe it.
    const prevShouldObserve = shouldObserve
    toggleObserving(true)
    observe(value)
    toggleObserving(prevShouldObserve)
  }
  if (
    process.env.NODE_ENV !== 'production' &&
    // skip validation for weex recycle-list child component props
    !(__WEEX__ && isObject(value) && ('@binding' in value))
  ) {
    assertProp(prop, key, value, vm, absent)
  }
  return value
}

//根据传入的type，在expectedTypes中找到相同输入类型的索引
//例如传入的type是Number，expectedTypes是[String,Object,Number],那么返回索引2
function getTypeIndex (type, expectedTypes): number {
  if (!Array.isArray(expectedTypes)) {
    return isSameType(expectedTypes, type) ? 0 : -1
  }
  for (let i = 0, len = expectedTypes.length; i < len; i++) {
    if (isSameType(expectedTypes[i], type)) {
      return i
    }
  }
  return -1
}

function isSameType (a, b) {
  return getType(a) === getType(b)
}

/**
 * Use function string name to check built-in types,
 * because a simple equality check will fail when running
 * across different vms / iframes.
 */
//通过将类似于String、Number之类的内置对象转变成字符串的
//形式去比较，不直接用===去比因为在不同的iframes中会失败
function getType (fn) {
  const match = fn && fn.toString().match(/^\s*function (\w+)/)
  return match ? match[1] : ''
}


/**
 * Get the default value of a prop.
 */
function getPropDefaultValue (vm: ?Component, prop: PropOptions, key: string): any {
  // no default, return undefined
  if (!hasOwn(prop, 'default')) {
    return undefined
  }
  const def = prop.default
  // warn against non-factory defaults for Object & Array
  //当默认值是一个对象的时候，必须用工厂函数返回默认值
  if (process.env.NODE_ENV !== 'production' && isObject(def)) {
    warn(
      'Invalid default value for prop "' + key + '": ' +
      'Props with type Object/Array must use a factory function ' +
      'to return the default value.',
      vm
    )
  }
  // the raw prop value was also undefined from previous render,
  // return previous default value to avoid unnecessary watcher trigger
  //当重新渲染时，如果没有这个逻辑，会再次调用工厂函数返回   
  //默认值，因为每次工厂函数返回的默认值是不同的对象，
  //触发setter再次造成重新渲染。加了这个逻辑之后
  //就直接使用上一次的值，避免重新渲染
  if (vm && vm.$options.propsData &&
    vm.$options.propsData[key] === undefined &&
    vm._props[key] !== undefined
  ) {
    return vm._props[key]
  }
  // call factory function for non-Function types
  // a value is Function if its prototype is function even across different execution context
  return typeof def === 'function' && getType(prop.type) !== 'Function'
    ? def.call(vm)
    : def
}
```

assertProp方法中，首先会将prop定义中的type数组进行循环，拿到每个类型，调用assertType方法，传入prop的值和type类型（例如Number、String这样的构造函数），该方法判断prop的值是否匹配类型。然后会调用自定义的校验器对prop的值进行校验。

```JavaScript
/**
 * Assert whether a prop is valid.
 */
function assertProp (
  prop: PropOptions,
  name: string,
  value: any,
  vm: ?Component,
  absent: boolean
) {
  if (prop.required && absent) {
    warn(
      'Missing required prop: "' + name + '"',
      vm
    )
    return
  }
  if (value == null && !prop.required) {
    return
  }
  let type = prop.type
  let valid = !type || type === true
  const expectedTypes = []
  if (type) {
    if (!Array.isArray(type)) {
      type = [type]
    }
    for (let i = 0; i < type.length && !valid; i++) {
      const assertedType = assertType(value, type[i])
      expectedTypes.push(assertedType.expectedType || '')
      valid = assertedType.valid
    }
  }
  if (!valid) {
    warn(
      `Invalid prop: type check failed for prop "${name}".` +
      ` Expected ${expectedTypes.map(capitalize).join(', ')}` +
      `, got ${toRawType(value)}.`,
      vm
    )
    return
  }
  const validator = prop.validator
  if (validator) {
    if (!validator(value)) {
      warn(
        'Invalid prop: custom validator check failed for prop "' + name + '".',
        vm
      )
    }
  }
}

const simpleCheckRE = /^(String|Number|Boolean|Function|Symbol)$/

function assertType (value: any, type: Function): {
  valid: boolean;
  expectedType: string;
} {
  let valid
  //拿到类型构造函数的字符串表示，
  //例如Number构造函数返回Number字符串
  const expectedType = getType(type)
  if (simpleCheckRE.test(expectedType)) {
    const t = typeof value
    valid = t === expectedType.toLowerCase()
    // for primitive wrapper objects
    if (!valid && t === 'object') {
      //当prop的值是调用构造函数生成的对象，
      //只要下面的判断为true，也是有效的
      //例如，value值是new String('ddd')就符合这种情况
      valid = value instanceof type
    }
  } else if (expectedType === 'Object') {
    valid = isPlainObject(value)
  } else if (expectedType === 'Array') {
    valid = Array.isArray(value)
  } else {
    valid = value instanceof type
  }
  return {
    valid,
    expectedType
  }
}
```

### 2.2实现响应式

initProps方法中，调用validateProp方法实现校验和求值后，再调用defineReactive方法实现prop的响应式，并将prop的值保存在vm的_props属性上。当访问vm. _props 上的相应key值的prop时，就会触发依赖收集。调用defineReactive方法传入了3个参数，vm. _props、 prop的key和prop的值。defineReactive方法内部判断当prop的值是一个对象的时候，会调用observe方法将这个对象也变成响应式的。因此，当prop值是一个对象，我们在父组件中改变这个对象的某个属性时，也会触发set属性描述符，导致组件渲染。

收集依赖：当我们在初次渲染组件时，render方法生成渲染vnode，该过程会访问vm. _props属性上的prop值。触发get描述符通知相关dep对象收集该组件的渲染watcher。

看下面的例子。在初次渲染时，CompA组件访问了vm. _prop 上的hobby属性，相关dep对象（id为8）收集到CompA组件的渲染watcher。当我们在点击change按钮的时候，会触发app重新渲染，在render生成渲染vnode的过程中，会访问到data中的hobby等属性，因此hobby的dep（此hobby是data中的hobby，和CompA组件实例 _prop属性上的hobby不一样）对象会收集app的渲染watcher。app在patch时，在进行patchVnode的过程中，遇到了CompA的占位符vnode，于是对vnode执行prepatch，然后执行updateChildComponent方法。在该方法中，因为对CompA实例vm的 _prop上的hobby的值进行了修改，所以会触发hobby的dep对象相关的渲染watcher（即CompA的渲染watcher）重新渲染。

```JavaScript
import Vue from 'vue'
let CompA = {
    template: `
        <div>
            <div>{{name}}</div>
            <div>{{hobby}}</div>
        </div>
    `,
    updated() {
        console.log("compa updated")
    },
    props: ['name', 'hobby']
}

let app = new Vue({
    el: '#app',
    components: {CompA},
    template: `
        <div>
            <div>{{title}}</div>
            <CompA :name="name" :hobby="hobby"/>
            <button @click="change">change</button>
            <button @click="toggle">toggle</button>
            <button @click="changeTitle">changeTitle</button>
        </div>
    `,
    data: {
        name: 'liuyong',
        hobby: 'diaoyu',
        title: 'des'
    },
    methods: {
        change() {
            this.hobby = {
                first: 'diaoyu',
                second: 'run'
            }
        },
        toggle() {
            this.hobby.first = 'diaoxia'
        },
        changeTitle() {
            this.title = 'descrition'
        }
    }
})
```

再看下面这个例子：当CompA在首次渲染时，props在初始化时，将vm. _props上的hobby属性设置为响应式的时候，因为hobby的值是一个对象，所以会调用observe方法将其也设置为响应式的，但是其实这个hobby的值是app传过来的，所以已经是响应式的了。后面CompA将hobby渲染在页面上，render生成渲染vnode的时候调用了JSON.stringify方法将hobby转化为字符串，因此造成了vm. _props中hobby以及hobby中first和second属性的访问，触发了相关的dep（ _props中hobby属性的dep的id为9，first属性的dep的id 为5）对象收集CompA的渲染watcher。

```JavaScript
import Vue from 'vue'
let CompA = {
    template: `
        <div>
            <div>{{hobby}}</div>
        </div>
    `,
    updated() {
        console.log("compa updated")
    },
    props: ['hobby']
}

let app = new Vue({
    el: '#app',
    components: {CompA},
    template: `
        <div>
            
            <CompA :hobby="hobby"/>
            <button @click="toggle">toggle</button>
        </div>
    `,
    data: {
        hobby: {
            first: 'diaoyu',
            second: 'run'
        },
    },
    updated() {
        console.log("app updated")
    },
    methods: {
        toggle() {
            this.hobby.first = 'diaoxia'
        },
    }

})
```



### 2.3 代理

调用proxy方法，将对vm实例上prop属性的访问代理到vm._prop上的相关key的prop上。如果是new Vue生成vue实例，那么就是在此时进行的prop代理。

但是如果是组件，prop代理发生的时刻并不在此时。而是在组件生成构造函数调用extend方法时，已经对prop进行了代理。这样就不需要在每个组件实例上给prop做代理，是一个优化。

```javascript
Vue.extend = function (extendOptions: Object): Function {
    ......
    const Sub = function VueComponent (options) {
      this._init(options)
    }
    ......
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    ......
    return Sub
}

function initProps (Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}
```



## 三、props的更新

 当父组件传递给子组件的 `props` 值变化，子组件对应的值也会改变，同时会触发子组件的重新渲染。 `prop` 数据的值变化在父组件，我们知道在父组件的 `render` 过程中会访问到这个 `prop` 数据，所以当 `prop` 数据变化一定会触发父组件的重新渲染，那么重新渲染是如何更新子组件对应的 `prop` 的值呢？ 

###  3.1子组件props的更新

在父组件重新渲染的最后，会执行 `patch` 过程，进而执行 `patchVnode` 函数，`patchVnode` 通常是一个递归过程，当它遇到组件 `vnode` （即占位符vnode）的时候，会执行组件更新过程的 `prepatch` 钩子函数，在 `src/core/vdom/patch.js` 中： 

```js
function patchVnode (
  oldVnode,
  vnode,
  insertedVnodeQueue,
  ownerArray,
  index,
  removeOnly
) {
  // ...

  let i
  const data = vnode.data
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode)
  }
  // ...
}
```

`prepatch` 函数定义在 `src/core/vdom/create-component.js` 中：

```js
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
```

内部会调用 `updateChildComponent` 方法来更新 `props`，注意第二个参数就是父组件的 `propData`，那么为什么 `vnode.componentOptions.propsData` 就是父组件传递给子组件的 `prop` 数据呢（这个也同样解释了第一次渲染的 `propsData` 来源）？原来在组件的 `render` 过程中，对于组件节点会通过 `createComponent` 方法来创建组件 `vnode`：

```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // ...

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // ...
  
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // ...
  
  return vnode
}
```

 在创建组件 `vnode` 的过程中，首先从 `data` 中提取出 `propData`，然后在 `new VNode` 的时候，作为第七个参数 `VNodeComponentOptions` 中的一个属性传入，所以我们可以通过 `vnode.componentOptions.propsData` 拿到 `prop` 数据。 

接着看 `updateChildComponent` 函数，它的定义在 `src/core/instance/lifecycle.js` 中：

```js
export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  // ...

  // update props
  if (propsData && vm.$options.props) {
    toggleObserving(false)
    const props = vm._props
    const propKeys = vm.$options._propKeys || []
    for (let i = 0; i < propKeys.length; i++) {
      const key = propKeys[i]
      const propOptions: any = vm.$options.props // wtf flow?
      props[key] = validateProp(key, propOptions, propsData, vm)
    }
    toggleObserving(true)
    // keep a copy of raw propsData
    vm.$options.propsData = propsData
  }

  // ...
}
```

我们重点来看更新 `props` 的相关逻辑，这里的 `propsData` 是父组件传递的 `props` 数据，`vm` 是子组件的实例。`vm._props` 指向的就是子组件的 `props` 值，`propKeys` 就是在之前 `initProps` 过程中，缓存的子组件中定义的所有 `prop` 的 `key`。主要逻辑就是遍历 `propKeys`，然后执行 `props[key] = validateProp(key, propOptions, propsData, vm)` 重新验证和计算新的 `prop` 数据，更新 `vm._props`，也就是子组件的 `props`，这个就是子组件 `props` 的更新过程。

### 3.2子组件的重新渲染

子组件重新渲染的两种情况：1.父组件中修改了子组件的prop值，触发prop的setter，进而使子组件重新渲染；2.子组件的某个prop值是一个对象，父组件中没有修改prop的值，但是在父组件或者子组件中修改了该prop对象的某个属性，触发了子组件的渲染。（详情看2.2节）

