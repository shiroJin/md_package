---
title: Promise基础介绍
---

### 一、promise介绍

promise原是前端社区中为解决回调陷阱而提出的一种方案。后在ES6中，官方对此进行了统一支持。promise写的代码便于阅读，代码相较优雅。
<!--more-->

### 二、promise的基础使用

promise是一个对象，构造方法中的参数是一个function，这里我们称之为executor，executor有两个参数——resolve和reject，两个参数也都是function。executor中执行相应的逻辑，在逻辑处理完成之后，根据结果的状态调用对应的函数——reject或者resolve。

*简单的promise构造代码：*

```javascript
var promise1 = new Promise(function(resolve, reject) {
  /**
    处理业务逻辑
    ...
    */
  if (ok){
    resolve('foo')
  } else {
		reject('err')
  }
})
```

*下图为promise的流程图：*

![promise状态流转](https://mdn.mozillademos.org/files/15911/promises.png)

当executor的事务处理完成后，promise的状态变为pending，待定的状态。resolve会调用promise.then()函数，reject会调用promise.catch函数。

```javascript
promise
  .then(res => {
    // 接下去的逻辑
  })
  .catch(err => {
    // 处理错误
  })
```

### 三、promise使用实例

网络请求处理的场景下，一般的代码如下：

```javascript
getData(method, url, successFun, failFun){
  var xmlHttp = new XMLHttpRequest();
  xmlHttp.open(method, url);
  xmlHttp.send();
  xmlHttp.onload = function () {
    if (this.status == 200 ) {
      successFun(this.response);
    } else {
      failFun(this.statusText);
    }
  };
  xmlHttp.onerror = function () {
    failFun(this.statusText);
  };
}

getData('GET', 'http://www.url.com', res => {
  // handle response
}, err => {
  // handle err
})
```

promise的写法如下：

```javascript
getData(method, url){
  var promise = new Promise(function(resolve, reject){
    var xmlHttp = new XMLHttpRequest();
    xmlHttp.open(method, url);
    xmlHttp.send();
    xmlHttp.onload = function () {
      if (this.status == 200 ) {
        resolve(this.response);
      } else {
        reject(this.statusText);
      }
    };
    xmlHttp.onerror = function () {
      reject(this.statusText);
    };
  })
  return promise;
}

getData()
  .then(res => {
  	// handle response
  }).catch(err => {
    // handle error
  })
```

对比两种方式，两者实现的功能一致。而promise链式的调用可读性会更强，整体逻辑更加的清楚。

### 四、async和await

ES6中async和await是对promise在异步处理中补充。

以下代码模拟异步任务的处理：

```javascript
function getData () {
  return promise = new Promise(function(resolve, reject) {
    setTimeout(() => {
      // ...
      if (ok) {
			  resolve()
      } else {
			  reject()
      }
    }, 1000)
  })
}

async function foo () {
  try {
    let res = await get()
    console.log(res)
  } catch (error) {
    console.log(error);
  }
}

foo()
```

Async/await是一种编写异步代码的新方法，两者一定要配合使用。因为是建立在promises之上的，所以也是非阻塞。使用Async/await编写异步代码看起来更靠近同步代码。逻辑会更加的清楚，便于阅读。

### 总结

promise是一种很好的设计，讲事务的处理封装成对象，整体编程更加的面向对象，链式的调用让代码更加的优雅。在OC、Swift语言中也有如此的设计，程序的思想是共通的嘛。



