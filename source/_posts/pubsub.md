---
title: 发布订阅模式
date: 2019-03-26 16:37:07

categories:
  - 技术

tags:
  - 设计模式
---

## [维基百科][1]中的定义

在软件架构中，发布-订阅是一种消息范式，消息的发送者（称为发布者）不会将消息直接发送给特定的接收者（称为订阅者）

而是将发布的消息分为不同的类别，无需了解哪些订阅者（如果有的话）可能存在

同样的，订阅者可以表达对一个或多个类别的兴趣，只接收感兴趣的消息，无需了解哪些发布者（如果有的话）存在

```
 ╭─────────────╮                 ╭───────────────╮   Fire Event   ╭──────────────╮
 │             │  Publish Event  │               │───────────────>│              │
 │  Publisher  │────────────────>│ Event Channel │                │  Subscriber  │
 │             │                 │               │<───────────────│              │
 ╰─────────────╯                 ╰───────────────╯    Subscribe   ╰──────────────╯
```

## 实现思路

1. 作为发布者提供 subscribe 方法给订阅者

2. 发布者提供了订阅的方法后应该将这些订阅者都存起来，记录以便日后给他们推送消息

3. 作为发布者要有 publish 方法推送消息

4. 订阅者如何订阅消息或者事件，订阅者还可以通过 unsubscribe 取消订阅

## 代码实现

```
class PubSub {
  topics = {};

  subUid = -1;

  subscribe = (topic, func) => {
    const token = (this.subUid += 1).toString();
    // 如果没有订阅过此类消息，创建一个缓存列表
    if (!this.topics[topic]) {
      this.topics[topic] = [];
    }
    this.topics[topic].push({
      token,
      func,
    });
    return token;
  };

  unsubscribe = token => {
    Object.values(this.topics).forEach(value => {
      value.forEach((item, idx) => {
        if (item.token === token) {
          value.splice(idx, 1);
          console.log(`unsubscribe: ${token}`);
        }
      });
    });
  };

  publish = (topic, ...args) =>
    this.topics[topic]?.forEach(({ func }) => {
      func(args);
    });
}

// 一个简单的消息记录器，记录通过我们收到的任何主题和数据
const messageLogger = msg => {
  console.log(`Logging: ${msg}`);
};

const pubSub = new PubSub();

const subscription1 = pubSub.subscribe("friend1", messageLogger);
const subscription2 = pubSub.subscribe("friend2", messageLogger);

pubSub.publish("friend1", "hello, friend1!");
pubSub.publish("friend2", "hello, friend2!");

pubSub.unsubscribe(subscription1);

pubSub.publish("friend1", "goodbye, friend1!");
pubSub.publish("friend2", "goodbye, friend2!");

// console
// Logging: hello, friend1!
// Logging: hello, friend2!
// unsubscribe: 0
// Logging: goodbye, friend2!
```

[1]: https://zh.wikipedia.org/wiki/%E5%8F%91%E5%B8%83/%E8%AE%A2%E9%98%85
