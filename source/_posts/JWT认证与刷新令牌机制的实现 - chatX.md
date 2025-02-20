---
title: JWT认证与刷新令牌机制的实现 - chatX
date: 2025-02-20 21:05:16
tags: [JWT]
---
## 什么是JWT
JWT (JSON Web Token) 是一种开放标准，用于在各方之间安全地传输信息。token 由三部分组成：

- Header（头部）：主要声明使用的算法。
- Payload（负载）：消息体，存放实际的内容，也就是Token的数据声明，例如用户的id。
- Signature（签名）：是对头部和载荷内容进行签名，设置一个secretKey，对前两个的结果进行HMACSHA25算法。保证一旦前面两部分数据被篡改，只要服务器加密用的密钥没有泄露，得到的签名肯定和之前的签名不一致。

## JWT的用处
在目前前后端分离的开发过程中，使用token鉴权机制用于身份验证是最常见的方案，流程如下：
- 服务器当验证用户账号和密码正确的时候，给用户颁发一个令牌，这个令牌作为后续用户访问一些接口的凭证。
- 后续访问会根据这个令牌判断用户是否有权限进行访问。

## 如何使用JWT
首先要借助第三方库`jsonwebtoken`。

### 生成token
通过`sign(payload, secretKey, option)`方法生成一个token：
- payload: 上面说的消息体。
- secretKey: 后端存储的密钥。
- option: 可以设置token过期时间。
```js
/** 设置一个15分钟有效期的token */
export const genToken = payload => {
  return jwt.sign(payload, secretKey, {
    expiresIn: '15m',
  });
};
```

在前端接收到token后，一般情况会通过localStorage进行缓存，然后将token放到HTTP请求头Authorization中，关于Authorization的设置，前面要加上Bearer，注意中间带有空格。

### 验证token
通过`verify(token, secretKey)`方法验证一个token：
- token: 需要验证的token。
- secretKey: 密钥用于解密验证。

### 生成RefreshToken
通常登录后会提供用户一个访问令牌和一个刷新令牌，如果访问令牌过期了就使用刷新令牌去重新生成一个访问令牌，这样做的好处是：
- 访问令牌短期有效 增加安全性
- 用户无需频繁登录
- 减少令牌被盗的风险

所以认证的逻辑是：如果访问令牌过期，查看刷新令牌是否有效，如果有效则使用它来再申请一块访问令牌，如果实效就让用户重新登录。
## JWT的优缺点
优点：
- json具有通用性，所以可以跨语言
- 组成简单，字节占用小，便于传输
- 服务端无需保存会话信息，很容易进行水平扩展
- 一处生成，多处使用，可以在分布式系统中，解决单点登录问题
- 可防护CSRF攻击

缺点：
- payload部分仅仅是进行简单编码，所以只能用于存储逻辑必需的非敏感信息
- 需要保护好加密密钥，一旦泄露后果不堪设想
- 为避免token被劫持，最好使用https协议

## 在chatX项目中的实现
本项目使用axios进行服务器请求。
### 一、在登录成功后发放token和refreshToken
```js
  const token = authUtils.genToken({ userId: user.id, username: user.username });
  const refreshToken = authUtils.genRefreshToken({ userId: user.id, username: user.username });

  // 登录成功，返回用户信息和 JWT
  res.status(StatusCodes.OK).json({
    message: '登录成功',
    user: {
      id: user.id,
      username: user.username,
      avatar: user.avatar,
    },
    token,
    refreshToken,
  });
```

### 二、前端将两个令牌存储在localstorage

### 三、前端创建全局axios拦截器附带token
```js
  /**
   * 设置请求拦截器
   */
  axios.interceptors.request.use(
    config => {
      const token = localStorage.getItem('authToken');
      if (token) {
        config.headers['Authorization'] = `Bearer ${token}`;
      }
      return config;
    },
    error => {
      return Promise.reject(error);
    }
  );
```

### 四、使用中间件对后端路由进行拦截验证token
```js
  //middleware
  export const authMiddleware = (req, res, next) => {
    const authHeader = req.headers.authorization;
    const token = authHeader && authHeader.split(' ')[1]; // Bearer Token

    if (!token) {
      return res.status(StatusCodes.UNAUTHORIZED).json({ message: '未提供Token，拒绝访问' });
    }

    const decoded = authUtils.verifyToken(token);

    if (!decoded) {
      return res.status(StatusCodes.UNAUTHORIZED).json({ message: 'Token无效或已过期' }); // 401 - Token 无效或过期
    }

    req.user = decoded;
    next(); // 跳到路由
  };

  //routes
  const router = express.Router();

  router.get('/', authMiddleware, getFriends);
  router.get('/requests', authMiddleware, getFriendRequests);
  router.post('/requests', authMiddleware, sendFriendRequests);
  router.patch('/requests/:requestId', authMiddleware, updateFriendRequestStatus);
```

### 五、当后端返回token过期时，执行刷新令牌机制
```js
  /**
   * 设置响应拦截器
   */
  const setupResponseInterceptor = () => {
    axios.interceptors.response.use(
      response => response,
      async error => {
        const originalRequest = error.config;

        // 处理401错误（未授权）
        if (error.response?.status === 401) {
          // 如果是刷新token的请求返回401，说明refresh token已过期 否则是token过期
          if (originalRequest.url === 'http://localhost:3001/auth/refresh') {
            console.log('Refresh token 已过期');
            handleLogout();
            return Promise.reject(error);
          }

          // 确保同一个请求不会重试多次
          if (!originalRequest._retry) {
            // 如果已经在刷新token，则将请求加入等待队列
            if (isRefreshing) {
              return new Promise((resolve, reject) => {
                failedQueue.push({ resolve, reject });
              })
                .then(token => {
                  // 用新token更新请求头
                  originalRequest.headers!['Authorization'] = `Bearer ${token}`;
                  // 重试请求
                  return axios(originalRequest);
                })
                .catch(err => Promise.reject(err));
            }

            // 标记该请求正在重试
            originalRequest._retry = true;
            isRefreshing = true;

            // 获取refresh token
            const refreshToken = localStorage.getItem('refreshToken');
            if (!refreshToken) {
              console.log('没有 refresh token');
              handleLogout();
              return Promise.reject(error);
            }

            try {
              // 尝试刷新token
              const response = await axios.post('http://localhost:3001/auth/refresh', {
                refreshToken: refreshToken,
              });

              // 保存新的access token
              const newToken = response.data.token;
              localStorage.setItem('authToken', newToken);

              // 更新当前请求的token
              originalRequest.headers!['Authorization'] = `Bearer ${newToken}`;

              // 处理队列中的请求
              processQueue(null, newToken);
              isRefreshing = false;

              // 重试当前请求
              return axios(originalRequest);
            } catch (err) {
              // 刷新token失败
              processQueue(err, null);
              isRefreshing = false;
              handleLogout();
              return Promise.reject(err);
            }
          }
        }
        return Promise.reject(error);
      }
    );
  };
```

### 刷新令牌变量功能解释列表
- **`isRefreshing`**:
    - 类型：`boolean`
    - 功能：这是一个标志位，用于**跟踪当前是否正在进行刷新令牌的请求**。
    - 作用：
        - 当值为 `true` 时，表示刷新令牌请求正在进行中。
        - 当值为 `false` 时，表示当前没有刷新令牌请求正在进行。
        - 主要用于**防止在短时间内并发收到多个 401 错误时，重复发起多个刷新令牌请求**，确保同一时刻只有一个刷新令牌请求在处理。

- **`failedQueue`**:
    - 类型：`QueueItem[]` (接口 `QueueItem` 定义了 `resolve` 和 `reject` 属性)
    - 功能：这是一个**请求队列**，用于**存储因访问令牌过期（401 错误）而被拦截，但尚未重试的请求**。
    - 作用：
        - 当 `isRefreshing` 为 `true` 时，后续收到的 401 错误请求会被添加到 `failedQueue` 队列中。
        - 这些请求会**等待刷新令牌过程完成后再进行重试**，确保在获得新令牌后，能够按顺序重新发送之前因令牌过期而失败的请求。

- **`processQueue` (函数)**:
    - 类型：`function(error: any, token: string | null = null)`
    - 功能：这是一个**处理请求队列的函数**，负责**遍历 `failedQueue` 队列，并根据刷新令牌的结果（成功或失败）来处理队列中的每个请求**。
    - 参数：
        - `error`:  如果刷新令牌**失败**，则传入一个错误对象，用于 `reject` 队列中等待的所有 Promise。
        - `token`: 如果刷新令牌**成功**，则传入新的访问令牌 `token`，用于 `resolve` 队列中等待的所有 Promise。
    - 作用：
        - 当刷新令牌成功时，使用新的令牌 `token` **解决（`resolve`）** 队列中每个请求对应的 Promise，使得等待的请求可以继续 `.then()` 链式调用并重试。
        - 当刷新令牌失败时，使用错误对象 **拒绝（`reject`）** 队列中每个请求对应的 Promise，通知等待的请求刷新令牌失败。
        - 最后，**清空 `failedQueue` 队列**，表示队列中的请求都已处理完毕。

- **`originalRequest`**:
    - 类型：`AxiosRequestConfig & CustomAxiosRequestConfig` (实际上在拦截器中被断言为 `CustomAxiosRequestConfig`)
    - 功能：这是一个变量，**代表导致当前 401 错误响应的原始请求的配置信息**。
    - 作用：
        - 存储了原始请求的所有配置，例如 URL、请求头、请求方法、请求数据等。
        - 在刷新令牌成功后，会使用新的访问令牌**更新 `originalRequest.headers['Authorization']`**。
        - 最终会使用 `axios(originalRequest)` **重新发送**这个原始请求。

- **`originalRequest._retry`**:
    - 类型：`boolean | undefined`
    - 功能：这是一个**自定义的属性**，添加到 `originalRequest` 配置对象中，用于**标记当前请求是否已经尝试过重试** (刷新令牌后重发)。
    - 作用：
        - 初始状态下，`originalRequest._retry` 通常为 `undefined` 或 `false`。
        - 当检测到 401 错误并且准备发起刷新令牌请求时，会将 `originalRequest._retry` 设置为 `true`。
        - 在拦截器逻辑的开始处，会检查 `!originalRequest._retry`，**确保同一个请求只进行一次刷新令牌和重试流程，防止无限循环重试**。
