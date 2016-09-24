---
title: 使用promise-memorize提升系统性能
---
从我开始接触node.js开发之后，一直都习惯使用varnish来做缓存处理，它的强大让我少考虑了很多问题，几个月前有朋友找我，说他们的系统访问量在增长，数据库的访问太频繁，有没简单的方法做优化。由于他们不希望再增加其它的缓存数据库(redis)之类，因此我介绍使用`async/memoize`来做函数级的缓存，将访问数据库的函数分类，将可缓存的缓存处理。

言归正传，对于`async/memoize`个人认为有以下不方便之处：

- 无法设置缓存时间

- 无法清除缓存

- 无相应的事件触发机制

因此我基于promise的形式编写了[`promise-memorize`](https://github.com/vicanso/promise-memorize)，包括以下特性：

- 能设置缓存的`ttl`，如果设置为`0`表示只用于同时并发的函数调用

- 能通过`hash key`获取缓存和删除缓存

- 事件触发，包括：`add`, `hit`, `resolve`, `reject`, `delete`, `expire`

- 对于promise的返回，无论是resolve、reject都做缓存

![img](/pics/promise-memorize.png)

### Get Stock From Yahoo

从yahoo api中获取股票信息(用于对实时性不高的展示用)，对同样的参数请求做10秒的缓存处理，若接口返回出错，则立即清除缓存。每隔60秒定时清除过期缓存。

```js
'use strict';
const request = require('superagent');
const memorize = require('promise-memorize');

function getStock(code, start, end, period) {
  const startArr = start.split('-');
  const endArr = end.split('-');
  const url = `http://ichart.yahoo.com/table.csv?s=${code}&a=${startArr[1]}&b=${startArr[2]}&c=${startArr[0]}&d=${endArr[1]}&e=${endArr[2]}&f=${endArr[0]}&g=${period}`;
  
  return new Promise((resolve, reject) => {
    console.info('GET yahoo:%s', url);
    request.get(url).end((err, res) => {
      if (err) {
        reject(err);
      } else {
        resolve(res.text);
      }
    });
  });
}

const memorizeGetStock = memorize(getStock, (code, start, end, period) => {
  // hash key function
  return `${code}-${start}-${end}-${period}`;
}, 10 * 1000);
// 设置60秒定时清除过期缓存
memorizeGetStock.periodicClear(60 * 1000);
memorizeGetStock.on('add', key => {
  console.info(`${key} add to cache`);
});
memorizeGetStock.on('delete', key => {
  console.info(`${key} delete from cache`);
});
memorizeGetStock.on('resolve', key => {
  console.info(`${key} resolve`);
});
memorizeGetStock.on('reject', key => {
  console.info(`${key} reject`);
  // when reject emit, remove the cache(key)
  memorizeGetStock.delete(key);
});

memorizeGetStock('600000.SS', '2016-01-01', '2016-03-31', 'd').then((csv) => {
  console.info('day data size:%d', csv.length);
}).catch(console.error);
```

通过使用上面的方式，将获取股票信息的调用频率从原来的每分钟10K的级别降低到0.1K，大大的提升了系统性能。

[![npm](http://img.shields.io/npm/v/promise-memorize.svg?style=flat-square)](https://www.npmjs.org/package/promise-memorize)
