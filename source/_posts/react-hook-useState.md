---
title: 从零开始实现 React hook useState
date: 2019-12-31 11:11

categories:
  - 技术

tags:
  - react
  - hook
---

## 什么是`hook`

`hook`可以让你在不编写`class`组件的情况下使用`state`以及其他的`React`特性

具体的使用方法可见[官网][0]

## 实现`useState`

我们都知道`useState`会返回一个状态变量和修改它的函数，就像这样

```
const [state, setState] = useState();
```

### 单一状态

那么针对一个变量的情况就简单很多了，只需要用一个全局变量就能简单的实现了

```
const state = null;

const useState = defaultState => {
  if (!state) {
    state = defaultState;
  }
  const setState = newState => {
    state = newState;
  };
  return [state, setState()];
};

const singleState = () => {
  const [count, setCount] = useState(0);
  console.log(count);
  setCount(count + 1);
};
```

### 多个状态

而对于多个状态可能就需要一个格外的变量`num`来标记，我们用的哪一 state

```
const states = {};
let num = 0;

const useState = defaultState => {
  if (!states[num]) {
    states[num] = defaultState;
  }
  const setState = newState => {
    states[num] = newState;
  };
  const result = [states[num], setState];
  num += 1;
  return result;
};

const withHook = renderFunc => {
  return (...args) => {
    // 重置
    num = 0;
    return renderFunc(...args);
  };
};

withHook(function multipleState() {
  const [count1, setCount1] = useState(0);
  const [count2, setCount2] = useState(0);

  console.log(count1, count2);
  setCount1(count1 + 1);
  setCount2(count2 + 2);
});
```

注意每次调用函数组件的时候应该把`num`重置

### 多个函数

对于多个函数组件的话，函数相互调用会打乱我们`num`的顺序，那应该怎么保持有序执行呢？这里就要用一个`stack`，像我们平时调试程序的时候会在浏览器控制台里面看到一个`call stack`调用栈，每次运行一个函数就把它入栈，执行完毕就出栈，这样就保证了顺序

```
const contextStack = [];

const useState = defaultState => {
  const context = contextStack[contextStack.length - 1];
  const { num, states } = context;

  if (!states[num]) {
    states[num] = defaultState;
  }
  const setState = newState => {
    states[num] = newState;
  };
  const result = [states[num], setState];
  context.num += 1;
  return result;
};

const withHook = renderFunc => {
  return (...args) => {
    contextStack.push({ num: 0, states: [] });
    const result = renderFunc(...args);
    contextStack.pop();
    return result;
  };
};

withHook(function state1() {
  const [count1, setCount1] = useState(0);

  console.log(count1);
  setCount1(count1 + 1);
});

withHook(function state2() {
  const [count2, setCount2] = useState(0);

  console.log(count2);
  setCount2(count2 + 2);
});
```

### 与`React`结合起来

`React`是基于`Fiber`来实现，对于`16.6`以下的版本，我们用类组件来实现

根据上面的代码，我们只需要在`setState`里面去更新函数就可以了，代码如下

```
import { Component } from 'react';

const contextStack = [];

const useState = defaultState => {
  const context = contextStack[contextStack.length - 1];
  const { component } = context;
  const states = Object.values(component.state || {});

  if (component.firstRender) {
    states[context.num] = defaultState;
  }

  const setState = num => val => {
    component.setState({ [num]: val });
  };

  const result = [states[context.num], setState(context.num, component)];
  context.num += 1;
  return result;
};

const withHook = renderFunc => {
  const HooksComponent = class extends Component {
    constructor(props) {
      super(props);
      // 标记组件是否是第一次渲染，对于useState初始值的优化
      this.firstRender = true;
    }

    componentDidMount() {
      this.firstRender = false;
    }

    render() {
      contextStack.push({ num: 0, component: this });
      const result = renderFunc(this.props);
      contextStack.pop();
      return result;
    }
  };
  HooksComponent.displayName = renderFunc.name;
  return HooksComponent;
};

export { withHook, useState };
```

## 总结

整个实现过程还是比较容易，先从最简单的一个函数一个状态到多个函数多个状态循序渐进

实现过程参考[这篇文章][1]

[0]: https://zh-hans.reactjs.org/docs/hooks-intro.html#___gatsby
[1]: https://zhuanlan.zhihu.com/p/50358654
