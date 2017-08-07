> 继续看 vue 的工具类实现

## 工具类

```
/**
 * Simple bind, faster than native
 *
 * @param {Function} fn
 * @param {Object} ctx
 * @return {Function}
 */

export function bind (fn, ctx) {
  return function (a) {
    var l = arguments.length
    return l
      ? l > 1
        ? fn.apply(ctx, arguments)
        : fn.call(ctx, a)
      : fn.call(ctx)
  }
}
```

绑定函数，利用 apply 和 call 方法进行 this 的绑定，

```
/**
 * Check if two values are loosely equal - that is,
 * if they are plain objects, do they have the same shape?
 *
 * @param {*} a
 * @param {*} b
 * @return {Boolean}
 */

export function looseEqual (a, b) {
  /* eslint-disable eqeqeq */
  return a == b || (
    isObject(a) && isObject(b)
      ? JSON.stringify(a) === JSON.stringify(b)
      : false
  )
```

### debug 类

```
import config from '../config'

let warn

if (process.env.NODE_ENV !== 'production') {
  const hasConsole = typeof console !== 'undefined'
  warn = function (msg, e) {
    if (hasConsole && (!config.silent || config.debug)) {
      console.warn('[Vue warn]: ' + msg)
      /* istanbul ignore if */
      if (config.debug) {
        if (e) {
          throw e
        } else {
          console.warn((new Error('Warning Stack Trace')).stack)
        }
      }
    }
  }
}

export { warn }

```

debug 类有意思的是根据环境是否是生产环境来确定
### dom 类
#### dom 元素插入删除替换操作
用惯了 JQ， 还记得怎么用原生函数添加元素么？
```


/**
 * Insert el before target
 *
 * @param {Element} el
 * @param {Element} target
 */

export function before (el, target) {
  target.parentNode.insertBefore(el, target)
}

/**
 * Insert el after target
 *
 * @param {Element} el
 * @param {Element} target
 */

export function after (el, target) {
  if (target.nextSibling) {
    before(el, target.nextSibling)
  } else {
    target.parentNode.appendChild(el)
  }
}

/**
 * Remove el from DOM
 *
 * @param {Element} el
 */

export function remove (el) {
  el.parentNode.removeChild(el)
}

/**
 * Prepend el to target
 *
 * @param {Element} el
 * @param {Element} target
 */

export function prepend (el, target) {
  if (target.firstChild) {
    before(el, target.firstChild)
  } else {
    target.appendChild(el)
  }
}

/**
 * Replace target with el
 *
 * @param {Element} target
 * @param {Element} el
 */

export function replace (target, el) {
  var parent = target.parentNode
  if (parent) {
    parent.replaceChild(el, target)
  }
}
```

#### 元素类的添加删除

```
/**
 * Add class with compatibility for IE & SVG
 *
 * @param {Element} el
 * @param {Strong} cls
 */

export function addClass (el, cls) {
  if (el.classList) {
    el.classList.add(cls)
  } else {
    var cur = ' ' + (el.getAttribute('class') || '') + ' '
    if (cur.indexOf(' ' + cls + ' ') < 0) {
      el.setAttribute('class', (cur + cls).trim())
    }
  }
}

/**
 * Remove class with compatibility for IE & SVG
 *
 * @param {Element} el
 * @param {Strong} cls
 */

export function removeClass (el, cls) {
  if (el.classList) {
    el.classList.remove(cls)
  } else {
    var cur = ' ' + (el.getAttribute('class') || '') + ' '
    var tar = ' ' + cls + ' '
    while (cur.indexOf(tar) >= 0) {
      cur = cur.replace(tar, ' ')
    }
    el.setAttribute('class', cur.trim())
  }
  if (!el.className) {
    el.removeAttribute('class')
  }
}
```

对元素的类的操作在老版本只支持 setAttribute 来操作，而随着版本的更新，浏览器开始支持 classList 属性 的 add  和 remove 方法来操作元素的类，但依旧要兼容旧版本的浏览器

#### DocumentFragment

```
/**
 * Extract raw content inside an element into a temporary
 * container div
 *
 * @param {Element} el
 * @param {Boolean} asFragment
 * @return {Element}
 */
export function extractContent (el, asFragment) {
  var child
  var rawContent
  /* istanbul ignore if */
  if (
    isTemplate(el) &&
    el.content instanceof DocumentFragment
  ) {
    el = el.content
  }
  if (el.hasChildNodes()) {
    trimNode(el)
    rawContent = asFragment
      ? document.createDocumentFragment()
      : document.createElement('div')
    /* eslint-disable no-cond-assign */
    while (child = el.firstChild) {
    /* eslint-enable no-cond-assign */
      rawContent.appendChild(child)
    }
  }
  return rawContent
}
```

>DocumentFragment 接口表示一个没有父级文件的最小文档对象。它被当做一个轻量版本的 Document 使用，用于存储已排好版的或尚未打理好格式的XML片段。最大的区别是因为DocumentFragment不是真实DOM树的其中一部分，它的变化不会引起DOM树的重新渲染的操作(reflow) ，或者导致性能影响的问题出现。

更多信息可以参考[MDN 文档](https://developer.mozilla.org/zh-CN/docs/Web/API/DocumentFragment)


#### 元素的生命周期相关函数

```
import { removeWithTransition } from '../transition/index'

....

/**
 * Map a function to a range of nodes .
 *
 * @param {Node} node
 * @param {Node} end
 * @param {Function} op
 */

export function mapNodeRange (node, end, op) {
  var next
  while (node !== end) {
    next = node.nextSibling
    op(node)
    node = next
  }
  op(end)
}

/**
 * Remove a range of nodes with transition, store
 * the nodes in a fragment with correct ordering,
 * and call callback when done.
 *
 * @param {Node} start
 * @param {Node} end
 * @param {Vue} vm
 * @param {DocumentFragment} frag
 * @param {Function} cb
 */

export function removeNodeRange (start, end, vm, frag, cb) {
  var done = false
  var removed = 0
  var nodes = []
  mapNodeRange(start, end, function (node) {
    if (node === end) done = true
    nodes.push(node)
    removeWithTransition(node, vm, onRemoved)
  })
  function onRemoved () {
    removed++
    if (done && removed >= nodes.length) {
      for (var i = 0; i < nodes.length; i++) {
        frag.appendChild(nodes[i])
      }
      cb && cb()
    }
  }
}

```

两个函数实现的是将一段范围内的元素删除的实现


### env 类

env 提供了监测是否是浏览器环境？属于IE？ 属于安卓？css transition 和 animation 是否需要 webkit 前缀？

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
最后一个关于 nextTick  的实现暂时看不懂

### options 类

