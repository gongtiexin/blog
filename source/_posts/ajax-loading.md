---
title: ajax相应时间过快，页面loading闪烁？
date: 2020-04-01 12:56:00

categories:
  - 技术

tags:
  - ajax
  - loading
---

## 背景

现在绝大部分异步请求都有如下类似的套路代码

```
loading = true;
ajax().finally(() => {
  loading = false;
});
```

都 0202 年了，高速的网络会导致 loading 出现闪烁情况

## Promise.all 的解决方案

假设我有两个 ajax 请求时间分别是`50ms`和`150ms`，我现在希望不管是`50ms`还是`150ms`，loading 动画都有一个比较完整的展示时间

这种情况只需要用一个延迟的 Promise.resolve()，通过 Promise.all 方法去拉长响应时间

```
const delay = ms => new Promise((resolve, _) => setTimeout(resolve, ms));

loading = true;
Promise.all([ajaxPromise, delay(300)])
  .then(handleSuccess)
  .catch(handleError)
  .finally(() => {
    loading = false;
  });
```

这种解决方案对于响应快的情况有点本末倒置的感觉

## Promise.race 的解决方案

现在我希望响应时间超过`100ms`的情况才展示 loading 动画

这种情况只需要用一个延迟的 Promise.reject()，通过 Promise.race 方法去和 ajax 竞态

```
const timeout = ms =>
  new Promise((_, reject) => setTimeout(() => reject(Symbol.for('timeout')), ms));

Promise.race([ajaxPromise, timeout(100)])
  .then(handleSuccess)
  .catch(err => {
    if (Symbol.for('timeout') === err) {
      loading = true;
      return ajaxPromise
        .then(handleSuccess)
        .catch(handleError)
        .finally(() => {
          loading = false;
        });
    }
    return handleError(err);
  });
```

当我的响应时间为`101ms`的时候，闪烁还是无法避免的

## Promise.all 和 Promise.race 的解决方案

现在我希望响应时间小于`100ms`时不展示 loading 动画，大于`100ms`时展示`300ms`的 loading 动画时间

```
const timeout = ms =>
  new Promise((_, reject) => setTimeout(() => reject(Symbol.for('timeout')), ms));

const delay = ms => new Promise((resolve, _) => setTimeout(resolve, ms));

const request = ({ config, target, timeoutTime = 100, delayTime = 300 }) => {
  // 返回promise的ajax请求
  const promise = axios(config);

  const startLoading = target => {
    if (!target) {
      return;
    }
    // startLoading
  };

  const endLoading = () => {
    // endLoading
  };

  const handleSuccess = data => {
    // 兼容Promise.all和Promise.race不同的返回值
    const response = Array.isArray(data) ? data[0] : data;
    // 处理成功的情况
    return Promise.resolve(response.data);
  };

  const handleError = ({ response }) => {
    // 处理失败的情况
    return Promise.reject(response);
  };

  return Promise.race([promise, timeout(timeoutTime)])
    .then(handleSuccess)
    .catch(err => {
      if (Symbol.for('timeout') === err) {
        startLoading(target);
        return Promise.all([promise, delay(delayTime)])
          .then(handleSuccess)
          .catch(handleError)
          .finally(() => {
            endLoading();
          });
      }
      return handleError(err);
    });
};
```

timeoutTime 和 delayTime 可以根据自己的网站调整
