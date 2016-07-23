---
title: 自译：如何使用服务端渲染加速React APP首屏加载
date: 2016-03-03 20:11:17
tags:
- front-end
- React
categories:
- 学习笔记
- 前端
---

原文：[How to build React apps that load quickly using server side rendering (by Stefan Fidanov)](https://www.terlici.com/2015/03/18/fast-react-loading-server-rendering.html)

使用客户端框架（译者注：此处指大多数在浏览器端运行的前端MV*框架）可快速开发用户交互丰富、性能高效的web app，前端开发者都非常喜欢使用该类框架。
不幸的是，客户端框架也有缺点，其中最主要的问题是首屏加载速度。
客户端首先从服务器接收少量的HTML代码，但是之后却需要接收大量的JavaScript代码。
然后，它们（指前端框架）需要向服务器请求数据，等待收到数据，进行必要的数据处理，并最终渲染到用户的浏览器上。
相比之下，传统的web做法是，全部数据由服务端进行渲染，当服务端向用户首次递交HTML时，用户端浏览器就收到了渲染完成的页面了。
再者，大多数情况下，web服务器的渲染速度要快于客户端的渲染。所以，（传统web）的首屏渲染是非常快速的。

React的解决方案
-------------
很自然的，你会想同时拥有上述两者（分别指：使用了MV*框架的web app、传统的web站点）的全部优点。快速的首屏加载、高度的交互性和快速的响应。React可以帮助你同时做到这几点。
React是这样做到的：首先，它可以在服务端渲染任意的组件（Component），包括这些组件的数据，这样渲染得到的结果是一些HTML代码，这些HTML代码在这之后可以直接发送到浏览器。
当这些HTML在用户浏览器上被显示出来时，React会在本地（这里的本地指用户浏览器）进行计算。它的智能算法将进行判断并得出：React即将要在浏览器端动态渲染出来的结果，跟当前已经被显示出来的页面一样。
在这之后，除了添加必要的事件处理，React不会对页面做任何的修改。
那么为什么这样会更快呢？我们不是在做几乎跟客户端一样的事情吗？
是的。但仅仅是“几乎”而已。
首先，当服务器响应浏览器请求时，用户马上就能看到整个页面了。所以页面响应速度更快了。
其次，因为React能够判断出无需再对DOM做修改，它就不会再去碰DOM。修改DOM是前端渲染中最慢的部分。
再者，这样可以节省请求次数。因为所有数据已经被渲染所以React不需要再向服务器请求。

**那么有没有可能：当页面加载时，页面已经显示出来但是用户不能对其进行交互，因为这时事件处理尚未被添加？**  
理论上这种情况是有可能发生的。但是因为用了服务端渲染，我们就避免了所有的高开销操作，而且这样不但加速了页面响应速度，添加事件处理的速度也会变得很快。
所以，你的应用将总是可交互的，并且用户不会察觉到有什么问题。

示例
---
光说无用，我们来看看如何在代码中实现。我们的第一个示例是非常简单的。我们要显示一个"hello"消息，并且点击后会有提示。
我们的示例将使用NodeJS作为服务端部分，不过这里的一切都可以应用在其他平台，比如PHP, Ruby, Python, Java或者.NET。

我们需要以下Node模块：
```shell
$ npm install babel react react-dom express jade
```
我们将使用`express`和`jade`来做一个示例服务器。
`react`和`react-dom`包可提供React组件的服务端渲染。
`babel`包允许我们通过node直接加载JSX模块，比如`require('some-component.jsx')`或者`require('some-component.js')`。
`babel`实际上更加强大。现在你可以用ES6支持。
我们的应用只有3个文件，文件结构如下：
```html
public/components.js
views/index.jade
app.js
```
`components.js`包含了我们的React组件；`index.jade`是网站的基本模板文件，将会加载全部JavaScript；`app.js`是node服务器。
让我们来看看`index.jade`里面有什么内容：
```html
doctype
html
  head
    title React Server Side Rendering Example
  body
    div(id='react-root')!= react

    script(src='https://fb.me/react-0.14.0.js')
    script(src='https://fb.me/react-dom-0.14.0.js')
    script(src='https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.23/browser.min.js')

    script(src='/components.js', type='text/babel')
```
`div(id='react-root')!= react`是最关键的部分。它的作用是作为React根组件的容器。另外，`react`变量的值是服务端渲染React组件后得到的HTML。
前两个引用进来的JavaScript文件是React本身，如果你想要在组件里面使用JSX，还需要引用一个Babel。
最后一个引用的文件是具体的组件。我们要把type设成`text/babel`好让Babel来处理这个文件。
这将提供一个基本的HTML结构，并加载全部的JavaScript和你需要的React组件。
来看看这个简单的服务器：
```JavaScript
require('babel/register')

var express = require('express')
  , app = express()
  , React = require('react')
  , ReactDOM = require('react-dom/server')
  , components = require('./public/components.js')

var HelloMessage = React.createFactory(components.HelloMessage)


app.engine('jade', require('jade').__express)
app.set('view engine', 'jade')

app.use(express.static(__dirname + '/public'))

app.get('/', function(req, res){
  res.render('index', {
    react: ReactDOM.renderToString(HelloMessage({name: "John"}))
  })
})

app.listen(3000, function() {
  console.log('Listening on port 3000...')
})
```
这部分代码中，大部分和一个普通的express应用程序没有多大区别。但是其中有些行需要注意。
第一行：
```JavaScript
require('babel/register')
```
加载Babel到你的依赖。这么做，你可以直接导入(`require()`)由JSX组成的React组件，它们会被自动翻译为JavaScript，就像后面的两行：
```JavaScript
var components = require('./public/components.js')

var HelloMessage = React.createFactory(components.HelloMessage)
```
在上面的代码中，第一行导入JSX编写的React组件。然后，由`React.createFactory`生成一个函数，该函数可以创建`HelloMessage`的组件。
```JavaScript
app.get('/', function(req, res){
  res.render('index', {
    react: ReactDOM.renderToString(HelloMessage({name: "John"}))
  })
})
```
上面这里就是渲染React组件的代码，并且渲染包含该组件的页面然后发送至浏览器。
首先，使用值为`John`的`name`属性创建一个新的`HelloMessage`组件，然后使用`React.renderToString`将这个组件渲染为HTML。
这里需要注意的是，组件仅仅被渲染(rendered)，而没有被挂载(mounted)，所以 **所有关于挂载的方法都不会被调用** 。
在创建组件之后，将组件的HTML传递到index模版。
我们的组件看起来是这样的：
```JavaScript
var isNode = typeof module !== 'undefined' && module.exports
  , React = isNode ? require('react') : window.React
  , ReactDOM = isNode ? require('react') : window.ReactDOM

var HelloMessage = React.createClass({
  handleClick: function () {
    alert('You clicked!')
  },

  render: function() {
    return <div onClick={this.handleClick}>Hello {this.props.name}</div>
  }
})

if (isNode) {
  exports.HelloMessage = HelloMessage
} else {
  ReactDOM.render(<HelloMessage name="John" />, document.getElementById('react-root'))
}
```
你可以看见，这跟一般的由JSX编写的React组件没有什么不同，除了开头和结尾。这里就是你要让组件能同时在浏览器和Node端运行所需要注意的地方。
（译者注：这里`isNode`的获取方法不适用于使用webpack, browserify等bundler构建的组件，因为构建工具本身也需要借助node运行。这种情况下，通过判断`express`是否存在，或者手动指定该值会更好。）

高级示例：加载服务端数据
--------------------
真正的Web app做的事情通常远不止你看见的这些。它们经常需要跟服务器交互并从服务器加载数据。
但是，我们不希望这在服务端渲染时发生。
我们来对这个示例程序做一些小修改。首先，模版文件需要引用jQuery，在这里它的唯一作用是从服务端请求数据。
```html
doctype
html
  head
    title React Server Side Rendering Example
  body
    div(id='react-root')!= react

    script(src='https://fb.me/react-0.14.0.js')
    script(src='https://fb.me/react-dom-0.14.0.js')
    script(src='https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.23/browser.min.js')
    script(src='http://code.jquery.com/jquery-2.1.3.js')

    script(src='/components.js', type='text/babel')
```
我们的服务器现在需要增加一个请求路由。
```JavaScript
require('babel/register')

var express = require('express')
  , app = express()
  , React = require('react')
  , ReactDOM = require('react-dom/server')
  , components = require('./public/components.js')

var HelloMessage = React.createFactory(components.HelloMessage)


app.engine('jade', require('jade').__express)
app.set('view engine', 'jade')

app.use(express.static(__dirname + '/public'))

app.get('/', function(req, res){
  res.render('index', {
    react: React.renderToString(HelloMessage({name: "John"}))
  })
})

app.get('/name', function(req, res){
  res.send("Paul, " + new Date().toString())
})

app.listen(3000, function() {
  console.log('Listening on port 3000...')
})
```
这里跟之前的例子唯一的不同之处在于这三行：
```JavaScript
app.get('/name', function(req, res){
  res.send("Paul, " + new Date().toString())
})
```
这三行代码的作用是，当`/name`被请求时，返回名字`Paul`和当前时间。
来看看这整个应用最有趣和最重要的部分，即React组件：
```JavaScript
var isNode = typeof module !== 'undefined' && module.exports
  , React = isNode ? require('react') : window.React
  , ReactDOM = isNode ? require('react-dom') : window.ReactDOM

var HelloMessage = React.createClass({
  getInitialState: function () {
    return {}
  },

  loadServerData: function() {
    $.get('/name', function(result) {
      if (this.isMounted()) {
        this.setState({name: result})
      }
    }.bind(this))
  },

  componentDidMount: function () {
    this.intervalID = setInterval(this.loadServerData, 3000)
  },

  componentWillUnmount: function() {
    clearInterval(this.intervalID)
  },

  handleClick: function () {
    alert('You clicked!')
  },

  render: function() {
    var name = this.state.name ? this.state.name : this.props.name
    return <div onClick={this.handleClick}>Hello {name}</div>
  }
})

if (isNode) {
  exports.HelloMessage = HelloMessage
} else {
  ReactDOM.render(<HelloMessage name="John" />, document.getElementById('react-root'))
}
```
我们只添加了这4个方法，其他和之前的例子相同：
```JavaScript
getInitialState: function () {
  return {}
},

loadServerData: function() {
  $.get('/name', function(result) {
    if (this.isMounted()) {
      this.setState({name: result})
    }
  }.bind(this))
},

componentDidMount: function () {
  this.intervalID = setInterval(this.loadServerData, 3000)
},

componentWillUnmount: function() {
  clearInterval(this.intervalID)
},
```
当组件被挂载后，每隔3秒它会向服务器请求数据`/name`并且显示出来。
`componentDidMount`和`componentWillUnmount`在组件被渲染时是不会被调用的，它们只有在组件被挂载时才会被调用。
所以这两个方法在服务端渲染时不会被调用，`loadServerData`方法也不会被调用。
这三个方法只有当组件被挂载时才会被执行，而这只会发生在浏览器端。
由此可见，想要从整体中分离出只在浏览器运行的那部分，并且保持代码的复用是很简单的。

在这之后？
-------
你已经学会了如何借助服务端渲染创建一个能被快速加载的React应用程序。但是，我的这个示例只是针对NodeJS服务器。
如果你在使用其他技术（比如PHP, .NET, Ruby, Python或者Java），你一样可以利用React服务端渲染的优点，这将会是你下一步要研究的方向。
另外，我直接在浏览器端使用了JSX，这将多亏于Babel，但是这也会降低性能。在生产环境中，在将JSX提供给浏览器之前先将之转换为JavaScript会更快。
我相信你一定可以找到你最喜欢的开发语言和Web框架下的类似解决方案。
