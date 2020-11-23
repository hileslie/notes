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

### 异步并发控制

```javascript
function asyncPool(urls, limit, handle) {
    return new Promise((resolve, reject) => {
        const len = urls.length
        let counter = 0
        const start = async () => {
            while(counter < len && limit > 0) {
                limit--
                axios.post('xxx').then(() => {
                    limit++
                    counter++
                    if (counter === len) {
                        resolve()
                    } else {
                        start()
                    }
                })
            }
        }
        start()
    })
}
```

## 观察者模式（发布订阅模式、消息机制）

```javascript
const Observe = (function() {
    let message = {}
    return {
        // 订阅消息
        on: function(type, fn) {
            if (typeof message[type] === 'undefined') {
                message[type] = [fn]
            } else {
                message[type].push(fn)
            }
        },
        // 发布消息
        subscribe: function(type, args) {
            if (!message[type]) return
            let events = {
                type: type,
                args: args || {}
            }
            let len = message[type].length
            for(let i = 0; i < len; i++) {
                message[type][i].call(this, events)
            }
        },
        // 移除消息
        off: function(type, fn) {
            if (message[type] instanceof Array) {
                let i = message[type].length - 1
                for(; i >=0; i--) {
                    message[type][i] === fn && message[type].splice(i, 1)
                }
            }
        }
    }
})()
//订阅消息
Observe.on('say', function (data) {
    console.log(data.args.text);
})
Observe.on('success',function () {
    console.log('success')
});
//发布消息
Observe.subscribe('say', { text : 'hello world' } )
Observe.subscribe('success');
```

## 函数继承

### 构造函数继承

```javascript
// 组合继承
function inheritPrototype(subType, superType) {
    var prototype = Object.create(superType.prototype) // 创建对象，创建父类原型的一个副本
    prototype.constructor = subType // 增强对象，弥补因重写原型而失去的默认的constructor 属性
    subType.prototype = prototype
}
function SuperType() {}
function SubType() {
    SuperType.call(this)
}
inheritPrototype(SubType, SuperType)
var instance1 = new SubType()
```

### class继承



## 业务代码

### 实现一个方法，让页面平滑滑动到顶部

```javascript
function scrollToTopAnimation(time) {
    const scrollTop = (document.documentElement || document.body).scrollTop;
    const frameTime = 100 / 60 // 每一帧的时间
    const frameNum = time / frameTime // 需要执行的次数
    const stepLength = scrollTop / frameNum // 步长
    const step = () => {
        const scrollTop = (document.documentElement || document.body).scrollTop
        if (scrollTop > 0) {
            window.scrollTo(0, scrollTop - stepLength)
            requestAnimationFrame(step)
        }
    }
    step()
}
```

### 实现一个方法，可以并发加载图片，数量可以控制

```javascript
let imgs = [
    "https://wx4.sinaimg.cn/mw690/006eLk0Pgy1gc8yvba15bj31hc0u0npd.jpg",
    "https://wx1.sinaimg.cn/mw690/006fUZ8Mgy1gey9irj9xbj31v018o1kx.jpg",
    "https://wx4.sinaimg.cn/mw690/006AItRTly1gf3gx0ex0cj32iu1ojkjm.jpg",
    "https://wx1.sinaimg.cn/mw690/006eFiQily1gezdq016jij30c80bigm9.jpg",
    "https://wx2.sinaimg.cn/mw690/006eFiQily1ge8pj7moj5j30u0190tb0.jpg",
];
const loadImg = (url) => new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = function() {
        resolve()
    }
    img.onerror = reject
    img.src = url
})
function concurrentRequest(urls, max) {
    const firstQueue = urls.splice(0, max)
    const promises = firstQueue.map((url, index) => {
        return loadImg(url).then(() => {
            return index
        })
    })
    return urls.reduce((acc, cur) => {
        return acc.then(() => {
            return Promise.race(promises)
        }).then((index) => {
            promises[index] = loadImg(cur).then(() => {
                return index
            })
        })
    }, Promise.resolve()).then(() => {
        return Promise.all(promises)
    })
}
concurrentRequest(imgs, 3).then(() => {
    console.log('finish')
})
```

### 字符串去重，并统计出出现字数最多的字符串

```javascript
function getUnicleStringAndMaxChar(str) {
    const map = new Map()
    for(const char of str) {
        if (map.has(char)) {
            map.set(char, map.get(char) + 1)
        } else {
            map.set(char, 1)
        }
    }
    const res = [...map.keys()].join('')
    const arr = [...map.entries()].sort((a, b) => {
        return b[1] - a[1]
    })
    const maxChar = arr[0][0]
    return {
        res,
        maxChar
    }
}
```





## 考题

### 写一个函数timeBitmapToRanges，将上述规则描述的时间位图转换成一个选中时间区间的数组

```javascript
// 将一天24小时按每半小划分成48段，我们用一个位图表示选中的时间区间，
// 例如`110000000000000000000000000000000000000000000000`，
// 表示第一个半小时和第二个半小时被选中了，其余时间段都没有被选中，
// 也就是对应00:00~01:00这个时间区间。一个位图中可能有多个不连续的
// 时间区间被选中，例如`110010000000000000000000000000000000000000000000`，
// 表示00:00-1:00和02:00-02:30这两个时间区间被选中了。

// 要求：写一个函数timeBitmapToRanges，将上述规则描述的时间位图转换成一个选中时间区间的数组。
// 示例输入：`"110010000000000000000000000000000000000000000000"`
// 示例输出：`["00:00~01:00", "02:00~02:30"]`

function format(start, end) {
    let endHour = (end / 2).toFixed(1);
    let startHour = (start / 2).toFixed(1);
    let reg = /(\d+)\.(\d+)/;
    const endRes = endHour.match(reg);
    const startRes = startHour.match(reg);
    return (
        addZero(startRes[1]) +
        ':' +
        addZero(startRes[2]) +
        '~' +
        addZero(endRes[1]) +
        ':' +
        addZero(endRes[2])
    );
}
function addZero(num) {
    num = num === '5' ? '30' : num;
    return num.length > 1 ? num : '0' + num;
}
function timeBitmapToRanges(timeBitmap) {
    let timeArr = timeBitmap.split('').map(v => +v);
    const res = [];
    let range = {};
    let start = 0;
    for (let i = 0; i <= timeArr.length; i++) {
        if (timeArr[i]) {
            start++;
        }
        if (!timeArr[i] && timeArr[i - 1]) {
            range[i] = start;
            start = 0;
        }
    }
    for (let j in range) {
        res.push(format(parseInt(j - range[j]), parseInt(j)));
    }
    return res;
}
```

### 转化为驼峰命名

```javascript
let s1 = 'get-element-by-id'
function sc(str) {
  return str.replace(/-\w/g, (x) => {
    return x.slice(1).toUpperCase()
  })
}
```

### 解析 URL Params 为对象

```javascript
function parseParam(url) {
  const paramsStr = /.+\?(.+)$/.exec(url)[1]; // 将 ? 后面的字符串取出来
  const paramsArr = paramsStr.split('&'); // 将字符串以 & 分割后存到数组中
  let paramsObj = {};
  // 将 params 存到对象中
  paramsArr.forEach(param => {
    if (/=/.test(param)) { // 处理有 value 的参数
      let [key, val] = param.split('='); // 分割 key 和 value
      val = decodeURIComponent(val); // 解码
      val = /^\d+$/.test(val) ? parseFloat(val) : val; // 判断是否转为数字

      if (paramsObj.hasOwnProperty(key)) { // 如果对象有 key，则添加一个值
        paramsObj[key] = [].concat(paramsObj[key], val);
      } else { // 如果对象没有这个 key，创建 key 并设置值
        paramsObj[key] = val;
      }
    } else { // 处理没有 value 的参数
      paramsObj[param] = true;
    }
  })

  return paramsObj;
}
```

