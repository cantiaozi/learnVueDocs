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
    //判断是否是vue自定义属性，以v-或者：开头
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

