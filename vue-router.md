vue-router源码的版本未v3.0.1。以下面的例子来说明vue-router的源码。

```javascript
import Vue from '../node_modules/vue/dist/vue.esm'
import VueRouter from 'vue-router'

Vue.use(VueRouter)

const Foo = {
    template: '<div>foo</div>'
}

const Bar = {
    template: '<div>bar</div>'
}

const routes = [
    {path: '/foo', component: Foo},
    {path: '/bar', component: Bar},
]

const router = new VueRouter({
    routes
})

new Vue({
    el: '#app',
    render(h) {
        return h(App)
    },
    router
})
```

```
<template>
  <div id="app">
    <h1>hello app</h1>
    <p>
      <router-link to="/foo">go to foo</router-link>
      <router-link to="/bar">go to bar</router-link>
    </p>
    <router-view></router-view>
  </div>
</template>

<script>
// import HelloWorld from "./components/HelloWorld";
export default {
  name: "App",
};
</script>
```

vue-router的入口是src下的index.js。index.js中定义了VueRouter类，当通过import VueRouter引入的就是这个类。

## 一、路由注册

先看一下Vue.use的定义。定义在core/global-api/use.js中。

```JavaScript
Vue.use = function (plugin: Function | Object) {
    //保存所有注册到Vue上的插件
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    //判断插件是否已经注册过
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    //拿到除plugin外的所有参数
    const args = toArray(arguments, 1)
    //将Vue设为第一个参数
    args.unshift(this)
    //如果plugin是一个对象且又install方法，回去执行该方法。
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
}
```

VueRouter这个类的install方法如下：

```JavaScript
export function install (Vue) {
  //当多次注册的时候直接return
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  //通过mixin将两个钩子函数混入到所有的vue组件中
  Vue.mixin({
    beforeCreate () {
      //在new Vue的时候传入了router对象
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        //让每一个组件实例的_routerRoot指向根vue实例的_routerRoot对象
        //因为beforeCreate钩子的执行是先父后子，所以所有子组件都能拿到
        //根实例的_routerRoot对象
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  //注册router-view和router-link两个组件
  Vue.component('router-view', View)
  Vue.component('router-link', Link)

  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}

```

## 二、vueRouter对象

vueRouter类的构造函数如下。当实例化vueRouter的时候会创建一些变量。vueRouter的初始化是在上面的beforeCreate钩子中进行的。

```JavaScript
constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    //history路由模式是用html5的history.pushState来实现的，所以需要判断
    //浏览器是否支持
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
}

export const supportsPushState = inBrowser && (function () {
  const ua = window.navigator.userAgent

  if (
    (ua.indexOf('Android 2.') !== -1 || ua.indexOf('Android 4.0') !== -1) &&
    ua.indexOf('Mobile Safari') !== -1 &&
    ua.indexOf('Chrome') === -1 &&
    ua.indexOf('Windows Phone') === -1
  ) {
    return false
  }

  return window.history && 'pushState' in window.history
})()
```

init函数在执行的时候，将vue的实例作为参数传入的。

```JavaScript
init (app: any /* Vue component instance */) {
    process.env.NODE_ENV !== 'production' && assert(
      install.installed,
      `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
      `before creating root instance.`
    )

    this.apps.push(app)

    // main app already initialized.
    //vueRouter是支持多哥vueRouter实例的，但是init函数中这下面的逻辑只能执行一次
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      //transitionTo方法定义在History的类中，起路径切换的作用
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
}

transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const route = this.router.match(location, this.current)
    this.confirmTransition(route, () => {
      this.updateRoute(route)
      onComplete && onComplete(route)
      this.ensureURL()

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => { cb(route) })
      }
    }, err => {
      if (onAbort) {
        onAbort(err)
      }
      if (err && !this.ready) {
        this.ready = true
        this.readyErrorCbs.forEach(cb => { cb(err) })
      }
    })
}
```

## 三、matcher

在初始化VueRouter类调用构造函数的时候，会调用createMatcher函数生成matcher对象。生成的matcher对象包含两个函数match和addRoutes

```JavaScript
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  //调用createRouteMap方法生成routes中所有的path，并且生成path与对应的路由记录record
  //对象的map映射关系，和name与对应的路由记录record的map映射关系
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  //当动态添加route的时候，调用createRouteMap生成相应的路由record并生成映射关系
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }

  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }

      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      if (record) {
        location.path = fillParams(record.path, location.params, `named route "${name}"`)
        return _createRoute(record, location, redirectedFrom)
      }
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        //如果location.path匹配上了record.regex这个正则，调用_createRoute函数
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }

  function redirect (
    record: RouteRecord,
    location: Location
  ): Route {
    const originalRedirect = record.redirect
    let redirect = typeof originalRedirect === 'function'
        ? originalRedirect(createRoute(record, location, null, router))
        : originalRedirect

    if (typeof redirect === 'string') {
      redirect = { path: redirect }
    }

    if (!redirect || typeof redirect !== 'object') {
      if (process.env.NODE_ENV !== 'production') {
        warn(
          false, `invalid redirect option: ${JSON.stringify(redirect)}`
        )
      }
      return _createRoute(null, location)
    }

    const re: Object = redirect
    const { name, path } = re
    let { query, hash, params } = location
    query = re.hasOwnProperty('query') ? re.query : query
    hash = re.hasOwnProperty('hash') ? re.hash : hash
    params = re.hasOwnProperty('params') ? re.params : params

    if (name) {
      // resolved named direct
      const targetRecord = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        assert(targetRecord, `redirect failed: named route "${name}" not found.`)
      }
      return match({
        _normalized: true,
        name,
        query,
        hash,
        params
      }, undefined, location)
    } else if (path) {
      // 1. resolve relative redirect
      const rawPath = resolveRecordPath(path, record)
      // 2. resolve params
      const resolvedPath = fillParams(rawPath, params, `redirect route with path "${rawPath}"`)
      // 3. rematch with existing query and hash
      return match({
        _normalized: true,
        path: resolvedPath,
        query,
        hash
      }, undefined, location)
    } else {
      if (process.env.NODE_ENV !== 'production') {
        warn(false, `invalid redirect option: ${JSON.stringify(redirect)}`)
      }
      return _createRoute(null, location)
    }
  }

  function alias (
    record: RouteRecord,
    location: Location,
    matchAs: string
  ): Route {
    const aliasedPath = fillParams(matchAs, location.params, `aliased route with path "${matchAs}"`)
    const aliasedMatch = match({
      _normalized: true,
      path: aliasedPath
    })
    if (aliasedMatch) {
      const matched = aliasedMatch.matched
      const aliasedRecord = matched[matched.length - 1]
      location.params = aliasedMatch.params
      return _createRoute(aliasedRecord, location)
    }
    return _createRoute(null, location)
  }

  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    return createRoute(record, location, redirectedFrom, router)
  }

  return {
    match,
    addRoutes
  }
}
```

createMatcher函数中首先会调用createRouteMap生成三个值，pathList, pathMap, nameMap。该函数中会遍历routes中的每个对象并调用addRouteRecord函数。

```JavaScript
export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>
): {
  pathList: Array<string>;
  pathMap: Dictionary<RouteRecord>;
  nameMap: Dictionary<RouteRecord>;
} {
  // the path list is used to control path matching priority
  const pathList: Array<string> = oldPathList || []
  // $flow-disable-line
  const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
  // $flow-disable-line
  const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)

  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })

  // ensure wildcard routes are always at the end
  //当pathList中有通配符*的时候，将其放在pathList数组最后
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }

  return {
    pathList,
    pathMap,
    nameMap
  }
}

//addRouteRecord对象对每个route生成一个record对象，并且根据
//path和name生成映射关系，方便以后可以根据path或者name找到
//对应的record
function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  const { path, name } = route
  if (process.env.NODE_ENV !== 'production') {
    assert(path != null, `"path" is required in a route configuration.`)
    assert(
      typeof route.component !== 'string',
      `route config "component" for path: ${String(path || name)} cannot be a ` +
      `string id. Use an actual component instead.`
    )
  }

  const pathToRegexpOptions: PathToRegexpOptions = route.pathToRegexpOptions || {}
  //生成一个路径
  const normalizedPath = normalizePath(
    path,
    parent,
    pathToRegexpOptions.strict
  )

  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }
  
  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props: route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props }
  }

  if (route.children) {
    // Warn if route is named, does not redirect and has a default child route.
    // If users navigate to this route by name, the default child will
    // not be rendered (GH Issue #629)
    if (process.env.NODE_ENV !== 'production') {
      if (route.name && !route.redirect && route.children.some(child => /^\/?$/.test(child.path))) {
        warn(
          false,
          `Named Route '${route.name}' has a default child route. ` +
          `When navigating to this named route (:to="{name: '${route.name}'"), ` +
          `the default child route will not be rendered. Remove the name from ` +
          `this route and use the name of the default child route for named ` +
          `links instead.`
        )
      }
    }
    //遍历route对象的children中的每个值，对其递归调用addRouteRecord
    //传入的参数record即父的record
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }

  //alias不是主线逻辑，可不看
  if (route.alias !== undefined) {
    ......
  }
  //根路由route调用addRouteRecord时传入的pathList是一个空数组
  //每一个路由route都会生成一个对应的record对象
  //pathList用来记录record对象的path属性
  //pathMap记录path对应的record
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }

  //如果route对象中存在name属性，那么nameMap对象中会记录name对应的record对象
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      warn(
        false,
        `Duplicate named routes definition: ` +
        `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}

function normalizePath (path: string, parent?: RouteRecord, strict?: boolean): string {
  //将path的结尾的/替换为空字符串，即删除
  if (!strict) path = path.replace(/\/$/, '')
  if (path[0] === '/') return path
  if (parent == null) return path
  //cleanPath会将路径中连续的两个/替换成一个/，避免合并过程中意外出现的错误
  return cleanPath(`${parent.path}/${path}`)
}
```

在执行router对象的init函数的时候，会执行router对象的history对象的transitionTo方法。该方法中又会执行router对象的match函数，router对象的match函数其实就是调用matcher对象中的match函数。matcher对象中的match函数的前两个参数分别为将要前往的location位置（即浏览器地址栏中的地址），当前的route对象（即当前的路由route）,该函数会返回一个新的路径。matcher对象中的match函数中，首先会调用normalizeLocation函数。normalizeLocation 方法的作用是根据 `raw`，`current` 计算出新的 location。

计算出新的 `location` 后，对 `location` 的 `name` 和 `path` 的两种情况做了处理。

- `name`

有 `name` 的情况下就根据 `nameMap` 匹配到 `record`，它就是一个 `RouterRecord` 对象，如果 `record` 不存在，则匹配失败，返回一个空路径；然后拿到 `record` 对应的 `paramNames`，再对比 `currentRoute` 中的 `params`，把交集部分的 `params` 添加到 `location` 中，然后在通过 `fillParams` 方法根据 `record.path` 和 `location.path` 计算出 `location.path`，最后调用 `_createRoute(record, location, redirectedFrom)` 去生成一条新路径，该方法我们之后会介绍。

- `path`

通过 `name` 我们可以很快的找到 `record`，但是通过 `path` 并不能，因为我们计算后的 `location.path` 是一个真实路径，而 `record` 中的 `path` 可能会有 `param`，因此需要对所有的 `pathList` 做顺序遍历， 然后通过 `matchRoute` 方法根据 `record.regex`、`location.path`、`location.params` 匹配，如果匹配到则也通过 `_createRoute(record, location, redirectedFrom)` 去生成一条新路径。因为是顺序遍历，所以我们书写路由配置要注意路径的顺序，因为写在前面的会优先尝试匹配。

```JavaScript
export function normalizeLocation (
  raw: RawLocation,
  current: ?Route,
  append: ?boolean,
  router: ?VueRouter
): Location {
  let next: Location = typeof raw === 'string' ? { path: raw } : raw
  // named target
  if (next.name || next._normalized) {
    return next
  }

  // relative params
  //处理一些参数
  if (!next.path && next.params && current) {
    next = assign({}, next)
    next._normalized = true
    const params: any = assign(assign({}, current.params), next.params)
    if (current.name) {
      next.name = current.name
      next.params = params
    } else if (current.matched.length) {
      const rawPath = current.matched[current.matched.length - 1].path
      next.path = fillParams(rawPath, params, `path ${current.path}`)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(false, `relative params navigation requires a current route.`)
    }
    return next
  }
  //调用parsePath解析当前路径中的path，query和hash值
  const parsedPath = parsePath(next.path || '')
  const basePath = (current && current.path) || '/'
  const path = parsedPath.path
    ? resolvePath(parsedPath.path, basePath, append || next.append)
    : basePath

  const query = resolveQuery(
    parsedPath.query,
    next.query,
    router && router.options.parseQuery
  )

  let hash = next.hash || parsedPath.hash
  if (hash && hash.charAt(0) !== '#') {
    hash = `#${hash}`
  }

  return {
    _normalized: true,
    path,
    query,
    hash
  }
}

//解析路径中的path，query和hash值，并返回这些值组成的对象
export function parsePath (path: string): {
  path: string;
  query: string;
  hash: string;
} {
  let hash = ''
  let query = ''

  const hashIndex = path.indexOf('#')
  if (hashIndex >= 0) {
    hash = path.slice(hashIndex)
    path = path.slice(0, hashIndex)
  }

  const queryIndex = path.indexOf('?')
  if (queryIndex >= 0) {
    query = path.slice(queryIndex + 1)
    path = path.slice(0, queryIndex)
  }

  return {
    path,
    query,
    hash
  }
}
```

最后我们来看一下 `_createRoute` 的实现：

```js
function _createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: Location
): Route {
  if (record && record.redirect) {
    return redirect(record, redirectedFrom || location)
  }
  if (record && record.matchAs) {
    return alias(record, location, record.matchAs)
  }
  return createRoute(record, location, redirectedFrom, router)
}
```

我们先不考虑 `record.redirect` 和 `record.matchAs` 的情况，最终会调用 `createRoute` 方法，它的定义在 `src/uitl/route.js` 中：

```js
export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  const stringifyQuery = router && router.options.stringifyQuery

  let query: any = location.query || {}
  try {
    query = clone(query)
  } catch (e) {}

  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location, stringifyQuery),
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  return Object.freeze(route)
}
```

`createRoute` 可以根据 `record` 和 `location` 创建出来，最终返回的是一条 `Route` 路径，我们之前也介绍过它的数据结构。在 Vue-Router 中，所有的 `Route` 最终都会通过 `createRoute` 函数创建，并且它最后是不可以被外部修改的。`Route` 对象中有一个非常重要属性是 `matched`，它通过 `formatMatch(record)` 计算而来：

```js
function formatMatch (record: ?RouteRecord): Array<RouteRecord> {
  const res = []
  while (record) {
    res.unshift(record)
    record = record.parent
  }
  return res
}
```

可以看它是通过 `record` 循环向上找 `parent`，直到找到最外层，并把所有的 `record` 都 push 到一个数组中，最终返回的就是 `record` 的数组，它记录了一条线路上的所有 `record`。`matched` 属性非常有用，它为之后渲染组件提供了依据。

## 四、路由守卫钩子执行逻辑

路由守卫钩子函数可能是同步，也可能是异步的（在其中调用了异步方法）。如何保证它们按照一定的顺序执行呢？

```JavaScript
//第三个参数cb函数是所有路由钩子全部执行完成的回调
function runQueue (queue, fn, cb) {
  const step = index => {
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  step(0)
}

const iterator = (hook, next) => {
        hook((to) => {
          if (to === false) {
            //停止向后继续执行路由钩子函数
          } else {
            next(to)
          }
        })   
}

//queue是要执行的函数数组
runQueue(queue, iterator, () => {
    console.log("all done")
})
```

可以用下面的例子进行验证。

```JavaScript
function fn1(next) {
  console.log("fn1");
  next();
}
function fn2(next) {
  console.log("fn2");
  next(false);
}
function fn3(next) {
  console.log("fn3");
  next()
}
const queue = [fn1, fn2, fn3];
```

## 五、url

### 5.1 hash模式

当点击route-link标签进行路由跳转的时候，浏览器的地址栏的地址是怎么切换的呢？

当点击route-link标签进行路由跳转的时候，执行的是router的push方法。该方法中又掉用了history对象的push方法。

```JavaScript
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.push(location, onComplete, onAbort)
}
```

hash.js的文件中

```JavaScript
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this
    //当切换视图成功时，调用回调函数
    //回调函数中执行pushHash
    this.transitionTo(location, route => {
      pushHash(route.fullPath)
      handleScroll(this.router, route, fromRoute, false)
      onComplete && onComplete(route)
    }, onAbort)
}

//pushState的情况下执行if中的逻辑
function pushHash (path) {
  if (supportsPushState) {
    //getUrl拼接路径
    pushState(getUrl(path))
  } else {
    window.location.hash = path
  }
}
//getUrl返回要进入的地址的完整url
//例如原来的url是 http://www.runoob.com/test.htm＃PART2,
//path为/aaa
//则返回的路径是http://www.runoob.com/test.htm＃/aaa
function getUrl (path) {
  //window.location.href返回浏览器url中的完整的路径
  const href = window.location.href
  const i = href.indexOf('#')
  const base = i >= 0 ? href.slice(0, i) : href
  return `${base}#${path}`
}

export function pushState (url?: string, replace?: boolean) {
  //与滚动有关，可不看
  saveScrollPosition()
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  const history = window.history
  try {
    if (replace) {
      //url替换，不计入历史栈
      //第一个参数是一个对象，表示状态，第二个参数是title
      history.replaceState({ key: _key }, '', url)
    } else {
      //记入历史栈
      //第一个参数是一个对象，表示状态，第二个参数是title
      _key = genKey()
      history.pushState({ key: _key }, '', url)
    }
  } catch (e) {
    window.location[replace ? 'replace' : 'assign'](url)
  }
}
```

当点击浏览器回退或者前进，或者调用history.go()等方法，触发了浏览器url改变时，会触发页面重新渲染。因为vue-router代码中做了这样的处理。router类的init方法中会给window添加popstate或者hashchange事件。当浏览器url改变时，会触发切换视图。replaceState和pushState不触发popstate事件。

```JavaScript
if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      //当初始化的时候切换视图，会调用setupHashListener
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
}

setupListeners () {
    const router = this.router
    const expectScroll = router.options.scrollBehavior
    const supportsScroll = supportsPushState && expectScroll

    if (supportsScroll) {
      setupScroll()
    }
	//当支持pushState的时候，监听popstate事件，否则监听hashChange事件
    window.addEventListener(supportsPushState ? 'popstate' : 'hashchange', () => {
      const current = this.current
      if (!ensureSlash()) {
        return
      }
      this.transitionTo(getHash(), route => {
        if (supportsScroll) {
          handleScroll(this.router, route, current, true)
        }
        if (!supportsPushState) {
          replaceHash(route.fullPath)
        }
      })
    })
}

export function getHash (): string {
  // We can't use window.location.hash here because it's not
  // consistent across browsers - Firefox will pre-decode it!
  const href = window.location.href
  const index = href.indexOf('#')
  return index === -1 ? '' : href.slice(index + 1)
}
```

当hash模式下，是怎么自动给浏览器url中加上#号的。因为HashHistory的构造函数中执行了ensureSlash函数。

```JavaScript
function ensureSlash (): boolean {
  //当页面初始化，浏览器url为http://localhost:8081/的时候，
  //getHash函数返回空字符串，执行replaceHash
  const path = getHash()
  if (path.charAt(0) === '/') {
    return true
  }
  replaceHash('/' + path)
  return false
}

function replaceHash (path) {
  if (supportsPushState) {
    //getUrl返回要进入的地址的完整url
    replaceState(getUrl(path))
  } else {
    window.location.replace(getUrl(path))
  }
}

export function replaceState (url?: string) {
  pushState(url, true)
}
```



### 5.2 history模式

当HTML5History执行构造函数的时候，构造函数中监听了popstate事件

```JavaScript
window.addEventListener('popstate', e => {
      const current = this.current

      // Avoiding first `popstate` event dispatched in some browsers but first
      // history route not updated since async guard at the same time.
      //base是在HTML5History构造函数中赋值的，基础路径，是一个空字符串
      const location = getLocation(this.base)
      if (this.current === START && location === initLocation) {
        return
      }
      //切换视图
      this.transitionTo(location, route => {
        if (expectScroll) {
          handleScroll(router, route, current, true)
        }
      })
})

export function getLocation (base: string): string {
  let path = window.location.pathname
  if (base && path.indexOf(base) === 0) {
    path = path.slice(base.length)
  }
  return (path || '/') + window.location.search + window.location.hash
}
```

当点击route-link标签进行路由跳转的时候，执行的是router的push方法。该方法中又掉用了history对象的push方法。

```JavaScript
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this
    //当视图切换成功后，调用pushState方法
    this.transitionTo(location, route => {
      //这个pushState和hash模式中的那个pushState是同一个函数
      pushState(cleanPath(this.base + route.fullPath))
      handleScroll(this.router, route, fromRoute, false)
      onComplete && onComplete(route)
    }, onAbort)
}

//将'//’替换为 '/',防止出现两个/连在一起的情况
export function cleanPath (path: string): string {
  return path.replace(/\/\//g, '/')
}
```

### 5.3 总结

hash模式下，当支持history.pushState方法的时候，url切换调用的是history.pushState或者history.replaceState完成的；当不支持history.pushState方法的时候，url切换是通过直接修改window.location.hash完成的。当通过history.go()等方法或者浏览器前进后退按钮可以实现页面的切换，是通过监听popstate或者hashchange事件来实现的（当浏览器支持popstate的时候，监听的就是popstate，不支持就监听hashchange事件）。

history模式下，url切换调用的是history.pushState或者history.replaceState完成的。当通过history.go()等方法或者浏览器前进后退按钮可以实现页面的切换，是通过监听popstate事件来实现的。

## 六、视图渲染

配合下面的实例来讲解视图渲染的源码

```vue
/* eslint-disable */
import Vue from '../node_modules/vue/dist/vue.esm'
import VueRouter from '../node_modules/vue/dist/vue-router.esm.js'
import App from './App.vue'

Vue.use(VueRouter)

const Foo = {
    template: '<div><div>foo</div>' + 
     '<router-link to="/foo/bar">go to foo.bar</router-link>' + 
     '<router-view></router-view>' + '</div>',
}

const Bar = {
    template: '<div>bar</div>',
}

const routes = [
    {
        path: '/foo', 
        component: Foo,
        children: [
            {
                path: 'bar',
                component: Bar,
            }
        ]
    },
    {path: '/bar', component: Bar},
]

const router = new VueRouter({
    routes,
    // mode: 'history',
})

new Vue({
    el: '#app',
    render(h) {
        return h(App)
    },
    router
})
```

### 6.1 router-view组件

router-view组件的定义如下。这是一个函数式组件，其render函数中可以接收第二个参数。

```JavaScript
export default {
  name: 'router-view',
  functional: true,
  props: {
    name: {
      type: String,
      default: 'default'
    }
  },
  render (_, { props, children, parent, data }) {
    //标志是router-view组件
    data.routerView = true

    // directly use parent context's createElement() function
    // so that components rendered by router-view can resolve named slots
    //parent指的是父实例。例如App.vue中的router-view的parent就是vue实例。Foo中的router-view的parent就是Foo的实例
    const h = parent.$createElement
    //拿到router-view的props中的name属性
    const name = props.name
    //通过父实例拿到$route。因为函数式组件没有实例
    //在路由插件被注册的时候，在Vue.prototype上定义了$route，返回的是
    //history.current，即当前的路由route
    const route = parent.$route
    const cache = parent._routerViewCache || (parent._routerViewCache = {})

    // determine current view depth, also check to see if the tree
    // has been toggled inactive but kept-alive.
    let depth = 0
    let inactive = false
    //计算router-view组件的层级，根据层级可以匹配每个router-view组件该渲染成哪个路由对应的组件
    while (parent && parent._routerRoot !== parent) {
      //当父实例是router-view，那么它的占位符vnode的data.routerView
      //为true，此时将该router-view组件的深度加1
      //如此通过不停地往上查找父实例来计算最终的深度
      if (parent.$vnode && parent.$vnode.data.routerView) {
        depth++
      }
      if (parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }
    data.routerViewDepth = depth

    // render previous view if the tree is inactive and kept-alive
    if (inactive) {
      return h(cache[name], data, children)
    }

    //路由route的matched属性是一个数组，记录了当前路由所匹配到的所有的组件，最外层的
    //组件位于数组首位，通过深度值depth可以拿到当前router-view应该渲染成哪个组件。
    //例如在本文的例子中，当路径是/foo/bar时,App.vue中的router-view的深度为0，Foo组件
    //中的router-view的深度为1，那么App.vue中的router-view渲染成foo组件，Foo组件
    //中的router-view渲染成bar组件
    //route.matched数组中的值都是record对象，其components[name]的值对应的才是组件
    const matched = route.matched[depth]
    // render empty node if no matched route
    if (!matched) {
      cache[name] = null
      return h()
    }

    const component = cache[name] = matched.components[name]

    // attach instance registration hook
    // this will be called in the instance's injected lifecycle hooks
    //该函数用于给路由钩子函数绑定vm实例
    data.registerRouteInstance = (vm, val) => {
      // val could be undefined for unregistration
      const current = matched.instances[name]
      if (
        (val && current !== vm) ||
        (!val && current === vm)
      ) {
        matched.instances[name] = val
      }
    }

    // also register instance in prepatch hook
    // in case the same component instance is reused across different routes
    ;(data.hook || (data.hook = {})).prepatch = (_, vnode) => {
      matched.instances[name] = vnode.componentInstance
    }

    // resolve props
    let propsToPass = data.props = resolveProps(route, matched.props && matched.props[name])
    if (propsToPass) {
      // clone to prevent mutation
      propsToPass = data.props = extend({}, propsToPass)
      // pass non-declared props as attrs
      const attrs = data.attrs = data.attrs || {}
      for (const key in propsToPass) {
        if (!component.props || !(key in component.props)) {
          attrs[key] = propsToPass[key]
          delete propsToPass[key]
        }
      }
    }
	
    //最后router-view的vnode的值渲染成component的vnode
    return h(component, data, children)
  }
}

function resolveProps (route, config) {
  switch (typeof config) {
    case 'undefined':
      return
    case 'object':
      return config
    case 'function':
      return config(route)
    case 'boolean':
      return config ? route.params : undefined
    default:
      if (process.env.NODE_ENV !== 'production') {
        warn(
          false,
          `props in "${route.path}" is a ${typeof config}, ` +
          `expecting an object, function or boolean.`
        )
      }
  }
}

function extend (to, from) {
  for (const key in from) {
    to[key] = from[key]
  }
  return to
}

```



```JavaScript
Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })
Vue.util.defineReactive(this, '_route', this._router.history.current)
```

在路由的守卫钩子函数中，vm实例是由routeRecord对象中的instances属性决定的。当Foo组件实例化执行beforeCreate钩子的时候，调用了registerInstance函数，将Foo的组件实例vm作为两个参数。registerInstance函数中调用router-view中的data.registerRouteInstance函数。registerRouteInstance函数中给routeRecord对象的instances赋值。

```JavaScript
const registerInstance = (vm, callVal) => {
    //Foo.vue的$options._parentVnode是Foo的占位符vnode
    //Foo的占位符vnode的data中有registerRouteInstance
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })
```

当页面初始化渲染的时候，在App.vue进行渲染过程中，首先进行 _render生成vnode(App.vue的render函数如下），在 _render的过程中，执行 _c("router-view")生成vnode，即调用 _createElement生成vnode。在 _createElement函数中，因为router-view是一个组件，所以执行createComponent生成vnode。在createComponent函数中，因为router-view是一个函数式组件，所以执行createFunctionalComponent函数（代码下面）。在该函数中，会去执行router-view组件中定义的render函数。在render函数中，计算出App.vue中的router-view的深度为0，但是此时的路径为'/'，因此route.matched为空数组，因此router-view渲染成为一个注释节点。

当点击’go to foo'的时候，页面重新渲染。此时router-view匹配到route.matched数组中的第一个值，即path为‘/foo'的routeRecord对象，最终router-view的render函数返回的vnode是h(component, data, children)的值，其中component是foo组件的定义。当Foo组件渲染执行beforeCreate钩子的时候，执行registerInstance函数，将Foo组件实例vm赋值给path为’/foo'的routeRecord对象的instances属性。

```JavaScript
function() {
  var _vm = this
  var _h = _vm.$createElement
  var _c = _vm._self._c || _h
  return _c(
    "div",
    { attrs: { id: "app" } },
    [
      _c("h1", [_vm._v("hello app")]),
      _vm._v(" "),
      _c(
        "p",
        [
          _c("router-link", { attrs: { to: "/foo" } }, [_vm._v("go to foo")]),
          _vm._v(" "),
          _c("router-link", { attrs: { to: "/bar" } }, [_vm._v("go to bar")])
        ],
        1
      ),
      _vm._v(" "),
      _c("router-view")
    ],
    1
  )
}

if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
}
```

接下来Foo组件在渲染的过程中首先会生成vnode，在render生成vnode的过程中，遇到了Foo中的router-view组件。挡在计算深度的时候，while第一次循环时，parent.$vnode为Foo组件的占位符vnode，它的data.routerView属性为true，因此该router-view组件的深度为1。但是因为此时path属性为’/foo‘的routeRecord对象的matched数组长度为1，所以Foo中的router-view组件最终渲染成注释节点。

```JavaScript
function anonymous(
) {
with(this){return _c('div',[_c('div',[_v("foo")]),_c('router-link',{attrs:{"to":"/foo/bar"}},[_v("go to foo.bar")]),_c('router-view')],1)}
}
```

那路由切换时为什么会导致页面重新渲染呢？

那是因为我们在new vue实例执行beforeCreate钩子的时候，将其 _route属性设定为响应式的。当我们在router-view组件中访问parent.$route的时候，就访问到了new vue实例的 _route属性，那么就会去收集依赖，即渲染watcher。

在VueRouter的init方法中（如下面代码所示），当路由发生改变时，会触发回调函数执行，回调函数中会给new vue实例的 _route属性重新赋值，触发watcher重新渲染。

```JavaScript
beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        //将new vue实例的_route属性设定为响应式的
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
```

```
history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
```

### 6.2 router-link

router-link最终会被渲染成一个a标签.源码如下.

```javascript
export default {
  name: 'router-link',
  props: {
    to: {
      type: toTypes,
      required: true
    },
    //router-link可以渲染成任何标签,默认为a标签
    tag: {
      type: String,
      default: 'a'
    },
    exact: Boolean,
    append: Boolean,
    replace: Boolean,
    activeClass: String,
    exactActiveClass: String,
    event: {
      type: eventTypes,
      default: 'click'
    }
  },
  render (h: Function) {
    //当router-link组件渲染时，首先_render生成vnode, _render会执行这个render函数
    //这个render函数执行时，执行环境被绑定成了router-link的实例
    //又因为Vue.prototype上绑定了$router和$route，因此这里可以拿到
    const router = this.$router
    const current = this.$route
    //调用router对象的resolve方法,计算router-link组件的to属性想要跳转到的路径等信息
    const { location, route, href } = router.resolve(this.to, current, this.append)

    const classes = {}
    const globalActiveClass = router.options.linkActiveClass
    const globalExactActiveClass = router.options.linkExactActiveClass
    // Support global empty active class
    const activeClassFallback = globalActiveClass == null
            ? 'router-link-active'
            : globalActiveClass
    const exactActiveClassFallback = globalExactActiveClass == null
            ? 'router-link-exact-active'
            : globalExactActiveClass
    const activeClass = this.activeClass == null
            ? activeClassFallback
            : this.activeClass
    const exactActiveClass = this.exactActiveClass == null
            ? exactActiveClassFallback
            : this.exactActiveClass
    const compareTarget = location.path
      ? createRoute(null, location, null, router)
      : route
	//判断当前路由是否完全匹配router-link的to属性，是的话给加上类名exactActiveClass
    classes[exactActiveClass] = isSameRoute(current, compareTarget)
    //判断当前路由是否部分匹配router-link的to属性，是的话给加上类名activeClass
    //例如，当前路由为'/foo/bar',而to属性为'/foo'，那么就是部分匹配
    classes[activeClass] = this.exact
      ? classes[exactActiveClass]
      : isIncludedRoute(current, compareTarget)

    const handler = e => {
      if (guardEvent(e)) {
        if (this.replace) {
          router.replace(location)
        } else {
          router.push(location)
        }
      }
    }
	//给event属性添加处理函数
    const on = { click: guardEvent }
    if (Array.isArray(this.event)) {
      this.event.forEach(e => { on[e] = handler })
    } else {
      on[this.event] = handler
    }

    const data: any = {
      class: classes
    }

    if (this.tag === 'a') {
      data.on = on
      data.attrs = { href }
    } else {
      //查找slots中是否有a标签
      // find the first <a> child and apply listener and href
      const a = findAnchor(this.$slots.default)
      //有a标签的话,将on和href属性赋给a标签
      if (a) {
        // in case the <a> is a static node
        a.isStatic = false
        const extend = _Vue.util.extend
        const aData = a.data = extend({}, a.data)
        aData.on = on
        const aAttrs = a.data.attrs = extend({}, a.data.attrs)
        aAttrs.href = href
      } else {
        // doesn't have <a> child, apply listener to self
        data.on = on
      }
    }

    return h(this.tag, data, this.$slots.default)
  }
}

function guardEvent (e) {
  // don't redirect with control keys
  if (e.metaKey || e.altKey || e.ctrlKey || e.shiftKey) return
  // don't redirect when preventDefault called
  if (e.defaultPrevented) return
  // don't redirect on right click
  if (e.button !== undefined && e.button !== 0) return
  // don't redirect if `target="_blank"`
  if (e.currentTarget && e.currentTarget.getAttribute) {
    const target = e.currentTarget.getAttribute('target')
    if (/\b_blank\b/i.test(target)) return
  }
  // this may be a Weex event which doesn't have this method
  if (e.preventDefault) {
    e.preventDefault()
  }
  return true
}

function findAnchor (children) {
  if (children) {
    let child
    for (let i = 0; i < children.length; i++) {
      child = children[i]
      if (child.tag === 'a') {
        return child
      }
      if (child.children && (child = findAnchor(child.children))) {
        return child
      }
    }
  }
}

resolve (
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
    location: Location,
    route: Route,
    href: string,
    // for backwards compat
    normalizedTo: Location,
    resolved: Route
  } {
    //normalizeLocation生成要切换的路径
    const location = normalizeLocation(
      to,
      current || this.history.current,
      append,
      this
    )
    //生成要切换的route对象
    const route = this.match(location, current)
    const fullPath = route.redirectedFrom || route.fullPath
    const base = this.history.base
    //根据根路径和相对路径fullPath计算最终的完整路径
    const href = createHref(base, fullPath, this.mode)
    return {
      location,
      route,
      href,
      // for backwards compat
      normalizedTo: location,
      resolved: route
    }
}

function createHref (base: string, fullPath: string, mode) {
  var path = mode === 'hash' ? '#' + fullPath : fullPath
  return base ? cleanPath(base + '/' + path) : path
}

```

router-link标签在渲染的过程和router-view不同，因为router-view是函数式组件。当App.vue在渲染时，router-link执行_render函数生成vnode（不会执行router-link的render），当router-link在渲染时，会去执行init生成vue实例，然后执行router-link文件中定义的render函数生成vnode，最后patch到页面上。

