---
title: 浅谈CSRF的原理与防范
date: 2025-03-18 17:46:44
tags: ['浏览器安全']
---
## 什么是CSRF
Cross-site request forgery（跨站请求伪造）简称CSRF。本质是攻击者诱导用户点击恶意网站并向后台发送请求，冒充用户已有的登录凭证（cookie）来做到获取数据。

上面运用了浏览器的特性做到冒充，常见流程如下：

1. 用户登录a.com，留下了登录后的cookie。
2. 攻击者在a.com发布了一个恶意链接b.com。
3. 用户好奇点击了恶意链接，恶意链接对a.com的服务端发出请求。
4. 由于浏览器的策略，对于已有cookie的网页会自动携带之前的cookie。
5. 服务端校验cookie通过，返回了攻击者想要的数据。

## CSRF常见场景
### GET类型的CSRF
一般是诱导用户点击某个链接。或是一个资源如（img标签）会自动对其链接发起一次GET请求，假设这个资源就是请求网站的服务端。服务端就会收到一次跨域请求。
### POST类型的CSRF
一般是使用一个自动提交的表单。
```html
<form action="http://bank.example/withdraw" method=POST>
    <input type="hidden" name="account" value="xiaoming" />
    <input type="hidden" name="amount" value="10000" />
    <input type="hidden" name="for" value="hacker" />
</form>
<script> document.forms[0].submit(); </script> 
```
## CSRF预防
要做到预防CSRF，首先需要了解它的要素：
- 攻击都发生在第三方网站，在自身网站无法做到配置预防。
- 攻击者只能做到携带cookie，而无法知道cookie的值。
- 浏览器访问资源时会自动携带之前的cookie。

### 使用Origin Header确定来源域名
### 使用Referer Header确定来源域名
**由于一些因素，当Origin和Referer都不存在的时候，就无法做到验证。**
### CSRF token
我们可以要求用户携带一个攻击者无法获取到的token，这个token不放在cookie而一般放在localstorage这种不会随着请求自动携带的地方。

常见的做法是：登录网站后服务端生成一个Token发送给客户端，客户端在每次请求的时候需要携带这个token去让服务端校验，服务端可以设置一个中间拦截器对token进行检验。

这里又有两种做法：
1. token随机生成并存在服务器的session中检验，但是可能会对服务器造成比较大的存储压力。
2. 无状态Token。运用统一的算法和用户信息生成唯一的token，后端只需要再一次计算比对，无需存储在服务器本地。常见的做法是JWT。

### 双重cookie
上面说到攻击者无法知道具体的cookie，所以我们可以让服务端生成额外的cookie发送给客户端，客户端在请求携带这个cookie。

### Samesite Cookie
为了从源头上解决这个问题，Google起草了一份草案来改进HTTP协议，那就是为Set-Cookie响应头新增Samesite属性，它用来标明这个 Cookie是个“同站 Cookie”。
- Samesite=Strict：严格模式。表明这个cookie一定不会被第三方携带。
- Samesite=Lax：宽松模式。当用户从 a.com 点击链接进入 b.com 时，b.com的Lax-cookie会被携带。而其他场景（异步请求、post提交表单）不会携带。

### 我们应该如何使用SamesiteCookie
如果SamesiteCookie被设置为Strict，浏览器在任何跨域请求中都不会携带Cookie，新标签重新打开也不携带，所以说CSRF攻击基本没有机会。

但是跳转子域名或者是新标签重新打开刚登陆的网站，之前的Cookie都不会存在。尤其是有登录的网站，那么我们新打开一个标签进入，或者跳转到子域名的网站，都需要重新登录。对于用户来讲，可能体验不会很好。

如果SamesiteCookie被设置为Lax，那么其他网站通过页面跳转过来的时候可以使用Cookie，可以保障外域连接打开页面时用户的登录状态。但相应的，其安全性也比较低。

另外一个问题是Samesite的兼容性不是很好，现阶段除了从新版Chrome和Firefox支持以外，Safari以及iOS Safari都还不支持，现阶段看来暂时还不能普及。

而且，SamesiteCookie目前有一个致命的缺陷：不支持子域。例如，种在topic.a.com下的Cookie，并不能使用a.com下种植的SamesiteCookie。这就导致了当我们网站有多个子域名时，不能使用SamesiteCookie在主域名存储用户登录信息。每个子域名都需要用户重新登录一次。

---
参考文章：
[美团 - 前端安全系列（二）：如何防止CSRF攻击？](https://tech.meituan.com/2018/10/11/fe-security-csrf.html)