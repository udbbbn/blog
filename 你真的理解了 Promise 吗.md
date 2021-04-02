### 你真的理解了 Promise 吗

该文章后续补充完整！

```js
Promise.resolve 等于 return new Promise((resolve) => resolve())
而 Promise.then 遇到 Promise.resolve 时会调用其 .then 方法
```
但实际结果可能是因为 `Promise` 异步的规则，导致这里并不是直接调用的 `then` 方法。而是抛了一个 `microTask(() => Promise.then)` 也就是 两个微任务。思考过程来自于下面这道题。
```js
1. new Promise((resolve,reject) => {
2.     console.log('外部promise')
3.     resolve()
4. })
5.    .then(() => {
6.    console.log('外部第一个then')
7.    new Promise((resolve,reject) => {
8.        console.log('内部promise')
9.        resolve()
10.    })
11.        .then(() => {
12.        console.log('内部第一个then')
13.        return Promise.resolve()
14.    })
15.        .then(() => {
16.        console.log('内部第二个then')
17.    })
18.})
19.    .then(() => {
20.    console.log('外部第二个then')
21.})
22.    .then(() => {
23.    console.log('外部第三个then')
24.})
25.    .then(() => {
26.    console.log('外部第四个then')
17.})
```
结果为
```js
外部 promise
外部第一个 then
内部 promise
内部第一个 then
外部第二个 then
外部第三个 then
外部第四个 then
内部第二个 then
```

这里要注意两个点。
```js 
以下代码皆为了便于理解思想 并未真实源码

1. return Promise.resolve()
首先需要了解
(1) Promise.resolve = () => new Promise((resolve) => resolve())
(2) Promise.prototype.then = () => { /* ... */ return new Promise(/* ...*/) { /* ... */ }  }
当 then 的返回值为 具有 then 方法的对象时（也就是这里的 Promise.resolve ）时，
then 所创建的 promise0 的状态（pending/resolve/reject）受
具有 then 方法的对象时（也就是这里的 Promise.resolve ）所创建的 promise1 控制。
而且，就算 promise1 的状态为 resolve ，promise0 也不会立即 resolve。
因为 promise 要保证过程全是异步的 不会有时而同步时而异步的情况出现!!!
所以这里的 return Promise.resolve() 可以理解为

microTask() => {
    promise1.then(() => {
        promise0.resolve()
    })
}

2. 执行顺序
当处理到 '内部第一个then' 时，是将回调函数抛到了队列中。而后 '外部第一个then' 处理完成，
'外部第一个then' 所生成的 Promise 状态变为 resolve 且将 '外部第二个then' 的回调函数
抛到队列中。反复进行 Event Loop。
```

**这一条后续整合成一篇文章！**
需要注意的是 `Promise/A+` 实现 跟 `原生 Promise` 实现不一致。
特指： `promise.js` 跟  `es6 Promise`。
参考 [https://juejin.im/post/6844903987183894535](https://juejin.im/post/6844903987183894535)

[https://segmentfault.com/q/1010000016147496](https://segmentfault.com/q/1010000016147496)