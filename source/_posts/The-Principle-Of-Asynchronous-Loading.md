---
title: 浅谈异步加载组件的原理
date: 2025-08-25 23:22:22
tags:
---
## 什么是异步加载组件
异步加载组件是指在需要的时候才加载组件。通常可以用于优化首屏加载时间，提高用户体验。其出现的原因有一点就是现在的应用都是SPA应用，会将所有代码打包到一个chunk中，导致解析代码体积过大。

在应用层面我们可以在合适的时机再去请求组件的代码，比如在路由切换的时候再去请求组件的代码。

## 技术实现
通常是通过ES的import()语法实现的。import()语法返回一个Promise对象，当Promise对象状态变为fulfilled时，就会返回一个模块对象。模块对象中包含了组件的代码。

而在框架层面，比如React，会提供一个React.lazy()方法，用于异步加载组件。其本质上是一个高阶组件，返回一个特殊的React组件。这个特殊的React组件会在需要渲染的时候，再去请求组件的代码。
```javascript
// React.lazy 的简化实现原理
function lazy(ctor) {
  return {
    $$typeof: REACT_LAZY_TYPE,
    _ctor: ctor,
    _status: -1, // -1: 未初始化, 0: 加载中, 1: 成功, 2: 失败
    _result: null,
    _init: function() {
      // 这是关键方法，用于初始化异步组件
      if (this._status === -1) {
        this._status = 0;
        this._result = ctor().then(
          module => {
            this._status = 1;
            this._result = module.default || module;
          },
          error => {
            this._status = 2;
            this._result = error;
          }
        );
      }
      return this._result;
    }
  };
}
```
可以看到，ctor这个参数是一个Promise，因此我们通常会这样调用。
```javascript
const MyComponent = React.lazy(() => import('./MyComponent'));
```

但生成单独chunk的过程还需要webpack参与，当webpack遇到动态import()语法时，会调用__webpack_require__.e()方法来将其拆分成一个单独的chunk。