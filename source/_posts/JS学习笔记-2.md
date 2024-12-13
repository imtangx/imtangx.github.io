---
title: JS学习笔记(2)
date: 2024-12-13 14:19:13
tags: Javascript
---
## `...`运算符
剩余参数：在函数定义中用于接受传入的多余参数。\
扩展运算符：可以将一个数组or对象的元素展开。
```js
const func = (a, b, ...args) => { // 接受剩余的[4, 5, 6]
    console.log(a, b, ...args); // 1, 2, 4, 5, 6
    console.log(a, b, args); // 1, 2, [4, 5, 6]
};

func(1, 2, 4, 5, 6);
```

---

## 数组
```js
let arr = [1, 2, 3];
arr[5] = 10;
console.log(arr); // [ 1, 2, 3, <2 empty items>, 10 ]
```
`[]`访问超过数组长度的下标时，数组长度会自动扩充，剩余为空槽。
```js
const cmp = (a, b) => {
    if (a < b) return -1; // a排在b前面
    if (a > b) return 1; // b排在a前面
    return 0; // 不交换
}
```
`JS`中的自定义比较函数有三个返回值，`-1, 1, 0`。

---

## 带键的集合`Map 、Set`
- Map中的键可以任意类型， 而Object的键只能是String或Symbol（接受String参数的别名）。
- Map将原始值存储为键， 而Object将每个键都视为字符串。
- Map和Set的遍历顺序都是其插入的顺序。

---

## 继承与原型链
在`JS`中， 继承是通过对象实现的。每个对象都有一条链接到另一个称为原型的对象的内部链，以此类推，直到原型为`null`的对象作为原型链的最后的一环。这个过程是动态的，即你可以在运行时修改原型链上的任何成员。
### 继承属性
每个对象都有一条指向原型对象的链， 当需要访问对象的属性时， 会先访问该对象本身，如果没找到会继续往它的原型上查找属性。如果跟原型都存在相同的属性，原型上的属性值会被遮蔽。\
>备注：根据 ECMAScript 标准，符号 `someObject.[[Prototype]]` 用于指定 `someObject` 的原型。使用 `Object.getPrototypeOf()` 和 `Object.setPrototypeOf()` 函数分别访问和修改 [[Prototype]] 内部插槽。这与 JavaScript 访问器 `__proto__` 是等价的，后者是非标准的，但许多 JavaScript 引擎实际上实现了它。为了保持简洁和避免困惑，在我们的表示法中，我们会避免使用 `obj.__proto__`，而是使用 `obj.[[Prototype]]`。其对应于 `Object.getPrototypeOf(obj)`。

注意：虽然`obj.__proto__`访问器时非标准的，但是`{ __proto__: ... }`语法是标准的。
```js
const myObj = {
    a: 1,
    b: 2,

    __proto__: {
        b: 3,
        d: 4,
    },
    // { a: 1, b: 2 } ---> { b: 3, c: 4 } ---> Object.prototype ---> null
};

console.log(myObj.a); // 1
console.log(myObj.b); // 2
console.log(myObj.c); // 4
```
注意：`Object.prototype`隐式的成为了原型，`Array` 和 `RegExp`、`Function`也是一样的都有自己的原型， 并且它们的原型的原型还是`Object.prototype`。但**箭头函数**没有原型。
```js
function doSomething() {}
console.log(doSomething.prototype); // {}

const doSomethingFromArrowFunction = () => {};
console.log(doSomethingFromArrowFunction.prototype); // undefined
```
### 创建对象和改变原型链的方法总结
#### 语法结构创建
```js
const myObj = {
    a: 1, 
    __proto_: { //只能指向对象
        b: 2,
    },
};
```
#### 构造函数
使用 `new ()`创建对象，调用原型的构造函数。
#### 使用`Object.create()`
调用 `Object.create()` 会创建一个新对象。该对象的 `[[Prototype]]` 是该函数的第一个参数，如果不传参， 则为`Object.prototype`。
#### 使用类`extends`
声明类的后面跟上`extends`指定要继承的类。
#### 使用`Object.setPrototypeOf()`
传入两个参数`(a, b)`，表示吧a的原型设为b。
#### ~~使用`__proto__`访问器~~(非标准)。

### 性能
查找位于原型链上层的属性花费的时间可能会对性能有负面影响。所以在检查对象本身是否存在某个属性的时候，可以使用`hasOwnProperty`或者`Object.hasOwn`方法。

---

## 类
### 私有字段
在属性or方法前面加上`#`， 即可将其定义为私有字段。私有属性只能在类内部访问，私有方法只能在类内部调用。

### 静态属性
在方法前面加上`static`， 这个方法不能从实例中访问， 只能通过类调用。

### 扩展与继承
如上面的`extends`， 即派生类。

---

## 迭代器和生成器
### 迭代器
迭代器是一个对象，它通过`next()`方法去迭代某个实现了迭代器协议的对象。其返回拥有两个属性的对象：
- value: 迭代序列的下一个值。
- done: `true`则为迭代结束。
```js
function makeRangeIterator(start = 0, end = Infinity, step = 1) {
  let nextIndex = start;
  let iterationCount = 0;

  const rangeIterator = {
    next() {
      let result;
      if (nextIndex < end) {
        result = { value: nextIndex, done: false };
        nextIndex += step;
        iterationCount++;
        return result;
      }
      return { value: iterationCount, done: true };
    },
  };
  return rangeIterator;
}
```
上面的代码，首先从创建了`makeRangeIterator`函数，然后在里面创建了`rangeIterator`对象，然后使用它的`next()`方法来创建迭代器。函数的返回值是一个迭代器对象，它可以用`next()`来进行持续迭代。

### 生成器函数
上面实现的是自定义迭代器，但是`JS`中提供了更方便的工具（Generator 函数）来作为迭代算法。它使用`function*`语法编写。\
这种生成器函数返回一种特殊迭代器-**生成器**。每次调用`next()`将会执行，直至遇到`yield`关键字得到值。
```js
function* makeRangeIterator(start = 0, end = Infinity, step = 1) {
  let iterationCount = 0;
  for (let i = start; i < end; i += step) {
    iterationCount++;
    yield i;
  }
  return iterationCount;
}
```
这是使用`function*`对上面代码的改写。

### 可迭代对象
常见的可迭代对象有`Array`和`Map`等。它们可以通过`for...of`循环遍历。\
但如果要对一些不可迭代对象迭代呢?比如`Object`。可以自己在对象内部实现`[Symbol.iterator]()`方法，使其成为可迭代对象。
```js
const myIterator = {
  arr: [1, 2, 3, 4],
  [Symbol.iterator]() {
    let idx = 0;
    const self = this;  // 保留对原始对象的引用
    return {
      next() {
        if (idx < self.arr.length) {
          return { value: self.arr[idx++], done: false };
        }
        return { done: true };
      },
      [Symbol.iterator]() {
        return this;  // 返回自己作为迭代器
      }
    };
  },
};

const it = myIterator[Symbol.iterator]();  // 获取迭代器对象
for (const item of it) {
  console.log(item);  // 输出: 1, 2, 3, 4
}
```

### 高级生成器
`next()`也可以接受参数以修改生成器的状态，常见做法是根据参数重启生成器。











