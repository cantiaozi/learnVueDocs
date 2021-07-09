



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

initProps方法中，调用validateProp方法实现校验和求值后，再调用

defineReactive方法实现prop的响应式，并将