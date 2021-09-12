VueX

## 一、初始化

### 1.1 vuex注册

vuex的入口文件是src/index.js。当vuex注册的时候调用了vuex.install方法。

```JavaScript
export function install (_Vue) {
  //保证只注册一次
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}

function applyMixin (Vue) {
  const version = Number(Vue.version.split('.')[0])

  //只需要关注vue版本大于2的时候
  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    //new Vue的配置中传入了store，执行if逻辑
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    //对于其它vue组件其beforeCreate函数在父组件之后执行，因此都可以拿到$store
    //这也是每个组件都可以通过vm.$store拿到store对象的原因。
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}

```

### 1.2 VueX.Store初始化

Store类的构造函数

```javascript
constructor (options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }
	//vuex必须先注册在创建Store类
    //vuex必须支持store类
    //store类必须通过构造函数方式调用
    if (process.env.NODE_ENV !== 'production') {
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `Store must be called with the new operator.`)
    }

    const {
      plugins = [],
      strict = false
    } = options

    //定义一些变量并进行初始化赋值，_actionSubscribers和_subscribers与订阅者相关
    // store internal state
    this._committing = false
    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()

    // bind commit and dispatch to self
    const store = this
    //拿到store对象的dispatch和commit方法
    const { dispatch, commit } = this
    //给dispatch和commit函数绑定this为store实例
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    this.strict = strict

    const state = this._modules.root.state

    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], this._modules.root)

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state)

    // apply plugins
    plugins.forEach(plugin => plugin(this))

    if (Vue.config.devtools) {
      devtoolPlugin(this)
    }
  }
```

#### 1.2.1 变量modules的初始化

在store的构造函数中，_modules变量是一个ModuleCollection实例化对象。

```JavaScript
export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  get (path) {
    //reduce方法会将每次函数执行的输出值作为下一次的输入值
    //第一次函数执行的输入值是this.root
    return path.reduce((module, key) => {
      //module.getChild就是拿到module的_children中key值对应的对象
      return module.getChild(key)
    }, this.root)
  }

  getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }

  update (rawRootModule) {
    update([], this.root, rawRootModule)
  }

  register (path, rawModule, runtime = true) {
    if (process.env.NODE_ENV !== 'production') {
      assertRawModule(path, rawModule)
    }
	//实例化一个Module对象
    const newModule = new Module(rawModule, runtime)
    //当path路径为空数组时，将newModule对象作为根的module，即root
    //当path不是空数组时，注册的是子的module
    if (path.length === 0) {
      //将根的module对象保留在ModuleCollection实例的root属性上
      this.root = newModule
    } else {
      //通过get方法拿到父的module对象
      const parent = this.get(path.slice(0, -1))
      //建立父子关系，例如本例中将根module的children的a属性值设置为moduleA对象
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    //当store的配置中有modules属性的时候
    if (rawModule.modules) {
      //forEachValue遍历rawModule.modules对象的每一个自定义属性，并对每个属性值执行
      //函数，rawChildModule为每个子module对象，key为rawModule.modules中的属性key，例如
      //例子中的a，b等
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        //对每个子的module执行register函数
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  unregister (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]
    if (!parent.getChild(key).runtime) return

    parent.removeChild(key)
  }
}
```

Module.js

```javascript
export default class Module {
  constructor (rawModule, runtime) {
    this.runtime = runtime
    // Store some children item
    //_children保存所有的子的module
    this._children = Object.create(null)
    // Store the origin module object which passed by programmer
    this._rawModule = rawModule
    const rawState = rawModule.state

    // Store the origin module's state
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }

  get namespaced () {
    return !!this._rawModule.namespaced
  }

  addChild (key, module) {
    this._children[key] = module
  }

  removeChild (key) {
    delete this._children[key]
  }

  getChild (key) {
    return this._children[key]
  }

  update (rawModule) {
    this._rawModule.namespaced = rawModule.namespaced
    if (rawModule.actions) {
      this._rawModule.actions = rawModule.actions
    }
    if (rawModule.mutations) {
      this._rawModule.mutations = rawModule.mutations
    }
    if (rawModule.getters) {
      this._rawModule.getters = rawModule.getters
    }
  }

  forEachChild (fn) {
    forEachValue(this._children, fn)
  }

  forEachGetter (fn) {
    if (this._rawModule.getters) {
      forEachValue(this._rawModule.getters, fn)
    }
  }

  forEachAction (fn) {
    if (this._rawModule.actions) {
      forEachValue(this._rawModule.actions, fn)
    }
  }

  forEachMutation (fn) {
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn)
    }
  }
}

```

#### 1.2.2 installModule

installModule函数的五个参数分别为store对象，根module的state对象，空数组，根module对象和undefined。installModule函数第一次调用是注册根module中的mutations、actions和getters等方法，在该函数的最后会递归调用注册子的module的mutations、actions和getters等方法。

```JavaScript
function installModule (store, rootState, path, module, hot) {
  //当path长度为0时，注册的是根module的方法
  var isRoot = !path.length;
  //通过getNamespace函数生成模块的命名空间，即一个字符串，moduleA的namespace就是'a/''
  var namespace = store._modules.getNamespace(path);

  // register in namespace map
  if (module.namespaced) {
    //将模块与其命名空间生成一种映射关系，Map
    store._modulesNamespaceMap[namespace] = module;
  }

  // set state
  if (!isRoot && !hot) {
    //拿到父的module的state对象
    var parentState = getNestedState(rootState, path.slice(0, -1));
    //拿到模块名称
    var moduleName = path[path.length - 1];
    store._withCommit(function () {
      //给父的module的state对象设置一个属性，属性key为子的模块名称
      //属性值为子模块的state对象
      //在这里构建state的父子关系结构
      Vue.set(parentState, moduleName, module.state);
    });
  }
  //makeLocalContext函数的作用是让每个模块内的dispatch, commit, getters and state私有化
  //互不干扰，如果没有命名空间就使用根的相应对象
  var local = module.context = makeLocalContext(store, namespace, path);
  
  //遍历module中mutations并注册
  module.forEachMutation(function (mutation, key) {
    //拼接namespace
    var namespacedType = namespace + key;
    registerMutation(store, namespacedType, mutation, local);
  });

  //遍历module中action并注册
  module.forEachAction(function (action, key) {
    //根据action中root的定义是否拼接namespace
    var type = action.root ? key : namespace + key;
    var handler = action.handler || action;
    registerAction(store, type, handler, local);
  });
  //遍历module中getter并注册
  module.forEachGetter(function (getter, key) {
    var namespacedType = namespace + key;
    registerGetter(store, namespacedType, getter, local);
  });
  //遍历子的模块，递归调用installModule进行注册
  module.forEachChild(function (child, key) {
    installModule(store, rootState, path.concat(key), child, hot);
  });
}

function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    //对于dispatch来说，如果没有命名空间，那么就使用根的dispatch
    //如果有命名空间，那么需要将namespace拼接，最总使用的是store.dispatch(type, payload)
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
          console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
          return
        }
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
          console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
          return
        }
      }

      store.commit(type, payload, options)
    }
  }

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  //对于getters而言，如果有命名空间，执行makeLocalGetters返回
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}

//注册mutation
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  //handler方法就是定义的mutation，故第一个参数能拿到local.state
  //当通过commit方式调用mutation执行的就是wrappedMutationHandler，可以传入
  //参数payload 
  entry.push(function wrappedMutationHandler (payload) {
    handler.call(store, local.state, payload)
  })
}

//注册action
function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload, cb) {
    //handler方法就是定义的action，故第一个参数context可以拿到很多方法和对象
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload, cb)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}

//注册getter
function registerGetter (store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] duplicate getter key: ${type}`)
    }
    return
  }
  store._wrappedGetters[type] = function wrappedGetter (store) {
    //rawGetter方法就是定义的getter
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
}
```

子模块的namespace为true的store对象如下图：

![](E:\开发文档\新建文件夹\learnVueDocs\others\store对象.png)

子模块的namespace为false的store对象如下图：

![](E:\开发文档\新建文件夹\learnVueDocs\others\namespace为false的store对象.png)

#### 1.2.3 resetStoreVM

```JavaScript
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm

  // bind store public getters
  //_wrappedGetters是一个私有属性，所以需要提供一个供用户使用的getters属性
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    //在computed对象中定义一个属性，属性key为wrappedGetters中的key，属性值为
    //wrappedGetter函数的返回值，即rawGetter函数，也就是getters中用户定义的
    //getters函数
    computed[key] = () => fn(store)
    //当我们通过vm.$store.getters访问getters的时候，返回的就是computed对象
    //中相应的值，因为computed对象是new Vue实例的computed对象
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }

  //销毁旧的vm
  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}

//当我们访问store.state的时候，访问的是store._vm._data.$$state
//这就是我们通过vm.$store.state可以访问module中的state的原因
get state () {
    return this._vm._data.$$state
}
```

## 二、API

### 2.1 state

vm.$store.state可以访问根module上的state，也可以访问所有模块的state，因为所有state都挂在根module上的state上。vm.$store.state的定义在store.js文件中。

```JavaScript
//vm.$store.state访问的是store上的_vm._data.$$state
get state () {
    return this._vm._data.$$state
}

set state (v) {
    if (process.env.NODE_ENV !== 'production') {
    	assert(false, `Use store.replaceState() to explicit replace store state.`)
    }
}
```

### 2.2 getters

vm.$store.getters可以访问到所有module上的getter属性。vm.$store.getters其实访问的是vm.$store. _vm（vm.$store. _vm 是一个vue实例）上相应的key的computed属性。

```JavaScript
const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
```

某个getter的四个参数分别为

```JavaScript
return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
```

在本文的实例中，可以通过store.getters["a/computedCount"]和store.getters.computedCount分别访问根module和子module中的getter。

![](E:\开发文档\新建文件夹\learnVueDocs\others\getters.png)



### 2.3 commit

触发mutation，mutation方法必须是同步的。

```javascript
commit (_type, _payload, _options) {
    // check object-style commit
    //对参数进行处理
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    //拿到_mutations相应的mutation，是一个数组
    const entry = this._mutations[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown mutation type: ${type}`)
      }
      return
    }
    //_withCommit确保里面的函数在执行的过程中_committing保持为true
    //因为_committing为false的时候，代表是外部操作改变state的值，是不允许的会报错
    this._withCommit(() => {
      //遍历数组中的每个函数，执行commitIterator
      entry.forEach(function commitIterator (handler) {
        //这个handler是registerMutation中的wrappedMutationHandler函数
        handler(payload)
      })
    })
    this._subscribers.forEach(sub => sub(mutation, this.state))

    if (
      process.env.NODE_ENV !== 'production' &&
      options && options.silent
    ) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
        'Use the filter functionality in the vue-devtools'
      )
    }
  }
```

### 2.4 dispatch

mutation中不能有异步操作。异步操作可以在action中完成。在action中的异步操作完成后，最终仍然调用的是commit方法改变state数据。当然我们也可以不在action中操作异步方法，而是自己定义异步方法，异步方法完成后调用commit方法改变state数据也是可以的。

```JavaScript
function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload, cb) {
    //handler对应的就是我们在store的actions配置中定义的函数
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload, cb)
    //如果handler函数的返回结果不是Promise对象，就调用Promise.resolve将其转换为Promise
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}

dispatch (_type, _payload) {
    // check object-style dispatch
    //对参数进行处理
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    //entry数组中的每个函数对应的就是wrappedActionHandler函数
    const entry = this._actions[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown action type: ${type}`)
      }
      return
    }

    this._actionSubscribers.forEach(sub => sub(action, this.state))
	//entry数组中的每个函数执行返回的是一个Promise对象
    return entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)
}
```

在子模块的action方法内部，commit的第一个参数是'increment'而不是'a/increment'，这是因为在makeLocalContext的commit方法中对namespace进行了拼接。

### 2.5 mapState

```vue
<template>
  <div id="app">
    <h1>hello app</h1>
    <div>root state count: {{count}}</div>
    <div>root getter computedCount: {{computedCount}}</div>
    <div>module a state count: {{aCount}}</div>
    <div>module a getter computedCount: {{aComputedCount}}</div>
  </div>
</template>

<script>
// import HelloWorld from "./components/HelloWorld";
import {mapState, mapGetters} from '../node_modules/vue/dist/vuex.esm.js'
export default {
  name: "App",
  computed: {
    ...mapState(['count']),
    ...mapGetters(['computedCount']),
    ...mapState('a', {
      aCount: 'count'
    }),
    ...mapGetters('a', {
      aComputedCount: 'computedCount'
    }),
  }
};
</script>
```

mapState是normalizeNamespace执行后返回的函数。

```javascript
export const mapState = normalizeNamespace((namespace, states) => {
  //mapState最后应该返回一个对象，然后用...解析，所以这里最后返回res
  //res的每个属性值是一个函数，因为computed的每个值也是函数
  const res = {}
  normalizeMap(states).forEach(({ key, val }) => {
    //当我们在例子中的组件中访问count的时候，执行的是这个mappedState函数
    res[key] = function mappedState () {
      let state = this.$store.state
      let getters = this.$store.getters
      //当有namespace的时候，例如本文实例中访问aCount
      if (namespace) {
        //首先拿到相应的子module
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        //通过context拿到state和getters
        state = module.context.state
        getters = module.context.getters
      }
      //当没有namespace的时候，例如本文实例中访问count，
      //返回的就是state[val],val为'count'
      return typeof val === 'function'
        ? val.call(this, state, getters)
        : state[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```



```JavaScript
//对namespace进行处理
function normalizeNamespace (fn) {
  //在vue组件中使用的mapState或者mapGetters是这个返回的这个箭头函数
  return (namespace, map) => {
    //可以不传namespace，不传时，第一个参数是map
    if (typeof namespace !== 'string') {
      map = namespace
      namespace = ''
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}

//map可能是数组或者对象，对其处理进行统一化
/**
 * Normalize the map
 * normalizeMap([1, 2, 3]) => [ { key: 1, val: 1 }, { key: 2, val: 2 }, { key: 3, val: 3 } ]
 * normalizeMap({a: 1, b: 2, c: 3}) => [ { key: 'a', val: 1 }, { key: 'b', val: 2 }, { key: 'c', val: 3 } ]
 * @param {Array|Object} map
 * @return {Object}
 */
function normalizeMap (map) {
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}

//在store的_modulesNamespaceMap对象中拿到子module并返回
function getModuleByNamespace (store, helper, namespace) {
  const module = store._modulesNamespaceMap[namespace]
  if (process.env.NODE_ENV !== 'production' && !module) {
    console.error(`[vuex] module namespace not found in ${helper}(): ${namespace}`)
  }
  return module
}
```

### 2.5 mapGetters

mapGetters的逻辑和mapState的类似。

```JavaScript
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  normalizeMap(getters).forEach(({ key, val }) => {
    // thie namespace has been mutate by normalizeNamespace
    //首先拼接namespace
    val = namespace + val
    res[key] = function mappedGetter () {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      if (process.env.NODE_ENV !== 'production' && !(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      //最终返回this.$store.getters中相应的getter值
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```

### 2.6 mapMutations和mapActions

mapMutations

```JavaScript
export const mapMutations = normalizeNamespace((namespace, mutations) => {
  const res = {}
  normalizeMap(mutations).forEach(({ key, val }) => {
    //最终vue的methods中的方法是mappedMutation，例如本文实例中执行increment方法就是
    //执行mappedMutation方法
    res[key] = function mappedMutation (...args) {
      // Get the commit method from store
      //拿到commit方法
      let commit = this.$store.commit
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapMutations', namespace)
        if (!module) {
          return
        }
        commit = module.context.commit
      }
      return typeof val === 'function'
        ? val.apply(this, [commit].concat(args))
        : commit.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```

mapActions

```JavaScript
export const mapActions = normalizeNamespace((namespace, actions) => {
  const res = {}
  normalizeMap(actions).forEach(({ key, val }) => {
    res[key] = function mappedAction (...args) {
      // get dispatch function from store
      let dispatch = this.$store.dispatch
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapActions', namespace)
        if (!module) {
          return
        }
        dispatch = module.context.dispatch
      }
      return typeof val === 'function'
        ? val.apply(this, [dispatch].concat(args))
        : dispatch.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```

### 2.7 registerModule

```JavaScript
registerModule (path, rawModule, options = {}) {
    if (typeof path === 'string') path = [path]

    if (process.env.NODE_ENV !== 'production') {
      assert(Array.isArray(path), `module path must be a string or an Array.`)
      assert(path.length > 0, 'cannot register the root module by using registerModule.')
    }
	//调用ModuleCollection上的公共方法register对ModuleCollection上的root属性（即
    //根module）进行扩展
    this._modules.register(path, rawModule)
    //对新模块的state、getters、mutations和actions进行处理
    installModule(this, this.state, path, this._modules.get(path), options.preserveState)
    // reset store to update getters...
    //更新响应式vm
    resetStoreVM(this, this.state)
  }
```

三、插件

在store.js中Store的构造函数中，

```JavaScript
const {
      plugins = [],
      strict = false
    } = options
plugins.forEach(plugin => plugin(this))
```

createLogger函数

```JavaScript
export default function createLogger ({
  collapsed = true,
  filter = (mutation, stateBefore, stateAfter) => true,
  transformer = state => state,
  mutationTransformer = mut => mut,
  logger = console
} = {}) {
  //createLogger函数返回该函数在Store的构造函数中执行
  return store => {
    //深拷贝
    let prevState = deepCopy(store.state)
	//store.subscribe订阅mutation的变化
    store.subscribe((mutation, state) => {
      //logger默认为console，用console.log输出日志
      if (typeof logger === 'undefined') {
        return
      }
      const nextState = deepCopy(state)

      //当filter执行返回true的时候才会执行后面的日志输出
      //filter默认是会返回true的，也可以自己进行定义哪些日志需要过滤掉
      if (filter(mutation, prevState, nextState)) {
        const time = new Date()
        const formattedTime = ` @ ${pad(time.getHours(), 2)}:${pad(time.getMinutes(), 2)}:${pad(time.getSeconds(), 2)}.${pad(time.getMilliseconds(), 3)}`
        //mutationTransformer是用来修改mutation的函数，默认是原样返回的
        const formattedMutation = mutationTransformer(mutation)
        const message = `mutation ${mutation.type}${formattedTime}`
        //设定输出的console.log日志是否折叠
        const startMessage = collapsed
          ? logger.groupCollapsed
          : logger.group

        // render
        try {
          startMessage.call(logger, message)
        } catch (e) {
          console.log(message)
        }
		//transformer修改state的函数，默认原样返回
        logger.log('%c prev state', 'color: #9E9E9E; font-weight: bold', transformer(prevState))
        logger.log('%c mutation', 'color: #03A9F4; font-weight: bold', formattedMutation)
        logger.log('%c next state', 'color: #4CAF50; font-weight: bold', transformer(nextState))

        try {
          logger.groupEnd()
        } catch (e) {
          logger.log('—— log end ——')
        }
      }

      prevState = nextState
    })
  }
}
//将fn添加到_subscribers中
//_subscribers在commit函数中，当改变了state之后会遍历执行，也就会执行
//上面定义在store.subscribe中的箭头函数
//_subscribers可以理解为订阅state变化的订阅者
subscribe (fn) {
    return genericSubscribe(fn, this._subscribers)
}

//将fn添加到subs中，并返回一个函数，该函数执行后可以将fn从subs中删除
function genericSubscribe (fn, subs) {
  if (subs.indexOf(fn) < 0) {
    subs.push(fn)
  }
  return () => {
    const i = subs.indexOf(fn)
    if (i > -1) {
      subs.splice(i, 1)
    }
  }
}
```

