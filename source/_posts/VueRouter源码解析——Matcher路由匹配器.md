---
title: VueRouter源码解析——Matcher路由匹配器
date: 2020-02-23 18:04:59
tags: [前端,Vue]
---
## `createMatcher`
`createMatcher`函数返回一个Matcher对象，它的定义如下：
```js
export type Matcher = {
  match: (raw: RawLocation, current?: Route, redirectedFrom?: Location) => Route;
  addRoutes: (routes: Array<RouteConfig>) => void;
};
```

当执行`createMatcher`函数时，主要步骤是建立路由映射表，以及路由映射Record之间的父子关系，以此建立路由映射树。

```js
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  // pathList<Array[RouteRecord]>
  // pathMap<path, RouteRecord>
  // nameMap<path, RouteRecord>
  const { pathList, pathMap, nameMap } = createRouteMap(routes)

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

      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location, redirectedFrom)
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
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
    //....
  }

  function alias (
    record: RouteRecord,
    location: Location,
    matchAs: string
  ): Route {
    //....
  }

  function _createRoute (
    //....
  }

  return {
    match,
    addRoutes
  }
}
```

因此，先会执行``const { pathList, pathMap, nameMap } = createRouteMap(routes)``创建路由映射表，其中`pathList`记录所有的路由Record，`pathMap`和`nameMap`分别是以`path`和`name`作为键值对路由Record的快速映射表。

```js
export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>
): {
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>
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
  // 确保 * 任意匹配的路由必须在 pathList 的最末端
  // 也就是最后再匹配
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }

  // 非嵌套路由必须以 / 开头
  // 否则开发环境下会给警告
  if (process.env.NODE_ENV === 'development') {
    // warn if routes do not include leading slashes
    const found = pathList
    // check for missing leading slash
      .filter(path => path && path.charAt(0) !== '*' && path.charAt(0) !== '/')

    if (found.length > 0) {
      const pathNames = found.map(path => `- ${path}`).join('\n')
      warn(false, `Non-nested routes must include a leading slash character. Fix the following routes: \n${pathNames}`)
    }
  }

  // 返回
  return {
    pathList,
    pathMap,
    nameMap
  }
}
```

`createRouteMap`的核心在于递归`routes`（它是我们定义路由时的配置），对每个路由创建一个Record，并且递归以建立Record之间的父子关系。

```js
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
      `route config "component" for path: ${String(
        path || name
      )} cannot be a ` + `string id. Use an actual component instead.`
    )
  }

  const pathToRegexpOptions: PathToRegexpOptions =
    route.pathToRegexpOptions || {}
  // 标准化路径
  const normalizedPath = normalizePath(path, parent, pathToRegexpOptions.strict)

  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }

  const record: RouteRecord = {
    // path 是一个完整路径
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    // 如果没有传组件，那么就创建个空
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props:
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props }
  }

  // 递归 children 构建路由表
  if (route.children) {
    // 如果一个命名路由没有重定向并且有一个默认的子路由
    // 当通过 name 进行路由转到该路由时，它的默认子路由就不会渲染
    // Warn if route is named, does not redirect and has a default child route.
    // If users navigate to this route by name, the default child will
    // not be rendered (GH Issue #629)
    if (process.env.NODE_ENV !== 'production') {
      if (
        route.name &&
        !route.redirect &&
        route.children.some(child => /^\/?$/.test(child.path))
      ) {
        warn(
          false,
          `Named Route '${route.name}' has a default child route. ` +
            `When navigating to this named route (:to="{name: '${
              route.name
            }'"), ` +
            `the default child route will not be rendered. Remove the name from ` +
            `this route and use the name of the default child route for named ` +
            `links instead.`
        )
      }
    }
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }

  // pathMap 是以 <path, Record> 类型定义的字典
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }

  if (route.alias !== undefined) {
    const aliases = Array.isArray(route.alias) ? route.alias : [route.alias]
    for (let i = 0; i < aliases.length; ++i) {
      const alias = aliases[i]
      if (process.env.NODE_ENV !== 'production' && alias === path) {
        warn(
          false,
          `Found an alias with the same value as the path: "${path}". You have to remove that alias. It will be ignored in development.`
        )
        // skip in dev to make it work
        continue
      }

      // 别名实际上就是为 alias 创建一个跟原来基本一致的路由表（含子路由）
      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs
      )
    }
  }

  // 如果 RouteConfig 中有 name 属性，也就是命名路由
  // 就添加到 nameMap 中
  // nameMap 是以 <name, Record> 为类型的字典
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
```

首先是对`path`的标准化，对于嵌套在父Record中的子Record，如果`path`不以`/`开头，那么就会加上父Record的`path`在前，比如有如下配置：
```js
const routes: [
{
    path: '/aaa',
    children: [
    {
        path: 'bbb'
    }, {
        path: '/ccc'
    }, {
        path: '/aaa/ddd'
    }
    ]
}
]
```

它们在normalize之后的`path`分别为：`/aaa/bbb`、`/ccc`、`/aaa/ddd`。可以理解为`/`开头的`path`会被路由当成是绝对路径，而没有`/`开头的是相对路径，相对路径会拼接上父路由的`path`。

normalize后就是创建一个record实例：
```js
  const record: RouteRecord = {
    // path 是一个完整路径
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    // 如果没有传组件，那么就创建个空
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props:
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props }
  }
```

因为路由是支持命名视图的，也就是说可以在同一个页面中嵌套多个`router-view`，通过`name`属性区分。如果我们没有使用命名视图，则会被分配到`default`的路由视图，props同理。

然后就是对`route.children`递归创建路由表，把当前record作为参数parent传入，以建立父子关系。

```js
  // 递归 children 构建路由表
  if (route.children) {
    // 如果一个命名路由没有重定向并且有一个默认的子路由
    // 当通过 name 进行路由转到该路由时，它的默认子路由就不会渲染
    // Warn if route is named, does not redirect and has a default child route.
    // If users navigate to this route by name, the default child will
    // not be rendered (GH Issue #629)
    if (process.env.NODE_ENV !== 'production') {
      if (
        route.name &&
        !route.redirect &&
        route.children.some(child => /^\/?$/.test(child.path))
      ) {
        warn(
          false,
          `Named Route '${route.name}' has a default child route. ` +
            `When navigating to this named route (:to="{name: '${
              route.name
            }'"), ` +
            `the default child route will not be rendered. Remove the name from ` +
            `this route and use the name of the default child route for named ` +
            `links instead.`
        )
      }
    }
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
```

因为每一个路由都有`path`，所以需要添加到`pathMap`中，方便匹配的时候快速定位。

```js
  // pathMap 是以 <path, Record> 类型定义的字典
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
```

然后是处理`alias`，路由别名。但实际上它相当于copy了一份原路由：

```js
  if (route.alias !== undefined) {
    const aliases = Array.isArray(route.alias) ? route.alias : [route.alias]
    for (let i = 0; i < aliases.length; ++i) {
      const alias = aliases[i]
      if (process.env.NODE_ENV !== 'production' && alias === path) {
        warn(
          false,
          `Found an alias with the same value as the path: "${path}". You have to remove that alias. It will be ignored in development.`
        )
        // skip in dev to make it work
        continue
      }

      // 别名实际上就是为 alias 创建一个跟原来基本一致的路由表（含子路由）
      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs
      )
    }
  }
```

对于命名路由，加入`nameMap`以便匹配时快速定位。
```js
  // 如果 RouteConfig 中有 name 属性，也就是命名路由
  // 就添加到 nameMap 中
  // nameMap 是以 <name, Record> 为类型的字典
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
```

这样，routerMap的创建过程就完成了。`createRouteMap`把`pathList`、`nameMap`以及`pathMap`返回给matcher，用于匹配。

### `addRoutes`
VueRouter允许开发者通过addRoutes方法动态添加路由，因为它实际上就是调用了`createRouteMap`函数，再次创建一个新的路由表。

```js
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }
```

### `match`
当进行路由的跳转时，会调用match方法匹配路由，因此，match方法实际上是VueRouter的核心部分。

```js
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

      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location, redirectedFrom)
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
```

`match`方法主要是通过传入的`rawLocation`定位到对应的`Record`，然后解析`params`、`query`这些参数。

首先需要做的是将`Location`标准化，便于后续判断是通过命名路由还是`path`做路由的跳转。

```js
    const location = normalizeLocation(raw, currentRoute, false, router)
```

```js
export function normalizeLocation (
  raw: RawLocation,
  current: ?Route,
  append: ?boolean,
  router: ?VueRouter
): Location {
  let next: Location = typeof raw === 'string' ? { path: raw } : raw
  // named target
  if (next._normalized) {
    return next
  } else if (next.name) {
    // 如果通过 { name: 'xxx' } 跳转，序列化 params后直接返回
    next = extend({}, raw)
    const params = next.params
    if (params && typeof params === 'object') {
      next.params = extend({}, params)
    }
    return next
  }

  // relative params
  // 如果没有提供 path 和 name
  // 说明只在本路由做了一次参数的更新，即：对于路由 /user/:id ，变化是 /user/1 => /user/2
  if (!next.path && next.params && current) {
    next = extend({}, next)
    next._normalized = true
    const params: any = extend(extend({}, current.params), next.params)
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

  // 如果并没有提供 name 而是 提供了 path
  // 此时 params 参数无效

  // parsedPath = { path, hash, query } ，三者皆为字符串
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
```

跳转的情况主要有三种：
1. 通过`name`属性进行跳转，这时传入`params`对象作为参数。
2. 只在同一个路由进行跳转，只有params发生变化。例如，对于路由`/user/:id`发生了这样的跳转：`/user/1 => /user/2`。
3. 通过`path`属性进行跳转，这时传入`params`对象无效，`params`必须在`path`中解析。

首先，第一种的效率肯定是最高的，因为它不需要通过`path`解析params，而是直接通过用户传入params对象获得。

对于第二种，也是常见的，但是它并不确定通过命名路由跳转还是路径跳转，因此它需要获取当前路由的Record，同时获取path和name（如果有），然后把params都序列化到path上以备用（如果name不存在的话）。

对于通过path进行跳转，是效率最低的，因为它涉及到对路径解析params的问题。

当`location`标准化之后，就会先通过`name`判断是否通过命名路由进行跳转。

```js
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

      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location, redirectedFrom)
    }
```

如果传入了`name`，但没有对应的Record，它会创建一个空的Route对象。

否则，就会通过`nameMap`快速查找到对应的Record，然后合并params，创建完整的`path`，一同传入`_createRoute`方法中，创建一个Route并返回。

如果通过`path`跳转路由，效率就会稍低：
```js
else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
```

因为`path`中涉及到`params`，因此无法单独通过`path`比对确定路由，那么就需要通过遍历所有路由，通过`matchRoute`方法匹配所有路由的`path`以获得正确的Record。

## `_createRoute`
当获取到Record之后，就就会通过调用`_createRoute`方法创建Route。

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

暂时先不看重定向和别名的情况，实际上是调用了`createRoute`方法：

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

这个方法很简单，结合之前的location填充Route实例中的数据，然后通过`Object.freeze(route)`约束整个对象不可更改。

需要关注的点是：
```js
matched: record ? formatMatch(record) : []
```

`matched`中存放的是当前路由匹配的组件，什么意思呢？就是如果我们有这样的布局：
```
--------------------
|     Layout A     |
|------------------|
|  Router-view     |
|     PageB        |
|  --------------- |
|  |  Router-view| |
|  |  SubPageC   | |
|  |-------------| |
|------------------|
```

`LayoutA`中有一个`router-view`，当前路由是`PageB`，而`PageB`中也有一个`router-view`，当前路由是`SubPageC`，那么，我们定义routes时创建的Record父子关系就是这样的：
```
LayoutA <- PageB <- SubPageC
```

如果我们当前的路由是关联与`SubPageC`的，那么很显然我们也需要创建组件`PageB`和`LayoutA`，那么，我们就需要把匹配的组件都记录到一个数组中，并且按父到子的顺序排列。那么，`formatMatch`就是帮我们做这样的工作的：

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

还记得在`createRouteMap`中会堆归创建路由表吗？`record.parent`就是在这时候建立的联系。这样，我们就可以通过`record.parent`找到父组件，以及上层的组件。
