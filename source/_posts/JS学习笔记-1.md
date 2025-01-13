---
title: JS学习笔记(1)
date: 2024-12-11 19:41:33
tags: Javascript
---

## 三种声明方式 `var let const`
首先， js中对于访问未声明的变量会抛出`ReferenceError`， 对于未初始化的const变量会抛出`SyntaxError`。\
`let`和`const`都作用于块状`{}`作用域， `var`不同。
```js
if (true) {
    var x = 1;
    let y = 2;
    const z = 3;
}

console.log(x); // x = 1
console.log(y); // ReferenceError
console.log(z); // ReferenceError
```
并且`var`类型的变量会在**作用域内**被**提升**到顶部。
```js
console.log(x); // undifined
var x = 1;

(function () {
    console.log(x); //undifined 
    var x;
})();
```
第一个`x`被提升到全局作用域顶部，第二个`x`被提升到函数作用域顶部，所以第二次读取不到`x = 1`。

---

## 数据结构和类型
七种基本数据类型: 
- boolean
- null
- undifined
- number
- bigInt
- string
- symbol \
以及:
- object

在`JS`中，数据类型无需在声明时指定，执行期间会自动推断。并且可以重新赋值为不同类型。\
涉及字符串和数字相加时，会把数字转换为字符串。\
将字符串转为数字有两种方法：
```js
parseInt("101", 2) // 5
(+"1.1") + (+"1.1") = 2.2 // 在字符串前面加上+
```

---

## 字面量
### 数组字面量
在其连续放置两个`, ,`，会为其留下一个空槽`<1 empty item>`。如果在末尾加上多个`,`，最后一个会被忽略，其余会被视为空槽。
### 对象字面量
对象的属性有两种访问方式`.`和`[]`。字符串和数字常量也可作为对象属性名，访问时使用`[]`。
```js
const myObj = {
    name: "imtx",
    "7": 123,
}
console.log(myObj[7]);
```

---
## 错误处理
`throw`用于抛出异常(变量or常量)，通常与`try...catch`搭配抓取异常。
```js
try {
    throw new Error('异常消息');
} catch (err) {
    console.error(err.name); // error
    console.error(err.message); // '异常消息'
} finally { //总是执行
  
}
```

---
## 循环
### `Label`语句
类似于`cpp`的`goto`语句，可以跳到指定的位置。
```js
outPoint: for (let i = 0; i < 10; i++) {
    for (let j = 0; j < 10; j++) {
        if (i == 5 && j == 5) {
            break outPoint; // 会直接跳出整个循环 执行到outPoint块下方
        }
    }
}
```
### `for...in` 和 `for...of`
```js
let arr = [7, 1, 5]
arr.foo = "imtx"

for (let i in arr) {
    console.log(i); // 0, 1, 2, foo
}

for (let i of arr) {
    console.log(i); // 7, 1, 5
}
```
`for...in`会遍历可迭代对象的所有键名， 注意不仅仅是索引。`for...of`会遍历可迭代对象的所有元素值。

---

## 函数
### 传参
如果函数传参是普通类型，在函数内部修改只是对拷贝值进行修改。但如果传入对象或数组，这样的修改是有效的。
```js
let person = {
    name: 'imtx',
}   
function myfunc(person) {
    person.name = 'gamejoye';
}

console.log(person.name); // imtx
myfunc(person);
console.log(person.name); // gamejoye
```
### 函数提升
```js
console.log(func(2)); // 4

function func(x) {
    return 2 * x;
}
```
函数的声明会被提升到作用域顶部，但函数表达式不会。
```js
console.log(F(2)); // error

const F = function func(x) {
    return 2 * x;
}
```
### 嵌套函数和闭包
嵌套（内部）函数对其容器（外部）函数是私有的。它自身形成了一个闭包， 闭包拥有自己的独立变量和它的作用域内的参数、变量。换句话说， 内部函数包含外部函数的作用域。
- 内部函数只可以在外部函数中访问。
- 内部函数形成了一个闭包：它可以访问外部函数的参数和变量，但是外部函数却不能使用它的参数和变量。
```js
function outSide(x) {
    function inSide(y) {
        return x + y;
    }
    return inSide;
}

const fnInside = outside(3); // 可以这样想：给我一个可以将提供的值加上 3 的函数
console.log(fnInside(5)); // 8
console.log(outside(3)(5)); // 8
```
这是嵌套函数的参数调用方式。
```js
const countCalc = function () {
    let count = 0;
    return {
        add() {
            return ++count;
        },
        sub() {
            return --count;
        },
        getValue() {
            return count;
        }
    }
}

const counter = countCalc();

console.log(counter.sub()); // -1
console.log(counter.add()); // 0
console.log(counter.getValue()); // 0
console.log(counter.count); //undifined 访问不到内部变量
```
上面的代码上， 外部的`count`变量对外界不可访问，但对于内部函数是可以访问的， 因为它们形成了**闭包**。通过暴露公开接口来访问数据，是一种封装的体现。

### arguments对象
函数的实际传参会被保存在`arguments`对象中，它是一个类数组，可以访问下标和调用`.length`来遍历。
```js
function myConnect (ch) {
    let res = '';
    for (let i = 1; i < arguments.length; i++) {
        res += arguments[i] + ch;
    }
  return res;
}
console.log(myConnect(',', '大象', '猴子', '猪')) // 大象,猴子,猪,
```
### 默认参数
### 剩余参数
`...args`是一个剩余参数可以读取函数传入的多余参数。
```js
function multiply(multiplier, ...theArgs) {
    return theArgs.map((x) => multiplier * x);
}

const arr = multiply(2, 1, 2, 3);
console.log(arr); // [2, 4, 6]
```
### 箭头函数
`=>`是一种非常简洁的函数语法。
```js
const pw = (x) => (x * x);
console.log(pw(3)); // 9
```