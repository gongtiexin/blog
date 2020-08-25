---
title: 手写一个Promise
date: 2020-08-25 16:38:45

categories:
  - 技术

tags:
  - promise
---

## Promise/A+

下面是一个简单的`Promise`的实例

```
const promise1 = new Promise((resolve, reject) => {
    console.log(1);
    resolve(2);
    console.log(3);
    reject(4);
});

const promise2 = new Promise((resolve, reject) => {
    reject(5);
});

console.log(6);

promise1.then(console.log, console.log);
promise2.then(console.log, console.log);
```

它对应的输出结果为

```
1
3
6
2
5
```

可以总结出`Promise`主要特征如下所示

1. `new Promise`会返回一个`Promise`对象
2. `new Promise`时，会传入一个`executor`函数，`executor`接受`resolve`和`reject`两个回调函数
3. `executor`会同步执行，`resolve`和`reject`会异步执行
4. `Promise`初始状态为`pending`，只能从`pending`修改为`fulfilled`或者`rejected`，修改以后无法再次修改
5. `then`会返回一个`Promise`对象

再根据[`Promise/A+`][0]规范，可以分析出`Promise`的基本特征如下所示(`promise`表示`Promise`的实例)

1. `promise`只有三个状态，pending，fulfilled，rejected
2. `new Promise`时,需要传入一个`executor`，`executor`会立即执行
3. `executor`接受`resolve`和`reject`两个回调函数
4. `promise`初始状态为`pending`
5. `promise`有一个`value`属性，用来保存成功后返回的值，可以是`undefined/thenable/promise`
6. `promise`有一个`reason`属性,用来保存失败后返回的值
7. `promise`只能从`pending`修改为`fulfilled`或者`rejected`，修改以后无法再次修改
8. `promise`有一个`then`方法，接收 `Promise`成功的回调`onFulfilled`和`Promise`失败的回调`onRejected`

## Promise 基本结构

下面是一个 Promise 的基本结构

```
const PromiseStatusEnum = {
    PENDING: 'pending',
    FULFILLED: 'fulfilled',
    REJECTED: 'rejected'
};

class MyPromise {
    constructor(executor) {
        this.status = PromiseStatusEnum.PENDING;
        this.value = null;
        this.reason = null;
        this.resolveQueue = [];
        this.rejectQueue = [];
    }

    then = (onFulfilled, onRejected) => {};

    catch = (onRejected) => {};

    finally = (callback) => {};

    static resolve = (value) => {};

    static reject = (error) => {};

    static all = (promiseArr) => {};

    static race = (promiseArr) => {};
}
```

### executor

`executor`接受两个异步执行的回调函数`resolve`和`reject`，通过箭头函数绑定当前实例的`this`，通过`setTimeout`异步执行

```
const resolve = (value) => {
    const run = () => {};
    setTimeout(run);
};

const reject = (error) => {
    const run = () => {};
    setTimeout(run);
};
```

`run`主要负责

1. 判断当前的状态是否为`pending`，并对其修改
2. 记录成功或者失败的返回值

基本的`executor`如下

```
const resolve = (value) => {
    const run = () => {
        if (this.status === PromiseStatusEnum.PENDING) {
            this.status = PromiseStatusEnum.FULFILLED;
            this.value = value;
        }
    };
    setTimeout(run);
};

const reject = (error) => {
    const run = () => {
        if (this.status === PromiseStatusEnum.PENDING) {
            this.status = PromiseStatusEnum.REJECTED;
            this.reason = error;
        }
    };
    setTimeout(run);
};

try {
    executor(resolve, reject);
} catch (e) {
    reject(e);
}
```

`try catch`主要为了兼容传入的`executor`不符合要求的情况

### then

`then`接受两个回调函数`onFulfilled`和`onRejected`，并返回一个`Promise`对象

```
then = (onFulfilled, onRejected) => {
    // onFulfilled或者onRejected不是函数时，返回当前的值
    typeof onFulfilled !== 'function' ? (onFulfilled = (value) => value) : null;
    typeof onRejected !== 'function' ? (onRejected = (error) => error) : null;

    return new MyPromise((resolve, reject) => {});
};
```

再根据`state`的状态进行不同的操作

1. 当`state === 'fulfilled'`时，执行 onFulfilled
2. 当`state === 'rejected'`时，执行 onRejected
3. 当`state === 'pending'`时，需要将回调函数放入队列中，等待执行
4. 如果再执行回调函数时抛出了异常，那么就会把这个异常作为参数，直接`reject`
5. 回调函数返回的值不是`Promise`，直接`resolve`
6. 回调函数返回的值是`Promise`，调用`then`方法，保证`Promise`会被全部执行

```
then = (onFulfilled, onRejected) => {
    return new MyPromise((resolve, reject) => {
        const resolveFn = (value) => {
            try {
                const x = onFulfilled(value);
                x instanceof MyPromise ? x.then(resolve, reject) : resolve(x);
            } catch (error) {
                reject(error);
            }
        };

        const rejectFn = (error) => {
            try {
                const x = onRejected(error);
                x instanceof MyPromise ? x.then(resolve, reject) : resolve(x);
            } catch (error) {
                reject(error);
            }
        };

        switch (this.status) {
            case PromiseStatusEnum.PENDING:
                this.resolveQueue.push(resolveFn);
                this.rejectQueue.push(rejectFn);
                break;
            case PromiseStatusEnum.FULFILLED:
                resolveFn(this.value);
                break;
            case PromiseStatusEnum.REJECTED:
                rejectFn(this.reason);
                break;
        }
    });
};
```

`resolveQueue`和`rejectQueue`收集的回调，需要在`executor`中的`resolve`和`reject`执行

## 完整代码

```
const PromiseStatusEnum = {
    PENDING: 'pending',
    FULFILLED: 'fulfilled',
    REJECTED: 'rejected'
};

class MyPromise {
    constructor(executor) {
        this.status = PromiseStatusEnum.PENDING;
        this.value = null;
        this.reason = null;
        this.resolveQueue = [];
        this.rejectQueue = [];

        const resolve = (value) => {
            const run = () => {
                if (this.status === PromiseStatusEnum.PENDING) {
                    this.status = PromiseStatusEnum.FULFILLED;
                    this.value = value;

                    while (this.resolveQueue.length) {
                        const callback = this.resolveQueue.shift();
                        callback(value);
                    }
                }
            };

            setTimeout(run);
        };
        const reject = (error) => {
            const run = () => {
                if (this.status === PromiseStatusEnum.PENDING) {
                    this.status = PromiseStatusEnum.REJECTED;
                    this.reason = error;

                    while (this.rejectQueue.length) {
                        const callback = this.rejectQueue.shift();
                        callback(error);
                    }
                }
            };

            setTimeout(run);
        };

        try {
            executor(resolve, reject);
        } catch (e) {
            reject(e);
        }
    }

    then = (onFulfilled, onRejected) => {
        return new MyPromise((resolve, reject) => {
            const resolveFn = (value) => {
                try {
                    const x = onFulfilled(value);
                    x instanceof MyPromise ? x.then(resolve, reject) : resolve(x);
                } catch (error) {
                    reject(error);
                }
            };

            const rejectFn = (error) => {
                try {
                    const x = onRejected(error);
                    x instanceof MyPromise ? x.then(resolve, reject) : resolve(x);
                } catch (error) {
                    reject(error);
                }
            };

            switch (this.status) {
                case PromiseStatusEnum.PENDING:
                    this.resolveQueue.push(resolveFn);
                    this.rejectQueue.push(rejectFn);
                    break;
                case PromiseStatusEnum.FULFILLED:
                    resolveFn(this.value);
                    break;
                case PromiseStatusEnum.REJECTED:
                    rejectFn(this.reason);
                    break;
            }
        });
    };

    catch = (onRejected) => this.then(undefined, onRejected);

    finally = (callback) =>
        this.then(
            (value) => MyPromise.resolve(callback()).then(() => value),
            (error) => MyPromise.resolve(callback()).then(() => MyPromise.reject(error))
        );

    static resolve = (value) =>
        new MyPromise((resolve, reject) =>
            value instanceof MyPromise ? value : new MyPromise((resolve, reject) => resolve(value))
        );

    static reject = (error) => new MyPromise((resolve, reject) => reject(error));

    // 静态all方法
    static all = (promiseArr) => {
        let count = 0;
        let result = [];
        return new MyPromise((resolve, reject) => {
            if (!promiseArr.length) {
                return resolve(result);
            }
            promiseArr.forEach((p, i) => {
                MyPromise.resolve(p).then(
                    (value) => {
                        count++;
                        result[i] = value;
                        if (count === promiseArr.length) {
                            resolve(result);
                        }
                    },
                    (error) => {
                        reject(error);
                    }
                );
            });
        });
    };

    // 静态race方法
    static race = (promiseArr) =>
        new MyPromise((resolve, reject) => {
            promiseArr.forEach((p) => {
                MyPromise.resolve(p).then(
                    (value) => {
                        resolve(value);
                    },
                    (error) => {
                        reject(error);
                    }
                );
            });
        });
}

const promise1 = new MyPromise((resolve, reject) => {
    console.log(1);
    resolve(2);
    console.log(3);
    reject(4);
});

const promise2 = new MyPromise((resolve, reject) => {
    reject(5);
});

console.log(6);

promise1.then(console.log, console.log);
promise2.then(console.log, console.log);
```

它对应的输出结果与之前一致

```
1
3
6
2
5
```

[0]: https://promisesaplus.com/
