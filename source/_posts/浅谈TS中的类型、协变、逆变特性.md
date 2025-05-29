---
title: 浅谈TS中的类型、协变、逆变特性
date: 2025-05-30 01:17:53
tags: ['TypeScript']
---

## 父/子类型

在真实的开发中，常常需要用到 TS 的父/子类型进行代码类型的复用维护。所以怎么写出完善的类型是比较重要的。

首先需要搞清楚父/子类型的定义：

```ts
interface Animal {
  age: number;
}

interface Dog extends Animal {
  bark(): void;
}
```

上面的 Dog 就是 Animal 的子类型。子类型通常由父类型扩展而来，它可以拥有更多的属性，更加的"具体"。

但是在集合论中，子集是父集的子集，也就是子集的属性相比父集会更少，这是容易搞混的。一个联合类型的例子可以很好的解释：

```ts
type A = string;
type B = string | number;

A extends B; // true
```

联合类型可以看作一个集合， A 是 B 的子集，因为它的可能性更少，也就更"具体"。所以在类型上看，A 是 B 的子类型。

## 可赋值性

在开发中难免会遇到不同类型之间的互相赋值。比如：

```ts
let dog: Dog = {
  age: 1,
  bark: () => {},
};

let animal: Animal = {
  age: 2,
};

animal = dog; // ok
dog = animal; // error
```

上面的代码中， Dog 是 Animal 的子类型，Dog 有更多的属性，所以一定可以安全的赋值给 Animal。但是反过来就不行了， 因为 Animal 可能会缺失 bark()方法。

再来看一个联合类型的例子：

```ts
type A = 'a';
type B = 'a' | 'b';

let s1: A = 'a';
let s2: B = 'b';
s2 = s1; // ok
s1 = s2; // error
```

上面的代码中， A 是 B 的子集也是子类型，所以可以安全的赋值给 B。但是反过来就不行了，因为 s1 可能会缺失 'b' 这个属性。

## 协变

稍微总结一下，上面的赋值关系貌似是父类型 <- 子类型，也就是不具体 <- 具体。这也叫做协变。

同样满足协变特性的还有数组：

```ts
let arr1: string[] = ['a'];
let arr2: string[] | number[] = ['b', 1];
arr2 = arr1; // ok
arr1 = arr2; // error
```

以及函数的返回值：

```ts
let f1: () => string = () => 'imtx';
let f2: () => string | number = () => 123;

f2 = f1; // ok
f1 = f2; // error
```

## 逆变

除了上面的情况，还有一种情况是子类型 <- 父类型，也就是具体 <- 不具体。这也叫做逆变。
逆变的情况主要出现在函数的参数上：

```ts
let f3: (arg: string) => void = (arg: string) => {};
let f4: (arg: string | number) => void = (arg: string | number) => {};

f3 = f4; // ok
f4 = f3; // error
```

上面的代码中，f3 的参数是 number，f4 的参数是 number | string，所以 f3 参数是 f4 参数的子类型。那是不是意味着 f3 可以安全的赋值给 f4 呢？
答案是否定的，假如 f3 的实现是这样的：

```ts
f3 = (arg: string) => {
  console.log(arg.length);
};
```

执行`f4 = f3`后，这时再调用`f4(1)`就会因为没有`.length`属性而报错。
反过来`f3 = f4`，因为 f3 只能传入 string，而 f4 函数体内一定存在处理 string 的方法，所以就不会有报错的可能。

## 双变

在老版本的 TS 中，函数参数是双向协变的。也就是说，既可以协变又可以逆变，但是这并不是类型安全的。 在新版本 TS (2.6+) 中 ，你可以通过开启 strictFunctionTypes 或 strict 来修复这个问题。设置之后，函数参数就不再是双向协变的了。

## 不变

不变就非常好理解了，就是不允许变型。比如基本类型（除 any/unknown）之间不能互相赋值，

## 总结

不管是协变还是逆变，归根到底都是在保证类型安全的前提下，提供一些灵活性。
