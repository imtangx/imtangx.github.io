---
title: websocket心跳检测与重连机制的实现 - chatX
date: 2025-03-01 00:39:13
tags: [Websocket]
---
## 什么是Websocket
一种处于应用层的基于TCP的网络传输协议，支持在1个TCP连接上全双工通信，达到实时通信的效果。
- 一次握手之后即可双向数据传输。
- 采用二进制帧结构但与HTTP2.0不兼容。不支持多路复用等特性。
- 协议名为ws或wss，对应http和https的关系，且默认端口也一致。
- 握手过程需要设定`Upgrade: websocket`字样，1次HTTP请求就可以从HTTP协议升级至Websocket协议。

## 为什么需要心跳检测？是什么？
虽然Websocket支持长连接，但是并不能保证握手成功后就永远畅通，可能会发生若干情况导致未触发close函数就失去连接，包括但不限于：

- 浏览器端断网了
- 服务器还未发送关闭帧就关闭了连接

这在即时通讯的场景是很致命的，可能导致A看到自己成功发送了消息给B但其实服务器并没有收到并及时转交给B。

## 心跳检测的代码实现
这里指的是客户端发起的心跳检测，即握手成功后（触发open函数）就进行周期性的ping，看服务器能否在一段时间后应答，如果未收到应答代表连接实际已经断开，手动执行close函数关闭连接，避免资源浪费。

执行流程：
- 触发open函数，开始执行心跳检测
- 发送ping到服务器，并把心跳状态设为等待中，并设定一个setTimeout
- 服务器如果收到ping，就发送回应消息给客户端
- 客户端如果收到回应消息，就把心跳状态设为已收到应答
- setTimeout到期后，查看心跳状态，如果还在等待中，表示连接断开，执行close函数
  - 此时可以提醒用户断连，然后在代码中进行新的websocket连接。
- 如果是已收到应答，本次检测成功，继续下一次心跳检测，保证周期性

以下是经过处理简化的伪代码，不能直接运行：
```jsx
  heartbeatStatus: 'waiting' | 'received';

  socket.onopen = event => {
    startHeartbeat();
  };

  startHeartbeat: () => {
    set({ heartbeatStatus: 'waiting' });
    get().sendHeartbeat(); /** 发送心跳检测消息 */
    setTimeout(() => {
      if (get().heartbeatStatus === 'received') {
        /** 下一轮心跳检测 */
        get().startHeartbeat();
      } else {
        /** 手动关闭连接 */
        socket.close();
      }
    }, 3000);
  },
```

## 心跳检测优化
可以看到上面的逻辑其实非常简单，但实际生产环境中我们还需要考虑更多东西。
- 单次心跳检测失败，可能是网络暂时波动，直接关闭连接/重连会造成用户体验不佳
  - 解决方法：当发现心跳检测失败时，多发起两次心跳检测，如果还是没有心跳再提示用户掉线。提高用户体验，减轻服务器压力。
- 点触发close函数并重连时，需要区分是被动触发close还是用户手动触发，如果是手动触发就不执行重连。
- 当心跳检测频繁失败，网络不稳定时，考虑降级服务到HTTP长、短轮询等服务。
- 重连时可以使用指数退避重连策略，逐渐加长重连间隔，减少服务器压力。

在chatX的具体实现，存储在Zustand状态管理中，源码查阅：[wsStore](https://github.com/imtangx/chatX/blob/master/packages/chat-fe/src/store/wsStore.ts)。

最后，如果发现本篇博客有遗漏 ｜ 错误的地方，欢迎指出我会更正。