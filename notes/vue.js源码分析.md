# Vue 源码解析 — 响应式原理

- Vue.js 的静态成员和实例成员初始化过程
- 首次渲染的过程
- 数据响应式原理

## 准备工作

### Vue 源码的获取

- 项目地址：https://github.com/vuejs/vue
- Fork 一份到自己仓库，克隆到本地，可以写注释提交到 github

### 源码目录结构

```
src
  |- compiler       编译相关
  |- core           Vue 核心库
  |- platforms      平台相关代码
  |- server         SSR,服务端渲染
  |- sfc            .vue 文件编译为 js 对象
  |- shared         公共的代码
```

### Flow

- 官网：https://flow.org/

- JavaScript 的静态类型检查器

- Flow 的静态类型检查错误是通过静态类型推断实现的

  - 文件开头通过 ``` //@flow ``` 或者 ``` /* @flow */``` 声明

  ```js
  /* @flow */
  function square(n: number): number {
  	return n * n;
  }
  square('2'); // Error!
  ```

### 调试设置

#### 打包

- 打包工具 Rollup

  - Vue.js 源码的打包工具使用的是 Rollup，比 Webpack 轻量
  - Webpack 把所有文件当作模块，Rollup 只处理 js 文件，更适合在 Vue.js 这样的库中使用
  - Rollup 打包不会生成冗余的代码

- 安装依赖

  ```bash
  $ npm i
  # 或者 yarn
  ```

- 设置 sourcemap

  - package.json 文件中的 dev 脚本中添加参数 --sourcemap

    ```json
    "dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev",
    ```

  - 执行 dev

    - ```npm run dev``` 执行打包，用的是 rollup, -w 参数是监听文件的变化，文件变化自动重新打包

### [Vue 的不同构建版本](https://cn.vuejs.org/v2/guide/installation.html#%E5%AF%B9%E4%B8%8D%E5%90%8C%E6%9E%84%E5%BB%BA%E7%89%88%E6%9C%AC%E7%9A%84%E8%A7%A3%E9%87%8A)

- ```yarn build``` 重新打包所有文件
- [官方文档-对不同构建版本的解释](https://cn.vuejs.org/v2/guide/installation.html#%E5%AF%B9%E4%B8%8D%E5%90%8C%E6%9E%84%E5%BB%BA%E7%89%88%E6%9C%AC%E7%9A%84%E8%A7%A3%E9%87%8A)
- dist\README.md

|                          | UMD                | CommonJS              | ES Module          |
| ------------------------ | ------------------ | --------------------- | ------------------ |
| Full                     | vue.js             | vue.common.js         | vue.esm.js         |
| Runtime-only             | vue.runtime.js     | vue.runtime.common.js | vue.runtime.esm.js |
| Full(production)         | vue.min.js         |                       |                    |
| Runtime-only(production) | vue.runtime.min.js |                       |                    |

#### 术语

- **完整版**：同时包含**编译器**和**运行时**的版本
- **编译器**：用来将模板字符串编译成为 JavaScript 渲染函数的代码，体积大，效率低
- **运行时**：用来创建 Vue 实例，渲染并处理虚拟 DOM 等的体积，体积小，效率高。基本上就是除去编译器的代码
- **[UMD](https://github.com/umdjs/umd)**：UMD 版本是**通用的模块版本**，支持多种模块方式。vue.js 默认文件就是运行时 + 编译器的 UMD 版本
- **[CommonJS(cjs)](http://wiki.commonjs.org/wiki/CommonJS)**：CommonJS 版本用来配合老的打包工具，比如 [Browserify](http://browserify.org/) 或 [webpack 1](https://webpack.github.io/)
- [ES Module](https://exploringjs.com/es6/ch_modules.html)：从 2.6 开始 Vue 会提供两个 ES Modules (ESM) 构建文件，为现代打包工具提供的版本
  - ESM 格式被设计为可以被静态分析，所以打包工具可以利用这一点来进行"tree-shaking"并将用不到的代码排除出最终的包
  - [ES6 模块于 CommonJS 模块的差异](https://es6.ruanyifeng.com/#docs/module-loader#ES6-%E6%A8%A1%E5%9D%97%E4%B8%8E-CommonJS-%E6%A8%A1%E5%9D%97%E7%9A%84%E5%B7%AE%E5%BC%82)

## 寻找入口文件

- 查看 dist/vue.js 的构建过程

### 执行构建

```bash
$ npm run dev
# "dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev",
# --environment TARGET:web-full-dev 设置环境变量 TARGET
```

- scripts/config.js 的执行过程
  - 作用：生成 rollup 构建的配置文件
  - 使用环境变量 TARGET = web-full-dev

```js
// config.js

// 判断环境变量是否有 TARGET
// 如果有的话使用 getConfig() 生成 rollup 配置文件
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  // 否则获取全部配置
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}

```

- genConfig(name)
  - 根据环境变量 TARGET 获取配置信息
  - builds[name] 获取生成配置的信息
- resolve()——获取入口和出口文件的绝对路径

### 结果

- 把 src/platforms/web/entry-runtime-with-compiler.js 构建成 dist/vue.js，如果设置 --sourcemap 会生成 vue.js.map
- src/platforms 文件夹下是 Vue 可以构建成不同平台下使用的库，目前有 weex 和 web，还有服务端渲染的库

## 从入口开始

- src/platform/web/entry-runtime-with-compiler.js

### 通过查看源码解决下面问题

- 观察以下代码，通过阅读源码，回答在页面上输出的结果

```js
const vm = new Vue({
  el: '#app',
  template: '<h3>Hello template</h3>',
  render (h) {
  	return h('h4', 'Hello render')
  }
})
```

渲染的是 render 方法里面的内容

- 阅读源码记录

  - el 不能是 body 或者 html 标签
  - 如果没有 render，把 template 转换成 render 函数
  - 如果有 render 方法，直接调用 mount 挂载 DOM

  ```js
    // 1.el 不能是 body 或者 html
    if (el === document.body || el === document.documentElement) {
      process.env.NODE_ENV !== 'production' && warn(
        `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
      )
      return this
    }
  
    const options = this.$options
    // 2.把 template/el 转换成 render 函数
    if (!options.render) {
  	...
    }
    // 3.调用 mount 方法，渲染 DOM
    return mount.call(this, el, hydrating)
  ```

- 调试代码

  - 调试的方法：在入口文件里设置断点，查看调用堆栈可以知道  Vue.$mount 在哪里调用

>Vue 的构造函数在哪？通过调用堆栈可以找到 index.js
>
>Vue 实例的成员 Vue 的静态成员从哪里来的?

## Vue 的构造函数在哪

- src/platform/web/entry-runtime-with-compiler.js 中引用了 ‘./runtime/index'

- src/platform/web/runtime/index.js

  - 设置 Vue.config
  - 设置平台相关的指令和组件
    - 指令 v-model、v-show
    - 组件 transition、transition-group
  - 设置平台相关的 \_pathc\_ 方法（打补丁方法，对比新旧的 VNode）
  - 设置 $mount 方法，挂载 DOM

  ```js
  // install platform runtime directives & components
  extend(Vue.options.directives, platformDirectives)
  extend(Vue.options.components, platformComponents)
  
  // install platform patch function
  // path 把虚拟 DOM 转换为真实 DOM
  Vue.prototype.__patch__ = inBrowser ? patch : noop
  
  // public mount method
  Vue.prototype.$mount = function (
    el?: string | Element,
    hydrating?: boolean
  ): Component {
    el = el && inBrowser ? query(el) : undefined
    return mountComponent(this, el, hydrating)
  }
  
  ```

- src/platform/web/runtime/index.js 中引用了 ’core/index‘

- src/core/index.js

  - 定义了 Vue 的静态方法
  - initGlobalAPI(Vue)

- src/core/index.js 中引用了 ''./instance/index'

- src/core/instance/index.js

  - 定义了 Vue 的构造函数

  ```js
  function Vue(options) {
    if (process.env.NODE_ENV !== 'production' &&
      !(this instanceof Vue)
    ) {
      warn('Vue is a constructor and should be called with the `new` keyword')
    }
    // 调用 _init() 方法
    this._init(options)
  }
  
  // 注册 vm 的 _init() 方法，初始化 vm
  initMixin(Vue)
  // 注册 vm 的 $data/$props/$set/$delete/$watch
  stateMixin(Vue)
  // 初始化事件相关方法
  // $on/$once/$off/$emit
  eventsMixin(Vue)
  // 初始化生命周期相关的混入方法
  // _update/$forceUpdate/$destroy
  lifecycleMixin(Vue)
  // 混入 render
  // $nextTick/_render
  renderMixin(Vue)
  
  export default Vue
  ```

## 四个导出 Vue 的模块

- src/**platforms/web**/entry-runtime-with-compiler.js
  - web 平台相关的入口
  - 重写了平台相关的 $mount 方法
  - 注册了 Vue.compile()方法，传递一个 HTML 字符串返回 render 函数
- src/**platforms/web**/runtime/index.js
  - web 平台相关
  - 注册和平台相关的全局指令：v-model、v-show
  - 注册和平台相关的全局组件：v-transition、v-transition-group
  - 全局方法：
    - \_patch\_ ：把虚拟 DOM 转换成真实 DOM
    - $mount: 挂载方法
- src/**core**/index.js
  - 与平台无关
  - 设置了 Vue 的静态方法，initGlobalAPI(Vue)
- src/**core**/instance/index.js
  - 与平台无关
  - 定义了构造函数、调用了 this._init(options) 方法
  - 给 Vue 中混入了常用的实例成员

## Vue 的初始化

### src/core/global-api/index.js

- 初始化 Vue 的静态方法

```js
// \src\core\index.js
// 注册 Vue 的静态属性/方法
initGlobalAPI(Vue)

// src/core/global-api/index.js
// 初始化 Vue.config 对象
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  // 这些工具方法不视作全局API的一部分，除非你已经意识到某些风险，否则不要去依赖他们
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  // 静态方法 set/delete/nextTick
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  // 2.6 explicit observable API
  // 让一个对象可响应
  Vue.observable = <T>(obj: T): T => {
    observe(obj)
    return obj
  }

  // 初始化 Vue.options 对象，并给其扩展
  // components/directives/filters
  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
      Vue.options[type + 's'] = Object.create(null)
    })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  // 设置 keep-alive 组件
  extend(Vue.options.components, builtInComponents)

  // 注册 Vue.use() 用来注册插件
  initUse(Vue)
  // 注册 Vue.mixin() 实现插入
  initMixin(Vue)
  // 注册 Vue.extend() 基于传入的 options 返回一个组件的构造函数
  initExtend(Vue)
  // 注册 Vue.directive()、Vue.component()、Vue.filter()
  initAssetRegisters(Vue)
```

### src/core/instance/index.js

- 定义 Vue 的构造函数
- 初始化 Vue 的实例成员

```js
// 此处不用 class 的原因是因为方便后续给 Vue 实例混入实例成员
function Vue(options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // 调用 _init() 方法
  this._init(options)
}

// 注册 vm 的 _init() 方法，初始化 vm
initMixin(Vue)
// 注册 vm 的 $data/$props/$set/$delete/$watch
stateMixin(Vue)
// 初始化事件相关方法
// $on/$once/$off/$emit
eventsMixin(Vue)
// 初始化生命周期相关的混入方法
// _update/$forceUpdate/$destroy
lifecycleMixin(Vue)
// 混入 render
// $nextTick/_render
renderMixin(Vue)
```

- initMixin(Vue)

  - 初始化 _init() 方法

  ```js
  // src\core\instance\init.js
  
  export function initMixin (Vue: Class<Component>) {
    // 给 Vue 实例增加 init() 方法
    // 合并 options / 初始化操作
    Vue.prototype._init = function (options?: Object) {
      const vm: Component = this
      // a uid
      vm._uid = uid++
  
      let startTag, endTag
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        startTag = `vue-perf-start:${vm._uid}`
        endTag = `vue-perf-end:${vm._uid}`
        mark(startTag)
      }
  
      // a flag to avoid this being observed
      // 如果是 Vue 实例不需要被 Observe
      vm._isVue = true
      // merge options
      // 合并 options
      if (options && options._isComponent) {
        // optimize internal component instantiation
        // since dynamic options merging is pretty slow, and none of the
        // internal component options needs special treatment.
        initInternalComponent(vm, options)
      } else {
        vm.$options = mergeOptions(
          resolveConstructorOptions(vm.constructor),
          options || {},
          vm
        )
      }
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        initProxy(vm)
      } else {
        vm._renderProxy = vm
      }
      // expose real self
      vm._self = vm
      // vm 的生命周期相关变量初始化
      // $children/$parent/$root/$refs
      initLifecycle(vm)
      // vm 的事件监听初始化，父组件绑定在当前组件上的事件
      initEvents(vm)
      // vm 的编译 render 初始化
      // $slots/$scopedSlots/_c/$createElement/$attrs/$listeners
      initRender(vm)
      // beforceCreate 生命钩子的回调
      callHook(vm, 'beforeCreate')
      // 把 inject 的成员注入到 vm 上
      initInjections(vm) // resolve injections before data/props
      // 初始化 vm 的 _props/methods/_data/computed/watch
      initState(vm)
      // 初始化 provide
      initProvide(vm) // resolve provide after data/props
      // created 生命钩子的回调
      callHook(vm, 'created')
  
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        vm._name = formatComponentName(vm, false)
        mark(endTag)
        measure(`vue ${vm._name} init`, startTag, endTag)
      }
  
      if (vm.$options.el) {
        vm.$mount(vm.$options.el)
      }
    }
  }
  ```

## 首次渲染过程

- Vue 初始化完毕，开始真正的执行
- 调用 new Vue() 之前，已经初始化完毕
- 通过调试代码，记录首次渲染过程（详见 首次渲染过程.xmind）

## 数据响应式原理

### 通过查看源码解决下面问题

- vm.msg = { count: 0 },重新给属性赋值，是否是响应式的？ 是
- vm.arr[0] = 4, 给数组元素赋值，视图是否会更新？不会
- vm.arr.length = 0, 修改数组的 length,视图是否会更新？不会
- vm.arr.push(4)，视图是否会更新？会

### 响应式处理的入口

整个响应式处理过程比较复杂

- src\core\instance\init.js

  - ```initState(vm)``` vm 状态的初始化
  - 初始化了 \_data、\_props、 methods 等

- src\core\instance\state.js

  ```js
    if (opts.data) {
      // 数据的初始化
      initData(vm)
    } else {
      observe(vm._data = {}, true /* asRootData */)
    }
  ```

- initData(vm) vm 数据的初始化

  ```js
  function initData (vm: Component) {
    let data = vm.$options.data
    // 初始化 _data, 组件中 data 是函数，调用函数返回结果
    // 否则直接返回 data
    data = vm._data = typeof data === 'function'
      ? getData(data, vm)
      : data || {}
    ......
    // proxy data on instance
    // 获取 data 中的所有属性
    const keys = Object.keys(data)
    // 获取 props / methods
    const props = vm.$options.props
    const methods = vm.$options.methods
    let i = keys.length
    // 判断 data 上的成员是否和 props/methods 重名
    ......
    // observe data
    // 响应式处理
    observe(data, true /* asRootData */)
  }
  ```

- src\core\observer\index.js

  - observer(value, asRootData)
  - 负责为每一个 Object 类型的 value 创建一个 observer 实例

  ```js
  export function observe(value: any, asRootData: ?boolean): Observer | void {
    // 判断 value 是否是对象
    if (!isObject(value) || value instanceof VNode) {
      return
    }
    let ob: Observer | void
    // 如果 value 有 __ob__(observer 对象) 属性 结束
    if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
      ob = value.__ob__
    } else if (
      shouldObserve &&
      !isServerRendering() &&
      (Array.isArray(value) || isPlainObject(value)) &&
      Object.isExtensible(value) &&
      !value._isVue
    ) {
      // 创建一个 Observer 对象
      ob = new Observer(value)
    }
    if (asRootData && ob) {
      ob.vmCount++
    }
    return ob
  }
  ```

### Observer

- src\core\observer\index.js

  - 对对象做响应化处理
  - 对数组做响应化处理

  ```js
  export class Observer {
    // 观测对象
    value: any;
    // 依赖对象
    dep: Dep;
    // 实例计数器
    vmCount: number; // number of vms that have this object as root $data
  
    constructor(value: any) {
      this.value = value
      this.dep = new Dep()
      // 初始化实例的 vmCount 为0
      this.vmCount = 0
      // 将实例挂载到观察对象的 __ob__ 属性
      def(value, '__ob__', this)
      // 数组的响应式处理
      if (Array.isArray(value)) {
        if (hasProto) {
          protoAugment(value, arrayMethods)
        } else {
          copyAugment(value, arrayMethods, arrayKeys)
        }
        // 为数组中的每一个对象创建一个 observer 实例
        this.observeArray(value)
      } else {
        // 遍历对象中的每一个属性，转换成 setter/getter
        this.walk(value)
      }
    }
  
    /**
     * Walk through all properties and convert them into
     * getter/setters. This method should only be called when
     * value type is Object.
     */
    walk(obj: Object) {
      // 获取观察对象的每一个属性
      const keys = Object.keys(obj)
      // 遍历每一个属性，设置为响应式数据
      for (let i = 0; i < keys.length; i++) {
        defineReactive(obj, keys[i])
      }
    }
  
    /**
     * Observe a list of Array items.
     */
    observeArray(items: Array<any>) {
      for (let i = 0, l = items.length; i < l; i++) {
        observe(items[i])
      }
    }
  }
  ```

- walk(obj)

  - 遍历 obj 的所有属性，为每一个属性调用 defineReactive() 方法，设置 getter/setter

### defineReactive()

- src\core\observer\index.js
- defineReactive(obj, key, val, customSetter, shallow)
  - 为一个对象定义一个响应式的属性，每一个属性对应一个 dep 对象
  - 如果该属性的值是对象，继续调用 observe
  - 如果给属性赋新值，继续调用 observe
  - 如果数据更新发送通知

### 对象响应式处理

```js
// 为一个对象定义一个响应式的属性
/**
 * Define a reactive property on an Object.
 */
export function defineReactive(
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 1.为每一个属性，创建依赖对象实例
  const dep = new Dep()
  // 获取 obj 的属性描述符对象
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }
  // 提供预定义的存取器函数
  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 2.判断是否递归观察子对象，并将子对象属性都转换成 getter/setter，返回子观察对象
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      // 如果预定义的 getter 存在则 value 等于 getter 调用的返回值
      // 否则直接赋予属性值
      const value = getter ? getter.call(obj) : val
      // 如果存在当前依赖目标，即 watcher 对象，则建立依赖
      if (Dep.target) {
        // dep() 添加相互的依赖
        // 一个组件对应一个 watcher 对象
        // 一个 watcher 会对应多个 dep (要观察的属性很多)
        // 可以手动创建多个 watcher 监听一个属性的变化，一个 dep 可以对应多个 watcher
        dep.depend()
        // 如果子观察目标存在，建立子对象的依赖关系，将来 Vue.set() 会用到
        if (childOb) {
          childOb.dep.depend()
          // 如果属性是数组，则特殊处理收集数组对象依赖
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      // 返回属性值
      return value
    },
    set: function reactiveSetter(newVal) {
      // 如果预定义的 getter 存在则 value 等于 getter 调用的返回值
      // 否则直接赋予属性值
      const value = getter ? getter.call(obj) : val
      // 如果新值等于旧值或者新值旧值为 NaN 则不执行
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // 如果没有 setter 直接返回
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      // 如果预定义 setter 存在则调用，否则直接更新新值
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 3.如果新值是对象，观察子对象并返回子的 observer 对象
      childOb = !shallow && observe(newVal)
      // 4.派发更新（发布更改通知）
      dep.notify()
    }
  })
}
```

### 数组的响应式处理

- Observer 的构造函数中

  ```js
      // 数组的响应式处理
      if (Array.isArray(value)) {
        if (hasProto) {
          protoAugment(value, arrayMethods)
        } else {
          copyAugment(value, arrayMethods, arrayKeys)
        }
        // 为数组中的每一个对象创建一个 observer 实例
        this.observeArray(value)
      } else {
        // 遍历对象中的每一个属性，转换成 setter/getter
        this.walk(value)
      }
      
  function protoAugment(target, src: Object) {
    /* eslint-disable no-proto */
    target.__proto__ = src
    /* eslint-enable no-proto */
  }
  
  /* istanbul ignore next */
  function copyAugment(target: Object, src: Object, keys: Array<string>) {
    for (let i = 0, l = keys.length; i < l; i++) {
      const key = keys[i]
      def(target, key, src[key])
    }
  }
  ```

- 处理数组修改数据的方法

  - src\core\observer\array.js

  ```js
  const arrayProto = Array.prototype
  // 使用数组的原型创建一个新的对象(克隆数组的原型)
  export const arrayMethods = Object.create(arrayProto)
  // 修改数组元素的方法
  const methodsToPatch = [
    'push',
    'pop',
    'shift',
    'unshift',
    'splice',
    'sort',
    'reverse'
  ]
  
  /**
   * Intercept mutating methods and emit events
   */
  methodsToPatch.forEach(function (method) {
    // cache original method
    // 保存数组原方法
    const original = arrayProto[method]
    // 调用 Object.defineProperty() 重新定义修改数组的方法
    def(arrayMethods, method, function mutator (...args) {
      // 执行数组的原始方法
      const result = original.apply(this, args)
      // 获取数组对象的 ob 对象
      const ob = this.__ob__
      let inserted
      switch (method) {
        case 'push':
        case 'unshift':
          inserted = args
          break
        case 'splice':
          inserted = args.slice(2)
          break
      }
      // 对插入的新元素，重新遍历数组元素设置为响应式数据
      if (inserted) ob.observeArray(inserted)
      // notify change
      // 调用了修改数组的方法，调用数组的 ob 对象发送通知
      ob.dep.notify()
      return result
    })
  })
  ```

### Dep 类

- src\core\observer\dep.js
- 依赖对象
- 记录 watcher 对象
- depend() -- watcher 记录对应的 dep
- 发布通知

```tex
1. 在 defineReactive() 的 getter 中创建 dep 对象，并判断 Dep.target 是否有值，调用 dep.depend()

2. dep.depend() 内部调用 Dep.target.addDep(this),也就是 watcher 的 addDep() 方法，它内部调用 dep.addSub(this), 把 watcher 对象添加到 dep.subs.push(watcher) 中，也就是i订阅者添加到 dep 的 subs 数组中，当数据变化的时候调用 watcher 对象的 update() 方法

3. 什么时候设置的 Dep.target?
   通过简单的案例调试观察，调用 mountComponent() 方法的时候，创建了渲染 watcher 对象，执行 watcher 中的 get() 方法
   
4. get() 方法内部调用 pushTarget(this),把当前 Dep.target = watcher，同时把当前 watcher 入栈。因为有父子组件嵌套的时候先把父组件对应的 watcher 入栈，再去处理子组件的 watcher ，子组件处理完毕后，再把父组件对应的 watcher 出栈，继续操作

5. Dep.target 用来存放目前正在使用的 watcher。全局唯一，并且一次也只能有一个 watcher 被使用
```

```js
// dep 是一个可观察对象，可以有多个指令订阅它
/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
export default class Dep {
  // 静态属性，watcher 对象
  static target: ?Watcher;
  // dep 实例 id
  id: number;
  // dep 实例对应的 watcher 对象/订阅者数组
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  // 添加新的订阅者 watcher 对象
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  // 移除订阅者
  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  // 将观察对象和 watcher 建立依赖
  depend () {
    if (Dep.target) {
      // 如果 target 存在，把 dep 对象添加到 watcher 的依赖中
      Dep.target.addDep(this)
    }
  }

  // 发布通知
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    // 调用每个订阅者的 update 方法实现更新
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
// Dep.target 用来存放目前正在使用的 watcher
// 全局唯一，并且一次也只能有一个 watcher 被调用
// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []
// 入栈并将当前 watcher 赋值给 Dep.target
export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  // 出栈操作 
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

### Watcher 类

- Watcher 分为三种，Computed Watcher、用户 Watcher（侦听器）、**渲染Watcher**

- 渲染 Watcher 的创建时机

  - /src/core/instance/lifecycle.js

  ```js
  export function mountComponent (
    vm: Component,
    el: ?Element,
    hydrating?: boolean
  ): Component {
    vm.$el = el
    ......
    callHook(vm, 'beforeMount')
  
    let updateComponent
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      ......
    } else {
      updateComponent = () => {
        vm._update(vm._render(), hydrating)
      }
    }
  
    // 创建渲染 Watcher.expOrFn 为 update Component
    // we set this to vm._watcher inside the watcher's constructor
    // since the watcher's initial patch may call $forceUpdate (e.g. inside child
    // component's mounted hook), which relies on vm._watcher being already defined
    new Watcher(vm, updateComponent, noop, {
      before () {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, 'beforeUpdate')
        }
      }
    }, true /* isRenderWatcher */)
    hydrating = false
  
    // manually mounted instance, call mounted on self
    // mounted is called for render-created child components in its inserted hook
    if (vm.$vnode == null) {
      vm._isMounted = true
      callHook(vm, 'mounted')
    }
    return vm
  }
  ```

- 渲染 watcher 创建的位置在 lifecycle.js 的 mountComponent 函数中

- Wacher 的构造函数初始化，处理 expOrFn （渲染 Watcher 和侦听器处理不同）

- 调用 this.get(), 它里面调用 pushTarget() 然后 this.getter.call(vm, vm)(对于渲染 watcher 调用 updateComponent)，如果是用户 watcher 会获取属性的值（触发 get 操作）

- 当数据更新的时候，dep 中调用 notify() 方法，notify() 中调用 watcher 的 update() 方法

- update() 中调用 queueWatcher()

- queueWatcher() 是一个核心方法，去除重复操作，调用 flushSchedulerQueue() 刷新队列并执行 watcher

- flushScheduleQueue() 中对 watcher 排序，遍历所有 watcher ，如果有before, 触发生命周期的钩子函数 beforeUpdate，执行 watcher.run(), 它内部调用 this.get()，然后调用 this.cb() （渲染 watcher 的 cb 是 noop）

- 整个流程结束

## [实例方法/数据](https://cn.vuejs.org/v2/api/#%E5%AE%9E%E4%BE%8B%E6%96%B9%E6%B3%95-%E6%95%B0%E6%8D%AE)

### [vm.$set](https://cn.vuejs.org/v2/api/#vm-set)

- 功能：向响应式对象中添加一个属性，并确保这个新属性同样是响应式的，且触发视图更新。它必须用于向响应式对象上添加新属性，因为 VUe 无法探测普通的新增属性（比如： this.myObject.newProperty = 'hi'）
- **注意：对象不能是 Vue 实例，或者 Vue 实例的根数据对象**
- 示例：```vm.$set(obj, 'foo', 'test')```

#### 定义位置

- Vue.set()—— global-api/index.js

  ````js
    // 静态方法 set/delete/nextTick
    Vue.set = set
    Vue.delete = del
    Vue.nextTick = nextTick
  ````

- vue.$set —— instance/index.js

  ```js
  // instance/index.js
  // 注册 vm 的 $data/$props/$set/$delete/$watch
  // instance/state.js
  stateMixin(Vue)
  ```

  ```js
  // instance/state.js
  Vue.prototype.$set = set
  ```

#### 源码

- set() 方法 —— observer/index.js

  ```js
  /**
   * Set a property on an object. Adds the new property and
   * triggers change notification if the property doesn't
   * already exist.
   */
  export function set(target: Array<any> | Object, key: any, val: any): any {
    if (process.env.NODE_ENV !== 'production' &&
      (isUndef(target) || isPrimitive(target))
    ) {
      warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
    }
    // 判断 target 是否是数组，key 是否是合法的索引
    if (Array.isArray(target) && isValidArrayIndex(key)) {
      target.length = Math.max(target.length, key)
      // 通过 splice 对 key 位置的元素进行替换
      // splice 在 array.js 进行了响应式的处理
      target.splice(key, 1, val)
      return val
    }
    // 如果 key 在对象中已经存在，直接赋值
    if (key in target && !(key in Object.prototype)) {
      target[key] = val
      return val
    }
    // 获取 target 中的 observer 对象
    const ob = (target: any).__ob__
    // 如果 target 是 vue 实例或者 $data 直接返回
    if (target._isVue || (ob && ob.vmCount)) {
      process.env.NODE_ENV !== 'production' && warn(
        'Avoid adding reactive properties to a Vue instance or its root $data ' +
        'at runtime - declare it upfront in the data option.'
      )
      return val
    }
    // 如果 ob 不存在，target 不是响应式对象，直接赋值
    if (!ob) {
      target[key] = val
      return val
    }
    // 把 key 设置为响应式属性
    defineReactive(ob.value, key, val)
   
      // 发送通知
    ob.dep.notify()
    return val
  }
  ```

### [vm.$delete](https://cn.vuejs.org/v2/api/#vm-delete)

- 功能：删除对象的属性，如果对象是响应式的，确保删除能触发更新视图。这个方法主要用于避开 Vue 不能检测到属性被删除的限制。但是很少会使用它
- **注意：目标对象不是一个 Vue 实例或 Vue 实例的根数据对象**
- 示例：```vm.$delete(vm.obj, 'msg')```

#### 定义位置

- Vue.delete() —— global-api/index.js

  ```js
    // 静态方法 set/delete/nextTick
    Vue.set = set
    Vue.delete = del
    Vue.nextTick = nextTick
  ```

- vm.$delete() —— instance/index.js

  ```js
  // instance/index.js
  // 注册 vm 的 $data/$props/$set/$delete/$watch
  // instance/state.js
  stateMixin(Vue)
  
  // instance/state.js
  Vue.prototype.$delete = del
  ```

#### 源码

- src\core\observer\index.js

  ```js
  /**
   * Delete a property and trigger change if necessary.
   */
  export function del(target: Array<any> | Object, key: any) {
    if (process.env.NODE_ENV !== 'production' &&
      (isUndef(target) || isPrimitive(target))
    ) {
      warn(`Cannot delete reactive property on undefined, null, or primitive value: ${(target: any)}`)
    }
    // 判断是否是数组，以及 key 是否合法
    if (Array.isArray(target) && isValidArrayIndex(key)) {
      // 如果是数组通过 splice 删除
      // splice 做过响应式处理
      target.splice(key, 1)
      return
    }
    // 获取 target 的 ob 对象
    const ob = (target: any).__ob__
    // target 如果是 Vue 实例或者 $data 对象，直接返回
    if (target._isVue || (ob && ob.vmCount)) {
      process.env.NODE_ENV !== 'production' && warn(
        'Avoid deleting properties on a Vue instance or its root $data ' +
        '- just set it to null.'
      )
      return
    }
    // 如果 target 对象没有 key 属性直接返回
    if (!hasOwn(target, key)) {
      return
    }
    // 删除属性
    delete target[key]
    if (!ob) {
      return
    }
    // 通过 ob 发送通知
    ob.dep.notify()
  }
  
  ```

### [vm.$watch](https://cn.vuejs.org/v2/api/#vm-watch)

- 功能：观察 Vue 实例变化的一个表达式或计算属性函数，回调函数得到的参数为新值和旧值。表达式只接受监督的键的路径，对于更复杂的表达式，用一个函数取代。

- 参数

  - expOrFn：要监视的 $data 中的属性，，可以是表达式或函数
  - callback：数据变化后执行的函数
    - 函数：回调函数
    - 对象：具有 handler 属性（字符串或者函数），如果该属性为字符串则 methods 中相应的定义
  - options：可选的选项
    - deep：布尔类型，深度监听
    - immediate：布尔类型，是否立即执行一次回调函数

- 示例：

  ```js
        const vm = new Vue({
          el: "#app",
          data: {
            a: '1',
            b: '2',
            msg: 'Hello Vue',
            user: {
              firstName: 'xxx',
              lastName: 'cccc',
              fullName: ''
            }
          }
        });
  
        // expOrFn 是表达式
        vm.$watch('msg', function(newValue, oldValue) {
          console.log(newValue, oldValue)
        })
        vm.$watch('user.firstName', function(newValue, oldValue) {
          console.log(newValue)
        })
  
        // expOrFn 是函数
        vm.$watch(function() {
          return this.a + this.b
        }, function(newValue, oldValue) {
          console.log(newValue)
        })
        
        vm.$watch('user', function(newValue, oldValue) {
          this.user.fullName = newValue.firstName + ' ' + newValue.lastName
        }, {
          immediate: true,
          deep: true
        });
  ```

#### 三种类型的 Watcher 对象

- 没有静态方法，因为 $watch 方法中要使用 Vue 的实例
- Watcher 分三种：计算属性 Watcher、用户 Watcher(侦听器)、渲染 Watcher
- 创建顺序：计算属性 Watcher、用户 Watcher(侦听器)、渲染 Watcher
- vm.$watch()
  - src\core\iinstance\state.js

#### 源码

```js
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    // 获取 Vue 实例 this
    const vm: Component = this
    if (isPlainObject(cb)) {
      // 判断如果 cb 是对象执行 createWatcher
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    // 标记为用户 watcher
    options.user = true
    // 创建用户 watcher 对象
    const watcher = new Watcher(vm, expOrFn, cb, options)
    // 判断 immediate 如果为 true
    if (options.immediate) {
      // 立即执行一次 cb 回调，并且把当前值传入
      try {
        cb.call(vm, watcher.value)
      } catch (error) {
        handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
      }
    }
    // 返回取消监听的方法
    return function unwatchFn () {
      watcher.teardown()
    }
  }
```

#### 调试

- 查看 watcher 的创建顺序
  - 计算属性 watcher
  - 用户 watcher(侦听器)
  - 渲染 watcher
- 查看渲染 watcher 的执行过程
  - 当数据更新，defineReactive 的 set 方法中调用 dep.notify()
  - 调用 watcher 的 update()
  - 调用 queueWatcher()，把 watcher 存入队列，如果已经存在，不重复添加
  - 循环调用 flushSchedulerQueue()
    - 通过 nextTick()，在消息循环结束之前时候调用 flushSchedulerQueue()
  - 调用 watcher.run()
    - 调用 watcher.get() 获取最新值
    - 如果是渲染 watcher 结束
    - 如果是用户 watcher ，调用 this.cb()

### [异步更新队列-nextTick()](https://cn.vuejs.org/v2/guide/reactivity.html#%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0%E9%98%9F%E5%88%97)

- Vue 更新 Dom 是异步执行的，批量的
  - 在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM
  - vm.$nextTick(function () { /* 操作 DOM */}) / Vue.nextTick(function () {})

#### 代码演示

```html
<div id="app">
<p ref="p1">{{ msg }}</p>
</div>
<script src="../../dist/vue.js"></script>
<script>
const vm = new Vue({
  el: '#app',
  data: {
    msg: 'Hello nextTick',
    name: 'Vue.js',
    title: 'Title'
  },
  mounted() {
    this.msg = 'Hello World'
    this.name = 'Hello snabbdom'
    this.title = 'Vue.js'
    this.$nextTick(() => {
        console.log(this.$refs.p1.textContent)
    })
  }
})
</script>

```

#### 定义位置

- src\core\instance\render.js

  ```js
    Vue.prototype.$nextTick = function (fn: Function) {
      return nextTick(fn, this)
    }
  ```

#### 源码

- 手动调用 vm.$nextTick()

- 在 Watcher 的 queueWatcher 中执行 nextTick()

- src\core\util\next-tick.js

  ```js
  
  export function nextTick (cb?: Function, ctx?: Object) {
    let _resolve
    // 把 cb 加上异常处理存入 callbacks 数组中
    callbacks.push(() => {
      if (cb) {
        try {
          // 调用 cb()
          cb.call(ctx)
        } catch (e) {
          handleError(e, ctx, 'nextTick')
        }
      } else if (_resolve) {
        _resolve(ctx)
      }
    })
    if (!pending) {
      pending = true
      // 调用
      timerFunc()
    }
    // $flow-disable-line
    if (!cb && typeof Promise !== 'undefined') {
      // 返回 promise 对象
      return new Promise(resolve => {
        _resolve = resolve
      })
    }
  }
  ```

----------------------

# Vue.js 源码剖析——虚拟 DOM

## 虚拟 DOM 概念回顾

什么是虚拟 DOM？

- 虚拟 DOM(Virtual DOM) 是使用 JavaScript 对象描述真实 DOM
- Vue.js 中的虚拟 DOM 借鉴 Snabbdom，并添加了 Vue.js 的特性
  - 例如：指令和组件机制

为什么要使用虚拟DOM？

- 避免直接操作 DOM，提高开发效率
- 作为一个中间层可以跨平台
- 虚拟 DOM 不一定可以提高性能
  - 首次渲染的时候会增加开销
  - 复杂视图情况下提升渲染性能

## h 函数

- vm.$createElement(tag, data, children, normalizeChildren)
- tag —— 标签名称或者组件对象
- data —— 描述 tag, 可以设置 DOM 的属性或者标签的属性
- children —— tag 中的文本内容或者子节点

## Vnode

Vnode 的核心属性

- tag
- data
- children
- text
- elm 记录真实的 element 对象
- key 用来复用当前元素

## createElement

### 功能

createElement() 函数用来创建虚拟节点（VNode），render 函数中的参数 h，就是 createElement()

```js
render(h) {
  // 此处的 h 就是 vm.$createElement
  return h('h1', this.msg)
}
```

### 定义

在 vm._render() 中调用了用户传递的或编译生成的 render 函数，这个时候传递了 createElement

- src/core/instance/render.js

  ```js
    // 对编译生成的 render 进行渲染的方法
    vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
    // normalization is always applied for the public version, used in
    // user-written render functions.
    // 对手写 render 函数进行渲染的方法
    vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
  ```

  vm.c 和 vm.$createElement 内部都调用了 createElement ，不同的是最后一个参数。vm.c 在编译生成的 render 函数内部会调用，vm.$createElement 在用户传入的 render 函数内部调用。当用户传入 render 函数的时候，要对用户传入的参数做处理

- src/core/vdom/create-element.js

  执行完 createElement 之后创建好了 VNode，把创建好的 VNode 传递给 vm._update() 继续处理

  ```js
  export function createElement (
    context: Component,
    tag: any,
    data: any,
    children: any,
    normalizationType: any,
    alwaysNormalize: boolean
  ): VNode | Array<VNode> {
    // 判断第三个参数
    // 如果 data 是数组或者原始值的话就是 children,实现类似函数重载的机制
    if (Array.isArray(data) || isPrimitive(data)) {
      normalizationType = children
      children = data
      data = undefined
    }
    if (isTrue(alwaysNormalize)) {
      normalizationType = ALWAYS_NORMALIZE
    }
    return _createElement(context, tag, data, children, normalizationType)
  }
  
  export function _createElement (
    context: Component,
    tag?: string | Class<Component> | Function | Object,
    data?: VNodeData,
    children?: any,
    normalizationType?: number
  ): VNode | Array<VNode> {
    if (isDef(data) && isDef((data: any).__ob__)) {
  	...
    }
    // <component v-bind:is="currentTabComponent"></component>
    // object syntax in v-bind
    if (isDef(data) && isDef(data.is)) {
      tag = data.is
    }
    if (!tag) {
      // in case of component :is set to falsy value
      return createEmptyVNode()
    }
    // warn against non-primitive key
    if (process.env.NODE_ENV !== 'production' &&
      isDef(data) && isDef(data.key) && !isPrimitive(data.key)
    ) {
      if (!__WEEX__ || !('@binding' in data.key)) {
        warn(
          'Avoid using non-primitive value as key, ' +
          'use string/number value instead.',
          context
        )
      }
    }
    // support single function children as default scoped slot
    if (Array.isArray(children) &&
      typeof children[0] === 'function'
    ) {
      data = data || {}
      data.scopedSlots = { default: children[0] }
      children.length = 0
    }
    // 去处理 children
    if (normalizationType === ALWAYS_NORMALIZE) {
      // 返回一维数组，处理用户手写的 render
      // 当手写 render 函数的时候调用
      // 判断 children 的类型，如果是原始值的话转换成 VNode 的数组
      // 如果是数组的话，继续处理数组中的元素
      // 如果数组中的子元素又是数组（slot template），递归处理
      // 如果连续两个节点都是字符串会合并文本节点
      children = normalizeChildren(children)
    } else if (normalizationType === SIMPLE_NORMALIZE) {
      // 把二维数组，转换成一维数组
      // 如果 children 中有函数组件的话，函数组件会返回数组形式
      // 这时候 children 就是一个二维数组，只需要把二维数组转换为一维数组
      children = simpleNormalizeChildren(children)
    }
    let vnode, ns
    // 判断 tag 是字符串还是组件
    if (typeof tag === 'string') {
      let Ctor
      ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
      // 是否是 html 的保留标签，创建对应的 VNode
      if (config.isReservedTag(tag)) {
        // platform built-in elements
        if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
          warn(
            `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
            context
          )
        }
        vnode = new VNode(
          config.parsePlatformTagName(tag), data, children,
          undefined, undefined, context
        )
      // 判断是否是 自定义组件
      } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
        // 查找自定义组件构造函数的声明
        // 根据 Ctor 创建组建的 VNode
        // component
        vnode = createComponent(Ctor, data, context, children, tag)
      } else {
        // unknown or unlisted namespaced elements
        // check at runtime because it may get assigned a namespace when its
        // parent normalizes children
        vnode = new VNode(
          tag, data, children,
          undefined, undefined, context
        )
      }
    } else {
      // direct component options / constructor
      vnode = createComponent(tag, data, context, children)
    }
    if (Array.isArray(vnode)) {
      return vnode
    } else if (isDef(vnode)) {
      if (isDef(ns)) applyNS(vnode, ns)
      if (isDef(data)) registerDeepBindings(data)
      return vnode
    } else {
      return createEmptyVNode()
    }
  }
  ```

## Update

### 功能

内部调用 vm.\_\_pathc\_\_() 把虚拟 DOM 转换成真实 DOM

### 定义

- src/core/instance/lifecycle.js

  ```js
    // _update 方法的作用是把 VNode 渲染成真实的 DOM
    // 首次渲染会调用，数据更新会调用
    Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
      const vm: Component = this
      const prevEl = vm.$el
      const prevVnode = vm._vnode
      const restoreActiveInstance = setActiveInstance(vm)
      vm._vnode = vnode
      // Vue.prototype.__patch__ is injected in entry points
      // based on the rendering backend used.
      if (!prevVnode) {
        // initial render
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
      } else {
        // updates
        vm.$el = vm.__patch__(prevVnode, vnode)
      }
      restoreActiveInstance()
      // update __vue__ reference
      if (prevEl) {
        prevEl.__vue__ = null
      }
      if (vm.$el) {
        vm.$el.__vue__ = vm
      }
      // if parent is an HOC, update its $el as well
      if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
        vm.$parent.$el = vm.$el
      }
      // updated hook is called by the scheduler to ensure that children are
      // updated in a parent's updated hook.
    }
  ```

## patch 函数初始化

### 功能

对比两个 VNode 的差异，把差异更新到真实的 DOM，如果是首次渲染的话，会把真实 DOM 先转换成 VNode

### Snabbdom 中的 patch 函数的初始化

- src/snabbdom.ts

  ```ts
  export function init (modules: Array<Partial<Module>>, domApi?: DOMAPI) {
    return function patch (oldVnode: VNode | Element, vnode: VNode): VNode {
    }
  }
  ```

- vnode  src/vnode.ts

  ```ts
  export function vnode(sel: string | undefined,
                        data: any | undefined,
                        children: Array<VNode | string> | undefined,
                        text: string | undefined,
                        elm: Element | Text | undefined): VNode {
    let key = data === undefined ? undefined : data.key;
    return {sel, data, children, text, elm, key};
  }
  ```

### Vue.js 中的 patch 函数的初始化

- src/platforms/web/runtime/index.js

  ```js
  import { patch } from './patch'
  
  // install platform patch function
  // path 把虚拟 DOM 转换为真实 DOM
  Vue.prototype.__patch__ = inBrowser ? patch : noop
  ```

- src/platforms/web/runtime/patch.js

  ```js
  import * as nodeOps from 'web/runtime/node-ops'
  import { createPatchFunction } from 'core/vdom/patch'
  import baseModules from 'core/vdom/modules/index'
  import platformModules from 'web/runtime/modules/index'
  
  // the directive module should be applied last, after all
  // built-in modules have been applied.
  const modules = platformModules.concat(baseModules)
  
  export const patch: Function = createPatchFunction({ nodeOps, modules })
  
  ```

- src/core/vdom/patch.js

  ```js
  export function createPatchFunction (backend) {
    let i, j
    const cbs = {}
    const { modules, nodeOps } = backend
    // 把模块中的钩子函数全部设置到 cbs 中，将来统一触发
    // cbs --> { 'create': [fn1, fn2], ... }
    for (i = 0; i < hooks.length; ++i) {
      cbs[hooks[i]] = []
      for (j = 0; j < modules.length; ++j) {
        if (isDef(modules[j][hooks[i]])) {
          cbs[hooks[i]].push(modules[j][hooks[i]])
        }
      }
    }
  ……
  ……
  ……
    return function patch (oldVnode, vnode, hydrating,      removeOnly) {
    }
  }
  
  ```

## patch 函数执行过程

- src/core/vdom/patch.js

```js
  // 函数柯里化，让一个函数返回一个函数
  // createPatchFunction({ nodeOps, modules }) 传入平台相关的两个参数

  // core 中的 createPatchFunction (backend), const { modules, nodeOps } = backend
  // core 中方法和平台无关，传入两个参数后，可以在上面的函数中使用这两个参数
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    // 新的 VNode 不存在
    if (isUndef(vnode)) {
      // 老的 VNode 存在，执行 Destroy 钩子函数
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    // 老的 VNode 不存在
    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      // 创建新的 VNode
      createElm(vnode, insertedVnodeQueue)
    } else {
      // 新的和老的 VNode 都存在，更新
      const isRealElement = isDef(oldVnode.nodeType)
      // 判断参数1是否是真实 DOM，不是真实 DOM
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // 更新操作，diff 算法
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        // 第一个参数是真实 DOM，创建 VNode
        // 初始化
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        // 创建 DOM 节点
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
```

## createElm

### 功能

把 VNode 转换成真实 DOM，插入到 DOM 树上

- src/core/vdom/patch.js

  ```js
    function createElm (
      vnode,
      insertedVnodeQueue,
      parentElm,
      refElm,
      nested,
      ownerArray,
      index
    ) {
      if (isDef(vnode.elm) && isDef(ownerArray)) {
        // This vnode was used in a previous render!
        // now it's used as a new node, overwriting its elm would cause
        // potential patch errors down the road when it's used as an insertion
        // reference node. Instead, we clone the node on-demand before creating
        // associated DOM element for it.
        vnode = ownerArray[index] = cloneVNode(vnode)
      }
  
      vnode.isRootInsert = !nested // for transition enter check
      if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
        return
      }
  
      const data = vnode.data
      const children = vnode.children
      const tag = vnode.tag
      if (isDef(tag)) {
        if (process.env.NODE_ENV !== 'production') {
          if (data && data.pre) {
            creatingElmInVPre++
          }
          if (isUnknownElement(vnode, creatingElmInVPre)) {
            warn(
              'Unknown custom element: <' + tag + '> - did you ' +
              'register the component correctly? For recursive components, ' +
              'make sure to provide the "name" option.',
              vnode.context
            )
          }
        }
  
        vnode.elm = vnode.ns
          ? nodeOps.createElementNS(vnode.ns, tag)
          : nodeOps.createElement(tag, vnode)
        setScope(vnode)
  
        /* istanbul ignore if */
        if (__WEEX__) {
          ...
        } else {
          createChildren(vnode, children, insertedVnodeQueue)
          if (isDef(data)) {
            invokeCreateHooks(vnode, insertedVnodeQueue)
          }
          insert(parentElm, vnode.elm, refElm)
        }
  
        if (process.env.NODE_ENV !== 'production' && data && data.pre) {
          creatingElmInVPre--
        }
      } else if (isTrue(vnode.isComment)) {
        vnode.elm = nodeOps.createComment(vnode.text)
        insert(parentElm, vnode.elm, refElm)
      } else {
        vnode.elm = nodeOps.createTextNode(vnode.text)
        insert(parentElm, vnode.elm, refElm)
      }
    }
  ```

## patchVnode

- src/core/vdom/patch.js

  ```js
    function patchVnode (
      oldVnode,
      vnode,
      insertedVnodeQueue,
      ownerArray,
      index,
      removeOnly
    ) {
      // 如果新旧节点是完全相同的节点，直接返回
      if (oldVnode === vnode) {
        return
      }
  
      if (isDef(vnode.elm) && isDef(ownerArray)) {
        // clone reused vnode
        vnode = ownerArray[index] = cloneVNode(vnode)
      }
  
      const elm = vnode.elm = oldVnode.elm
  	......
      // 触发 prepatch 钩子函数
      let i
      const data = vnode.data
      if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
        i(oldVnode, vnode)
      }
  
      // 获取新旧 VNode 的子节点
      const oldCh = oldVnode.children
      const ch = vnode.children
      if (isDef(data) && isPatchable(vnode)) {
        // 调用 cbs 中的钩子函数，操作节点的属性/样式/事件...
        // 触发update 钩子函数
        for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
        // 用户的自定义钩子
        if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
      }
  
      // 如果 vnode 没有 text 属性(说明有可能有子元素)
      if (isUndef(vnode.text)) {
        // 新节点和老节点都有子节点
        // 对子节点进行 diff 操作，调用 updateChildren
        if (isDef(oldCh) && isDef(ch)) {
          if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
        } else if (isDef(ch)) {
          // 新的有子节点，老的没有子节点
          if (process.env.NODE_ENV !== 'production') {
            checkDuplicateKeys(ch)
          }
          // 先清空老节点 DOM 的文本内容，然后为当前 DOM 节点加入子节点
          if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
          addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
        } else if (isDef(oldCh)) {
          // 老节点有子节点，新节点没有子节点
          // 删除老节点中的子节点
          removeVnodes(oldCh, 0, oldCh.length - 1)
        } else if (isDef(oldVnode.text)) {
          // 老节点有文本，新节点没有文本
          // 修改文本
          nodeOps.setTextContent(elm, '')
        }
      } else if (oldVnode.text !== vnode.text) {
        // 新老节点都有文本节点
        // 修改文本
        nodeOps.setTextContent(elm, vnode.text)
      }
      if (isDef(data)) {
        if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
      }
    }
  ```

## updateChildren

updateChildren 和 Snabbdom 中的 updateChildren 整体算法一致。

key 的作用，在 patch 函数中，调用patchVnode 之前，会首先调用 sameVnode() 判断当前的新老 VNode 是否是相同节点，sameVnode() 中会首先判断 key 是否相同

```html
    <div id="app">
      <button @click="handler">button</button>
      <ul>
        <!-- <li v-for="value in arr">{{ value }}</li> -->
        <li v-for="value in arr" :key="value">{{ value }}</li>
      </ul>
    </div>
    <script src="../../dist/vue.js"></script>
    <script>
      const vm = new Vue({
        el: "#app",
        data: {
          arr: ['a', 'b', 'c', 'd']
        },
        methods: {
          handler () {
            this.arr.splice(1, 0, 'x')
            // this.arr = ['a', 'x', 'b', 'c', 'd']
          }
        }
      });
    </script>
```



- 当没有设置 key 的时候

  在 updateChildren 中比较子节点的时候，会做三次更新 DOM 操作和一次插入 DOM 的操作

- 当设置 key 的时候

  在 updateChildren 中比较子节点的时候，因为 oldVnode 的子节点的 b,c, d 和 newVnode 的 x, b, c 的 key 相同，所以只做比较，没有更新 DOM 的操作， 当遍历完毕后，会再把 x 插入到 DOM 上的 DOM 操作只有一次插入操作

--------

# Vue.js 源码剖析——模板编译和组件化

## 模板编译简介

- 模板编译的主要目的是将模板（template）转换为渲染函数（render)

模板编译的作用

- Vue 2.x 使用 VNode 描述视图以及各种交互，用户自己编写 VNode 比较复杂
- 用户只需要编写类似 HTML 的代码—— Vue.js 模板，通过编译器将模板转换为返回 VNode 的 render 函数
- .vue 文件会被 webpack 在构建的过程中转换成 render 函数

## 什么是抽象语法树

- 抽象语法树简称 AST（Abstract Syntax Tree）
- 适用对象的形式描述树形的代码结构
- 次数的抽象语法树是用来描述树形结构的 HTMl 字符串

## 为什么要使用抽象语法树

- 模板字符串转成 AST 后，可以通过 AST 对模板做优化处理
- 标记模板中的静态内容，在 patch 的时候直接跳过静态内容
- 在 patch 的过程中静态内容不需要对比和重新渲染

## 组件化回顾

- 一个 Vue 组件就是一个拥有与定义选项的一个 Vue 实例
- 一个组件可以组成页面上一个功能完备的区域，组件可以包含脚本、样式、模板

## 组件注册方式

- 全局组件

  ```html
  <div id="app">
  </div>
  <script>
  	const Comp = Vue.component('comp', {
  		template: '<div>Hello component</div>'
  	})
  	const vm = new Vue({
  		el: '#app',
  		render (h) {
  			return h(comp)
  		}
  	})
  </script>
  ```

  

- 局部组件