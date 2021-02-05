### typescript 中的 This



### this

this 在 js 中可以根据运行上上下文决定，用法比较灵活。所以 typescript 中有时候无法准确的推导出我们代码中使用的 this 是什么类型的，所以 this 就变成了 any，各种类型提示/推导啥的，也都无法使用了(注意：只有开启了 noImplicitThis 配置， ts 才会对 this 的类型进行推导)。为了解决这个问题，typescript 中 function 的可以明确的写一个 this 参数。[官网](https://www.typescriptlang.org/docs/handbook/functions.html#this)中对 this 有很详细的介绍。

还可以通过`function(this:void){}` 的方式，将 this 设置为 void 来强行约束在 function 中不能使用 this



### ThisType

另外和 this 有关的，还有一个 [ThisType](https://www.typescriptlang.org/docs/handbook/utility-types.html#thistypetype)。ThisType 本身实现没有什么具体的内容：

```typescript
interface ThisType<T> { }
```

只是用来解决{}中this的类型。当我们开启 noImplicitThis 配置的时候，对于 `ThisType<T>`来说，{} 中的 this 就会被推导为类型 T。

对于 ThisType 由于本身只是一个空的对象，所以单独使用的话，作用不是很大。例如下面的例子

```typescript
type A = {
  name: string
}
const some:ThisType<A> = {
  method(){
    this.name = '123'
  }
}
```

可以将 some 指定为 `ThisType<A>`来约束 some 内部 this 的类型，但是 some 本身的类型也变成了 {}，内部可以写任意属性。所以实际中 ThisType 和 & 一起使用可以发挥更大的作用。

官网的例子略复杂，通过下面简单的 demo 来理解

```typescript
type A = {
  name: string
}
// B & ThisType<A> 之后，就强行定义了在 B 中 this 的类型为 A
type B = {
  method: ()=>void
  id: number
} & ThisType<A>

const some:B = {
  method(){
    // 这里就可以写 this.name 不会报错
    this.name = 'xxx'
  },
  id: 1
}
```

去掉 `ThisTyps<A>`的话，ts 会将 B 中的 this 自动推断为类型 B, 也就是说我们在函数 method 中通过 this 只能引用 `this.id` 和 `this.method`













