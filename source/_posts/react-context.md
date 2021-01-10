---
title: React 用 Context 和 Hook 实现全局状态管理
date: 2021-01-10 12:32:55

categories:
  - 技术

tags:
  - react
  - hook
  - store
---

## 简介

React 默认没有自己的全局状态管理，需要依赖第三方实现，比较常用的有 redux，mobx 等。随 16.8.0 的更新，context + hook 完全能实现全局状态管理功能。

## 简单的全局状态管理

使用`React.createContext`创建一个`Context`对象，每个`Context`对象都会返回一个`Provider`组件，然后在函数组件里面使用`useContext`就能订阅了，当`context`改变时，所有订阅了的组件都会触发更新。

要实现全局状态管理，自然而然想到的是在所有组件的最外层定义一个`Context`，让所有组件都能消费。

```
const useCount = () => {
  const [count, setCount] = useState(0);

  const increment = useCallback(() => {
    setCount((value) => value + 1);
  }, []);

  const decrement = useCallback(() => {
    setCount((value) => value - 1);
  }, []);

  return {
    count,
    increment,
    decrement,
  };
};

const StoreContext = React.createContext(null);

const Count = () => {
  const { count } = useContext(StoreContext);

  console.log('Count render');

  return <span>{count}</span>;
};

const Btn = () => {
  const { increment, decrement } = useContext(StoreContext);

  console.log('Btn render');

  return (
    <>
      <button onClick={increment}>+1</button>
      <button onClick={decrement}>-1</button>
    </>
  );
};

const Example = () => {
  return (
    <StoreContext.Provider value={useCount()}>
      <Count />
      <Btn />
    </StoreContext.Provider>
  );
};
```

## 优化渲染

### 拆分`context`

最开始提到，当`context`改变时，所有订阅了的组件都会触发更新，但是`Btn`组件并不需要更新，因此可以将`context`拆分成两个，使`Btn`不更新`Count`更新

```
const useCount = () => {
  const [count, setCount] = useState(0);
  const [count2, setCount2] = useState(0);

  const increment = useCallback(() => {
    setCount((value) => value + 1);
  }, []);

  const decrement = useCallback(() => {
    setCount((value) => value - 1);
  }, []);

  return {
    count,
    count2,
    increment,
    decrement,
  };
};

const StateContext = React.createContext(null);
const DispatchContext = React.createContext(null);

const Count = () => {
  const { count } = useContext(StateContext);

  console.log('Count render');

  return <span>{count}</span>;
};

const Count2 = () => {
  const { count2 } = useContext(StateContext);

  console.log('Count2 render');

  return <span>{count2}</span>;
};

const Btn = () => {
  const { increment, decrement } = useContext(DispatchContext);

  console.log('Btn render');

  return (
    <>
      <button onClick={increment}>+1</button>
      <button onClick={decrement}>-1</button>
    </>
  );
};

const StoreProvider = ({ children }) => {
  const { count, count2, increment, decrement } = useCount();
  const dispatchRef = useRef({ increment, decrement });

  return (
    <StateContext.Provider value={{ count, count2 }}>
      <DispatchContext.Provider value={dispatchRef.current}>{children}</DispatchContext.Provider>
    </StateContext.Provider>
  );
};

const Example = () => {
  return (
    <StoreProvider>
      <Count />
      <Count2 />
      <Btn />
    </StoreProvider>
  );
};
```

### PubSub

当点击任意`button`时会发现，`Count`和`Count2`同时更新了。实际上`Count`组件只想在`count`变化才更新，上面的实现就满足不了需求了。

换一种思路，当组件有依赖的值，就用`useState`根据依赖项创建一个`局部state`，当`context`改变的时候就通知组件，组件判断当值改变时就`setState`。当组件没有依赖的值就直接消费`context`

所以，需要两个`context`，一个提供给所有不需要依赖项的组件消费，一个是发布者，在`context`改变的时候通知组件，组件就是订阅者。

```
// store工厂函数
const createStore = (model) => {
  const AllContext = createContext({});
  const DepsContext = createContext({});

  return {
    useStore: (deps) => {
      const sealedDeps = useMemo(() => deps, []);

      if (!sealedDeps || sealedDeps.length === 0) {
        return useContext(AllContext);
      }

      const container = useContext(DepsContext);
      const [state, setState] = useState(container.state);
      const prevDepsRef = useRef(deps(container.state));

      useEffect(() => {
        const subscription = () => {
          const prev = prevDepsRef.current;
          const curr = deps(container.state);
          if (!isEqual(prev, curr)) {
            setState(container.state);
          }
          prevDepsRef.current = curr;
        };

        container.subscribe(subscription);

        return () => {
          container.unSubscribe(subscription);
        };
      }, []);

      return state;
    },
    Provider: ({ children }) => {
      const containerRef = useRef(new PubSub());
      const state = model();

      containerRef.current.state = state;

      useEffect(() => {
        containerRef.current.publish();
      });

      return (
        <AllContext.Provider value={state}>
          <DepsContext.Provider value={containerRef.current}>{children}</DepsContext.Provider>
        </AllContext.Provider>
      );
    },
  };
};
```

```
// 发布订阅
class PubSub {
  state = null;

  subscriptions = [];

  publish() {
    this.subscriptions.forEach((subscription) => subscription());
  }

  subscribe(subscription) {
    this.subscriptions.push(subscription);
  }

  unSubscribe(subscription) {
    const index = this.subscriptions.findIndex((item) => item === subscription);
    if (index > -1) {
      this.subscriptions.splice(index, 1);
    }
  }
}
```

```
// 合并多个sotre
const StoreProvider = ({ value, children }) => {
  const ProviderUnion = useMemo(
    () => ({ children: accChildren }) =>
      value.reduceRight((acc, { Provider }) => <Provider>{acc}</Provider>, accChildren),
    []
  );
  return <ProviderUnion>{children}</ProviderUnion>;
};
```

```
const useExampleCount = () => {
  const [count, setCount] = useState(0);
  const [count2, setCount2] = useState(0);

  const increment = useCallback(() => {
    setCount((value) => value + 1);
  }, []);
  const decrement = useCallback(() => {
    setCount((value) => value - 1);
  }, []);

  const increment2 = useCallback(() => {
    setCount2((value) => value + 2);
  }, []);
  const decrement2 = useCallback(() => {
    setCount2((value) => value - 2);
  }, []);

  return { count, count2, increment, decrement, increment2, decrement2 };
};

const exampleCountStore = createStore(useExampleCount);

const Count = () => {
  const { count } = exampleCountStore.useStore((m) => [m.count]);

  console.log('Count render');

  return <span>{count}</span>;
};

const Btn = () => {
  const { increment, decrement } = exampleCountStore.useStore((m) => []);

  console.log('Btn render');

  return (
    <>
      <button onClick={increment}>increment</button>
      <button onClick={decrement}>decrement</button>
    </>
  );
};

const Count2 = () => {
  const { count2 } = exampleCountStore.useStore((m) => [m.count2]);

  console.log('Count2 render');

  return <span>{count2}</span>;
};

const Btn2 = () => {
  const { increment2, decrement2 } = exampleCountStore.useStore((m) => []);

  console.log('Btn2 render');

  return (
    <>
      <button onClick={increment2}>increment2</button>
      <button onClick={decrement2}>decrement2</button>
    </>
  );
};

const Example = () => {
  return (
    <>
      <Count />
      <Btn />
      <Count2 />
      <Btn2 />
    </>
  );
};
```
