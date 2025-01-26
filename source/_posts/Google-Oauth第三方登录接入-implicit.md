---
title: Google-Oauth2.0第三方登录接入(implicit)
date: 2025-01-25 15:48:28
tags: Oauth
---
## 前言：`Oauth2.0`的四种方式
- 授权码模式。这是最常用的一种方式，前端对第三方应用请求一个授权码`code`，第三方应用将授权码`code`发到后端，然后后端用这个授权码`code`去请求一个用户令牌`token`，保证通信交换都在后端完成，令牌对用户不可见。
- 简化模式。比上面少了授权码的步骤，在浏览器中直接申请认证`token`，令牌对用户是可见的。
- 密码式。直接把第三方账号密码交给A网站去登录，只建立在用户极度信任网站的前提下。
- 凭证式（命令行）。适用于没有前端的命令行应用，即在命令行下请求`token`。
`Google Oauth`文档明确指出`Web`应用只能使用`token`类型。也就是简化模式(`implicit`)。\
![alt text](/images/google-oauth-mode.jpg)

### 简化模式工作步骤
- 用户访问客户端，后者将前者导向认证服务器。
- 用户选择是否给予客户端授权。
- 假设用户给予授权，认证服务器将用户导向客户端指定的"重定向`URI`"，并在`URI`的`Hash`部分包含了访问`token`。
- 浏览器向资源服务器发出请求，其中不包括上一步收到的`Hash`值。
- 资源服务器返回一个网页，其中包含的代码可以获取`Hash`值中的`token`。
- 浏览器执行上一步获得的脚本，提取出`token`。
- 浏览器将`token`发给客户端。

## Google `Oauth`接入Google登录(`implicit`)实例
### 开启相关`GoogleAPI`接口
我这里是读取`google`的个人资料，`api`接口为`https://people.googleapis.com/v1/people/me`。需要在[这里](https://console.cloud.google.com/apis/library)找到`Google People API`启用。

### 创建`Oauth`应用
- 创建新`project`
- 配置`Oauth`同意屏幕
- 配置`Oauth`凭据
- 拿到`Clinet_id`和`Client_secret`

注意，该模式主要用于浏览器或移动端等无法安全存储 `Client_secret` 的场景，因此不要求客户端提供`Client_secret`。

### 重定向至`Oauth`授权服务器
授权`URL`为`https://accounts.google.com/o/oauth2/v2/auth`。需要跟上一些查询参数。
- `client ID`: 用于识别你对应的`Oauth`应用。
- `redirect_uri`: 用户同意授权后跳向的重定向`URL`。
- `scope`: 权限控制。即A网站可以获得用户在`Google`上的权限大小。
- `response_type`: 后跟`=token`。代表现在以`token`方式请求。

### 获得`token`
当用户同意授权之后，会在回调`URL`附上`hash`片段。以#分隔，后跟若干键值对包含`access_token`，需要对`URL`进行处理。与授权码模式不同的是，`hash`片段无法通过请求发到后端，所以这里的做法是：
- 当用户到达回调`URL`时，后端将其重定向至一个新的网页用于在前端处理`hash`片段。
- 重定向后，`hash`片段不会消失，比较省事的方法是重定向到初始登录页面上。
- 在初始登录页面的`js`逻辑中同时写上登录功能和读取`access_token`功能。

```js
  const fragment = window.location.hash.substring(1);
  const params = new URLSearchParams(fragment);
  const accessToken = params.get('access_token');
```

### 用`token`获取信息
然后向指定的`API`接口发送`token`，注意这里还需要跟上`personFields`参数，代表你需要哪些资料。
```js
  fetch('https://people.googleapis.com/v1/people/me?personFields=names,emailAddresses,photos', {
    headers: {
      Authorization: `Bearer ${accessToken}`,
    },
  })
```

参考文档：
- [https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
- [https://developers.google.com/identity/protocols/oauth2/javascript-implicit-flow?hl=zh-cn](https://developers.google.com/identity/protocols/oauth2/javascript-implicit-flow?hl=zh-cn)
