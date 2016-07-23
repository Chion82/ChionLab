---
title: 自译：不要再用mixins了（Mixins Considered Harmful）［上篇］
date: 2016-07-23 20:12:22
tags:
- front-end
- React
categories:
- 学习笔记
- 前端
---

原文：[Facebook React: Mixins Considered Harmful](https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html)

> “How do I share the code between several components?” is one of the first questions that people ask when they learn React. Our answer has always been to use component composition for code reuse. You can define a component and use it in several other components.

“我如何在多个组件（components）之间共享代码？”，这是React初学者的问题之一。我们的答案一直都是，通过组件组合的方法来实现代码复用。你可以定义一个组件，并在其它的组件中使用它。

> It is not always obvious how a certain pattern can be solved with composition. React is influenced by functional programming but it came into the field that was dominated by object-oriented libraries. It was hard for engineers both inside and outside of Facebook to give up on the patterns they were used to.

通过组件组合的方式来解决某一种情况不总是显而易见的。React受函数式编程影响，但结果它成为了由面向对象库组成的存在。无伦是Facebook内部员工，还是非Facebook的程序员，抛弃以往的开发方式都是困难的。

> To ease the initial adoption and learning, we included certain escape hatches into React. The mixin system was one of those escape hatches, and its goal was to give you a way to reuse code between components when you aren’t sure how to solve the same problem with composition.

为了让入门学习变得简单，我们引入了一些解决方案（原文“escape hatches”即逃生舱，此处语义为解决问题的一些trick）。Mixin系统是其中的一个方法，它的目的是，当你不知道如何通过组件组合来解决问题时，来给你一个方法来实现组件间的代码复用。

> Three years passed since React was released. The landscape has changed. Multiple view libraries now adopt a component model similar to React. Using composition over inheritance to build declarative user interfaces is no longer a novelty. We are also more confident in the React component model, and we have seen many creative uses of it both internally and in the community.
> In this post, we will consider the problems commonly caused by mixins. Then we will suggest several alternative patterns for the same use cases. We have found those patterns to scale better with the complexity of the codebase than mixins.

React发布后三年过去了，大环境发生了改变。大多数视图库现在都采用类似React的组件模型。通过多个组件在继承关系之上的组合来构建用户界面不再是一个新奇的方式。我们也对React的组件模型更加自信，并且在内部和社区中，都看到了许多具有创新性的使用方式。
在这篇文章中，我们会讨论由mixins造成的普遍问题。然后我们会提出一些同等情况下的可选替代方案。这些新的方案，在同等的代码复杂度下，比用mixins的可扩展性更好。

为什么说Mixins不好？
-----------------
> At Facebook, React usage has grown from a few components to thousands of them. This gives us a window into how people use React. Thanks to declarative rendering and top-down data flow, many teams were able to fix a bunch of bugs while shipping new features as they adopted React.

在Facebook，React的使用从少量的组件演变成上千的组件数量。这给我们看见了人们是如何使用React的。多亏于声明性的渲染和自上而下的数据流，很多团队能够在迁移项目到React的时候修复一些bug。

> However it’s inevitable that some of our code using React gradually became incomprehensible. Occasionally, the React team would see groups of components in different projects that people were afraid to touch. These components were too easy to break accidentally, were confusing to new developers, and eventually became just as confusing to the people who wrote them in the first place. Much of this confusion was caused by mixins. At the time, I wasn’t working at Facebook but I came to the same conclusions after writing my fair share of terrible mixins.

但是，一个很难避免的情况是，一些代码在使用了React了之后逐渐降低了可读性。有时，使用React的开发团队中会出现一些人们不太愿意去触碰的组件，而这些组件在不同的项目中被使用了。这些组件太容易意外损坏，这不但困扰了新加入的开发者，最终也困扰了一开始编写这些组件的人。这些麻烦的问题大多是由mixins造成的。在那时，我还未在Facebook工作，但在使用了一系列糟糕的mixins之后，我也能得出跟现在一样的结论。

> This doesn’t mean that mixins themselves are bad. People successfully employ them in different languages and paradigms, including some functional languages. At Facebook, we extensively use traits in Hack which are fairly similar to mixins. Nevertheless, we think that mixins are unnecessary and problematic in React codebases. Here’s why.

这并不代表mixins都是不好的。人们成功地在不同的语言和范例中应用了mixins，其中包括了一些函数式语言。在Facebook，我们大量使用了类似mixins的一些比较hack的实现方式。我们认为mixins在React中是不再必要的，而且是非常容易出问题的。接下来讨论这是为什么。

Mixins引入了隐性的依赖
-------------------
> Sometimes a component relies on a certain method defined in the mixin, such as getClassName(). Sometimes it’s the other way around, and mixin calls a method like renderHeader() on the component. JavaScript is a dynamic language so it’s hard to enforce or document these dependencies.
> Mixins break the common and usually safe assumption that you can rename a state key or a method by searching for its occurrences in the component file. You might write a stateful component and then your coworker might add a mixin that reads this state. In a few months, you might want to move that state up to the parent component so it can be shared with a sibling. Will you remember to update the mixin to read a prop instead? What if, by now, other components also use this mixin?

有时候一个组件依赖一个在mixin中定义的确定的方法，比如`getClassName()`。有时候在另一个场景下，mixin在组件上调用了一个方法，比如`renderHeader()`。JavaScript是一种动态语言，所以去强制定义或者记录这些依赖是很困难的。
Mixins打破了一个通用的、通常是安全的假设：你可以通过在组件源码文件中搜索的方式来重命名一个方法或者一个状态的key。你写了一个具有状态的组件，然后你的组员加入了一个mixin来读取它的状态。过了一两个月，你想把这个状态挪到父组件上，来实现跟相邻组件共享。你会记得同时更新这个mixin的代码，把它改为读取prop吗？再如果，现在还有其它组件也使用了这个mixin？

> These implicit dependencies make it hard for new team members to contribute to a codebase. A component’s render() method might reference some method that isn’t defined on the class. Is it safe to remove? Perhaps it’s defined in one of the mixins. But which one of them? You need to scroll up to the mixin list, open each of those files, and look for this method. Worse, mixins can specify their own mixins, so the search can be deep.
> Often, mixins come to depend on other mixins, and removing one of them breaks the other. In these situations it is very tricky to tell how the data flows in and out of mixins, and what their dependency graph looks like. Unlike components, mixins don’t form a hierarchy: they are flattened and operate in the same namespace.

这些隐形的依赖使得新成员在现有代码基础上继续开发变得困难。一个组件的`render()`方法也许引用了一些不在本类中定义的方法，删除它们是否安全？也许它们定义在mixins中，但是在哪个里面呢？你需要滚动到mixin列表，打开每个mixin的源码，来找这些方法。更坏的是，mixins可以定义它们自己的mixins，所以这次查找是一次深度查找。
经常地，mixins还依赖其它的mixins，如果你删除其中之一，可能会波及到另外的。在这种情况下，说明数据如何在mixins流入流出就变得很棘手了，更别说画出它们之间的依赖关系图。不像组件，mixins不会构成继承链：它们是扁平化的，并在同一个命名空间中起作用。

Mixins造成命名冲突
----------------
> There is no guarantee that two particular mixins can be used together. For example, if FluxListenerMixin defines handleChange() and WindowSizeMixin defines handleChange(), you can’t use them together. You also can’t define a method with this name on your own component.
> It’s not a big deal if you control the mixin code. When you have a conflict, you can rename that method on one of the mixins. However it’s tricky because some components or other mixins may already be calling this method directly, and you need to find and fix those calls as well.

从没有保证说任意两个mixins可以在一起使用。比如，如果`FluxListenerMixin`定义了`handleChange()`，`WindowSizeMixin`也定义了`handleChange()`，你就不能把它们拿在一块用。你也不能在你的组件中用这个名字来命名方法。
如果你能控制mixin的代码，那问题是不大的。当你遇到了命名冲突，你可以在其中的mixin中修改那个方法的名字。但是，如果有另外的mixins或是组件已经直接调用了这个方法，这就变得很棘手了，你需要同时找到和修复这些调用。

> If you have a name conflict with a mixin from a third party package, you can’t just rename a method on it. Instead, you have to use awkward method names on your component to avoid clashes.
> The situation is no better for mixin authors. Even adding a new method to a mixin is always a potentially breaking change because a method with the same name might already exist on some of the components using it, either directly or through another mixin. Once written, mixins are hard to remove or change. Bad ideas don’t get refactored away because refactoring is too risky.

如果你在使用一个第三方包的mixin时遇到了命名冲突，你就不能改它的方法名了。取而代之，你需要在你的组件中使用很蹩脚的方法名来避免冲突。
这样的情况对于mixin作者来说并没有好多少。加入一个新方法到mixin中总是一个潜在的风险，因为在已经使用了这个mixin的组件中，可能早就存在同名的方法了，无伦是直接调用还是通过其它mixin来调用。一旦mixins写好，就很困难去修改或者移除其中的东西。一些欠佳的实现方式得不到重构，因为重构的风险太大。

Mixins造成滚雪球式的复杂性
-----------------------
> Even when mixins start out simple, they tend to become complex over time. The example below is based on a real scenario I’ve seen play out in a codebase.
> A component needs some state to track mouse hover. To keep this logic reusable, you might extract handleMouseEnter(), handleMouseLeave() and isHovering() into a HoverMixin. Next, somebody needs to implement a tooltip. They don’t want to duplicate the logic in HoverMixin so they create a TooltipMixin that uses HoverMixin. TooltipMixin reads isHovering() provided by HoverMixin in its componentDidUpdate() and either shows or hides the tooltip.

虽然mixins是从简单开始的，但它们会随着时间变得越来越复杂。下面的例子是基于一个真实的情况。
一个组件需要一些状态来跟踪鼠标的悬浮（hover）。为了使这个逻辑可复用，你抽取了`handleMouseEnter()`、`handleMouseLeave()`、`isHovering()`方法到一个`HoverMixin`里。接下来，有人需要实现一个悬浮提示框（tooltip）。他们不想拷贝`HoverMixin`里的逻辑代码，因此创建了一个`TooltipMixin`，这个`TooltipMixin`引用了`HoverMixin`，`TooltipMixin`在它的`componentDidUpdate()`中读取由`HoverMixin`提供的`isHovering()`来显示或者隐藏提示框。

> A few months later, somebody wants to make the tooltip direction configurable. In an effort to avoid code duplication, they add support for a new optional method called getTooltipOptions() to TooltipMixin. By this time, components that show popovers also use HoverMixin. However popovers need a different hover delay. To solve this, somebody adds support for an optional getHoverOptions() method and implements it in TooltipMixin. Those mixins are now tightly coupled.
> This is fine while there are no new requirements. However this solution doesn’t scale well. What if you want to support displaying multiple tooltips in a single component? You can’t define the same mixin twice in a component. What if the tooltips need to be displayed automatically in a guided tour instead of on hover? Good luck decoupling TooltipMixin from HoverMixin. What if you need to support the case where the hover area and the tooltip anchor are located in different components? You can’t easily hoist the state used by mixin up into the parent component. Unlike components, mixins don’t lend themselves naturally to such changes.

几个月后，有人想让这个提示框的弹出方向变得可配置。为了避免代码重复，他们添加了一个新的配置方法`getTooltipOptions()`到`TooltipMixin`。在这时，需要弹出浮层的组件也使用了`HoverMixin`。但是浮层需要不同的鼠标悬浮延时。为了解决这个问题，有人添加并实现了一个配置方法`getHoverOptions()`到`TooltipMixin`中。这两个mixins现在紧紧耦合在一起了。
如果没有新的需求，这样是没有问题的。但是这个方法的可扩展性并不强。如果你想在同一个组件里面支持显示多个提示框呢？你不能在一个组件里面定义两次同一个mixin。如果提示框需要在用户引导里自动弹出，而不是在鼠标悬浮时弹出呢？你想解耦`TooltipMixin`和`HoverMixin`？祝你好运。如果你想让鼠标悬浮点和提示框锚点在不同的组件中呢？你不能轻易地将mixin使用的状态抬升到父组件中。不像组件，mixins在遇到这些改变时并不能很自然地交付。

> Every new requirement makes the mixins harder to understand. Components using the same mixin become increasingly coupled with time. Any new capability gets added to all of the components using that mixin. There is no way to split a “simpler” part of the mixin without either duplicating the code or introducing more dependencies and indirection between mixins. Gradually, the encapsulation boundaries erode, and since it’s hard to change or remove the existing mixins, they keep getting more abstract until nobody understands how they work.
> These are the same problems we faced building apps before React. We found that they are solved by declarative rendering, top-down data flow, and encapsulated components. At Facebook, we have been migrating our code to use alternative patterns to mixins, and we are generally happy with the results. You can read about those patterns below.

每个新需求让mixins变得越来越难以理解。随着时间，使用同一个mixin的组件之间的耦合度变得越来越高。任何新的功能都会同时被附加到所有使用了这个mixin的组件。没有方法去分离这个mixin的“更简单”的部分，除非去拷贝其中的代码，或者在mixins之间引入更多的依赖和奇技淫巧。逐渐地，原来的封装会瓦解，并且因为更改或者移除已经存在的mixins是困难的，它们会变得更抽象，直到没人理解它们是怎么工作的。
这些问题跟我们在React出来之前构建应用程序时遇到的问题是一样的。我们认为这些问题可以通过声明性的渲染、自上而下的数据流和组件封装来解决。在Facebook，我们已经将代码的实现方式从mixins迁移到了取而代之的模式，并且我们对结果很乐观。你可以继续阅读来了解我们的新模式。
