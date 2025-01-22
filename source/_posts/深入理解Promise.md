---
title: 深入理解Promise与async/await
date: 2025-01-22 21:43:15
tags: [Javascript, Promise]
cover:
---
# `Promise`
`Promise`是在`ES6`被引入用于简化异步编程的一个语言特性。它是一个对象，表示某个异步操作的结果，因此它有一些状态分别为`pending(待定)`、`fulfill(兑现)`和`reject(拒绝)`。它只可能从`pending->fuifill/reject`，也就是说一旦这个`Promise`被`settle(敲定)`，就不会再被改变了。
## 好处
传统的异步编程如果需要多次回调，会嵌套形成回调地狱，而在下面说到的`Promise`链就可以解决这个问题。其次传统异步编程如果想要捕获异常错误，需要使用回调参数层层传递错误，而`Promise`也可以优雅的解决这个问题。
## 使用
对于一个异步操作例如`fetch`请求网页数据，可能会在此之后进行一些操作比如展示数据。就需要用到`then()`方法，它可以理解为在上次异步操作结束之后会执行这个`then()`。无可避免的是，涉及网络请求有时会发生一些错误，所以`then()`可以传入两个回调函数，第一个在`Promise`兑现时调用，第二个在`Promise`被拒绝/抛出异常时调用。但是在日常使用`Promise`时更多会使用`catch()`来处理第二种情况，它其实就是`then(null, )`的一种简写方式，不仅更加语义化并且有下面这个情景的好处。
```js
getJSON('api/user/profile').then(displayUserProfile, handleError);
getJSON('api/user/profile').then(displayUserProfile).catch(handleError);
```
第二种写法的好处在于，如果在`displayUserProfile`中抛出了异常，也能被`handleError`处理。\
此外还有`finally()`方法，用于在`Promise`被敲定（无论兑现还是拒绝）时调用，它不接受参数并且只会在抛出异常的时候被注意到返回值。一般用于运行一些清理代码。\
## 创建自己的`Promise`
上面的操作都建立在可以返回`Promise`的函数上进行操作，我们可以基于构造函数创建属于自己的`Promise`。
```js
new Promise((resolve, reject) => {
  if (...) {
    reject(new Error("..."));
  }

  resolve(...);
})
```
其需要传入两个参数，分别代表解决或拒绝这个`Promise`。为什么说是解决(resolved)而不是兑现呢，因为如果你在`resolve()`传入了一个新的`Promise`，将会以这个`Promise`作为最后的返回值，但一般我们会传入一个值，这样就会使用这个值返回并立即兑现。\
根据开头提到的，一个`Promise`一旦被敲定就不会再变化了，所以如果代码如下：
```js
new Promise((resolve, reject) => {
  resolve(...1); // 执行
  reject(...2); // 不执行
  console.log('imtx'); // 执行 因为它并不影响Promise的结果
  resolve(...3); // 不执行
  console.log('bubu'); // 执行 同理
})
```
## `Promise`的其他方法
### `Promise.all()`
也可以叫作并行`Promise`，当一些`Promise`它们之间没有相互依赖的时候，你可以并行执行它们以提高效率。
```js
Promise.all([promise1, promise2, ...])
```
它接受一个数组作为参数并输出一个数组作为这些`Promise`的结果，如果数组存在非`Promise`值原封不动的输出。当某个`Promise`被拒绝的时候，会立即返回并输出这个`Promise`拒绝的原因，只有当所有`Promise`都被兑现的时候，才会输出这个数组。
### `Promise.allSettled()`
它跟上面的`all()`唯一的区别就是，在某个`Promise`拒绝的时候不会直接返回，而是等到所有`Promise`敲定后输出结果。
### `Promise.race()`
它跟all()的区别是，会立即返回第一个兑现/拒绝的`Promise`，如果数组中存在非`Promise`会直接返回。
### `Promise.any()`
正如名字所言，数组中只要有兑现的`Promise`就会返回这个兑现结果，如果全部都被拒绝才会返回错误。
## 顺序串行`Promise`
如果你想要让一些`Promise`以某种顺序运行，通常可以使用`.then()`的方法依次向下写，但如果这个数组的内容未知，通常需要自己动态的构建`Promise`链。以下是犀牛书的示例代码：
```js
function fetchSequentially(urls) {
  const bodies = [];

  // 返回一个Promise 保存一个响应体
  function fetchOne(url) {
      return fetch(url)
          .then(response => response.text())
          .then(body => {
              bodies.push(body);
          });
  }

  // 初始化成一个已经兑现的Promise
  let p = Promise.resolve(undefined);

  // 动态构建
  for(const url of urls) {
      p = p.then(() => fetchOne(url));
  }

  // Promise全部兑现后 将bodies作为Promise返回
  return p.then(() => bodies);
}

fetchSequentially(urls)
    .then(bodies => { ... })
    .catch(e => console.error(e));
```

---

# `async/await`
`Promise`是用于简化异步编程，而这两个关键字是用于简化`Promise`的使用。它们会让`Promise`看起来同步代码一样，提高代码的可读性。
## 使用
通常情况下`async`和`await`是一同使用的，如果你想使用`await`表达式就必须处于一个`async`函数中，代码如下：
```js
async function getName() {
  const response = await fetch('/api/user/profile');
  const profile = await response.json();
  return proflie.name;
} 
```
原本在`Promise`需要两个`then()`来连接，如果使用`async/await`就可以编写的像同步代码一样优雅。\
在较新的`ES`版本中，支持了`顶级await`，即不需要在`async`函数中也可以使用，比如你想在代码的头部加载一个外部模块，这可能是一个外部地址需要用到`fetch`请求，代码如下。并且这个写法只能在`ES`模块文件中使用。
```js
const response = await fetch('https://api.example.com/data');
const data = await response.json();
console.log(data);
```
## 并行`await`
`await`会阻塞当前的异步操作，在上面的代码中是没有问题的，因为`profile`依赖于`response`。但如果是对两个不同的网址进行`fetch`，这个阻塞操作就会降低效率。就可以用到上面的`Promise.all()`。
```js
const res1 = await fetch(url1);
const res2 = await fetch(url2);
```
```js
const [res1, res2] = await Promise.all([fetch(url1), fetch(url2)]);
```
## 串行`await`
如果想像上面一样串行的执行`Promise`，也可以结合`await`和`for`循环来实现。
```js
async function processPromises(promises) {
  for (const promise of promises) {
    const result = await promise;
    console.log(result); 
  }
}
```
## `async`函数细节
`async`的函数工作原理可以想象看作这样：
```js
async function f(x) { /* 函数体 */ };

function f(x) {
  return new Promise((resolve, reject) => {
    try {
      resolve((function(x) { /* 函数体 */ })(x));
    } catch (err) {
      reject(err);
    }
  })
}
```

---

# `Promise、async/await`语法规则
以下内容可能会在代码输出题中考察。
- 如果进入`catch`并处理抛出这个错误才会停止向下传播，否则就会继续往下传播。
- `finally`并不代表最后一次操作，在它之后的`then`语句依然会执行。
- `then()`如果传入的不是一个函数而是一个数字或`Promise`对象，那么会被忽略。
- `await`后面的代码会被放入微任务队列，相当于放入`then()`。