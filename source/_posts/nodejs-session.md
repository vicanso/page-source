---
title: 打造更高效node.js的session处理
---
对于node.js的WEB开发，大部分的开发者都会熟悉或者了解[`express`](http://expressjs.com/)，因为它的中间件形式简单易懂，但也因为这个原因，很多人一直使用了不恰当的形式来使用中间件，首先我们来看看express的session主要处理流程：

![img](/pics/express-session.png)

注：shouldSave的判断包括isModified与isSaved的判断

- `isModified` 主要判断 originalHash 是否与 hash(session) 不等

- `isSaved` 主要判断 savedHash 是否与 hash(session) 不等

```js
'use strict';

const uuid = require('node-uuid');
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

// about 4K bytes
const basicInfo = require('./basic-info');
 
const app = express();

const getUserInfo = () => Object.assign({
  name: uuid.v4(),
}, basicInfo);

app.use(session({
  store: new RedisStore({
    ttl: 3600,
  }),
  resave: false,
  saveUninitialized: false,
  secret: 'keyboard cat',
}));

app.get('/user', (req, res) => {
  if (!req.session.user) {
    console.info('Create new session');
    req.session.user = getUserInfo();
  }
  res.json(req.session.user);
});
app.post('/user/shoppingcart', (req, res) => {
  const user = req.session.user;

  if (!user) {
    return res.status(403).json({
      error: '请先登录'
    });
  }
  if (!user.food) {
    user.food = [];
  }
  user.food.push(req.body);
  res.json(user.food);
});

app.get('/food', (req, res) => {
  res.json([]);
});

app.listen(3000);
console.info('Get user information from http://127.0.0.1:3000/user');
```

上面的例子中session middleware是每一个请求都会调用，这种写法很简单，不考虑后面的route是否需要用户信息，都从sesion中获取，牺牲了系统的性能来换取开发的难度减少，两个route的处理：

- `/user` 需要用到用户信息，因此需要调用 session middleware

- `/food` 不需要用到用户信息，因此可以无需调用 session middleware

下面我们来测试一下 `/food` 使用session(每次请求的cookie一致，session中的数据都可以从redis中获取，非重新创建)与不使用session的性能比较(不使用session的方式通过直接屏蔽session中间件临时处理):

```
<!-- use session middleware -->
Document Path:          /food
Document Length:        2 bytes

Concurrency Level:      100
Time taken for tests:   15.066 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      2020000 bytes
HTML transferred:       20000 bytes
Requests per second:    663.74 [#/sec] (mean)
Time per request:       150.661 [ms] (mean)
Time per request:       1.507 [ms] (mean, across all concurrent requests)
Transfer rate:          130.93 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   1.9      2      16
Processing:    62  148  42.6    139     407
Waiting:       44  115  34.5    109     352
Total:         66  150  42.6    141     408
```

```
<!-- not use session middleware -->
Document Path:          /food
Document Length:        2 bytes

Concurrency Level:      100
Time taken for tests:   3.416 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      2020000 bytes
HTML transferred:       20000 bytes
Requests per second:    2927.43 [#/sec] (mean)
Time per request:       34.160 [ms] (mean)
Time per request:       0.342 [ms] (mean, across all concurrent requests)
Transfer rate:          577.48 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   1.5      1      17
Processing:    23   32   8.2     30     116
Waiting:       20   32   8.2     30     116
Total:         26   34   8.3     32     118
```

从上面两个例子可以看出，使用session middleware的平均时间大概是140ms，而不使用session middleware则是不到40ms，session的处理大概要100ms，而对于 `/food` 请求，这100ms完全是一种浪费，大家可以想想自己平时开发的应用，如果也是以这种方式使用session，那么是浪费了多少无用的时间。可能很多人觉得对于小网站，这种偷懒的写法无可厚非，但是如果做点小调整就可以避免这种浪费，那么我们有什么理由不做呢？下面是调整之后的代码：

```js
'use strict';

const uuid = require('node-uuid');
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

// about 4K bytes
const basicInfo = require('./basic-info');
 
const app = express();
const sessionMiddleware = session({
  store: new RedisStore({
    ttl: 3600,
  }),
  resave: false,
  saveUninitialized: false,
  secret: 'keyboard cat',
});

const getUserInfo = () => Object.assign({
  name: uuid.v4(),
}, basicInfo);

app.get('/user', sessionMiddleware, (req, res) => {
  if (!req.session.user) {
    console.info('Create new session');
    req.session.user = getUserInfo();
  }
  res.json(req.session.user);
});

app.post('/user/shoppingcart', sessionMiddleware, (req, res) => {
  const user = req.session.user;

  if (!user) {
    return res.status(403).json({
      error: '请先登录'
    });
  }
  if (!user.food) {
    user.food = [];
  }
  user.food.push(req.body);
  res.json(user.food);
});

app.get('/food', (req, res) => {
  res.json([]);
});

app.listen(3000);
console.info('Get user information from http://127.0.0.1:3000/user');
```

通过上面的调整，已经可以把使用session middleware的route分开处理，对于最开始的流程图，那么是不是还有可以优化的处理呢？

- `只读session` 是否可以减少shouldSave的判断来提升性能

- `session ttl的更新` 由于session的ttl时间比较长，是否可以不需要每次都更新ttl

- `session cookie的创建` 是否可以固定的/user请求中才生成cookie

先来讨论一下只读session的处理，所谓的只读就是确保这个route的处理中，是需要获取到用户的session信息，但是不需要对这个信息做任何的更新。看了一下express-session的代码，只需要把req.sessionID与req.session清空，则可以达到这个目的（这里吐槽一下，express的middleware设计在next之后，前面的middleware在数据返回的时候就无法做处理，只能通过重载res.write, res.end来实现，koa则对此做了改进，建议还是使用koa）

```js
// 此 middleware 也不会去更新 ttl
const sessionReadonly = (req, res, next) => {
  if (!req.cookies['connect.sid']) {
    return res.status(403).json({
      error: '请先登录'
    });
  }
  // 在此我偷懒只对res.json做重载
  var _json = res.json;
  res.json = (data) => {
    delete req.sessionID;
    delete req.session;
    _json.call(res, data);
  };
  sessionMiddleware(req, res, next);
};
<!-- 中间部分代码省略 -->
app.get('/user/readonly', sessionReadonly, (req, res) => {
  const user = req.session.user;
  // user.count 的自增会无效，因为该session不会写到store
  if (!user.count) {
    user.count = 0;
  }
  user.count += 1;
  res.json(user);
});
```
通过使用sessionReadonly将该请求的平均处理时间由150ms降低到100ms，而对于减少touch的处理，不对express-session的代码做调整，暂时没有办法，而且增加readonly的处理之后，touch的调用也会少了很多，个人觉得这个也没有调整了必要了。而`session cookie的创建`，这个不是对性能上优化的考虑，而是我自己开发的时候，比较希望能将session的处理只固定在一处，因此将其它的session都会请求先判断是否有cookie，因此增加了`sessionWritable`

```js
const sessionWritable = (req, res, next) => {
  if (!req.cookies['connect.sid']) {
    return res.status(403).json({
      error: '请先登录'
    });
  }
  sessionMiddleware(req, res, next);
};
```

注：上面例子的[source code](https://github.com/vicanso/articles/tree/master/express-session)，使用koa的朋友可以使用我写的模块[koa-simple-session](https://github.com/vicanso/koa-simple-session)