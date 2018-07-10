---
title: mixins是有害的（Mixins Considered Harmful）［下篇］
date: 2016-08-28 16:32:22
tags:
- front-end
- React
categories:
- 学习笔记
- 前端
---

[上篇](/2016/07/23/mixins-considered-harmful/)

原文：[Facebook React: Mixins Considered Harmful](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html)

> Migrating from Mixins
> Let’s make it clear that mixins are not technically deprecated. If you use React.createClass(), you may keep using them. We only say that they didn’t work well for us, and so we won’t recommend using them in the future.
Every section below corresponds to a mixin usage pattern that we found in the Facebook codebase. For each of them, we describe the problem and a solution that we think works better than mixins. The examples are written in ES5 but once you don’t need mixins, you can switch to ES6 classes if you’d like.
> We hope that you find this list helpful. Please let us know if we missed important use cases so we can either amend the list or be proven wrong!

从Mixins迁移
-----------
有一点需要说明的是，从技术上来讲，mixins不是被弃用的。如果你在使用`React.createClass()`，你可以继续使用它们。我们只是说它们对我们而言不能很好地运用，并且我们不推荐在未来中继续使用它们。下面的每一章节对应了我们在Facebook代码库中发现的mixin的使用场景。对于每种情况，我们会说明问题所在，并展示我们认为比使用mixins更好的解决方案。示例都使用ES5编写，但当你不再需要mixins时，你可以随心所欲地切换到ES6 classes。
我们希望你能从这个列表中得到帮助。如果我们缺漏了一些比较重要的应用场景，请告知我们，因此我们能拓展这个列表，或者证明其中的部分是错误的。

> Performance Optimizations
> One of the most commonly used mixins is PureRenderMixin. You might be using it in some components to prevent unnecessary re-renders when the props and state are shallowly equal to the previous props and state:

性能优化
-------
使用率最高的mixins之一是 [PureRenderMixin](https://facebook.github.io/react/docs/pure-render-mixin.html) 。你可能正在一些组件中使用它，当props和state跟上次的值是浅层相等时，可[避免不必要的重渲染](https://facebook.github.io/react/docs/advanced-performance.html#shouldcomponentupdate-in-action)。

```JavaScript
var PureRenderMixin = require('react-addons-pure-render-mixin');

var Button = React.createClass({
  mixins: [PureRenderMixin],

  // ...

});
```

### 解决方案
> To express the same without mixins, you can use the shallowCompare function directly instead:

为了达到相同的效果而不使用mixins，你可以直接使用[shallowCompare](https://facebook.github.io/react/docs/shallow-compare.html)。

```JavaScript
var shallowCompare = require('react-addons-shallow-compare');

var Button = React.createClass({
  shouldComponentUpdate: function(nextProps, nextState) {
    return shallowCompare(this, nextProps, nextState);
  },

  // ...

});
```

> If you use a custom mixin implementing a shouldComponentUpdate function with different algorithm, we suggest exporting just that single function from a module and calling it directly from your components.
>
> We understand that more typing can be annoying. For the most common case, we plan to introduce a new base class called React.PureComponent in the next minor release. It uses the same shallow comparison as PureRenderMixin does today.

如果你使用一个自定义的mixin，以不同的算法实现 `shouldComponentUpdate` 方法，我们建议从模块中导出该单一的方法，并在你的组件中直接调用它。
我们理解频繁的编码是令人不快的。对于更普遍的情况，我们计划在下一个小版本发布中引入一个新的基类`React.PureComponent`。它将使用浅层对比算法，正如今天的`PureRenderMixin`。

> Subscriptions and Side Effects
> The second most common type of mixins that we encountered are mixins that subscribe a React component to a third-party data source. Whether this data source is a Flux Store, an Rx Observable, or something else, the pattern is very similar: the subscription is created in componentDidMount, destroyed in componentWillUnmount, and the change handler calls this.setState().

订阅和副作用
----------
我们遇到的第二种最常见的mixins类型是那些用来订阅React组件到第三方数据源的mixins。无论这些数据源是一个Flux Store，还是一个Rx Observable，抑或是其他的，该模式都是相似的：订阅在`componentDidMount`中产生，在`componentWillUnmount`中被销毁，而变更处理函数将调用 `this.setState()`。
```JavaScript
var SubscriptionMixin = {
  getInitialState: function() {
    return {
      comments: DataSource.getComments()
    };
  },

  componentDidMount: function() {
    DataSource.addChangeListener(this.handleChange);
  },

  componentWillUnmount: function() {
    DataSource.removeChangeListener(this.handleChange);
  },

  handleChange: function() {
    this.setState({
      comments: DataSource.getComments()
    });
  }
};

var CommentList = React.createClass({
  mixins: [SubscriptionMixin],

  render: function() {
    // Reading comments from state managed by mixin.
    var comments = this.state.comments;
    return (
      <div>
        {comments.map(function(comment) {
          return <Comment comment={comment} key={comment.id} />
        })}
      </div>
    )
  }
});

module.exports = CommentList;
```

> Solution
>
> If there is just one component subscribed to this data source, it is fine to embed the subscription logic right into the component. Avoid premature abstractions.
>
> If several components used this mixin to subscribe to a data source, a nice way to avoid repetition is to use a pattern called “higher-order components”. It can sound intimidating so we will take a closer look at how this pattern naturally emerges from the component model.

### 解决方案

如果只有一个组件被订阅到这个数据源，直接将订阅逻辑内嵌到该组件中不失为一个良策。避免草率的抽象。

如果多个组件都使用这个mixin来订阅到一个数据源，一个好的避免重复冗余的方法是使用一种被称为“[高阶组件(higher-order components，又称HOC)](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750)”的模式。这听起来让人生畏，所以我们将仔细分析这个模式如何自然地套用到组件模型上。

> Higher-Order Components Explained
> Let’s forget about React for a second. Consider these two functions that add and multiply numbers, logging the results as they do that:

### 高阶组件的解释

让我们暂时忘记React。想想这两个实现相加和相乘的函数，通过这样来实现记录计算结果：
```JavaScript
function addAndLog(x, y) {
  var result = x + y;
  console.log('result:', result);
  return result;
}

function multiplyAndLog(x, y) {
  var result = x * y;
  console.log('result:', result);
  return result;
}
```

> These two functions are not very useful but they help us demonstrate a pattern that we can later apply to components.
>
> Let’s say that we want to extract the logging logic out of these functions without changing their signatures. How can we do this? An elegant solution is to write a higher-order function, that is, a function that takes a function as an argument and returns a function.
>
> Again, it sounds more intimidating than it really is:

这两个函数并不是十分有用，但它们可以帮助我们描述一个典型的模式，这个模式我们之后将把它应用到组件上。

假设我们想从这些函数中抽离记录逻辑而不修改它们的签名。如何做到这点？一个优雅的方案是，写一个更高阶的函数，这个更高阶的函数实际上是一个将函数作为其参数，并返回一个新函数的函数。

又一次，它听起来让人生畏，但实际上它是更简单的：

```JavaScript
function withLogging(wrappedFunction) {
  // Return a function with the same API...
  return function(x, y) {
    // ... that calls the original function
    var result = wrappedFunction(x, y);
    // ... but also logs its result!
    console.log('result:', result);
    return result;
  };
}
```

> The withLogging higher-order function lets us write add and multiply without the logging statements, and later wrap them to get addAndLog and multiplyAndLog with exactly the same signatures as before:

这个 `withLogging` 高阶函数让我们在实现相加和相乘逻辑时不需考虑记录逻辑，在这之后我们通过嵌套的方式来得到与之前签名一致的 `addAndLog` 和 `multiplyAndLog`。

```JavaScript
function add(x, y) {
  return x + y;
}

function multiply(x, y) {
  return x * y;
}

function withLogging(wrappedFunction) {
  return function(x, y) {
    var result = wrappedFunction(x, y);
    console.log('result:', result);
    return result;
  };
}

// Equivalent to writing addAndLog by hand:
var addAndLog = withLogging(add);

// Equivalent to writing multiplyAndLog by hand:
var multiplyAndLog = withLogging(multiply);
```

> Higher-order components are a very similar pattern, but applied to components in React. We will apply this transformation from mixins in two steps.
>
> As a first step, we will split our CommentList component in two, a child and a parent. The child will be only concerned with rendering the comments. The parent will set up the subscription and pass the up-to-date data to the child via props.

高阶组件是一种非常相似的模式，只不过它是应用在React组件上的而已。我们将这种转换应用到mixins上，只需要两步即可。

第一步，我们将`CommentList`组件分为子和父两部分。子组件只关心渲染评论，而父组件将设置订阅，并将最新的数据通过props传递到子组件上。
```JavaScript
// This is a child component.
// It only renders the comments it receives as props.
var CommentList = React.createClass({
  render: function() {
    // Note: now reading from props rather than state.
    var comments = this.props.comments;
    return (
      <div>
        {comments.map(function(comment) {
          return <Comment comment={comment} key={comment.id} />
        })}
      </div>
    )
  }
});

// This is a parent component.
// It subscribes to the data source and renders <CommentList />.
var CommentListWithSubscription = React.createClass({
  getInitialState: function() {
    return {
      comments: DataSource.getComments()
    };
  },

  componentDidMount: function() {
    DataSource.addChangeListener(this.handleChange);
  },

  componentWillUnmount: function() {
    DataSource.removeChangeListener(this.handleChange);
  },

  handleChange: function() {
    this.setState({
      comments: DataSource.getComments()
    });
  },

  render: function() {
    // We pass the current state as props to CommentList.
    return <CommentList comments={this.state.comments} />;
  }
});

module.exports = CommentListWithSubscription;
```

> There is just one final step left to do.
>
> Remember how we made withLogging() take a function and return another function wrapping it? We can apply a similar pattern to React components.
>
> We will write a new function called withSubscription(WrappedComponent). Its argument could be any React component. We will pass CommentList as WrappedComponent, but we could also apply withSubscription() to any other component in our codebase.
>
> This function would return another component. The returned component would manage the subscription and render <WrappedComponent /> with the current data.
>
> We call this pattern a “higher-order component”.
>
> The composition happens at React rendering level rather than with a direct function call. This is why it doesn’t matter whether the wrapped component is defined with createClass(), as an ES6 class or a function. If WrappedComponent is a React component, the component created by withSubscription() can render it.

只剩下最后一步了。

还记得我们如何使得`withLogging()`传入一个函数并返回另一个嵌套它的函数吗？我们可以将相似的模式应用到React组件上来。

我们将编写一个新的函数，叫做`withSubscription(WrappedComponent)`。它的参数可以是任意的React组件。我们将传递`CommentList`作为`WrappedComponent`，但我们也可以在我们的代码基中将`withSubscription()`应用到任意其他的组件上。

这个函数会返回另一个组件。返回的组件将会管理好订阅，并渲染包含数据的`<WrappedComponent />`。

我们把这种模式称为一个“高阶组件”。

这种合成发生在React的渲染层，而不是通过一个直接的函数调用。这就是为什么无论内嵌的组件是由`createClass()`创建的，还是由ES6 class生成的，抑或是一个函数，都无关紧要了。如果`WrappedComponent`是一个React组件，通过`withSubscription()`创建的组件都能渲染它。

```JavaScript
// This function takes a component...
function withSubscription(WrappedComponent) {
  // ...and returns another component...
  return React.createClass({
    getInitialState: function() {
      return {
        comments: DataSource.getComments()
      };
    },

    componentDidMount: function() {
      // ... that takes care of the subscription...
      DataSource.addChangeListener(this.handleChange);
    },

    componentWillUnmount: function() {
      DataSource.removeChangeListener(this.handleChange);
    },

    handleChange: function() {
      this.setState({
        comments: DataSource.getComments()
      });
    },

    render: function() {
      // ... and renders the wrapped component with the fresh data!
      return <WrappedComponent comments={this.state.comments} />;
    }
  });
}
```

> Now we can declare CommentListWithSubscription by applying withSubscription to CommentList:

现在我们可以通过应用`withSubscription`到`CommentList`上来声明`CommentListWithSubscription`了。

```JavaScript
var CommentList = React.createClass({
  render: function() {
    var comments = this.props.comments;
    return (
      <div>
        {comments.map(function(comment) {
          return <Comment comment={comment} key={comment.id} />
        })}
      </div>
    )
  }
});

// withSubscription() returns a new component that
// is subscribed to the data source and renders
// <CommentList /> with up-to-date data.
var CommentListWithSubscription = withSubscription(CommentList);

// The rest of the app is interested in the subscribed component
// so we export it instead of the original unwrapped CommentList.
module.exports = CommentListWithSubscription;
```

> Solution, Revisited
> Now that we understand higher-order components better, let’s take another look at the complete solution that doesn’t involve mixins. There are a few minor changes that are annotated with inline comments:

### 解决方案，重现
现在我们能更好的理解高阶组件了，让我们来再看一次完整的、无需涉及mixins的解决方案。内联的注释有少量修改。

```JavaScript
function withSubscription(WrappedComponent) {
  return React.createClass({
    getInitialState: function() {
      return {
        comments: DataSource.getComments()
      };
    },

    componentDidMount: function() {
      DataSource.addChangeListener(this.handleChange);
    },

    componentWillUnmount: function() {
      DataSource.removeChangeListener(this.handleChange);
    },

    handleChange: function() {
      this.setState({
        comments: DataSource.getComments()
      });
    },

    render: function() {
      // Use JSX spread syntax to pass all props and state down automatically.
      return <WrappedComponent {...this.props} {...this.state} />;
    }
  });
}

// Optional change: convert CommentList to a functional component
// because it doesn't use lifecycle hooks or state.
function CommentList(props) {
  var comments = props.comments;
  return (
    <div>
      {comments.map(function(comment) {
        return <Comment comment={comment} key={comment.id} />
      })}
    </div>
  )
}

// Instead of declaring CommentListWithSubscription,
// we export the wrapped component right away.
module.exports = withSubscription(CommentList);
```

> Higher-order components are a powerful pattern. You can pass additional arguments to them if you want to further customize their behavior. After all, they are not even a feature of React. They are just functions that receive components and return components that wrap them.
>
> Like any solution, higher-order components have their own pitfalls. For example, if you heavily use refs, you might notice that wrapping something into a higher-order component changes the ref to point to the wrapping component. In practice we discourage using refs for component communication so we don’t think it’s a big issue. In the future, we might consider adding ref forwarding to React to solve this annoyance.

高阶组件是一个强大的模式。你可以给它们传递更多的参数，如果你想要进一步高度定制它们的行为。毕境，它们甚至不是React的特性之一。它们只是接受传入组件，并返回嵌套了传入组件的新组件的函数而已。

就像其它解决方案，高阶函数同样有他们的潜在风险。比如，如果你大量地使用refs（组件引用），你可能会发现，将任意组件嵌套进高阶组件里面时，内层组件的ref会被改变。在实践中我们不建议使用refs来实现组件间通信，所以我们不认为这是个大问题。在未来，我们将考虑引入ref重定向到React中来解决这个问题。

> Rendering Logic
> The next most common use case for mixins that we discovered in our codebase is sharing rendering logic between components.
>
> Here is a typical example of this pattern:

渲染逻辑
-------
在我们的代码库中，我们发现的下一个常见的mixins用例是组件间渲染逻辑的共享。

以下是这个模式的典型例子：

```JavaScript
var RowMixin = {
  // Called by components from render()
  renderHeader: function() {
    return (
      <div className='row-header'>
        <h1>
          {this.getHeaderText() /* Defined by components */}
        </h1>
      </div>
    );
  }
};

var UserRow = React.createClass({
  mixins: [RowMixin],

  // Called by RowMixin.renderHeader()
  getHeaderText: function() {
    return this.props.user.fullName;
  },

  render: function() {
    return (
      <div>
        {this.renderHeader() /* Defined by RowMixin */}
        <h2>{this.props.user.biography}</h2>
      </div>
    )
  }
});
```

> Multiple components may be sharing RowMixin to render the header, and each of them would need to define getHeaderText().

多个组件可能共享了`RowMixin`来渲染行头，而每个这些组件都需要定义一个`getHeaderText()`方法。

> Solution
>
> If you see rendering logic inside a mixin, it’s time to extract a component!
>
> Instead of RowMixin, we will define a <Row> component. We will also replace the convention of defining a getHeaderText() method with the standard mechanism of top-data flow in React: passing props.
>
> Finally, since neither of those components currently need lifecycle hooks or state, we can declare them as simple functions:

### 解决方案

如果你看见了一个mixin里面含有渲染逻辑，那么是时候把它们抽离到组件中了！

我们将定义一个`<Row>`组件来取代`RowMixin`。我们也将会把借由定义一个`getHeaderText()`方法来实现转换的方式替换成React中标准的自顶向下数据流机制：传递props。

最后，因为这些组件现在都不再需要生命周期钩子和状态了，我们会把他们定义为简单的函数：

```JavaScript
function RowHeader(props) {
  return (
    <div className='row-header'>
      <h1>{props.text}</h1>
    </div>
  );
}

function UserRow(props) {
  return (
    <div>
      <RowHeader text={props.user.fullName} />
      <h2>{props.user.biography}</h2>
    </div>
  );
}
```

> Props keep component dependencies explicit, easy to replace, and enforceable with tools like Flow and TypeScript.

Props使得组件依赖保持显式、易于替换、对诸如Flow和TypeScript一类的工具更易执行。

> Note:
>
> Defining components as functions is not required. There is also nothing wrong with using lifecycle hooks and state—they are first-class React features. We use functional components in this example because they are easier to read and we didn’t need those extra features, but classes would work just as fine.

备注：
将组件定义为函数不是必需的。使用React的头等特性：生命周期钩子和状态也是没有任何错误的。我们在这个示例中使用函数式组件，因为它们可以更易于阅读，并且我们不需要那些另外的特性，但使用classes也是一样的效果。

> Context
> Another group of mixins we discovered were helpers for providing and consuming React context. Context is an experimental unstable feature, has certain issues, and will likely change its API in the future. We don’t recommend using it unless you’re confident there is no other way of solving your problem.
>
> Nevertheless, if you already use context today, you might have been hiding its usage with mixins like this:

上下文（Context）
--------------
我们发现的另外一系列mixins是提供和消费React Context的辅助器。Context是一个实验性的不稳定特性，存在确定的缺陷，而且它的API在未来可能会被改变。我们不推荐使用它，除非你十分确定没有其他方法来解决你的问题。

尽管如此，如果你已经使用了context，你可能把它的使用隐藏在了mixins里，就像这样：

```JavaScript
var RouterMixin = {
  contextTypes: {
    router: React.PropTypes.object.isRequired
  },

  // The mixin provides a method so that components
  // don't have to use the context API directly.
  push: function(path) {
    this.context.router.push(path)
  }
};

var Link = React.createClass({
  mixins: [RouterMixin],

  handleClick: function(e) {
    e.stopPropagation();

    // This method is defined in RouterMixin.
    this.push(this.props.to);
  },

  render: function() {
    return (
      <a onClick={this.handleClick}>
        {this.props.children}
      </a>
    );
  }
});

module.exports = Link;
```

> Solution
> We agree that hiding context usage from consuming components is a good idea until the context API stabilizes. However, we recommend using higher-order components instead of mixins for this.
>
> Let the wrapping component grab something from the context, and pass it down with props to the wrapped component:

### 解决方案
在context的API稳定之前，我们认为，将context的调用在组件中隐藏起来是个好主意。不过，我们推荐使用高阶组件来取代mixins来实现这点。

让外层组件从context中获取数据，并通过props传递到内层组件中：
```JavaScript
function withRouter(WrappedComponent) {
  return React.createClass({
    contextTypes: {
      router: React.PropTypes.object.isRequired
    },

    render: function() {
      // The wrapper component reads something from the context
      // and passes it down as a prop to the wrapped component.
      var router = this.context.router;
      return <WrappedComponent {...this.props} router={router} />;
    }
  });
};

var Link = React.createClass({
  handleClick: function(e) {
    e.stopPropagation();

    // The wrapped component uses props instead of context.
    this.props.router.push(this.props.to);
  },

  render: function() {
    return (
      <a onClick={this.handleClick}>
        {this.props.children}
      </a>
    );
  }
});

// Don't forget to wrap the component!
module.exports = withRouter(Link);
```

> If you’re using a third party library that only provides a mixin, we encourage you to file an issue with them linking to this post so that they can provide a higher-order component instead. In the meantime, you can create a higher-order component around it yourself in exactly the same way.

如果你在使用一个只提供mixin的第三方库，我们建议你去提交一个issue，引用本文链接，让他们去做成高阶组件。在这期间，通过完全一样的方式，你可以自己动手围绕它做一个高阶组件。

> Utility Methods
> Sometimes, mixins are used solely to share utility functions between components:

通用方法
-------
有时候，mixins仅仅是用作在组件间共享的通用工具函数。
```JavaScript
var ColorMixin = {
  getLuminance(color) {
    var c = parseInt(color, 16);
    var r = (c & 0xFF0000) >> 16;
    var g = (c & 0x00FF00) >> 8;
    var b = (c & 0x0000FF);
    return (0.299 * r + 0.587 * g + 0.114 * b);
  }
};

var Button = React.createClass({
  mixins: [ColorMixin],

  render: function() {
    var theme = this.getLuminance(this.props.color) > 160 ? 'dark' : 'light';
    return (
      <div className={theme}>
        {this.props.children}
      </div>
    )
  }
});
```

> Solution
> Put utility functions into regular JavaScript modules and import them. This also makes it easier to test them or use them outside of your components:

### 解决方案
将通用的工具方法放入常规的JavaScript模块中，并引入它们。这同样使得测试和组件外调用变得简单：

```JavaScript
var getLuminance = require('../utils/getLuminance');

var Button = React.createClass({
  render: function() {
    var theme = getLuminance(this.props.color) > 160 ? 'dark' : 'light';
    return (
      <div className={theme}>
        {this.props.children}
      </div>
    )
  }
});
```

> Other Use Cases
Sometimes people use mixins to selectively add logging to lifecycle hooks in some components. In the future, we intend to provide an official DevTools API that would let you implement something similar without touching the components. However it’s still very much a work in progress. If you heavily depend on logging mixins for debugging, you might want to keep using those mixins for a little longer.
>
> If you can’t accomplish something with a component, a higher-order component, or a utility module, it could be mean that React should provide this out of the box. File an issue to tell us about your use case for mixins, and we’ll help you consider alternatives or perhaps implement your feature request.
>
> Mixins are not deprecated in the traditional sense. You can keep using them with React.createClass(), as we won’t be changing it further. Eventually, as ES6 classes gain more adoption and their usability problems in React are solved, we might split React.createClass() into a separate package because most people wouldn’t need it. Even in that case, your old mixins would keep working.
>
> We believe that the alternatives above are better for the vast majority of cases, and we invite you to try writing React apps without using mixins.

其他用例
------
有时候，人们使用mixins来向一些组件添加选择性的生命周期钩子日志记录。在未来，我们计划提供一个官方的开发工具API来实现相似功能，而无需触碰组件代码。虽然这仍有大量正在进度中的工作需要完成。如果你十分依赖日志记录mixins来调试，你可能还要继续保持使用它们一段时间。

如果你借助一个组件、一个高阶组件、或者一个通用模块，仍然不能完成一些事情，这意味着React应该是难以完成这样的事情的。向我们提交一个issue，告诉我们你的mixins使用场景，我们会帮助你考虑可选的方案，或者是在未来实现你的新特性请求。

Mixins在传统感官中不是完全抛弃的。你可以通过`React.createClass()`继续使用它们，因为我们不会在未来修改它。最终，当ES6 classes得到更广泛的采用，并且它们在React中使用上的问题得到解决时，我们也许会将`React.createClass()`分离到独立的包之中，因为大多数人不再需要它。即使是在那样的情况下，你的老mixins仍然能够继续工作。

我们相信，以上所提到的可选方案对于绝大多数的场景是更好的选择，我们邀请你来尝试在不使用mixins的情况下编写React应用。
