本文以下面的例子来讲解v-model的原理。

```JavaScript
import Vue from 'vue'
new Vue({
    el: '#app',
    template: '<div>' + '<p>Message is: {{ message }}</p>'
        + '<input v-model="message" placeholder="edit me">' + '</div>',
    data() {
        return {
            message: ''
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
      //普通指令的检测，v-model会执行这里面的逻辑
      } else { // normal directives
        //去掉v-
        name = name.replace(dirRE, '')
        // parse arg
        const argMatch = name.match(argRE)
        const arg = argMatch && argMatch[1]
        if (arg) {
          name = name.slice(0, -(arg.length + 1))
        }
        //调用addDirective方法将指令添加到ast对象的               //directives属性上
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

export function addDirective (
  el: ASTElement,
  name: string,
  rawName: string,
  value: string,
  arg: ?string,
  modifiers: ?ASTModifiers
) {
  (el.directives || (el.directives = [])).push({ name, rawName, value, arg, modifiers })
  el.plain = false
}
```

## 二、codegen阶段

在ast对象生成代码字符串阶段，会执行genData方法生成节点的data数据。该方法中，首先会执行genDirectives方法。genDirectives方法中首先会拿到编译model指令的函数并调用，model函数定义在platforms/web/compiler/directives/model.js中。model函数中会根据v-model指令所绑定的tag的类型执行不同的分支逻辑。在本文例子中，执行的是genDefaultModel方法。最后model函数返回true。genDefaultModel中首先会生成一段代码字符串，然后调用addProp和addHandler方法。

```JavaScript
export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'

  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  ......
  return data
}

function genDirectives (el: ASTElement, state: CodegenState): string | void {
  const dirs = el.directives
  if (!dirs) return
  let res = 'directives:['
  let hasRuntime = false
  let i, l, dir, needRuntime
  //遍历ast节点的directives上的属性
  for (i = 0, l = dirs.length; i < l; i++) {
    dir = dirs[i]
    needRuntime = true
    //通过state.directives拿到编译各种指令的函数
    //编译指令的函数定义在platforms/web/compiler/directives
    const gen: DirectiveFunction = state.directives[dir.name]
    if (gen) {
      // compile-time directive that manipulates AST.
      // returns true if it also needs a runtime counterpart.
      //model函数返回true
      needRuntime = !!gen(el, dir, state.warn)
    }
    if (needRuntime) {
      hasRuntime = true
      res += `{name:"${dir.name}",rawName:"${dir.rawName}"${
        dir.value ? `,value:(${dir.value}),expression:${JSON.stringify(dir.value)}` : ''
      }${
        dir.arg ? `,arg:"${dir.arg}"` : ''
      }${
        dir.modifiers ? `,modifiers:${JSON.stringify(dir.modifiers)}` : ''
      }},`
    }
  }
  if (hasRuntime) {
    return res.slice(0, -1) + ']'
  }
}
```



```JavaScript
export default function model (
  el: ASTElement,
  dir: ASTDirective,
  _warn: Function
): ?boolean {
  warn = _warn
  const value = dir.value
  const modifiers = dir.modifiers
  const tag = el.tag
  const type = el.attrsMap.type

  if (process.env.NODE_ENV !== 'production') {
    // inputs with type="file" are read only and setting the input's
    // value will throw an error.
    if (tag === 'input' && type === 'file') {
      warn(
        `<${el.tag} v-model="${value}" type="file">:\n` +
        `File inputs are read only. Use a v-on:change listener instead.`
      )
    }
  }

  if (el.component) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  //判断不同的tag，执行不同的逻辑
  } else if (tag === 'select') {
    genSelect(el, value, modifiers)
  } else if (tag === 'input' && type === 'checkbox') {
    genCheckboxModel(el, value, modifiers)
  } else if (tag === 'input' && type === 'radio') {
    genRadioModel(el, value, modifiers)
  } else if (tag === 'input' || tag === 'textarea') {
    //本文例子走的是这里的逻辑。
    genDefaultModel(el, value, modifiers)
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `<${el.tag} v-model="${value}">: ` +
      `v-model is not supported on this element type. ` +
      'If you are working with contenteditable, it\'s recommended to ' +
      'wrap a library dedicated for that purpose inside a custom component.'
    )
  }

  // ensure runtime directive metadata
  return true
}

function genDefaultModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
): ?boolean {
  const type = el.attrsMap.type

  // warn if v-bind:value conflicts with v-model
  // except for inputs with v-bind:type
  //当v-bind和v-model同时存在时，会报一个错误提示
  if (process.env.NODE_ENV !== 'production') {
    const value = el.attrsMap['v-bind:value'] || el.attrsMap[':value']
    const typeBinding = el.attrsMap['v-bind:type'] || el.attrsMap[':type']
    if (value && !typeBinding) {
      const binding = el.attrsMap['v-bind:value'] ? 'v-bind:value' : ':value'
      warn(
        `${binding}="${value}" conflicts with v-model on the same element ` +
        'because the latter already expands to a value binding internally'
      )
    }
  }

  const { lazy, number, trim } = modifiers || {}
  const needCompositionGuard = !lazy && type !== 'range'
  const event = lazy
    ? 'change'
    : type === 'range'
      ? RANGE_TOKEN
      : 'input'

  let valueExpression = '$event.target.value'
  if (trim) {
    valueExpression = `$event.target.value.trim()`
  }
  if (number) {
    valueExpression = `_n(${valueExpression})`
  }
  //调用genAssignmentCode生成一段代码字符串
  let code = genAssignmentCode(value, valueExpression)
  if (needCompositionGuard) {
    code = `if($event.target.composing)return;${code}`
  }

  addProp(el, 'value', `(${value})`)
  addHandler(el, event, code, null, true)
  if (trim || number) {
    addHandler(el, 'blur', '$forceUpdate()')
  }
}

export function genAssignmentCode (
  value: string,
  assignment: string
): string {
  const res = parseModel(value)
  if (res.key === null) {
    return `${value}=${assignment}`
  } else {
    return `$set(${res.exp}, ${res.key}, ${assignment})`
  }
}
```

addProp和addHandler是v-model实现的核心。其本质就是v-bind:value=“xxx”和@input=“xxx=$event.target.value”的语法糖。

addProp方法向ast节点的props数组上添加一个value属性。addHandler方法向ast节点的events属性上添加事件和事件处理函数。

```JavaScript
export function addProp (el: ASTElement, name: string, value: string) {
  (el.props || (el.props = [])).push({ name, value })
  el.plain = false
}

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
    if (modifiers.right) {
      name = 'contextmenu'
      delete modifiers.right
    } else if (modifiers.middle) {
      name = 'mouseup'
    }
  }

  let events
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

  const handlers = events[name]
  /* istanbul ignore if */
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

本文例子最后codegen生成的代码字符串`"with(this){return _c('div',[_c('p',[_v(\"Message is: \"+_s(message))]),_c('input',{directives:[{name:\"model\",rawName:\"v-model\",value:(message),expression:\"message\"}],attrs:{\"placeholder\":\"edit me\"},domProps:{\"value\":(message)},on:{\"input\":function($event){if($event.target.composing)return;message=$event.target.value}}})])}"`

## 三、运行时阶段

v-model是v-bind和v-on的语法糖。因此，在本文例子中，下面的写法是等效的。

```JavaScript
new Vue({
    el: '#app',
    template: '<div>' + '<p>Message is: {{ message }}</p>'
        + '<input :value="message" @input="message=$event.target.value" placeholder="edit me">' + '</div>',
    data() {
        return {
            message: ''
        }
    }
})
```

但是两种方式也不完全相同。当输入一个中文的时候，没有输入结束，v-model绑定的值是不变的，而v-bind绑定的值是变化了的。当使用v-model的时候，见下图。

![](D:\前端开发笔记\JavaScript\learnVueDocs-main\others\v-model与语法糖的区别1.png)

当使用v-bind和v-on的时候，情况如下图。![](D:\前端开发笔记\JavaScript\learnVueDocs-main\others\v-model和语法糖的区别2.png)

### 1.表单上使用v-model

上一篇文章我们在分析events的时候得知，当组件在patch的时候会执行很多的钩子函数。与directives指令相关的钩子函数定义在core/vdom/modules/directives下面的，包括创建、更新和销毁三个钩子函数。

```JavaScript
export default {
  create: updateDirectives,
  update: updateDirectives,
  destroy: function unbindDirectives (vnode: VNodeWithData) {
    updateDirectives(vnode, emptyNode)
  }
}

function updateDirectives (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (oldVnode.data.directives || vnode.data.directives) {
    _update(oldVnode, vnode)
  }
}

function _update (oldVnode, vnode) {
  const isCreate = oldVnode === emptyNode
  const isDestroy = vnode === emptyNode
  //parse生成的指令都存在于vnode.data.directives上
  //拿到指令的定义
  const oldDirs = normalizeDirectives(oldVnode.data.directives, oldVnode.context)
  const newDirs = normalizeDirectives(vnode.data.directives, vnode.context)

  const dirsWithInsert = []
  const dirsWithPostpatch = []

  let key, oldDir, dir
  for (key in newDirs) {
    oldDir = oldDirs[key]
    dir = newDirs[key]
    if (!oldDir) {
      // new directive, bind
      //调用callHook函数，执行指令定义中的bind函数
      callHook(dir, 'bind', vnode, oldVnode)
      //如果指令的定义中存在inserted钩子函数，将其push到数组中
      if (dir.def && dir.def.inserted) {
        dirsWithInsert.push(dir)
      }
    } else {
      // existing directive, update
      dir.oldValue = oldDir.value
      //调用callHook函数，执行指令定义中的update函数
      //因为指令更新的钩子函数是在patchVnode中执行的，此时正是处于更新的时机
      callHook(dir, 'update', vnode, oldVnode)
      if (dir.def && dir.def.componentUpdated) {
        dirsWithPostpatch.push(dir)
      }
    }
  }

  if (dirsWithInsert.length) {
    const callInsert = () => {
      for (let i = 0; i < dirsWithInsert.length; i++) {
        callHook(dirsWithInsert[i], 'inserted', vnode, oldVnode)
      }
    }
    if (isCreate) {
      //将callInsert函数设置为vnode的insert钩子函数
      mergeVNodeHook(vnode, 'insert', callInsert)
    } else {
      callInsert()
    }
  }

  if (dirsWithPostpatch.length) {
    mergeVNodeHook(vnode, 'postpatch', () => {
      for (let i = 0; i < dirsWithPostpatch.length; i++) {
        callHook(dirsWithPostpatch[i], 'componentUpdated', vnode, oldVnode)
      }
    })
  }

  if (!isCreate) {
    for (key in oldDirs) {
      if (!newDirs[key]) {
        // no longer present, unbind
        callHook(oldDirs[key], 'unbind', oldVnode, oldVnode, isDestroy)
      }
    }
  }
}

export function mergeVNodeHook (def: Object, hookKey: string, hook: Function) {
  if (def instanceof VNode) {
    def = def.data.hook || (def.data.hook = {})
  }
  let invoker
  const oldHook = def[hookKey]

  function wrappedHook () {
    hook.apply(this, arguments)
    // important: remove merged hook to ensure it's called only once
    // and prevent memory leak
    remove(invoker.fns, wrappedHook)
  }

  if (isUndef(oldHook)) {
    // no existing hook
    //createFnInvoker在events一文中提到过，返回一个invoker函数，
    //invoker函数内会执行invoker.fns函数
    //如果invoker.fns是一个函数数组的话，会遍历执行数组中的每个函数
    invoker = createFnInvoker([wrappedHook])
  } else {
    /* istanbul ignore if */
    //如果原来的vnode.data.hook中有这个钩子函数，那么将新的钩子函数
    //push到invoker.fns数组上
    if (isDef(oldHook.fns) && isTrue(oldHook.merged)) {
      // already a merged invoker
      invoker = oldHook
      invoker.fns.push(wrappedHook)
    } else {
      // existing plain hook
      invoker = createFnInvoker([oldHook, wrappedHook])
    }
  }

  invoker.merged = true
  def[hookKey] = invoker
}
```

创建和更新是相同的钩子函数updateDirectives。updateDirectives中调用了 _update函数。 _update函数中调用了normalizeDirectives函数，normalizeDirectives函数会遍历组件上定义的每个指令，拿到每个指令的定义。然后在 _update中判断如果指令是新添加的，就会调用指令的定义中的bind钩子函数；如果是组件更新的话，就会调用指令的定义中的update钩子函数。针对v-model这个指令而言，接着会调用mergeVNodeHook函数在vnode.data.hook上生成一个insert钩子函数。而这个vnode.data.hook上的insert钩子函数，其实就是调用指令中定义的inserted函数。

```JavaScript
function normalizeDirectives (
  dirs: ?Array<VNodeDirective>,
  vm: Component
): { [key: string]: VNodeDirective } {
  const res = Object.create(null)
  if (!dirs) {
    // $flow-disable-line
    return res
  }
  let i, dir
  //遍历所有指令进行处理
  for (i = 0; i < dirs.length; i++) {
    dir = dirs[i]
    if (!dir.modifiers) {
      // $flow-disable-line
      dir.modifiers = emptyModifiers
    }
    res[getRawDirName(dir)] = dir
    //调用resolveAsset方法拿到vm.$options的directives属性上的某个指令的定义
    dir.def = resolveAsset(vm.$options, 'directives', dir.name, true)
  }
  // $flow-disable-line
  return res
}

function getRawDirName (dir: VNodeDirective): string {
  return dir.rawName || `${dir.name}.${Object.keys(dir.modifiers || {}).join('.')}`
}
```

那么vnode.data.hook上的insert钩子函数是在什么时候调用的呢？

是在patch函数的最后调用的。此时在patch函数中已经执行完了createElm函数，createElm会生成vnode的真实dom节点并且插入到父节点中，因此已经完成了dom的插入，正是执行指令的inserted钩子的时机。

```JavaScript
function patch (oldVnode, vnode, hydrating, removeOnly) {
    ......
    //调用vnode.data.hook上的inserted钩子
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
}

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
```

当我们在组件或者vue实例初始化的时候，会在Vue.options上生成一个directives属性。然后在platforms/web/runtime/index.js中将vue原生的指令合并到Vue.options.directives上。vue的原生的指令platformDirectives都定义在platforms/web/runtime/directives 中。	

```JavaScript
function initGlobalAPI (Vue: GlobalAPI) {
	ASSET_TYPES.forEach(type => {
        Vue.options[type + 's'] = Object.create(null)
    })
}

//platforms/web/runtime/index.js
extend(Vue.options.directives, platformDirectives)
```

v-model的指令的定义如下，定义了两个钩子函数inserted和componentUpdated。

inserted钩子函数中，会往真实dom对象上添加两个事件compositionstart和compositionend。compositionstart的事件处理函数中，会给e.target添加一个composing属性，值为true。compositionend事件的处理函数中，会给e.target.composing置为false

```JavaScript
const directive = {
  //el是真实的dom对象，binding是指令对象，包括指令名，指令定义和指令绑定的值等
  inserted (el, binding, vnode, oldVnode) {
    if (vnode.tag === 'select') {
      // #6903
      if (oldVnode.elm && !oldVnode.elm._vOptions) {
        mergeVNodeHook(vnode, 'postpatch', () => {
          directive.componentUpdated(el, binding, vnode)
        })
      } else {
        setSelected(el, binding, vnode.context)
      }
      el._vOptions = [].map.call(el.options, getValue)
    //本文中的例子走的是这个逻辑
    } else if (vnode.tag === 'textarea' || isTextInputType(el.type)) {
      el._vModifiers = binding.modifiers
      if (!binding.modifiers.lazy) {
        //开始新的输入合成时会触发 compositionstart 事件.
        //例如，当用户使用拼音输入法开始输入汉字时，这个事件就会被触发
        el.addEventListener('compositionstart', onCompositionStart)
        //当用户使用拼音输入法输入汉字结束时，这个compositionend事件就会被触发
        el.addEventListener('compositionend', onCompositionEnd)
        // Safari < 10.2 & UIWebView doesn't fire compositionend when
        // switching focus before confirming composition choice
        // this also fixes the issue where some browsers e.g. iOS Chrome
        // fires "change" instead of "input" on autocomplete.
        el.addEventListener('change', onCompositionEnd)
        /* istanbul ignore if */
        if (isIE9) {
          el.vmodel = true
        }
      }
    }
  },

  componentUpdated (el, binding, vnode) {
    if (vnode.tag === 'select') {
      setSelected(el, binding, vnode.context)
      // in case the options rendered by v-for have changed,
      // it's possible that the value is out-of-sync with the rendered options.
      // detect such cases and filter out values that no longer has a matching
      // option in the DOM.
      const prevOptions = el._vOptions
      const curOptions = el._vOptions = [].map.call(el.options, getValue)
      if (curOptions.some((o, i) => !looseEqual(o, prevOptions[i]))) {
        // trigger change event if
        // no matching option found for at least one value
        const needReset = el.multiple
          ? binding.value.some(v => hasNoMatchingOption(v, curOptions))
          : binding.value !== binding.oldValue && hasNoMatchingOption(binding.value, curOptions)
        if (needReset) {
          trigger(el, 'change')
        }
      }
    }
  }
}

function onCompositionStart (e) {
  e.target.composing = true
}

function onCompositionEnd (e) {
  // prevent triggering an input event for no reason
  if (!e.target.composing) return
  e.target.composing = false
  trigger(e.target, 'input')
}

function trigger (el, type) {
  //创建一个事件，定义事件名为input
  const e = document.createEvent('HTMLEvents')
  e.initEvent(type, true, true)
  //在dom对象el上触发这个名为input的事件
  el.dispatchEvent(e)
}
```

在上一小节中，可以得知，v-model编译后会给绑定的元素添加一个input事件。`on:{\"input\":function($event){if($event.target.composing)return;message=$event.target.value}}`。当我们在使用中文输入法输入中文时，会同时触发compositionstart和input事件。因为compositionstart的处理函数中将e.target.composing置为了true，因此在input的处理函数中就会直接返回，不做任何操作。当中文输入结束后，会触发compositionend事件，将e.target.composing置为false，同时触发一次input事件，此时input的处理函数中就会执行 `message=$event.target.value` 这段代码。这就是使用v-model输入中文时，在中文未输入完成时，v-model绑定的数据不改变的原因。

对本文中的实例而言，v-model指令定义中的componentUpdated是未涉及的。

### 2.组件上使用v-model

看下面的这个例子。这个实例可以在组件上实现双向绑定。

```JavaScript
import Vue from '../node_modules/vue/dist/vue.esm'
let child = {
    template: '<div>' + '<input :value="value" @input="updateValue" placeholder="edit me">' 
        + '</div>',
    props: ['value'],
    methods: {
        updateValue(e) {
            this.$emit('input', e.target.value)
        }
    }
}
new Vue({
    el: '#app',
    template: '<div>' + '<child v-model="message"></child>' 
        + '<p>message is {{message}}</p>' + '</div>',
    data() {
        return {
            message: ''
        }
    },
    components: {
        child
    }
})
```

在组件上使用v-model，它的parse阶段和在表单上是一样的。但是codegen阶段的model函数中，调用的是genComponentModel函数。然后在genData函数中，判断ast节点el上是否有model属性，如果有的话会给最终生成的代码中加上一段代码。

```JavaScript
export function genComponentModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
): ?boolean {
  const { number, trim } = modifiers || {}

  const baseValueExpression = '$$v'
  let valueExpression = baseValueExpression
  if (trim) {
    valueExpression =
      `(typeof ${baseValueExpression} === 'string'` +
      `? ${baseValueExpression}.trim()` +
      `: ${baseValueExpression})`
  }
  if (number) {
    valueExpression = `_n(${valueExpression})`
  }
  //genAssignmentCode生成代码字符串
  const assignment = genAssignmentCode(value, valueExpression)
  //在ast节点上生成model属性
  el.model = {
    value: `(${value})`,
    expression: `"${value}"`,
    callback: `function (${baseValueExpression}) {${assignment}}`
  }
}

export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'

  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  if (dirs) data += dirs + ','
  ......
  
  // component v-model
  if (el.model) {
    data += `model:{value:${
      el.model.value
    },callback:${
      el.model.callback
    },expression:${
      el.model.expression
    }},`
  }
  // inline-template
  if (el.inlineTemplate) {
    const inlineTemplate = genInlineTemplate(el, state)
    if (inlineTemplate) {
      data += `${inlineTemplate},`
    }
  }
  data = data.replace(/,$/, '') + '}'
  // v-bind data wrap
  if (el.wrapData) {
    data = el.wrapData(data)
  }
  // v-on data wrap
  if (el.wrapListeners) {
    data = el.wrapListeners(data)
  }
  return data
}

```

给ast节点添加的model属性还在哪里用到了呢？在生成组件的占位符vnode时调用createComponent方法，该方法中用到了model对象。

```JavaScript
function createComponent {
  ......
  // transform component v-model data into props & events
  //本文实例生成的data.model为{model:{value:(message),callback:function ($$v)           {message=$$v},expression:\"message\"}}
  if (isDef(data.model)) {
    //将v-model转化为prop和自定义事件
    transformModel(Ctor.options, data)
  }
  ......
}
  
function transformModel (options, data: any) {
  const prop = (options.model && options.model.prop) || 'value'
  const event = (options.model && options.model.event) || 'input'
  ;(data.props || (data.props = {}))[prop] = data.model.value
  const on = data.on || (data.on = {})
  if (isDef(on[event])) {
    on[event] = [data.model.callback].concat(on[event])
  } else {
    on[event] = data.model.callback
  }
}
```

最终，new Vue实例编译生成的代码 `with(this){return _c('div',[_c('child',{model:{value:(message),callback:function ($$v) {message=$$v},expression:\"message\"}}),_c('p',[_v(\"message is \"+_s(message))])],1)}`。v-model经过转化相当于是在组件上设置了prop和events，在本文的例子中如下图所示。

![](E:\开发文档\新建文件夹\learnVueDocs\others\组件上v-model.png)

当在child组件上定义下面的配置时，那么v-model经过转化的prop的key是checked，监听的自定义事件的名称为change。

```JavaScript
model: {
    prop: 'checked',
    event: 'change'
},
```

