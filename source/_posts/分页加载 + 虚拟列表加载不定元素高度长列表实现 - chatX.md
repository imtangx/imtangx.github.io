---
title: 分页加载 + 虚拟列表加载不定元素高度长列表实现 - chatX
date: 2025-02-28 21:08:26
tags: [React]
---
## 项目场景
最近在开发一款聊天项目，其中需要加载用户之间的消息列表，正常逻辑是从后端-数据库获取对应消息并渲染，但是如果消息过多会使得网络请求压力大，页面DOM渲染量大，导致性能不佳，所以了解到这两种优化方式。

## 什么是分页加载
分页加载是一种按需获取数据的策略，只在需要的时候加载数据，减少初次加载时间和内存占用。

在消息列表中，由于用户视窗内只能看到少数消息，我们可以一次只加载15条消息，当用户需要查看更多（上拉）时再从后端获取一次消息。

### 实现逻辑
我们在获取消息的时候可以传入lastId和limit，分别代表最后获取的一条消息id（如果按id递增）和消息限制。这里分页的逻辑更简单的是使用offset跳过前xx条消息，属于SQL语言范畴。然后后端可以通过数据量是否 < limit来判断消息是否全获取完了，给前端返回一个结果。

## 什么是虚拟列表
虚拟列表是一种只渲染可见区域内元素的优化技术，避免渲染所有列表项导致页面内DOM数量过多，提高渲染性能，提高滚动流畅性。

在消息列表中，我们只需要让用户看到视窗内的消息即可，当滚动时，实时渲染视窗内元素即可。

### 实现逻辑
对于一个存在滚动条的容器，scrollTop属性是滚动条距离容器顶部的距离。那么我们就只需要把元素放在距离顶部[scrollTop, scrollTop + windowHeight]的范围内，它就会被渲染出来。

所以我们需要知道哪些元素应该被放在这个范围内，假设每个元素的height为50px, scrollTop为100px以及windowHeight为100px。那么第3，4条消息就应该被渲染出来。所以我们需要知道每条消息相对于外部容器顶部的高度，需要设定外部容器为position: relative，内部元素为position: absolute，计算内部元素的top属性来使元素渲染在正确的位置，这是比较简单常用的方法。

还有一种方法是获取元素和外部容器相对于全局的距离getBoundingClientRect()再相减，获取相对高度。

这里面还有分为元素固定高度为元素不定高度，固定高度只需要使用下标计算即可，不定元素需要获取每项渲染后的高度实时计算渲染。

## 项目演示
我这里使用了react-infinite-scroll-component库的InfiniteScroll组件来实现下拉加载和分页逻辑，虚拟列表是手搓，也可以使用成熟的react-window库。

另外，正常列表是往下加载更多的消息，但聊天列表是需要从上面获取更多的消息，也就是下拉加载，恰好InfiniteScroll有这个属性，非常方便^^

### InfiniteScroll使用 & 分页逻辑
```jsx
    import InfiniteScroll from 'react-infinite-scroll-component';

    <InfiniteScroll
    dataLength={messages.length} // 数据数组长度
    next={loadMessages} // 触发加载后执行的函数 向后端请求新消息 只有hasMore = true时才会触发
    hasMore={hasMore} // 是否还有更多消息 
    loader={<h4>Loading...</h4>} // 当hasMore = true 且触发加载函数时 展示的内容
    scrollableTarget='container' // 外部容器id
    inverse={true} // 变为上拉逻辑
    style={{ display: 'flex', flexDirection: 'column-reverse' }} // 列表反向加载 搭配inverse
    endMessage={<h4>没有更多消息啦！</h4>} // 当hasMore = false 且触发加载函数的时候 展示的内容
  >
```
还有一些属性可以在官方文档查看，这里没有用到。

```js
  // fe
  const lastMessageId = useRef<number | null>(null); // 上次返回的最后一次id
  const pageSize = 15; // 每次分页获取的数量

  // be
  if (last_message_id) {
    query += ` AND id < ?`;
    params = [...params, last_message_id];
  }

  query += `
    ORDER BY id DESC
    LIMIT ?
  `;
  params = [...params, page_size];
```
这是分页的部分代码逻辑。

所以有了InfiniteScroll组件就可以很容易的实现上/下拉加载结合分页，如果手写的话，需要监听scrollTop到一定阈值触发加载函数，代码会比较麻烦。

### 手写不定元素高度虚拟列表
```tsx
  const [visibleMessages, setVisibleMessages] = useState<WebSocketMessage[]>([]); // 需要渲染的元素
  const itemHeight = 70; // 每个元素的默认高度 渲染时会替换为实际高度
  const totalHeight = useRef<number>(0); // 外部容器总高度 因为元素高度不定 需要实时更新
  const heightCache = useRef<{ [id: number]: number }>({}); // 高度缓存 存储每个元素ref的实际DOM高度
  const itemOffset = useRef<{ [id: number]: number }>({}); // scrollTop缓存 用于设置元素的top值

  /**
   * 更新单个元素的高度 然后更新所有元素的Offset
   */
  const updateHeight = (id: number, height: number) => {
    if (heightCache.current[id] === height) return;
    heightCache.current[id] = height;

    let prefix = 0;
    for (let i = messages.length - 1; i >= 0; i--) { // 反向 因为消息从下至上
      const msgid = messages[i].id;
      itemOffset.current[msgid] = prefix;
      prefix += heightCache.current[msgid] || itemHeight;
    }

    totalHeight.current = prefix;
    calcVisibleMessages(); /** 更新完高度后需要重新计算 & 渲染可视列表 */
  };

  /**
   * 计算视窗内应该有的元素
   */
  const calcVisibleMessages = () => {
    if (!containerRef.current) return;
    const scrollTop = -containerRef.current.scrollTop; // 负数 因为列对齐是反向 顶部变为底部

    const containerHeight = containerRef.current.clientHeight || 500;
    let startIndex = 0,
      endIndex = 0,
      currentOffset = 0;

    /**
     * 前缀offset + 当前消息高度 小于 滚动条距离顶部高度
     */
    while (
      startIndex < messages.length - 1 &&
      currentOffset + (heightCache.current[messages[startIndex].id] || itemHeight) < scrollTop
    ) {
      currentOffset += heightCache.current[messages[startIndex].id] || 0;
      startIndex++;
    }

    /**
     * 前缀offset 小于滚动条距离顶部高度 + 窗口高度
     */
    endIndex = startIndex;
    while (endIndex < messages.length && currentOffset < scrollTop + containerHeight) {
      currentOffset += heightCache.current[messages[endIndex].id] || 0;
      endIndex++;
    }
    
    /**
     * 多渲染两个头尾 避免白屏观感不好
     */
    startIndex = Math.max(0, startIndex - 2);
    endIndex = Math.min(messages.length - 1, endIndex + 2);

    setVisibleMessages(messages.slice(startIndex, endIndex + 1));
  };

  /**
   * 渲染出上面计算出的元素
   */
  const renderMessageItem = (message: WebSocketMessage): ReactNode => {
    const offset = itemOffset.current[message.id];
    return (
      <div
        key={message.id}
        /**
         * ref里的函数会在组件挂载 ｜ 更新时执行 获取它的高度并执行更新函数
         */
        ref={el => {
          if (!el) return;
          // 测量实际高度
          const height = el.getBoundingClientRect()?.height;
          updateHeight(message.id, height);
        }}
        style={{
          position: 'absolute',
          top: `${offset}px`,
          width: '100%',
        }}
      >
        <MessageItem message={message.text!} isSelf={message.sender === username} timestamp={message.timestamp} />
      </div>
    );
  };
```

上面是主体逻辑代码，在实际写的时候还需要注意它们的执行时机以及更新顺序，避免因为useState的异步更新机制读取到旧值，可以使用useRef避免，但不要过度使用。

具体代码查阅：[MessageList源码](https://github.com/imtangx/chatX/blob/master/packages/chat-fe/src/components/Chat/MessageList.tsx)。

最后，如果发现本篇博客有遗漏 ｜ 错误的地方，欢迎指出我会更正。