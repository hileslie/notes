# JavaScript基础

## Call、Apply、bind

### 实现call函数

```javascript
Function.prototype.myCall = function(context, ...args) {
    if (this === Function.prototype) {
        return undefined
    }
    var context = context || window
    var fn = Symbol()
    context[fn] = this
    var result = context[fn](...args)
    delete context[fn]
    return result
}
```

### 实现apply函数

````javascript
Function.prototype.myApply = function(context, args) {
    if (this === Function.prototype) {
        return undefined
    }
    var context = context || window
    var fn = Symbol()
    context[fn] = this
    var result = null
    if (Array.isArray(args)) {
        result = context[fn](...args)
    } else {
        result = context[fn]()
    }
    delete context[fn]
    return result
}
````

### 实现bind函数

```javascript
Function.prototype.myBind = function(obj) {
    // 第0位是this，所以得从第一位开始裁剪 
    var args = Array.prototype.slice.call(arguments, 1)
    var fn = this
    // 创建中介函数
    var _fn = function() {}
    var bound = function() {
        // 二次调用我们也抓取arguments对象 
        var params = Array.prototype.slice.call(arguments)
        // 通过constructor判断调用方式，为true this指向实例，否则为obj 
        fn.apply(this.constructor === fn ? this : obj, args.concat(params))
    }
    _fn.prototype = fn.prototype
    bound.prototype = new _fn()
    return bound
}
```

## 变量类型判断

### 实现一个判断变量类型的函数

```javascript
function getVarType(val) {
    let type = typeof val
    if(type === 'object') {
        let typeStr = Object.prototype.toString.call(val)
        typeStr = typeStr.split(' ')[1]
        type = typeStr.substring(0, typeStr.length - 1)
    }
    return type
}
```

## 节流与防抖

### 函数节流

```javascript
// 函数节流保证一个事件一定时间内只执行一次
function throttle(fn, threshold,scope) {
    let timer
    let prev = Date.now()
    return function() {
        let context = scope || this
        let args = arguments
        let now = Date.now()
        if(now - prev > threshold) {
            prev = now
            fn.apply(context, args)
        }
    }
}
```

### 函数防抖

```javascript
// 函数防抖可以把多个顺序的调用合并成一次
function debounce(fn, delay, scope) {
    let timer = null
    return function() {
        let context = scope || this
        let args = arguments
        clearTimeout(timer)
        timer = setTimeout(function () {
            fn.apply(context, args)
        }, delay)
    }
}
```

## Promise、async/await

### Promise

```javascript
// var p = new Promise((resolve, reject) => {
//     resolve('FULFILLED')
// })
// p.then((val) => {})
/**
 * 1. new Promise时，需要传递一个 executor 执行器，执行器立刻执行
 * 2. executor 接受两个参数，分别是 resolve 和 reject
 * 3. promise 只能从 pending 到 rejected, 或者从 pending 到 fulfilled
 * 4. promise 的状态一旦确认，就不会再改变
 * 5. promise 都有 then 方法，then 接收两个参数，分别是 promise 成功的回调 onFulfilled, 
 *      和 promise 失败的回调 onRejected
 * 6. 如果调用 then 时，promise已经成功，则执行 onFulfilled，并将promise的值作为参数传递进去。
 *      如果promise已经失败，那么执行 onRejected, 并将 promise 失败的原因作为参数传递进去。
 *      如果promise的状态是pending，需要将onFulfilled和onRejected函数存放起来，等待状态确定后，再依次将对应的函数执行(发布订阅)
 * 7. then 的参数 onFulfilled 和 onRejected 可以缺省
 * 8. promise 可以then多次，promise 的then 方法返回一个 promise
 * 9. 如果 then 返回的是一个结果，那么就会把这个结果作为参数，传递给下一个then的成功的回调(onFulfilled)
 * 10. 如果 then 中抛出了异常，那么就会把这个异常作为参数，传递给下一个then的失败的回调(onRejected)
 * 11.如果 then 返回的是一个promise,那么需要等这个promise，那么会等这个promise执行完，promise如果成功，
 *   就走下一个then的成功，如果失败，就走下一个then的失败
 */
const PENDING = 'PENDING'
const FULFILLED = 'FULFILLED'
const REJECTED = 'REJECTED'
const isFunction = function(variable) {
    typeof variable === 'function'
}
function MyPromise(handle) {
    if (!isFunction(handle)) {
        throw new Error('handle is sholud a function')
    }
    let self = this
    self.status = PENDING
    self.value = undefined
    self.reason = ''
    self.onFulfilled = [] // 成功回调
    self.onRejected = [] // 失败回调
    function resolve(val) {
        if (self.status === PENDING) {
            self.status = FULFILLED
            self.value = val
            self.onFulfilled.forEach(fn => fn())
        }
    }
    function reject(reason) {
        if (self.status === PENDING) {
            self.reason = reason
            self.onRejected.forEach(fn => fn())
        }
    }
    try {
        handle(resolve, reject)
    } catch (e) {
        reject(e)
    }
}
MyPromise.prototype.then = function(onFulfilled, onRejected) {
    onFulfilled = isFunction(onFulfilled) ? onFulfilled : (val) => val
    onRejected = isFunction(onRejected) ? onRejected : (val) => val
    let self = this
    let promise = new MyPromise((resolve, reject) => {
        if (self.status === FULFILLED) {
            setTimeout(() => {
                try {
                    let val = onFulfilled(self.value)
                    resolvePromise(promise, val, resolve, reject)
                } catch (e) {
                    reject(e)
                }
            })
        } else if (self.status === REJECTED) {
            setTimeout(() => {
                try {
                    let val = onRejected(self.reason)
                    resolvePromise(promise, val, resolve, reject)
                } catch (e) {
                    reject(e)
                }
            })
        } else {
            self.onFulfilled.push(() => {
                setTimeout(() => {
                    try {
                        let val = onRejected(self.value)
                        resolvePromise(promise, val, resolve, reject)
                    } catch (e) {
                        reject(e)
                    }
                })
            })
            self.onRejected.push(() => {
                setTimeout(() => {
                    try {
                        let val = onRejected(self.reason)
                        resolvePromise(promise, val, resolve, reject)
                    } catch (e) {
                        reject(e)
                    }
                })
            })
        }
    })
    return promise
}
function resolvePromise(promise, val, resolve, reject) {
    let self = this
    if (promise === val) {
        reject(new TypeError(''))
    }
    if (val && typeof val === 'object' || typeof val === 'function') {
        let used
        try {
            let then = val.then
            if (typeof then === 'function') {
                then.call(val, (parmas) => {
                    if (used) return
                    used = true
                    resolvePromise(promise, parmas, resolve, reject)
                }, (r) => {
                    if (used) return
                    used = true
                    reject(r)
                })
            } else {
                if (used) return
                used = true
                reject(val)  
            }
        } catch (e) {
            if (used) return
            used = true
            reject(e)
        }
    } else {
        resolve(val)
    }
}
MyPromise.resolve = function(promise) {
    if (promise instanceof MyPromise) {
        return promise
    }
    return new MyPromise((resolve, reject) => {
        if (promise && promise.then && typeof promise.then === 'function') {
            setTimeout(() => {
                promise.then(resolve, reject)
            })
        } else {
            resolve(promise)
        }
    })
}
MyPromise.all = function(promises) {
    return new MyPromise((resolve, reject) => {
        let index = 0
        let result = []
        if (promises.length === 0) {
            resolve(result)
        } else {
            function processValue(i, data) {
                result[i] = data
                if (++index === promises.length) {
                    resolve(result)
                }
            }
            for(let i = 0; i < promises.length; i++) {
                MyPromise.resolve(promises[i]).then(data => {
                    processValue(i, data)
                }, err => {
                    reject(err)
                    return
                })
            }
        }
    })
}
MyPromise.race = function(promises) {
    return new MyPromise((resolve, reject) => {
        if (promises.length === 0) {
            return
        } else {
            for(let i = 0; i < promises.length; i++) {
                MyPromise.resolve(promises[i]).then(data => {
                    resolve(data)
                    return
                }, err => {
                    reject(err)
                    return
                })
            }
        }
    })
}
```

### 延时函数

```javascript
// 延时函数
function sleep(time) {
    return new Promise(resolve => {
        setTimeout(resolve, time)
    })
}

// promise版本
const t1 = +new Date()
sleep(3000).then(() => {
    const t2 = +new Date()
    console.log(t2 - t1)
})
// async版本
async function test() {
    const t1 = +new Date()
    await sleep(3000)
    const t2 = +new Date()
    console.log(t2 - t1)
}
```

