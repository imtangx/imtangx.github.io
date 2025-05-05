---
title: SOLID原则在前端中的实践
date: 2025-05-05 13:28:06
tags: ['设计模式']
---
## 什么是SOLID原则
其是面向对象设计的五大基本原则，为了创建维护性高、扩展性高的代码。起源是Java、C++这类强面向对象对语言，但在前端、JS中也能有很好的借鉴作用。

## 单一职责原则(Single Responsibility Principle, SRP)
**每个类/模块应该只有单一的职责，降低类之间的耦合度，使代码可维护性高。**
例如react组件有时会同时负责UI、业务逻辑。
```jsx
const UserProfile = (userId) => {
  const [username, setUsername] = useState();
  useEffect(() => {
    request(userid).then(data => setUsername(data.username));
  })
  return <div>{username}</div>;
}
```
我们其实可以将业务逻辑抽离为hooks，使组件只负责UI渲染。
```jsx
const useUserData = (userId) => {
  const [username, setUsername] = useState();
  useEffect(() => {
    request(userid).then(data => setUsername(data.username));
  })
  return username;
}

const UserProfile = (userId) => {
  const username = useUserData(userId);
  return <div>{username}</div>;
}
```

## 开放/封闭原则（OCP）
**类/模块应该对扩展开放，对修改封闭。避免主体函数被修改可能的风险。**
例如有一个验证规则函数，有时新增规则需要在主体函数内新增判断分支。
```jsx
const verifyEmail = (email) => {
  if (email.length < 5) {
    return false;
  } else if (!email.includes('@')) {
    return false;
  } // TODO else if ...更多的规则

  return true;
}
```
我们可以把规则集作为可扩充的数组或函数参数列表传入，而不修改原来的主体函数。
```jsx
const verifyEmail = (email, rules) => {
  return rules.every(rule => rule(email));
}

const lengthRule = input => input.length >= 5;
const emailRule = input => input.includes('@');

verifyEmail('imtangx@gmail.com', [lengthRule, emailRule]);
```

## 里氏替换原则（LSP）
**子类应该能够完全替代父类，确保在原来使用父类的地方使用子类，代码的正确性不受到影响。**
例如有一个子类的逻辑函数与父类是相悖的，那就不应该成为它的子类。
```jsx
class Bird {
  fly() {
    console.log('我是鸟所以我会飞');
  }
}

class Sparrow extends Bird {
  fly() {
    console.log("我是麻雀所以我会飞");
  }
}

class Penguin extends Bird {
  fly() {
    console.log("我是企鹅我不会飞只会游泳");
  }
}
```
一种做法是把Fly行为分离出去作为不同的Bird类。
```jsx

class FlyingBird {
  fly() {
    // 具体的飞行行为由子类实现
  }
}

class SparrowLSP extends FlyingBird {
  fly() {
    console.log("我是麻雀所以我会飞");
  }
}

class SwimmingBird {
  swim() {
    // 具体的游泳行为由子类实现
  }
}

class PenguinLSP extends SwimmingBird {
  swim() {
    console.log("我是企鹅所以我会游泳");
  }
}
```

## 接口隔离原则（ISP）
**应该减少一个类依赖它不需要使用的方法/参数，应该将接口分解以实现更精确的依赖关系。**
例如在react组件中有时会传入它不需要的props，导致耦合度高。
```jsx
function MultiPurposeComponent({ user, posts, comments }) {
  return (
    <div>
      <UserProfile user={user} />
      <UserPosts posts={posts} />
      <UserComments comments={comments} />
    </div>
  );
}
```
可以适当的把组件进行拆分，使其只依赖自己的数据。
```jsx
function UserProfileComponent({ user }) {
  return <UserProfile user={user} />;
}

function UserPostsComponent({ posts }) {
  return <UserPosts posts={posts} />;
}

function UserCommentsComponent({ comments }) {
  return <UserComments comments={comments} />;
}
```

## 依赖倒置原则（DIP）
**高级模块不应该依赖于低级模块。两者都应该依赖于抽象（例如接口）。**
例如一个UserComponent组件依赖于fetchUser的实现。
```jsx
function fetchUser(userId) {
  return fetch(`/api/users/${userId}`).then(res => res.json());
}

function UserComponent({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);

  return <div>{user?.name}</div>;
}
```
应该把数据请求放到一个抽象接口中，使所有实现了fetchUser的类都能被userComponment所用。
```jsx
// 1. 定义抽象的数据获取接口
interface UserDataService {
  getUser(userId: string): Promise<any>;
}

// 2. 创建一个具体的服务类，使用 fetch 实现数据获取
class FetchUserDataService implements UserDataService {
  async getUser(userId: string): Promise<any> {
    const res = await fetch(`/api/users/${userId}`);
    return res.json();
  }
}

// 3. 创建另一个具体的服务类，例如使用本地缓存
class LocalCacheUserDataService implements UserDataService {
  private cache: Record<string, any> = {};

  async getUser(userId: string): Promise<any> {
    if (this.cache[userId]) {
      return Promise.resolve(this.cache[userId]);
    }
    
    const userData = { id: userId, name: `Cached User ${userId}` };
    this.cache[userId] = userData;
    return Promise.resolve(userData);
  }
}

// 4. 修改 UserComponent，使其依赖于 UserDataService 接口
interface UserComponentProps {
  userId: string;
  userDataService: UserDataService;
}

function UserComponent({ userId, userDataService }: UserComponentProps) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    userDataService.getUser(userId).then(setUser);
  }, [userId, userDataService]);

  return <div>{user?.name}</div>;
}

// 自己决定要使用哪个接口
<UserComponent userId="1" userDataService={apiService} />
<UserComponent userId="2" userDataService={cacheService} />
```

---

参考文章：
[🚀🚀🚀 (在 JavaScript 和 TypeScript 框架中应用 SOLID 原则](https://juejin.cn/post/7430514568088797218?searchId=20250421174102C7BE3610D16104FF473A)