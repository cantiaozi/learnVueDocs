本文以下面的例子来解析vue中的事件机制。

```JavaScript
let child = {
    template: '<button @click="clickHandler($event)">' + 'click me' + '</button>',
    methods: {
        clickHandler(e) {
            console.log("button clicked", e)
            this.$emit("select")
        }
    }
}
new Vue({
    el: '#app',
    components: {
        child
    },
    template: '<div>' + '<child @select="selectHandler" @click.native.prevent="clickHandler">'
        + '</child></div>',
    methods: {
        clickHandler() {
            console.log("child clicked")
        },
        selectHandler() {
            console.log("child selected")
        }
    }
})
```

## 一、parse阶段

在编译生成ast节点的过程中，template字符串匹配到开始标签，调用start方法生成ast对象的过程中，会先创建ast对象，然后管理ast对象，最后管理树结构。在管理ast对象的过程中，会调用processElement方法处理标签上的属性。processElement方法中会调用processAttrs方法处理某些属性。

```JavaScript
function processAttrs (el) {
  const list = el.attrsList
  let i, l, name, rawName, value, modifiers, isProp
  //遍历ast对象上定义的所有属性
  for (i = 0, l = list.length; i < l; i++) {
    name = rawName = list[i].name
    value = list[i].value
    //判断是否是vue自定义属性，以v-或者：@开头
    if (dirRE.test(name)) {
      // mark element as dynamic
      el.hasBindings = true
      // modifiers
      //parseModifiers解析属性中的修饰符
      modifiers = parseModifiers(name)
      if (modifiers) {
        //如果事件上有修饰符的话，就去掉修饰符
        name = name.replace(modifierRE, '')
      }
      //v-bind或者：属性的检测
      if (bindRE.test(name)) { // v-bind
        name = name.replace(bindRE, '')
        value = parseFilters(value)
        isProp = false
        if (modifiers) {
          if (modifiers.prop) {
            isProp = true
            name = camelize(name)
            if (name === 'innerHtml') name = 'innerHTML'
          }
          if (modifiers.camel) {
            name = camelize(name)
          }
          if (modifiers.sync) {
            addHandler(
              el,
              `update:${camelize(name)}`,
              genAssignmentCode(value, `$event`)
            )
          }
        }
        if (isProp || (
          !el.component && platformMustUseProp(el.tag, el.attrsMap.type, name)
        )) {
          addProp(el, name, value)
        } else {
          addAttr(el, name, value)
        }
      //v-on或者@属性的检测
      } else if (onRE.test(name)) { // v-on
        //去掉v-bind或者@
        name = name.replace(onRE, '')
        addHandler(el, name, value, modifiers, false, warn)
      } else { // normal directives
        name = name.replace(dirRE, '')
        // parse arg
        const argMatch = name.match(argRE)
        const arg = argMatch && argMatch[1]
        if (arg) {
          name = name.slice(0, -(arg.length + 1))
        }
        addDirective(el, name, rawName, value, arg, modifiers)
        if (process.env.NODE_ENV !== 'production' && name === 'model') {
          checkForAliasModel(el, value)
        }
      }
    } else {
      // literal attribute
      if (process.env.NODE_ENV !== 'production') {
        const res = parseText(value, delimiters)
        if (res) {
          warn(
            `${name}="${value}": ` +
            'Interpolation inside attributes has been removed. ' +
            'Use v-bind or the colon shorthand instead. For example, ' +
            'instead of <div id="{{ val }}">, use <div :id="val">.'
          )
        }
      }
      addAttr(el, name, JSON.stringify(value))
      // #6887 firefox doesn't update muted state if set via attribute
      // even immediately after element creation
      if (!el.component &&
          name === 'muted' &&
          platformMustUseProp(el.tag, el.attrsMap.type, name)) {
        addProp(el, name, 'true')
      }
    }
  }
}

function parseModifiers (name: string): Object | void {
  const match = name.match(modifierRE)
  if (match) {
    const ret = {}
    //slice方法去掉匹配到的修饰符前的点
    match.forEach(m => { ret[m.slice(1)] = true })
    return ret
  }
}

const modifierRE = /\.[^.]+/g
export const dirRE = /^v-|^@|^:/
```

processAttrs方法中，检测某个属性如果是v-on或者@开头的事件监听，那么就会调用addHandler方法。

```JavaScript
export function addHandler (
  el: ASTElement,
  name: string,
  value: string,
  modifiers: ?ASTModifiers,
  important?: boolean,
  warn?: Function
) {
  modifiers = modifiers || emptyObject
  // warn prevent and passive modifier
  /* istanbul ignore if */
  if (
    process.env.NODE_ENV !== 'production' && warn &&
    modifiers.prevent && modifiers.passive
  ) {
    warn(
      'passive and prevent can\'t be used together. ' +
      'Passive handler can\'t prevent default event.'
    )
  }

  // check capture modifier
  if (modifiers.capture) {
    delete modifiers.capture
    name = '!' + name // mark the event as captured
  }
  if (modifiers.once) {
    delete modifiers.once
    name = '~' + name // mark the event as once
  }
  /* istanbul ignore if */
  if (modifiers.passive) {
    delete modifiers.passive
    name = '&' + name // mark the event as passive
  }

  // normalize click.right and click.middle since they don't actually fire
  // this is technically browser-specific, but at least for now browsers are
  // the only target envs that have right/middle clicks.
  if (name === 'click') {
    //监听鼠标右键的事件
    if (modifiers.right) {
      name = 'contextmenu'
      delete modifiers.right
    //监听鼠标中键的事件
    } else if (modifiers.middle) {
      name = 'mouseup'
    }
  }

  let events
  //事件上是否有native修饰符
  if (modifiers.native) {
    delete modifiers.native
    events = el.nativeEvents || (el.nativeEvents = {})
  } else {
    events = el.events || (el.events = {})
  }

  const newHandler: any = {
    value: value.trim()
  }
  if (modifiers !== emptyObject) {
    newHandler.modifiers = modifiers
  }

  //第一次处理某个事件时，handlers是undefined
  const handlers = events[name]
  /* istanbul ignore if */
  //可以给同一个事件绑定多个事件处理函数
  //当给ast节点第一次某事件时，走else逻辑
  //第二次走else if逻辑，其后走if中的逻辑
  //important表示是否有优先级
  if (Array.isArray(handlers)) {
    important ? handlers.unshift(newHandler) : handlers.push(newHandler)
  } else if (handlers) {
    events[name] = important ? [newHandler, handlers] : [handlers, newHandler]
  } else {
    events[name] = newHandler
  }

  el.plain = false
}
```

## 二、codegen阶段

在ast对象生成代码字符串阶段，会执行genData方法生成节点的data数据。该方法中，判断ast节点上是否有events或者nativeEvents属性，如果有的话，执行genHandlers方法。

```JavaScript
export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'

  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  if (dirs) data += dirs + ','
  ......
  
  // event handlers
  if (el.events) {
    data += `${genHandlers(el.events, false, state.warn)},`
  }
  if (el.nativeEvents) {
    data += `${genHandlers(el.nativeEvents, true, state.warn)},`
  }
  ......
  data = data.replace(/,$/, '') + '}'
  ......
  return data
}

export function genHandlers (
  events: ASTElementHandlers,
  isNative: boolean,
  warn: Function
): string {
  let res = isNative ? 'nativeOn:{' : 'on:{'
  for (const name in events) {
    res += `"${name}":${genHandler(name, events[name])},`
  }
  //去掉res最后面的逗号
  return res.slice(0, -1) + '}'
}

function genHandler (
  name: string,
  handler: ASTElementHandler | Array<ASTElementHandler>
): string {
  if (!handler) {
    return 'function(){}'
  }

  if (Array.isArray(handler)) {
    return `[${handler.map(handler => genHandler(name, handler)).join(',')}]`
  }
  //匹配事件处理函数，类似于这样的形式method、method.a、method['a']等
  const isMethodPath = simplePathRE.test(handler.value)
  //匹配函数表达式的形式
  const isFunctionExpression = fnExpRE.test(handler.value)
  //上面两种情况没匹配到函数调用的形式，例如本的clickHandler($event)就不能
  //匹配到上面两个正则表达式

  //没有修饰符的情况
  if (!handler.modifiers) {
    if (isMethodPath || isFunctionExpression) {
      return handler.value
    }
    /* istanbul ignore if */
    if (__WEEX__ && handler.params) {
      return genWeexHandler(handler.params, handler.value)
    }
    //当事件处理函数是函数调用的形式，返回下面的字符串
    //例如本例中的clickHandler($event)，走的就是这一步
    //这也是为什么我们在事件处理函数中可以调用$event的原因
    return `function($event){${handler.value}}` // inline statement
  
  //有修饰符的情况
  } else {
    let code = ''
    let genModifierCode = ''
    const keys = []
    for (const key in handler.modifiers) {
      if (modifierCode[key]) {
        //根据key拿到modifierCode中相应的代码字符串
        genModifierCode += modifierCode[key]
        // left/right
        if (keyCodes[key]) {
          keys.push(key)
        }
      } else if (key === 'exact') {
        const modifiers: ASTModifiers = (handler.modifiers: any)
        genModifierCode += genGuard(
          ['ctrl', 'shift', 'alt', 'meta']
            .filter(keyModifier => !modifiers[keyModifier])
            .map(keyModifier => `$event.${keyModifier}Key`)
            .join('||')
        )
      } else {
        keys.push(key)
      }
    }
    if (keys.length) {
      code += genKeyFilter(keys)
    }
    // Make sure modifiers like prevent and stop get executed after key filtering
    if (genModifierCode) {
      code += genModifierCode
    }
    const handlerCode = isMethodPath
      ? `return ${handler.value}($event)`
      : isFunctionExpression
        ? `return (${handler.value})($event)`
        : handler.value
    /* istanbul ignore if */
    if (__WEEX__ && handler.params) {
      return genWeexHandler(handler.params, code + handlerCode)
    }
    //code是修饰符相关的代码，handlerCode是处理函数相关的code
    return `function($event){${code}${handlerCode}}`
  }
}

const modifierCode: { [key: string]: string } = {
  stop: '$event.stopPropagation();',
  prevent: '$event.preventDefault();',
  self: genGuard(`$event.target !== $event.currentTarget`),
  ctrl: genGuard(`!$event.ctrlKey`),
  shift: genGuard(`!$event.shiftKey`),
  alt: genGuard(`!$event.altKey`),
  meta: genGuard(`!$event.metaKey`),
  left: genGuard(`'button' in $event && $event.button !== 0`),
  middle: genGuard(`'button' in $event && $event.button !== 1`),
  right: genGuard(`'button' in $event && $event.button !== 2`)
}

function genKeyFilter (keys: Array<string>): string {
  return `if(!('button' in $event)&&${keys.map(genFilterCode).join('&&')})return null;`
}

function genFilterCode (key: string): string {
  const keyVal = parseInt(key, 10)
  if (keyVal) {
    return `$event.keyCode!==${keyVal}`
  }
  const keyCode = keyCodes[key]
  const keyName = keyNames[key]
  return (
    `_k($event.keyCode,` +
    `${JSON.stringify(key)},` +
    `${JSON.stringify(keyCode)},` +
    `$event.key,` +
    `${JSON.stringify(keyName)}` +
    `)`
  )
}
```

最终，child组件上select事件生成的代码字符串为 `on:{"select":selectHandler}`，click事件生成的代码字符串为 `nativeOn:{"click":function($event){$event.preventDefault();return clickHandler($event)}}`。button标签上click事件生成的代码字符串为 `on:{"click":function($event){clickHandler($event)}}`。

## 三、原生事件

在组件的渲染过程中，render生成渲染vnode后，会调用patch方法。patch方法是调用createPatchFunction生成的。createPatchFunction执行时，会生成一系列钩子函数。这些钩子函数定义在core/vdom/modules/index和web/runtime/modules/index中。

```JavaScript
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

// the directive module should be applied last, after all
// built-in modules have been applied.
const modules = platformModules.concat(baseModules)

export const patch: Function = createPatchFunction({ nodeOps, modules })
```

```JavaScript
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']
export function createPatchFunction (backend) {
	......
	const { modules, nodeOps } = backend
	for (i = 0; i < hooks.length; ++i) {
        cbs[hooks[i]] = []
        for (j = 0; j < modules.length; ++j) {
          if (isDef(modules[j][hooks[i]])) {
            cbs[hooks[i]].push(modules[j][hooks[i]])
          }
        }
    }
    ...
}
```

与事件有关的钩子函数web/runtime/modules/events.js中，有create和update两个钩子。create钩子是在invokeCreateHooks函数中执行的。invokeCreateHooks函数执行有两个时机。一是在createComponent函数中，当组件的占位符vnode渲染成实际dom插入到页面之前会调用initComponent方法。initComponent方法中会调用invokeCreateHooks函数。二是在createElm方法中，当渲染vnode生成实际dom插入到页面之前会调用。

```javascript
function invokeCreateHooks (vnode, insertedVnodeQueue) {
    //执行cbs中的所有create钩子函数
    for (let i = 0; i < cbs.create.length; ++i) {
      cbs.create[i](emptyNode, vnode)
    }
    i = vnode.data.hook // Reuse variable
    if (isDef(i)) {
      if (isDef(i.create)) i.create(emptyNode, vnode)
      if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
    }
  }
```

update钩子是在patchVnode方法中调用的

```JavaScript
if (isDef(data) && isPatchable(vnode)) {
      //这里for循环调用了update钩子函数
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
}
```

事件的create和update钩子函数对应的是一个函数updateDOMListeners。

```JavaScript
function updateDOMListeners (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  //执行create的时候，传入的oldVnode是一个空的vnode
  //oldVnode.data.on是编译生成的on对象。例如例子中的on:{"select":selectHandler}
  if (isUndef(oldVnode.data.on) && isUndef(vnode.data.on)) {
    return
  }
  const on = vnode.data.on || {}
  const oldOn = oldVnode.data.on || {}
  target = vnode.elm
  //处理v-model的，可以暂时不看
  normalizeEvents(on)
  updateListeners(on, oldOn, add, remove, vnode.context)
  target = undefined
}

export function updateListeners (
  on: Object,
  oldOn: Object,
  add: Function,
  remove: Function,
  vm: Component
) {
  let name, def, cur, old, event
  for (name in on) {
    def = cur = on[name]
    old = oldOn[name]
    //解析出事件名中的once、passive和capture修饰符
    event = normalizeEvent(name)
    /* istanbul ignore if */
    if (__WEEX__ && isPlainObject(def)) {
      cur = def.handler
      event.params = def.params
    }
    if (isUndef(cur)) {
      process.env.NODE_ENV !== 'production' && warn(
        `Invalid handler for event "${event.name}": got ` + String(cur),
        vm
      )
    //old不存在，说明是create，创建事件的情况
    } else if (isUndef(old)) {
      //刚开始事件处理函数上的fns属性不存在
      if (isUndef(cur.fns)) {
        //创建最终的事件回调函数
        cur = on[name] = createFnInvoker(cur)
      }
      add(event.name, cur, event.once, event.capture, event.passive, event.params)
    //执行update，更新事件的情况
    } else if (cur !== old) {
      //直接将新的事件处理函数赋给invoker的fns属性
      //因为最终的事件处理函数是invoker函数，invoker函数中会执行invoker的fns属性
      old.fns = cur
      on[name] = old
    }
  }
  for (name in oldOn) {
    if (isUndef(on[name])) {
      event = normalizeEvent(name)
      remove(event.name, oldOn[name], event.capture)
    }
  }
}

const normalizeEvent = cached((name: string): {
  name: string,
  once: boolean,
  capture: boolean,
  passive: boolean,
  handler?: Function,
  params?: Array<any>
} => {
  //在编译阶段，遇到事件上的passive、once和capture修饰符，会添加一些标记
  //这里会解析出标记代表的修饰符
  const passive = name.charAt(0) === '&'
  name = passive ? name.slice(1) : name
  const once = name.charAt(0) === '~' // Prefixed last, checked first
  name = once ? name.slice(1) : name
  const capture = name.charAt(0) === '!'
  name = capture ? name.slice(1) : name
  return {
    name,
    once,
    capture,
    passive
  }
})

//可以给事件定义多个处理函数，因此fns可能是函数，可能是数组
export function createFnInvoker (fns: Function | Array<Function>): Function {
  //invoker是最终的事件回调函数
  function invoker () {
    const fns = invoker.fns
    //执行真正的定义的回调函数
    if (Array.isArray(fns)) {
      const cloned = fns.slice()
      for (let i = 0; i < cloned.length; i++) {
        cloned[i].apply(null, arguments)
      }
    } else {
      // return handler return value for single handlers
      return fns.apply(null, arguments)
    }
  }
  invoker.fns = fns
  return invoker
}
```

在updateDOMListeners方法中，调用add方法给dom添加原生事件。

```JavaScript
function add (
  event: string,
  handler: Function,
  once: boolean,
  capture: boolean,
  passive: boolean
) {
  //调用withMacroTask给事件处理函数做了一层包装，保证事件处理函数在执行时，组件渲染更新走的是
  //宏任务
  handler = withMacroTask(handler)
  //如果有once修饰符，在给事件处理函数包装一层
  //在执行完一次事件处理函数后，调用remove移除监听
  if (once) handler = createOnceHandler(handler, event, capture)
  //调用dom的addEventListener给其添加原生事件监听函数
  target.addEventListener(
    event,
    handler,
    supportsPassive
      ? { capture, passive }
      : capture
  )
}

export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}

function createOnceHandler (handler, event, capture) {
  const _target = target // save current target element in closure
  return function onceHandler () {
    const res = handler.apply(null, arguments)
    if (res !== null) {
      remove(event, onceHandler, capture, _target)
    }
  }
}
```

对于原生的dom来说，只能定义原生事件。对于组件来说，事件上有修饰符native，那么parse最终生成的代码是类似于 `nativeOn:{"click":function($event){$event.preventDefault();return clickHandler($event)}}`这样的。没有的话生成的代码是on。但是给dom添加原生事件的时候看的是on而不是native，因为createComponent方法中做了下面的处理。

```
const listeners = data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn
```

在本文的例子中，child组件上的原生事件click事件处理函数也被添加到了button这个dom元素上。因为child组件的渲染vnode的elm属性指向的是button这个dom。因为在渲染过程中子组件比父组件先patch生成dom，因此子组件先执行invokeCreateHooks函数。所以本文中的button这个dom上的两个原生的click事件，在child组件上的click事件处理函数后注册，后执行。

在点击button按钮的时候，会触发click的事件处理函数clickHandler。因为渲染vnode是调用render函数生成的，在访问clickHandler的时候拿到的是render函数中的clickHandler。而render函数中包了一层this，改变了作用域，访问的是this即组件vm上的clickHandler。

四、自定义事件

自定义事件保存在组件实例vm的$options. _parentListeners上。在组件执行init的过程中，会调用initEvents方法。

```JavaScript
export function initEvents (vm: Component) {
  //_events保留事件中心的所有事件
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}

export function updateComponentListeners (
  vm: Component,
  listeners: Object,
  oldListeners: ?Object
) {
  target = vm
  updateListeners(listeners, oldListeners || {}, add, remove, vm)
  target = undefined
}
```

initEvents中调用了updateComponentListeners方法。updateComponentListeners中又调用了updateListeners方法，与原生事件中调用的那个updateListeners一样，但是传入的add和remove方法是不一样的。

```JavaScript
function add (event, fn, once) {
  if (once) {
    target.$once(event, fn)
  } else {
    target.$on(event, fn)
  }
}

function remove (event, fn) {
  target.$off(event, fn)
}
```

$on、$off、$once和$emit都定义在core/instance/events.js中。这是一个典型的事件中心的实现。	

```JavaScript
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    const vm: Component = this
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        this.$on(event[i], fn)
      }
    } else {
      //看事件中心是否又对应的事件，如果有将处理函数push到数组中，如果没有，新生成
      //一个空数组存放该事件的所有事件处理函数
      (vm._events[event] || (vm._events[event] = [])).push(fn)
      // optimize hook:event cost by using a boolean flag marked at registration
      // instead of a hash lookup
      if (hookRE.test(event)) {
        vm._hasHookEvent = true
      }
    }
    return vm
}

Vue.prototype.$once = function (event: string, fn: Function): Component {
    const vm: Component = this
    //$once保证事件执行一次，然后就被$off注销掉
    function on () {
      vm.$off(event, on)
      fn.apply(vm, arguments)
    }
    on.fn = fn
    vm.$on(event, on)
    return vm
}

Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    const vm: Component = this
    // all
    //当$off不传任何参数时，清空组件vm上的所有事件
    if (!arguments.length) {
      vm._events = Object.create(null)
      return vm
    }
    // array of events
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        this.$off(event[i], fn)
      }
      return vm
    }
    // specific event
    const cbs = vm._events[event]
    if (!cbs) {
      return vm
    }
    //当$off只传了一个参数时，会清楚该事件的所有处理函数
    if (!fn) {
      vm._events[event] = null
      return vm
    }
    if (fn) {
      // specific handler
      let cb
      let i = cbs.length
      while (i--) {
        cb = cbs[i]
        if (cb === fn || cb.fn === fn) {
          cbs.splice(i, 1)
          break
        }
      }
    }
    return vm
}

Vue.prototype.$emit = function (event: string): Component {
    const vm: Component = this
    if (process.env.NODE_ENV !== 'production') {
      const lowerCaseEvent = event.toLowerCase()
      if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
        tip(
          `Event "${lowerCaseEvent}" is emitted in component ` +
          `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
          `Note that HTML attributes are case-insensitive and you cannot use ` +
          `v-on to listen to camelCase events when using in-DOM templates. ` +
          `You should probably use "${hyphenate(event)}" instead of "${event}".`
        )
      }
    }
    let cbs = vm._events[event]
    if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      for (let i = 0, l = cbs.length; i < l; i++) {
        try {
          cbs[i].apply(vm, args)
        } catch (e) {
          handleError(e, vm, `event handler for "${event}"`)
        }
      }
    }
    return vm
}
```

vue的自定义事件是典型的事件中心的实现。有一个vm. _events对象保存所有的事件处理函数。其中 _events对象中的key是事件名，对应的值是一个数组，保存对应的事件处理函数。当调用$on的时候将事件处理函数push到 _events对象相应的事件的数组中。当调用$emit的时候，拿到vm _events中相应的事件处理函数，对该数组循环执行一遍。$emit和$on实现父子组件间的通信，看起来像是父组件监听子组件的事件，其实不是。其实事件都保存在子组件上，只不过事件处理函数是定义在父组件中罢了。

例如本文例子中，child组件上自定义了select事件。select事件及其事件处理函数被保存在child组件的vm. _events对象中。当在child组件内部调用$emit的时候会执行select的事件处理函数。