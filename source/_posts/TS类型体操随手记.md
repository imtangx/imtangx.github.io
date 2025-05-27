---
title: TS类型体操随手记（持续更新）
date: 2025-05-28 00:20:35
tags: ['TypeScript']
---

## 前言

在学习 Typescript 的时候都只在项目简单的使用，很多高级类型都没有接触过。所以最近开始学习 TypeScript 的类型体操，一方面熟悉高级类型，也锻炼自己的手写 TS 类型的能力。

过程中也发现同一个关键字在不同场景下的用法是不一样的，所以记录一下。

## 常见语法

| 特性              | `interface`                       | `type`                                                       |
| ----------------- | --------------------------------- | ------------------------------------------------------------ | -------- |
| **语法**          | `interface User { name: string }` | `type User = { name: string }`                               |
| **扩展方式**      | `extends` 继承                    | `&` 交叉类型                                                 |
| **声明合并**      | ✅ 支持（自动合并同名接口）       | ❌ 不支持                                                    |
| **类实现**        | ✅ 可被 `class` 实现              | ❌ 不能被类直接实现                                          |
| **基本类型别名**  | ❌ 不能                           | ✅ 如 `type ID = string                                      | number`  |
| **联合类型/元组** | ❌ 不能直接定义                   | ✅ 如 `type Result = Success                                 | Failure` |
| **映射类型**      | ❌ 不能直接使用                   | ✅ 如 `type Readonly<T> = { readonly [P in keyof T]: T[P] }` |
| **适用场景**      | 对象类型、类声明、需要声明合并时  | 复杂类型、联合类型、工具类型时                               |

### extends

| 用途               | 语法示例                        | 说明                                        |
| ------------------ | ------------------------------- | ------------------------------------------- |
| **接口继承**       | `interface B extends A { ... }` | B 继承 A 的所有类型属性                     |
| **泛型约束**       | `<T extends U>`                 | 限制泛型参 T 必须是 U 的子类型              |
| **条件类型**       | `T extends U ? X : Y`           | 判断 T 是否是 U 的子类型                    |
| **分布式条件类型** | `T extends any ? T[] : never`   | 当 T 是联合类型时取出每个类型进行子类型判断 |

### infer

| 用途         | 语法示例                                                                | 说明                     |
| ------------ | ----------------------------------------------------------------------- | ------------------------ |
| **类型推断** | `type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any;` | 推断函数的返回值存储为 R |

### keyof

| 用途             | 语法示例                                   | 说明  |
| ---------------- | ------------------------------------------ | ----- | ---------- | ----------------------------------------- |
| **获取联合类型** | `type PersonKeys = keyof Person; // "name" | "age" | "address"` | 可以取出 interface 的所有类型变成联合类型 |

### in

| 用途         | 语法示例             | 说明                             |
| ------------ | -------------------- | -------------------------------- |
| **映射类型** | `[S in Keys]: T[S];` | 遍历 Keys 这个联合类型创建新类型 |
| **类型保护** | `xxx in T`           | 检查对象是否包含特定属性         |

### as

| 用途         | 语法示例                                   | 说明                                                     |
| ------------ | ------------------------------------------ | -------------------------------------------------------- |
| **类型断言** | `const value = xxx as string;`             | 把 xxx 变量断言成 string 类型                            |
| **重映射**   | `[K in keyof T as NewKeyExpression]: T[K]` | K 是 T 联合类型的映射类型然后被重映射到 NewKeyExpression |

## 类型体操
### 实现Pick
Pick<T, K> 用于从类型 T 中选出符合 K 的属性，构造一个新的类型，T是对象类型，K是联合类型。
```ts
type MyPick<T, K extends keyof T> = {
  [S in K]: T[S]
} 
```
思路：
1. 限制K是T联合类型的子类型，使用keyof取出T的所有类型
2. 使用in遍历K，把K的类型映射到新的类型中，使用T[S]取出T的属性值

### 实现Omit
Omit<T, K> 用于从类型 T 中选出不符合 K 的属性，构造一个新的类型，T是对象类型，K是联合类型。
```ts
type MyOmit<T, K extends keyof T> = {
  [S in keyof T as S extends K ? never : S]: T[S];
};
```
思路：
1. 限制K是T联合类型的子类型，使用keyof取出T的所有类型
2. 使用in遍历T，把T的类型as映射到新的类型中，使用S extends K ? never : S判断S是否是K的子类型
3. 是则返回never，不是则返回S

### 元组转换为对象
TS中的类型数组叫做元组，你需要将这个元组类型转换成对象类型，这个对象类型的键/值都是从元组中遍历出来。
```ts
type TupleToObject<T extends readonly (string | number | symbol)[]> = {
  [S in T[number]]: S
}
```
思路：
1. 限制T是元组类型，使用readonly限制数组只读，使用(string | number | symbol)[]限制对象key类型
2. 使用in遍历T，因为T是个数组，使用T[number]取出T的下标/属性值，使用S作为value

### Includes
Includes<T, K>这个类型接受两个参数，查看K是否是T的某一项，T是元组类型，K是单一类型。
```ts
// 比较两个类型是否严格相等
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends <T>() => T extends Y ? 1 : 2
  ? true
  : false
  
type Includes<T extends readonly any[], U> = T extends [infer S, ...infer Rest]
  ? Equal<U, S> extends true
    ? true
    : Includes<Rest, U>
  : false;
```
思路：
1. 使用解构语法和infer取出T的第一个元素和剩余元素
2. 使用Equal比较U和S是否相等
3. 相等则返回true，否则递归调用Includes

### Awaited
`Promise<ExampleType>`，请你返回 ExampleType 类型。
```ts
type MyAwaited<T extends PromiseLike<any>> = T extends PromiseLike<infer S>
  ? S extends PromiseLike<any>
    ? MyAwaited<S>
    : S
  : never;
```
思路：
1. 使用infer取出T的类型
2. 使用S extends PromiseLike<any>判断S是否是PromiseLike类型
3. 是则递归调用MyAwaited，否则返回S
