## 上一篇漏掉的
### filter
作为 1.0 版本频繁使用的 filter，在 构造函数里以 ```Vue.Options.filters``` 引入:
```
...
import filters from './filters/index'
...

Vue.version = '1.0.8'
...
Vue.options = {
  directives,
  elementDirectives,
  filters,
  transitions: {},
  components: {},
  partials: {},
  replace: true
}

export default Vue

...
```

filter 类实现了多种[过滤方法](https://v1.vuejs.org/api/#Filters),包括
* orderBy
* filterBy
* limitBy
* json
* capitalize
* uppercase
* lowercase
* currency
* pluralize
* debounce

#### JSON
```
  json: {
    read: function (value, indent) {
      return typeof value === 'string'
        ? value
        : JSON.stringify(value, null, Number(indent) || 2)
    },
    write: function (value) {
      try {
        return JSON.parse(value)
      } catch (e) {
        return value
      }
    }
  },
```
json 方法的实现其实是对 js 的 JSON 对象的方法做了一层封装， 有意思的是发现 ``` JSON.stringify(value, null, Number(indent) || 2)```, 发现 stringify 方法的第二参数和第三个参数，查了[mdn](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify) 才知道后两个参数分别代表要提取的属性以及缩进格式，在这里是提取所有属性并以两个空格作为缩进，而且有趣的是这个方法会忽略值为 undefined 或者 function 的属性，如果值为数组且数组里面有 undefined 或者 function，则转换为 null，更多特性请参考**MDN**文档

#### currency
```
  const digitsRE = /(\d{3})(?=\d)/g
  /**
   * 12345 => $12,345.00
   *
   * @param {String} sign
   */

  currency (value, currency) {
    value = parseFloat(value)
    if (!isFinite(value) || (!value && value !== 0)) return ''
    currency = currency != null ? currency : '$'
    var stringified = Math.abs(value).toFixed(2)
    var _int = stringified.slice(0, -3)
    var i = _int.length % 3
    var head = i > 0
      ? (_int.slice(0, i) + (_int.length > 3 ? ',' : ''))
      : ''
    var _float = stringified.slice(-3)
    var sign = value < 0 ? '-' : ''
    return currency + sign + head +
      _int.slice(i).replace(digitsRE, '$1,') +
      _float
  }
```

**currency** 的实现亮点在于 **digitsRE** 这个正则表达式，它使用了 **?=**（正向预查）来匹配连续三个数字且后续一个字符还是数字的情况

```
export function filterBy (arr, search, delimiter) {
  arr = convertArray(arr)
  if (search == null) {
    return arr
  }
  if (typeof search === 'function') {
    return arr.filter(search)
  }
  // cast to lowercase string
  search = ('' + search).toLowerCase()
  // allow optional `in` delimiter
  // because why not
  var n = delimiter === 'in' ? 3 : 2
  // extract and flatten keys
  var keys = toArray(arguments, n).reduce(function (prev, cur) {
    return prev.concat(cur)
  }, [])
  var res = []
  var item, key, val, j
  for (var i = 0, l = arr.length; i < l; i++) {
    item = arr[i]
    val = (item && item.$value) || item
    j = keys.length
    if (j) {
      while (j--) {
        key = keys[j]
        if ((key === '$key' && contains(item.$key, search)) ||
            contains(getPath(val, key), search)) {
          res.push(item)
          break
        }
      }
    } else if (contains(item, search)) {
      res.push(item)
    }
  }
  return res
}
```
**filterBy** 功能的实现值得一看

## 工具类 util
```
export * from './lang'
export * from './env'
export * from './dom'
export * from './options'
export * from './component'
export * from './debug'
export { defineReactive } from '../observer/index'
```
**util** 实现了很多工具类的方法供 vue 使用，属于底层且非业务逻辑的代码实现，因此应该是比较好理解的一个模块

### util/lang.js

#### set 方法
lang 实现的第一个方法是 ```set(obj, key, val)```, 不过和理解的不同，工具类方法的实现是可能包含业务逻辑的，set 会为 object 的某个属性设置新的值，并且如果属性不存在，会触发通知

```
export function set (obj, key, val) {
  if (hasOwn(obj, key)) { // 如果存在这个属性，直接更新该值并返回
    obj[key] = val
    return
  }
  if (obj._isVue) { // 如果是 vue 对象，则更新 obj 为 obj._data
    set(obj._data, key, val)
    return
  }
  var ob = obj.__ob__  // 不存在该属性时，获取 obj 的 __ob__ 属性值
  if (!ob) { // 不存在时直接更新返回
    obj[key] = val
    return
  }
  ob.convert(key, val) // 存在的话调用 convert 函数
  ob.dep.notify() // 这里应该就是触发更新通知吧
  if (ob.vms) { // vms 暂时不知道是什么
    var i = ob.vms.length
    while (i--) {
      var vm = ob.vms[i]
      vm._proxy(key)
      vm._digest()
    }
  }
}

var hasOwnProperty = Object.prototype.hasOwnProperty
/**
 * Check whether the object has the property.
 *
 * @param {Object} obj
 * @param {String} key
 * @return {Boolean}
 */
export function hasOwn (obj, key) {
  return hasOwnProperty.call(obj, key)
}

```
hasOwnProperty 用来检测对象是否含有某个属性，不包括继承的，为了避免 hasOwnProperty 被重写，最好使用 ```Object.prototype.hasOwnProperty.call(obj, key)``` 的方式来调用


** del(obj,key)** 用来删除相关的属性也是类似的操作

#### isLiteral
```
/**
 * Check if an expression is a literal value.
 *
 * @param {String} exp
 * @return {Boolean}
 */

var literalValueRE = /^\s?(true|false|[\d\.]+|'[^']*'|"[^"]*")\s?$/
export function isLiteral (exp) {
  return literalValueRE.test(exp)
}
```
暂时不知道是用在哪里，从正则表达式看来是匹配一段文本字符串

#### _toString

_toString 函数确保值为 null 或为 undefinded 输出为空字符串， 这里利用的模糊等号 '==' 来匹配 null 和 undefined：

```
export function _toString (value) {
  return value == null
    ? ''
    : value.toString()
}

```

#### camelize(驼峰) 和 hyphenate(连字符)

```
var camelizeRE = /-(\w)/g
export function camelize (str) {
  return str.replace(camelizeRE, toUpper)
}

function toUpper (_, c) {
  console.log(arguments);
  return c ? c.toUpperCase() : ''
}

var hyphenateRE = /([a-z\d])([A-Z])/g
export function hyphenate (str) {
  return str
    .replace(hyphenateRE, '$1-$2')
    .toLowerCase()
}
/**
 * Converts hyphen/underscore/slash delimitered names into
 * camelized classNames.
 *
 * e.g. my-component => MyComponent
 *      some_else    => SomeElse
 *      some/comp    => SomeComp
 *
 * @param {String} str
 * @return {String}
 */

var classifyRE = /(?:^|[-_\/])(\w)/g // 使用了非获取匹配
export function classify (str) {
  return str.replace(classifyRE, toUpper)
}


```

