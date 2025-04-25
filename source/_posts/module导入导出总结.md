---
title: module导入导出总结
date: 2025-04-05 16:39:48
tags: [javascript]
---
由于本人对不同导出及相应的导入方式有时候会搞混，做下面的总结用于备忘录。

## Name命名导出

顾名思义就是导出时需要声明名字。因此在导入时也需要声明其名字。

```js
/** 1.js */
export const name = 'imtx';

/** 2.js */
import { name } from './1.js';
console.log(name); // 'imtx'
```



## Default默认导出

与上面的区别就是导入不需要声明名字。因此在导入时可以任意选择你想要的名字。

也是因为这个原因，一个文件最多存在一个Default默认导出。

```js
/** 1.js */
export default 'imtx';

/** 2.js */
import anyYourWantName from './1.js';
console.log(anyYourWantName); // 'imtx'
```



## 混合导出（Name、Default）

```js
/** 1.js */
export default 'imtx';
export const name = 'bubu';

/** 2.js */
import anyYourWantName, {name} from './1.js';
console.log(anyYourWantName, name); // 'imtx' 'bubu'
```



## 列表导出

其实也是命名导出，不过这次你可以使用`{}`一次性导出多个变量。

```js
/** 1.js */
const name1 = 'imtx';
const name2 = 'bubu';

export {
  name1,
  name2
}

/** 2.js */
import {name1, name2} from './1.js';
console.log(name1, name2); // 'imtx' 'bubu'
```



## 导出别名

你可以在命名导出时使用`as`给不满意的名字换个名字。

```js
/** 1.js */
const name = 'imtx';

export {
  name as newName
}

/** 2.js */
import { newName } from './1.js';
console.log(newName); // 'imtx'
console.log(name); // undefined
```



## 导入别名

同样的，你可以在导入的时候再给它换个名字。

```js
/** 1.js */
export const name = 'imtx';

/** 2.js */
import { name as newName } from './1.js';
console.log(newName); // 'imtx'
console.log(name); // undefined
```



## 全部导入

对于刚刚的混合导入，你需要区分默认导入与`{}`命名导入，有一种更加统一的方法。

```js
/** 1.js */
export const name = 'value';
export default 'defaultValue'

/** 2.js */
import * as All from "./1.js";
console.log(All.default); // 'defaultValue'
console.log(All.name); // 'value'
```



