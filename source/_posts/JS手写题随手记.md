---
title: JS手写题随手记
date: 2025-01-16 17:59:17
tags:
---
## 力扣2618.检查是否是类的对象实例
> 编写一个函数，检查给定的值是否是给定类或超类的实例。传参数据类型没有限制。可能是`undefined`或`null`。
```ts
function checkIfInstanceOf(obj: any, classFunction: any): boolean {
  
};

/**
 * checkIfInstanceOf(new Date(), Date); // true
 */
```
### 前置
这题考察的是继承和原型链。首先JS使用对象实现继承。每个对象都有一条链接到另一个称作原型的对象的内部链。该原型对象有自己的原型，依此类推，直到原型是`null`的对象。然后需要认识以下概念：
- `prototype`: 在JS中，每个构造函数（或类）都会自动创建一个`prototype`对象。包含所有通过该构造函数创建的实例共享的属性和方法。
- `__proto__`: 每个被构造函数创建的对象都自动带有`__proto__`属性，指向构造函数的`prototype`对象。
- `Object.getPrototypeOf()`: 作用与上面相同。`__proto__`逐渐被弃用而它更为标准。若传入原始类型，会自动转为对应的包装对象。
- `instanceof`: 检查对象是否是某个构造函数（类）的实例。它会检查对象的原型链上是否包含构造函数的`prototype`对象。左侧操作数是一个对象，右侧操作数必须是一个有效的构造函数。

### 解题
首先特判`obj`为`null`和`undefined`与`classFunction`不是构造函数的情况。
```ts
  if (obj === null || obj === undefined || typeof classFunction !== 'function') {
    return false;
  }
```
- 解法1使用`instanceof`: 先把`obj`强行包装为`Object`，然后使用`instanceof`判断。
```ts
  return Object(obj) instanceof classFunction;
```
- 解法2查找原型链: `Object.getPrototypeOf()`返回一个对象的原型。`.prototype`返回一个构造函数的原型对象。每次向上查找`obj`的原型。直到与`classFunction.prototype`相等或变为`null`。
```ts
  while ((obj = Object.getPrototypeOf(obj)) && obj !== classFunction.prototype) ;
  return obj === classFunction.prototype;
```

---

