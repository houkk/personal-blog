---
title: promise 简单实现
date: 2018-01-27 19:48:37
categories: promise
tags: [promise]
---
Promises/A+规范的简单实现, 纯代码
<!-- more -->

看了一下其他的, 感觉和我的不太一样啊 😂 。。。。。。 <br>
有时间仔细研究下怎么回事

``` js
const states = {
  pending: 'Pending',
  resolved: 'Resolved',
  rejected: 'Rejected'
}

class MyPromise {
  constructor (fn) {
    this.state = states.pending

    this.thenHandlers = {
      onResolve: null,
      onReject: null,
      childResolve: null,
      childReject: null
    }

    try {
      fn(this._resolve.bind(this), this._reject.bind(this))
    } catch (error) {
      // 处理 fn 报错
      this._reject(error)
    }
  }

  _resolve (value) {
    if (value === this) throw new Error('TypeError')
    if (this.state === states.rejected) return

    // value is object(... or promise) function
    if (value && (typeof value === 'object' || typeof value === 'function')) {
      const then = value.then
      if (typeof then === 'function') {
        // 遵循 value-promise 的状态流转, 将当前的 this.thenHandler 处理放在 value-then 中处理
        //! 一定要不要忘记 _reject 函数, 不然当 value-promise 中报错且没有 catch 函数时, 将会无法捕捉
        then.call(value, this._resolve.bind(this), this._reject.bind(this))
        return
      }
    }

    setTimeout(() => {
      // 确保 onResolve, onReject 处理都在下一个时间循环中
      this.state = states.resolved
      this._handle(
        this.thenHandlers.onResolve,
        this.thenHandlers.childResolve,
        value
      )
    }, 0)
  }

  _handle (cb, childCb, res) {
    if (!childCb) return // promise 链到此为止
    if (!cb || typeof cb !== 'function') {
      // 没有提供 onResolve/onReject, 或者不是函数, 值穿透
      childCb(res)
      return
    }

    try {
      const res2 = cb(res)
      this.thenHandlers.childResolve(res2)
    } catch (e) {
      this.thenHandlers.childReject(e)
    }
  }

  _reject (err) {
    if (this.state === states.resolved) return

    err = err || new Error('promise rejected')

    setTimeout(() => {
      // 确保 状态变更, onResolve, onReject 处理都在下一个事件循环中, 方便 then catch 的赋值
      this.state = states.rejected
      // 注意这个 onReject
      // 我们通常使用 then (onResolve, onReject), 是不会提供 onReject 函数的
      // 这种情况, 在 _handle 中, 就直接跳过处理了
      // console.log('this thenhandler onRject ====> ', this.thenHandlers)
      this._handle(
        this.thenHandlers.onReject,
        this.thenHandlers.childReject,
        err
      )
    }, 0)
  }

  catch (onError) {
    // 处理报错, 本质还是 then, 仅提供前面流程中对 reject 抛错的接收函数
    // 注意: onError 接收到的是 err 信息, 如果在 onError 中不继续 throw 的话, 将会进入 resolve 流程
    return this.then(null, onError)
  }

  then (onResolve, onReject) {
    // 要求返回 promise
    return new MyPromise((resolve, reject) => {
      // 既然返回了新实例, 那就在当前实例 resolve/reject 后调用子实例的 resolve/reject
      // 即实例嵌套, 当前实例调用子实例的 (child)resolve, (child)reject 函数
      this.thenHandlers.onResolve = onResolve
      this.thenHandlers.onReject = onReject

      this.thenHandlers.childResolve = resolve
      this.thenHandlers.childReject = reject
    })
  }

  finally (fn) {
    if (!fn || typeof fn !== 'function') return this.then()
    // 和 callback 类似, 不同的是不知道状态, 不接受参数, 保留原输出或报错
    return this.then(
      v => {
        return MyPromise.resolve(fn()).then(() => v)
      },
      e => {
        return MyPromise.resolve(fn()).then(() => {
          throw e
        })
      }
    )
  }

  static resolve (value) {
    // if (value instanceof MyPromise) return value
    if (typeof value === 'object' && typeof value.then === 'function') {
      const then = value.then
      return new MyPromise(resolve => {
        then(resolve)
      })
    }

    return new MyPromise(resolve => (value ? resolve(value) : resolve()))
  }

  static reject (value) {
    // if (value instanceof MyPromise) return value
    if (typeof value === 'object' && typeof value.then === 'function') {
      // thenable
      const then = value.then
      return new MyPromise((resolve, reject) => {
        then(reject)
      })
    }

    value = value || new Error('promise rejected')
    return new MyPromise((resolve, reject) => reject(value))
  }
}
```

> 参考： [【翻译】Promises/A+规范](https://www.ituring.com.cn/article/66566)
 