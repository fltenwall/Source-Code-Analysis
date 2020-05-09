###### 感慨：react-thunk真的是经典的不能再经典的中间件，它对applyMiddleware的利用真的是登峰造极。

## react-thunk源码分析

#### react-thunk源码：
```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    /**
     *如果ation是函数，就调用这个函数，并且传入dispatch和getState及 extraArgument 为参数
     *extraArgument是给返回函数传入的额外参数
     *后面applyMiddleware函数相当于是对dispatch(action)派发到reducer之前进行了拦截
     *拦截过程中就是经过了一系列中间件处理
     *reducer是个纯函数到达reducer的action必须是一个plain object类型
     *所以后面applyMiddleware函数处理完所有中间件之后返回的是一个普通的对象
     *react-thunk就是充分利用了applyMiddleware函数最后会返回普通对象的原理！！！
     *所以react-thunk的作用就只是将函数类型的action传递进了applyMiddleware！仅此而已！！
     *经典到不能再经典的中间件！
    */
    if (typeof action === 'function') {
     // 这里的dispatch是在applyMiddleware中改写过的
      return action(dispatch, getState, extraArgument);
    }
    
    // 如果传过来的是普通对象，直接调用下一个middleware
    return next(action);
  };
}

//向外暴露
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;

```

#### redux中间件处理函数:applyMiddleware.js源码
```javascript
 // middlewares是传递给applyMiddleware函数的一系列中间件函数
export default function applyMiddleware(...middlewares) {
  //(...args)就是相当于(reducer, preloadedState)
  return createStore => (...args) => {
  // 创建 store
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }
    
    //给middlewareAPI对象挂载getState和dispatch
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    
     // 遍历运行中间件函数，将middlewareAPI作为参数传入
     //chain是保存中间件新返回函数的数组，里面的元素是接收middlewareAPI参数执行后的每个中间件函数
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    
     // 将所有中间件传入到compose中,每个中间件都会用到dispatch
     // 最后返回这个被所有中间件扩展过的dispatch
    dispatch = compose(...chain)(store.dispatch)

     //返回一个普通store对象，它可以被reducer直接处理
     //返回的dispatch是被中间件扩展过的
    return {
      ...store,
      dispatch
    }
  }
}
```

#### compose.js源码:
```javascript
export default function compose(...funcs) {
  //如果没有中间件作为参数传递，直接返回一个函数，接收对应的参数并且返回这个参数
  if (funcs.length === 0) {
    return arg => arg
  }
    
  //如果这个中间件参数只有一个，那么直接返回这个中间件函数
  if (funcs.length === 1) {
    return funcs[0]
  }

  //遍历所有中间件函数后，将其聚合返回一个单一的dispatch函数
  //reduce是es7提供的方法，就相当于是上一次迭代的结果可以传入下一次迭代作为参数
  //多个中间件传递进来的时候，他会借用reduce方法组合(这个放在后面), 会有个...args参数，就是(store.dispatch)
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
