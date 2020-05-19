#### 说明：Redux源码ES5版本的逐行解析，所有的英文注释全部做了意译。
#### 版本：4.0.5
#### 源码逐行解析（注释逐行翻译）
```javascript
'use strict';

Object.defineProperty(exports, '__esModule', { value: true });

function _interopDefault (ex) { return (ex && (typeof ex === 'object') && 'default' in ex) ? ex['default'] : ex; }

var $$observable = _interopDefault(require('symbol-observable'));

/**
 * 这些是Redux保留的私有操作类型。
 * These are private action types reserved by Redux.
 * 对于任何未知操作，必须返回当前状态。
 * For any unknown actions, you must return the current state.
 * 如果当前状态未定义，则必须返回初始状态。
 * If the current state is undefined, you must return the initial state.
 * 不要在代码中直接引用这些操作类型。
 * Do not reference these action types directly in your code.
 */
var randomString = function randomString() {
  return Math.random().toString(36).substring(7).split('').join('.');
};

var ActionTypes = {
  INIT: "@@redux/INIT" + randomString(),
  REPLACE: "@@redux/REPLACE" + randomString(),
  PROBE_UNKNOWN_ACTION: function PROBE_UNKNOWN_ACTION() {
    return "@@redux/PROBE_UNKNOWN_ACTION" + randomString();
  }
};

/**
 * 参数为一个对象。这个对象会被进行类型检查。
 * @param {any} obj The object to inspect.
 * 返回值是一个布尔类型。如果参数是一个普通对象，则为真
 * @returns {boolean} True if the argument appears to be a plain object.
 */
//判断是否为普通对象，它在下面的dispatch函数中被用来判断传入的action是否为普通对象
function isPlainObject(obj) {
  //首先判断是否为一个对象
  if (typeof obj !== 'object' || obj === null) return false;

  var proto = obj;

  //再判断对象是否为普通对象
  //普通对象就是该对象的__proto__等于Object.prototype
  //用Object.getPrototypeOf的方法返回对象的原型，原型存在就取到它的原型再赋值给proto
  while (Object.getPrototypeOf(proto) !== null) {
    proto = Object.getPrototypeOf(proto);
  }

  //最后返回的是一个bool值
  return Object.getPrototypeOf(obj) === proto;
}

/**
 * 创建一个store用于保存状态树
 * Creates a Redux store that holds the state tree.
 * 改变store中数据的唯一方式是通过store调用dispatch()这个方法
 * The only way to change the data in the store is to call `dispatch()` on it.
 *
 * 在你的应用中应该只有一个store。想要状态树的不同部分去响应action，你需要将多个reducer放在一个reducer函数中组成一个大的reducer。
 * 您可以通过使用“combineReducers”将多个reducer函数组合成单个reducer函数。这个函数在后面第543行。
 * There should only be a single store in your app. To specify how different
 * parts of the state tree respond to actions, you may combine several reducers
 * into a single reducer function by using `combineReducers`.
 *
 * 第一个参数reducer是返回新的状态树的函数，reducer函数本身需要需要当前的state状态树和操作action的句柄（dispatch）作为参数
 * @param {Function} reducer A function that returns the next state tree, given
 * the current state tree and the action to handle.
 *
 * 第二个参数preloadedState可以是任意类型。它是state的初始值。你可以用它来从服务器获取状态，或者恢复以前序列化的用户会话
 * @param {any} [preloadedState] The initial state. You may optionally specify it
 * to hydrate the state from the server in universal apps, or to restore a
 * previously serialized user session.
 *
 * 如果使用combineReducers函数去合成了根reducer函数，那么这个对象的结构必须与combineReducers的键相似。
 * If you use `combineReducers` to produce the root reducer function, this must be
 * an object with the same shape as `combineReducers` keys.
 *
 * 第三个参数是用来增强store自身功能的一个函数。你可以任意的传入第三方代码（例如中间件）去扩展store的功能
 * @param {Function} [enhancer] The store enhancer. You may optionally specify it
 * to enhance the store with third-party capabilities such as middleware,
 * time travel, persistence, etc. The only store enhancer that ships with Redux
 * is `applyMiddleware()`.
 *
 * 返回值是一个新的store，你可以通过它读取状态、用dispatch方法派发action描述要对state进行的改变
 * @returns {Store} A Redux store that lets you read the state, dispatch actions
 * and subscribe to changes.
 */

function createStore(reducer, preloadedState, enhancer) {
  var _ref2;

  //对参数进行判断
  if (typeof preloadedState === 'function' && typeof enhancer === 'function' || typeof enhancer === 'function' && typeof arguments[3] === 'function') {
    throw new Error('It looks like you are passing several store enhancers to ' + 'createStore(). This is not supported. Instead, compose them ' + 'together to a single function.');
  }

  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState;
    preloadedState = undefined;
  }

  //enhancer（第三个参数--中间件）必须是一个函数，用它来"包"createStore函数，以增强store的功能
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.');
    }
    //enhancer(createStore)(reducer,preloadState)是一柯里化函数
    //把createStore在applyMiddleware()函数里面执行。
    return enhancer(createStore)(reducer, preloadedState);
  }

  //reducer必须是一个函数
  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.');
  }

  //初始化当前参数
  var currentReducer = reducer;// 当前store中的reducer
  var currentState = preloadedState; // 当前store中存储的状态State

  //当前store中放置的监听者（函数）列表，是以数组的形式存储的
  var currentListeners = [];
  // 下一次dispatch时的监听函数
  var nextListeners = currentListeners;
  //dispatch的状态，初始化是未进行任何派发即标志位为false.当新添加一个监听函数时，只会在下一次dispatch的时候生效。
  var isDispatching = false;

  /**
   * 对当前监听者列表进行一个浅拷贝，这样能让我们在进行dispatch派发的时候将nextListeners作为一个临时的监听者容器
   * This makes a shallow copy of currentListeners so we can use
   * nextListeners as a temporary list while dispatching.
   *
   * 这样做能够阻止一些consumers调用subscribe/unsubscribe（监听和取消监听）方法时在派发dispatch中间的错误
   * This prevents any bugs around consumers calling
   * subscribe/unsubscribe in the middle of a dispatch.
   */

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      //利用Array.prototype.slice进行浅拷贝，它接收两个参数：begin,end。都为空时默认全部进行浅拷贝
      nextListeners = currentListeners.slice();
    }
  }


  /**
   * 通过store读取state状态管理树
   * Reads the state tree managed by the store.
   *
   * 返回值任意类型的数据，返回的是应用当前最新的状态树
   * @returns {any} The current state tree of your application.
   */

  //使用getState可以读取store中的状态树
  function getState() {
    //isDispatching是一个bool值，默认是false，当有派发时修改为true.先判断一下是不是通过dispatch这个方法进行派发的，是的话才能够访问，不是则报错。
    //这一步就实现了读取store中状态树只能通过dispatch这个唯一途径
    //为了保证数据的一致性，当在reducer操作的时候，isDispatching被改为false，不能读取当前的state值
    if (isDispatching) {
      throw new Error('You may not call store.getState() while the reducer is executing. ' + 'The reducer has already received the state as an argument. ' + 'Pass it down from the top reducer instead of reading it from the store.');
    }

    //这里直接返回了当前状态，并没有进行拷贝
    //从这里来看，store通过getState得出的state是可以直接被更改的，但是redux不允许这么做，因为这样不会通知订阅者更新数据
    return currentState;
  }


  /**
   * 增加监听者。在action被dispatch进行派发时或者状态树有可能改变时它将会被立即调用。然后你可以在你的回调函数中调用getState()方法读取到最新的状态树
   * Adds a change listener. It will be called any time an action is dispatched,
   * and some part of the state tree may potentially have changed. You may then
   * call `getState()` to read the current state tree inside the callback.
   *
   * 你可以为监听者用下面的方式调用dispatch进行派发
   * You may call `dispatch()` from a change listener, with the following
   * caveats:
   *
   * 1.订阅将在每次调用dispatch()之前保留快照
   * 1. The subscriptions are snapshotted just before every `dispatch()` call.
   * 如果侦听器正在被调用，无论是执行subscribe（订阅）还是unsubscribe（取消订阅），都将不会对本次的dispatch造成任何影响
   * If you subscribe or unsubscribe while the listeners are being invoked, this
   * will not have any effect on the `dispatch()` that is currently in progress.
   * 但是，当下一次dispatch被调用时，订阅函数无论是否被派发，都会使用最新订阅列表的快照
   * However, the next `dispatch()` call, whether nested or not, will use a more
   * recent snapshot of the subscription list.
   *
   * 2.监听者不能够监听到所有状态的改变，因为在调用侦听器之前，state可能在dispatch()期间多次更新。
   * 但是，它保证在dispatch()启动之前注册的所有订阅者都能在退出时将调用到最新的状态
   * 2. The listener should not expect to see all state changes, as the state
   * might have been updated multiple times during a nested `dispatch()` before
   * the listener is called. It is, however, guaranteed that all subscribers
   * registered before the `dispatch()` started will be called with the latest
   * state by the time it exits.
   *
   * 第一个参数是一个监听者函数，它将在每一个次执行dispatch时被调用
   * @param {Function} listener A callback to be invoked on every dispatch.
   * 第二份参数是一个函数，它是被移除的侦听器函数
   * @returns {Function} A function to remove this change listener.
   */

  //订阅函数. 在界面组件添加一个监听函数时，每次当dispatch被调用的时候都会执行这个监听函数
  function subscribe(listener) {
    //对参数进行判断，必须是一个函数
    if (typeof listener !== 'function') {
      throw new Error('Expected the listener to be a function.');
    }

    //如果注册之前已经执行了dispatch则报错（dispatch派发也意味着reducer在修改数据，为了保证数据的一致性，是不能够在这个时候进行订阅的）
    if (isDispatching) {
      throw new Error('You may not call store.subscribe() while the reducer is executing. ' + 'If you would like to be notified after the store has been updated, subscribe from a ' + 'component and invoke store.getState() in the callback to access the latest state. ' + 'See https://redux.js.org/api-reference/store#subscribelistener for more details.');
    }

    //订阅状态，执行subscribe时即为true.设置这个标志表示该监听器已经进行了订阅
    var isSubscribed = true;
    //这个函数在上面134行定义，它是用来对当前注册监听的函数列表进行浅拷贝的
    ensureCanMutateNextListeners();
    //将当前调用subscribe的回调函数添加到订阅者列表（添加新的订阅者）.但是新的订阅在下一次dispatch时订阅才会生效
    nextListeners.push(listener);

    //返回取消订阅的方法，即从数组中删除该监听函数。
    //这是非常必要的，因为当添加的监听器没用了之后，就应该从store中清理掉。不然每次dispatch都会调用这个没用的监听器。
    return function unsubscribe() {
      //判断订阅状态，如果订阅状态为ture才能继续执行，为false则直接return（已取消订阅的不必执行）
      if (!isSubscribed) {
        return;
      }

      //判断是已经执行了dispatch，如果已经进行了dispatch，则报错（即判断是否有reducer正在进行数据修改，同样是为了保证数据的一致性）
      if (isDispatching) {
        throw new Error('You may not unsubscribe from a store listener while the reducer is executing. ' + 'See https://redux.js.org/api-reference/store#subscribelistener for more details.');
      }

      //将订阅状态更改为false，即未订阅
      isSubscribed = false;
      //再次更新订阅者列表
      ensureCanMutateNextListeners();
      //取出当前订阅者的index.从下一轮的监听函数数组（用于下一次dispatch）中删除这个监听器。
      var index = nextListeners.indexOf(listener);
      //将当前订阅者从下一轮订阅者列表中去除
      nextListeners.splice(index, 1);
      //将当前的订阅者列表置为null
      currentListeners = null;
    };
  }


  /**
   * dispatch（派发）一个action。这是引发state改变的唯一方式。
   * Dispatches an action. It is the only way to trigger a state change.
   *
   * reducer函数用以创建store，它被调用时需要当前状态树state和action作为参数。它会返回是最新的state状态树，同时会将状态树State的改变通知订阅者
   * The `reducer` function, used to create the store, will be called with the
   * current state tree and the given `action`. Its return value will
   * be considered the **next** state of the tree, and the change listeners
   * will be notified.
   *
   * 基本的reducer实现只支持dispatch派发的action为普通对象。如果你想用dispatch派发Promise,Observable，thunk或者其他什么，
   * 你需要用响应的中间件在store的创建函数createStore外面包一层。比如redux-thunk包。
   * 中间件最后也会使用这个方法来dispatch（派发）普通对象
   * The base implementation only supports plain object actions. If you want to
   * dispatch a Promise, an Observable, a thunk, or something else, you need to
   * wrap your store creating function into the corresponding middleware. For
   * example, see the documentation for the `redux-thunk` package. Even the
   * middleware will eventually dispatch plain object actions using this method.
   *
   * 它的参数action是一个描述reducer需要对state做什么处理的普通对象。
   * 保持action的可序列化（保存每一次的action）是个好主意，这样你就可以记录和回溯用户会话，或者使用时间旅行的“redu -devtools”调试工具。
   * 一个action对象必须要有一个值不是undefined的type属性。一个好的建议是用一个专门的字符串常量定义action的type属性。
   * 也就是专门创建一个ActionType.js文件，用常量保存action的type属性
   * @param {Object} action A plain object representing “what changed”. It is
   * a good idea to keep actions serializable so you can record and replay user
   * sessions, or use the time travelling `redux-devtools`. An action must have
   * a `type` property which may not be `undefined`. It is a good idea to use
   * string constants for action types.
   *
   * 返回值是一个对象。为了方便起见，应该为dispatch函数传入相似的action对象（action对象的结构要类似）
   * @returns {Object} For convenience, the same action object you dispatched.
   *
   *
   * 注意，如果你使用自定义中间件，它会将对' dispatch() '进行包装，使dispatch能够接受普通对象之后的更多类型
   * 这样能得到普通dispatch得不到的东西(例如，将promise异步转为同步的await)
   * Note that, if you use a custom middleware, it may wrap `dispatch()` to
   * return something else (for example, a Promise you can await).
   */

  //dispatch一个用于初始化的action，相当于调用一次reducer，得到的新的state，并且执行所有添加到store中的订阅函数。这样订阅的函数就能取到更新后的state
  function dispatch(action) {
    //isPlainObject是一个判断是否为普通对象的函数，在最前面第38行有定义
    if (!isPlainObject(action)) {
      throw new Error('Actions must be plain objects. ' + 'Use custom middleware for async actions.');
    }

    //判断action的type属性是否为undefined，要求不能为undefined
    if (typeof action.type === 'undefined') {
      throw new Error('Actions may not have an undefined "type" property. ' + 'Have you misspelled a constant?');
    }

    //判断action的派发状态，不能是已经执行dispatch的action，也就是说同一个action同一时间只能被派发一次！！！
    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.');
    }

    //当dispatch状态为true（已派发）时，执行当前的reducer函数，根据action对当前的状态state进行处理
    //执行前将isDispatching设置为true，阻止后续的action进来触发reducer操作
    try {
      isDispatching = true;
      //调用reducer，将得到的最新state赋值给currentState
      currentState = currentReducer(currentState, action);
    } finally {
      //完成之后再finally里将isDispatching再改为false，允许后续的action进来触发reducer操作
      isDispatching = false;
    }

    //接下来通知订阅者做数据更新，不传入任何参数。最后返回当前的action。
    //更新监听数组
    var listeners = currentListeners = nextListeners;

    //遍历订阅者列表，并执行每一个订阅者的回调函数，这就实现了在状态改变后(reducer执行后)对每一个订阅者的发布，订阅者就能够取到最新的state
    for (var i = 0; i < listeners.length; i++) {
      var listener = listeners[i];
      listener();
    }

    //返回action
    return action;
  }


  /**
   * 替换当前的reducer
   * Replaces the reducer currently used by the store to calculate the state.
   *
   * 如果你的应用程序实现了代码分割，并且你想要动态加载一些reducer，你可能还需要实现了Redux的热重载机制。
   * You might need this if your app implements code splitting and you want to
   * load some of the reducers dynamically. You might also need this if you
   * implement a hot reloading mechanism for Redux.
   *
   * 第一个参数是一个reducer函数。它是用来替换当前reducer的。
   * @param {Function} nextReducer The reducer for the store to use instead.
   * 返回值为空
   * @returns {void}
   */

  //替换reducer，平时项目里基本很难用到
  function replaceReducer(nextReducer) {
    //判断传入的reducer必须为函数
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.');
    }

    //替换当前的reducer
    currentReducer = nextReducer;
    //这个action的作用与ActionTypes.INIT类似。
    // This action has a similiar effect to ActionTypes.INIT.

    //任何存在于新旧根renducer中的renducer都将接收到以前的状态。这将有效地用旧状态树中的任何相关state填充新状态树
    // Any reducers that existed in both the new and old rootReducer
    // will receive the previous state. This effectively populates
    // the new state tree with any relevant data from the old one.

    //替换reducer之后重新派发
    dispatch({
      type: ActionTypes.REPLACE
    });
  }


  /**
   * 用于观察/反应库的互操作性
   * Interoperability point for observable/reactive libraries.
   *
   *第一个返回值为observable 状态变化的最小可观测值
   * @returns {observable} A minimal observable of state changes.
   *
   * 这是为了适配 ECMA TC39 会议的一个有关 Observable 的提案所写的一个函数。
   * 作用是订阅 store 的变化，适用于所有实现了 Observable 的类库
   * For more information, see the observable proposal:
   * https://github.com/tc39/proposal-observable
   */

  /***********************这个没明白具体是想要做什么,定义的这个observable函数也没有进行任何调用***************************/
  function observable() {
    var _ref;

    //这里的subscribe是前面定义的订阅函数，赋值给outerSubscribe（外部订阅者）
    var outerSubscribe = subscribe;
    return _ref = {
      /**
       * 最小可观察订阅方法
       * The minimal observable subscription method.
       * 接收一个观察者对象。任何一个对象都能够作为观察者
       * @param {Object} observer Any object that can be used as an observer.
       * 这个观察者对象需要有一个next方法
       * The observer object should have a `next` method.
       * 返回值是一个订阅者对象。这个对象有一个unsubscribe取消订阅方法，这个方法能够使观察者从store中移除订阅，
       * 防止从观察者值的进一步发射值
       * @returns {subscription} An object with an `unsubscribe` method that can
       * be used to unsubscribe the observable from the store, and prevent further
       * emission of values from the observable.
       */

      subscribe: function subscribe(observer) {
        //判断观察者不能为空或者非对象
        if (typeof observer !== 'object' || observer === null) {
          throw new TypeError('Expected the observer to be an object.');
        }

        //用于给 subscribe 注册的函数，严格按照 Observable 的规范实现，observer 必须有一个 next 属性。
        function observeState() {
          if (observer.next) {
            observer.next(getState());
          }
        }

        observeState();
        var unsubscribe = outerSubscribe(observeState);
        return {
          unsubscribe: unsubscribe
        };
      }
    }, _ref[$$observable] = function () {// $$observable 即为 Symbol.observable，也属于 Observable 的规范，返回自身。
      return this;
    }, _ref;
  }
  //当store被创建时，action的初始化状态ActionTypes为INIT并被dispatch，每一个reducer函数都返回它们自己的初始状态state
  //这是初始化状态树的有效方式
  // When a store is created, an "INIT" action is dispatched so that every
  // reducer returns their initial state. This effectively populates
  // the initial state tree.

  dispatch({
    type: ActionTypes.INIT
  });

  // 返回一个 store 对象。
  return _ref2 = {
    dispatch: dispatch,
    subscribe: subscribe,
    getState: getState,
    //这两个是主要面向库开发者的方法
    replaceReducer: replaceReducer
  }, _ref2[$$observable] = observable, _ref2;
}

/**
 * 控制台打印警告
 * Prints a warning in the console if it exists.
 *
 * @param {String} message The warning message.
 * @returns {void}
 */
function warning(message) {
  /* eslint-disable no-console */
  //console加了一层判断是为了兼容IE89以下
  if (typeof console !== 'undefined' && typeof console.error === 'function') {
    console.error(message);
  }
  /* eslint-enable no-console */


  try {
    // This error was thrown as a convenience so that if you enable
    // "break on all exceptions" in your console,
    // it would pause the execution at this line.
    throw new Error(message);
  } catch (e) {} // eslint-disable-line no-empty

}

function getUndefinedStateErrorMessage(key, action) {
  var actionType = action && action.type;
  var actionDescription = actionType && "action \"" + String(actionType) + "\"" || 'an action';
  return "Given " + actionDescription + ", reducer \"" + key + "\" returned undefined. " + "To ignore an action, you must explicitly return the previous state. " + "If you want this reducer to hold no value, you can return null instead of undefined.";
}

function getUnexpectedStateShapeWarningMessage(inputState, reducers, action, unexpectedKeyCache) {
  var reducerKeys = Object.keys(reducers);
  var argumentName = action && action.type === ActionTypes.INIT ? 'preloadedState argument passed to createStore' : 'previous state received by the reducer';

  //保证reducers集合不为空
  if (reducerKeys.length === 0) {
    return 'Store does not have a valid reducer. Make sure the argument passed ' + 'to combineReducers is an object whose values are reducers.';
  }

  //判断inputState是否为普通对象
  if (!isPlainObject(inputState)) {
    return "The " + argumentName + " has unexpected type of \"" + {}.toString.call(inputState).match(/\s([a-z|A-Z]+)/)[1] + "\". Expected argument to be an object with the following " + ("keys: \"" + reducerKeys.join('", "') + "\"");
  }

  //筛选出inputState里有但是reducers集合里没有key
  var unexpectedKeys = Object.keys(inputState).filter(function (key) {
    return !reducers.hasOwnProperty(key) && !unexpectedKeyCache[key];
  });
  unexpectedKeys.forEach(function (key) {
    unexpectedKeyCache[key] = true;
  });

  //如果是替换reducer的action,跳过下面的代码，不打印异常信息
  if (action && action.type === ActionTypes.REPLACE) return;

  //打印所有异常的key
  if (unexpectedKeys.length > 0) {
    return "Unexpected " + (unexpectedKeys.length > 1 ? 'keys' : 'key') + " " + ("\"" + unexpectedKeys.join('", "') + "\" found in " + argumentName + ". ") + "Expected to find one of the known reducer keys instead: " + ("\"" + reducerKeys.join('", "') + "\". Unexpected keys will be ignored.");
  }
}

//这个函数在第584行检查finalReducer中的reducer接受一个初始action或一个未知的action时，是否依旧能够返回有效的值。
function assertReducerShape(reducers) {
  Object.keys(reducers).forEach(function (key) {
    var reducer = reducers[key];
    var initialState = reducer(undefined, {
      type: ActionTypes.INIT
    });

    if (typeof initialState === 'undefined') {
      throw new Error("Reducer \"" + key + "\" returned undefined during initialization. " + "If the state passed to the reducer is undefined, you must " + "explicitly return the initial state. The initial state may " + "not be undefined. If you don't want to set a value for this reducer, " + "you can use null instead of undefined.");
    }

    if (typeof reducer(undefined, {
      type: ActionTypes.PROBE_UNKNOWN_ACTION()
    }) === 'undefined') {
      throw new Error("Reducer \"" + key + "\" returned undefined when probed with a random type. " + ("Don't try to handle " + ActionTypes.INIT + " or other actions in \"redux/*\" ") + "namespace. They are considered private. Instead, you must return the " + "current state for any unknown actions, unless it is undefined, " + "in which case you must return the initial state, regardless of the " + "action type. The initial state may not be undefined, but can be null.");
    }
  });
}
/**
 * 将值为不同的reducer函数的对象转换为单个reducer函数。
 * 它将调用每一个reducer函数，并将它们的结果放在一个单一的state对象容器中，这个对象的键对应于传递的reducer函数的键。
 * Turns an object whose values are different reducer functions, into a single
 * reducer function. It will call every child reducer, and gather their results
 * into a single state object, whose keys correspond to the keys of the passed
 * reducer functions.
 *
 * 参数是一个reducer对象。这个对象的值对应于不同的reducer函数，需要组合成一个的不同reducer函数
 * 用ES6的import可以方便的引入reducer。reducer永远不会返回undefined。
 * 如果传递给reducer的状态是未定义的，或者任何未识别操作的当前状态都应该返回state的初始状态。
 * @param {Object} reducers An object whose values correspond to different
 * reducer functions that need to be combined into one. One handy way to obtain
 * it is to use ES6 `import * as reducers` syntax. The reducers may never return
 * undefined for any action. Instead, they should return their initial state
 * if the state passed to them was undefined, and the current state for any
 * unrecognized action.
 *
 * 返回值是一个函数。一个调用传递对象中的每个reducer的函数，并构建一个具有相同形状的状态对象
 * @returns {Function} A reducer function that invokes every reducer inside the
 * passed object, and builds a state object with the same shape.
 */


/**
 * 这个函数的目的为了用多个reducer实现不同的逻辑
 * 当然可以用一个reducer去处理所有状态的维护，但是就会使一个reducer函数的逻辑太多，非常容易混乱
 * 因此可以将逻辑（reducer）按照模块划分，每个模块再细分成各个子模块，开发完每个模块的逻辑后，再将reducer合并起来
 * 对于redux提供了combineReducers方法，它可以把子reducer合并成一个总的reduce
 * 通过combineReducers函数合并reducer函数，返回一个新的函数combination（这个函数负责循环遍历运行reducer函数，返回全部state）。
 * 当combination函数被调用时，实际就是循环调用传入的reducer函数，返回一个挂载全部state的对象
 * 将这个新函数作为参数传入createStore函数，函数内部通过dispatch，初始化运行传入的action和state，返回store对象。
*/
function combineReducers(reducers) {
  //获取传入reducers对象的所有key。前面createStore第57行的注释对这里有说明。
  var reducerKeys = Object.keys(reducers);
  // 最后要合成的reducer
  var finalReducers = {};

  //遍历传入的reducers，并根据reducers的key值筛选出有效的reducer
  //前面createStore那里第70行有注释：
  //如果使用combineReducers函数去合成了根reducer函数，那么这个对象的结构必须与combineReducers的键相似
  for (var i = 0; i < reducerKeys.length; i++) {
    var key = reducerKeys[i];

    //判断是否为生产环境
    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        //warning函数的定义在444行
        warning("No reducer provided for key \"" + key + "\"");
      }
    }

    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key];
    }
  }

  var finalReducerKeys = Object.keys(finalReducers);
  //这是用于确保我们不会被警告有相同的键
  // This is used to make sure we don't warn about the same keys multiple times.

  var unexpectedKeyCache;

  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {};
  }

  //assertReducerShape函数的定义在第493行，它在这里的作用是：
  //检查finalReducer中的reducer接受一个初始action或一个未知的action时，是否依旧能够返回有效的值。

  var shapeAssertionError;

  try {
    assertReducerShape(finalReducers);
  } catch (e) {
    shapeAssertionError = e;
  }


  //返回合并后的reducer函数，用于代理所有的reducer
  return function combination(state, action) {
    if (state === void 0) {
      state = {};
    }

    if (shapeAssertionError) {
      throw shapeAssertionError;
    }

    if (process.env.NODE_ENV !== 'production') {
      //getUnexpectedStateShapeWarningMessage函数的定义在467行
      var warningMessage = getUnexpectedStateShapeWarningMessage(state, finalReducers, action, unexpectedKeyCache);

      if (warningMessage) {
        warning(warningMessage);
      }
    }


    var hasChanged = false;//标志state是否有变化
    var nextState = {};

    //遍历reducers集合，取得每个子reducer对应的state，与action一起作为参数给每个子reducer执行。
    for (var _i = 0; _i < finalReducerKeys.length; _i++) {
      // finalReducerKeys 是传入reducers对象的key值
      var _key = finalReducerKeys[_i];
      // finalReducers相当于传入的每个reducers
      var reducer = finalReducers[_key];
      //得到子reducer的旧状态
      var previousStateForKey = state[_key];
      //调用子reducer得到新状态
      var nextStateForKey = reducer(previousStateForKey, action);

      if (typeof nextStateForKey === 'undefined') {
        var errorMessage = getUndefinedStateErrorMessage(_key, action);
        throw new Error(errorMessage);
      }

      //nextState是总的状态
      nextState[_key] = nextStateForKey;

      //就是如果子reducer不能处理该action，那么会返回previousStateForKey（旧状态）
      //也就是当所有状态都没改变时，直接返回之前的state就可以了
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey;
    }

    hasChanged = hasChanged || finalReducerKeys.length !== Object.keys(state).length;
    return hasChanged ? nextState : state;
  };
}


//接收两个参数：actionCreator函数，用于创建action的工厂函数；dispatch 函数，即Store.dispatch ()
//作用：将dispatch和actionCreator绑定返回一个新的函数，使用这个函数时会直接 dispatch 一个 action
//actionCreator帮我们省略了重复书写action.type和dispatch，而且还可以隐藏 dispatch 参数
function bindActionCreator(actionCreator, dispatch) {
  return function () {
    return dispatch(actionCreator.apply(this, arguments));
  };
}
/**
 * 将值为action creator的对象转换为具有相同键的对象，但是将每个函数封装到“dispatch”调用中，因此可以直接调用它们。
 * 这只是一个方便的方法，你可以自己像这样进行调用' store.dispatch(myactioncreator . dosomething() ')。
 * Turns an object whose values are action creators, into an object with the
 * same keys, but with every function wrapped into a `dispatch` call so they
 * may be invoked directly. This is just a convenience method, as you can call
 * `store.dispatch(MyActionCreators.doSomething())` yourself just fine.
 *
 * 为方便起见，你可以将action creator作为第一个参数传递，并返回一个dispatch包装函数。
 * For convenience, you can also pass an action creator as the first argument,
 * and get a dispatch wrapped function in return.
 *
 * 第一个参数actionCreators的类型可以为函数或者对象。它的值是action creator函数。
 * 引入可以使用ES6的 ' import * as ' 语法。你也可以只传递一个函数
 * @param {Function|Object} actionCreators An object whose values are action
 * creator functions. One handy way to obtain it is to use ES6 `import * as`
 * syntax. You may also pass a single function.
 *
 * 第二个参数dispatch是一个函数。
 * @param {Function} dispatch The `dispatch` function available on your Redux
 * store.
 *
 * 返回值可以是函数或者对象。该对象模仿原生对象，但是每个action creator都被包装到“dispatch”调用中。
 * 如果你将一个函数作为“actioncreator”进行传递，那么返回值也将是一个函数
 * @returns {Function|Object} The object mimicking the original object, but with
 * every action creator wrapped into the `dispatch` call. If you passed a
 * function as `actionCreators`, the return value will also be a single
 * function.
 */

/**
 * 这个方法并不常用，根据redux官网自己的介绍：
 * 惟一会使用到 bindActionCreator 的场景是当你需要把 action creator 往下传到一个组件上，却不想让这个组件觉察到 Redux 的存在
 * 而且不希望把 dispatch 或 Redux store 传给它
 */

//如果 actions需要传入多个参数时，可以借助 bindActionCreators来精简的代码
function bindActionCreators(actionCreators, dispatch) {
  //判断第一个参数是否为函数，如果是函数则直接绑定返回一个新的函数
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch);
  }

  //判断第二个参数是否为对象
  if (typeof actionCreators !== 'object' || actionCreators === null) {
    throw new Error("bindActionCreators expected an object or a function, instead received " + (actionCreators === null ? 'null' : typeof actionCreators) + ". " + "Did you write \"import ActionCreators from\" instead of \"import * as ActionCreators from\"?");
  }

  //创建一个容器对象，是最终要返回的对象
  var boundActionCreators = {};

  //遍历传入的actionCreators
  for (var key in actionCreators) {
    var actionCreator = actionCreators[key];
    /// 进行过滤。如果第一个参数actionCreators对象的字段是actionCreator函数，则利用bindActionCreator将dispatch和actionCreator绑定
    // 并将结果存入boundActionCreators容器对象中。
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch);
    }
  }

  //返回最终绑定好的actionCreator对象
  return boundActionCreators;
}

function _defineProperty(obj, key, value) {
  if (key in obj) {
    Object.defineProperty(obj, key, {
      value: value,
      enumerable: true,
      configurable: true,
      writable: true
    });
  } else {
    obj[key] = value;
  }

  return obj;
}

function ownKeys(object, enumerableOnly) {
  var keys = Object.keys(object);

  if (Object.getOwnPropertySymbols) {
    keys.push.apply(keys, Object.getOwnPropertySymbols(object));
  }

  if (enumerableOnly) keys = keys.filter(function (sym) {
    return Object.getOwnPropertyDescriptor(object, sym).enumerable;
  });
  return keys;
}

function _objectSpread2(target) {
  for (var i = 1; i < arguments.length; i++) {
    var source = arguments[i] != null ? arguments[i] : {};

    if (i % 2) {
      ownKeys(source, true).forEach(function (key) {
        _defineProperty(target, key, source[key]);
      });
    } else if (Object.getOwnPropertyDescriptors) {
      Object.defineProperties(target, Object.getOwnPropertyDescriptors(source));
    } else {
      ownKeys(source).forEach(function (key) {
        Object.defineProperty(target, key, Object.getOwnPropertyDescriptor(source, key));
      });
    }
  }

  return target;
}

/**
 * 从右到左组合单参数函数。最右边的函数可以接受多个参数，因为它为生成的复合函数提供了签名
 * Composes single-argument functions from right to left. The rightmost
 * function can take multiple arguments as it provides the signature for
 * the resulting composite function.
 *
 * 接收一个函数数组。这些函数将被组合起来。
 * @param {...Function} funcs The functions to compose.
 * 返回值是一个函数。由右至左组合中间件函数，最后得到一个组合函数。例如compose(f, g, h)将被以这样的形式执行：
 * (...args) => f(g(h(...args)))
 * @returns {Function} A function obtained by composing the argument functions
 * from right to left. For example, compose(f, g, h) is identical to doing
 * (...args) => f(g(h(...args))).
 */

//将多个中间件函数合成一个函数
function compose() {
  //_len是传入的中间件函数长度，funcs是新建的数组，它将浅拷贝中间件函数数组
  for (var _len = arguments.length, funcs = new Array(_len), _key = 0; _key < _len; _key++) {
    funcs[_key] = arguments[_key];
  }

  //如果传入的中间件函数长度为0，则返回传入的参数
  if (funcs.length === 0) {
    return function (arg) {
      return arg;
    };
  }

  //如果传入的中间件函数只有一个，则返回这一个中间件函数
  if (funcs.length === 1) {
    return funcs[0];
  }

  //将多个中间件函数进行组合，并返回组合后的函数
  //reduce是es7提供的一个高阶函数，它对数组中的每个元素执行一个由你提供的reducer函数(升序执行)，并将结果组合为一个函数返回
  //reduce接收四个参数，第一个参数是一个累计器，它将每次元素执行的结果作为参数传给下一个元素执行
  //这里正是利用reduce第一个参数是累计器的特性将每次函数执行的结果作为参数给下一个参数，这样就使得每一个传入的中间件函数依次对dispatch进行了包装
  return funcs.reduce(function (a, b) {
    return function () {
      return a(b.apply(void 0, arguments));
    };
  });
}

/**
 * 创建一个增强dispatch方法的中间件处理函数。这对于各种操作都很方便，比如以简洁的方式表达异步操作，或者记录每个action
 * Creates a store enhancer that applies middleware to the dispatch method
 * of the Redux store. This is handy for a variety of tasks, such as expressing
 * asynchronous actions in a concise manner, or logging every action payload.
 *
 * redux-chunk是redux中间件的一个很好的例子。
 * See `redux-thunk` package as an example of the Redux middleware.
 *
 *因为中间件可能是异步的，所以它(指redux-chunk)应该是组合链chain中的第一个执行的（就是说要放在中间件函数的第一个参数位置）
 * Because middleware is potentially asynchronous, this should be the first
 * store enhancer in the composition chain.
 *
 *注意，每个中间件都将提供' dispatch '和' getState '函数两个参数。
 * Note that each middleware will be given the `dispatch` and `getState` functions
 * as named arguments.
 *
 * 参数是中间件函数数组。这个中间件数组就是要应用的中间件链。
 * @param {...Function} middlewares The middleware chain to be applied.
 * 返回值是一个函数。它是应用中间件以后的增强函数。
 * @returns {Function} A store enhancer applying the middleware.
 */

//中间件机制（重要）
//applyMiddleware函数中间件的主要目的就是修改dispatch函数，返回经过中间件处理的新的dispatch函数
function applyMiddleware() {
  //这里与上面componse函数完全相同。_len是传入的中间件函数长度，funcs是新建的数组，它将浅拷贝中间件函数数组
  for (var _len = arguments.length, middlewares = new Array(_len), _key = 0; _key < _len; _key++) {
    middlewares[_key] = arguments[_key];
  }

  // 返回一个函数A，函数A的参数是一个createStore函数。
  // 函数A的返回值是函数B，其实也就是一个加强后的createStore函数，大括号内的是函数B的函数体
  return function (createStore) {
    return function () {
      //利用createStore函数创建store
      var store = createStore.apply(void 0, arguments);

      var _dispatch = function dispatch() {
        //作用是在dispatch改造完成前调用dispatch只会打印错误信息
        throw new Error('Dispatching while constructing your middleware is not allowed. ' + 'Other middleware would not be applied to this dispatch.');
      };

      //创建一个由每个中间件处理的对象，包含getState与dispatch（这个dispatch是自己创建的，不是redux中的）
      //下面将每个中间件与state关联起来（通过传入getState方法），得到改造函数。
      var middlewareAPI = {
        getState: store.getState,
        dispatch: function dispatch() {
          return _dispatch.apply(void 0, arguments);
        }
      };

      //遍历执行每一个中间件函数，每一个中间件函数都将操作middlewareAPI
      //chain是保存中间件新返回函数的数组，里面的元素是接收middlewareAPI参数执行后的每个中间件函数
      //middlewares是一个中间件函数数组，中间件函数的返回值是一个改造dispatch的函数
      //调用数组中的每个中间件函数，得到所有的改造函数
      var chain = middlewares.map(function (middleware) {
        return middleware(middlewareAPI);
      });
      //调用compose遍历所有中间件函数后,将每一个chain中间件函数进行组合.返回一个单一的dispatch函数

      // compose方法的作用是，例如这样调用：
      // compose(func1,func2,func3)
      // 返回一个函数: (...args) => func1( func2( func3(...args) ) )
      // 即传入的dispatch被func3改造后得到一个新的dispatch，新的dispatch继续被func2改造...
      // 这里用compose整合chain数组，并赋值给dispatch
      _dispatch = compose.apply(void 0, chain)(store.dispatch);

      // 返回store，用改造后的dispatch方法替换store中的dispatch
      //_objectSpread2方法在上面745行
      return _objectSpread2({}, store, {
        //将新的dispatch替换原先的store.dispatch
        dispatch: _dispatch
      });
    };
  };
}

/**
 * 这是一个伪函数，用来检查函数名是否被压缩修改了。
 * This is a dummy function to check if the function name has been altered by minification.
 * 如果函数名已经被压缩修改了，并且NODE_ENV !== 'production'，则给用户发出警告。
 * If the function has been minified and NODE_ENV !== 'production', warn the user.
 */

function isCrushed() {}

if (process.env.NODE_ENV !== 'production' && typeof isCrushed.name === 'string' && isCrushed.name !== 'isCrushed') {
  warning('You are currently using minified code outside of NODE_ENV === "production". ' + 'This means that you are running a slower development build of Redux. ' + 'You can use loose-envify (https://github.com/zertosh/loose-envify) for browserify ' + 'or setting mode to production in webpack (https://webpack.js.org/concepts/mode/) ' + 'to ensure you have the correct code for your production build.');
}

exports.__DO_NOT_USE__ActionTypes = ActionTypes;
exports.applyMiddleware = applyMiddleware;
exports.bindActionCreators = bindActionCreators;
exports.combineReducers = combineReducers;
exports.compose = compose;
exports.createStore = createStore;
```
