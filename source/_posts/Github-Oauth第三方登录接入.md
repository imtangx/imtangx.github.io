---
title: Github-Oauth2.0第三方登录接入(authCode)
date: 2025-01-15 00:58:17
tags: Oauth
---
## 第三方登录
用户想要登录A网站，A网站需要获取用户某个第三方平台的信息（头像、昵称等）。所以用户需要“登录”第三方平台才能获取到对应的信息，再把信息给到A网站。
## 为什么要用到Oauth?
通过上面的需求，最简单的方法用户给予A网站第三方应用的账号密码，让A网站去登录再取得信息，这显然是不够安全的。\
所以有了Oauth授权，简单来说是：A网站请求授权，用户同意授权，然后第三方应用会生成一个令牌code，它的作用和密码类似，但是令牌权限有限，并且是有效期的，还可以随时被用户撤销，所以是一种很好的解决方案。\
而A网站要如何取到这个令牌，就有了以下的几种方式。
## Oauth2.0的四种方式：
- 授权码模式。这是最常用的一种方式，前端对第三方应用请求一个授权码code，第三方应用将授权码code发到后端，然后后端用这个授权码code去请求一个用户令牌token，保证通信交换都在后端完成，令牌对用户不可见。
- 简化模式。比上面少了授权码的步骤，在浏览器中直接申请认证令牌，令牌对用户是可见的。
- 密码式。直接把第三方账号密码交给A网站去登录，只建立在用户极度信任网站的前提下。
- 凭证式（命令行）。适用于没有前端的命令行应用，即在命令行下请求令牌。
  
不管是上面的哪种方式，都需要到对应的第三方备案，取得两个身份识别码`client ID`和`client secret`用于说明自己的身份，防止令牌滥用。注意`client secret`是必须保密的，会放在后端。

由于Github Oauth文档明确指出Web应用只能使用授权码模式。![alt text](/images/github-oauth-mode.jpg)

### 授权码模式工作流程
- 用户访问客户端，后者将前者导向认证服务器。
- 用户选择是否给予客户端授权。
- 假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个授权码。
- 客户端收到授权码，附上早先的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。
- 认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）。

## Github Oauth接入Github登录(authCode)实例
### 在Github注册一个Oauth应用
像上面说的，你需要到对应的第三方备案，在[这里](https://github.com/settings/applications/new)填好对应信息。
- `Application name`: 你的网站应用名称
- `Homepage URL`: 调用接口的网址。本地开发可以设为`http://localhost:8080`。
- `Authorization callback URL`: 回调地址。就是获得授权之后会重定向到的地址。本地开发可以设为`http://localhost:8080/auth/github/callback`。

提交之后，你就会获得属于这个Oauth应用的`client ID`和`client secret`。
### 如何获得授权码
向第三方取得授权码code，需要在`https://github.com/login/oauth/authorize`后加上查询参数，来让第三方知道你的身份及你要怎么申请。
- `client ID`: 必填。用于识别你对应的Oauth应用。
- `redirect_uri`: 获得授权码后的回调地址。它可以是`Authorization callback URL`的子路径。
- `scope`: 权限控制。即A网站可以获得用户在Github上的权限大小。
- `response_type`: 后跟`=code`。代表现在以授权码方式请求。

### 如何获得令牌
获得授权码之后，重定向到`redirect_uri`。URL后会跟上`?code=xxx`就是你的授权码。然后要对`https://github.com/login/oauth/access_token`发出POST请求令牌。
- `client ID`: 上文一致。
- `client secret`: 上文一致。
- `code`: 获得的授权码。
- `redirect_uri`: 用于双重验证。

### 用令牌获取信息
你已经取得了令牌。现在对`https://api.github.com/user`发出GET请求，需要在请求头带上你刚刚取得的令牌accessToken。
```ts
  headers: {
    Authorization: `Bearer ${accessToken}`,
    Accept: 'application/json',
  },
```

参考文档：
- [https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
- [https://docs.github.com/zh/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps](https://docs.github.com/zh/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)