### 关于 (0, module.fn)() 和 Object(module.fn) 的小知识



对于下面的例子

```javascript
// main.js
import { A } from './a.js'
A()
```

早期webpack 编译之后会变成

```javascript
var _a_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(1);
(0, _a_js__WEBPACK_IMPORTED_MODULE_0__/* .test */ .A)()
}();
```

在某个版本之后升级为下面的样子：

```javascript
var _a_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(1);
Object(_a_js__WEBPACK_IMPORTED_MODULE_0__/* .test */ .A)()
}();
```



这里对于 A 的调用需要使用 `(0, module.fn)`或者 `Object(module.fn)`包裹了一层，其实是为了修正上下文。我们知道 javascript 中，下面的代码

```javascript
var module = {
  a: function(){}
}
module.a()
var b = module.a
b()
```

我们直接调用 `module.a()` 和将其赋值给 b 之后再调用 `b()` ，在函数执行的时候，所在的上下文是不同的。对于 `module.a()`的方式，会多一层 module 的 context。

webpack 处理我们代码中的 import/export 时候，其实类似上面例子中，会把 export 出来的内容都 module 这个对象中。所以我们源码中 `A()` ，如果不加处理的话，会变成

```javascript
module.A()
```

这就和实际中，我们源码的 A 运行时候的上下文不一致了。

要修正这个方法，最简单的就是先让代码对 module.A 进行一下取值操作，得到返回值后再调用。这也就是 `(0, module.fn)`的作用，其执行的过程类似于

```javascript
var some = (0, module.fn)
some()
```

但是这种写法有一个弊端，那就是“分号”的问题。我们现在写 js 代码的时候，都是不写分号的。但是用上面这种方式，如果这句话前面还有一个函数，例如

```javascript
function b(){}
(0, module.fn)()
```

这时，`(0, module.fn)`就会被错误的解析为前面函数 b 的参数，从而造成错误。

用 `Object(module.fn)`这样的方式就是安全的，同时 `Object(module.fn)`的返回值也就是 fn



参考：[官方的issuse](https://github.com/webpack/webpack/issues/9615)

Issuse 中有个名词 ASI: Automatic Semicolon Insertion

(end)