---
title: "再谈React同构应用：服务端下复用Redux Effects的实践"
date: 2016-12-21 18:07:32
tags:
- front-end
- React
categories:
- 学习笔记
- 前端
---
同构 (universal/isomorphic) React应用旨在服务端（或者是网关层、中途岛层）和客户端（浏览器端）尽可能地复用UI组件的代码，以提高项目的可维护性。当同构应用引入以 [Redux](https://github.com/reactjs/redux) 为首的数据流管理、以 [react-router](https://github.com/ReactTraining/react-router) 为主的SPA前端路由后，同构应用将变得复杂：我们需要在服务端和客户端之间同步状态（store）和路由信息，并且尽可能地复用这些数据逻辑（如reducers）和路由配置。关于如何搭建这样的一个项目框架，你可以阅读 [Server Side Rendering with React and Redux](https://blog.tableflip.io/server-side-rendering-with-react-and-redux/)。

本文假设你已经熟悉如何搭建一个 React + Redux + react-router 的同构应用，我们来讨论Redux副作用（side effects，后面简称effects）在服务端复用的逐步尝试和实践。

目前的典型场景
------------
目前大多数React同构脚手架均不在服务端复用effects，而是通过直接调用Service模块的方式来加载数据，这使得我们可以直接获知异步任务何时完成，并在回调函数中直接执行我们的渲染逻辑。在渲染逻辑中，因为页面初始数据已经取得，从创建store到调用`store.getState()`来初始化渲染模板都是同步的，没有任何坑点，它看起来是这样的：
```javascript
APIService.getTodos().then((initialData) => {
  const store = configureStore(makeInitialState(initialData));
  const html = ReactDOMServer.renderToString( /* ... */ );
  const state = store.getState()
  renderFullPage(html, state);
});
```
### 例：universal-react-starter-kit
以国内比较流行的脚手架 [bodyno/universal-react-starter-kit](https://github.com/bodyno/universal-react-starter-kit) 为例，其渲染部分的关键代码是这样的：
*server/main.js*
```javascript
const initialState = await router(ctx)
const store = createStore(initialState, memoryHistory)
/* ... */
match({history, routes, location: ctx.req.url}, async (err, redirect, props) => {
  /* ... */
  let layout = {
      /* ... */
      {type: 'text/javascript', innerHTML: `___INITIAL_STATE__ = ${JSON.stringify(store.getState())}`},
      /* ... */
    ]}
  /* ... */
  content = renderToString(
    <AppContainer layout={layout} />
  );
});
```
其中 `await router(ctx)`的`router`部分代码如下：
*server/router.js*
```javascript
export default async function (ctx) {
  return new Promise((resolve, reject) => {
    /* ... */
    axios.get('https://api.github.com/zen').then(({data}) => {
      resolve({zen: { text: [{text: data}]} })
    })
  })
}
```
`await router(ctx)`在此处就是一次Service API调用。先不论这个`router`是否名不符实（可能因为是脚手架的原因。router.js应该是给开发者填入代码来实现对应不同路由调用不同的Service），这个脚手架的渲染逻辑跟上文的示例大同小异——直接调用Service模块异步取得初始数据，在回调（await）中通过全同步的方式用初始数据产生store并`getState()`，然后调用`renderToString()`渲染。

在服务端通过“直捅Service”的方式来获取页面初始数据，是最直接、最简单的方法。当然我们在客户端绝对不会这么做，在客户端我们会设计好同步的actions和reducers，并通过触发effects来实现异步数据获取。为了使我们的服务端代码更优雅、维护性更强、代码复用度更高，我们希望在服务端能够复用这些actions、reducers和effects。

使用redux-thunk的场景
-------------------
在服务端执行一个effect是很简单的，我们只需要调用在服务端和客户端间共享的`configureStore()`函数来创建一个空的store（这时你将拥有effects所必须的middleware），然后调用`store.dispatch()`来触发一个绑定了effects的action即可。难点是：程序如何得知一个异步effects已经执行完成？这样我们才能在effects完成后调用`store.getState()`来取得带初始数据的state。
如果你的项目所使用的effects是 [redux-thunk](https://github.com/gaearon/redux-thunk)，你可以很容易地在服务端复用它们：你只需要在thunk函数中返回一个promise即可——而这是官方建议的标准写法。这样，`store.dispatch()`可以直接返回这个promise。
你的async thunk action creator看起来是这样的：
```javascript
function fetchTodos() {
  return function(dispatch) {
    dispatch({ type: 'todos/get' });
    return APIService.getTodos()
      .then(payload => dispatch({
        type: 'todos/get/success',
        payload,
      }));
  }
}
```
APIService看起来是这样的：
```javascript
const APIService = {
  getTodos: () => fetch('/api/todos').then(response => response.json()),
}
```
这样，在服务端的渲染逻辑，你可以这样写：
```javascript
const store = configureStore({});
store.dispatch({ type: 'todos/get' })
  .then(() => {
    const html = ReactDOMServer.renderToString( /* ... */ );
    const state = store.getState()
    renderFullPage(html, state);
  });
```
另外，还有 [redux-promise](https://github.com/acdlite/redux-promise) 的effects解决方案。在服务端复用方面，redux-promise和redux-thunk极为相似，因为使用redux-promise同样可以通过`store.dispatch()`获得异步任务的promise。
唯一的不同之处是，当使用redux-promise时，async action creator看起来是这样的：
```javascript
function getTodos() {
  return {
    type: 'todos/get',
    payload: APIService.getTodos(), //action.payload是一个promise
  }
}
```

### 例：react-redux-universal-hot-example
让我们来看看GitHub上stars最多的Universal React脚手架 [erikras/react-redux-universal-hot-example](https://github.com/erikras/react-redux-universal-hot-example) 是怎么解决的。
这个脚手架使用了 [redux-async-connect](https://github.com/Rezonans/redux-async-connect) middleware，这使得我们可以绑定一个promise给每一个container，并在服务端使用它提供的`loadOnServer()`方法获得待渲染的container的异步任务及其promise。
*src/containers/App/App.js*
```javascript
@asyncConnect([{
  promise: ({store: {dispatch, getState}}) => {
    const promises = [];
    if (!isInfoLoaded(getState())) {
      promises.push(dispatch(loadInfo()));
    }
    if (!isAuthLoaded(getState())) {
      promises.push(dispatch(loadAuth()));
    }
    return Promise.all(promises);
  }
}])
@connect(
  state => ({user: state.auth.user}),
  {logout, pushState: push})
export default class App extends Component {
  /* ... */
}
```
*src/server.js*
```javascript
loadOnServer({...renderProps, store, helpers: {client}}).then(() => {
  const component = (
    <Provider store={store} key="provider">
      <ReduxAsyncConnect {...renderProps} />
    </Provider>
  );
  res.status(200);
  global.navigator = {userAgent: req.headers['user-agent']};
  res.send('<!doctype html>\n' +
    ReactDOM.renderToString(<Html assets={webpackIsomorphicTools.assets()} component={component} store={store}/>));
});
```
从上面的代码中，我们看到：
* 作者使用redux-async-connect将container和一个promise绑定，这个promise执行多个`dispatch()`调用，当它们返回的promise都resolve时才resolve自身。
* 服务端通过调用已经绑定的`loadOnServer()`方法得到上述的这个promise，从而可以直接在`.then()`中填写该promise执行完成后的同步渲染逻辑。
* 之所以能够这么做，还是依赖于redux-thunk的`store.dispatch()`调用能够返回异步任务对应的promise。

使用redux-saga的场景
------------------
然而，对于业务逻辑逐渐复杂的Web APP，redux-thunk或许不能满足复杂的数据流场景。现在国内最流行的Effects方案莫过于  [redux-saga](https://github.com/yelouafi/redux-saga) 了。

redux-saga使得异步effects完全脱离于原生Redux数据流，没有Async Action creator（你甚至不需要多余的Action Creator）。Saga effects更像是运行于另一个线程的一组任务（除了Web Worker外目前客户端JavaScript还没有真正意义上的多线程），这些任务可以监听特定的action，并在不直接影响Redux数据流的前提下执行异步操作。

因为redux-saga的这些优点，使得它可以实现更复杂的异步数据流，保留更纯净的原生Redux流，这非常优雅。而正因如此，它不会对`store.dispatch()`的返回值做任何更改——这意味着，在服务端我们不能指望仅仅通过`store.dispatch()`就能获知我们的初始数据何时到达。

这时我想到了参考已有的、使用redux-saga的同构脚手架。

### dva提供的同构脚手架
[dva](https://github.com/dvajs/dva) ——蚂蚁金服推出的一个轻量级框架，基于redux、redux-saga和react-router，让你能够使用类似 [elm-lang](http://elm-lang.org) 的声明性风格来组织你的代码。

dva官方提供的同构脚手架是 [sorrycc/dva-boilerplate-isomorphic](https://github.com/sorrycc/dva-boilerplate-isomorphic) 。让我们来看看它是怎么解决saga在服务端下的渲染的。
*server/ssrMiddleware.js*
```javascript
import { fetchList } from '../common/services/user';
// ...
fetchList()
  .then(({ err, data }) => {
    const initialState = { user: data };
    const app = createApp({
      history: createMemoryHistory(),
      initialState,
    }, /* isServer */true);
    const html = renderToString(app.start()({ renderProps }));
    res.end(renderFullPage(html, initialState));
  });
```
*common/services/user.js*
```javascript
import request from '../utils/request';
export function fetchList() {
  return request('/api/users');
}
```
看到这里，相信大家都明白了。dva在这里的服务端逻辑是“直捅Service”的。dva的官方脚手架并没有解决我们的问题。

### 官方建议的runSaga()
事实上，对于redux-saga的服务端渲染问题，早就有关于这个的讨论，参考 [issue #13](https://github.com/yelouafi/redux-saga/issues/13) 。而redux-saga已添加了 [runSaga()](http://yelouafi.github.io/redux-saga/docs/api/index.html#runsagaiterator-options) 方法来实现在服务端复用saga effects。

`runSaga()`接收一个`saga`对象和必须的store输入输出方法（`subscribe()`和`dispatch()`等），允许在store上下文之外执行一个saga任务，并返回一个`Task`实例对象。返回的`Task`对象中的`done`属性是一个promise对象的引用，该promise在传入的saga任务执行完成后resolve。

假设我们有这样的一个saga effect：
```javascript
function* getTodos() {
  const payload = yield call(APIService.getTodos);
  yield put({ type: 'todos/get/success', payload });
}
```
由于我们可以获得store上下文和`sagaMiddleware`，在这里我们可以直接使用`sagaMiddleware.run()`来代替`runSaga()`。`sagaMiddleware.run()`同样返回对应这个saga任务的`Task`对象。
```javascript
const sagaMiddleware = createSagaMiddleware();
const store = createStore(rootReducer, initialState, compose(applyMiddleware(sagaMiddleware)));
const task = sagaMiddleware.run(getTodos);
task.done.then(() => {
  const html = ReactDOMServer.renderToString(/* ... */);
  const state = store.getState();
  renderFullPage(html, state);
});
```
至此，我们貌似已经能够比较完美地在服务端复用saga effects了。

### 更为复杂的saga

如果我们的saga比较复杂呢？比如像这样的：
```javascript
function* loginFlow() {
  while (true) {
    yield take('user/login');
    const payload = yield call(APIService.login);
    yield put({ type: 'user/login/success', payload });
    yield take('user/logout');
    yield call(APIService.logout);
    yield put({ type: 'user/logout/success' });
  }
}
```
这个task是一个典型的infinite saga flow，也是redux-saga相对于其他effects所独有的特性：我们可以随心所欲地定义“看起来是阻塞”的数据流任务，来解决复杂的业务场景，而无需担心阻塞任务会对UI线程造成影响。
这样的死循环saga数据流在客户端用起来是很高效优雅的，但到了服务端，这将造成严重的问题——这个saga永远不会结束，因此`task.done.then()`永远不会被回调，我们无法知道我们所需的数据什么时候加载完成。

对于更为普遍的情况，我们是这样定义saga任务的，比如使用蚂蚁的 [ant-design/antd-init](https://github.com/ant-design/antd-init) 脚手架：
*src/sagas/todos.js* 中定义了todos的saga：
```javascript
function* getTodos() {
  const { jsonResult } = yield call(getAll);
  if (jsonResult.data) {
    yield put({
      type: 'todos/get/success',
      payload: jsonResult.data,
    });
  }
}
function* watchTodosGet() {
  yield takeLatest('todos/get', getTodos)
}
export default function* () {
  yield fork(watchTodosGet);
  yield put({ type: 'todos/get', });
}
```
*src/sagas/index.js* 负责组合全部model的saga（通过`fork()`调用），并导出一个`rootSaga`：
```javascript
const context = require.context('./', false, /\.js$/);
const keys = context.keys().filter(item => item !== './index.js' && item !== './SagaManager.js');
export default function* root() {
  for (let i = 0; i < keys.length; i ++) {
    yield fork(context(keys[i]));
  }
}
```
请注意这里的`takeLatest()`调用。`takeLatest()`是redux-saga的一个helper方法，而不是effect方法。参考 [redux-saga API文档中的takeLatest](http://yelouafi.github.io/redux-saga/docs/api/index.html#takelatestpattern-saga-args)，我们可以看到`takeLatest()`是这样实现的：
```javascript
function* takeLatest(pattern, saga, ...args) {
  const task = yield fork(function* () {
    let lastTask
    while (true) {
      const action = yield take(pattern)
      if (lastTask)
        yield cancel(lastTask)
      lastTask = yield fork(saga, ...args.concat(action))
    }
  })
  return task
}
```
所以，当我们在saga中进行了一次`yield takeLatest()`之后，实际上是`fork()`出了一个带死循环数据流的另一个saga，而这个死循环的saga当然是永远不会结束的，除非它被我们人为`cancel()`。
还有一个问题是关于redux-saga的fork模型：被`fork()`出来的子saga与其父saga有怎样的生命周期关联？[redux-saga的官方文档](http://yelouafi.github.io/redux-saga/docs/advanced/ForkModel.html) 给了我们最好的回答：
> In fact, attached forks shares the same semantics with the parallel Effect:
> * We're executing tasks in parallel
> * The parent will terminate after all launched tasks terminate

意思是，父saga只有当其所有`fork()`出来的子saga都结束后才会结束（这和操作系统的fork模型是类似的）。这意味着，因为其子saga中带有死循环流，我们的`rootSaga`也是永远不会自发结束的。这样的话，我们就 **不能** 这么写：
```javascript
const task = sagaMiddleware.run(rootSaga);
store.dispatch({ type: 'todos/get' });
task.done.then(() => {
  // 这里的代码不会被执行
});
```
我们只能够直接`run()`不带死循环流的saga来获得初始数据，像这样：
```javascript
const task = sagaMiddleware.run(getTodos);
task.done.then(() => {
  const html = ReactDOMServer.renderToString(/* ... */);
  const state = store.getState();
  renderFullPage(html, state);
});
```
这跟我们刚才提到的官方建议的方法没有任何区别。在服务端我们需要规避那些包含死循环流的saga，如`watchTodosGet`。

这将导致客户端和服务端出现大量的 **异构** ：在客户端，我们直接执行`rootSaga`，通过`dispatch()`特定的action来获取数据并同步到state；而在服务端，我们需要找到并执行可以获取到数据并且不带死循环的saga，如`getTodos`。

### 使用redux-wait-for-action来搭救
为了将 **同构** 进行到底，博主写了一个Redux middleware来解决这个问题： [redux-wait-for-action](https://github.com/Chion82/redux-wait-for-action) 。这个代码不到80行的middleware主要实现了：在dispatch一个action时，同时指定另外一个我们期望收到的action，`store.dispatch()`返回一个promise，当这个我们期望的action到达时，该promise将resolve。
这样，我们可以在服务端复用`rootSaga`而不需要关心这个`rootSaga`何时结束。同时，在服务端创建的`store`，其生命周期将在http响应完成后结束，我们甚至不需要手动`cancel()`这个看似不会自发结束的`rootSaga`——交给GC来杀死它们就行了。
我们不妨写一个在客户端和服务端通用的`configureStore()`方法来创建我们的`store`，并且执行我们的`rootSaga`：
```javascript
const configureStore = (initialState) => {
  const sagaMiddleware = createSagaMiddleware();
  let enhancer = compose(
    applyMiddleware(sagaMiddleware),
    applyMiddleware(createReduxWaitForMiddleware()),
  );
  const store = createStore(rootReducer, initialState, enhancer);
  sagaMiddleware.run(rootSaga);
  return store;
};
```
在服务端渲染逻辑中，我们只需要直接`dispatch()`这个action即可——这和在客户端获取数据的方式完全相同：
```javascript
const store = configureStore({});
store.dispatch({
  type: 'todos/get',
  [ WAIT_FOR_ACTION ]: 'todos/get/success',
}).then(() => {
  const html = ReactDOMServer.renderToString(/* ... */);
  const state = store.getState();
  renderFullPage(html, state);
})
```
在上面的示例代码中，我们在`dispatch()`一个action时，在这个action中增加了一个属性`WAIT_FOR_ACTION`（`WAIT_FOR_ACTION`是一个从`redux-wait-for-action`导入的ES6 Symbol对象，因此你不需担心这会污染你的action），该属性指定了另一个我们所期望的action `todos/get/success`。这个`store.dispatch()`调用返回一个promise，当action `todos/get/success`到达时，这个promise将resolve，因此我们可以在它的`.then()`中填写我们的渲染逻辑——因为这时我们所需的数据已经准备好。

由于redux-wait-for-action是基于等待action的，它将适用于近乎全部的effects方案（当然，对于redux-thunk和redux-promise则没有这个必要），当以后有更为流行的effects方案时，我们仍然可以使用这个middleware。
关于更具体的使用方法，大家可以参考 [README for redux-wait-for-action](https://github.com/Chion82/redux-wait-for-action) 。

更优雅地组织同构应用
----------------
以上示例都是基于在服务端进行路由判断并决策执行哪个effects的，当我们的数据模型变得多时，服务端代码将变得复杂。比如：该dispatch `todos/get`还是`profile/get`？我们需要对`req.url`进行一一判断。

借助react-router的`match()`方法，我们能够得到对应路由下的container组件，如果我们能在每个路由下的container组件中定义一个`fetchData()`方法来dispatch合适的action，我们就可以大大简化服务端的代码，并且可以同时在服务端和客户端都使用它来加载页面数据。

在每个路由节点对应的container的代码中，添加一个`fetchData()` **静态** 方法：
```javascript
class TodosContainer extends Component {
  static fetchData(dispatch) {
    return dispatch({
      type: 'todos/get',
      [ WAIT_FOR_ACTION ]: 'todos/get/success',
    });
  }
  componentDidMount() {
    // 这个钩子方法仅会在客户端被调用
    TodosContainer.fetchData(this.props.dispatch);
  }
  // ...
}
```
在服务端渲染代码中，我们定义一个`getReduxPromise()`函数，这个函数抽出当前路由下对应的container组件，并调用其中的`fetchData()`方法，从而得到一个promise。
```javascript
match({history, routes, location: req.url}, (error, redirectLocation, renderProps) => {
  /* 前面这里需要处理redirectLocation、error和renderProps为null的情况 */
  /* ... */
  const getReduxPromise = () => {
    const component = renderProps.components[renderProps.components.length - 1].WrappedComponent;
    const promise = component.fetchData ?
      component.fetchData(store.dispatch) :
      Promise.resolve();
    return promise;
  };
  getReduxPromise().then(() => {
    const initStateString = JSON.stringify(store.getState());
    const html = ReactDOMServer.renderToString(
      <Provider store={store}>
        { <RouterContext {...renderProps}/> }
      </Provider>
    );
    res.status(200).send(renderFullPage(html, initStateString));
  });
});
```
遇到需要传递cookie或参数的情况，我们可以稍微修改一下`fetchData()`：
```javascript
static fetchData(dispatch, query, cookies) {
  return dispatch({
    type: 'todos/get',
    [ WAIT_FOR_ACTION ]: 'todos/get/success',
    query, cookies,
  })
}
```
在服务端调用`fetchData()`时：
```javascript
component.fetchData(store.dispatch, req.query, req.cookies);
```
由于客户端一般不需要在XHR中显式加cookie，因此我们在客户端调用`fetchData()`时忽略`cookies`参数即可，并在`APIService`模块中做适当的判断。

另外，为了节省篇幅和便于理解，以上各处示例代码中均没有异常处理部分（或被去除）。在实际项目中，请务必在effects中添加`try-catch`逻辑，并在promise的处理部分添加`.catch()`异常处理方法。

博主的脚手架
----------
为了在实践中更好地理解以上所提到的最优化方案，博主写了这个脚手架，同时便于大家快速搭建同构React应用：
[react-redux-universal-minimal](https://github.com/Chion82/react-redux-universal-minimal)
