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

一、编译阶段

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

