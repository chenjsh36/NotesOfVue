### src/parsers

**parsers** 实现了解析器的功能，使用状态机、正则表达式等来对表达式、html、文本、指令等进行解析

### src/parsers/directive.js

在这里引用了 **cache**，**cache** 实现了一个双向链表，它会保存最近使用过的对象，在数量超出限制的情况下会抛弃最近没有使用的对象，思想基于 Least Recently Used， 参考自：[https://github.com/rsms/js-lru](https://github.com/rsms/js-lru)

directive 用于解析指令：
```
import { toNumber, stripQuotes } from '../util/index'
import Cache from '../cache'

const cache = new Cache(1000)
const filterTokenRE = /[^\s'"]+|'[^']*'|"[^"]*"/g
const reservedArgRE = /^in$|^-?\d+/

/**
 * Parser state
 */

var str, dir
var c, i, l, lastFilterIndex
var inSingle, inDouble, curly, square, paren

...

/**
 * Parse a directive value and extract the expression
 * and its filters into a descriptor.
 *
 * Example:
 *
 * "a + 1 | uppercase" will yield:
 * {
 *   expression: 'a + 1',
 *   filters: [
 *     { name: 'uppercase', args: null }
 *   ]
 * }
 *
 * @param {String} str
 * @return {Object}
 */

export function parseDirective (s) {

  var hit = cache.get(s)
  if (hit) {
    return hit
  }

  // reset parser state
  str = s
  inSingle = inDouble = false
  curly = square = paren = 0
  lastFilterIndex = 0
  dir = {}

  for (i = 0, l = str.length; i < l; i++) {
    c = str.charCodeAt(i)
    if (inSingle) {
      // check single quote
      if (c === 0x27) inSingle = !inSingle
    } else if (inDouble) {
      // check double quote
      if (c === 0x22) inDouble = !inDouble
    } else if (
      c === 0x7C && // pipe
      str.charCodeAt(i + 1) !== 0x7C &&
      str.charCodeAt(i - 1) !== 0x7C
    ) {
      if (dir.expression == null) {
        // first filter, end of expression
        lastFilterIndex = i + 1
        dir.expression = str.slice(0, i).trim()
      } else {
        // already has filter
        pushFilter()
      }
    } else {
      switch (c) {
        case 0x22: inDouble = true; break // "
        case 0x27: inSingle = true; break // '
        case 0x28: paren++; break         // (
        case 0x29: paren--; break         // )
        case 0x5B: square++; break        // [
        case 0x5D: square--; break        // ]
        case 0x7B: curly++; break         // {
        case 0x7D: curly--; break         // }
      }
    }
  }

  if (dir.expression == null) {
    dir.expression = str.slice(0, i).trim()
  } else if (lastFilterIndex !== 0) {
    pushFilter()
  }

  cache.put(s, dir)
  return dir
}

```
由于指令可能同时包含表达式和过滤器，如果两者同时存在会以 **|** 符号分隔开
循环遍历字符找出对应的表达式和过滤器


