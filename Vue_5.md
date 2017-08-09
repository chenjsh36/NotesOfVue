### src/observer

### src/observer/dep.js

**dep** 被 **watcher** 引用


> A dep is an observable that can have multiple
directives subscribing to it.
(dep 是一个可以被多个指令订阅的观察者)

```
// src/observer/dep.js

import { toArray } from '../util/index'

let uid = 0

export default function Dep () {
  this.id = uid++
  this.subs = []
}

Dep.target = null

Dep.prototype.addSub = function (sub) {
  this.subs.push(sub)
}

Dep.prototype.removeSub = function (sub) {
  this.subs.$remove(sub)
}

Dep.prototype.depend = function () {
  Dep.target.addDep(this)
}

Dep.prototype.notify = function () {
  var subs = toArray(this.subs)
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}

```

代码量很少，但是几个点暂时不明，一个是 ```this.subs.$remove(sub)```, 貌似是给数组添加了一个 remove 方法， **target** 拥有 **addDep** 方法， **subs**的 item 拥有 **update** 方法


### src/observer/array.js

实现了 **arrayMethods** 类，该类继承了 **array** 对象：
```
...
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)
...
```

同时还对 **array** 拓展了 **$set $remove** 方法， 这就是在 **dep.js** 里使用到的 **$remove** 方法
```

def(
  arrayProto,
  '$set',
  function $set (index, val) {
    if (index >= this.length) {
      this.length = index + 1
    }
    return this.splice(index, 1, val)[0]
  }
)
def(
  arrayProto,
  '$remove',
  function $remove (item) {
    /* istanbul ignore if */
    if (!this.length) return
    var index = indexOf(this, item)
    if (index > -1) {
      return this.splice(index, 1)
    }
  }
)

```

**def** 方法的实现：
```
export function def (obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}
```

除此之外， arrayMethods 重写了 
```
'push',
'pop',
'shift',
'unshift',
'splice',
'sort',
'reverse'
```

这些原型方法：

```
;[
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
.forEach(function (method) {
  // cache original method
  var original = arrayProto[method]
  def(arrayMethods, method, function mutator () {
    // avoid leaking arguments: 避免泄露参数 arguments
    // http://jsperf.com/closure-with-arguments
    var i = arguments.length
    var args = new Array(i)
    while (i--) {
      args[i] = arguments[i]
    }
    var result = original.apply(this, args)
    var ob = this.__ob__
    var inserted
    switch (method) {
      case 'push':
        inserted = args
        break
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})

```
通过改写数组的几个原型方法，从而能够在开发者操作 data 数组时，能够触发更新事件：** ob.dep.notify()**

不太懂的几点是：
* **while** 循环是倒序的，也就是**args** 复制的是 arguments 倒序后的数组
* **ob.observeArray** 暂时不确定是什么功能

### src/observer/index.js

**index.js** 实现了 **Observer 类**， 观察者类绑定了每一个被观察的对象，一旦绑定，Observer 类会将目标对象的属性**property keys** 转化为 getter/setters, 以收集依赖关系和分派更新

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
...
```

从构造函数可以知道之前代码里一直出现的 **\__ob__** 属性就是指 Observer 类

其中 hasProto 实现在 util/env 中：

```
export const hasProto = '__proto__' in {}
```

实现的原理是利用 Object 的 Geeter 和 Setter：

```

/**
 * Define a reactive property on an Object.
 *
 * @param {Object} obj
 * @param {String} key
 * @param {*} val
 */

export function defineReactive (obj, key, val) {
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

  var childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
        }
        if (isArray(value)) {
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
      childOb = observe(newVal)
      dep.notify()
    }
  })
}
```
