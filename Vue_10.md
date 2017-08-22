在实例化 vue 时，初始化数据和事件后，就到了 $mount， $mount 的源码如下：

```
// src/instance/api/lifecycle.js

  Vue.prototype.$mount = function (el) {
    if (this._isCompiled) {
      process.env.NODE_ENV !== 'production' && warn(
        '$mount() should be called only once.'
      )
      return
    }
    el = query(el)
    if (!el) { // 如果挂载函数不存在，新建一个 div
      el = document.createElement('div')
    }
    this._compile(el) // 编译
    this._initDOMHooks() // 初始化钩子
    if (inDoc(this.$el)) { // 触发编译完成的钩子
      this._callHook('attached')
      ready.call(this)
    } else {
      this.$once('hook:attached', ready)
    }
    return this
  }
```

**_compile** 源码如下：
```

  Vue.prototype._compile = function (el) {
    var options = this.$options

    // 第一个阶段， 内嵌和初始化元素
    // transclude and init element
    // transclude can potentially replace original
    // so we need to keep reference; this step also injects
    // the template and caches the original attributes
    // on the container node and replacer node.
    var original = el
    el = transclude(el, options)
    this._initElement(el)

    // 第二阶段， 遍历模板解析出模板的指令 和 第三阶段， 解析指令
    // root is always compiled per-instance, because
    // container attrs and props can be different every time.
    var contextOptions = this._context && this._context.$options
    var rootLinker = compileRoot(el, options, contextOptions)

    // compile and link the rest
    var contentLinkFn
    var ctor = this.constructor
    // component compilation can be cached
    // as long as it's not using inline-template
    // 当为组件时
    if (options._linkerCachable) {
      contentLinkFn = ctor.linker
      if (!contentLinkFn) {
        contentLinkFn = ctor.linker = compile(el, options)
      }
    }

    
    // link phase
    // make sure to link root with prop scope!
    var rootUnlinkFn = rootLinker(this, el, this._scope)
    var contentUnlinkFn = contentLinkFn
      ? contentLinkFn(this, el)
      : compile(el, options)(this, el)

    // register composite unlink function
    // to be called during instance destruction
    this._unlinkFn = function () {
      rootUnlinkFn()
      // passing destroying: true to avoid searching and
      // splicing the directives
      contentUnlinkFn(true)
    }

    // finally replace original
    if (options.replace) {
      replace(original, el)
    }

    this._isCompiled = true
    this._callHook('compiled')
    return el
  }

```

**_compile** 包含了 Vue 构建的三个阶段： transclude、compile 和 link

transclude 的意思是内嵌，这个步骤会把开发者的 template 的模板转换成一段 dom， 然后抽取出 el选项指定的 dom 内容， 再把dom 嵌入到 el 里面或者替换 el ，关键在于把 template 转换成 dom 的过程

compile 为**遍历模板解析出模板里的指令**

link 将指令解析成为指令描述对象(descriptor)，闭包在了link函数里，link函数会把descriptor传入Directive构造函数，创建出真正的指令实例。

transclude 源码如下：
```

export function transclude (el, options) {
  if (options) {
    options._containerAttrs = extractAttrs(el) // 提取元素属性列表
  }
  if (isTemplate(el)) { // 判断是否为 <template></template> 格式
    el = parseTemplate(el) 
  }
  if (options) {
    // 如果当前看 component， 并且没有模板且只有一个壳
    // 那么只需要处理内容的嵌入
    if (options._asComponent && !options.template) {
      options.template = '<slot></slot>'
    }
    if (options.template) {
      options._content = extractContent(el) // 将 el 内容（子元素和文本节点）抽取出来
      el = transcludeTemplate(el, options)
    }
  }
  if (el instanceof DocumentFragment) {
    prepend(createAnchor('v-start', true), el)
    el.appendChild(createAnchor('v-end', true))
  }
  return el
}
```

然后到了 transcludeTemplate:

```
function transcludeTemplate (el, options) {
  var template = options.template
  var frag = parseTemplate(template, true)
  if (frag) {
    var replacer = frag.firstChild
    var tag = replacer.tagName && replacer.tagName.toLowerCase()
    if (options.replace) {
      /* istanbul ignore if */
      if (el === document.body) {
        process.env.NODE_ENV !== 'production' && warn(
          'You are mounting an instance with a template to ' +
          '<body>. This will replace <body> entirely. You ' +
          'should probably use `replace: false` here.'
        )
      }
      // there are many cases where the instance must
      // become a fragment instance: basically anything that
      // can create more than 1 root nodes.
      if (
        // multi-children template
        frag.childNodes.length > 1 ||
        // non-element template
        replacer.nodeType !== 1 ||
        // single nested component
        tag === 'component' ||
        resolveAsset(options, 'components', tag) ||
        replacer.hasAttribute('is') ||
        replacer.hasAttribute(':is') ||
        replacer.hasAttribute('v-bind:is') ||
        // element directive
        resolveAsset(options, 'elementDirectives', tag) ||
        // for block
        replacer.hasAttribute('v-for') ||
        // if block
        replacer.hasAttribute('v-if')
      ) {
        return frag
      } else {
        options._replacerAttrs = extractAttrs(replacer)
        mergeAttrs(el, replacer)
        return replacer
      }
    } else {
      el.appendChild(frag)
      return el
    }
  } else {
    process.env.NODE_ENV !== 'production' && warn(
      'Invalid template option: ' + template
    )
  }
}
```





