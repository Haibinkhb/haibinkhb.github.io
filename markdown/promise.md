Promise 是异步编程的一种解决方案。

> 所谓Promise，简单说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。从语法上说，Promise 是一个对象，从它可以获取异步操作的消息。Promise 提供统一的 API，各种异步操作都可以用同样的方法进行处理。

#### Promise 的特点

Promise对象有以下两个特点：

1. 对象状态不受外界影响。Promise对象代表一个异步操作，有三种状态：pending（进行中）、fulfilled（已成功）和rejected（已失败）。只有异步操作的结果，可以决定当前是哪一种状态，任何其他操作都无法改变这个状态。
2. 状态只能改变一次。只有两种可能：从  pending变为 fulfilled 和从 pending 变为 rejected。

#### Promise 的基本用法

```js
let promise = new Promise((resolve, reject)=>{
    // ...一些代码（异步操作）

    // 调用第一个参数（函数）Promise状态会成功（fulfilled），成功的结果就是传入的 value
    resolve(value) // 成功
    // 或者失败（rejected）
    reject(reason) // 失败的原因是传入的 reason
})
```

Promise构造函数接受一个函数作为参数，该函数的两个参数分别是resolve和reject。它们两个也是函数，由 JavaScript 引擎提供，不用自己部署。

resolve函数的作用是，将Promise对象的状态从“未完成”变为“成功”（即从 pending 变为 resolved），在异步操作成功时调用，并将异步操作的结果，作为参数传递出去；

resolve函数的作用是，将Promise对象的状态从“未完成”变为“成功”（即从 pending 变为 resolved），在异步操作成功时调用，并将异步操作的结果，作为参数传递出去；reject函数的作用是，将Promise对象的状态从“未完成”变为“失败”（即从 pending 变为 rejected），在异步操作失败时调用，并将异步操作报出的错误，作为参数传递出去。

##### Promise.prototype.then()

Promise 实例生成后可以用 then 方法分别指定 resolved 状态和 rejected 状态的回调函数。

```js
promise.then(
    value => { // Promise对象的状态变为resolved时调用, value 就是Promise 传出的值（reslove(value)）
        // ...
    },
    reason => {// Promise对象的状态变为rejected时调用, value 就是Promise 传出的值（reject(reason)）
        // ...
    }
)
```

then方法可以接受两个回调函数作为参数。第一个回调函数是Promise对象的状态变为resolved时调用，第二个回调函数是Promise对象的状态变为rejected时调用。其中，第二个函数是可选的，不一定要提供。这两个函数都接受Promise对象传出的值作为参数。

then 方法会返回一个 **新** 的 Promise 实例。因此可以采用链式写法

```js
new Promise((reslove, reject)=>{
    reslove(value)
}).then(
    value => {
        // ...
    },
    reason => {
        // ...
    }
).then(
    value => {
        // ...
    },
    reason => {
        // ...
    }
)
```

第一个then方法指定的回调函数，返回的是另一个Promise对象。这时，第二个then方法指定的回调函数，就会等待这个新的Promise对象状态发生变化。如果变为resolved，就调用第一个回调函数，如果状态变为rejected，就调用第二个回调函数。

##### Promise.prototype.catch()

catch方法 是then(null, rejection)或.then(undefined, rejection)的别名，用于指定发生错误时的回调函数。

```js
new Promise((reslove, reject)=>{
    // ...
}).then(
    value => {
        // ...
    }
).then(
    value => {
        // ...
    }
).catch( // 上面任何一个抛出的错误，都会被最后一个catch捕获。
    reason =>{
        // ...
    }
)
```

如果 Promise 对象状态变为 resolved，则会调用 then 方法指定的回调函数；如果异步操作抛出错误，状态就会变为 rejected，就会调用 catch 方法指定的回调函数，处理这个错误。另外，then 方法指定的回调函数，如果运行中抛出错误，也会被 catch 方法捕获。

##### Promise.reslove(value)

Promise.resolve方法的参数分成四种情况。

1. 如果参数是 Promise 实例，那么Promise.resolve将不做任何修改、原封不动地返回这个实例。
2. 如果参数是 thenable对象(具有then方法的对象),方法会将这个对象转为 Promise 对象，然后就立即执行thenable对象的then方法。
3. 如果参数是一个原始值，或者是一个不具有then方法的对象，则Promise.resolve方法返回一个新的 Promise 对象，状态为resolved。
4. Promise.resolve()方法允许调用时不带参数，直接返回一个resolved状态的

```js
Promise.reslove(1) // <resolved>: 1
Promise.reslove() // <resolved>: undefined
Promise.reslove(Promise.reject('失败了')) // <rejected>: '失败了'
```

##### Promise.reject(reason)

返回一个状态为失败的Promise对象，并将给定的失败信息传递给对应的处理方法

```js
Promise.reject(1) // rejected>: 1
Promise.reject() // <rejected>: undefined
Promise.reject(Promise.resolve(2)) // rejected>: 2
```

##### Promise.all()

Promise.all()方法用于将多个 Promise 实例，包装成一个新的 Promise 实例。

```js
let pAll = Promise.all([p1, p2, p3, ...])
```

上面代码中，Promise.all()方法接受一个数组作为参数，p1、p2、p3都是 Promise 实例，如果不是，就会先调用下面讲到的Promise.resolve方法，将参数转为 Promise 实例，再进一步处理。另外，Promise.all()方法的参数可以不是数组，但必须具有 Iterator 接口，且返回的每个成员都是 Promise 实例。

pAll的状态由p1、p2、p3决定，分成两种情况。

（1）只有p1、p2、p3的状态都变成fulfilled，p的状态才会变成fulfilled，此时p1、p2、p3的返回值组成一个数组，传递给p的回调函数。

（2）只要p1、p2、p3之中有一个被rejected，p的状态就变成rejected，此时第一个被reject的实例的返回值，会传递给p的回调函数。

##### Promise.race()

Promise.race()方法同样是将多个 Promise 实例，包装成一个新的 Promise 实例。

```js
const p = Promise.race([p1, p2, p3]);
```

上面代码中，只要p1、p2、p3之中有一个实例率先改变状态，p的状态就跟着改变。那个率先改变的 Promise 实例的返回值，就传递给p的回调函数。

***

参考:

* [ECMAScript 6 入门](https://es6.ruanyifeng.com/#docs/promise)

说明：本文大量参考大佬[阮一峰 ECMAScript 6 入门](https://es6.ruanyifeng.com/)，本人出于复习和总结知识点的目的加入些许个人理解，如有冒犯，敬请谅解。
