### 基于typescript 的 IoC(Inversion of Control)

Ioc 实现底层依赖是 typescript 中的 装饰器 decorator 和 反射 reflact

#### decorator

ts 中的 decorator 介绍见[官网](https://www.typescriptlang.org/docs/handbook/decorators.html)

##### method decorator

下面简单的 demo，是针对 class 中方法的装饰器工厂

```typescript
function f() {
  console.log("f(): evaluated");
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value
    descriptor.value = function (...arg: any[]) {
      originalMethod.call(this, ...arg)
      console.log('wrap')
    }

    console.log("f(): called");
  };
}

class C {
  @f()
  method(a: string) {
    console.log('method call:', a)
  }
}

const ins = new C()
ins.method('123')
```

上面 @f 在 class C 定义的时候就会被调用。上面 target 是 class C，propertyKey 是 "method"。可以借助装饰器来实现在 method 外，再包裹一层逻辑。

**需要注意上面重新定义 descriptor.value 时候，不要用箭头函数，否则函数内部 this 的指向会有问题**



##### class decorator

class decorator 中可以得到 class 的 constructor。

##### property decorator

属性装饰器中并没有 property descriptor。也就是说，我们不能像 method decorator 一样，通过拿到 descriptor 来对原本的逻辑进行修改和再次封装，只能获取到当前 class 中定义了的属性名称。官网 demo 中结合 reflect-metadata 可以实现 @format 的行为。关于 reflect-metadata 下面介绍。

```typescript
class Greeter {
  @format("Hello, %s")
  greeting: string;

  constructor(message: string) {
    this.greeting = message;
  }
  greet() {
    let formatString = getFormat(this, "greeting");
    return formatString.replace("%s", this.greeting);
  }
}
```

```typescript
import "reflect-metadata";

const formatMetadataKey = Symbol("format");

function format(formatString: string) {
  return Reflect.metadata(formatMetadataKey, formatString);
}

function getFormat(target: any, propertyKey: string) {
  return Reflect.getMetadata(formatMetadataKey, target, propertyKey);
}
```



##### Accessor Decorators

可以在 class 中的 set 和 get 前定义 decorators。accessor decorators 中可以获取到 property descriptor。也正是因为如此，ts 中禁止在同一个属性的 set 和 get 中同时使用 decorators，并且需要把 decorators 写在第一个 accessor 上。

在method 和 access descorator 中可以返回一个 PropertyDescriptor 的值，这个值会直接被作用于这个属性。例如下面的 demo

```typescript
function Overwrite(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  descriptor.enumerable = true
  return {
    set: function(v: number) {
      descriptor.value = 2
    }
  }
}

class Point {
  private _x: number;
  private _y: number;
  constructor(x: number, y: number) {
    this._x = x;
    this._y = y;
  }

  @Overwrite
  get y() {
    return this._y;
  }

  set y(v: number) {
    this._y = v
  }
}

const ins = new Point(1, 2)
ins.y = 4
console.log(ins.y) // 2
```

由于 Overwrite 中返回了一个新的 PropertyDescriptor，里面有一个 set，所以之前 y 的 set 被覆盖。



##### Parameter Decorators

这个参数可以是 constructor 的参数，也可以是方法的参数。parameter decorators 的返回值会被忽略。像官方 demo 中一样，可以结合 reflect-metadata 发挥很多作用。



#### Reflect-metadata

关于 Reflect 的介绍可以参考[MDN 中介绍](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect)。

> 反射的概念是由Smith在1982年首次提出的，主要是指程序可以访问、检测和修改它本身状态或行为的一种能力。

在上面 decorator 中多次提到 reflect-metada ，则可以用来定义元数据。可以认为不管是调用 `Reflect.definedProperty` 还是 `Reflect.has` 我们都是操作的原本那个 target 的属性。但是有些时候，例如上面 property decorator 的 demo 中，我们想在 `greeting` 的 decorator `@format("Hello, %s")` 执行时，把 format 内部的参数保存下来。后面我们可以通过 Greeter, 'greeting' 就能再次获取到这个信息。要保存这个信息，简单粗暴的方式存在一个全局的 map 中，但是注意这里是在 Greeter 的定义中。如果我们后面要在实例中或者继承的类中也能获取到，我们保存和获取时需要考虑原型链。可能需要自己写一大堆的逻辑来实现。而 Reflect-metadata 则为我们提供了一个方便的方法，如下面的 demo 

```typescript
class MyClassA {
  // someKey 是元数据的 key，'someStringA' 则是要保存的数据
  @Reflect.metadata('someKey', 'someStringA')
  methodA() {}
}
class MyClassB {
  @Reflect.metadata('someKey', 'someStringB')
  methodA() {}
}

Reflect.getMetadata(MyClassA, 'methodA', 'someKey') // 输出结果 'someStringA'
Reflect.getMetadata(MyClassB, 'methodA', 'someKey') // 输出结果 'someStringB'
```

关于 Reflect-metadata 的更多的使用和 API，参考[官方介绍](https://www.npmjs.com/package/reflect-metadata)。Reflect-metadata 中会内置了三个 metadata: 

-  `design:type` 获取类型
- `design:paramtypes`  获取参数类型
- `desing:returntypes` 获取返回值类型



#### IoC

了解了 decorator 和 reflect 之后，我们来看如何利用他们实现 IoC。以 DI(依赖注入，IocC 常见的应用方式) 为例。假设下面的两个模块 Logger 和 WorkStream

```typescript
class Logger {
  constructor() {
  
  }
  printLog(log: string) {
  	console.log(log)
  }
}

class WorkStream {
  run() {
    // 输出 log
  }
}
```

如果需要在 WorkStream run 方法调用时输出 log，没有 DI 的情况下，我们会用下面的写法

```typescript
class Logger {
  printLog(log: string) {
  	console.log(log)
  }
}

class WorkStream {
  private logger: Logger
  constructor() {
    this.logger = new Logger()
  }
  run() {
    this.logger.printLog('run')
  }
}
```



使用 DI 的话，可以用类似下面这样的实现：

```typescript
function inject(
  target: any,
  propertyKey: string
) {
  const ctor = Reflect.getMetadata('design:type', target, propertyKey)
  const ins = new ctor()
  console.log('new Logger')
  Reflect.defineProperty(target, propertyKey, {
    value: ins,
    writable: false,
    configurable: false,
    enumerable: true
  })
}

class Logger {
  printLog(log: string) {
  	console.log(log)
  }
}

class WorkStream {
  @inject
  private logger: Logger
  constructor() {
    console.log('new WorkStream')
  }
  run() {
    this.logger.printLog('run')
  }
}

const test = new WorkStream()
test.run()
```

在 `@inject`  中可以通过反射来获得类型 Logger，生成实例之后挂载到 WorkStream 的原型链上。这样虽然可以实现，但是这样一来，logger 其实就不是私有的，而且所有 WorkStream 的实例都共享了同一个 Logger 的实例。例如下面这样就会有问题

```
const test2 = new WorkStream()
const test = new WorkStream()
test.logger.printLog = function () {
  console.log('xxxxx')
}
test.run()
test2.run()
```

要解决这个问题的话，我们就不能在 WorkStream 的 constructor 中进行 `defineProperty`，而需要在其 new 出来的实例上操作。所以对于 InversifyJS ，还是 tsyringe ，是通过类似 container.resolve 的方式来获取实例(当然这么设计还有其他的原因, resolve 中有更大的自由权，可以做更多的事情)。

上面只是一个简单的实现，假如我们希望 Logger 是一个单例的，或者实现一些复杂情况的话，就需要额外引入一个 container 来存放和管理。下面的 demo

```typescript
import "reflect-metadata"

const container = new Map()
function injectable(name: Symbol): ClassDecorator {
  return function(target: any) { 
    if (container.has(name)) {
      console.error(`${name} has been set`)
    }
    container.set(name, {
      ctor: target,
      ins: null
    })
    return target
  }
}

function inject(name: Symbol) {
  return function (
    target: any,
    propertyKey: string
  ) {
    const res = container.get(name)
    if (!res.ins) {
      res.ins = new res.ctor()
    }
    Reflect.defineProperty(target, propertyKey, {
      value: res.ins,
      writable: false,
      configurable: false,
      enumerable: true
    })
  }
}

const LoggerInjector = Symbol('Logger')

@injectable(LoggerInjector)
class Logger {
  printLog(log: string) {
  	console.log(log)
  }
}

class WorkStream {
  @inject(LoggerInjector)
  private logger: Logger
  constructor() {
    console.log('new WorkStream')
  }
  run() {
    this.logger.printLog('run')
  }
}

const test = new WorkStream()
test.run()
```

上面 Container 只是一个简单的 Map 对象。可以再进一步，将 Container 设计为一个 class ，这样就可以对外提供一下方法，例如 通过`Container.resolve('Logger')` 直接获取 Logger 实例等。



#### IoC 相关的库

Github 上也可以找到开源的 LoC 的库，例如：[tsyringe](https://github.com/microsoft/tsyringe), [InversifyJS](https://github.com/inversify/InversifyJS)。

inversifyJS 提供的关于 IoC 各种的 decorator 和方法。而 tsyringe 相比来说，是一个略为轻量的库。下面主要看一下 tsyringe 库(InversifyJS库，逻辑有点绕，先放放)，前面实现简单的 DI 的时候说到过 demo 中的不足： Logger 变成了所有实例共享的。但是用 tsyringe 来跑一下

```typescript
import "reflect-metadata"
import { injectable, inject, container } from './tsyringe/index'

@injectable()
class Log {
  print() {
    console.log('log print')
  }
}
container.register<Log>('Log', Log)

@injectable()
class Foo {
  constructor(@inject('Log') public log?: Log) {
    console.log('xxx')
  }
  run() {
    this.log.print()
  }
}

const a = container.resolve(Foo)
const b = container.resolve(Foo)
a.log.print = function () {
  console.log('yyyy')
}
a.run() // yyyy
b.run() // log print
```

这样才是正确的结果。具体怎么做呢？首先在 `@inject` 和 `@injectable` 这两个 decorator 中，只是做了信息的收集。然后在container. resolve 的时候拿到之前收集的信息，生成 Log 实例，并作为 `new Foo` 的参数传入。参照 tsyringe 的源码整理出了一个简单的实现

```typescript
type constructor<T> = {
  new (...args: any[]): T;
}

const INJECTION_TOKEN_METADATA_KEY = 'injectionTokens'
// 保存 injectable 的类的参数类型
const typeInfo = new Map()

class Container {
  private registerMap = new Map()
  
  // 在 container.resolve(WorkStream) 执行的时候，typeInfo 中对应的信息为：params[0] = 'Log'
  // 然后从 registerMap 中根据 Log 得到对应的 constructor，就可以执行  new Log()，从而有 params[0] = new Log()
  // 最后将 params 作为参数传给 WorkStream
  resolve<T>(ctor: constructor<T>): T {
    const paramsInfo = typeInfo.get(ctor)
    const params = paramsInfo.map((p: string) => {
      if (this.registerMap.has(p)) {
        const paramCtor = this.registerMap.get(p)
        return new paramCtor()
      } else {
        console.error(`${p}没有注册`)
      }
    })
    return new ctor(...params)
  }
  register<T>(token: string, ctor: constructor<T>) {
    this.registerMap.set(token, ctor)
  }
}
const container = new Container()

function inject(
  token: string
): (target: any, propertyKey: string, parameterIndex: number) => any {
  return function(
    target: any,
    _propertyKey: string,
    parameterIndex: number
  ): any {
    const descriptors = {} as {[key: string]: any}
    descriptors[parameterIndex] = token
    // 把标识 inject 的参数信息存入 metadata 中
    Reflect.defineMetadata(INJECTION_TOKEN_METADATA_KEY, descriptors, target)
  }
}

function injectable<T>(): (target: constructor<T>) => void  {
  // 获取 WorkStream 的 constructor 的参数信息，这里主要需要拿到参数的个数
  // 由于 inject 的时候会保存信息：第0个参数是 'Log'，所以会设置 params[0] = 'Log'
  // 然后保存到 typeInfo 中，在后期 resolve 的时候使用
  return function (target: constructor<T>) {
    const params: any[] = Reflect.getMetadata("design:paramtypes", target) || []
    const injectionTokens = Reflect.getOwnMetadata(INJECTION_TOKEN_METADATA_KEY, target) || {}
    Object.keys(injectionTokens).forEach(key => {
      params[+key] = injectionTokens[key]
    })
    typeInfo.set(target, params)
  }
}

@injectable()
class Logger {
  printLog(log: string) {
  	console.log(log)
  }
}
container.register<Logger>('Log', Logger)

@injectable()
class WorkStream {
  constructor(@inject('Log') private logger: Logger) {
    console.log('new WorkStream')
  }
  run() {
    this.logger.printLog('run')
  }
}

const test = container.resolve(WorkStream)
test.run()
```

这个简单的例子中，注入依赖放在 constructor 的参数中。这也是 tsyringe 中提供的依赖注入方法。而 inversifyJS 中除了 constructor 的参数提供注入，还有属性声明中的注入，例如下面的用法

```typescript
@injectable()
class Book {

  private _author: Author;
  private _summary: Summary;

  @inject("PrintService")
  private _printService: PrintService;

  public constructor(
      @inject("Author") author: Author,
      @inject("Summary") summary: Summary
) {
    this._author = author;
    this._summary = summary;
  }

  public print() {
     this._printService.print(this);
  }
}
```

但是官方文档中有这么一句

>InversifyJS supports property injection because sometimes constructor injection is not the best kind of injection pattern. However, you should try to avoid using property injection and prefer constructor injection in most cases.
>
>> If the class cannot do its job without the dependency, then add it to the constructor. The class needs the new dependency, so you want your change to break things. Also, creating a class that is not fully initialized ("two-step construction") is an anti-pattern (IMHO). If the class can work without the dependency, a setter is fine.
>
>Source: [http://stackoverflow.com/](http://stackoverflow.com/questions/1503584/dependency-injection-through-constructors-or-property-setters)

如果这个 dependency 是我们的 class 运行所必需的，缺少后功能无法正常执行，那么需要写在 constructor 中的参数中。通过在 new 的时候将 inject 的实例传入，然后得到结果，和上面 demo 中类似。但通过属性注入的，那么执行的过程类似下面：

```typescript
// 1.先生成 propery injections 的实例
const injectInstance = printServiceInstance
// 2.执行 new
const result = new ctor() 
// 3.设置属性
result['_printService'] = injectInstance
```

new 完之后还需要进行属性赋值操作，所以官方说  two-step construction 比如反设计模式。除此外，另外一个原因，就是作为必需的 dependency 的话，如果我们进行了修改，那么按照道理来说，其实是需要有一个 breaking change 的，写成 property inject 的话，这个 breaking change 就没有了。
