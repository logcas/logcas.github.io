---
title: 从原理上理解Promise
date: 2020-01-15 00:04:38
tags: [JavaScript]
---

## Promise 常见试题
### 题目一
```js
setTimeout(function(){
    console.log(1);
}, 0)
new Promise(function(resolve){
    console.log(2);
    resolve();
    console.log(3);
}).then(function(){
    console.log(4);
})
console.log(5);
```

**解析：**
```
关键点：
1. setTimeout是宏任务，Promise.then是微任务
2. new Promise(fn)传入的函数是同步执行的
3. resolve 可以把Promise从Pending态转换到Fulfilled态，但回调是异步执行的。
4. resolve 和 rejetc 都不是 return，后面的语句能继续执行。
答案：
2 3 5 4 1
```


### 题目二
```js
Promise.resolve(1)
  .then((res) => {
    console.log(res);
    return 2;
  })
  .catch((res) => {
    console.log(res);
    return 3;
  })
  .then((res) => {
    console.log(res);
  });
```

**解析：**
```
关键点：
1. Promise.then 中的回调，会返回一个Promise，后续的执行会移交到新Promise去控制。
2. Promise.catch(fn) 实际上是 Promise.then(null, fn) 的语法糖，当then中的回调函数不传的时候，会出现参数透传。因此，第一个then返回的参数会透传到最后一个then中。
答案：
1 2
```

### 题目三
```js
Promise.resolve()
  .then( () => {
    return new Error('error!')
  })
  .then( res => {
    console.log('then: ', res)
  })
  .catch( err => {
    console.log('catch: ', err)
  });
```

**解析：**
```
关键点：
1. 当then中的回调抛出错误时，会通过reject处理，同时then的回调返回的都是一个新的Promise。
2. 与前一题同理，关键在于理解then和catch的实现。
答案：
catch: error!
```

### 题目四
```js
Promise.resolve(1)
  .then(2)
  .then(Promise.resolve(3))
  .then(console.log);
```

**解析：**
```
关键点：
1. 当then的参数不为函数时，会发生参数透传。
答案：
1
```

### 题目五
```js
Promise.resolve()
  .then(
    value => { throw new Error('error'); }, 
    reason => { console.error('fail1:', reason); }
  )
  .catch(
    reason => { console.error('fail2:', reason); }
  );
```

**解析：**
```
关键点：
1.第一个then返回一个新的Promise，由于then中抛出的异常，因此通过新的Promise的reject处理。catch实际上是新Promise的catch。
答案：
fail2: error
```

### 题目六
```js
console.log(1);
new Promise(function (resolve, reject){
    reject();
    resolve();
}).then(function(){
    console.log(2);
}, function(){
    console.log(3);
});
console.log(4);
```

**解析：**
```
关键点：
1. Promise 的状态一旦更改就不能再次修改。
答案：
1 4 3
```


### 题目七
```js
new Promise(resolve => { // p1
    resolve(1);
    
    // p2
    Promise.resolve().then(() => {
      console.log(2); // t1
    });

    console.log(4)
}).then(t => {
  console.log(t); // t2
});

console.log(3);
```

**解析：**
```
关键点：
1. new Promise 先执行，因此 t1 先被放到微任务队列
2. 执行完new Promise后，Promise的状态更改为resolved，通过then把t2添加到微任务队列中
答案：
4 3 2 1
```

### 题目八
```js
Promise.reject('a')
  .then(()=>{  
    console.log('a passed'); 
  })
  .catch(()=>{  
    console.log('a failed'); 
  });  
Promise
  .reject('b')
  .catch(()=>{  
    console.log('b failed'); 
  })
  .then(()=>{  
    console.log('b passed');
  })
```

**解析：**
```
关键点：
1. catch 实际上是 then
答案：
b failed
a passed
b passed
```

### 题目九
```js
async function async1() {
   console.log('async1 start')
   await async2()
   console.log('async1 end')
}

async function async2() {
   console.log('async2')
}

console.log('script start');

setTimeout(function () {
   console.log('settimeout')
})

async1();

new Promise(function (resolve) {
   console.log('promise1');
   resolve();
}).then(function () {
   console.log('promise2');
})

console.log('script end');
```

## 手写符合Promise/A+ 规范的Promise
深入理解Promise的机理的最好方式是：
1. 重复看阮一峰的ES6入门教程
2. 阅读规范，手写Promise

今天花了大半天时间，完成了符合规范的Promise编写，通过所有测试用例：

![image](http://static-cdn.lxzmww.xyz/PromiseA+.JPG)

```js
const PENDING = 'pending';
const FULFILLED = 'fulfilled';
const REJECTED = 'rejected';

class MyPromise {
  constructor(executor) {
    this.value = null;
    this.reason = null;
    this.status = PENDING;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];
    const resolve = value => {
      if (value instanceof MyPromise) {
        return value.then(resolve, reject);
      }
      setTimeout(() => {
        if (this.status === PENDING) {
          this.status = FULFILLED;
          this.value = value;
          this.onFulfilledCallbacks.map(cb => cb(this.value));
        }
      })
    };
    const reject = reason => {
      setTimeout(() => {
        if (this.status === PENDING) {
          this.status = REJECTED;
          this.reason = reason;
          this.onRejectedCallbacks.map(cb => cb(this.reason));
        }
      })
    };
    try {
      executor(resolve, reject);
    } catch (e) {
      reject(e);
    }
  }

  then(onFulfilled, onRejected) {
    let promise2;
    // onFulfilled 和 onRejected 函数是可选参数
    // 为了实现参数透传，这里需要设立默认的函数
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
    onRejected = typeof onRejected === 'function' ? onRejected : error => {
      throw error;
    };
    if (this.status === FULFILLED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onFulfilled(this.value);
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        })
      }));
    } else if (this.status === REJECTED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onRejected(this.reason);
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        });
      }));
    } else {
      return (promise2 = new MyPromise((resolve, reject) => {
        this.onFulfilledCallbacks.push(value => {
          try {
            let x = onFulfilled(value);
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        });
        this.onRejectedCallbacks.push(reason => {
          try {
            let x = onRejected(reason);
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        });
      }));
    }
  }

  catch (onRejected) {
    return this.then(null, onRejected);
  }
}

function resolvePromise(promise, x, resolve, reject) {
  // 如果 promise 和 x 指向同一对象，以 TypeError 为据因拒绝执行 promise。
  if (promise === x) {
    reject(TypeError('禁止循环引用'));
  }
  // 如果 x 为 Promise ，则使 promise 接受 x 的状态
  if (x instanceof MyPromise) {
    // 如果 x 处于等待态， promise 需保持为等待态直至 x 被执行或拒绝
    if (x.status === PENDING) {
      x.then(
        y => {
          resolvePromise(promise, y, resolve, reject);
        },
        reason => {
          reject(reason);
        }
      );
      // 如果 x 处于执行态，用相同的值执行 promise
      // 如果 x 处于拒绝态，用相同的据因拒绝 promise
    } else {
      x.then(resolve, reject);
    }
    // x 为对象或函数 (即 x 为 thenable)
  } else if (x && (typeof x === 'object' || typeof x === 'function')) {
    let called = false;
    try {
      // 把 x.then 赋值给 then。
      let then = x.then;
      //如果 then 是函数，将 x 作为函数的作用域 this 调用之。
      // 传递两个回调函数作为参数，第一个参数叫做 resolvePromise ，第二个参数叫做 rejectPromise:
      if (typeof then === 'function') {
        then.call(
          x,
          // 如果 resolvePromise 以值 y 为参数被调用，则运行 [[Resolve]](promise, y)
          // 如果 resolvePromise 和 rejectPromise 均被调用，或者被同一参数调用了多次，则优先采用首次调用并忽略剩下的调用
          y => {
            if (called) {
              return;
            }
            called = true;
            resolvePromise(promise, y, resolve, reject)
          },
          // 如果 rejectPromise 以据因 r 为参数被调用，则以据因 r 拒绝 promise
          // 如果 resolvePromise 和 rejectPromise 均被调用，或者被同一参数调用了多次，则优先采用首次调用并忽略剩下的调用
          r => {
            if (called) {
              return;
            }
            called = true;
            reject(r);
          }
        );
        // 如果 then 不是函数，以 x 为参数执行 promise
      } else {
        resolve(x);
      }
    } catch (e) {
      // 如果取 x.then 的值时抛出错误 e ，则以 e 为据因拒绝 promise。
      // 如果 resolvePromise 或 rejectPromise 已经被调用，则忽略之
      // 否则以 e 为据因拒绝 promise
      if (called) {
        return;
      }
      called = true;
      reject(e);
    }
    // 如果 x 不为对象或者函数，以 x 为参数执行 promise
  } else {
    resolve(x);
  }
}

MyPromise.deferred = function () {
  let defer = {};
  defer.promise = new MyPromise((resolve, reject) => {
    defer.resolve = resolve;
    defer.reject = reject;
  });
  return defer;
};

module.exports = MyPromise;
```

## 附：Promises/A+规范
https://www.ituring.com.cn/article/66566