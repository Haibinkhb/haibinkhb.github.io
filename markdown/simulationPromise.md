Promise 接收一个执行器（executor）函数，Promise 构造函数执行时立即调用该函数,该函数接受两个参数，分别是 resolve 函数和 reject 函数

```js
// 第一版
function Promise(executor){
    /*
        实例的属性
    */
   let that = this
   this.status = 'pending' // 初始状态
   this.data = ''   // 用来保存数据（成功时的value或失败时的reason）
   this.callBacks = [] // 用来存储待执行回调 格式：{onFulfilled:<function>, onRejected:<function>}

    /*
        resolve 和 reject 函数被调用时，分别将promise的状态改为 fulfilled（完成）和 rejected（失败）。
    */
    function resolve(value){ // resolve 函数接收一个参数作为 Promise 的结果值
        // Promise 状态只允许改变一次，如果已经被改变直接返回
        if(that.status !== 'pending'){
            return
        }
        // 状态改为 fulfilled
        that.status = 'fulfilled'
        // 保存数据
        that.data = value
        // 如果已经有待执行的回调函数了，立即异步执该回调函数
        if(that.callBacks.length){
            that.callBacks.forEach(item =>{
                setTimeout(() => {
                    item.onFulfilled(that.data)
                })
            })
        }
    }
    function reject(reason){ // reject 函数接收一个参数作为 Promise 失败的原因
        // Promise 状态只允许改变一次，如果已经被改变直接返回
        if(that.status !== 'pending'){
            return
        }
        // 状态改为 rejected
        that.status = 'rejected'
        // 保存数据
        that.data = reason
        // 如果已经有待执行的回调函数了，立即异步执该回调函数
        if(that.callBacks.length){
            that.callBacks.forEach(item =>{
                setTimeout(() => {
                    item.onRejected(that.data)
                })
            })
        }
    }
    try{
        // 执行 executor （执行器函数）
        executor(resolve, reject)
    }catch(error){ // 可能会抛出异常，需要捕获并 reject 异常
        reject(error)
    }
}
```

简单的实现了 Promise 构造函数，来试试效果

```js
let p = new Promise((reslove, reject)=>{
            reslove(1)
    })
console.log(p)
/*
    callBacks: []
    data: 1
    status: "fulfilled"
*/

let p = new Promise((reslove, reject)=>{
            reject(2)
    })
console.log(p)
/*
    callBacks: []
    data: 2
    status: "rejected"
*/
```

好像效果还行，再来实现一下 Promise.prototype.then()方法：

```js
Promise.prototype.then = function (onFulfilled, onRejected){
    // then 方法接收两个回调函数作为参数，会根据 Promise 的状态来决定执行哪个回调
    // 同样先保存 this
    let that = this

    /*
        判断当前 Promise 的状态
    */
    if (that.status === 'pending') { // 还没有改变状态，不能执行回调，所以先将回调保存到callBacks中
        that.callBacks.push({ onFulfilled, onRejected })
    } else if (that.status === 'fulfilled') {
        // Promise 已经成功 (fulfilled) 立即异步执行成功的回调函数 onFulfilled
        setTimeout(() => {
            onFulfilled(that.data)
        })
    } else {
        // Promise 已经失败 (rejected) 立即异步执行失败的回调函数 onRejected
        setTimeout(() => {
            onRejected(that.data)
        })
     }
}
```

好像很简单的实现了，试试效果：

```js
let p1 = new Promise((reslove, reject) => {
    reslove(1)
})
p1.then(
    v => {
        console.log(v) // 1
    }
)

let p2 = new Promise((reslove, reject) => {
    reject(2)
})
p1.then(
    v => {
        console.log(v)
    },
    r => {
        console.log(r) // 2
    }
)
```

可是 Promise 是支持链式调用的,也就是说 then 方法会返回一个新的Promise 实例，而且新返回的 Promise 实例的状态是由回调函数的结果来决定的。修改一下：

```js
Promise.prototype.then = function (onFulfilled, onRejected) {
    // then 方法接收两个回调函数作为参数，会根据 Promise 的状态来决定执行哪个回调
    // 同样先保存 this
    let that = this
    /* then 方法会返回一个新的 Promise, 新返回的 Promise 的状态由 onFulfilled 或 onRejected 的结果决定，会有三种情况
        1. result 是 Promise 的实例，要返回的 Promise 的状态 就是 result 的状态，值就是 result 的结果
        2. result 不是 Promise 的实例，要返回的 Promise 的状态为 resolve,值为 result
        3. 程序抛出异常 要返回的 Promise 的状态为 reject,值为抛出的异常
    */
    return new Promise((resolve, reject) => {
        /*
            判断当前 Promise 的状态
        */
        if (that.status === 'pending') { // 还没有改变状态，不能执行回调，所以先将回调保存到callBacks中
            that.callBacks.push({ onFulfilled, onRejected })
        } else if (that.status === 'fulfilled') {
            // Promise 已经成功 (fulfilled) 立即异步执行成功的回调函数 onFulfilled
            setTimeout(() => {
                try {
                    let result = onFulfilled(that.data)
                    if (result instanceof Promise) {
                        // 1. result 是 Promise的实例，要返回的 Promise 的状态 就是 result 的状态，值就是 result 的结果
                        result.then(resolve, reject)
                    } else {
                        // 2. result 不是 Promise的实例，要返回的 Promise 的状态为 resolve,值为 result
                        resolve(result)
                    }
                } catch (error) {
                    // 3. 程序抛出异常 要返回的 Promise 的状态为 reject, 值为抛出的异常
                    reject(error)
                }
            })
        } else {
            // Promise 已经失败 (rejected) 立即异步执行失败的回调函数 onRejected
            setTimeout(() => {
                try {
                    let result = onRejected(that.data)
                    if (result instanceof Promise) {
                        // 1. result 是 Promise的实例，要返回的 Promise 的状态 就是 result 的状态，值就是 result 的结果
                        result.then(resolve, reject)
                    } else {
                        // 2. result 不是 Promise的实例，要返回的 Promise 的状态为 resolve,值为 result
                        resolve(result)
                    }
                } catch (error) {
                    // 3. 程序抛出异常 要返回的 Promise 的状态为 reject, 值为抛出的异常
                    reject(error)
                }
            })
        }
    })
}
```

测试一下

```js
new Promise((reslove, reject) => {
    reslove(1)
}).then(
    v => {
        console.log(v) // 1
        return new Promise((reslove, reject)=>{
            reject(2)
        })
    },
    r => {
        console.log(r)
    }
).then(
    v => {
        console.log(v)
    },
    r => {
        console.log(r) // 2
    }
)
```

现在支持链式调用了,也能正确的处理返回的 Promise 的结果了，但是代码的封装性是不是太差了，而且当 Promise 是 pending 状态时，我们直接调用了回调函数，并没有返回新的 Promise 实例。另外如果 then 方法接收到的不是一个函数，会将值往下传（onFulfilled）或者抛出错误

```js
// 最终版
Promise.prototype.then = function (onFulfilled, onRejected) {
    // then 方法接收两个回调函数作为参数，会根据 Promise 的状态来决定执行哪个回调

    // 同样先保存 this
    let that = this

    // 如果 onResolved 不是一个函数，将值往下传
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value
    // 如果 onRejected 不是一个函数， 抛出一个异常
    onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason }

    /* then 方法会返回一个新的 Promise, 新返回的 Promise 的状态（resolve 或 reject）由 onResolved 或 onRejected 的结果决定，会有三种情况
        1. result 是 Promise的实例，要返回的 Promise 的状态 就是 result 的状态，值就是 result 的结果
        2. result 不是 Promise的实例，要返回的 Promise 的状态为 resolve,值为 result
        3. 程序抛出异常 要返回的 Promise 的状态为 reject,值为抛出的异常
    */
    return new Promise((resolve, reject) => {
        // 封装一个处理结果的函数
        function handleResult(callBack) {
            try {
                let result = callBack(that.data)
                if (result instanceof Promise) {
                    // 1. result 是 Promise的实例，要返回的 Promise 的状态 就是 result 的状态，值就是 result 的结果
                    result.then(resolve, reject)
                } else {
                    // 2. result 不是 Promise的实例，要返回的 Promise 的状态为 resolve,值为 result
                    resolve(result)
                }
            } catch (error) {
                // 3. 程序抛出异常 要返回的 Promise 的状态为 reject, 值为抛出的异常
                reject(error)
            }
        }
        /*
            判断当前 Promise 的状态
        */
        if (that.status === 'pending') { // 还没有改变状态，不能执行回调，所以先将回调保存到callBacks中
            that.callBacks.push({
                onFulfilled() {
                    handleResult(onFulfilled)
                },
                onRejected() {
                    handleResult(onRejected)
                }
            })
        } else if (that.status === 'fulfilled') {
            // Promise 已经成功 (fulfilled) 立即异步执行成功的回调函数 onFulfilled
            setTimeout(() => {
                handleResult(onFulfilled)
            })
        } else {
            // Promise 已经成功 (rejected) 立即异步执行成功的回调函数 onRejected
            setTimeout(() => {
                handleResult(onRejected)
            })
        }
    })
    }
```

then方法基本实现了,再实现一些常用的api

##### Promise.prototype.catch()

Promise.prototype.catch方法是.then(null, rejection)或.then(undefined, rejection)的别名，用于指定发生错误时的回调函数。所以很简单就能实现

```js
Promise.prototype.catch = function (onRejected) {
    return this.then(undefined, onRejected)
}
```

##### Promise.resolve(value)

返回一个状态由给定value决定的Promise对象。会有几种情况：

1. 如果参数是 Promise 实例，那么Promise.resolve将不做任何修改、原封不动地返回这个实例。
2. 如果参数是一个原始值，或者是一个不具有then方法的对象，则Promise.resolve方法返回一个新的 Promise 对象，状态为resolved。
3. Promise.resolve()方法允许调用时不带参数，直接返回一个resolved状态的 Promise 对象，值为undefined。

```js
Promise.resolve = function (value) {
    // 如果参数是MyPromise实例，直接返回这个实例
    if(value instanceof Promise) return value
    return new Promise(resolve => {
        // 否则返回一个新的 Promise 对象，状态为 resolved
        resolve(value)
    })
}
```

##### Promise.reject(value)

Promise.reject(reason)方法也会返回一个新的 Promise 实例，该实例的状态为rejected。

```js
Promise.reject = function (value) {
    return new Promise((resolve, reject) => reject(value))
}
```

##### Promise.all(iterable)

Promise.all()方法接受一个iterable作为参数，iterable里的元素都是 Promise 实例，如果不是，就会先调用下面讲到的Promise.resolve方法，将参数转为 Promise 实例。all方法会返回一个 Promise 实例，它的状态由参数的结果决定：

1. 只有 iterable 里的元素的状态都变成fulfilled，返回的 Promise 实例的状态才会变成fulfilled
2. 只要 iterable 里的元素之中有一个被rejected，返回的 Promise 实例的状态就变成rejected

```js
 Promise.all = function (list) {
    let tempArr = new Array(list.length),
        count = 0
    return new Promise((resolve, reject) => {
        list.forEach((item, index) => {
            // 数组参数如果不是 Promise 实例，先调用 Promise.resolve 转换
            Promise.resolve(item).then(
                value => {
                    count++
                    tempArr[index] = value
                    // 只有数组中所以元素的状态都变成fulfilled，返回的Promise状态才会变成fulfilled
                    if(count === list.length){
                        resolve(tempArr)
                    }
                },
                // 只要数组里的元素之中有一个被rejected，返回的 Promise 实例的状态就变成rejected
                reason => {
                    reject(reason)
                }
            )
        })
    })
}
```

##### Promise.race(iterable)

Promise.all()方法接受一个iterable作为参数,Promise.race()方法的参数与Promise.all()方法一样，如果不是 Promise 实例，就会先调用下面讲到的Promise.resolve()方法，将参数转为 Promise 实例。不同的是当iterable参数里的任意一个子promise被成功或失败后，父promise马上也会用子promise的成功返回值或失败详情作为参数调用父promise绑定的相应句柄，并返回该promise对象。

```js
Promise.race = function (list) {
    return new Promise((resolve, reject) => {
        list.forEach(item => {
            Promise.resolve(item).then(
                // 只要有一个实例率先改变状态，新的 Promise 的状态就跟着改变
                value => {
                    resolve(value)
                },
                reason => {
                    reject(reason)
                }
            )
        })
    })
}
```

***

说明：本文是本人对 Promise 的一些简单的理解，如有错误还望指出。
