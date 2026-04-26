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

#### 6.2.1 SchemaController 的完整结构

让我们深入分析 `lib/schema-controller.js` 中 `SchemaController` 类的完整结构：

```javascript
class SchemaController {
  constructor (parent, options) {
    this.opts = options || parent?.opts
    this.addedSchemas = false  // 标记是否添加了新 schema

    this.compilersFactory = this.opts.compilersFactory

    if (parent) {
      // 有父 scope：复制父 schemas，继承 compiler 配置
      this.schemaBucket = this.opts.bucket(parent.getSchemas())  // 复制父 schema
      this.validatorCompiler = parent.getValidatorCompiler()
      this.serializerCompiler = parent.getSerializerCompiler()
      this.isCustomValidatorCompiler = parent.isCustomValidatorCompiler
      this.isCustomSerializerCompiler = parent.isCustomSerializerCompiler
      this.parent = parent  // 保留父引用（用于向上追溯）
    } else {
      // 根 scope：创建新的 schema 存储
      this.schemaBucket = this.opts.bucket()
      this.isCustomValidatorCompiler = this.opts.isCustomValidatorCompiler || false
      this.isCustomSerializerCompiler = this.opts.isCustomSerializerCompiler || false
    }
  }

  // Bucket 接口 - 管理 schema 定义
  add (schema) {
    this.addedSchemas = true  // 标记有新 schema 添加
    return this.schemaBucket.add(schema)
  }

  getSchema (schemaId) {
    return this.schemaBucket.getSchema(schemaId)
  }

  getSchemas () {
    return this.schemaBucket.getSchemas()
  }

  // Compiler 管理 - 用于编译 schema 为验证/序列化函数
  setValidatorCompiler (validatorCompiler) {
    this.compilersFactory = Object.assign(
      {},
      this.compilersFactory,
      { buildValidator: () => validatorCompiler })
    this.validatorCompiler = validatorCompiler
    this.isCustomValidatorCompiler = true
  }

  setSerializerCompiler (serializerCompiler) {
    this.compilersFactory = Object.assign(
      {},
      this.compilersFactory,
      { buildSerializer: () => serializerCompiler })
    this.serializerCompiler = serializerCompiler
    this.isCustomSerializerCompiler = true
  }

  // 向上追溯获取 compiler（关键！）
  getValidatorCompiler () {
    return this.validatorCompiler || (this.parent && this.parent.getValidatorCompiler())
  }

  getSerializerCompiler () {
    return this.serializerCompiler || (this.parent && this.parent.getSerializerCompiler())
  }

  getSerializerBuilder () {
    return this.compilersFactory.buildSerializer || (this.parent && this.parent.getSerializerBuilder())
  }

  getValidatorBuilder () {
    return this.compilersFactory.buildValidator || (this.parent && this.parent.getValidatorBuilder())
  }

  // 在 preReady 阶段调用 - 实际编译
  setupValidator (serverOptions) {
    const isReady = this.validatorCompiler !== undefined && !this.addedSchemas
    if (isReady) {
      return
    }
    // 使用当前 schemaBucket 中的所有 schemas 来编译 validator
    this.validatorCompiler = this.getValidatorBuilder()(this.schemaBucket.getSchemas(), serverOptions.ajv)
  }

  setupSerializer (serverOptions) {
    const isReady = this.serializerCompiler !== undefined && !this.addedSchemas
    if (isReady) {
      return
    }
    this.serializerCompiler = this.getSerializerBuilder()(this.schemaBucket.getSchemas(), serverOptions.serializerOpts)
  }
}
```

#### 6.2.2 SchemaController 的隔离与继承机制

SchemaController 的设计体现了**隔离与继承的平衡**：

##### 1. Schema 定义的隔离（schemaBucket）

**隔离机制**：
- 每个 scope 的 `schemaBucket` 是**独立的 `Schemas` 实例**
- 创建时通过 `bucket(parent.getSchemas())` **复制父 scope 的 schemas**
- 子 scope 通过 `addSchema()` 添加的新 schema 只会添加到自己的 `schemaBucket`
- 父 scope 无法看到子 scope 新增的 schema

**源码证据** (`lib/plugin-override.js:67`)：
```javascript
instance[kSchemaController] = SchemaController.buildSchemaController(old[kSchemaController])
```

当创建新的封装实例时，会调用 `buildSchemaController(parentSchemaCtrl)`：

```javascript
// lib/schema-controller.js:11-38
function buildSchemaController (parentSchemaCtrl, opts) {
  if (parentSchemaCtrl) {
    return new SchemaController(parentSchemaCtrl, opts)  // 有父
  }
  // ... 根实例创建
}
```

**示例**：
```javascript
const fastify = Fastify()

// Root scope 添加 schema
fastify.addSchema({ $id: 'rootSchema', type: 'object' })

fastify.register(function childPlugin (instance, opts, done) {
  // 此时 child scope 的 schemaBucket:
  // - 包含 rootSchema（复制自父）
  // - 独立的存储
  
  instance.addSchema({ $id: 'childSchema', type: 'object' })
  
  // childSchema 只在 child scope 可见
  console.log(instance.getSchema('childSchema'))  // 存在
  console.log(fastify.getSchema('childSchema'))   // undefined（隔离）
  
  done()
})
```

##### 2. Compiler 的继承机制

与 schema 定义的**隔离**不同，compiler 是**可继承**的：

**继承机制**：
- `getValidatorCompiler()` 会**向上追溯 parent**：
  ```javascript
  getValidatorCompiler () {
    return this.validatorCompiler || (this.parent && this.parent.getValidatorCompiler())
  }
  ```
- 如果当前 scope 没有自己设置 `validatorCompiler`，则使用父 scope 的
- 这允许子 scope 共享父 scope 的自定义 compiler

**setupValidator/setupSerializer 的调用时机**：
- 在 **preReady 阶段**调用
- 用于根据当前 `schemaBucket` 中的所有 schemas 重新编译
- 只有当 `addedSchemas === true`（添加了新 schema）时才会重新编译

**测试用例验证** (`test/schema-feature.test.js:1669-1754`)：
```javascript
test('setSchemaController: Inherits correctly parent schemas with a customized validator instance', async t => {
  const server = Fastify()
  server.addSchema({ $id: 'some', type: 'array', items: { type: 'string' } })
  server.addSchema({ $id: 'error_response', type: 'object', ... })

  server.register((instance, _, done) => {
    instance.setSchemaController({
      compilersFactory: {
        buildValidator: function (externalSchemas) {
          // externalSchemas 包含父 scope 的 schemas！
          const schemaKeys = Object.keys(externalSchemas)
          t.assert.strictEqual(schemaKeys.length, 2, 'Contains same number of schemas')
          t.assert.deepStrictEqual([someSchema, errorResponseSchema], Object.values(externalSchemas))
          // ...
        }
      }
    })

    instance.get('/', {
      schema: {
        querystring: {
          type: 'object',
          properties: {
            msg: { $ref: 'some#' }  // 引用父 scope 的 schema
          }
        }
      }
    }, (req, reply) => { reply.send({ noop: 'noop' }) })

    done()
  })
})
```

#### 6.2.3 SchemaController 与 Scope 的关系

| 维度 | 隔离 | 继承 | 说明 |
|------|------|------|------|
| **schemaBucket（schema 定义）** | ✅ 是 | ✅ 创建时复制 | 每个 scope 独立存储，创建时复制父 |
| **validatorCompiler** | ✅ 可覆盖 | ✅ 向上追溯 | `getValidatorCompiler()` 会找 parent |
| **serializerCompiler** | ✅ 可覆盖 | ✅ 向上追溯 | `getSerializerCompiler()` 会找 parent |
| **compilersFactory** | ✅ 可覆盖 | ✅ 继承 opts | 通过 `parent?.opts` 继承 |

#### 6.2.4 fastify-plugin 对 SchemaController 的影响

| 场景 | 普通插件 | fastify-plugin |
|------|----------|----------------|
| **kSchemaController** | 新创建的 `SchemaController` 实例 | 共享父 scope 的 `SchemaController` |
| **schemaBucket** | 复制父 schemas，独立存储 | 直接使用父的 schemaBucket |
| **addSchema 影响** | 只在子 scope 可见 | 对父 scope 也可见 |
| **validatorCompiler** | 可继承，可覆盖 | 直接使用父的 |

**关键差异**：
- 普通插件：`override` 函数创建新的 `SchemaController` 实例
- fastify-plugin：`override` 函数直接返回 `old`，共享父的 `kSchemaController`

#### 6.2.5 Schema 隔离流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SchemaController 在 Scope 树中的隔离与继承                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  插件注册时（override 函数）：                                               │
│  ─────────────────────────────                                               │
│                                                                              │
│  检查 skip-override:                                                         │
│  ┌─────────────────┐                     ┌─────────────────┐               │
│  │ skip-override   │                     │ skip-override   │               │
│  │ === false       │                     │ === true        │               │
│  └────────┬────────┘                     └────────┬────────┘               │
│           │                                        │                        │
│           ▼                                        ▼                        │
│  ┌─────────────────────────────┐      ┌─────────────────────────────┐    │
│  │ 创建新的 SchemaController    │      │ 直接使用父的 SchemaController │    │
│  │                             │      │                             │    │
│  │ this.schemaBucket =         │      │ 共享父的 schemaBucket        │    │
│  │   bucket(parent.getSchemas())│      │ addSchema 对父也可见        │    │
│  │                             │      │                             │    │
│  │ 复制父的 schemas 定义        │      │ validatorCompiler 直接继承   │    │
│  │ 但子 scope 新增的 schema     │      │ serializerCompiler 直接继承  │    │
│  │ 不会传播到父                 │      │                             │    │
│  └─────────────────────────────┘      └─────────────────────────────┘    │
│                                                                              │
│  Compiler 继承（getValidatorCompiler）：                                    │
│  ─────────────────────────────────────────────────────────                  │
│                                                                              │
│  childSchemaController.getValidatorCompiler():                             │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────┐                                │
│  │ this.validatorCompiler ||                │                                │
│  │ (this.parent && this.parent.getValidatorCompiler()) │                    │
│  └───────────────┬─────────────────────────┘                                │
│                  │                                                            │
│        ┌─────────┴─────────┐                                                  │
│        ▼                   ▼                                                  │
│  ┌────────────┐    ┌────────────┐                                            │
│  │ 有自己的    │    │ 向上追溯    │                                            │
│  │ compiler   │    │ parent     │                                            │
│  └────────────┘    └────────────┘                                            │
│                                                                              │
│  这意味着：                                                                   │
│  - 子 scope 可以共享父 scope 的自定义 compiler                              │
│  - 子 scope 可以通过 setValidatorCompiler 覆盖                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Content-Type Parser 隔离

Fastify 通过 `kContentTypeParser` Symbol 管理每个 scope 的 body parser，这是请求处理流程中的关键组件——它负责将 HTTP 请求的 body 解析为 `request.body` 对象。

#### 6.3.1 ContentTypeParser 的完整结构

`lib/content-type-parser.js:33-42` 中的 `ContentTypeParser` 构造函数：

```javascript
function ContentTypeParser (bodyLimit, onProtoPoisoning, onConstructorPoisoning) {
  this[kDefaultJsonParse] = getDefaultJsonParser(onProtoPoisoning, onConstructorPoisoning)
  // 使用 Map 而不是普通对象，避免原型劫持攻击
  this.customParsers = new Map()
  // 默认注册的 parsers
  this.customParsers.set('application/json', new Parser(true, false, bodyLimit, this[kDefaultJsonParse]))
  this.customParsers.set('text/plain', new Parser(true, false, bodyLimit, defaultPlainTextParser))
  // 字符串类型的 parser 列表（用于精确匹配）
  this.parserList = ['application/json', 'text/plain']
  // 正则表达式类型的 parser 列表
  this.parserRegExpList = []
  // 缓存，提高查找效率
  this.cache = new Fifo(100)
}
```

**核心属性**：

| 属性 | 类型 | 作用 |
|------|------|------|
| `customParsers` | `Map` | content-type → `Parser` 实例的映射 |
| `parserList` | `Array` | 字符串类型的 parser 列表（精确匹配） |
| `parserRegExpList` | `Array` | 正则表达式类型的 parser 列表 |
| `cache` | `Fifo` | 100 条容量的 FIFO 缓存，提高查找效率 |
| `kDefaultJsonParse` | `Function` | 默认的 JSON 解析器（安全配置） |

**Parser 类的结构** (`lib/content-type-parser.js:328-333`)：
```javascript
function Parser (asString, asBuffer, bodyLimit, fn) {
  this.asString = asString    // 是否以字符串形式读取 body
  this.asBuffer = asBuffer    // 是否以 Buffer 形式读取 body
  this.bodyLimit = bodyLimit  // body 大小限制
  this.fn = fn                // 解析函数
}
```

#### 6.3.2 隔离与继承机制

与 `kHooks`、`kRequest`、`kSchemaController` 类似，`kContentTypeParser` 在创建新的封装实例时会被**复制**，实现隔离。

**源码证据** (`lib/plugin-override.js:46`)：
```javascript
instance[kContentTypeParser] = ContentTypeParser.helpers.buildContentTypeParser(instance[kContentTypeParser])
```

**`buildContentTypeParser` 函数** (`lib/content-type-parser.js:335-342`)：
```javascript
function buildContentTypeParser (c) {
  const contentTypeParser = new ContentTypeParser()
  contentTypeParser[kDefaultJsonParse] = c[kDefaultJsonParse]
  // 复制 customParsers（通过 Map 构造函数复制 entries）
  contentTypeParser.customParsers = new Map(c.customParsers.entries())
  // 复制 parserList（通过 slice()）
  contentTypeParser.parserList = c.parserList.slice()
  // 复制 parserRegExpList（通过 slice()）
  contentTypeParser.parserRegExpList = c.parserRegExpList.slice()
  return contentTypeParser
}
```

**关键机制**：
1. **创建新的 ContentTypeParser 实例**：子 scope 有自己独立的 `kContentTypeParser`
2. **复制 customParsers**：通过 `new Map(c.customParsers.entries())` 复制父 scope 的所有 parser
3. **复制 parserList**：通过 `slice()` 复制字符串类型的 parser 列表
4. **复制 parserRegExpList**：通过 `slice()` 复制正则表达式类型的 parser 列表

**这意味着**：
- 子 scope **继承**父 scope 已注册的所有 parser
- 子 scope 通过 `addContentTypeParser` 添加的新 parser **只在自己的 scope 内可见**
- 父 scope 无法看到子 scope 新增的 parser

**示例**：
```javascript
const fastify = Fastify()

// Root scope 注册自定义 parser
fastify.addContentTypeParser('application/xml', { parseAs: 'text' }, function (req, body, done) {
  // XML 解析逻辑
  done(null, { xml: body })
})

fastify.register(function childPlugin (instance, opts, done) {
  // 子 scope 继承了父 scope 的 parser
  // 可以使用 'application/xml' parser
  
  // 添加新的 parser（只在子 scope 可见）
  instance.addContentTypeParser('application/vnd.custom', { parseAs: 'buffer' }, function (req, body, done) {
    done(null, { custom: body })
  })
  
  console.log(instance.hasContentTypeParser('application/xml'))     // true（继承）
  console.log(instance.hasContentTypeParser('application/vnd.custom')) // true（新增）
  
  done()
})

fastify.ready(() => {
  console.log(fastify.hasContentTypeParser('application/xml'))        // true
  console.log(fastify.hasContentTypeParser('application/vnd.custom')) // false（隔离）
})
```

#### 6.3.3 运行时如何使用 ContentTypeParser

**Context 中的引用** (`lib/context.js:48`)：
```javascript
this.contentTypeParser = server[kContentTypeParser]
```

Context 直接引用创建它的 scope 的 `kContentTypeParser`。这意味着：
- 路由的 Context 使用创建该路由的 scope 的 parser
- 子 scope 的路由使用子 scope 的 parser（继承了父的 + 自己新增的）
- 父 scope 的路由使用父 scope 的 parser（看不到子 scope 新增的）

**请求处理流程中的使用** (`lib/handle-request.js:51, 63`)：
```javascript
// 1. 如果没有 Content-Type 头部，使用空字符串查找
request[kRouteContext].contentTypeParser.run('', handler, request, reply)

// 2. 有 Content-Type 头部时，使用解析后的 mediaType
request[kRouteContext].contentTypeParser.run(request[kRequestContentType].toString(), handler, request, reply)
```

**`ContentTypeParser.prototype.run` 函数** (`lib/content-type-parser.js:185-231`)：
```javascript
ContentTypeParser.prototype.run = function (contentType, handler, request, reply) {
  // 1. 查找 parser（精确匹配 → mediaType 匹配 → 正则匹配 → 通配符）
  const parser = this.getParser(contentType)

  if (parser === undefined) {
    if (request.is404 === true) {
      handler(request, reply)  // 404 路由不强制要求 parser
      return
    }
    // 返回 415 Unsupported Media Type
    reply[kReplyIsError] = true
    reply.send(new FST_ERR_CTP_INVALID_MEDIA_TYPE())
    return
  }

  // 2. 执行 parser
  if (parser.asString === true || parser.asBuffer === true) {
    // 先读取原始 body，再调用 parser.fn
    rawBody(request, reply, reply[kRouteContext]._parserOptions, parser, done)
    return
  }

  // 直接调用 parser.fn（流模式）
  const result = parser.fn(request, request[kRequestPayloadStream], done)
  if (result && typeof result.then === 'function') {
    result.then(body => { done(null, body) }, done)
  }
}
```

#### 6.3.4 skip-override 对 ContentTypeParser 的影响

| 场景 | 普通插件 | fastify-plugin |
|------|----------|----------------|
| **kContentTypeParser** | 新创建的 `ContentTypeParser` 实例 | 共享父 scope 的 `ContentTypeParser` |
| **buildContentTypeParser** | ✅ 调用，复制父 parsers | ❌ 不调用 |
| **addContentTypeParser 影响** | 只在子 scope 可见 | 对父 scope 也可见 |
| **路由使用的 parser** | 子 scope 的 parser（继承+新增） | 父 scope 的 parser |

**源码证据**：
- 普通插件：`override` 函数调用 `buildContentTypeParser` 创建新实例
- fastify-plugin：`override` 函数直接返回 `old`，**不调用 `buildContentTypeParser`**

**实际效果**：
```javascript
const fastify = Fastify()
const fp = require('fastify-plugin')

// ========== 场景 A: 普通插件 ==========
fastify.register(function normalPlugin (instance, opts, done) {
  // 1. override 函数调用 buildContentTypeParser
  // 2. instance[kContentTypeParser] 是新创建的实例
  // 3. 复制了 fastify[kContentTypeParser] 的所有 parser
  
  instance.addContentTypeParser('application/normal', { parseAs: 'text' }, (req, body, done) => {
    done(null, { type: 'normal', data: body })
  })
  
  // 这个 parser 只在 normalPlugin 内可见
  // fastify.hasContentTypeParser('application/normal') === false
  
  done()
})

// ========== 场景 B: fastify-plugin ==========
fastify.register(fp(function sharedPlugin (instance, opts, done) {
  // 1. override 函数直接返回 old（fastify 实例）
  // 2. instance === fastify
  // 3. instance[kContentTypeParser] === fastify[kContentTypeParser]
  
  instance.addContentTypeParser('application/shared', { parseAs: 'text' }, (req, body, done) => {
    done(null, { type: 'shared', data: body })
  })
  
  // 这个 parser 在所有 scope 可见
  // fastify.hasContentTypeParser('application/shared') === true
  
  done()
}))
```

#### 6.3.5 addContentTypeParser API

**添加 parser** (`lib/content-type-parser.js:344-364`)：
```javascript
function addContentTypeParser (contentType, opts, parser) {
  if (this[kState].started) {
    throw new FST_ERR_CTP_INSTANCE_ALREADY_STARTED('addContentTypeParser')
  }
  // ... 参数处理
  if (Array.isArray(contentType)) {
    contentType.forEach((type) => this[kContentTypeParser].add(type, opts, parser))
  } else {
    this[kContentTypeParser].add(contentType, opts, parser)
  }
  return this
}
```

**注意**：`addContentTypeParser` 只能在服务器启动前调用（`kState.started` 为 false）。

**其他 API**：
- `hasContentTypeParser(contentType)`：检查是否存在
- `removeContentTypeParser(contentType)`：移除
- `removeAllContentTypeParsers()`：移除所有

#### 6.3.6 ContentTypeParser 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              ContentTypeParser 在 Scope 树中的隔离与继承                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  插件注册时（override 函数）：                                               │
│  ─────────────────────────────                                               │
│                                                                              │
│  检查 skip-override:                                                         │
│  ┌─────────────────┐                     ┌─────────────────┐               │
│  │ skip-override   │                     │ skip-override   │               │
│  │ === false       │                     │ === true        │               │
│  └────────┬────────┘                     └────────┬────────┘               │
│           │                                        │                        │
│           ▼                                        ▼                        │
│  ┌─────────────────────────────┐      ┌─────────────────────────────┐    │
│  │ 调用 buildContentTypeParser │      │ 直接使用父的                │    │
│  │                             │      │ kContentTypeParser          │    │
│  │ 创建新的 ContentTypeParser   │      │                             │    │
│  │ 实例：                        │      │ addContentTypeParser 直接  │    │
│  │                             │      │ 在父实例上执行              │    │
│  │ customParsers = new Map(    │      │ 所有子 scope 都能看到       │    │
│  │   parent.customParsers.entries())│      │ 新增的 parser              │    │
│  │ parserList = parent.parserList.slice()│      │                             │    │
│  │ parserRegExpList = parent.parserRegExpList.slice()│      │                             │    │
│  │                             │      │                             │    │
│  │ 新增的 parser 只在           │      │                             │    │
│  │ 当前 scope 可见              │      │                             │    │
│  └─────────────────────────────┘      └─────────────────────────────┘    │
│                                                                              │
│  Context 中的使用：                                                          │
│  ───────────────────                                                         │
│                                                                              │
│  Context 构造函数中：                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ this.contentTypeParser = server[kContentTypeParser]                  │  │
│  │                                                                       │  │
│  │ 路由的 Context 使用创建该路由的 scope 的 parser：                      │  │
│  │ - 子 scope 路由：使用子 scope 的 parser（继承父的 + 自己新增的）      │  │
│  │ - 父 scope 路由：使用父 scope 的 parser（看不到子 scope 新增的）      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  运行时请求处理（preParsing 阶段）：                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ contentTypeParser.run(contentType, handler, request, reply)        │  │
│  │                                                                       │  │
│  │ 1. getParser(contentType) 查找 parser：                              │  │
│  │    - 精确匹配（带 parameters）                                        │  │
│  │    - 精确匹配（mediaType 只）                                         │  │
│  │    - 正则表达式匹配                                                   │  │
│  │    - 通配符 ''（* 注册）                                              │  │
│  │                                                                       │  │
│  │ 2. 执行 parser：                                                       │  │
│  │    - asString/asBuffer: 先读取 body，再调用 parser.fn                │  │
│  │    - 流模式: 直接调用 parser.fn(request, stream, done)               │  │
│  │                                                                       │  │
│  │ 3. 结果赋值给 request.body                                             │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 路由前缀隔离

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

## 7. Context 对象 - Scope 状态的运行时快照

前面的章节分析了 Fastify 的 scope 隔离机制、装饰器传播和 hooks 传播。但这些机制最终要落地到**请求处理**上——这就是 `Context` 对象的作用。

`Context` 是每个路由的**运行时上下文**，它在 `preReady` 阶段将 scope 积累的 hooks、decorators、schemas 等信息**固化**成一个独立对象，请求到来时直接从这个 Context 读取配置，不再访问 scope。

### 7.1 Context 对象的结构

`lib/context.js` 定义了 Context 构造函数：

```javascript
function Context ({
  schema,
  handler,
  config,
  requestIdLogLabel,
  childLoggerFactory,
  errorHandler,
  bodyLimit,
  logLevel,
  logSerializers,
  attachValidation,
  validatorCompiler,
  serializerCompiler,
  replySerializer,
  schemaErrorFormatter,
  exposeHeadRoute,
  prefixTrailingSlash,
  server,
  isFastify,
  handlerTimeout
}) {
  // 核心属性
  this.schema = schema
  this.handler = handler
  this.Reply = server[kReply]
  this.Request = server[kRequest]
  this.contentTypeParser = server[kContentTypeParser]
  
  // Hooks 链（初始为 null，preReady 阶段固化）
  this.onRequest = null
  this.onSend = null
  this.onError = null
  this.onTimeout = null
  this.preHandler = null
  this.onResponse = null
  this.preSerialization = null
  this.onRequestAbort = null
  
  // 配置属性
  this.config = config
  this.errorHandler = errorHandler || server[kErrorHandler]
  this.requestIdLogLabel = requestIdLogLabel || server[kOptions].requestIdLogLabel
  this.childLoggerFactory = childLoggerFactory || server[kChildLoggerFactory]
  this.logLevel = logLevel || server[kLogLevel]
  this.logSerializers = logSerializers
  this.handlerTimeout = handlerTimeout || server[kHandlerTimeout] || 0
  this.attachValidation = attachValidation
  
  // Schema 相关
  this.validatorCompiler = validatorCompiler || null
  this.serializerCompiler = serializerCompiler || null
  this.schemaErrorFormatter = schemaErrorFormatter || server[kSchemaErrorFormatter] || defaultSchemaErrorFormatter
  
  // 其他
  this.server = server
  // ...
}
```

**Context 对象的属性分类**：

| 分类 | 属性 | 来源 | 说明 |
|------|------|------|------|
| **核心** | `schema`, `handler`, `config` | 路由定义 | 路由的 schema、处理函数、配置 |
| **构造函数** | `Request`, `Reply` | `server[kRequest]`, `server[kReply]` | 当前 scope 的 Request/Reply 构造函数 |
| **Hooks 链** | `onRequest`, `preParsing`, `preValidation`, `preHandler`, `preSerialization`, `onSend`, `onResponse`, `onError`, `onTimeout`, `onRequestAbort` | 初始为 `null`，preReady 阶段固化 | 合并后的 hooks 数组 |
| **配置** | `logLevel`, `logSerializers`, `errorHandler`, `handlerTimeout`, `bodyLimit` | 路由定义 + 继承自 server | 日志级别、错误处理器、超时等 |
| **Schema** | `validatorCompiler`, `serializerCompiler`, `schemaErrorFormatter` | 路由定义 + 继承自 server | 验证/序列化编译器 |
| **引用** | `server`, `contentTypeParser` | 继承自 server | Fastify 实例引用、内容类型解析器 |

**关键点**：
- Hooks 链属性**初始为 `null`**，在 **preReady 阶段**才被赋值
- `Request` 和 `Reply` 直接引用 `server[kRequest]` 和 `server[kReply]`（当前 scope 的构造函数）
- `server` 是创建这个 Context 的 scope 实例的引用

### 7.2 Context 的创建与固化流程

Context 的创建和固化分为**三个阶段**：

#### 阶段 1: 同步创建 Context（路由注册时）

在 `lib/route.js:340-359` 的 `addNewRoute` 函数中：

```javascript
const context = new Context({
  schema: opts.schema,
  handler: opts.handler.bind(this),  // 绑定到当前 scope
  config,
  errorHandler: opts.errorHandler,
  childLoggerFactory: opts.childLoggerFactory,
  bodyLimit: opts.bodyLimit,
  logLevel: opts.logLevel,
  logSerializers: opts.logSerializers,
  attachValidation: opts.attachValidation,
  schemaErrorFormatter: opts.schemaErrorFormatter,
  replySerializer: this[kReplySerializerDefault],
  validatorCompiler: opts.validatorCompiler,
  serializerCompiler: opts.serializerCompiler,
  exposeHeadRoute: shouldExposeHead,
  prefixTrailingSlash: (opts.prefixTrailingSlash || 'both'),
  server: this,  // 当前 scope 实例
  isFastify,
  handlerTimeout: opts.handlerTimeout
})

// 立即注册到路由器
router.on(opts.method, opts.url, { constraints }, routeHandler, context)
```

**此时的 Context 状态**：
- `handler` 已绑定到当前 scope（`this`）
- `Request` = `server[kRequest]`（当前 scope 的 Request 构造函数）
- `Reply` = `server[kReply]`（当前 scope 的 Reply 构造函数）
- **所有 hooks 属性都是 `null`**
- 路由已注册到 `find-my-way` 路由器，Context 作为路由的用户数据

#### 阶段 2: this.after 回调中补充属性

`lib/route.js:380-449`：

```javascript
this.after((notHandledErr, done) => {
  // 补充 context 的属性
  context.errorHandler = opts.errorHandler
    ? buildErrorHandler(this[kErrorHandler], opts.errorHandler)
    : this[kErrorHandler]
  context._parserOptions.limit = opts.bodyLimit || null
  context.logLevel = opts.logLevel
  context.logSerializers = opts.logSerializers
  context.attachValidation = opts.attachValidation
  context[kReplySerializerDefault] = this[kReplySerializerDefault]
  context.schemaErrorFormatter =
    opts.schemaErrorFormatter || this[kSchemaErrorFormatter] || context.schemaErrorFormatter

  // 注册 preReady 回调（关键！）
  avvio.once('preReady', () => {
    // 第 3 阶段在这里执行
  })

  done(notHandledErr)
})
```

**关键点**：
- `this.after` 回调在**当前插件及其子插件加载完成后**执行
- 此时注册 `avvio.once('preReady')` 回调

#### 阶段 3: preReady 阶段 - 状态固化

`lib/route.js:394-446` 是**最关键的阶段**：

```javascript
avvio.once('preReady', () => {
  // ========== 操作 1: 固化 hooks 链 ==========
  for (const hook of lifecycleHooks) {
    const toSet = this[kHooks][hook]        // 实例级 hooks（来自 scope）
      .concat(opts[hook] || [])             // 路由级 hooks（来自路由定义）
      .map(h => h.bind(this))                // 绑定到当前 scope
    context[hook] = toSet.length ? toSet : null
  }

  // ========== 操作 2: 优化 Request/Reply 构造函数 ==========
  while (!context.Request[kHasBeenDecorated] && context.Request.parent) {
    context.Request = context.Request.parent
  }
  while (!context.Reply[kHasBeenDecorated] && context.Reply.parent) {
    context.Reply = context.Reply.parent
  }

  // ========== 操作 3: 设置 404 Context ==========
  fourOhFour.setContext(this, context)

  // ========== 操作 4: 编译 schema ==========
  if (opts.schema) {
    context.schema = normalizeSchema(context.schema, this.initialConfig)
    // ... 编译验证和序列化 schema
  }
})
```

这是整个 Context 机制的**核心**，让我们逐一分析：

### 7.3 preReady 阶段的关键操作深度分析

#### 操作 1: Hooks 链的固化

```javascript
for (const hook of lifecycleHooks) {
  const toSet = this[kHooks][hook]        // scope 积累的 hooks
    .concat(opts[hook] || [])             // 路由定义的 hooks
    .map(h => h.bind(this))                // 绑定到当前 scope
  context[hook] = toSet.length ? toSet : null
}
```

**重要发现**：

1. **合并来源**：
   - `this[kHooks][hook]`：当前 scope 积累的所有 hooks（包括从父 scope 传播来的）
   - `opts[hook]`：路由定义时指定的路由级 hooks（如 `fastify.get('/', { preHandler: [...] }, handler)`）

2. **顺序**：实例级 hooks **在前**，路由级 hooks **在后**
   - 执行顺序：父 scope hooks → 当前 scope hooks → 路由级 hooks

3. **绑定**：所有 hooks 通过 `.bind(this)` 绑定到**当前 scope**
   - 这就是为什么在 hook 中 `this` 指向创建该路由的 scope 实例

4. **固化时机**：此时 `this[kHooks]` 已经包含了所有通过 `addHook` 添加的 hooks（包括从父 scope 传播来的）
   - 一旦固化，**后续添加的 hooks 不会影响已注册的路由**

#### 操作 2: Request/Reply 构造函数的优化（关键！）

这是一个**性能优化**，但对于理解装饰器的传播机制至关重要：

```javascript
while (!context.Request[kHasBeenDecorated] && context.Request.parent) {
  context.Request = context.Request.parent
}
while (!context.Reply[kHasBeenDecorated] && context.Reply.parent) {
  context.Reply = context.Reply.parent
}
```

**背景回顾**：
- 每个 scope 都有自己的 `kRequest` 和 `kReply`（通过 `buildRequest` 和 `buildReply` 创建）
- 这些构造函数形成一个链表：`childRequest.parent = parentRequest`
- `kHasBeenDecorated` 标记表示该构造函数是否有自己的装饰器

**优化逻辑**：
- 如果当前 scope 的 `Request` 构造函数**没有被装饰**（`kHasBeenDecorated` 为 false）
- 并且存在父构造函数（`parent`）
- 则**直接使用父构造函数**，跳过当前层级

**目的**：
- 避免运行时每次创建 Request/Reply 实例时都要遍历原型链
- 如果没有自定义装饰器，直接使用最近被装饰过的祖先的构造函数

**实际效果**：
- 路由注册在子 scope，但 Context 中的 `Request`/`Reply` **可能是父 scope 的构造函数**
- 这确保了即使路由在深层嵌套的 scope 中，也能**正确访问所有祖先的装饰器**

**示例**：
```javascript
const fastify = Fastify()

// Root scope - 添加装饰器
fastify.decorateRequest('rootProp', 'root value')
// 此时 fastify[kRequest][kHasBeenDecorated] = true

fastify.register(function childScope (instance, opts, done) {
  // 子 scope 的 Request 构造函数：
  // - kHasBeenDecorated = false（没有新装饰器）
  // - parent = fastify[kRequest]
  
  instance.get('/test', (req, reply) => {
    // 路由注册在 childScope
    // 但 Context.Request 会被优化为 fastify[kRequest]
    // 因为 childScope 的 Request 没有被装饰
    console.log(req.rootProp)  // 'root value' - 可以访问！
    reply.send({ ok: true })
  })
  
  done()
})
```

**这解释了为什么装饰器能在子 scope 中工作**：
- Context 中的 `Request`/`Reply` 是**已优化的版本**
- 它们指向**最近被装饰过的祖先的构造函数**
- 运行时创建的 Request/Reply 实例直接包含所有必要的装饰器

#### 操作 3: 设置 404 Context

```javascript
fourOhFour.setContext(this, context)
```

这确保了 404 路由也能使用正确的 scope 配置（hooks、logLevel 等）。

#### 操作 4: 编译 Schema（核心！）

这是 schema 从 scope 的 `SchemaController` 固化到 Context 的关键步骤。让我们深入分析完整流程。

##### 4.1 编译前的准备：normalizeSchema

首先，路由定义的 schema 会被规范化：

```javascript
context.schema = normalizeSchema(context.schema, this.initialConfig)
```

`normalizeSchema` (`lib/schemas.js:58-115`) 的主要工作：
1. **标记 `kSchemaVisited`**：防止重复处理
2. **alias `query` to `querystring`**：兼容两种写法
3. **处理 Fluent Schema**：`schema.valueOf()` 转换为普通对象
4. **处理 headers 大小写不敏感**：在编译时处理
5. **验证 response schema 格式**：检查 status code 格式

##### 4.2 编译验证 Schema：compileSchemasForValidation

`lib/validation.js:57-116` 中的 `compileSchemasForValidation` 函数：

```javascript
function compileSchemasForValidation (context, compile, isCustom) {
  const { schema } = context
  if (!schema) {
    return
  }

  const { method, url } = context.config || {}

  // 1. 编译 headers schema（特殊处理：大小写不敏感）
  const headers = schema.headers
  if (headers && (isCustom || Object.getPrototypeOf(headers) !== Object.prototype)) {
    // 自定义 compiler（如 Joi、Typebox）
    context[headersSchema] = compile({ schema: headers, method, url, httpPart: 'headers' })
  } else if (headers) {
    // 标准 AJV 编译器：处理 headers 大小写不敏感
    const headersSchemaLowerCase = {}
    Object.keys(headers).forEach(k => { headersSchemaLowerCase[k] = headers[k] })
    if (headersSchemaLowerCase.required instanceof Array) {
      headersSchemaLowerCase.required = headersSchemaLowerCase.required.map(h => h.toLowerCase())
    }
    if (headers.properties) {
      headersSchemaLowerCase.properties = {}
      Object.keys(headers.properties).forEach(k => {
        headersSchemaLowerCase.properties[k.toLowerCase()] = headers.properties[k]
      })
    }
    context[headersSchema] = compile({ schema: headersSchemaLowerCase, method, url, httpPart: 'headers' })
  }

  // 2. 编译 body schema
  if (schema.body) {
    const contentProperty = schema.body.content
    if (contentProperty) {
      // Content-Type 特定的 schema（如 multipart/form-data）
      const contentTypeSchemas = {}
      for (const contentType of Object.keys(contentProperty)) {
        const contentSchema = contentProperty[contentType].schema
        contentTypeSchemas[contentType] = compile({ schema: contentSchema, method, url, httpPart: 'body', contentType })
      }
      context[bodySchema] = contentTypeSchemas
    } else {
      // 标准 body schema
      context[bodySchema] = compile({ schema: schema.body, method, url, httpPart: 'body' })
    }
  }

  // 3. 编译 querystring schema
  if (schema.querystring) {
    context[querystringSchema] = compile({ schema: schema.querystring, method, url, httpPart: 'querystring' })
  }

  // 4. 编译 params schema
  if (schema.params) {
    context[paramsSchema] = compile({ schema: schema.params, method, url, httpPart: 'params' })
  }
}
```

**关键发现**：
- 编译结果存储在 Context 的 **Symbol 属性**中：
  - `context[kSchemaHeaders]`：headers 验证函数
  - `context[kSchemaBody]`：body 验证函数（可能是对象，按 Content-Type 区分）
  - `context[kSchemaQuerystring]`：querystring 验证函数
  - `context[kSchemaParams]`：params 验证函数

##### 4.3 编译序列化 Schema：compileSchemasForSerialization

`lib/validation.js:18-55` 中的 `compileSchemasForSerialization` 函数：

```javascript
function compileSchemasForSerialization (context, compile) {
  if (!context.schema || !context.schema.response) {
    return
  }
  const { method, url } = context.config || {}
  context[responseSchema] = Object.keys(context.schema.response)
    .reduce(function (acc, statusCode) {
      const schema = context.schema.response[statusCode]
      statusCode = statusCode.toLowerCase()
      
      if (schema.content) {
        // Content-Type 特定的 response schema
        const contentTypesSchemas = {}
        for (const mediaName of Object.keys(schema.content)) {
          const contentSchema = schema.content[mediaName].schema
          contentTypesSchemas[mediaName] = compile({
            schema: contentSchema,
            url,
            method,
            httpStatus: statusCode,
            contentType: mediaName
          })
        }
        acc[statusCode] = contentTypesSchemas
      } else {
        // 标准 response schema（按状态码）
        acc[statusCode] = compile({
          schema,
          url,
          method,
          httpStatus: statusCode
        })
      }

      return acc
    }, {})
}
```

**关键发现**：
- 编译结果存储在 `context[kSchemaResponse]` 中
- 结构是**按状态码索引**的对象：
  ```javascript
  context[kSchemaResponse] = {
    '2xx': compiledFunction,
    '4xx': { 'application/json': compiledFunction, 'text/plain': anotherFunction },
    'default': compiledFunction
  }
  ```

##### 4.4 编译函数的来源

在 `lib/route.js` 中，编译函数的获取顺序：

```javascript
// 验证器：路由定义 > scope 的 SchemaController
const validatorCompiler = opts.validatorCompiler || schemaController.getValidatorCompiler()

// 序列化器：路由定义 > scope 的 SchemaController
const serializerCompiler = opts.serializerCompiler || schemaController.getSerializerCompiler()
```

**SchemaController 的 compiler 向上追溯** (`lib/schema-controller.js:119-133`)：
```javascript
getValidatorCompiler () {
  return this.validatorCompiler || (this.parent && this.parent.getValidatorCompiler())
}

getSerializerCompiler () {
  return this.serializerCompiler || (this.parent && this.parent.getSerializerCompiler())
}
```

这意味着：
- 如果当前 scope 没有自定义 `setValidatorCompiler`，会向上追溯父 scope
- 根 scope 使用默认的 `@fastify/ajv-compiler` 和 `@fastify/fast-json-stringify-compiler`

##### 4.5 编译结果固化到 Context 的 Symbol 属性

让我们查看 `lib/symbols.js` 中定义的 schema 相关 Symbol：

```javascript
const kSchemaHeaders = Symbol('fastify.schemaHeaders')
const kSchemaParams = Symbol('fastify.schemaParams')
const kSchemaQuerystring = Symbol('fastify.schemaQuerystring')
const kSchemaBody = Symbol('fastify.schemaBody')
const kSchemaResponse = Symbol('fastify.schemaResponse')
```

这些 Symbol 属性在编译后直接存储在 Context 中：

| Symbol | 存储内容 | 来源 |
|--------|----------|------|
| `kSchemaHeaders` | headers 验证函数 | `compileSchemasForValidation` |
| `kSchemaBody` | body 验证函数（或对象） | `compileSchemasForValidation` |
| `kSchemaQuerystring` | querystring 验证函数 | `compileSchemasForValidation` |
| `kSchemaParams` | params 验证函数 | `compileSchemasForValidation` |
| `kSchemaResponse` | 按状态码的序列化函数 | `compileSchemasForSerialization` |

**关键设计**：
- 使用 Symbol 是为了**避免属性名冲突**
- 这些属性**直接存储在 Context 实例上**，运行时访问非常快
- **preReady 阶段之后不会再修改**，真正实现了"固化"

##### 4.6 运行时如何使用编译后的 Schema

**验证阶段** (`lib/validation.js:146-201`)：
```javascript
function validate (context, request, execution) {
  // 按顺序验证 params → body → query → headers
  // 从 Context 的 Symbol 属性读取编译好的验证函数
  
  if (runExecution || !execution.skipParams) {
    const params = validateParam(context[paramsSchema], request, 'params')
    if (params) { /* 返回错误 */ }
  }
  // ... body, query, headers 类似
}
```

**序列化阶段** (`lib/schemas.js:145-202`)：
```javascript
function getSchemaSerializer (context, statusCode, contentType) {
  const responseSchemaDef = context[kSchemaResponse]
  if (!responseSchemaDef) {
    return false
  }
  // 按状态码查找：精确匹配 → 通配符 (2xx) → default
  if (responseSchemaDef[statusCode]) {
    // 检查是否有 Content-Type 特定的 schema
    // ...
    return responseSchemaDef[statusCode]
  }
  const fallbackStatusCode = (statusCode + '')[0] + 'xx'  // 404 → '4xx'
  if (responseSchemaDef[fallbackStatusCode]) { /* ... */ }
  if (responseSchemaDef.default) { /* ... */ }
  return false
}
```

##### 4.7 Schema 编译固化完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Schema 编译固化到 Context 的完整流程                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  preReady 阶段触发：                                                         │
│  ─────────────────                                                          │
│                                                                              │
│  avvio.once('preReady', () => {                                             │
│    // 1. 先 setup Compiler（如果需要）                                       │
│    schemaController.setupValidator(this[kOptions])                          │
│    schemaController.setupSerializer(this[kOptions])                         │
│                                                                              │
│    // 2. 编译 Schema 到 Context                                             │
│    if (opts.schema) {                                                        │
│      context.schema = normalizeSchema(context.schema, ...)                  │
│      compileSchemasForValidation(context, compiler, isCustom)              │
│      compileSchemasForSerialization(context, compiler)                      │
│    }                                                                          │
│  })                                                                           │
│                                                                              │
│  详细流程：                                                                   │
│  ──────────                                                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 1: setupValidator / setupSerializer                             │  │
│  │                                                                       │  │
│  │ setupValidator(serverOptions):                                       │  │
│  │   const isReady = this.validatorCompiler !== undefined               │  │
│  │               && !this.addedSchemas                                  │  │
│  │   if (isReady) return  // 不需要重新编译                             │  │
│  │                                                                       │  │
│  │   // 使用当前 schemaBucket 中的所有 schemas 重新编译                 │  │
│  │   this.validatorCompiler =                                            │  │
│  │     this.getValidatorBuilder()(                                       │  │
│  │       this.schemaBucket.getSchemas(),  // 包含父 scope 的           │  │
│  │       serverOptions.ajv                  // + 自己新增的             │  │
│  │     )                                                                 │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 2: compileSchemasForValidation                                  │  │
│  │                                                                       │  │
│  │ 编译 context.schema 中的：                                            │  │
│  │   - schema.headers   → context[kSchemaHeaders]                       │  │
│  │   - schema.body      → context[kSchemaBody]                          │  │
│  │   - schema.querystring → context[kSchemaQuerystring]                 │  │
│  │   - schema.params    → context[kSchemaParams]                        │  │
│  │                                                                       │  │
│  │ 每个都是编译后的函数，可直接调用验证                                   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段 3: compileSchemasForSerialization                               │  │
│  │                                                                       │  │
│  │ 编译 context.schema.response 中的：                                   │  │
│  │   context[kSchemaResponse] = {                                        │  │
│  │     '2xx': compiledFunction,                                          │  │
│  │     '4xx': { 'application/json': compiledFunction },                  │  │
│  │     'default': compiledFunction                                       │  │
│  │   }                                                                    │  │
│  │                                                                       │  │
│  │ 运行时通过 getSchemaSerializer() 按状态码查找                        │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  运行时使用：                                                                 │
│  ────────────                                                                 │
│                                                                              │
│  请求验证阶段（preValidation）：                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ validate(context, request)                                            │  │
│  │   │                                                                    │  │
│  │   ├── validateParam(context[kSchemaParams], request, 'params')      │  │
│  │   ├── validateParam(context[kSchemaBody], request, 'body')          │  │
│  │   ├── validateParam(context[kSchemaQuerystring], request, 'query')  │  │
│  │   └── validateParam(context[kSchemaHeaders], request, 'headers')    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  响应序列化阶段（preSerialization → onSend）：                               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ getSchemaSerializer(context, statusCode, contentType)                │  │
│  │   │                                                                    │  │
│  │   ├── 查找 context[kSchemaResponse][statusCode]                      │  │
│  │   ├── 或查找 context[kSchemaResponse][statusCode[0] + 'xx']         │  │
│  │   └── 或查找 context[kSchemaResponse]['default']                      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

##### 4.8 Schema 编译固化的关键设计要点

| 要点 | 说明 |
|------|------|
| **Symbol 属性存储** | 避免属性名冲突，内部属性不对外暴露 |
| **预编译** | preReady 阶段完成编译，运行时无需动态编译 |
| **按路由独立** | 每个路由的 Context 有独立的编译结果 |
| **Compiler 可继承** | `getValidatorCompiler()` 向上追溯 parent |
| **运行时零开销** | 直接调用编译好的函数，无需查找 scope |

##### 4.9 为什么 Schema 需要独立编译？

1. **Schema 按 scope 隔离**：每个 scope 的 `schemaBucket` 是独立的
   - 父 scope 的 schema 会被复制到子 scope
   - 子 scope 新增的 schema 不会影响父

2. **Compiler 可以自定义**：每个 scope 可以通过 `setValidatorCompiler` 覆盖
   - 根 scope：默认使用 AJV
   - 子 scope：可以使用 Joi、Zod 等其他验证库

3. **路由级别的自定义**：路由定义时可以指定 `validatorCompiler` 和 `serializerCompiler`
   - 优先级：路由定义 > scope 的 SchemaController > 父 scope

**这就是为什么每个路由的 schema 需要独立编译并固化到 Context**——不同的路由可能：
- 属于不同的 scope（schema 定义不同）
- 使用不同的 compiler（验证/序列化逻辑不同）
- 有不同的 schema 定义（每个路由独立）

### 7.4 Context 与 Scope 的关系总结

| 维度 | Scope | Context |
|------|-------|---------|
| **创建时机** | `fastify.register()` 时（`override` 函数） | 路由注册时（`addNewRoute`） |
| **生命周期** | 从注册到服务器关闭 | 从 preReady 固化到路由被移除 |
| **可变性** | 可变（可以继续 `addHook`、`decorate`） | **不可变**（preReady 后固化） |
| **作用范围** | 整个插件及其子插件 | **单个路由** |
| **运行时访问** | 不直接访问 | **请求处理时直接读取** |

**关键关系**：
- **Context 是 Scope 状态的快照**，在 preReady 阶段冻结
- **一个 Scope 可以有多个 Context**（每个路由一个）
- **Context 绑定到创建它的 Scope**（`context.server = this`）
- 但 Context 的 `Request`/`Reply`/`hooks` 可能包含来自**祖先 Scope** 的内容（通过传播机制）

### 7.5 运行时请求如何通过 Context 访问 Hooks 链

现在让我们看看请求到来时，Context 是如何被使用的。

#### 路由匹配阶段

`find-my-way` 路由器匹配 URL 后，返回对应的 Context：

```javascript
// 路由注册时：
router.on(opts.method, opts.url, { constraints }, routeHandler, context)

// 请求到来时，find-my-way 调用：
routeHandler(req, res, params, context, query)
```

#### routeHandler 执行（`lib/route.js:462-589`）

```javascript
function routeHandler (req, res, params, context, query) {
  // 1. 生成请求 ID
  const id = getGenReqId(context.server, req)

  // 2. 从 Context 创建 logger（使用 context.logLevel, context.logSerializers）
  const loggerOpts = {
    level: context.logLevel
  }
  if (context.logSerializers) {
    loggerOpts.serializers = context.logSerializers
  }
  const childLogger = createChildLogger(context, logger, req, id, loggerOpts)

  // 3. 从 Context 创建 Request 和 Reply 实例！
  // 注意：使用的是 context.Request 和 context.Reply（已优化的构造函数）
  const request = new context.Request(id, params, req, query, childLogger, context)
  const reply = new context.Reply(res, request, childLogger)

  // 4. 设置 handler 超时（使用 context.handlerTimeout）
  const handlerTimeout = context.handlerTimeout
  if (handlerTimeout > 0) {
    // ... 设置超时
  }

  // 5. 执行 onRequest hooks（从 context.onRequest 读取！）
  if (context.onRequest !== null) {
    onRequestHookRunner(
      context.onRequest,    // <-- 直接从 Context 读取
      request,
      reply,
      runPreParsing        // 下一个阶段的回调
    )
  } else {
    runPreParsing(null, request, reply)
  }

  // 6. 注册其他 hooks 的监听器
  if (context.onRequestAbort !== null) { /* ... */ }
  if (context.onTimeout !== null) { /* ... */ }
}
```

**关键点**：
- **所有配置都从 Context 读取**：`logLevel`、`logSerializers`、`handlerTimeout`、`onRequest` 等
- **Request/Reply 使用 Context 中的构造函数**：`new context.Request(...)`、`new context.Reply(...)`
- **Hooks 直接从 Context 读取**：`context.onRequest`，不再访问 `server[kHooks]`

#### Hooks 链执行流程

从 `lib/route.js` 和 `lib/handle-request.js` 可以看到完整的执行顺序：

```
请求到达 → find-my-way 匹配 → routeHandler(context, ...)
                                              │
                                              ▼
                    ┌─────────────────────────────────────────┐
                    │  1. context.onRequest (onRequestHookRunner) │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  2. context.preParsing (preParsingHookRunner) │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  3. 解析请求体 (handleRequest)          │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  4. context.preValidation (preValidationHookRunner) │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  5. Schema 验证 (validateSchema)       │
                    │     使用 context.validatorCompiler      │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  6. context.preHandler (preHandlerHookRunner) │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  7. context.handler (路由处理函数)      │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  8. context.preSerialization          │
                    │     (preSerializationHookRunner)        │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  9. 序列化响应                          │
                    │     使用 context.serializerCompiler     │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  10. context.onSend (onSendHookRunner) │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  11. 发送响应 (reply.send)              │
                    └───────────────────┬─────────────────────┘
                                        │
                                        ▼
                    ┌─────────────────────────────────────────┐
                    │  12. context.onResponse (onResponseHookRunner) │
                    └─────────────────────────────────────────┘
```

#### 错误处理

如果在任何阶段发生错误：

```javascript
// 1. 标记错误
reply[kReplyIsError] = true

// 2. 发送错误（触发 context.onError hooks）
reply.send(err)

// 3. errorHandler 处理（使用 context.errorHandler）
const context = reply[kRouteContext]
// errorHandler 从 context.errorHandler 读取
```

### 7.6 Request/Reply 与 Context 的双向绑定

让我们看看 Request 和 Reply 实例如何访问 Context：

**Request 构造函数**（`lib/request.js`）：
```javascript
function _Request (id, params, req, query, log, context) {
  this.id = id
  this[kRouteContext] = context  // <-- 存储 Context 引用
  this.params = params
  // ...
}
```

**Reply 的 kRouteContext getter**（`lib/reply.js:80-84`）：
```javascript
[kRouteContext]: {
  get () {
    return this.request[kRouteContext]  // 代理到 Request 的 Context
  }
}
```

**双向关系**：
```
┌─────────────────┐          ┌─────────────────┐
│    Request      │          │     Reply       │
├─────────────────┤          ├─────────────────┤
│ [kRouteContext] │─────────▶│    Context      │
│      context    │          └─────────────────┘
└─────────────────┘                   ▲
                                        │
                                        │
                              ┌─────────┴─────────┐
                              │  reply[kRouteContext]│
                              │    getter 代理      │
                              └─────────────────────┘
```

**实际使用**：
```javascript
// 在 handler 或 hook 中：
fastify.get('/', (req, reply) => {
  // req.routeContext 就是 Context 对象
  console.log(req.routeContext.config)    // 路由配置
  console.log(req.routeContext.logLevel)  // 日志级别
  console.log(req.routeContext.server)    // Fastify 实例
  
  // reply.routeContext 也可以访问（代理到 req.routeContext）
  console.log(reply.routeContext === req.routeContext)  // true
})
```

### 7.7 fastify-plugin 如何影响 Context

现在让我们理解 `fastify-plugin`（skip-override）如何影响 Context 的创建和固化。

#### 场景对比

```javascript
const fastify = Fastify()
const fp = require('fastify-plugin')

// ========== 场景 A: 普通插件 ==========
fastify.register(function normalPlugin (instance, opts, done) {
  // 1. override 函数创建了新的封装实例 instance
  // 2. instance.kHooks 是独立的（通过 buildHooks 复制父 hooks）
  
  instance.addHook('onRequest', (req, reply, done) => {
    console.log('normalPlugin onRequest')
    done()
  })
  
  instance.get('/normal', (req, reply) => {
    // 这个路由的 Context:
    // - server = instance（子 scope）
    // - onRequest = [父 hooks..., 'normalPlugin onRequest']
    // - Request = 已优化的构造函数
    reply.send({ from: 'normal' })
  })
  
  done()
})

// ========== 场景 B: fastify-plugin ==========
fastify.register(fp(function sharedPlugin (instance, opts, done) {
  // 1. override 函数直接返回 old（fastify 实例）
  // 2. instance === fastify
  
  instance.addHook('onRequest', (req, reply, done) => {
    console.log('sharedPlugin onRequest')
    done()
  })
  
  instance.get('/shared', (req, reply) => {
    // 这个路由的 Context:
    // - server = fastify（根实例）
    // - onRequest = [fastify 的 hooks..., 'sharedPlugin onRequest']
    // - Request = fastify 的 Request 构造函数
    reply.send({ from: 'shared' })
  })
  
  done()
}))
```

#### 关键差异

| 维度 | 普通插件 | fastify-plugin |
|------|----------|----------------|
| **Context.server** | 新创建的子 scope 实例 | 父 scope 实例（可能是 fastify） |
| **Context 中的 hooks** | 子 scope 的 `kHooks`（包含从父复制的 + 自己添加的） | 父 scope 的 `kHooks`（包含所有祖先的） |
| **对其他路由的影响** | 只影响当前插件内注册的路由 | 影响父 scope 及其所有子 scope 中**后续注册**的路由 |

#### 为什么 fastify-plugin 的 hooks 会影响"后续注册"的路由？

因为：

1. **Hooks 固化发生在 preReady 阶段**，而不是路由注册时
2. **preReady 是在所有插件注册完成后**才触发的
3. 即使路由在插件之前注册，只要在 **preReady 之前**，fastify-plugin 添加的 hooks 都会被包含

**时间线示例**：
```
时间点 1: fastify.get('/early', handler)
         - Context 创建，hooks 还是 null
         - 注册 avvio.once('preReady') 回调

时间点 2: fastify.register(fp(function (instance, done) {
           instance.addHook('onRequest', myHook)
           done()
         }))
         - 直接在 fastify 上执行
         - fastify[kHooks].onRequest.push(myHook)

时间点 3: avvio 触发 preReady 事件
         - /early 路由的 preReady 回调执行
         - context.onRequest = this[kHooks].onRequest  // 包含 myHook！
         - 即使路由在 fp 插件之前注册！
```

**这就是为什么 `test/404s.test.js` 中的测试能通过**：
```javascript
test('run non-encapsulated plugin hooks on default 404', (t, done) => {
  const fastify = Fastify()

  // 先注册路由
  fastify.get('/', function (req, reply) {
    reply.send({ hello: 'world' })
  })

  // 后注册 fp 插件
  fastify.register(fp(function (instance, options, done) {
    instance.addHook('onRequest', function (req, res, done) {
      t.assert.ok(true, 'onRequest called')  // 会在 404 时触发！
      done()
    })
    done()
  }))

  // preReady 时，所有路由（包括 404）的 Context 都会包含这个 hook
})
```

### 7.8 为什么 preReady 之后 addHook 不影响已注册的路由？

因为：

1. **已注册路由的 Context 的 `context[hook]` 已经固化**
2. 虽然新的 hook 会通过 `_addHook` 传播到子 scope 的 `kHooks`
3. 但**已固化的 Context 不会被更新**
4. 只有**之后注册**的路由会在它们的 preReady 阶段包含新的 hook

**示例**：
```javascript
const fastify = Fastify()

// 先注册路由
fastify.get('/first', (req, reply) => {
  reply.send({ route: 'first' })
})

// 等待 preReady 后再添加 hook
fastify.ready(() => {
  // 此时 /first 的 Context.onRequest 已经固化为 null 或初始值
  
  // 添加新 hook
  fastify.addHook('onRequest', (req, reply, done) => {
    console.log('late hook')  // 不会在 /first 路由触发！
    done()
  })
  
  // 注册新路由
  fastify.get('/second', (req, reply) => {
    reply.send({ route: 'second' })
  })
  
  // /second 的 Context 会在它自己的 preReady 阶段包含这个 hook
  // 但 /first 不会！
})
```

### 7.9 Context 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    Context 生命周期完整流程图                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  阶段 1: 路由注册时                                                                  │
│  ─────────────────                                                                    │
│                                                                                      │
│  fastify.get('/path', { preHandler: [routeHook] }, handler)                        │
│       │                                                                              │
│       ▼                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  addNewRoute()                                                                │   │
│  │                                                                               │   │
│  │  1. 创建 Context:                                                             │   │
│  │     context = new Context({                                                  │   │
│  │       schema: opts.schema,                                                   │   │
│  │       handler: opts.handler.bind(this),  // 绑定到当前 scope                │   │
│  │       server: this,                        // 当前 scope 实例                │   │
│  │       Request: this[kRequest],             // 当前 scope 的 Request          │   │
│  │       Reply: this[kReply],                 // 当前 scope 的 Reply            │   │
│  │       onRequest: null,                      // 初始为 null                   │   │
│  │       preHandler: null,                     // 初始为 null                   │   │
│  │       ...                                                                     │   │
│  │     })                                                                        │   │
│  │                                                                               │   │
│  │  2. 注册到路由器:                                                             │   │
│  │     router.on(method, url, constraints, routeHandler, context)             │   │
│  │                                                                               │   │
│  │  3. 注册 this.after 回调:                                                     │   │
│  │     this.after(() => {                                                       │   │
│  │       // 补充 context 属性                                                    │   │
│  │       avvio.once('preReady', () => { /* 固化阶段 */ })                      │   │
│  │     })                                                                        │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  阶段 2: 所有插件注册完成后                                                          │
│  ─────────────────────────                                                          │
│                                                                                      │
│  avvio 触发 'preReady' 事件                                                          │
│       │                                                                              │
│       ▼                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  preReady 阶段（状态固化）                                                    │   │
│  │                                                                               │   │
│  │  对每个已注册路由，执行其 preReady 回调:                                      │   │
│  │                                                                               │   │
│  │  1. 固化 hooks 链:                                                            │   │
│  │     for (const hook of lifecycleHooks) {                                     │   │
│  │       toSet = this[kHooks][hook]           // scope 的 hooks（继承+传播）   │   │
│  │               .concat(opts[hook] || [])    // 路由级 hooks                  │   │
│  │               .map(h => h.bind(this))       // 绑定到当前 scope             │   │
│  │       context[hook] = toSet.length ? toSet : null                           │   │
│  │     }                                                                         │   │
│  │                                                                               │   │
│  │  2. 优化 Request/Reply 构造函数:                                             │   │
│  │     while (!context.Request[kHasBeenDecorated] && context.Request.parent) { │   │
│  │       context.Request = context.Request.parent  // 使用父构造函数           │   │
│  │     }                                                                         │   │
│  │     // 同理优化 Reply                                                         │   │
│  │                                                                               │   │
│  │  3. 设置 404 Context:                                                        │   │
│  │     fourOhFour.setContext(this, context)                                     │   │
│  │                                                                               │   │
│  │  4. 编译 Schema:                                                              │   │
│  │     if (opts.schema) {                                                        │   │
│  │       compileSchemasForValidation(context, ...)                              │   │
│  │       compileSchemasForSerialization(context, ...)                           │   │
│  │     }                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  此时 Context 已完全固化，不再变化！                                                  │
│                                                                                      │
│  阶段 3: 运行时请求处理                                                              │
│  ───────────────────────                                                             │
│                                                                                      │
│  请求到达: GET /path                                                                  │
│       │                                                                              │
│       ▼                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  find-my-way 路由匹配                                                        │   │
│  │  找到对应的 routeHandler 和 context                                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│       │                                                                              │
│       ▼                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  routeHandler(req, res, params, context, query)                             │   │
│  │                                                                               │   │
│  │  1. 从 Context 创建 logger:                                                   │   │
│  │     loggerOpts = { level: context.logLevel, serializers: context.logSerializers }│
│  │                                                                               │   │
│  │  2. 从 Context 创建 Request/Reply:                                            │   │
│  │     request = new context.Request(id, params, req, query, logger, context) │   │
│  │     // request[kRouteContext] = context                                      │   │
│  │                                                                               │   │
│  │     reply = new context.Reply(res, request, logger)                          │   │
│  │     // reply[kRouteContext] getter → request[kRouteContext]                 │   │
│  │                                                                               │   │
│  │  3. 执行 hooks 链（全部从 Context 读取！）:                                   │   │
│  │                                                                               │   │
│  │     if (context.onRequest !== null) {                                        │   │
│  │       onRequestHookRunner(context.onRequest, request, reply, runPreParsing) │   │
│  │     }                                                                         │   │
│  │     // 后续: preParsing → preValidation → preHandler → handler              │   │
│  │     //      → preSerialization → onSend → onResponse                        │   │
│  │     // 全部使用 context[hook]                                                 │   │
│  │                                                                               │   │
│  │  4. 错误处理:                                                                 │   │
│  │     // 使用 context.errorHandler                                              │   │
│  │     // 执行 context.onError hooks                                            │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  整个过程中，不再访问 scope 的 kHooks、kRequest、kReply 等！                         │
│  所有配置都从 Context 读取！                                                          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 8. 完整流程图

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
