# Redux中间件
## 1.预备知识
1.函数式编程

2.连续箭头函数与柯里化 https://zhuanlan.zhihu.com/p/26794822

3.https://stackoverflow.com/questions/35411423/how-to-dispatch-a-redux-action-with-a-timeout/35415559#35415559

4.createStore()内部实现
```js
function createStore(reducer) {
    let state = null;
    const listeners = [];
    //设置监听，将需要监听的事件加入listeners数组
    const subscribe = (listener)=>listeners.push(listener);
    //获取reducer中的state
    const getState = ()=> state;
    //执行刷新操作
    const dispatch = (action) => {
        //根据action返回reducer中相应的state, 在connect函数中通过store.getState()获取最新state
        state = reducer(state, action);
        //执行监听
        listeners.forEach((listener)=>listener());
    }
    //参数为null时，返回reducer中的初始化state
    dispatch({}); 
    return {getState, dispatch, subscribe}
}
```

## 2.概念及原理
概念：中间件是一个函数，对store.dispatch方法进行改造，在发出Action和执行Reducer之间添加其他功能。
原理：
```js
//createStore函数参数
createStore(reducer, [preloadedState], enhancer)
//StoreCreator函数签名，该函数创建的是一个Store
Type StoreCreator = (reducer: Reducer, initialState: ?State) => Store
//enhancer函数签名，该函数是一个高阶函数，加工StoreCreator后返回一个新的StoreCreator
//enhancer就是applyMiddleware返回的函数
Type enhancer = (next: StoreCreator) => StoreCreator
1.applyMiddleware()
export default function applyMiddleware(...middlewares) {
  //这个返回函数就是enhancer
  return (createStore) => (reducer, preloadedState, enhancer) => {
    const store = createStore(reducer, preloadedState, enhancer)
    let dispatch = store.dispatch
    let chain = []
    //根据action
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    //注册中间件调用链，中间件最外层函数接受参数都是(getState, dispatch)

    chain = middlewares.map(middleware => middleware(middlewareAPI))
    //compose 函数起到代码组合的作用：compose(f, g, h)(...args) 效果等同于 f(g(h(...args)))。从此也可见：所有的中间件最二层函数接收的参数为 dispatch，一般我们在定义中间件时这个形参不叫 dispatch 而叫 next，是由于此时的 dispatch 不一定是原始 store.dispatch，有可能是被包装过的新的 dispatch。
   dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```


