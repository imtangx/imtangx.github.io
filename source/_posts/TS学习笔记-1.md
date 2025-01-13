---
title: TS学习笔记(1)
date: 2025-01-13 21:58:13
tags: TypeScript
---
## 基本类型与包装对象类型
在JS和TS中，`string`和`String`是有区别的。具体的说一个原始类型，它有字面量和包装对象两种情况。
```ts
  "hello"; // 字面量
  new String("hello"); // 包装对象
```
所以在对变量进行类型指定的时候也有所区别。
```ts
  const s1: string = "hello"; // 正确
  const s2: String = "hello"; //正确

  const s3: string = new String("hello"); // 报错
  const s4: String = new String("hello"); // 正确
```
可以看到`string`类型只能被赋予字面量而非包装对象。而`String`既可以被赋予字面量也可以赋予包装对象\\
对于TS来说，最好使用小写的类型，因为我们开发在使用这种原始类型的时候，通常都是以字面量进行初始化。并且TS中很多内置方法的函数需要接受的参数都定义为小写类型。
```ts
  const x: number = 1;
  const y: Number = 1;

  Math.abs(x); // 正确
  Math.abs(y); // 报错
```
`Math.abs()`的参数类型就被定义为小写类型，所以在传入`y`时会发生报错。
### Object类型与object类型
它是一种复合类型。大写的`Object`代表广义对象，除了`null`和`undefined`其他所有类型都能转成对象。在TS中，可以用`{}`空对象代指`Object`类型。
```ts
  const obj: Object = ...;
  const obj: {} = ...;
```
而小写的`object`是狭义上的对象，它只能接受字面量对象而不接受那些原始类型。
```ts
  const obj: object = {foo: 1}; // 正确
  const obj: object = 1; // 报错
```
需要注意的是，小写和大写类型都能调用对象的内置方法如`toString()`， 但是它们不能读取用户自定义的属性方法，如果需要读取用户的自定义属性需要在定义时直接给出属性。
```ts
  const obj: object = {foo: 1};
  obj.foo; // 报错

  const obj: {foo: number} = {foo: 1};
  obj.foo; // 1
```
### undefined和null的特殊性
它们既是值也是类型。并且在它们作为值的时候，所有类型的变量都能被赋值。
```ts
  const x: number = 1;
  x = undefined; // 正确
  x = null; // 正确
```
这是为了跟JS中的行为保持一致，但是可能会引发潜在的危险，导致TS的编译检查不出问题。
```ts
  const obj: object = undefined;
  obj.toString(); // 编译通过 运行时发现undefined并没有toString()方法 直接报错
```
需要避免的话在`tsconfig`中打开`strictNullChecks`，这样的话`null`和`undefined`只能被赋予给`any`和`unknown`类型，而且它们之间也不能互相赋值了，保证了一定的安全性。

---
## 值类型

在TS中，单个值也是一种类型。
```ts
  let x: "hello";
  x = "hello"; //正确
  x = "world"; //报错
```
上面代码意味着x只能被`hello`这个字符串赋值。还有一种写法也能做到上面的效果，那就是`const`定义的变量如果没有被指定类型且被赋值为一个字面量，TS就会自动推断。
```ts
  const x = "imtx";
  x = "bubu"; //报错
```
TS推断变量`x`只能被赋予字符串`imtx`，这是很合理的。因为在JS中，`const`类型定义的变量`x`确实不能再被改变了。但当`x`是对象类型的时候就不会推断了，因为JS中，`const`定义的对象只是本身不能被改变，但是它内部的属性方法可以被改变。

---
## 运算符
### keyof运算符
返回一个对象所有键名组合的联合类型。
```ts
  type obj = {
    foo: number,
    bar: string,
  }
  type keys = keyof obj; // 'foo' | 'bar' 
```
### in运算符
JS中的`in`运算符是确定对象是否包含某个属性。而TS中的`in`运算符是用于取出联合类型的每一个成员类型。
```ts
  type U = "a" | "b" | "c";

  type Foo = {
    [Prop in U]: number;
  };
  // 等同于
  type Foo = {
    a: number;
    b: number;
    c: number;
  };
```
### is运算符
用于确定一个变量是否属于某个属性。需要注意的是，它不是一个普通的操作符，不能以`A is type`一个表达式来单独写出。而通常放在函数返回类型中，用于类型保护。
```ts
  const x: number = 1;
  x is number; // 报错

  const isNumber: (arg: any) => arg is number = (arg) => {
    return typeof arg === 'number';
  };

  if (isNumber(x)) { // 正确
    ...
  }
```

---
## 类型映射
对于A和B两个类型，如果它们有很多相同的属性名，但类型不同。那么在定义的时候就略显繁琐。这时候可以结合上面的运算符来进行类型映射。
```ts
  type A = {
    foo: string;
    bar: number;
  }

  type B = {
    foo: boolean;
    bar: boolean;
  }
```
等价于：
```ts
  type A = {
    foo: string;
    bar: number;
  }

  type B = {
    [prop in keyof A]: boolean;
  }
```
这样做的话，会将所有B内的属性名都定义为boolean类型，如果需要A和B的属性名及类型都是相同的，可以使用[]运算符做到。
```ts
  type A = {
    foo: number;
    bar: string;
  }

  type B = {
    [prop in keyof A]: A[prop];
  }
```
对上面的代码分析如下:
- prop: 存储属性名的代指变量，可以任意取名。
- keyof: 取出A中所有成员组成的联合类型。
- in: 取出上述联合类型中的每一个成员。
- A[prop]: 使用prop取出A中属性对应的相应类型赋给B。

---
对于类型映射，还有一些进阶用法。
### `+ -`修饰符
普通映射无法映射属性的可读和可选属性，
- `+`修饰符：写成`+?`或`+readonly`，为映射属性添加`?`修饰符或`readonly`修饰符。
- `–`修饰符：写成`-?`或`-readonly`，为映射属性移除`?`修饰符或`readonly`修饰符。
### 键名重映射
在上面我们原封不动的复制了A的属性名，`TypeScript 4.1`引入键名重映射可以改变名字，用法如下:
```ts
  type A = {
    foo: number;
    bar: number;
  };

  type B = {
    [p in keyof A as `${p}ID`]: number;
  };

  // 等同于
  type B = {
    fooID: number;
    barID: number;
  }；
```
### 属性过滤
键名重映射还可以过滤掉某些属性。下面的例子是只保留字符串属性。
```ts
  type User = {
    name: string;
    age: number;
  };

  type Filter<T> = {
    [K in keyof T as T[K] extends string ? K : never]: string;
  };

  type FilteredUser = Filter<User>; // { name: string }
```