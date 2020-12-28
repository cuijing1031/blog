## ProvidePlugin 的演变



在 webpack4 版本中，ProvidePlugin 的实现，使用了一种看起来比较 hack 的方式。下面 demo

```
// main.js
TEST()
```

```javascript
// webpack 配置
plugins: [
    new webpack.ProvidePlugin({
      TEST: path.resolve(path.join(__dirname, './a.js'))
    })
  ],
```

代码经过编译后结果如下

```javascript
/* 0 */
/***/ (function(module, exports, __webpack_require__) {

/* WEBPACK VAR INJECTION */(function(TEST) {TEST()
/* WEBPACK VAR INJECTION */}.call(this, __webpack_require__(1)))

/***/ }),
/* 1 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
__webpack_require__.r(__webpack_exports__);
const mainFunction = function () {
  console.log('mainFunction')
}

mainFunction.use = function(name) {
  console.log('load something')
}
/* harmony default export */ __webpack_exports__["default"] = (mainFunction);

/***/ })
```

整个 main.js 中的内容，被一个IIFE的方式，TEST 作为一个参数传入。另外，webpack4 中处理 global 和 system 的时候，也使用了这种通过注入函数参数的方式。

ProvidePlugin源码主要逻辑

```javascript
ParserHelpers.addParsedVariableToModule(
  parser,
  nameIdentifier,
  expression // require('xxx')
)
```

addParsedVariableToModule 里面，会执行

```javascript
	if (!parser.state.current.addVariable) return false;
	var deps = [];
  // 解析一下，可以处理我们写 a.b.c 的情况， 和 require 的东西
	parser.parse(expression, {
		current: {
      // 在 parse 的过程中，parse.current 就是对应的 module
      // 而 module 中有一个 addDependency 的方法，用来添加依赖
      // 这里写了一个 addDependency 的方法，将 expression 的依赖收集起来，然后在后面统一添加到 parser.state.current 中
			addDependency: dep => {
				dep.userRequest = name;
				deps.push(dep);
			}
		},
		module: parser.state.module
	});
  // 把 variable 添加到 module 中
	parser.state.current.addVariable(name, expression, deps);
	return true;
```

```javascript
// 给每一个 variables 生成一个 DependenciesBlockVariable
addVariable(name, expression, dependencies) {
	for (let v of this.variables) {
		if (v.name === name && v.expression === expression) {
			return;
		}
	}
	this.variables.push(
		new DependenciesBlockVariable(name, expression, dependencies)
	);
}
```
这些执行完毕后，在 module 的 variables 中就多了一个 DependenciesBlockVariable，并且

```
name: 'TEST',
expression: "require("/Users/didi/Documents/learn/webpack-4-demo/banner-demo/a.js")"
dependencies: 
[ 
  CommonJsRequireDependency, // 解析 require 而来
  RequireHeaderDependency
]
```



**Parser 过程结束后，webpack 在后面会对依赖进行处理，其中有一块是专门处理 variables 中的内容**

```javascript
// Compilation.js
	processModuleDependencies(module, callback) {
		const dependencies = new Map();

		const addDependency = dep => {
			//...
		};

		const addDependenciesBlock = block => {
			if (block.dependencies) {
				iterationOfArrayCallback(block.dependencies, addDependency);
			}
			if (block.blocks) {
				iterationOfArrayCallback(block.blocks, addDependenciesBlock);
			}
      // 有一部分专门用来处理 variables 中的依赖
			if (block.variables) {
				iterationBlockVariable(block.variables, addDependency);
			}
		};
```

这样我们在 Parser 阶段添加到 module.variables 中的 dependency 就会被 webpack 处理。



**代码生成阶段**，在生成 main.js 这个 module 的代码的时候，

```javascript
sourceBlock(/.../){
  // 处理 dependency
  for (const dependency of block.dependencies) {
			this.sourceDependency(
				dependency,
				dependencyTemplates,
				source,
				runtimeTemplate
			);
  }
  // 处理 variables
  const vars = block.variables.reduce((result, value) => {
      // 在这个函数里面会走到每一个 variable 的 variable.expressionSource 方法，返回最终要添加的变量的名称(看下方分析)
			const variable = this.sourceVariables(
				value,
				availableVars,
				dependencyTemplates,
				runtimeTemplate
			);

			if (variable) {
				result.push(variable);
			}

			return result;
	}, []);
}
```

 `expressionSource`方法如下：

```javascript
// DependenciesBlockVariable.js
expressionSource(dependencyTemplates, runtimeTemplate) {
		const source = new ReplaceSource(new RawSource(this.expression));
    // 遍历 TEST 生成的 DependenciesBlockVariable 中的 dependency,即：CommonJsRequireDependency 和 RequireHeaderDependency
    // CommonJsRequireDependency 会将 'require(xx)' 中的 xxx 替换成 'moduleId', 变成 'require(moduleId)'
    // RequireHeaderDependency 会将 'require(xx)' 中的 require 替换成 '__webpack_require__'
		for (const dep of this.dependencies) {
			const template = dependencyTemplates.get(dep.constructor);
			if (!template) {
				throw new Error(`No template for dependency: ${dep.constructor.name}`);
			}
			template.apply(dep, source, runtimeTemplate, dependencyTemplates);
		}
		return source;
	}
```

经过各个 dependency 的处理过后，原本的 `requrie(path)` 将会转化为 `__webpack_require__(1)` 其中1 为对应的 moduleId。然后返回到上面 sourceBlock 中，这时就将这个结果 push 到了我们的 vars 中。vars 中元素结构如下

```
{
  name: 'TEST',
  expression: 上面生成的 ReplaceSource
}
```

在接下来 sourceBlock 的处理中，会包裹成IIFE的方式

```
/* WEBPACK VAR INJECTION */(function(TEST) {

  中间内容

/* WEBPACK VAR INJECTION */}.call(this, __webpack_require__(1)))
```

这就是webpack4的方式。这种方法生成的代码实际上多了一层 block，运行会有一定的开销，而且阅读上来说也很奇怪。在 webpack5 里面使用 addParsedVariableToModule 的方式就被废弃了，直接使用了 dependency 。



### webpack5 中的 ProvidePlugin

先看 webpack5 中最后生成的代码

```
/* provided dependency */ var TEST = __webpack_require__(1);
TEST()
}();
```

采用了和我们直接写 require 一样的方式来处理，看起来清晰很多。

插件中实现如下

```javascript
parser.hooks.expression.for(name).tap("ProvidePlugin", expr => {
							const nameIdentifier = name.includes(".")
								? `__webpack_provided_${name.replace(/\./g, "_dot_")}`
								: name;
							const dep = new ProvidedDependency(
								request[0],
								nameIdentifier,
								request.slice(1),
								expr.range
							);
							dep.loc = expr.loc;
							parser.state.module.addDependency(dep);
							return true;
						});
```

和4中加入一个 variable 相比，这里直接生成了一个 ProvidedDependency（继承自 ModuleDependency），并加入 module 中。在后面 processDependency 阶段，也之后对 block.dependencies 的处理

```
processDependency() {
   //...
   const processDependenciesBlock = block => {
			if (block.dependencies) {
				currentBlock = block;
				for (const dep of block.dependencies) processDependency(dep);
			}
			if (block.blocks) {
				for (const b of block.blocks) processDependenciesBlock(b);
			}
		};
  //...
}
```

这里会处理加入的 ProvidedDependency，找到对应的 js 文件，然后处理，生成 module。

继续到代码生成阶段，对于 ProvidedDependency 会进入到 ProvidedDependencyTemplate ，它继承自 ModuleDependencyTemplate

```javascript
  apply(
		dependency,
		source,
		{
			runtimeTemplate,
			moduleGraph,
			chunkGraph,
			initFragments,
			runtimeRequirements
		}
	) {
		const dep = /** @type {ProvidedDependency} */ (dependency);
		initFragments.push(
			new InitFragment(
				`/* provided dependency */ var ${
					dep.identifier // 我们定义的 TEST
				} = ${runtimeTemplate.moduleExports({
					module: moduleGraph.getModule(dep),
					chunkGraph,
					request: dep.request,
					runtimeRequirements
				})}${pathToString(dep.path)};\n`,
				InitFragment.STAGE_PROVIDES,
				1,
				`provided ${dep.identifier}`
			)
		);
		source.replace(dep.range[0], dep.range[1] - 1, dep.identifier);
	}
```

上面代码生成了一个

```
/* provided dependency */ var ${a} = ${b}${c}
```

a 部分是我们定义的 TEST； b 部分是由 runtimeTemplate.moduleExports 生成的引入TEST 对应 module 的函数，这里是`__webpack_require_(1)`,；c 部分这里为空。所以合起来就是

```
/* provided dependency */ var TEST = __webpack_require_(1)
```



(End)

