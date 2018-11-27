Promise实现原理
---
首先调试Promise源码的方法比较简单的一种方法是：<br>
1. 打开浏览器控制台，<br>
2. 将Promise的源码粘到控制台下，再写对应的方法调用Promise类，<br>
3. 在then方法中添加debugger，便可查看代码一步一步的执行逻辑，更有助于源码的理解。<br>

先看一下Promise的基本使用：<br>
```
function f1(resolve, reject) {
  // 异步代码...
}

var p1 = new Promise(f1);
p1.then(f2);
```
接下来是对Promise源码的解析：<br>
1. Promise 基本结构
2. Promise类
3. Promise 状态和值 pending, Fulfilled, rejected
4. Promise 的核心: then 方法

### Promise 基本结构  
```
new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('FULFILLED')
  }, 1000)
})

eg:
let func = function() {
    return new MyPromise((resolve, reject) => {
        resolve('返回值');
    });
};
```
构造函数Promise必须接受一个函数作为参数，我们称该函数为handle，handle又包含resolve和reject两个参数，它们是两个函数。

### MyPromise类
上面例子的func中，通过new MyPromise调用MyPromise类即：<br>
```
class MyPromise {
    constructor (handle) {
      if (!isFunction(handle)) {
        throw new Error('MyPromise must accept a function as a parameter')
      }
      // 添加状态
      this._status = PENDING
      // 添加状态
      this._value = undefined
      // 添加成功回调函数队列
      this._fulfilledQueues = []
      // 添加失败回调函数队列
      this._rejectedQueues = []
      // 执行handle
      try {
        handle(this._resolve.bind(this), this._reject.bind(this)) 
      } catch (err) {
        this._reject(err)
      }
    }
```
__promise的执行流程/步骤__
* MyPromise类设置基属性 _status，_value，_fulfilledQueues，_rejectedQueues<br>
* 然后在try中立即执行handle方法，即new MyPromise((resolve, reject))中的(resolve, reject)，<br>
 handle函数包含 resolve 和 reject 两个参数，它们是两个函数，可以用于改变 Promise 的状态和传入 Promise 的值，<br>
* 接下来执行--resolve('返回值')--：
 
### Promise 状态和值 pending, Fulfilled, rejected
```
    // 添加resovle时执行的函数
    _resolve (val) {
      console.log()
      const run = () => {
        if (this._status !== PENDING) return
        this._status = FULFILLED
        // 依次执行成功队列中的函数，并清空队列
        const runFulfilled = (value) => {
          let cb;
          while (cb = this._fulfilledQueues.shift()) {
            cb(value)
          }
        }
        // 依次执行失败队列中的函数，并清空队列
        const runRejected = (error) => {
          let cb;
          while (cb = this._rejectedQueues.shift()) {
            cb(error)
          }
        }
        /* 如果resolve的参数为Promise对象，则必须等待该Promise对象状态改变后,
          当前Promsie的状态才会改变，且状态取决于参数Promsie对象的状态
        */
        if (val instanceof MyPromise) {
          val.then(value => {
            this._value = value
            runFulfilled(value)
          }, err => {
            this._value = err
            runRejected(err)
          })
        } else {
          this._value = val
          runFulfilled(val)
        }
      }
      // 为了支持同步的Promise，这里采用异步调用
      setTimeout(run, 0)
    }
```
### Promise 的核心: then 方法
* 执行到_resolve中的run方法时，由于setTimeout(run, 0)，则异步执行.then()方法。虽然是延迟0秒执行，但是我们知道js是单线程+消息队列，必须等主线程代码执行完毕才能开始执行消息队列当中的代码。因此，会首先执行then这个方法。then执行完毕后，再执行setTimeout里面的方法。<br>
如果有多个.then()回调，则会接着执行主线程上的代码。通过 test1 运行结果可以看出，then方法内部先执行,只有等待resolve返回结果的在setTimeout后执行：<br>
__test1:__
```
let func = function() {
    return new Promise((resolve, reject) => {
        resolve('返回值');
    });
};

let cb = function() {
    return '新的值';
}

func().then(function () {console.log('test1');
    return cb();
}).then(resp => {console.log('test2');
    console.warn(resp);
    console.warn('1 =========<');
});
运行结果：
VM460:11 test1
VM460:13 test2
VM460:14 新的值
func.then.then.resp @ VM460:14
Promise.then (async)
(anonymous) @ VM460:13
VM460:15 1 =========<
```
__then方法的源码__
```
// 添加then方法
    then (onFulfilled, onRejected) {
      const { _value, _status } = this
      // 返回一个新的Promise对象
      return new MyPromise((onFulfilledNext, onRejectedNext) => {
        // 封装一个成功时执行的函数
        let fulfilled = value => {
          try {
            if (!isFunction(onFulfilled)) {
              onFulfilledNext(value)
            } else {
              let res =  onFulfilled(value);
              if (res instanceof MyPromise) {
                // 如果当前回调函数返回MyPromise对象，必须等待其状态改变后在执行下一个回调
                res.then(onFulfilledNext, onRejectedNext)
              } else {
                //否则会将返回结果直接作为参数，传入下一个then的回调函数，并立即执行下一个then的回调函数
                onFulfilledNext(res)
              }
            }
          } catch (err) {
            // 如果函数执行出错，新的Promise对象的状态为失败
            onRejectedNext(err)
          }
        }
        // 封装一个失败时执行的函数
        let rejected = error => {
          try {
            if (!isFunction(onRejected)) {
              onRejectedNext(error)
            } else {
                let res = onRejected(error);
                if (res instanceof MyPromise) {
                  // 如果当前回调函数返回MyPromise对象，必须等待其状态改变后在执行下一个回调
                  res.then(onFulfilledNext, onRejectedNext)
                } else {
                  //否则会将返回结果直接作为参数，传入下一个then的回调函数，并立即执行下一个then的回调函数
                  onFulfilledNext(res)
                }
            }
          } catch (err) {
            // 如果函数执行出错，新的Promise对象的状态为失败
            onRejectedNext(err)
          }
        }
        switch (_status) {
          // 当状态为pending时，将then方法回调函数加入执行队列等待执行
          case PENDING:console.log('test5')
            this._fulfilledQueues.push(fulfilled)
            this._rejectedQueues.push(rejected)
            break
          // 当状态已经改变时，立即执行对应的回调函数
          case FULFILLED:
            fulfilled(_value)
            break
          case REJECTED:
            rejected(_value)
            break
        }
      })
    }
```
### Promise 对象then 的结构
Promise 对象的 then 方法接受两个参数：
```
promise.then(onFulfilled, onRejected)
```
__参数可选__<br>
  * onFulfilled 和 onRejected 都是可选参数。
  * 如果 onFulfilled 或 onRejected 不是函数，其必须被忽略onFulfilled 特性
  
__onFulfilled 特性__
  * 如果 onFulfilled 是函数
    * 当 promise 状态变为成功时必须被调用，其第一个参数为 promise 成功状态传入的值（ resolve 执行时传入的值）
    * 在 promise 状态改变前其不可被调用
    * 其调用次数不可超过一次
    
__onRejected 特性__
  * 如果 onRejected 是函数
    * 当 promise 状态变为失败时必须被调用，其第一个参数为 promise 失败状态传入的值（ reject 执行时传入的值）
    * 在 promise 状态改变前其不可被调用
    * 其调用次数不可超过一次
    
__多次调用__
  * then 方法可以被同一个 promise 对象调用多次
  
__返回__
  * then 方法必须返回一个新的 promise 对象, 支持链式调用

__用通俗的话来说__
  * then方法提供一个供自定义的回调函数，若传入非函数，则会忽略当前then方法。<br>
  * 回调函数中会把上一个then中返回的值当做参数值供当前then方法调用。<br>
  * then方法执行完毕后需要返回一个新的值给下一个then调用（没有返回值默认使用undefined）。<br>
  * 每个then只可能使用前一个then的返回值。<br>
  
[结合源码，通过https://segmentfault.com/a/1190000010420744可更清晰的了解then](https://segmentfault.com/a/1190000010420744)

全部源码
-

```
// 判断变量否为function
  const isFunction = variable => typeof variable === 'function'
  // 定义Promise的三种状态常量
  const PENDING = 'PENDING'
  const FULFILLED = 'FULFILLED'
  const REJECTED = 'REJECTED'

  class MyPromise {
    constructor (handle) {
      if (!isFunction(handle)) {
        throw new Error('MyPromise must accept a function as a parameter')
      }
      // 添加状态
      this._status = PENDING
      // 添加状态
      this._value = undefined
      // 添加成功回调函数队列
      this._fulfilledQueues = []
      // 添加失败回调函数队列
      this._rejectedQueues = []
      // 执行handle
      try {
        handle(this._resolve.bind(this), this._reject.bind(this)) 
      } catch (err) {
        this._reject(err)
      }
    }
    // 添加resovle时执行的函数
    _resolve (val) {
      const run = () => {
        if (this._status !== PENDING) return
        this._status = FULFILLED
        // 依次执行成功队列中的函数，并清空队列
        const runFulfilled = (value) => {
          let cb;
          while (cb = this._fulfilledQueues.shift()) {
            cb(value)
          }
        }
        // 依次执行失败队列中的函数，并清空队列
        const runRejected = (error) => {
          let cb;
          while (cb = this._rejectedQueues.shift()) {
            cb(error)
          }
        }
        /* 如果resolve的参数为Promise对象，则必须等待该Promise对象状态改变后,
          当前Promsie的状态才会改变，且状态取决于参数Promsie对象的状态
        */
        if (val instanceof MyPromise) {
          val.then(value => {
            this._value = value
            runFulfilled(value)
          }, err => {
            this._value = err
            runRejected(err)
          })
        } else {
          this._value = val
          runFulfilled(val)
        }
      }
      // 为了支持同步的Promise，这里采用异步调用
      setTimeout(run, 0)
    }
    // 添加reject时执行的函数
    _reject (err) { 
      if (this._status !== PENDING) return
      // 依次执行失败队列中的函数，并清空队列
      const run = () => {
        this._status = REJECTED
        this._value = err
        let cb;
        while (cb = this._rejectedQueues.shift()) {
          cb(err)
        }
      }
      // 为了支持同步的Promise，这里采用异步调用
      setTimeout(run, 0)
    }
    // 添加then方法
    then (onFulfilled, onRejected) {
      const { _value, _status } = this
      // 返回一个新的Promise对象
      return new MyPromise((onFulfilledNext, onRejectedNext) => {
        // 封装一个成功时执行的函数
        let fulfilled = value => {
          try {
            if (!isFunction(onFulfilled)) {
              onFulfilledNext(value)
            } else {
              let res =  onFulfilled(value);
              if (res instanceof MyPromise) {
                // 如果当前回调函数返回MyPromise对象，必须等待其状态改变后在执行下一个回调
                res.then(onFulfilledNext, onRejectedNext)
              } else {
                //否则会将返回结果直接作为参数，传入下一个then的回调函数，并立即执行下一个then的回调函数
                onFulfilledNext(res)
              }
            }
          } catch (err) {
            // 如果函数执行出错，新的Promise对象的状态为失败
            onRejectedNext(err)
          }
        }
        // 封装一个失败时执行的函数
        let rejected = error => {
          try {
            if (!isFunction(onRejected)) {
              onRejectedNext(error)
            } else {
                let res = onRejected(error);
                if (res instanceof MyPromise) {
                  // 如果当前回调函数返回MyPromise对象，必须等待其状态改变后在执行下一个回调
                  res.then(onFulfilledNext, onRejectedNext)
                } else {
                  //否则会将返回结果直接作为参数，传入下一个then的回调函数，并立即执行下一个then的回调函数
                  onFulfilledNext(res)
                }
            }
          } catch (err) {
            // 如果函数执行出错，新的Promise对象的状态为失败
            onRejectedNext(err)
          }
        }
        switch (_status) {
          // 当状态为pending时，将then方法回调函数加入执行队列等待执行
          case PENDING:
            this._fulfilledQueues.push(fulfilled)
            this._rejectedQueues.push(rejected)
            break
          // 当状态已经改变时，立即执行对应的回调函数
          case FULFILLED:
            fulfilled(_value)
            break
          case REJECTED:
            rejected(_value)
            break
        }
      })
    }
    // 添加catch方法
    catch (onRejected) {
      return this.then(undefined, onRejected)
    }
    // 添加静态resolve方法
    static resolve (value) {
      // 如果参数是MyPromise实例，直接返回这个实例
      if (value instanceof MyPromise) return value
      return new MyPromise(resolve => resolve(value))
    }
    // 添加静态reject方法
    static reject (value) {
      return new MyPromise((resolve ,reject) => reject(value))
    }
    // 添加静态all方法
    static all (list) {
      return new MyPromise((resolve, reject) => {
        /**
         * 返回值的集合
         */
        let values = []
        let count = 0
        for (let [i, p] of list.entries()) {
          // 数组参数如果不是MyPromise实例，先调用MyPromise.resolve
          this.resolve(p).then(res => {
            values[i] = res
            count++
            // 所有状态都变成fulfilled时返回的MyPromise状态就变成fulfilled
            if (count === list.length) resolve(values)
          }, err => {
            // 有一个被rejected时返回的MyPromise状态就变成rejected
            reject(err)
          })
        }
      })
    }
    // 添加静态race方法
    static race (list) {
      return new MyPromise((resolve, reject) => {
        for (let p of list) {
          // 只要有一个实例率先改变状态，新的MyPromise的状态就跟着改变
          this.resolve(p).then(res => {
            resolve(res)
          }, err => {
            reject(err)
          })
        }
      })
    }
    finally (cb) {
      return this.then(
        value  => MyPromise.resolve(cb()).then(() => value),
        reason => MyPromise.resolve(cb()).then(() => { throw reason })
      );
    }
  }
  let func = function() {
    return new MyPromise((resolve, reject) => {
        resolve('返回值');
    });
};

let cb = function() {
    return '新的值';
}

func().then(function () {
    return cb();
}).then(resp => {
    console.warn(resp);
    console.warn('1 =========<');
});
```
