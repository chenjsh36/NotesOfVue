## src/transition


>With Vue.js’ transition system you can apply automatic transition effects when elements are inserted into or removed from the DOM. Vue.js will automatically add/remove CSS classes at appropriate times to trigger CSS transitions or animations for you, and you can also provide JavaScript hook functions to perform custom DOM manipulations during the transition.

当元素插入到 DOM 树或者从 DOM 树中移除的时候， transition 属性提供变换的效果，可以使用 css 来定义变化效果，也可以使用 JS 来定义

### src/transition/index.js

```
import {
  before,
  remove,
  transitionEndEvent
} from '../util/index'

/**
 * Append with transition.
 *
 * @param {Element} el
 * @param {Element} target
 * @param {Vue} vm
 * @param {Function} [cb]
 */

export function appendWithTransition (el, target, vm, cb) {
  applyTransition(el, 1, function () {
    target.appendChild(el)
  }, vm, cb)
}
...
```

首先第一个函数是将元素插入 DOM， 函数实现调用了 applyTransition, 实现代码如下：

```
/**
 * Apply transitions with an operation callback.
 *
 * @param {Element} el
 * @param {Number} direction
 *                  1: enter
 *                 -1: leave
 * @param {Function} op - the actual DOM operation
 * @param {Vue} vm
 * @param {Function} [cb]
 */

export function applyTransition (el, direction, op, vm, cb) {
  var transition = el.__v_trans
  if (
    !transition ||
    // skip if there are no js hooks and CSS transition is
    // not supported
    (!transition.hooks && !transitionEndEvent) ||
    // skip transitions for initial compile
    !vm._isCompiled ||
    // if the vm is being manipulated by a parent directive
    // during the parent's compilation phase, skip the
    // animation.
    (vm.$parent && !vm.$parent._isCompiled)
  ) {
    op()
    if (cb) cb()
    return
  }
  var action = direction > 0 ? 'enter' : 'leave'
  transition[action](op, cb)
}

```

写的好的代码就是文档，从注释和命名上就能很好的理解这个函数的作用， el 是要操作的元素， direction 代表是插入还是删除， op 代表具体的操作方法函数， vm 从之前的代码或者官方文档可以知道指 vue 实例对象， cb 是回调函数

vue 将解析后的transition作为 DOM 元素的属性 **__v_trans** ，这样每次操作 DOM 的时候都会做以下判断：

* 如果元素没有被定义了 transition
* 如果元素没有 jshook 且 css transition 的定义不支持
* 如果元素还没有编译完成
* 如果元素有父元素且父元素没有编译完成

存在以上其中一种情况的话则直接执行操作方法 op 而不做变化，否则执行：

```
var action = direction > 0 ? 'enter' : 'leave'
transition[action](op, cb)
```

除了添加，还有插入和删除两个操作方法：
```
export function beforeWithTransition (el, target, vm, cb) {
  applyTransition(el, 1, function () {
    before(el, target)
  }, vm, cb)
}

export function removeWithTransition (el, vm, cb) {
  applyTransition(el, -1, function () {
    remove(el)
  }, vm, cb)
}
```

那么 transitoin 即 **el.__v_trans** 是怎么实现的，这个还得继续深挖


### src/transition/queue.js
```
import { nextTick } from '../util/index'

let queue = []
let queued = false

/**
 * Push a job into the queue.
 *
 * @param {Function} job
 */

export function pushJob (job) {
  queue.push(job)
  if (!queued) {
    queued = true
    nextTick(flush)
  }
}

/**
 * Flush the queue, and do one forced reflow before
 * triggering transitions.
 */

function flush () {
  // Force layout
  var f = document.documentElement.offsetHeight
  for (var i = 0; i < queue.length; i++) {
    queue[i]()
  }
  queue = []
  queued = false
  // dummy return, so js linters don't complain about
  // unused variable f
  return f
}

```

这是 transition 三个文件中的第二个，从字面量上理解是一个队列，从代码上看实现的是一个任务队列，每当调用 **pushJob** 的时候，都会往任务队列 **queue** 里面推一个任务，并且有一个标识**queued**, 如果为 **false** 则会在 **nextTick** 的时候将 **queued** 置为 **true**同时调用 **flush** 方法，这个方法会执行所有在任务队列 **queue** 的方法，并将 **queued** 置为 **false**

还记得 **nextTick** 的实现吗？实现在 **src/util/env** 中：

```

/**
 * Defer a task to execute it asynchronously. Ideally this
 * should be executed as a microtask, so we leverage
 * MutationObserver if it's available, and fallback to
 * setTimeout(0).
 *
 * @param {Function} cb
 * @param {Object} ctx
 */

export const nextTick = (function () {
  var callbacks = []
  var pending = false
  var timerFunc
  function nextTickHandler () {
    pending = false
    var copies = callbacks.slice(0)
    callbacks = []
    for (var i = 0; i < copies.length; i++) {
      copies[i]()
    }
  }
  /* istanbul ignore if */
  if (typeof MutationObserver !== 'undefined') {
    var counter = 1
    var observer = new MutationObserver(nextTickHandler)
    var textNode = document.createTextNode(counter)
    observer.observe(textNode, {
      characterData: true
    })
    timerFunc = function () {
      counter = (counter + 1) % 2
      textNode.data = counter
    }
  } else {
    timerFunc = setTimeout
  }
  return function (cb, ctx) {
    var func = ctx
      ? function () { cb.call(ctx) }
      : cb
    callbacks.push(func)
    if (pending) return
    pending = true
    timerFunc(nextTickHandler, 0)
  }
})()

```

官网的解释如下
> Defer the callback to be executed after the next DOM update cycle. Use it immediately after you’ve changed some data to wait for the DOM update.

即在下一次 DOM 更新循环中执行回调，用在你需要等待 DOM 节点更新后才能执行的情况，实现的简单方法是利用 setTimeout 函数，我们知道 setTimeout 方法会将回调函数放入时间队列里，并在计时结束后放到事件队列里执行，从而实现异步执行的功能，当然尤大只把这种情况作为备用选择，而采用模拟DOM创建并利用观察者MutationObserver监听其更新来实现：
```
var observer = new MutationObserver(nextTickHandler) // 创建一个观察者
var textNode = document.createTextNode(counter) // 创建一个文本节点
observer.observe(textNode, { // 监听 textNode 的 characterData 是否为 true
  characterData: true
})
timerFunc = function () { // 每次调用 nextTick，都会调用timerFunc从而再次更新文本节点的值
  counter = (counter + 1) % 2 // 值一直在0和1中切换，有变化且不重复
  textNode.data = counter
}
```

不了解MutationObserver 和 characterData 的可以参考MDN的解释： [MutaitionObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)
 & [CharacterData](https://developer.mozilla.org/zh-CN/docs/Web/API/CharacterData)

[mutationObserver 例子](https://jsfiddle.net/chenjsh36/1b5LL0b7/)


**flush** 函数声明变量f: ```var f = document.documentElement.offsetHeight``` 从注释上看应该是强制DOM更新，因为调用offsetHeight的时候会让浏览器重新计算出文档的滚动高度的缘故吧

### src/transition/transition.js

