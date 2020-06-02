# Redux 源码(TS 版)

> 个人认为和 JS 版的逻辑方面没有太多差异, 主要是使用 TS 优化了代码, 比如`createStore`方法, JS 实现是内部 if 判断是否传入了 enhancer, 而 TS 版则是使用了函数重载的方式.

## API 复习

嗯哼, 还是先复习下 API 吧~

- `createStore(reducer, initialState, enhancer)`
- `store`
  - `getState()`
  - `dispatch(action)`
  - `subscribe(listener)`
  - `replaceReducer(nextReducer)`
- `combineReducers(reducers)`
- `applyMiddleware(...middleware)`
- `bindActionCreators(actionCreators, dispatch)`
- `compose(...functions)`

## 类型定义

首先来看一下, TS 必不可少的类型定义,

### action

```typescript
export interface Action<T = any> {
  type: T;
}

export interface AnyAction extends Action {
  [extraProps: string]: any;
}

export interface ActionCreator<A> {
  (...args: any[]): A;
}

export interface ActionCreatorsMapObject<A = any> {
  [key: string]: ActionCreator<A>;
}
```

这么来看, Redux 和 TS 地使用其实还可以进一步使用内置的类型接口来规范, 比如:

```typescript
interface ICreateAction extends Action {
  payload: Payload;
}

const createActionCreator: ActionCreator<ICreateAction> = () => {
  return {
    // ...
  };
};
```

(但是我不太清楚这样是否是有必要的)

> 2020-06-01

其实和 js 版本没有太多区别, 主要体现在类型的规范上, 有一个地方其实我不太明白这样写的意义, 以`compose.ts`为例:

```typescript
export default function compose(): <R>(a: R) => R;

export default function compose<F extends Function>(f: F): F;

/* two functions */
export default function compose<A, T extends any[], R>(
  f1: (a: A) => R,
  f2: Func<T, A>
): Func<T, R>;

/* three functions */
export default function compose<A, B, T extends any[], R>(
  f1: (b: B) => R,
  f2: (a: A) => B,
  f3: Func<T, A>
): Func<T, R>;

/* four functions */
export default function compose<A, B, C, T extends any[], R>(
  f1: (c: C) => R,
  f2: (b: B) => C,
  f3: (a: A) => B,
  f4: Func<T, A>
): Func<T, R>;

/* rest */
export default function compose<R>(
  f1: (a: any) => R,
  ...funcs: Function[]
): (...args: any[]) => R;
```

即 1-4 个函数的情况下, 能够提供完整的类型提示, 超过的话就变 any 了...

还有其他部分类型编程我也没太懂的, 但是看源码还是以逻辑为主吧~
