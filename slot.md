本文以下面的实例来讲解插槽的源码。

```JavaScript
/* eslint-disable */
import Vue from '../node_modules/vue/dist/vue.esm'
let child = {
    template: '<div class="content">' + 
        + '<div><slot name="header"></slot></div>'
        + '<main><slot>默认内容</slot></main>'
        + '<footer><slot name="footer"></slot></footer>'
        + '</div>',
}
new Vue({
    el: '#app',
    template: '<div>' + '<child>' + '<h1 slot="header">{{msg}}</h1>'
        + '<p>{{msg}}</p>'
        + '<p slot="footer">{{desc}}</p>'
        + '</child>'
        + '</div>',
    data() {
        return {
            title: '我是标题',
            msg: '我是内容',
            desc: '其它信息'
        }
    },
    components: {
        child
    }
})
```

## 一、编译阶段

在编译生成ast节点的过程中，template字符串匹配到开始标签，调用start方法生成ast对象的过程中，会先创建ast对象，然后管理ast对象，最后管理树结构。在管理ast对象的过程中，会调用processElement方法处理标签上的属性。processElement方法中会调用processSlot方法处理与插槽相关的东西。

在编译父模板时，父模板中没有slot标签，走else中的逻辑。

在编译子组件时，子组件中有slot标签，走if中的逻辑。

```JavaScript
function processSlot (el) {
  if (el.tag === 'slot') {
    //获取ast节点el上的name属性值
    el.slotName = getBindingAttr(el, 'name')
    if (process.env.NODE_ENV !== 'production' && el.key) {
      warn(
        `\`key\` does not work on <slot> because slots are abstract outlets ` +
        `and can possibly expand into multiple elements. ` +
        `Use the key on a wrapping element instead.`
      )
    }
  } else {
    let slotScope
    //if和else if中的逻辑与作用域插槽有关
    if (el.tag === 'template') {
      slotScope = getAndRemoveAttr(el, 'scope')
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && slotScope) {
        warn(
          `the "scope" attribute for scoped slots have been deprecated and ` +
          `replaced by "slot-scope" since 2.5. The new "slot-scope" attribute ` +
          `can also be used on plain elements in addition to <template> to ` +
          `denote scoped slots.`,
          true
        )
      }
      el.slotScope = slotScope || getAndRemoveAttr(el, 'slot-scope')
    } else if ((slotScope = getAndRemoveAttr(el, 'slot-scope'))) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && el.attrsMap['v-for']) {
        warn(
          `Ambiguous combined usage of slot-scope and v-for on <${el.tag}> ` +
          `(v-for takes higher priority). Use a wrapper <template> for the ` +
          `scoped slot to make it clearer.`,
          true
        )
      }
      el.slotScope = slotScope
    }
    //获取ast节点上的slot属性值
    const slotTarget = getBindingAttr(el, 'slot')
    if (slotTarget) {
      el.slotTarget = slotTarget === '""' ? '"default"' : slotTarget
      // preserve slot as an attribute for native shadow DOM compat
      // only for non-scoped slots.
      if (el.tag !== 'template' && !el.slotScope) {
        //在ast节点el上的attrs属性上添加slot属性值
        addAttr(el, 'slot', slotTarget)
      }
    }
  }
}

export function getBindingAttr (
  el: ASTElement,
  name: string,
  getStatic?: boolean
): ?string {
  //如果属性不是动态绑定的，那么dynamicValue为null
  const dynamicValue =
    getAndRemoveAttr(el, ':' + name) ||
    getAndRemoveAttr(el, 'v-bind:' + name)
  if (dynamicValue != null) {
    return parseFilters(dynamicValue)
  } else if (getStatic !== false) {
    //获取ast节点上的属性名为name的属性值，静态绑定
    const staticValue = getAndRemoveAttr(el, name)
    if (staticValue != null) {
      return JSON.stringify(staticValue)
    }
  }
}

```

在codegen阶段的genData函数中，和插槽有关的代码如下。

```JavaScript
export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'
  
  ......
  // slot target
  // only for non-scoped slots
  //有插槽且非作用域插槽的时候
  if (el.slotTarget && !el.slotScope) {
    data += `slot:${el.slotTarget},`
  }
  // scoped slots
  if (el.scopedSlots) {
    data += `${genScopedSlots(el.scopedSlots, state)},`
  }
  
  ......
  return data
}
```

在genElement函数中判断当标签名是slot的时候会执行了genSlot函数，这是和子组件的编译有关的，和父组件无关。

```JavaScript
function genSlot (el: ASTElement, state: CodegenState): string {
  const slotName = el.slotName || '"default"'
  //生成插槽slot标签内包裹的子节点，当父组件不传入插槽时，会默认渲染slot标签内包裹的子节点
  const children = genChildren(el, state)
  //这里的_t是renderSlot函数
  let res = `_t(${slotName}${children ? `,${children}` : ''}`
  //如果插槽slot标签上还有一些属性，需要拼接成字符串
  const attrs = el.attrs && `{${el.attrs.map(a => `${camelize(a.name)}:${a.value}`).join(',')}}`
  //处理动态绑定的相关内容
  const bind = el.attrsMap['v-bind']
  if ((attrs || bind) && !children) {
    res += `,null`
  }
  if (attrs) {
    res += `,${attrs}`
  }
  if (bind) {
    res += `${attrs ? '' : ',null'},${bind}`
  }
  return res + ')'
}
```

最终父组件new Vue编译生成的code字符串为 `"with(this){return _c('div',[_c('child',[_c('h1',{attrs:{\"slot\":\"header\"},slot:\"header\"},[_v(_s(msg))]),_c('p',[_v(_s(msg))]),_c('p',{attrs:{\"slot\":\"footer\"},slot:\"footer\"},[_v(_s(desc))])])],1)}"`

child组件生成的code字符串为 `"with(this){return _c('div',{staticClass:\"content\"},[_c('header',[_t(\"header\")],2),_c('main',[_t(\"default\",[_v(\"默认内容\")])],2),_c('footer',[_t(\"footer\")],2)])}"`

## 二、运行时阶段

slot插槽编译生成的代码中的_t代表renderSlot函数。renderSlot定义在core/instance/render-helpers/render-slot.js中。renderSlot里的逻辑分为两部分，一部分是作用域插槽的，一部分是普通插槽的。对于普通插槽来说，逻辑非常简单，就是拿到this.$slots中相应的插槽vnode并返回。这样，插槽slot标签就被渲染出来了。this.$slots是一个对象，key是插槽名称，值是就是组件中的相应的插槽生成的vnode对象。

```JavaScript
export function renderSlot (
  name: string,
  fallback: ?Array<VNode>,//fallback参数是父组件没有使用插槽时，插槽默认生成的vnode
  //在我们的例子中就是文本 ‘默认内容’
  props: ?Object,
  bindObject: ?Object
): ?Array<VNode> {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes
  //作用域插槽
  if (scopedSlotFn) { // scoped slot
    props = props || {}
    if (bindObject) {
      if (process.env.NODE_ENV !== 'production' && !isObject(bindObject)) {
        warn(
          'slot v-bind without argument expects an Object',
          this
        )
      }
      props = extend(extend({}, bindObject), props)
    }
    nodes = scopedSlotFn(props) || fallback
  } else {
    const slotNodes = this.$slots[name]
    // warn duplicate slot usage
    if (slotNodes) {
      if (process.env.NODE_ENV !== 'production' && slotNodes._rendered) {
        warn(
          `Duplicate presence of slot "${name}" found in the same render tree ` +
          `- this will likely cause render errors.`,
          this
        )
      }
      slotNodes._rendered = true
    }
    nodes = slotNodes || fallback
  }

  const target = props && props.slot
  if (target) {
    return this.$createElement('template', { slot: target }, nodes)
  } else {
    return nodes
  }
}
```

组件实例上的$slots是何时生成的呢？在组件实例化调用init的时候，init函数中会执行initRender函数。

```JavaScript
function initRender (vm: Component) {
    //占位符的context是组件的作用域或者说环境，即组件child的父组件Vue实例
	const renderContext = parentVnode && parentVnode.context
    vm.$slots = resolveSlots(options._renderChildren, renderContext)
}
```

initRender函数中调用了resolveSlots函数生成了$slots。options._renderChildren就是组件的子组件vnode，本文例子中child组件的children长度为3，分别是标签为h1的vnode，p标签的vnode和拥有slot属性的p标签的vnode。在child组件被生成占位符vnode的时候，调用的是createComponent函数生成的占位符vnode，生成的占位符vnode中的children属性是一个数组，包含了标签为h1的vnode，p标签的vnode和拥有slot属性的p标签的vnode。后面child组件在初始化执行init函数的时候，init函数内调用了initInternalComponent函数，该函数中将options的 _renderChildren指向了占位符vnode上的children。

```JavaScript
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  //vm.constructor.options是构造器的options，在global-api文件夹里面的extend.js中有定义
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

resolveSlots的定义如下。其参数context是组件实例。resolveSlots函数会遍历子节点的vnode，将vnode放置在slots这个对象的相应的key下的数组中。

```Javascript
export function resolveSlots (
  children: ?Array<VNode>,
  context: ?Component
): { [key: string]: Array<VNode> } {
  const slots = {}
  if (!children) {
    return slots
  }
  //遍历子节点渲染vnode
  for (let i = 0, l = children.length; i < l; i++) {
    const child = children[i]
    const data = child.data
    // remove slot attribute if the node is resolved as a Vue slot node
    if (data && data.attrs && data.attrs.slot) {
      delete data.attrs.slot
    }
    // named slots should only be respected if the vnode was rendered in the
    // same context.
    //组件vnode和组件内的插槽vnode是在同一个环境中生成的
    //本文实例中child的占位符vnode和插槽vnode，h1和p等是在同一个环境即
    //vue实例环境中生成的
    if ((child.context === context || child.fnContext === context) &&
      data && data.slot != null
    ) {
      //拿到子节点上的slot属性的值，作为slots对象的key
      const name = data.slot
      const slot = (slots[name] || (slots[name] = []))
      if (child.tag === 'template') {
        slot.push.apply(slot, child.children || [])
      } else {
        slot.push(child)
      }
    } else {
      (slots.default || (slots.default = [])).push(child)
    }
  }
  // ignore slots that contains only whitespace
  for (const name in slots) {
    if (slots[name].every(isWhitespace)) {
      delete slots[name]
    }
  }
  return slots
}
```

因为插槽的渲染vnode都是在父组件内生成的，因此插槽内的数据访问的都是父组件内的数据。在本文的例子中，插槽的渲染vnode有3个，插槽内的数据msg等，访问的是父组件new Vue内的数据。