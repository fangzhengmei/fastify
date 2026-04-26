# Fastify 插件 Scope 与装饰器传播机制深度分析

## 1. 概述

Fastify 的插件系统基于 `avvio` 依赖树构建，每个插件默认拥有独立的 scope 隔离。`fastify-plugin` 则能够打破这种隔离，使 `decorate` 定义的装饰器对父 scope 可见。本文将深入分析这套 encapsulation 机制的实现原理。

## 2. 核心架构

### 2.1 Avvio 依赖树机制

Fastify 使用 `avvio` 库来管理插件的依赖树和加载顺序。在 `fastify.js:361-369` 中可以看到核心配置：

```javascript
const avvio = Avvio(fastify, {
  autostart: false,
  timeout: isNaN(avvioPluginTimeout) === false ? avvioPluginTimeout : defaultInitOptions.pluginTimeout,
  expose: {
    use: 'register'
  }
})
// Override to allow the plugin encapsulation
avvio.override = override
```

**关键点**：
- `avvio` 将 `use` 方法暴露为 `register`，这就是 `fastify.register()` 的来源
- `avvio.override = override` 是封装机制的核心钩子，每次注册插件时都会调用
- Avvio 负责管理插件的依赖树、加载顺序和生命周期

### 2.2 插件注册流程

插件注册的完整流程如下：

1. 用户调用 `fastify.register(plugin, opts)`
2. Avvio 调用 `avvio.override(old, fn, opts)` 函数
3. `override` 函数决定是否创建新的封装上下文
4. 插件函数在返回的实例上执行

## 3. Scope 隔离的核心实现

### 3.1 override 函数分析

`lib/plugin-override.js` 中的 `override` 函数是封装机制的核心：

```javascript
module.exports = function override (old, fn, opts) {
  const shouldSkipOverride = pluginUtils.registerPlugin.call(old, fn)

  const fnName = pluginUtils.getPluginName(fn) || pluginUtils.getFuncPreview(fn)
  if (shouldSkipOverride) {
    old[kPluginNameChain].push(fnName)
    return old
  }

  const instance = Object.create(old)
  old[kChildren].push(instance)
  instance.ready = old[kAvvioBoot].bind(instance)
  instance[kChildren] = []

  instance[kReply] = Reply.buildReply(instance[kReply])
  instance[kRequest] = Request.buildRequest(instance[kRequest])

  instance[kContentTypeParser] = ContentTypeParser.helpers.buildContentTypeParser(instance[kContentTypeParser])
  instance[kHooks] = buildHooks(instance[kHooks])
  instance[kRoutePrefix] = buildRoutePrefix(instance[kRoutePrefix], opts.prefix)
  instance[kLogLevel] = opts.logLevel || instance[kLogLevel]
  instance[kSchemaController] = SchemaController.buildSchemaController(old[kSchemaController])
  instance.getSchema = instance[kSchemaController].getSchema.bind(instance[kSchemaController])
  instance.getSchemas = instance[kSchemaController].getSchemas.bind(instance[kSchemaController])

  instance[pluginUtils.kRegisteredPlugins] = Object.create(instance[pluginUtils.kRegisteredPlugins])

  instance[kPluginNameChain] = [fnName]
  instance[kErrorHandlerAlreadySet] = false

  if (instance[kLogSerializers] || opts.logSerializers) {
    instance[kLogSerializers] = Object.assign(Object.create(instance[kLogSerializers]), opts.logSerializers)
  }

  if (opts.prefix) {
    instance[kFourOhFour].arrange404(instance)
  }

  for (const hook of instance[kHooks].onRegister) hook.call(old, instance, opts)

  return instance
}
```

### 3.2 隔离的组件

当 `shouldSkipOverride` 为 `false` 时，会创建新的实例并隔离以下组件：

| 组件 | Symbol | 隔离方式 | 说明 |
|------|--------|----------|------|
| Reply 构造函数 | `kReply` | `Reply.buildReply()` | 创建新的 Reply 构造函数，独立管理装饰器 |
| Request 构造函数 | `kRequest` | `Request.buildRequest()` | 创建新的 Request 构造函数，独立管理装饰器 |
| Content-Type 解析器 | `kContentTypeParser` | `buildContentTypeParser()` | 独立的内容类型解析器注册表 |
| Hooks | `kHooks` | `buildHooks()` | 复制父 hooks，独立管理生命周期钩子 |
| 路由前缀 | `kRoutePrefix` | `buildRoutePrefix()` | 独立的路由前缀 |
| 日志级别 | `kLogLevel` | 直接赋值 | 独立的日志级别配置 |
| Schema 控制器 | `kSchemaController` | `buildSchemaController()` | 独立的 schema 管理 |
| 已注册插件 | `kRegisteredPlugins` | `Object.create()` | 独立的插件注册跟踪 |
| 插件名称链 | `kPluginNameChain` | 重新初始化 | 用于错误追踪和调试 |
| 子实例列表 | `kChildren` | 重新初始化 | 独立的子实例追踪 |

### 3.3 原型链继承

新实例通过 `Object.create(old)` 创建，这意味着：

1. **子实例可以访问父实例的所有属性和方法**（通过原型链）
2. **子实例的修改不会影响父实例**（除非修改原型链上的对象）
3. **装饰器的传播是单向的**：父→子，而非子→父

## 4. 装饰器传播机制

### 4.1 三种装饰器类型

Fastify 提供三种装饰器 API：

1. **`decorate(name, value)`** - 装饰 Fastify 实例本身
2. **`decorateRequest(name, value)`** - 装饰 Request 对象
3. **`decorateReply(name, value)`** - 装饰 Reply 对象

### 4.2 实例装饰器 (decorate)

实现位于 `lib/decorate.js:19-34`：

```javascript
function decorate (instance, name, fn, dependencies) {
  if (Object.hasOwn(instance, name)) {
    throw new FST_ERR_DEC_ALREADY_PRESENT(name)
  }

  checkDependencies(instance, name, dependencies)

  if (fn && (typeof fn.getter === 'function' || typeof fn.setter === 'function')) {
    Object.defineProperty(instance, name, {
      get: fn.getter,
      set: fn.setter
    })
  } else {
    instance[name] = fn
  }
}
```

**传播机制**：
- 装饰器直接添加到实例的自身属性（`Object.hasOwn` 检查）
- 子实例通过 `Object.create(old)` 的原型链可以访问父实例的装饰器
- 子实例添加的装饰器是自身属性，不会传播到父实例

**示例**：
```javascript
fastify.decorate('rootUtil', () => 'root')

fastify.register(async function childScope (instance) {
  console.log(instance.rootUtil)  // 通过原型链访问，输出 'root'
  instance.decorate('childUtil', () => 'child')
})

fastify.ready(() => {
  console.log(fastify.childUtil)  // undefined，子装饰器不会向上传播
})
```

### 4.3 Request/Reply 装饰器 (decorateRequest/decorateReply)

实现位于 `lib/decorate.js:48-67`：

```javascript
function decorateConstructor (konstructor, name, fn, dependencies) {
  const instance = konstructor.prototype
  if (Object.hasOwn(instance, name) || hasKey(konstructor, name)) {
    throw new FST_ERR_DEC_ALREADY_PRESENT(name)
  }

  konstructor[kHasBeenDecorated] = true
  checkDependencies(konstructor, name, dependencies)

  if (fn && (typeof fn.getter === 'function' || typeof fn.setter === 'function')) {
    Object.defineProperty(instance, name, {
      get: fn.getter,
      set: fn.setter
    })
  } else if (typeof fn === 'function') {
    instance[name] = fn
  } else {
    konstructor.props.push({ key: name, value: fn })
  }
}
```

**关键设计**：
- **函数类型的装饰器**：添加到构造函数的 `prototype`
- **非函数类型的装饰器**：存储在构造函数的 `props` 数组中
- `kHasBeenDecorated` 标记用于追踪是否已被装饰

### 4.4 Request/Reply 构造函数的构建

当创建新 scope 时，会调用 `buildRequest` 和 `buildReply`：

**Request 构建** (`lib/request.js:70-93`)：
```javascript
function buildRegularRequest (R) {
  const props = R.props.slice()  // 复制父构造函数的 props
  function _Request (id, params, req, query, log, context) {
    this.id = id
    this[kRouteContext] = context
    this.params = params
    this.raw = req
    this.query = query
    this.log = log
    this.body = undefined

    let prop
    for (let i = 0; i < props.length; i++) {
      prop = props[i]
      this[prop.key] = prop.value  // 初始化 props 中的装饰器
    }
  }
  Object.setPrototypeOf(_Request.prototype, R.prototype)  // 原型链继承
  Object.setPrototypeOf(_Request, R)
  _Request.props = props
  _Request.parent = R

  return _Request
}
```

**Reply 构建** (`lib/reply.js:957-985`)：
```javascript
function buildReply (R) {
  const props = R.props.slice()

  function _Reply (res, request, log) {
    this.raw = res
    this[kReplyIsError] = false
    // ... 初始化其他属性

    let prop
    for (let i = 0; i < props.length; i++) {
      prop = props[i]
      this[prop.key] = prop.value
    }
  }
  Object.setPrototypeOf(_Reply.prototype, R.prototype)
  Object.setPrototypeOf(_Reply, R)
  _Reply.parent = R
  _Reply.props = props
  return _Reply
}
```

**传播机制**：
1. **props 数组复制**：`R.props.slice()` 复制父构造函数的非函数装饰器
2. **原型链继承**：`Object.setPrototypeOf(_Request.prototype, R.prototype)` 继承函数装饰器
3. **独立的 props**：子构造函数有自己的 `props` 数组，添加新装饰器不会影响父构造函数

**示例**：
```javascript
fastify.decorateRequest('rootData', 'root')

fastify.register(async function childScope (instance) {
  // 通过原型链和 props 继承，可以访问 rootData
  instance.decorateRequest('childData', 'child')
  
  instance.get('/test', (req, reply) => {
    console.log(req.rootData)   // 'root' - 继承自父 scope
    console.log(req.childData)  // 'child' - 当前 scope 的装饰器
    reply.send({ ok: true })
  })
})

// 父 scope 的 Request 构造函数没有 childData
```

## 5. Skip-Override 标记与 fastify-plugin

### 5.1 skip-override 标记的定义

`lib/plugin-utils.js:60-62`：
```javascript
function shouldSkipOverride (fn) {
  return !!fn[Symbol.for('skip-override')]
}
```

**关键点**：
- 使用全局 Symbol `Symbol.for('skip-override')` 作为标记
- 只要函数上存在这个属性且为 truthy 值，就会跳过封装

### 5.2 registerPlugin 函数

`lib/plugin-utils.js:147-154`：
```javascript
function registerPlugin (fn) {
  const pluginName = registerPluginName.call(this, fn) || getPluginName(fn)
  checkPluginHealthiness.call(this, fn, pluginName)
  checkVersion.call(this, fn)
  checkDecorators.call(this, fn)
  checkDependencies.call(this, fn)
  return shouldSkipOverride(fn)  // 返回是否跳过封装
}
```

**执行流程**：
1. 注册插件名称
2. 检查插件健康性（异步函数参数校验）
3. 检查 Fastify 版本兼容性
4. 检查依赖的装饰器是否存在
5. 检查依赖的插件是否已注册
6. **返回 `shouldSkipOverride(fn)` 的结果**

### 5.3 在 override 中的应用

回到 `lib/plugin-override.js:28-36`：
```javascript
module.exports = function override (old, fn, opts) {
  const shouldSkipOverride = pluginUtils.registerPlugin.call(old, fn)

  const fnName = pluginUtils.getPluginName(fn) || pluginUtils.getFuncPreview(fn)
  if (shouldSkipOverride) {
    // after every plugin registration we will enter a new name
    old[kPluginNameChain].push(fnName)
    return old  // 直接返回旧实例，不创建新的封装上下文
  }
  // ... 创建新的封装上下文
}
```

**当 `shouldSkipOverride` 为 `true` 时**：
1. 不创建 `Object.create(old)` 的新实例
2. 直接返回 `old` 实例
3. 插件在父 scope 中执行
4. 装饰器直接添加到父实例上

### 5.4 fastify-plugin 的作用

`fastify-plugin` 库的核心功能就是给插件函数添加 `skip-override` 标记：

```javascript
// fastify-plugin 核心逻辑（简化版）
function fp (fn, opts = {}) {
  fn[Symbol.for('skip-override')] = true
  if (opts.name) {
    fn[Symbol.for('plugin-meta')] = { name: opts.name }
  }
  return fn
}
```

**实际效果**：
```javascript
const fp = require('fastify-plugin')

// 普通插件 - 有隔离
fastify.register(function normalPlugin (instance, opts, done) {
  instance.decorate('normal', '隔离的装饰器')
  done()
})

// 使用 fastify-plugin - 打破隔离
fastify.register(fp(function sharedPlugin (instance, opts, done) {
  instance.decorate('shared', '共享的装饰器')
  done()
}))

fastify.ready(() => {
  console.log(fastify.normal)  // undefined - 被隔离了
  console.log(fastify.shared)  // '共享的装饰器' - 打破了隔离
})
```

### 5.5 测试用例验证

从 `test/plugin.1.test.js:60-98` 的测试用例可以看到实际行为：

```javascript
test('fastify.register with fastify-plugin should not encapsulate his code', async t => {
  t.plan(9)
  const fastify = Fastify()

  fastify.register((instance, opts, done) => {
    instance.register(fp((i, o, n) => {
      i.decorate('test', () => {})
      t.assert.ok(i.test)
      n()
    }))

    t.assert.ok(!instance.test)  // 此时装饰器还未添加

    // 装饰器在插件执行完毕后才会对父实例可见
    instance.after(() => {
      t.assert.ok(instance.test)  // 现在可以访问了
    })

    instance.get('/', (req, reply) => {
      t.assert.ok(instance.test)
      reply.send({ hello: 'world' })
    })

    done()
  })

  fastify.ready(() => {
    t.assert.ok(!fastify.test)  // 根实例仍然无法访问
  })
  // ...
})
```

**关键观察**：
1. 使用 `fp` 包装的插件，其装饰器会传播到父 `instance`
2. 但传播是在插件执行完毕后（`after` 钩子中）才可见
3. 根 `fastify` 实例仍然无法访问，因为外层的 `register` 没有使用 `fp`

### 5.6 层级传播示例

```javascript
const fastify = Fastify()
const fp = require('fastify-plugin')

// Level 0: Root
fastify.decorate('level0', 'root')

// Level 1: 普通插件（有隔离）
fastify.register(function level1 (instance, opts, done) {
  instance.decorate('level1', 'level1')
  
  // Level 2: 使用 fastify-plugin（打破隔离）
  instance.register(fp(function level2 (i, o, n) {
    i.decorate('level2', 'level2')
    n()
  }))
  
  instance.after(() => {
    console.log('Level 1 can access:')
    console.log('  level0:', instance.level0)  // 'root' - 继承
    console.log('  level1:', instance.level1)  // 'level1' - 自身
    console.log('  level2:', instance.level2)  // 'level2' - 从 level2 传播上来
  })
  
  done()
})

fastify.ready(() => {
  console.log('Root can access:')
  console.log('  level0:', fastify.level0)  // 'root' - 自身
  console.log('  level1:', fastify.level1)  // undefined - 被隔离
  console.log('  level2:', fastify.level2)  // undefined - level1 是隔离的
})
```

## 6. 其他被隔离的组件

### 6.1 Hooks 隔离与传播机制（深度分析）

Hooks 系统是 Fastify 中最复杂的隔离机制之一，它有自己独特的传播规则。让我们深入分析。

#### 6.1.1 Hooks 的分类

`lib/hooks.js:3-22` 定义了两类 hooks：

```javascript
const applicationHooks = [
  'onRoute',
  'onRegister',
  'onReady',
  'onListen',
  'preClose',
  'onClose'
]
const lifecycleHooks = [
  'onTimeout',
  'onRequest',
  'preParsing',
  'preValidation',
  'preSerialization',
  'preHandler',
  'onSend',
  'onResponse',
  'onError',
  'onRequestAbort'
]
```

| 类型 | 包含的 Hooks | 特点 |
|------|-------------|------|
| **应用 Hooks** | `onRoute`, `onRegister`, `onReady`, `onListen`, `preClose`, `onClose` | 与插件生命周期相关 |
| **生命周期 Hooks** | `onRequest`, `preParsing`, `preValidation`, `preHandler`, `onSend`, `onResponse`, `onError` 等 | 与 HTTP 请求/响应生命周期相关 |

#### 6.1.2 addHook 的两种行为

`fastify.js:568-614` 中的 `addHook` 函数是理解 hooks 传播的关键：

```javascript
function addHook (name, fn) {
  throwIfAlreadyStarted('Cannot call "addHook"!')

  // ... 参数验证 ...

  if (name === 'onClose') {
    this.onClose(fn.bind(this))
  } else if (name === 'onReady' || name === 'onListen' || name === 'onRoute') {
    // 行为 1: 直接添加到当前实例，不传播
    this[kHooks].add(name, fn)
  } else {
    // 行为 2: 通过 _addHook 递归传播到所有子 scope
    this.after((err, done) => {
      try {
        _addHook.call(this, name, fn)
        done(err)
      } catch (err) {
        done(err)
      }
    })
  }
  return this

  function _addHook (name, fn) {
    this[kHooks].add(name, fn)
    // 关键：递归传播到所有子实例
    this[kChildren].forEach(child => _addHook.call(child, name, fn))
  }
}
```

**关键发现**：

1. **`onReady`, `onListen`, `onRoute`**：直接添加到当前实例的 `kHooks`，**不传播**到子 scope
2. **其他所有 hooks**（生命周期 hooks）：通过 `_addHook` 函数**递归传播**到当前实例和所有已存在的子实例
3. **`_addHook` 的递归机制**：不仅添加到 `this[kHooks]`，还会遍历 `this[kChildren]` 并递归调用

#### 6.1.3 buildHooks 的继承规则

`lib/hooks.js:73-90` 中的 `buildHooks` 函数在创建新封装实例时被调用：

```javascript
function buildHooks (h) {
  const hooks = new Hooks()
  // 生命周期 hooks：通过 .slice() 浅拷贝父实例的 hooks
  hooks.onRequest = h.onRequest.slice()
  hooks.preParsing = h.preParsing.slice()
  hooks.preValidation = h.preValidation.slice()
  hooks.preSerialization = h.preSerialization.slice()
  hooks.preHandler = h.preHandler.slice()
  hooks.onSend = h.onSend.slice()
  hooks.onResponse = h.onResponse.slice()
  hooks.onError = h.onError.slice()
  hooks.onRoute = h.onRoute.slice()
  hooks.onRegister = h.onRegister.slice()
  hooks.onTimeout = h.onTimeout.slice()
  hooks.onRequestAbort = h.onRequestAbort.slice()
  // 应用 hooks：不继承，每个 scope 独立
  hooks.onReady = []
  hooks.onListen = []
  hooks.preClose = []
  return hooks
}
```

**继承规则**：

| Hook 类型 | 继承方式 | 说明 |
|-----------|----------|------|
| 生命周期 hooks | `slice()` 浅拷贝 | 新实例继承父实例已有的 hooks |
| `onRoute`, `onRegister` | `slice()` 浅拷贝 | 这两个应用 hooks 也会被继承 |
| `onReady`, `onListen`, `preClose` | 初始化为空数组 `[]` | **不继承**，每个 scope 完全独立 |

#### 6.1.4 Hooks 的双重传播机制

结合 `addHook` 和 `buildHooks`，hooks 实际上有**双重传播保障**：

**机制 1：创建时继承（buildHooks）**
- 当创建新的子 scope 时，通过 `buildHooks` 复制父实例已有的生命周期 hooks
- 这确保了子 scope 能继承注册时父实例已有的 hooks

**机制 2：运行时传播（_addHook）**
- 当父实例后续添加新的生命周期 hooks 时，通过 `_addHook` 递归传播到已存在的子实例
- 这确保了后续添加的 hooks 也能到达已创建的子 scope

**流程图**：
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Hooks 双重传播机制                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  时机 1: 创建新 scope 时（buildHooks）                              │
│  ┌─────────────┐                                                    │
│  │  父实例     │  已有 hooks: [hook1, hook2]                       │
│  │  kHooks     │                                                    │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼ Object.create + buildHooks                                │
│  ┌─────────────┐                                                    │
│  │  子实例     │  kHooks = {                                        │
│  │  kHooks     │    onRequest: [hook1, hook2].slice()  // 复制   │
│  └─────────────┘    ...                                            │
│                  }                                                  │
│                                                                      │
│  时机 2: 父实例后续添加 hooks 时（_addHook）                        │
│  ┌─────────────┐                                                    │
│  │  父实例     │  addHook('onRequest', hook3)                      │
│  │             │                                                    │
│  │  _addHook:  │  1. this[kHooks].add(name, fn)  // 添加到自己   │
│  │             │  2. 遍历 kChildren，递归调用 _addHook             │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼ 递归传播                                                  │
│  ┌─────────────┐                                                    │
│  │  子实例     │  也收到 hook3                                      │
│  │  kHooks     │                                                    │
│  └─────────────┘                                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 6.1.5 路由注册时 hooks 的收集与合并

当路由注册时，在 `preReady` 阶段会收集和合并 hooks。`lib/route.js:394-400`：

```javascript
avvio.once('preReady', () => {
  for (const hook of lifecycleHooks) {
    const toSet = this[kHooks][hook]
      .concat(opts[hook] || [])  // 合并路由级别的 hooks
      .map(h => h.bind(this))     // 绑定到当前实例
    context[hook] = toSet.length ? toSet : null
  }
  // ...
})
```

**关键流程**：

1. **收集时机**：在 `avvio.once('preReady')` 回调中执行
2. **合并来源**：
   - `this[kHooks][hook]`：当前 scope 实例级别的 hooks
   - `opts[hook]`：路由定义时指定的路由级 hooks
3. **绑定上下文**：所有 hooks 通过 `.bind(this)` 绑定到当前 Fastify 实例
4. **存储位置**：合并后的 hooks 存储在 `context[hook]` 中

**Context 与 hooks 的关系**：

每个路由都有一个独立的 `Context` 对象（`lib/context.js`），它存储了该路由执行时需要的所有 hooks：

```javascript
// Context 构造函数中引用的 hooks 相关属性
function Context ({
  // ...
  onRequest,     // 来自 context[hook] 的赋值
  onSend,        // 来自 context[hook] 的赋值
  onError,       // 来自 context[hook] 的赋值
  onTimeout,     // 来自 context[hook] 的赋值
  preHandler,    // 来自 context[hook] 的赋值
  preParsing,    // 来自 context[hook] 的赋值
  preValidation, // 来自 context[hook] 的赋值
  preSerialization, // 来自 context[hook] 的赋值
  onResponse,    // 来自 context[hook] 的赋值
  onRequestAbort // 来自 context[hook] 的赋值
  // ...
})
```

这意味着：**每个路由的 hooks 在 `preReady` 阶段就已经确定并固化在 Context 中**，后续的修改不会影响已注册的路由。

#### 6.1.6 Skip-Override 对 Hooks 的影响

现在让我们分析 `skip-override` 标记如何影响 hooks 的注册目标和传播。

**场景对比**：

```javascript
const fastify = Fastify()
const fp = require('fastify-plugin')

// 场景 A: 普通插件（无 skip-override）
fastify.register(function normalPlugin (instance, opts, done) {
  // 1. override 函数创建了新的封装实例
  // 2. instance 是新创建的子实例
  // 3. addHook 会调用 _addHook，传播到 instance 的子 scope
  // 4. 但不会传播到 fastify（父实例）
  instance.addHook('onRequest', (req, reply, done) => {
    console.log('normalPlugin onRequest')
    done()
  })
  done()
})

// 场景 B: 使用 fastify-plugin（有 skip-override）
fastify.register(fp(function sharedPlugin (instance, opts, done) {
  // 1. override 函数直接返回 old（fastify 实例）
  // 2. instance === fastify
  // 3. addHook 会调用 _addHook，传播到 fastify 的所有子 scope
  instance.addHook('onRequest', (req, reply, done) => {
    console.log('sharedPlugin onRequest')
    done()
  })
  done()
}))
```

**核心差异分析**：

| 维度 | 普通插件（无 skip-override） | fastify-plugin（有 skip-override） |
|------|------------------------------|------------------------------------|
| **注册目标** | 新创建的子实例 `instance` | 父实例 `old`（直接是 fastify 或上层实例） |
| **Hook 传播范围** | 只传播到 `instance` 的子 scope | 传播到 `old` 的所有子 scope（即整个插件树） |
| **父实例可见性** | 父实例（如 fastify）的 `kHooks` 不会包含这个 hook | 父实例的 `kHooks` 直接包含这个 hook |
| **对已有路由的影响** | 不影响父实例已注册路由的 Context | 会影响父实例已注册路由吗？**答案：不会** |

**关键问题：为什么不会影响已有路由？**

答案在 `lib/route.js:394-400` 的 `preReady` 阶段：

```javascript
avvio.once('preReady', () => {
  // hooks 在此时被收集并固化到 Context 中
  for (const hook of lifecycleHooks) {
    const toSet = this[kHooks][hook].concat(opts[hook] || [])
    context[hook] = toSet.length ? toSet : null
  }
})
```

**时间线分析**：

```
时间点 1: fastify.register(plugin) 调用
         → override 函数执行
         → 插件函数执行
         → 插件中的 addHook 执行（添加到 kHooks）

时间点 2: avvio 内部处理，等待所有插件注册完成

时间点 3: preReady 事件触发
         → 所有路由的 hooks 被收集到 Context 中
         → 此时 kHooks 中的所有 hooks 都会被包含

时间点 4: ready 事件触发
         → 服务器可以开始接收请求
```

**所以**：
- 如果插件使用 `skip-override`，并且在 `preReady` 之前注册，它的 hooks 会被包含在父实例所有路由的 Context 中
- 这就是为什么 `test/404s.test.js` 中的测试能通过：

```javascript
// 来自 test/404s.test.js:661-703
test('run non-encapsulated plugin hooks on default 404', (t, done) => {
  const fastify = Fastify()

  fastify.register(fp(function (instance, options, done) {
    instance.addHook('onRequest', function (req, res, done) {
      t.assert.ok(true, 'onRequest called')  // 这个 hook 会在 404 时触发
      done()
    })
    // ... 其他 hooks
    done()
  }))

  fastify.get('/', function (req, reply) {
    reply.send({ hello: 'world' })
  })

  // 访问不存在的路由（404）也会触发 fp 插件的 hooks
  fastify.inject({ method: 'POST', url: '/', payload: { hello: 'world' } }, ...)
})
```

**原因**：`fp` 插件的 `addHook` 直接在 `fastify` 实例上执行，当 `preReady` 触发时，这些 hooks 已经在 `fastify[kHooks]` 中，会被收集到所有路由（包括 404 路由）的 Context 中。

#### 6.1.7 测试用例验证

让我们通过 `test/hooks.test.js:169-200` 的测试来验证隔离性：

```javascript
test('onRequest hook should support encapsulation / 1', (t, testDone) => {
  t.plan(5)
  const fastify = Fastify()

  fastify.register((instance, opts, done) => {
    // 子插件添加的 hook
    instance.addHook('onRequest', (req, reply, done) => {
      t.assert.strictEqual(req.raw.url, '/plugin')  // 只在 /plugin 路由触发
      done()
    })

    instance.get('/plugin', (request, reply) => {
      reply.send()
    })

    done()
  })

  fastify.get('/root', (request, reply) => {
    reply.send()
  })

  // 访问 /root：不会触发子插件的 hook
  fastify.inject('/root', (err, res) => {
    t.assert.ifError(err)
    t.assert.strictEqual(res.statusCode, 200)

    // 访问 /plugin：会触发子插件的 hook
    fastify.inject('/plugin', (err, res) => {
      t.assert.ifError(err)
      t.assert.strictEqual(res.statusCode, 200)
      testDone()
    })
  })
})
```

**验证结果**：
- 子插件中添加的 `onRequest` hook **只在子插件的路由**（`/plugin`）上触发
- 父实例的路由（`/root`）**不会触发**子插件的 hook
- 这证明了 hooks 的隔离性：子 scope 的 hooks 不会向上传播

再看 `test/plugin.3.test.js:22-65` 中 `fastify-plugin` 的行为：

```javascript
test('add hooks after route declaration', async t => {
  t.plan(2)
  const fastify = Fastify()

  function plugin (instance, opts, done) {
    instance.decorateRequest('check', null)
    // 使用 fp 包装，这个 hook 会传播到所有子 scope
    instance.addHook('onRequest', (req, reply, done) => {
      req.check = {}
      done()
    })
    setImmediate(done)
  }
  fastify.register(fp(plugin))  // 注意：使用了 fp

  fastify.register((instance, options, done) => {
    // 这个子插件的路由会触发 fp 插件的 onRequest hook
    instance.addHook('preHandler', function b (req, res, done) {
      req.check.hook2 = true  // req.check 已经被 fp 插件的 hook 初始化
      done()
    })

    instance.get('/', (req, reply) => {
      reply.send(req.check)  // 返回 { hook1: true, hook2: true, hook3: true }
    })
    // ...
    done()
  })

  // 根实例也添加 preHandler
  fastify.addHook('preHandler', function a (req, res, done) {
    req.check.hook1 = true
    done()
  })

  // 最终结果：所有 hooks 都被触发
  // req.check = { hook1: true, hook2: true, hook3: true }
})
```

**验证结果**：
- 使用 `fp` 包装的插件添加的 `onRequest` hook 会传播到所有子 scope
- 根实例添加的 `preHandler` hook 也会传播到子 scope
- 子插件的路由能访问所有这些 hooks

#### 6.1.8 Hooks 传播规则总结表

| Hook 类型 | 父→子（创建时） | 父→子（运行时） | 子→父 | 受 skip-override 影响 |
|-----------|-----------------|-----------------|-------|----------------------|
| `onRequest` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `preParsing` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `preValidation` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `preHandler` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `preSerialization` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `onSend` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `onResponse` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `onError` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `onTimeout` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `onRequestAbort` | ✅ `slice()` 复制 | ✅ `_addHook` 递归 | ❌ 否 | ✅ 是 |
| `onRoute` | ✅ `slice()` 复制 | ❌ 直接添加，不递归 | ❌ 否 | ✅ 是 |
| `onRegister` | ✅ `slice()` 复制 | ❌ 直接添加，不递归 | ❌ 否 | ✅ 是 |
| `onReady` | ❌ 初始化为 `[]` | ❌ 直接添加，不递归 | ❌ 否 | ✅ 是（但不继承） |
| `onListen` | ❌ 初始化为 `[]` | ❌ 直接添加，不递归 | ❌ 否 | ✅ 是（但不继承） |
| `preClose` | ❌ 初始化为 `[]` | ❌ 直接添加，不递归 | ❌ 否 | ✅ 是（但不继承） |

#### 6.1.9 Hooks 完整传播流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Hooks 在 Scope 树中的完整传播流程                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  阶段 1: 插件注册与 Scope 创建                                               │
│  ─────────────────────────────                                               │
│                                                                              │
│  fastify.register(plugin)                                                    │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  override(old, fn, opts) 函数                                        │   │
│  │                                                                       │   │
│  │  检查 skip-override:                                                  │   │
│  │  ┌─────────────────┐         ┌─────────────────┐                     │   │
│  │  │ skip-override   │         │ skip-override   │                     │   │
│  │  │ === false       │         │ === true        │                     │   │
│  │  └────────┬────────┘         └────────┬────────┘                     │   │
│  │           │                            │                              │   │
│  │           ▼                            ▼                              │   │
│  │  ┌─────────────────┐         ┌─────────────────┐                     │   │
│  │  │ 创建新的封装    │         │ 直接返回 old    │                     │   │
│  │  │ 实例 instance   │         │ (不创建新实例)  │                     │   │
│  │  │                 │         │                 │                     │   │
│  │  │ instance.kHooks │         │ 插件中的        │                     │   │
│  │  │ = buildHooks(  │         │ addHook 直接在  │                     │   │
│  │  │   old.kHooks)  │         │ old 上执行      │                     │   │
│  │  │                 │         │                 │                     │   │
│  │  │ 生命周期 hooks  │         │                 │                     │   │
│  │  │ 通过 slice()    │         │                 │                     │   │
│  │  │ 复制父实例的    │         │                 │                     │   │
│  │  │ hooks           │         │                 │                     │   │
│  │  └─────────────────┘         └─────────────────┘                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  阶段 2: 插件执行与 addHook 调用                                            │
│  ────────────────────────────────                                           │
│                                                                              │
│  插件函数执行：instance.addHook(name, fn)                                   │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  addHook(name, fn) 函数                                              │   │
│  │                                                                       │   │
│  │  判断 hook 类型:                                                      │   │
│  │  ┌─────────────────────────────┐    ┌─────────────────────────┐   │   │
│  │  │ onReady / onListen / onRoute│    │ 其他 hooks（生命周期等）│   │   │
│  │  └──────────────┬──────────────┘    └───────────┬─────────────┘   │   │
│  │                 │                                 │                  │   │
│  │                 ▼                                 ▼                  │   │
│  │  ┌─────────────────────────┐    ┌─────────────────────────────┐   │   │
│  │  │ 直接添加到              │    │ 通过 this.after 延迟执行    │   │   │
│  │  │ this[kHooks].add()     │    │                             │   │   │
│  │  │                         │    │ _addHook 函数:             │   │   │
│  │  │ ❌ 不传播到子 scope     │    │                             │   │   │
│  │  │                         │    │ 1. this[kHooks].add(name,fn)│   │   │
│  │  │                         │    │ 2. 递归遍历 kChildren       │   │   │
│  │  │                         │    │    并调用 _addHook          │   │   │
│  │  │                         │    │                             │   │   │
│  │  │                         │    │ ✅ 传播到所有已存在的子 scope│   │   │
│  │  └─────────────────────────┘    └─────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  阶段 3: preReady 阶段 - Hooks 固化到路由 Context                          │
│  ───────────────────────────────────────────────────────                   │
│                                                                              │
│  avvio.once('preReady', () => {                                             │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  对每个已注册的路由:                                                 │   │
│  │                                                                       │   │
│  │  for (const hook of lifecycleHooks) {                                │   │
│  │    // 合并实例级别 hooks 和路由级别 hooks                            │   │
│  │    const toSet = this[kHooks][hook]                                  │   │
│  │      .concat(opts[hook] || [])                                       │   │
│  │      .map(h => h.bind(this))                                          │   │
│  │                                                                       │   │
│  │    // 固化到 Context，以后不会再改变                                 │   │
│  │    context[hook] = toSet.length ? toSet : null                      │   │
│  │  }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  关键点:                                                                     │
│  - 此时点之后添加的 hooks 不会影响已注册的路由                             │
│  - 每个路由有独立的 Context，包含它执行时需要的所有 hooks                   │
│                                                                              │
│  阶段 4: 请求处理 - 从 Context 读取 Hooks                                   │
│  ────────────────────────────────────────────                               │
│                                                                              │
│  请求到达时:                                                                 │
│  1. find-my-way 路由匹配，找到对应的 Context                                │
│  2. 按顺序执行 Context 中的 hooks:                                          │
│     onRequest → preParsing → preValidation → preHandler → handler         │
│                → preSerialization → onSend → onResponse                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Schema 隔离

`lib/schema-controller.js:40-58` 中的 `SchemaController` 构造函数：

```javascript
class SchemaController {
  constructor (parent, options) {
    this.opts = options || parent?.opts
    this.addedSchemas = false

    this.compilersFactory = this.opts.compilersFactory

    if (parent) {
      this.schemaBucket = this.opts.bucket(parent.getSchemas())  // 复制父 schemas
      this.validatorCompiler = parent.getValidatorCompiler()
      this.serializerCompiler = parent.getSerializerCompiler()
      this.isCustomValidatorCompiler = parent.isCustomValidatorCompiler
      this.isCustomSerializerCompiler = parent.isCustomSerializerCompiler
      this.parent = parent
    } else {
      this.schemaBucket = this.opts.bucket()
      this.isCustomValidatorCompiler = this.opts.isCustomValidatorCompiler || false
      this.isCustomSerializerCompiler = this.opts.isCustomSerializerCompiler || false
    }
  }
  // ...
}
```

**隔离方式**：
- 通过 `bucket(parent.getSchemas())` 复制父 schema
- 子 scope 添加的 schema 不会传播到父 scope
- 但可以通过 `fastify-plugin` 打破隔离

### 6.3 路由前缀隔离

`lib/plugin-override.js:76-90` 中的 `buildRoutePrefix`：

```javascript
function buildRoutePrefix (instancePrefix, pluginPrefix) {
  if (!pluginPrefix) {
    return instancePrefix
  }

  // Ensure that there is a '/' between the prefixes
  if (instancePrefix.endsWith('/') && pluginPrefix[0] === '/') {
    pluginPrefix = pluginPrefix.slice(1)
  } else if (pluginPrefix[0] !== '/') {
    pluginPrefix = '/' + pluginPrefix
  }

  return instancePrefix + pluginPrefix
}
```

**特殊行为**：
- 路由前缀是累加的，不会被 `skip-override` 影响
- 即使使用 `fastify-plugin`，`prefix` 选项仍然会生效
- 这是因为前缀是在创建新实例时处理的，而 `skip-override` 跳过了实例创建

从 `test/plugin.1.test.js:18-40` 可以验证：

```javascript
test('plugin metadata - ignore prefix', (t, testDone) => {
  t.plan(2)
  const fastify = Fastify()

  plugin[Symbol.for('skip-override')] = true
  fastify.register(plugin, { prefix: 'foo' })  // 设置了 prefix

  fastify.inject({
    method: 'GET',
    url: '/'  // 但路由是在根路径，不是 /foo
  }, function (err, res) {
    t.assert.ifError(err)
    t.assert.strictEqual(res.payload, 'hello')
    testDone()
  })

  function plugin (instance, opts, done) {
    instance.get('/', function (request, reply) {
      reply.send('hello')
    })
    done()
  }
})
```

**注意**：这个测试的注释是 "ignore prefix"，但实际行为是：
- 当 `skip-override` 为 true 时，不会调用 `buildRoutePrefix`
- 路由注册在父实例的前缀上下文中
- 所以如果父实例有前缀，路由会继承那个前缀

## 7. 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    fastify.register(plugin, opts)               │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Avvio 内部处理                                │
│  - 管理依赖树                                                     │
│  - 确定加载顺序                                                   │
│  - 调用 avvio.override(old, plugin, opts)                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              override(old, fn, opts) 函数                       │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 1. pluginUtils.registerPlugin.call(old, fn)               │ │
│  │    - 注册插件名称                                             │ │
│  │    - 检查版本兼容性                                           │ │
│  │    - 检查装饰器依赖                                           │ │
│  │    - 检查插件依赖                                             │ │
│  │    - 返回 shouldSkipOverride(fn)                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│              ┌───────────────┴───────────────┐                  │
│              ▼                               ▼                  │
│   ┌──────────────────┐           ┌──────────────────┐          │
│   │ shouldSkipOverride│           │ shouldSkipOverride│          │
│   │     === true     │           │     === false    │          │
│   └────────┬─────────┘           └────────┬─────────┘          │
│            │                               │                     │
│            ▼                               ▼                     │
│   ┌──────────────────┐           ┌──────────────────┐          │
│   │  直接返回 old    │           │ 创建新的封装实例 │          │
│   │                  │           │                  │          │
│   │  1. 不创建新实例 │           │ 1. Object.create │          │
│   │  2. 插件在父scope│           │    (old)         │          │
│   │    中运行        │           │ 2. 构建独立的    │          │
│   │  3. 装饰器直接   │           │    kReply/kRequest│          │
│   │    添加到父实例  │           │ 3. 独立的 hooks/ │          │
│   │                  │           │    schemas等     │          │
│   └──────────────────┘           └────────┬─────────┘          │
│                                            │                     │
│                                            ▼                     │
│                                   ┌──────────────────┐          │
│                                   │  原型链继承父实例 │          │
│                                   │  的所有属性     │          │
│                                   │  但修改是隔离的  │          │
│                                   └──────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    插件函数执行                                  │
│  plugin(instance, opts, done)                                   │
│                                                                  │
│  - 如果 skip-override: instance 就是父实例                      │
│  - 如果封装: instance 是新创建的子实例                          │
│                                                                  │
│  装饰器传播:                                                      │
│  - 父 → 子: 通过原型链/继承                                      │
│  - 子 → 父: 需要 skip-override 打破隔离                         │
└─────────────────────────────────────────────────────────────────┘
```

## 8. 设计意图与最佳实践

### 8.1 为什么需要封装？

1. **模块化隔离**：每个插件可以独立开发、测试，不影响其他部分
2. **路由前缀**：方便地为一组路由添加统一前缀
3. **配置隔离**：不同插件可以有不同的日志级别、错误处理器等
4. **避免命名冲突**：不同插件的装饰器不会相互覆盖
5. **清晰的依赖关系**：装饰器只能向下传播，强制明确的依赖方向

### 8.2 什么时候使用 fastify-plugin？

**应该使用**：
- 定义工具库、数据库连接等需要全局共享的装饰器
- 注册全局 hooks（如错误处理、日志）
- 添加全局 schema 和验证器
- 插件本身就是为了扩展 Fastify 实例

**不应该使用**：
- 业务逻辑插件，应该保持隔离
- 路由组，应该使用 prefix 选项
- 需要独立配置的插件

### 8.3 装饰器传播规则总结

| 场景 | 父 → 子 | 子 → 父 | 说明 |
|------|---------|---------|------|
| 普通插件 | ✅ 是 | ❌ 否 | 默认行为，单向继承 |
| fastify-plugin | ✅ 是 | ✅ 是 | 打破隔离，双向共享 |
| 嵌套 fastify-plugin | ✅ 是 | ✅ 向上一层 | 只传播到直接父 scope |

### 8.4 代码示例

```javascript
const fastify = require('fastify')()
const fp = require('fastify-plugin')

// ========== 场景 1: 全局工具插件（使用 fastify-plugin）==========
fastify.register(fp(function dbPlugin (instance, opts, done) {
  // 这个装饰器会对所有 scope 可见
  instance.decorate('db', {
    query: (sql) => Promise.resolve([])
  })
  
  done()
}))

// ========== 场景 2: 业务模块（不使用 fastify-plugin）==========
fastify.register(function userModule (instance, opts, done) {
  // 可以访问全局的 db
  console.log('User module can access db:', !!instance.db)  // true
  
  // 这个装饰器只在本模块内可见
  instance.decorate('userService', {
    getUsers: () => []
  })
  
  instance.get('/users', (req, reply) => {
    return instance.userService.getUsers()
  })
  
  done()
}, { prefix: '/api' })

// ========== 场景 3: 嵌套模块 ==========
fastify.register(function adminModule (instance, opts, done) {
  // 可以访问 db，但不能访问 userService
  console.log('Admin can access db:', !!instance.db)        // true
  console.log('Admin can access userService:', !!instance.userService)  // false
  
  // 子模块
  instance.register(fp(function sharedAdminUtil (i, o, n) {
    i.decorate('adminUtil', { checkPermission: () => true })
    n()
  }))
  
  instance.after(() => {
    // 因为使用了 fp，adminUtil 传播到了父 scope
    console.log('Admin can access adminUtil:', !!instance.adminUtil)  // true
  })
  
  done()
}, { prefix: '/admin' })

// ========== 根实例视角 ==========
fastify.ready(() => {
  console.log('Root can access:')
  console.log('  db:', !!fastify.db)              // true (fp)
  console.log('  userService:', !!fastify.userService)  // false (隔离)
  console.log('  adminUtil:', !!fastify.adminUtil)      // false (adminModule 是隔离的)
})
```

## 9. 关键源码位置总结

| 功能 | 文件位置 | 关键函数/行号 |
|------|----------|---------------|
| Avvio 集成 | `fastify.js:361-373` | `Avvio()` 初始化，`avvio.override = override` |
| 封装核心 | `lib/plugin-override.js:28-74` | `override()` 函数 |
| skip-override 检查 | `lib/plugin-utils.js:60-62` | `shouldSkipOverride()` |
| 插件注册流程 | `lib/plugin-utils.js:147-154` | `registerPlugin()` |
| 实例装饰器 | `lib/decorate.js:19-34` | `decorate()` |
| Request/Reply 装饰器 | `lib/decorate.js:48-67` | `decorateConstructor()` |
| Request 构建 | `lib/request.js:62-149` | `buildRequest()` |
| Reply 构建 | `lib/reply.js:957-985` | `buildReply()` |
| Hooks 构建 | `lib/hooks.js:73-90` | `buildHooks()` |
| Schema 控制器 | `lib/schema-controller.js:11-38` | `buildSchemaController()` |

## 10. 结论

Fastify 的插件封装机制是其架构的核心优势之一：

1. **基于原型链的隔离**：通过 `Object.create(old)` 创建新实例，结合原型链实现单向继承
2. **多维度隔离**：不仅装饰器，hooks、schemas、路由前缀、日志级别等都是隔离的
3. **可控的打破隔离**：通过 `Symbol.for('skip-override')` 标记和 `fastify-plugin` 库，提供了明确的方式来共享功能
4. **依赖树管理**：基于 `avvio` 实现可靠的插件依赖管理和加载顺序

这种设计使得 Fastify 既能够支持大型应用的模块化架构，又能够灵活地共享公共组件，是其高性能和可扩展性的重要基础。
