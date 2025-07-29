---
title: Chrome浏览器运行机制
date: 2025-07-30 02:05:24
tags: ['浏览器']
---

## 进程和线程

当启用一个应用时，操作系统会为其创建一个进程以及私有的内存空间。这片空间用来存储所有程序相关的数据的状态，当应用关闭，这片空间也随之释放。而线程是跑在进程里面的。

有时候为了满足某些功能的需要，当前进程会请求系统创建另外的进程去处理其他任务，这些进程是互相独立的，只能通过 IPC 通信。很多应用程序都会采用多进程的方式来工作，因为这些进程互相独立不影响。所以当一个进程崩溃，并不会使整个程序崩溃。

## 浏览器架构

浏览器的运作当然也需要进程和线程来工作。一种是单进程架构，进程内有若干个线程工作；一种是多进程架构，每个进程都有若干个线程工作，不同进程通过 IPC 通信。

以 Chrome 举例，它就是多进程架构：

![](https://developer.chrome.com/static/blog/inside-browser-part1/image/browser-architecture-998609758999a_1920.png?hl=zh-cn)

这里的多进程体现在渲染进程(Renderer Process)，浏览器会为每个标签页都分配一个单独的渲染进程，接下来把目光移到单个标签页时：

![](https://developer.chrome.com/static/blog/inside-browser-part1/image/chrome-processes-79aaecca78d23_1920.png?hl=zh-cn)

| Browser Process(浏览器进程) | 浏览器的核心进程。控制地址栏、书签、返回、前进按钮，以及网络请求和文件访问。 |
| --------------------------- | ---------------------------------------------------------------------------- |
| GPU Process(GPU 进程)       | 隔离地处理 GPU 任务。                                                        |
| Renderer Process(渲染进程)  | 控制网页的显示内容。                                                         |
| Plugin Process(插件进程)    | 控制网站使用的插件。                                                         |
| Extension Process(扩展进程) | 控制网站使用的扩展。                                                         |

## Chrome 使用多进程架构的好处

就像上面说的，每个进程是独立互不影响的，即使一个进程崩溃了也不会影响到别的进程。映射到浏览器的行为，每个标签页分配的进程是独立互不影响的，当一个标签页崩溃卡死时，你只需要关闭那个标签页，而其他标签页不会受到影响。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image.png)

## Chrome 是如何工作的

从最经典的场景、面试题说起，用户在地址栏输入了一串 URL，直到页面内容呈现，发生了哪些事情？

## 从浏览器进程开始说起

首先地址栏是受浏览器进程控制的，那么浏览器进程内部又分配了不同的线程来完成相应的工作。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%201.png)

可以看到浏览器进程内部有网络进程、存储(文件访问)线程、UI 线程。很明显地址栏是受 UI 线程所控制的，所以它会处理用户的输入。

## 一次简单的导航

### 1. 处理输入

回忆一下生活中的场景，当你需要 Google 某个关键字时，在地址栏直接输入关键字并按下回车，浏览器会识别 Google 搜索结果的页面；而当你输入某个网址 xxx.com 时，浏览器会直接把你带到那个网站。

这件事就是 UI 线程需要做的，它需要分辨用户是输入了 URL 还是关键字。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%202.png)

### 2. 开始导航

无论第 1 步的结果如何，按下回车后，UI 线程会唤起网络线程对输入内容进行请求，这时在地址栏上就会出现我们每天都能看到的 Loading 圆圈动画。

而在请求的过程中，网络线程还会处理一些额外的情况，比如重定向。此时就会与 UI 线程再次通信，并换一个新的网址发起网络请求。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%203.png)

### 3. 处理回复

当网络请求结束后，会受到响应正文。浏览器会先读取响应头中的 Content-Type 来判断返回类型。

如果是一个 HTML 文件，就准备把数据丢给渲染进程；而如果单纯是一个文件，则会把数据丢给下载管理器。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%204.png)

### 4. 查找渲染程序进程

当网络线程处理完回复后，就会告知 UI 线程。然后 UI 线程开始查找一个渲染进程来渲染这个网页。

这里有一个优化点，网络请求的时间一般在几百毫秒内。在这段时间里，因为网址已经确定，UI 线程可以先去查找一个渲染进程待命，如果发生网站重定向再开一个新的渲染进程，否则就可以直接使用。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%205.png)

### 5. 提交导航

到现在渲染进程已经可以开始工作，等待它渲染完成后，地址栏的 Loading 圆圈动画就会停止，代表浏览器进入了一个新的网页，同时也会留下历史记录。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%206.png)

## 渲染进程是如何工作的

### 解析

渲染进程开始接受 HTML 数据，并将 HTML 解析成 DOM 树，对错误语法的 HTML 会有一些错误处理，保证你的 HTML 不会抛出错误。

在解析 HTML 时，对于图片(img 标签)、CSS(link 标签)这些资源，渲染进程会额外开启一个预解析线程，发送单独的网络请求去请求资源，这个过程是并行的，并且对于 link 标签还有一个”rel=preload”的属性，可以更近一步地提前关键资源的加载时机。

对于 JavaScript(script 标签)资源，它会暂停 HTML 的解析，转而去加载、执行这些 JavaScript 代码，这是因为 JavaScript 中可能对 DOM 有所读写。你可能会想到：

- 我的 JavaScript 代码如果没有对 DOM 的读写逻辑，我希望我可以异步地去加载、执行代码。
- 我的 JavaScript 代码中有对 DOM 的读写逻辑，但是我希望先加载，等到合适时机再执行。

> script 标签的 defer 和 async 属性就是用来完成这件事情的。
> defer 属性：异步加载完毕后，等待 HTML 解析完毕，DOMContentLoaded 之前执行，并且可以保证多个 defer 标签的执行顺序。
> async 属性：异步加载完毕后立即执行并阻塞 HTML，不能保证多个 async 标签的执行顺序。
>
> ![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%207.png)

### 构建渲染树

刚刚说了 CSS 资源的解析是并行的，它会生成一棵 CSSOM 树给渲染进程，而渲染树是由 DOM 树和 CSSOM 树合并而来的，所以在这里需要经历一个递归的过程，从根节点开始将对应的 CSSOM 节点合并到 DOM 节点中。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%208.png)

### 构建布局树

上一步的渲染树只是存储了 DOM 的结构和样式，但还没有确定每个元素的坐标和大小，所以浏览器需要为每一个元素计算出布局信息，也就是一棵布局树。

这棵布局树与渲染树不同的是，如果有一个元素应用了 display:none，那么它将不会出现在布局树上。但是如果一个元素是伪元素，或是应用了 visibility:hidden，它会出现在布局树上。

> 设置了 display:none 的元素，它真正意义的从文档流移除，不会占据任何空间也不会影响其他元素的布局。
> 设置了 visibility:hidden 的元素，它只是对用户的交互无影响(看不到、点不着)，但它会保留占位，因此可能会影响到其他元素的布局。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%209.png)

### 绘制

现在的 DOM 又多了布局信息，但还不足以呈现网页。因为网页的每个元素是有可能互相覆盖的，也就是需要知道绘制它们的顺序，z-index 这个属性就是用来改变默认绘制顺序的。这也导致 HTML 不能简单地按文档顺序从上到下绘制元素。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2010.png)

因此在绘制过程中，需要遍历整棵布局树来创建一连串的绘制指令。例如：先绘制背景、再绘制文本…

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2011.png)

在渲染流水线上，每个步骤都会使用上一步操作的结果来创建新数据，例如如果布局树某个节点的布局信息产生了变化，则需要为受影响的部分重新生成布局信息和绘制指令，这个成本是很高的。

[https://developer.chrome.com/static/blog/inside-browser-part3/video/T4FyVKpzu4WKF1kBNvXepbi08t52/d7zOpwpNIXIoVnoZCtI9.mp4?hl=zh-cn](https://developer.chrome.com/static/blog/inside-browser-part3/video/T4FyVKpzu4WKF1kBNvXepbi08t52/d7zOpwpNIXIoVnoZCtI9.mp4?hl=zh-cn)

假如我们需要为元素添加动画效果，那么节点的布局信息就一定会改动，浏览器则需要在 1 帧内完成这些事情，如果在 1000ms/60fps = 16.6ms 中没能完成这次更新，就会跳过一些中间帧而造成卡顿。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2012.png)

另外当 JavaScript 代码运行占用主线程时，这些布局的计算也会被阻塞。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2013.png)

你可以将 JavaScript 操作划分成小块，使用 requestAnimationFrame()安排在每个帧中运行。或是使用 Web Worker 在另外的线程运行代码。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2014.png)

### 合成

现在，浏览器知道了 DOM 结构、样式、布局、绘制顺序，它终于要开始绘制页面了。将这些信息转换成屏幕上的像素的行为叫做光栅化。

也就是对视口内的部分进行光栅化处理，当用户滚动网页时，移动已光栅化的帧，并光栅化新出现的部分。

[https://developer.chrome.com/static/blog/inside-browser-part3/video/T4FyVKpzu4WKF1kBNvXepbi08t52/AiIny83Lk4rTzsM8bxSn.mp4?hl=zh-cn](https://developer.chrome.com/static/blog/inside-browser-part3/video/T4FyVKpzu4WKF1kBNvXepbi08t52/AiIny83Lk4rTzsM8bxSn.mp4?hl=zh-cn)

现代的浏览器会采用一种更先进的方式叫做合成。合成的原理是将网页的各个部分拆分成不同的图层，并且单独光栅化这些图层。

如果用户滚动页面，不需要再光栅化新的部分了，只需要用图层合成新帧即可。合成同样也可以应用在动画上，这样动画就可以跳过布局和重绘，只需要调整对应图层的变换矩阵，无需重新光栅化，效率 upup：

> 当动画使用 transform 时：这些位移(translate)、旋转(rotate)、缩放(scale)只是对元素的视觉呈现进行变换，并不改变布局树中的原始位置和尺寸。

[https://developer.chrome.com/static/blog/inside-browser-part3/video/T4FyVKpzu4WKF1kBNvXepbi08t52/Aggd8YLFPckZrBjEj74H.mp4?hl=zh-cn](https://developer.chrome.com/static/blog/inside-browser-part3/video/T4FyVKpzu4WKF1kBNvXepbi08t52/Aggd8YLFPckZrBjEj74H.mp4?hl=zh-cn)

### 分层

因此主线程会重新遍历布局树，对其创建一个新的层树。这样做的好处还有：当你对某个层的元素进行改动，只会对该层进行处理，效率 upup。

will-change 可以让你手动地操作分层策略，但是过多的层数也会影响合成过程的速度。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2015.png)

### 分块

在对每个图层的光栅化过程中，由于每个图层可能更大，合成线程会将其划分为多个图块，再发送到 GPU 进程进行光栅化。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2016.png)

### 显示

GPU 进程会开启多个线程来完成光栅化，并且优先处理靠近视口的块。最后合成到屏幕上。

## 事件处理让合成线程头疼

### 非快速滚动区域

上面说到，对各个图层光栅化后可以流畅的滚动、合成、滚动、合成。但是如果页面被附加了一些事件监听，比如 JavaScript 想要监听元素与视口的相对位置进行懒加载，这时候就需要把事件发送到主线程。这块区域也被标记为“非快速滚动区域”。

### 编写事件处理程序需要注意

事件委托是一种基于事件冒泡机制的处理模式，例如你可以在 document 注册一个监听事件，根据事件目标来执行对应任务：

```jsx
document.body.addEventListener('touchstart', event => {
  if (event.target === area) {
    event.preventDefault();
  }
});
```

这个写法非常地省事，你只需要写一个监听函数就能处理所有的 touchstart，但这也意味这整个 body 都被标记为“非快速滚动区域”。

也就意味着每当有 touchstart 事件发生，整个页面都必须等待主线程通信，这就失去了合成器流畅滚动的意义。

![image.png](Chrome%E6%B5%8F%E8%A7%88%E5%99%A8%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%201e080e8eea258048b176c2a07510a02a/image%2017.png)

你可以在监听函数中传入 passive:true，表示浏览器即希望在主线程监听事件，合成器也可以继续合成新帧。

```jsx
document.body.addEventListener(
  'touchstart',
  event => {
    if (event.target === area) {
      event.preventDefault();
    }
  },
  { passive: true },
);
```

### 查找事件的目标对象

当合成线程往主线程发送事件时，主线程首先需要通过命中测试去找到事件的目标对象。具体的来说，合成线程会发送触发事件的 x、y 坐标位置，主线程通过遍历绘制指令/布局树来确定处于 x、y 坐标的是哪个对象。

---

参考文章：
[深入了解现代网络浏览器](https://developer.chrome.com/blog/inside-browser-part1/)
