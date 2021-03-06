# vue2简单实现
```
import { observe } from './observe'
import { Compile } from './compile'

class MyVue {
  constructor (options) {
    // 初始化 这里省略一些初始化工作...
    this.data = options.data
    this.methods = options.methods
    // 代理对象属性
    this._proxy(options.data)
    // 添加数据监听
    observe(this.data)
    // 模板解析: parse, optimize, generate, 其中 watcher 写在 complile 中了
    // 当 解析生成的render function 被渲染的时候，因为会读取所需对象的值，
    // 所以会触发 getter 函数进行依赖收集，依赖收集的目的是
    // 将观察者 Watcher 对象存放到当前闭包中的订阅者 Dep 的 subs 中。
    new Compile(options.el, this)
    // 挂载DOM
    options.mounted.call(this)
  }

  // 将 key 代理待 this 上
  _proxy (data) {
    const that = this
    Object.keys(data).forEach(key => {
      Object.defineProperties(that, key, {
        enumerable: true,
        configurable: true,
        get () {
          return that.data[key]
        },
        set (nv) {
          that.data[key] = nv
        }
      })
    })
  }
}

```
# vue3响应式原理 实现 reactive()
- 前置知识回顾
proxy, Reflect, weakSet, weakMap...
```
/**
 * vue3 响应式前置知识回顾
 */

// ***** Set, new Set([iterable]) *****
let set = new Set()
// Prop: size
// API: add has delete clear
set.add({ hello: 1 }).add({ hello: 1 }).add(1)
// 迭代器遍历, for...of, forEach, set.keys(), set.values()等
// for (const item of set.keys()) {
//   console.log(item)
// }

// ***** WeakSet, new WeakSet([iterable]) *****
// WeakSet 的成员只能是 对象! 而不能是其他类型的值
// WeakSet 中的对象都是弱引用,, 垃圾回收机制不考虑 WeakSet 对该对象的引用
// WeakSet 里面的引用都不计入垃圾回收机制, 因此适合临时存放一组对象，以及存放跟对象绑定的信息。
// 只要这些对象在外部消失，它在 WeakSet 里面的引用就会自动消失; 且不可枚举
let wset = new WeakSet()
// Prop: 无
// API: add has delete
let o1 = { h: 1 }
wset.add(o1)
console.log(wset.has(o1))

// ***** Map, new Map([iterable]) *****
// Map 的键实际上是跟内存地址绑定的，只要内存地址不一样，就视为两个键, 所以事先用变量来声明
let map = new Map([
  ['title', 'map']
])
// Prop: length, Map.prototype.size
// API: set get has delete clear 迭代方法forEach, keys()...
// Map与对象,数组互转 http://es6.ruanyifeng.com/#docs/set-map
console.log(map)

// ***** WeakMap, new WeakMap([iterable]) *****
// WeakMap只接受对象作为键名（null除外)
// WeakMap的键名所指向的对象，不计入垃圾回收机制,
// 当发生多处对象引用时, 被引用的对象被清除时就会自动在内存中被释放, 无需做手动清除
// 只要外部的引用消失，WeakMap 内部的引用，就会自动被垃圾回收清除
let wmap = new WeakMap()
// API: get()、set()、has()、delete()
let o = { 'name': 1 }
wmap.set(o, 11111)
console.log(wmap, wmap.get(o))
o = null
console.log(wmap)

/**
 * Proxy 代理器
 * 在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，
 * 因此提供了一种机制，可以对外界的访问进行过滤和改写
 * http://es6.ruanyifeng.com/#docs/proxy
 */

// 语法
let obj = { name: 'huhua', age: 25 }
let handler = {
  get (target, key) {
    return `${key} is visited`
  }
}
let proxy = new Proxy(obj, handler)
// 在操作对象前先经过一层代理
console.log(proxy.name) // name is visited

// Proxy 支持的拦截操作一览，一共 13 种。handler对象中 { 8 }

// get(target, propKey, receiver)：拦截对象属性的读取，比如proxy.foo和proxy['foo']。
// set(target, propKey, value, receiver)：拦截对象属性的设置，比如proxy.foo = v或proxy['foo'] = v，返回一个布尔值。
// has(target, propKey)：拦截propKey in proxy的操作，返回一个布尔值。
// deleteProperty(target, propKey)：拦截delete proxy[propKey]的操作，返回一个布尔值。
// ownKeys(target)：拦截Object.getOwnPropertyNames(proxy)、Object.getOwnPropertySymbols(proxy)、Object.keys(proxy)、for...in循环，返回一个数组。该方法返回目标对象所有自身的属性的属性名，而Object.keys()的返回结果仅包括目标对象自身的可遍历属性。
// getOwnPropertyDescriptor(target, propKey)：拦截Object.getOwnPropertyDescriptor(proxy, propKey)，返回属性的描述对象。
// defineProperty(target, propKey, propDesc)：拦截Object.defineProperty(proxy, propKey, propDesc）、Object.defineProperties(proxy, propDescs)，返回一个布尔值。
// preventExtensions(target)：拦截Object.preventExtensions(proxy)，返回一个布尔值。
// getPrototypeOf(target)：拦截Object.getPrototypeOf(proxy)，返回一个对象。
// isExtensible(target)：拦截Object.isExtensible(proxy)，返回一个布尔值。
// setPrototypeOf(target, proto)：拦截Object.setPrototypeOf(proxy, proto)，返回一个布尔值。如果目标对象是函数，那么还有两种额外操作可以拦截。
// apply(target, object, args)：拦截 Proxy 实例作为函数调用的操作，比如proxy(...args)、proxy.call(object, ...args)、proxy.apply(...)。
// construct(target, args)：拦截 Proxy 实例作为构造函数调用的操作，比如new proxy(...args)。

/**
 * Reflect对象, 替代原型的操作的方式, 简化代码
 * Reflect是一个内置的对象，它提供拦截 JavaScript 操作的方法
 * Reflect对象一共有 13 个静态方法, 而且它与Proxy handler对象的方法是一一对应的
 */

// Reflect对象一共有 13 个静态方法, 而且它与Proxy对象的方法是一一对应的
// Reflect.apply(target, thisArg, args) 对一个函数进行调用操作，同时可以传入一个数组作为调用参数
// Reflect.construct(target, args) 对构造函数进行 new 操作，相当于执行 new target(...args)
// Reflect.get(target, name, receiver) 获取对象身上某个属性的值，类似于 target[name]
// Reflect.set(target, name, value, receiver) 将值分配给属性的函数。返回一个Boolean，如果更新成功，则返回true
// Reflect.defineProperty(target, name, desc) 和 Object.defineProperty() 类似
// Reflect.deleteProperty(target, name) 作为函数的delete操作符，相当于执行 delete target[name]
// Reflect.has(target, name) 判断一个对象是否存在某个属性，和 in 运算符 的功能完全相同
// Reflect.ownKeys(target) 返回一个包含所有自身属性（不包含继承属性）的数组
// Reflect.isExtensible(target) 类似于 Object.isExtensible()
// Reflect.preventExtensions(target) 类似于 Object.preventExtensions()
// Reflect.getOwnPropertyDescriptor(target, name) 类似于 Object.getOwnPropertyDescriptor()
// Reflect.getPrototypeOf(target) 类似于 Object.getPrototypeOf()
// Reflect.setPrototypeOf(target, prototype) 类似于 Object.setPrototypeOf()

// 获取
console.log(Reflect.get(Object, 'assign'))
// 设置
let oo = Object.create(null)
Reflect.set(oo, 'name', 'hello world')
console.log(oo)
// Object 是否存在 assign 属性
console.log(Reflect.has(Object, 'assign'))
// ...

// 简单的观察者模式
const data = {
  name: 'huhua',
  age: 25
}

const queue = new Set()

// 监听
function observer (obj) {
  return new Proxy(obj, {
    set (target, key, value, receiver) {
      // Reflect.set一旦传入receiver，就会将属性赋值到receiver上面（即proxy实例)
      const result = Reflect.set(target, key, value, receiver)
      // 执行相关操作
      queue.forEach(watcher => watcher())

      return result
    }
  })
}
// 观测, 执行回调
function watcher (cb) {
  queue.add(cb)
}
function change () { console.log(`${data.name} is changed`) }
watcher(change)
let proxyData = observer(data)
proxyData.name = 'hello world' // hello world is changed

```
- 相关代码
```
// 负责记录所有obj的依赖, 类似 Dep 的作用
const objMap = new WeakMap()
// Symbol创建一个唯一值来作为target一个特殊的key，我们去循环我们的代理对象时，
// 这个key会收集依赖，当我们的代理对象有新增Key或删除Key时触发 相关依赖
const ONLY_KEY = Symbol('only')

let activeEffect = null // 当前正在处理的依赖

// 触发 getter 时收集依赖
function track (obj, type, key) {
  if (!activeEffect) return
  // target obj
  let depsMap = objMap.get('obj')
  if (!depsMap) {
    depsMap = new Map()
    objMap.set(obj, depsMap)
  }

  // target key
  let deps = depsMap.get(key)
  if (!deps) {
    deps = new Set()
    depsMap.set(key, deps)
  }

  // 记录
  deps.add(key, deps)
  activeEffect.deps.push(deps)
  activeEffect.options.onTrack({
    effect: activeEffect,
    obj,
    type,
    key
  })
}

// 触发 setter 是触发依赖更新
function trigger (obj, type, key) {
  const depsMap = objMap.get(obj)
  if (depsMap) {
    if (type === 'add' || type === 'delete') {
      key = ONLY_KEY
    }
    const deps = depsMap.get(key)
    if (deps) {
      depsMap.forEach(effect => {
        effect.options.onTrigger(effect, obj, type, key)
        effect()
      })
    }
  }
}
// reactive可以理解为vue的Observer，负责将复杂对象转换成响应式数据
function reactive (obj) {
  return new Proxy(obj, handler)
}

// effect可以理解为vue的Watcher, 即依赖追踪,观察者
function effect (cb, options) {
  const effect = () => {
    // 缓存
    activeEffect = effect
    const res = cb()
    activeEffect = null
    return res
  }
  effect.deps = []
  effect.options = options
  effect()
  return effect
}

// proxy 的 handler
const handler = {
  get (obj, key, receiver) {
    const res = Reflect.get(obj, key)
    // 收集,追踪依赖
    track(obj, 'get', key)
    return typeof res === 'object' ? reactive(res) : res
  },
  set (obj, key, value, receiver) {
    const hasKey = Object.prototype.hasOwnProperty.call(obj, key)
    const res = Reflect.set(obj, key, value)
    // 捕捉,更新依赖
    trigger(obj, hasKey ? 'set' : 'add', key)
    return res
  }
}

export default {
  reactive,
  effect
}

// demo
let data = {
  name: 'huhua',
  age: 25
}

let reactiveData = reactive(data)

// effect(fn, options)
// fn中读取的Key会收集依赖，当key被设置时触发，并再次执行fn
let options = {
  onTrack (effect, target, key, type) {
    // key被读取时的回调
    // console.log(...arguments)
    console.log(target, key)
    // console.log('this is my origin name', target[key])
  },
  onTrigger (effect, target, key, type) {
    // key被设置时的回调
    console.log(...arguments)
    // console.log('this is my change name', target[key])
  }
}
effect(() => {
  Object.keys(reactiveData).forEach(key => {
    console.log(key, reactiveData[key])
  })
}, options)

reactiveData.name = 'vue3'

```