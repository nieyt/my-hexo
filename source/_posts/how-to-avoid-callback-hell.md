---
title: 异步编程，如何避免回调地狱
date: 2017-06-20 16:36:03
tags: [es6,es7,javascript,nodejs]
categories: 前端
id: 6
---
在js编程中，很容易遇到异步的情况。比如前端的发送ajax请求、定时器，后端nodejs的数据库读取，文件读取等。传统的异步编程方式，都是基于回调函数的。但由于回调函数嵌套过多，会造成代码复杂难以维护，且难以捕捉到代码异常，于是，新版本的ECMAScript针对这一情况有了改进。
<!-- more -->
#### 什么是回调地狱？
```js
exports.findList = function (findObj,callback) {
    Info.find(findObj,function (err,info) {
        if (err) {
            callback(err);
        }
        info.forEach((f, i) => {
             f.image=f.images.split(',')[0];
             User.findById(f.author_id,function (error,user) {
                 f.author_name=user.name;
                 f.author_avatar=user.avatar;
                 if(i==info.length-1){
                     callback(error,info);
                 }
             })
        })
    })
}
```

这是我毕设中的一段node代码的原始版本。其中`Info.find`、`User.findById`等操作是异步的，执行时不会立马得到结果，在进行数据库操作的时候，js引擎不需要等待它，而是继续执行后面的代码。如果想要在它结束后进行某种操作，就必须把代码写在它的回调函数中。在回调函数中继续进行异步操作，执行回调，即为回调嵌套，如果涉及到多个回调嵌套，就陷入回调地狱了。

此外，从上面代码可以看出，我们需要在所有`User.findById`执行完后，再执行`callback`，于是采用了计数的办法，当最后一个`User.findById`完成后，`callback`执行。但从某种意义上来看这种方法有些丑陋，代码的可读性不高。


#### 下面我将用以下几种方式对这段代码进行改造：
* `Promise`
* `async/await`
* `co/Generator`

**改造开始↓↓↓↓↓↓**

##### ES6中的`Promise`对象
```js
exports.findlist=function (findObj,callback) {
    // part 1 ->
    const promise1 = new Promise((res, rej) => {
        Info.find(findObj,function (err,info) {
            if(err) {
                rej(err);
                return;
            }
            res(info);
        }
    })
    promise1.then(info => {
        // part 2 ->
        const promises = info.map(f => new Promise((res,rej) =>{
            f.image=f.images.split(',')[0];
            User.findById(f.author_id,function (error,user) {
                if(error){
                    rej(error);
                    return;
                }
                f.author_name = user.name;
                f.author_avatar = user.avatar;
                res(f);
            })
        }));
        Promise.all(promises)
            .then(values =>{
                // 成功的时候，这个 values 是所有 info 对象，作为一个数组返回出来，而不是某一个
                callback(null, values);
            })
            .catch(error =>{
                // 注意这里 error 是第一个失败 error，不是所有的 error
                callback(error);
            })
    }).catch(err => {
        callback(err);
    })
}
```
* part1
将第一个异步操作转换成Promise实例。由于这里`.then`内部还有大量的异步操作，无法返回单个的Promise实例，不方便再使用Promise的链式写法，代码量略为庞大。

* part2
代码将所有的`User.findById`都加入了一个数组，数组的每一项都是一个`Promise`实例，然后用`Promise.all().then()`，当数组中所有的`Promise`实例完成后，`.then()`中的函数才执行。
    
##### ES7中的`async/await`
* way1
```js
exports.findList = async function (findObj, callback) {

    const promise1 = new Promise((res, rej) => {
        Info.find(findObj,function (err,info) {
            if(err) {
                rej(err);
                return;
            }
            res(info);
        }
    });

    try{
        // way2将重写这部分 ->
        const info = await promise1;
        const promises = info.map(f => new Promise((res,rej) => {
            f.image=f.images.split(',')[0];
            User.findById(f.author_id,function (error,user) {
                if(error){
                    rej(error);
                    return;
                }
                f.author_name = user.name;
                f.author_avatar = user.avatar;
                res(f);
            })
        }));
        const values = await Promise.all(promises);
        // await 后面是一个 promise 对象，但不用写 then，其返回值 values 即为传入 res 的参数。
        callback(null, values);

    } catch (error){
        callback(error);
    }
}
```
* way2
```js
// 仅列出与way1不同部分的代码，即try代码块中的代码

const info = await promise1;
const values;
info.forEach(f => {
    let result = await new Promise((res,rej) =>{
        f.image=f.images.split(',')[0];
        User.findById(f.author_id,function (error,user) {
            if(error){
                rej(error);
                return;
            }
            f.author_name = user.name;
            f.author_avatar = user.avatar;
            res(f);
        })
    })
    values.push(result);
    callback(null, values);
})

```
way1的写法执行效果同`Promise.all().then()...`,Promise.all()是并行执行的，也就是说，所有的`User.findById`是串联调用，并行执行的，执行时间总和为最慢的那个的执行时间。而way2的写法，所有的`User.findById`是串联调用且串联执行的，执行时间总和为所有的单个`User.findById`的执行时间总和。

使用`async/await`写异步代码，就像写同步代码一样，`await`后面跟的是`Promise`对象，在`async`函数中，遇到`await`，函数将会暂停执行，在等待`await`后的`Promise`执行完成后，继续执行`async`函数并返回结果。另外，`async`函数会返回一个`Promise`对象。当在`async`函数内部返回一个值时，`Promise`的`resolve`方法会负责传递这个值；当`async`函数抛出异常时，`Promise`的`reject`方法也会传递这个异常值。

##### [co函数](https://github.com/tj/co)，实现原理为ES6中的`Generator`

`co`函数出现在ES7`async/await`还未被普遍支持之前，代码风格类似于`async`，使用同步的方式书写异步代码，原理为利用在`Generate`函数中的`next`函数中传参，可以赋值给上一个`yield`的返回值，实现获取`Promise`对象的`resolve`传值。由于目前`async`函数已得到node和浏览器较新版本，以及babel库的支持，所以项目中很少再需要使用`co`函数库了。这里我就不再继续写一遍代码了，写法可参见[使用Generator解决回调地狱](http://www.alloyteam.com/2015/04/solve-callback-hell-with-generator/)，原理篇可参见[yield方式的异步代码什么原理？（刨根向）](https://cnodejs.org/topic/577494649ea5dce84ff27cba)。

#### 总结
`async/await`函数相对于`promise.then()`来说更加简洁，更加适合大部分的异步场景，但是在有大量异步操作，需要并行执行节省时间时，还是`Promise.all()`更加耐打，毕竟它天生就是为了支持这种场景的。另外，在一般代码量较少、嵌套不深的情况下，我认为还是直接使用回调函数更加适合，因为采用其他几种方法，都会耗费更多的执行时间，且增大代码量。以上具体哪一种解决方式更快，我们可以在各个平台自己试一试。选择哪一种解决方式，也要根据不同的代码场景来决定。