---
title: promise 简单实现
date: 2018-01-27 19:48:37
categories: promise
tags: [promise]
---
Promise 简单实现，阅读[深刻理解Promise系列](https://www.jianshu.com/p/a89a8e5d6636), 先上代码，个人理解后续。
<!-- more -->

```
function Promise(fn) {
    let cbs = [];
    let state = 'pending';
    let value = null;

    function resolve(newValue) {
        if(newValue && (typeof(newValue) === 'object') && (typeof(newValue) === 'function')){
            let then = newValue.then;
            then.call(newValue, resolve)
        }
        state = 'resolved'
        value = newValue
        setTimeout(function(){
            console.log('in resolve ')
            
            cbs.map(_cb => handle(_cb))
        }, 0)
    }

    function handle(cb){
        if(state == 'pending'){
            console.log('in then: pending')
            cbs.push(cb)
            return;
        }
        let ret = cb.cb(value);
        if(ret){
            cb.resolve(ret)
        } else {
            cb.resolve(value)
        }
    }

    this.then = function(cb){
        return new Promise(function(resolve, reject){
            handle({
                cb: cb || null,
                resolve: resolve
            })
        })
    }

    fn(resolve)
}

new Promise(function(resolve, reject){
    let a = 11;
    console.log('111 ---> ', a)
    setTimeout(()=>{
        resolve(a)
    }, 0)
}).then(function(value){
    setTimeout(()=>{

        console.log('2222 ----> ', value)
    }, 0)
}).then(function(value){
    console.log('3333 ----> ', value)
})

```

> 参考： [深刻理解Promise系列](https://www.jianshu.com/p/a89a8e5d6636)
 