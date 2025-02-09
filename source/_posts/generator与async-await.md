---
title: Generator与async/await
date: 2025-01-27 15:43:49
tags: [JavaScript]
---
## 协程
协程（coroutine）是一种异步编程解决方案。意思是多个线程互相协作，完成异步任务。

协程的运行流程大致如下：
- 第一步，协程A开始执行。
- 第二步，协程A执行到一半，进入暂停，执行权转移到协程B。
- 第三步，（一段时间后）协程B交还执行权。
- 第四步，协程A恢复执行。
上面流程的协程A，就是异步任务，因为它分成两段（或多段）执行。这可以类比到平时的网络请求。

Generator就是一种在ES6的协程实现。它内部有yield关键字可以交出函数的执行权，等到需要执行时再执行。

## 异步应用
因为generator可以暂停执行和继续执行，所以它很适合封装异步任务。
假如有三次api请求，每一次都依赖上一次返回的结果，就可以用yield暂停执行。
```js
function* asyncTaskGenerator() {
    try {
        const result1 = yield mockRequest('https://api1.com');
        console.log(result1);
        const result2 = yield mockRequest('https://api2.com');
        console.log(result2);
        const result3 = yield mockRequest('https://api3.com');
        console.log(result3);
        return [result1, result2, result3];
    } catch (error) {
        console.error('Error:', error);
    }
}
```
## 自动流程管理
但是这段代码不能做到自动流程管理，倘若有非常多个api请求写的代码就比较冗余。所以你可以使用Thunk函数或Co模块进行自动流程管理。

## async/await的出现
async/await的出现是为了简化Promise的使用，其实它们是Generator的语法糖。
你可以看作将*改为async，将yield改为await。它的改进如下：
- 内置执行器
- await可以接受更多的类型比如原始类型值
- 返回值是Promise而不是Iterator
