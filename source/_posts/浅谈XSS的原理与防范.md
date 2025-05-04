---
title: 浅谈XSS的原理与防范
date: 2025-05-03 13:58:53
tag: ['浏览器安全']
---
## 什么是XSS
Cross-Site Scripting（跨站脚本攻击）简称XSS。本质是将恶意代码/脚本注入目标网站，从而读取用户的敏感信息（如cookie，sessionId）危害数据安全。
## XSS常见场景
例如，我们有时候需要从URL取参数在页面上展示。
```html
<body>
  用户搜索的关键词为：<p></p>
  <script>
    const params = new URLSearchParams(location.search);
    const keyword = params.get('keyword');
    const pDom = document.querySelector('p');
    pDom.innerHTML = keyword; 
  </script>
</body>
```
如果这时访问的URL查询参数为:`keyword=<img src="x" onerror="alert(1)">`。 

网站会尝试从一个不存在的地方下载img，然后触发`onerror`之后调用`alert(1)`，如果换成读取用户的cookie之类敏感消息，就会造成危险。

## XSS分类
### 存储型XSS
- 攻击者把恶意代码提交到目标网站的数据库。
- 服务端把恶意代码从数据库取出返回给客户端（如拼接URL参数）。
- 客户端解析代码中的恶意代码并执行。
- 恶意代码可能将用户的敏感信息取出并发送到自己的网站。
### 反射型XSS
- 攻击者构造出一个恶意URL诱导用户点击。
- 打开URL后，服务器取出恶意代码并返回给客户端。
- 客户端解析代码中的恶意代码并执行。
- 恶意代码可能将用户的敏感信息取出并发送到自己的网站。

**存储型XSS和反射型XSS的区别在于：前者是将恶意代码注入数据库，后者是存储在URL中。**
### DOM型XSS
- 攻击者构造出一个恶意URL诱导用户点击。
- 打开URL后，前端JS读取URL参数中的恶意代码并执行。
- 恶意代码可能将用户的敏感信息取出并发送到自己的网站。

**DOM型XSS和前两者的区别在于：其被攻击的过程由浏览器（前端JS）完成，不涉及服务端。**
## XSS预防
我们可以从XSS攻击的两个要素来预防。

1. 攻击者提交恶意代码。
2. 浏览器执行恶意代码。

### 输入过滤/转义
比如前端对用户的输入内容进行HTML转义后再发送给服务端，是否可行？\
答案是不可行。攻击者可以构造请求做到跳过前端的过滤。\
那么换个角度，我们在服务端进行代码的转义，然后把安全的代码存储到数据库，之后做到将安全的代码返回给前端，是否可行？\
这种做法有两种潜在的问题：
1. 转义后的代码有可能被输出到浏览器前端/JS客户端执行。浏览器会对转义字符进行编码，而JS不会。
2. 转义的代码如果拼接在HTML中可以编码后正常运行，但是如果返回后赋值给JS变量也无法编码。

### 小心的拼接HTML
如果必须要有拼接HTML的操作，可以使用合适的转义库。或尽可能使用`.innerText`或`.setAttribute()`之类的API进行赋值拼接，减少使用`.innerHTML`此类操作。

### Content Security Policy（CSP）
有时候注入的恶意脚本是某个外部src的script脚本，这时候我们可以配置CSP白名单来阻止未知来源的的script资源加载。\
你可以在HTTP头或是meta标签进行配置。这里简单介绍一下HTTP头如何配置。
```
Content-Security-Policy: default-src 'self'
```
上面代码表示所有的资源都只能从当前域名加载。\
还有常见的场景是对script脚本进行单独配置，例如nonce值来发送一个token，当加载脚本时进行验证。
```html
Content-Security-Policy: script-src 'nonce-EDNnf03nceIOfn39fn3e9h3sdfa'

<script nonce=EDNnf03nceIOfn39fn3e9h3sdfa>
  // 可以执行
</script>
```

### 其他安全措施
- HTTP-only：对cookie设置HTTP-only属性，防止在XSS攻击后直接读取cookie。
- 验证码：在进行操作时需要读取屏幕的验证码输入验证，辨别人类与机器。

---
参考文章：

[美团 - 前端安全系列（一）：如何防止XSS攻击？](https://tech.meituan.com/2018/09/27/fe-security.html)

[阮一峰 - Content Security Policy 入门教程](https://www.ruanyifeng.com/blog/2016/09/csp.html)