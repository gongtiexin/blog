---
title: 实现一个低配版的Vuex或Mobx
date: 2018-03-01 14:52:41

categories:
  - 技术

tags:
  - defineProperty
---

## 实现原理

vuex 或者 mobx 都是通过 Object.defineProperty 来实现数据劫持。当我们访问或设置对象的属性的时候，都会触发相对应的函数，通过劫持对象属性的 setter 和 getter 操作，来进行双向绑定。其主要由下面三个部分组成:

- Observer 负责数据劫持，把所有的属性定义成被观察者，达到对数据进行观测的目的

- Watcher 数据的观察者，在数据发生变化之后执行的相应的回调函数

- Dep 每一个 observer 会创建一个 Dep 实例，实例在 get 数据的时候为数据收集 watcher，在 set 的时候执行 watcher 内的回调方法

## 代码实现

```
// Dep
class Dep {
  sub = [];
  addSub = watcher => this.sub.push(watcher);
  notify = () => this.sub.forEach(watcher => watcher.fn());
}

// Observer
class Observer {
  constructor(obj, key, value) {
    const dep = new Dep();
    let backup = value;
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get() {
        if (Dep.target) {
          // 存储依赖
          dep.addSub(Dep.target);
        }
        return backup;
      },
      set(newVal) {
        if (backup === newVal) return;
        backup = newVal;
        // 执行依赖
        dep.notify();
      },
    });
  }
}

// Watcher
class Watcher {
  constructor(data, k, fn) {
    this.fn = fn;
    Dep.target = this;
    // 触发get方法
    data[k];
    Dep.target = null;
  }
}

const observable = obj => {
  if (Object.prototype.toString.call(obj) === "[object Object]") {
      // 递归
    Object.entries(obj).forEach(([key, value]) => {
      if (Object.prototype.toString.call(value) === "[object Object]") {
        observable(value);
      }
      new Observer(obj, key, value);
      new Watcher(obj, key, (v, key) => {
        console.log("你修改了数据");
        // ...你想要的操作
      });
    });
  }
};
```
