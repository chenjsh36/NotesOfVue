## 数据响应

Vue 的实例构造函数非常简单粗暴：

```
// src/instance/vue.js

function Vue (options) {
  this._init(options)
}
```

而 _init 函数里声明了一堆实例属性包括 $el, $parent, $root 等，然后初始化数据、触发钩子 、初始化 event， **\_initState** 初始化数据，让 VUE 能够监听到数据的变化，而 **$mount()** 方法则承载了绝大部分的代码量，负责模板的嵌入、编译、link、指令和watcher的生成、批处理的执行等等。 

```
// src/instance/internal/init.js
    ...
    
    // set ref
    this._updateRef()

    // initialize data as empty object.
    // it will be filled up in _initScope().
    this._data = {}

    // call init hook
    this._callHook('init')

    // initialize data observation and scope inheritance.
    this._initState()

    // setup event system and option events.
    this._initEvents()

    // call created hook
    this._callHook('created')

    // if `el` option is passed, start compilation.
    if (options.el) {
      this.$mount(options.el)
    }
...

```

响应数据的实现部分依赖于 **Object.defineProperty**, 通过遍历传入 data 的每一个属性：

* 如果是普通属性，直接使用 defineProperty 处理；
* 如果属性还是一个对象，递归遍历这个对象
* 如果是数组，递归处理数组的元素

处理的时候也就是使用 Object.defineProperty， 我们可以改写属性的 getter/setter，这样当改写了对象的属性的值的时候，就有办法监听到并作出响应。VUE 的具体实现如下：

```
// src/instance/internal/state.js

...

import {
  observe,
  defineReactive
} from '../../observer/index'

...

  // _init 调用 _initState 做数据的初始化
  Vue.prototype._initState = function () {
    this._initProps()
    this._initMeta()
    this._initMethods()
    this._initData() // 实现了响应化数据
    this._initComputed()
  }
  
  ...

  Vue.prototype._initData = function () {
    var propsData = this._data
    var optionsDataFn = this.$options.data
    var optionsData = optionsDataFn && optionsDataFn() // 开发者传入的 data
    if (optionsData) {
      this._data = optionsData
      for (var prop in propsData) {
        if (process.env.NODE_ENV !== 'production' &&
            hasOwn(optionsData, prop)) {
          warn(
            'Data field "' + prop + '" is already defined ' +
            'as a prop. Use prop default value instead.'
          )
        }
        if (this._props[prop].raw !== null ||
            !hasOwn(optionsData, prop)) {
          set(optionsData, prop, propsData[prop])
        }
      }
    }
    var data = this._data
    // proxy data on instance
    var keys = Object.keys(data)
    var i, key
    i = keys.length
    while (i--) {
      key = keys[i]
      this._proxy(key) // 这里代理 key： this[key] = this._data[key]
    }
    // observe data
    observe(data, this) // 具体实现
  }
  
```

observe 的实现在 **src/observer/index.js**, index.js 实现了 Observer 对象和相关方法如 **observe** 方法：

```
export function observe (value, vm) {
  if (!value || typeof value !== 'object') { // 传入的只能为对象
    return
  }
  var ob
  if (
    // 如果这个数据身上已经有 __ob__，已经实现监听，则直接返回 ob
    hasOwn(value, '__ob__') &&
    value.__ob__ instanceof Observer
  ) {
    ob = value.__ob__
  } else if (
    (isArray(value) || isPlainObject(value)) &&
    !Object.isFrozen(value) &&
    !value._isVue
  ) {
    // 是对象的话就深入进去遍历属性， observe 每个属性
    ob = new Observer(value)
  }
  if (ob && vm) {
    // 将 vm 加入到 ob 的 vms 中， 使得 $set 和 $delete 操作时，提示 vm 实例这个行为的发生
    ob.addVm(vm)
  }
  return ob
}
```

通过 new 一个对象来实现： **ob = new Observer(value)**, Observer 构造函数的实现如下：

```

/**
 * Observer class that are attached to each observed
 * object. Once attached, the observer converts target
 * object's property keys into getter/setters that
 * collect dependencies and dispatches updates.
 *
 * @param {Array|Object} value
 * @constructor
 */

export function Observer (value) {
  this.value = value
  this.dep = new Dep()
  def(value, '__ob__', this)
  if (isArray(value)) { // 如果是数组
    var augment = hasProto
      ? protoAugment
      : copyAugment
    augment(value, arrayMethods, arrayKeys)
    this.observeArray(value) 
  } else { // 处理对象情况
    this.walk(value)
  }
}
```

如果是数组， 调用 **this.observeArray(value)**:

```
Observer.prototype.observeArray = function (items) {
  var i = items.length
  while (i--) {
    observe(items[i]) // 递归调用
  }
}
```

如果是对象， 调用 **this.walk(value)**:

```
Observer.prototype.walk = function (obj) {
  var keys = Object.keys(obj)
  var i = keys.length
  while (i--) {
    this.convert(keys[i], obj[keys[i]])
  }
}
Observer.prototype.convert = function (key, val) {
  defineReactive(this.value, key, val)
}
```

defineReactive 源码如下：
```
export function defineReactive (obj, key, val) {
  // 生成一个新的 Dep 实例，这个实例会被闭包到 getter 和 setter 中
  var dep = new Dep()

  // cater for pre-defined getter/setters
  var getter, setter
  if (config.convertAllProperties) {
    var property = Object.getOwnPropertyDescriptor(obj, key)
    if (property && property.configurable === false) {
      return
    }
    getter = property && property.get
    setter = property && property.set
  }
  // 对属性的值继续执行 observe， 属性的值如果是一个对象，继续递归对属性执行 defineReactive，实现了遍历所有层次的属性
  var childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val
      // 只在有 Dep.target 时才说明是 Vue 内部依赖收集过程触发的 getter
      // 那么这个时候就需要执行 dep.depend(), 将 watcher（Dep.target 的实际值)添加到 dep 的 subs 数组中
      // 对于其他情况，比如 DOM 事件回调函数中访问这个变量导致触发的 getter 并不需要执行依赖收集，直接返回 value 
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          // 如果 value 是对象， 让底层的 Observer 实例当中的 dep 也收集依赖
          childOb.dep.depend()
        }
        if (isArray(value)) {
          // 如果数组元素也是对象， 同理收集依赖
          for (var e, i = 0, l = value.length; i < l; i++) {
            e = value[i]
            e && e.__ob__ && e.__ob__.dep.depend()
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val
      if (newVal === value) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // observe 新的值
      childOb = observe(newVal)
      // 通知订阅了这个 dep 的 watchers 值更新了
      dep.notify()
    }
  })
}

```

以上就是Vue 实现响应化数据的具体过程。这里面涉及到一个订阅-观察者对象 Dep， Dep 类包含了一个 id、一个 subs 数组, subs 数组用来存放订阅了本身的观察者， subs 存放了订阅的订阅者 watcher

```
// src/observer/dep.js

export default function Dep () {
  this.id = uid++
  this.subs = [] // 存放订阅者 watchers
}

Dep.target = null // target 存放 watchers

Dep.prototype.depend = function () {
  Dep.target.addDep(this)
}

Dep.prototype.notify = function () {
  // stablize the subscriber list first
  var subs = toArray(this.subs)
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}

... 
```
这里列出了几个 Dep 的方法，在 getter 和 setter 里面用到了

dep 作为getter 和 setter 里的一个闭包，这意味着每一个被监听的属性都包含一个观察者 Dep， 所以每次获取数据或者修改数据的时候都会触发观察者订阅的订阅者回调函数, 在 getter 里面调用了 **dep.depend()**, 实际上是 **Dep.target.addDep(this)**, 说明target 是一个对象且只有一个，**addDep** 的实现如下：

```
// src/watcher.js"

Watcher.prototype.addDep = function (dep) {
  var id = dep.id
  if (!this.newDeps[id]) {
    this.newDeps[id] = dep // 是新的依赖否
    if (!this.deps[id]) { // 原始存在否
      this.deps[id] = dep
      dep.addSub(this) // 反过来将 Watcher 添加会 dep 
    }
  }
}

```

在 **setter** 里面调用的是 notify， 负责完成数据变动时通知订阅者，遍历所有订阅者并更新：
```
// src/watcher.js"
Dep.prototype.notify = function () {
  // stablize the subscriber list first
  var subs = toArray(this.subs)
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

在 Observer 构造函数中，也实例化了一个 dep: 
```
export function Observer (value) {
  this.value = value
  this.dep = new Dep()
  def(value, '__ob__', this)
  if (isArray(value)) {
    var augment = hasProto
      ? protoAugment
      : copyAugment
    augment(value, arrayMethods, arrayKeys)
    this.observeArray(value)
  } else {
    this.walk(value)
  }
}
```

且在 defineReactive 函数中的 getter 中执行了 childOb.dep.depend()

getter 和setter 的问题在于只能监听到属性的更改，不能监听到属性的删除和添加， Vue 的解决方法是提供外部API： vm.$set / vm.$delete / Vue.set / Vue.delete 以及数组的 $remove

例如执行 ```defineReactive(dasta, 'a'， {b:1})``` 时, 先创造了存放在 getter / setter 里面的 dep， 然后 ```var childOb = observe(val)``` 是 {b:1} , 会为 {b:1} 也创建一个 Ob， 并放在 val.__ob__ 上

watcher 则同时订阅了两个 dep， 实现 delete 的时候， 由于对像存放了属性 **__ob__**， 于是能通过 __ob__ 来调用 dep 来通知所有的 watcher 做更新


```
// src/util/lang.js
export function del (obj, key) {
  if (!hasOwn(obj, key)) {
    return
  }
  delete obj[key]
  var ob = obj.__ob__
  if (!ob) {
    return
  }
  ob.dep.notify()
  if (ob.vms) {
    var i = ob.vms.length
    while (i--) {
      var vm = ob.vms[i]
      vm._unproxy(key)
      vm._digest()
    }
  }
}
```
