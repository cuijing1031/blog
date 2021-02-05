## vue 中的 defineComponent

defineComponent 本身的功能很简单，但是最主要的功能是为了 ts 下的类型推到。对于一个 ts 文件，如果我们直接写

```
export default {}
```

这个时候，对于编辑器而言，{} 只是一个 Object 的类型，无法有针对性的提示我们对于 vue 组件来说 {} 里应该有哪些属性。但是增加一层 defineComponet 的话，

```
export default defineComponent({})
```

这时，{} 就变成了 defineComponent 的参数，那么对参数类型的提示，就可以实现对 {} 中属性的提示，外还可以进行对参数的一些类型推导等操作。

> 但是上面的例子，如果你在 vscode 的用 .vue 文件中尝试的话，会发现不写 defineComponent 也一样有提示。这个其实是 Vetur 插件进行了处理。



下面看 defineComponent 的实现，有4个重载，先看最简单的第一个，这里先不关心 DefineComponent 是什么，后面细看。

```typescript
// overload 1: direct setup function
// (uses user defined props interface)
export function defineComponent<Props, RawBindings = object>(
  setup: (
    props: Readonly<Props>,
    ctx: SetupContext
  ) => RawBindings | RenderFunction
): DefineComponent<Props, RawBindings>
```

defineComponet 参数为 function， function 有两个参数 props 和 ctx，返回值类型为 RawBingds 或者 RenderFunction。defineComponet 的返回值类型为 `DefineComponent<Props, RawBindings>`。这其中有两个泛型 Props 和 RawBingding。Props 会根据我们实际写的时候给 setup 第一个参数传入的类型而确定，RawBindings 则根据我们 setup 返回值来确定。一大段话比较绕，写一个类似的简单的例子来看:

- 类似 props 用法的简易 demo 如下，我们给 a 传入不同类型的参数，define 返回值的类型也不同。这种叫 [Generic Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html)

  ```typescript
  declare function define<Props>(a: Props): Props
  
  const arg1:string = '123'
  const result1 = define(arg1) // result1：string
  
  const arg2:number = 1
  const result2 = define(arg2) // result2: number
  ```

  

- 类似 RawBingdings 的简易 demo如下： setup 返回值类型不同，define 返回值的类型也不同

  ```typescript
  declare function define<T>(setup: ()=>T): T
  
  const arg1:string = '123'
  const resul1 = define(() => {
    return arg1
  })
  
  const arg2:number = 1
  const result2 = define(() => {
    return arg2
  })
  ```

由上面两个简易的 demo，可以理解重载1的意思，defineComponet 返回类型为`DefineComponent<Props, RawBindings>`，其中 Props 为 setup 第一个参数类型；RawBingings 为 setup 返回值类型，如果我们返回值为函数的时候，取默认值 object。从中可以掌握一个 ts 推导的基本用法，对于下面的定义

```typescript
declare function define<T>(a: T): T
```

**可以根据运行时传入的参数，来动态决定 T 的类型**  这种方式也是运行时类型和 typescript 静态类型的唯一联系，很多我们想通过运行时传入参数类型，来决定其他相关类型的时候，就可以使用这种方式。



接着看 definComponent，它的重载2，3，4分别是为了处理 options 中 props 的不同类型。看最常见的 object 类型的 props 的声明

```typescript
export function defineComponent<
  // the Readonly constraint allows TS to treat the type of { required: true }
  // as constant instead of boolean.
  PropsOptions extends Readonly<ComponentPropsOptions>,
  RawBindings,
  D,
  C extends ComputedOptions = {},
  M extends MethodOptions = {},
  Mixin extends ComponentOptionsMixin = ComponentOptionsMixin,
  Extends extends ComponentOptionsMixin = ComponentOptionsMixin,
  E extends EmitsOptions = Record<string, any>,
  EE extends string = string
>(
  options: ComponentOptionsWithObjectProps<
    PropsOptions,
    RawBindings,
    D,
    C,
    M,
    Mixin,
    Extends,
    E,
    EE
  >
): DefineComponent<PropsOptions, RawBindings, D, C, M, Mixin, Extends, E, EE>
```

和上面重载1差不多的思想，核心思想也是根据运行时写的 options 中的内容推导出各种泛型。在 vue3 中 setup 的第一个参数是 props，这个 props 的类型需要和我们在 options 传入的一致。这个就是在`ComponentOptionsWithObjectProps`中实现的。代码如下

```typescript
export type ComponentOptionsWithObjectProps<
  PropsOptions = ComponentObjectPropsOptions,
  RawBindings = {},
  D = {},
  C extends ComputedOptions = {},
  M extends MethodOptions = {},
  Mixin extends ComponentOptionsMixin = ComponentOptionsMixin,
  Extends extends ComponentOptionsMixin = ComponentOptionsMixin,
  E extends EmitsOptions = EmitsOptions,
  EE extends string = string,
  Props = Readonly<ExtractPropTypes<PropsOptions>>,
  Defaults = ExtractDefaultPropTypes<PropsOptions>
> = ComponentOptionsBase<
  Props,
  RawBindings,
  D,
  C,
  M,
  Mixin,
  Extends,
  E,
  EE,
  Defaults
> & {
  props: PropsOptions & ThisType<void>
} & ThisType<
    CreateComponentPublicInstance<
      Props,
      RawBindings,
      D,
      C,
      M,
      Mixin,
      Extends,
      E,
      Props,
      Defaults,
      false
    >
  >
    
export interface ComponentOptionsBase<
  Props,
  RawBindings,
  D,
  C extends ComputedOptions,
  M extends MethodOptions,
  Mixin extends ComponentOptionsMixin,
  Extends extends ComponentOptionsMixin,
  E extends EmitsOptions,
  EE extends string = string,
  Defaults = {}
>
  extends LegacyOptions<Props, D, C, M, Mixin, Extends>,
    ComponentInternalOptions,
    ComponentCustomOptions {
      setup?: (
        this: void,
        props: Props,
        ctx: SetupContext<E, Props>
      ) => Promise<RawBindings> | RawBindings | RenderFunction | void
    //...
  }
```



很长一段，同样的先用一个简化版的 demo 来理解一下：

```typescript
type TypeA<T1, T2, T3> = {
  a: T1,
  b: T2,
  c: T3
}
declare function define<T1, T2, T3>(options: TypeA<T1, T2, T3>): T1
const result = define({
  a: '1',
  b: 1,
  c: {}
}) // result: string
```

根据传入的 options 参数 ts 会推断出 T1,T2,T3的类型。得到 T1, T2, T3 之后，可以利用他们进行其他的推断。稍微改动一下上面的 demo，假设 c 是一个函数，里面的参数类型由 a 的类型来决定：

```typescript
type TypeA<T1, T2, T3> = TypeB<T1, T2>
type TypeB<T1, T2> = {
  a: T1
  b: T2,
  c: (arg:T1)=>{}
}
const result = define({
  a: '1',
  b: 1,
  c: (arg) => {  // arg 这里就被会推导为一个 string 的类型
    return arg
  }
})
```

然后来看 vue 中的代码，首先 `defineComponent` 可以推导出 PropsOptions。但是 props 如果是对象类型的话，写法如下

```
props: {
   name: {
     type: String,
     //... 其他的属性
   }
}
```

而 setup 中的 props 参数，则需要从中提取出 type 这个类型。所以在 ComponentOptionsWithObjectProps 中

```typescript
export type ComponentOptionsWithObjectProps<
  PropsOptions = ComponentObjectPropsOptions,
  //...
  Props = Readonly<ExtractPropTypes<PropsOptions>>,
  //...
>
```

通过 ExtracPropTypes 对 PropsOptions 进行转化，然后得到 Props，再传入 ComponentOptionsBase，在这个里面，作为 setup 参数的类型

```typescript
export interface ComponentOptionsBase<
  Props,
  //...
>
  extends LegacyOptions<Props, D, C, M, Mixin, Extends>,
    ComponentInternalOptions,
    ComponentCustomOptions {
  setup?: (
    this: void,
    props: Props,
    ctx: SetupContext<E, Props>
  ) => Promise<RawBindings> | RawBindings | RenderFunction | void
```

这样就实现了对 props 的推导。

- this 的作用

  在 setup 定义中第一个是 this:void 。我们在 setup 函数中写逻辑的时候，会发现如果使用了 `this.xxx` IDE 中会有错误提示

  > Property 'xxx' does not exist on type 'void'

  这里通过设置 `this:void`来避免我们在 setup 中使用 this。

  this 在 js 中是一个比较特殊的存在，它是根据运行上上下文决定的，所以 typescript 中有时候无法准确的推导出我们代码中使用的 this 是什么类型的，所以 this 就变成了 any，各种类型提示/推导啥的，也都无法使用了(注意：只有开启了 noImplicitThis 配置， ts 才会对 this 的类型进行推导)。为了解决这个问题，typescript 中 function 的可以明确的写一个 this 参数，例如官网的例子：

  ```typescript
  interface Card {
    suit: string;
    card: number;
  }
  
  interface Deck {
    suits: string[];
    cards: number[];
    createCardPicker(this: Deck): () => Card;
  }
  
  let deck: Deck = {
    suits: ["hearts", "spades", "clubs", "diamonds"],
    cards: Array(52),
    // NOTE: The function now explicitly specifies that its callee must be of type Deck
    createCardPicker: function (this: Deck) {
      return () => {
        let pickedCard = Math.floor(Math.random() * 52);
        let pickedSuit = Math.floor(pickedCard / 13);
  
        return { suit: this.suits[pickedSuit], card: pickedCard % 13 };
      };
    },
  };
  
  let cardPicker = deck.createCardPicker();
  let pickedCard = cardPicker();
  
  alert("card: " + pickedCard.card + " of " + pickedCard.suit);
  ```

  明确的定义出在 createCardPicker 中的 this 是 Deck 的类型。这样在 `createCardPicker` 中 this 下可使用的属性/方法，就被限定为 Deck 中的。

  另外和 this 有关的，还有一个 [`ThisType`](https://www.typescriptlang.org/docs/handbook/utility-types.html#thistypetype)。



### ExtractPropTypes 和 ExtractDefaultPropTypes

上面提到了，我们写的 props 

```
{
  props: {
    name1: {
      type: String,
      require: true
    },
    name2: {
      type: Number
    }
  }
}
```

经过 defineComponent 的推导之后，会被转换为 ts 的类型

```
ReadOnly<{
  name1: string,
  name2?: number | undefined
}>
```

这个过程就是利用 ExtractPropTypes 实现的。

```typescript
export type ExtractPropTypes<O> = O extends object
  ? { [K in RequiredKeys<O>]: InferPropType<O[K]> } &
      { [K in OptionalKeys<O>]?: InferPropType<O[K]> }
  : { [K in string]: any }
```

根据类型中清晰的命名，很好理解：利用 `RrequiredKeys<O>` 和 `OptionsKeys<O>` 将 O 按照是否有 required 进行拆分(以前面props为例子)

```
{
  name1
} & {
  name2？
}
```

然后每一组里，用 `InferPropType<O[K]>` 推导出类型。

- InferPropType

  在理解这个之前，先理解一些简单的推导。首先我们在代码中写

  ```
  props = {
    type: String
  }
  ```

  的话，经过 ts 的推导，props.type 的类型是 StringConstructor。所以第一步需要从 StringConstructor/ NumberConstructor 等 xxConstrucror 中得到对应的类型 string/number 等。可以通过 infer 来实现

  ```typescript
  type a = StringConstructor
  type ConstructorToType<T> = T extends  { (): infer V } ? V : never
  type c = ConstructorToType<a> // type c = String
  ```

  上面我们通过 `():infer V` 来获取到类型。之所以可以这样用，和 String/Number 等类型的实现有关。javascript 中可以写

  ```
  const key = String('a')
  ```

  此时，key 是一个 string 的类型。还可以看一下 StringConstructor 接口类型表示

  ```typescript
  interface StringConstructor {
      new(value?: any): String;
      (value?: any): string;
      readonly prototype: String;
      fromCharCode(...codes: number[]): string;
  }
  ```

  上面有一个 `():string` ，所以通过 `extends {(): infer V}` 推断出来的  V 就是 string。

  然后再进一步，将上面的 a 修改成 propsOptions 中的内容，然后把 ConstructorToType 中的 infer V 提到外面一层来判断

  ```typescript
  type a = StringConstructor
  type ConstructorType<T> = { (): T }
  type b = a extends {
    type: ConstructorType<infer V>
    required?: boolean
  } ? V : never // type b = String
  ```

  这样就简单实现了将 props 中的内容转化为 type 中的类型。

  因为 props 的 type 支持很多中写法，vue3 中实际的代码实现要比较复杂

  ```typescript
  type InferPropType<T> = T extends null
    ? any // null & true would fail to infer
    : T extends { type: null | true }
      ? any 
      // As TS issue https://github.com/Microsoft/TypeScript/issues/14829 // somehow `ObjectConstructor` when inferred from { (): T } becomes `any` // `BooleanConstructor` when inferred from PropConstructor(with PropMethod) becomes `Boolean`
      // 这里单独判断了 ObjectConstructor 和 BooleanConstructor
      : T extends ObjectConstructor | { type: ObjectConstructor }
        ? Record<string, any>
        : T extends BooleanConstructor | { type: BooleanConstructor }
          ? boolean
          : T extends Prop<infer V, infer D> ? (unknown extends V ? D : V) : T
  
  // 支持 PropOptions 和 PropType 两种形式
  type Prop<T, D = T> = PropOptions<T, D> | PropType<T>
  interface PropOptions<T = any, D = T> {
    type?: PropType<T> | true | null
    required?: boolean
    default?: D | DefaultFactory<D> | null | undefined | object
    validator?(value: unknown): boolean
  }
  
  export type PropType<T> = PropConstructor<T> | PropConstructor<T>[]
  
  type PropConstructor<T = any> =
    | { new (...args: any[]): T & object } // 可以匹配到其他的 Constructor
    | { (): T }  // 可以匹配到 StringConstructor/NumberConstructor 和 () => string 等
    | PropMethod<T> // 匹配到 type: (a: number, b: string) => string 等 Function 的形式
  
  // 对于 Function 的形式，通过 PropMethod 构造成了一个和 stringConstructor 类型的类型
  // PropMethod 作为 PropType 类型之一
  // 我们写 type: Function as PropType<(a: string) => {b: string}> 的时候，就会被转化为
  // type: (new (...args: any[]) => ((a: number, b: string) => {
  //    a: boolean;
  // }) & object) | (() => (a: number, b: string) => {
  //    a: boolean;
  // }) | {
  //     (): (a: number, b: string) => {
  //         a: boolean;
  //     };
  //     new (): any;
  //     readonly prototype: any;
  // }
  // 然后在 InferPropType 中就可以推断出 （a:number,b:string）=> {a: boolean}
  type PropMethod<T, TConstructor = any> = 
    T extends (...args: any) => any // if is function with args
    ? { 
        new (): TConstructor; 
        (): T; 
        readonly prototype: TConstructor 
      } // Create Function like constructor
    : never
  ```



- RequiredKeys

  这个用来从 props 中分离出一定会有值的 key，源码如下

  ```typescript
  type RequiredKeys<T> = {
    [K in keyof T]: T[K] extends
      | { required: true }
      | { default: any }
      // don't mark Boolean props as undefined
      | BooleanConstructor
      | { type: BooleanConstructor }
      ? K
      : never
  }[keyof T]
  ```

  除了明确定义 reqruied 以外，还包含有 default 值，或者 boolean 类型。因为对于 boolean 来说如果我们不传入，就默认为 false；而有 default 值的 prop，一定不会是 undefined

- OptionalKeys

  有了 RequiredKeys, OptionsKeys 就很简单了：排除了 RequiredKeys 即可

  ```
  type OptionalKeys<T> = Exclude<keyof T, RequiredKeys<T>>
  ```

ExtractDefaultPropTypes 和 ExtractPropTypes 类似，就不写了。



推导 options 中的 method，computed, data 返回值， 都和上面推导 props 类似。



### emits options

vue3 的 options 中新增加了一个 emits 配置，可以显示的配置我们在组件中要派发的事件。配置在 emits 中的事件，在我们写 `$emit` 的时候，会作为函数的第一个参数进行提示。

对获取 emits 中配置值的方式和上面获取 props 中的类型是类似的。`$emit`的提示，则是通过 `ThisType` 来实现的(关于 ThisType 参考另外一篇文章介绍)。下面是简化的 demo

```typescript
declare function define<T>(props:{
  emits: T,
  method?: {[key: string]: (...arg: any) => any}
} & ThisType<{
  $emits: (arg: T) => void
}>):T

const result = define({
  emits: {
    key: '123'
  },
  method: {
    fn() {
      this.$emits(/*这里会提示：arg: {
          key: string;
      }*/)
    }
  }
})
```

上面会推导出 T 为 emits 中的类型。然后 `& ThisType` ，使得在 method 中就可以使用 `this.$emit`。再将 T 作为 $emit 的参数类型，就可以在写 `this.$emit`的时候进行提示了。

然后看 vue3 中的实现

```typescript
export function defineComponent<
  //... 省却其他的
  E extends EmitsOptions = Record<string, any>,
  //...
>(
  options: ComponentOptionsWithObjectProps<
    //...
    E,
    //...
  >
): DefineComponent<PropsOptions, RawBindings, D, C, M, Mixin, Extends, E, EE>

export type ComponentOptionsWithObjectProps<
  //..
  E extends EmitsOptions = EmitsOptions,
  //...
> = ComponentOptionsBase< // 定义一个 E 的泛型
  //...
  E,
  //...
> & {
  props: PropsOptions & ThisType<void>
} & ThisType<
    CreateComponentPublicInstance<  // 利用 ThisType 实现 $emit 中的提示
      //...
      E,
      //...
    >
  >
    
// ComponentOptionsWithObjectProps 中 包含了 ComponentOptionsBase
export interface ComponentOptionsBase<
  //...
  E extends EmitsOptions, // type EmitsOptions = Record<string, ((...args: any[]) => any) | null> | string[]
  EE extends string = string,
  Defaults = {}
>
  extends LegacyOptions<Props, D, C, M, Mixin, Extends>,
    ComponentInternalOptions,
    ComponentCustomOptions {
      //..
      emits?: (E | EE[]) & ThisType<void>  // 推断出 E 的类型
}
      
export type ComponentPublicInstance<
  //...
  E extends EmitsOptions = {},
  //...
> = {
  //...
  $emit: EmitFn<E>  // EmitFn 来提取出 E 中的 key
  //...
}
```



在一边学习一边实践的时候踩到一个坑。踩坑过程：将 emits 的推导过程实现了一下

```typescript
export type ObjectEmitsOptions = Record<
  string,
  ((...args: any[]) => any) | null
>
export type EmitsOptions = ObjectEmitsOptions | string[];

declare function define<E extends EmitsOptions = Record<string, any>, EE extends string = string>(options: E| EE[]): (E | EE[]) & ThisType<void>
```

然后用下面的方式来验证结果

```
const emit = ['key1', 'key2']
const a = define(emit)
```

看 ts 提示的时候发现，a 的类型是 `const b: string[] & ThisType<void>`，但是实际中 vue3 中写同样数组的话，提示是 `const a: (("key1" | "key2")[] & ThisType<void>) | (("key1" | "key2")[] & ThisType<void>)`

纠结好久，最终发现写法的不同：用下面写法的话推导出来结果一致

```
define(['key1', 'key2'])
```

但是用之前的写法，通过变量传入的时候，ts 在拿到 emit 时候，就已经将其类型推导成了 `string[]`，所以 define 函数中拿到的类型就变成了 `string[]`，而不是原始的 `['key1', 'key2']`

**因此需要注意：在 vue3 中定义 emits 的时候，建议直接写在 emits 中写，不要提取为单独的变量再传给 emits**



真的要放在单独变量里的话，需要进行处理，使得 `'[key1', 'key2']` 的变量定义返回类型为 `['key1', 'key2']` 而非 `string[]`。可以使用下面两种方式：

- 方式一

  ```typescript
  const keys = ["key1", "key2"] as const; // const keys: readonly ["key1", "key2"]
  ```

  这种方式写起来比较简单。但是有一个弊端，keys 为转为 readonly 了，后期无法对 keys 进行修改。

  参考文章[2 ways to create a Union from an Array in Typescript](https://dev.to/shakyshane/2-ways-to-create-a-union-from-an-array-in-typescript-1kd6)

- 方式二

  ```typescript
  type UnionToIntersection<T> = (T extends any ? (v: T) => void : never) extends (v: infer V) => void ? V : never
  type LastOf<T> = UnionToIntersection<T extends any ? () => T : never> extends () => infer R ? R : never
  type Push<T extends any[], V> = [ ...T, V]
  
  type UnionToTuple<T, L = LastOf<T>, N = [T] extends [never] ? true : false> = N extends true ? [] : Push<UnionToTuple<Exclude<T, L>>, L>
  
  declare function tuple<T extends string>(arr: T[]): UnionToTuple<T>
  
  const c = tuple(['key1', 'key2']) // const c: ["key1", "key2"]
  ```

  首先通过 `arr: T[]` 将 `['key1', 'key2']` 转为 union，然后通过递归的方式， `LastOf` 获取 union 中的最后一个，`Push`到数组中。



#### mixins 和 extends 

vue3 中写在 mixins 或 extends 中的内容可以在 `this` 中进行提示。对于 mixins 和 extends 来说，与上面其他类型的推断有一个很大的区别：递归。所以在进行类型判断的时候，也需要进行递归处理。举个简单的例子，如下

```
const AComp = {
  methods: {
    someA(){}
  }
}
const BComp = {
  mixins: [AComp],
  methods: {
    someB() {}
  }
}
const CComp = {
  mixins: [BComp],
  methods: {
    someC() {}
  }
}
```

对于 CComp 中的 this 的提示，应该有方法 someB 和 someA。为了实现这个提示，在进行类型推断的时候，需要一个类似下面的 ThisType

```
ThisType<{
  someA
} & {
  someB
} & {
  someC
}>
```

所以对于 mixins 的处理，就需要递归获取 component 中的 mixins 中的内容，然后将嵌套的类型转化为扁平化的，通过 & 来链接。看源码中实现：

```typescript
// 判断 T 中是否有 mixin
// 如果 T 含有 mixin 那么这里结果为 false，以为 {mixin: any} {mixin?: any} 是无法互相 extends 的
type IsDefaultMixinComponent<T> = T extends ComponentOptionsMixin
  ? ComponentOptionsMixin extends T ? true : false
  : false

// 
type IntersectionMixin<T> = IsDefaultMixinComponent<T> extends true
  ? OptionTypesType<{}, {}, {}, {}, {}>  // T 不包含 mixin，那么递归结束，返回 {}
  : UnionToIntersection<ExtractMixin<T>> // 获取 T 中 Mixin 的内容进行递归

// ExtractMixin(map type) is used to resolve circularly references
type ExtractMixin<T> = {
  Mixin: MixinToOptionTypes<T>
}[T extends ComponentOptionsMixin ? 'Mixin' : never]

// 通过 infer 获取到 T 中 Mixin, 然后递归调用 IntersectionMixin<Mixin>
type MixinToOptionTypes<T> = T extends ComponentOptionsBase<
  infer P,
  infer B,
  infer D,
  infer C,
  infer M,
  infer Mixin,
  infer Extends,
  any,
  any,
  infer Defaults
>
  ? OptionTypesType<P & {}, B & {}, D & {}, C & {}, M & {}, Defaults & {}> &
      IntersectionMixin<Mixin> &
      IntersectionMixin<Extends>
  : never
```

extends 和 mixin 的过程相同。然后看 ThisType 中的处理

```typescript
ThisType<
    CreateComponentPublicInstance<
      Props,
      RawBindings,
      D,
      C,
      M,
      Mixin,
      Extends,
      E,
      Props,
      Defaults,
      false
    >
  >
export type CreateComponentPublicInstance<
  P = {},
  B = {},
  D = {},
  C extends ComputedOptions = {},
  M extends MethodOptions = {},
  Mixin extends ComponentOptionsMixin = ComponentOptionsMixin,
  Extends extends ComponentOptionsMixin = ComponentOptionsMixin,
  E extends EmitsOptions = {},
  PublicProps = P,
  Defaults = {},
  MakeDefaultsOptional extends boolean = false,
  // 将嵌套的结构转为扁平化的
  PublicMixin = IntersectionMixin<Mixin> & IntersectionMixin<Extends>,
  // 提取 props
  PublicP = UnwrapMixinsType<PublicMixin, 'P'> & EnsureNonVoid<P>,
  // 提取 RawBindings，也就是 setup 返回的内容
  PublicB = UnwrapMixinsType<PublicMixin, 'B'> & EnsureNonVoid<B>,
  // 提取 data 返回的内容
  PublicD = UnwrapMixinsType<PublicMixin, 'D'> & EnsureNonVoid<D>,
  PublicC extends ComputedOptions = UnwrapMixinsType<PublicMixin, 'C'> &
    EnsureNonVoid<C>,
  PublicM extends MethodOptions = UnwrapMixinsType<PublicMixin, 'M'> &
    EnsureNonVoid<M>,
  PublicDefaults = UnwrapMixinsType<PublicMixin, 'Defaults'> &
    EnsureNonVoid<Defaults>
> = ComponentPublicInstance< // 上面结果传给 ComponentPublicInstance，生成 this context 中的内容
  PublicP,
  PublicB,
  PublicD,
  PublicC,
  PublicM,
  E,
  PublicProps,
  PublicDefaults,
  MakeDefaultsOptional,
  ComponentOptionsBase<P, B, D, C, M, Mixin, Extends, E, string, Defaults>
>
```



（end）

