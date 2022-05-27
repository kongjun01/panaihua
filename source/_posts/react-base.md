---
layout: post
title: "React 入门 （基础概念）"
date: 2018-09-12
excerpt: "开始的时候我是很抵触的，因为觉得后端写前端最后肯定会不三不四，之前也写过前后端全包的项目，但是对后面的工作都没什么帮助，所以觉得对个人发展也不是很好，而且全栈工程师在杭州需求也不多。没办法，公司就是不给配前端，只能一个人整，想想不能影响项目进度啊，最后倒霉还是自己，所以硬着头皮跳下去了..."
categories: 
- 前端
tags: 
- React
comments: true
---

### 前言

>准备写这篇文章时，才发现一年都没有总结了。之前还想一个月总结一次，对比现在简直太夸张了。其实今年的收获还挺多的，比如将学习的python、react应用到工作中，个人爬虫网站上线。

开始的时候我是很抵触的，因为觉得后端写前端最后肯定会不三不四，之前也写过前后端全包的项目，但是对后面的工作都没什么帮助，所以觉得对个人发展也不是很好，而且全栈工程师在杭州需求也不多。没办法，公司就是不给配前端，只能一个人整，想想不能影响项目进度啊，最后倒霉还是自己，所以硬着头皮跳下去了。

项目大概做了4个月，对前端其实还是皮毛，但是完成项目需求还是搓搓有余。能撑到现在是因为有个念头：往全栈工程师发展也不一定是件坏事，说不定就有奇迹了呢。

### 简介
React.js 是一个帮助你构建页面 UI 的库。如果你熟悉 MVC 概念的话，那么 React 的组件就相当于 MVC 里面的 View。说白点就是帮助我们将界面分成各个独立的小块，每一个块就是组件，这些组件之间可以组合、嵌套，就成了我们的页面。

一个组件的显示形态和行为有可能是由某些数据决定的。而数据是可能发生改变的，这时候组件的显示形态就会发生相应的改变。而 React.js 也提供了一种非常高效的方式帮助我们做到了数据和组件显示形态之间的同步。

React.js 不是一个框架，它只是一个库。它只提供 UI （view）层面的解决方案。在实际的项目当中，它并不能解决我们所有的问题，需要结合其它的库，例如 Redux、React-router 等来协助提供完整的解决方法。

### JSX
React的核心机制就是实现了一个虚拟DOM，利用虚拟DOM来减少对实际DOM的操作从而提升性能，组件DOM结构就是映射到这个虚拟的DOM上，React在这个虚拟DOM上实现了一个diff算法，当要更新组件的时候，会通过diff寻找要变更的DOM节点，再把这个修改更新到浏览器实际的DOM节点上，所以实际上不是真的渲染整个DOM树，这个虚拟的DOM是一个纯粹的JS数据结构，所以性能比原生DOM会提高很多；  

虚拟DOM(virtual-dom)实际上是对实际DOM的一个抽象，是一个js对象。react所有的表层操作实际上是在操作虚拟DOM。经过diff算法计算出虚拟DOM的差异，然后将这些差异进行实际的DOM操作更新页面


从 JSX 到页面经历的过程：

![](/images/jsx-banyi.png)

>* JSX 是 JavaScript 语言的一种语法扩展，长得像 HTML，但并不是 HTML。
* React.js 可以用 JSX 来描述你的组件长什么样的。
* JSX 在编译的时候会变成相应的 JavaScript 对象描述。
* react-dom 负责把这个用来描述 UI 信息的 JavaScript 对象变成 DOM 元素，并且渲染到页面上。


### 组件化

虚拟DOM(virtual-dom)不仅带来了简单的UI开发逻辑，同时也带来了组件化开发的思想，所谓组件，即封装起来的具有独立功能的UI部件。React推荐以组件的方式去重新思考UI构成，将UI上每一个功能相对独立的模块定义成组件，然后将小的组件通过组合或者嵌套的方式构成大的组件，最终完成整体UI的构建。

可以按照下面的规则进行划分组件：

* 容器组件：容器型组件是一个页面容器，用来放置当前页面的所有展示型组件和业务组件组合成一个页面，通过数据的驱动进行控制展示组件和业务组件。

* 展示组件：展示型组件是具体到某一个小的组件模块，比如一个按钮，一个卡片，一个进度条等，我们在用react做组件化开发的时候，先定义好一个个小的展示型组件，然后把这些组件都导入容器型组件，最终组合成一个完整的页面。

* 业务组件：页面中某个业务模块的拆分，涉及到数据交互，有自己独立的业务逻辑

React 认为一个组件应该具有如下特征:
>* 可组合（Composeable）：一个组件易于和其它组件一起使用，或者嵌套在另一个组件内部。如果一个组件内部创建了另一个组件，那么说父组件拥有（own）它创建的子组件，通过这个特性，一个复杂的UI可以拆分成多个简单的UI组件。 
* 可重用（Reusable）：每个组件都是具有独立功能的，它可以被使用在多个UI场景。
* 可维护（Maintainable）：每个小的组件仅仅包含自身的逻辑，更容易被理解和维护。

### 组件种类

**function-based(函数组件)**：函数组件也称为无状态组件，使用纯函数创建，创建后不会产生组件实例，也就是说无法使用ref获取，主要用于展示组件的开发，性能高，没有生命周期，没有state，但是可以接收props，相当于一个只有render生命周期的组件

```javascript
function Component (props) {
    return <div>{props.children}</div>
}
//使用
<Component>组件</Component>
```
**class-based(类组件)**：使用es6 class的方式创建，通过继承React.component实现，可以有自己独立的生命周期，state状态，必须有render生命周期，state定义在constructor构造函数中,所用的容器组件都是通过class创建

```javascript
class Component extends React.component {
   constructor (props) {
     super(props)
     this.state = {
         age: 100
     }
   }
   render () {
       const {age} = this.state
       return <div>{age}</div>
   }
}
//使用
<Component />
```

**createClass组件**：不常用

>在react中最小的单位是元素，元素分为dom元素，组件元素，区分方法是dom元素小写，组件元素首字母大写

### 组件状态

每个组件都有一个状态，下面几点仅适用于class-based组件。  
一个组件的显示形态是可以由它数据状态和配置参数决定的。在react中组件的状态使用state定义，使用setState修改，使用this.state读取。

***setState***

`setState` 方法由父类 Component 所提供。当我们调用这个函数的时候，React.js 会更新组件的状态 state ，并且重新调用 render 方法，然后再把 render 方法所渲染的最新的内容显示到页面上。

`setState`是“异步”的，调用`setState`只会提交一次state修改到队列中，不会直接修改`this.state`。等到满足一定条件时，react会合并队列中的所有修改，触发一次update流程，更新`this.state`。因此setState机制减少了update流程的触发次数，从而提高了性能。

`setState` 的更新是一个浅合并（Shallow Merge）的过程

*特殊情况：*

>在实际开发中，setState的表现有时会不同于理想情况。主要是以下两种:  
1. 在mount流程中调用setState。  
2. 在setTimeout/Promise回调中调用setState。  
在第一种情况下，不会进入update流程，队列在mount时合并修改并render。  
在第二种情况下，setState将不会进行队列的批更新，而是直接触发一次update流程。这是由于setState的两种更新机制导致的，只有在批量更新模式中，才会是“异步”的。


***State 与 Props 区别***

`state` 的主要作用是用于组件保存、控制、修改自己的可变状态。state 在组件内部初始化，可以被组件自身修改，而外部不能访问也不能修改。你可以认为 state 是一个局部的、只能被组件自身控制的数据源。state 中状态可以通过 this.setState 方法进行更新，setState 会导致组件的重新渲染。

`props` 的主要作用是让使用该组件的父组件可以传入参数来配置该组件。它是外部传进来的配置参数，组件内部无法控制也无法修改。除非外部组件主动传入新的 props，否则组件的 props 永远保持不变。

>state 是让组件控制自己的状态，props 是让外部对组件自己进行配置


### 组件的生命周期

对于 React 组件来说，生命周期主要包含三个阶段：创建(挂载)过程、销毁(卸载)过程和存在期。

React.js 将组件渲染，并且构造 DOM 元素然后塞入页面的过程称为组件的挂载。过程如下：

```javascript
-> constructor()
-> componentWillMount()
-> render()
// 然后构造 DOM 元素插入页面
-> componentDidMount()
```
`componentWillMount` 和 `componentDidMount` 都是可以像 render 方法一样自定义在组件的内部。挂载的时候，React.js 会在组件的 render 之前调用 `componentWillMount`，在 DOM 元素塞入页面以后调用 `componentDidMount`。

一个组件可以插入页面，当然也可以从页面中删除

```javascript
-> constructor()
-> componentWillMount()
-> render()
// 然后构造 DOM 元素插入页面
-> componentDidMount()
// ...
// 即将从页面中删除
-> componentWillUnmount()
// 从页面中删除
```

setState更新机制大致流程：

![](/images/state-update.png)


>* 当一个组件的state或者其父组件传递的props发生改变的时候组件就会重新渲染.
* 如果是后者即props发生改变时 React 会调用另外一个周期函数`componentWillReceiveProps`
* 如果两者都发生改变，React会做一个重要决策，该组件是否需要在浏览器里被更新？这也是为什么会调用另外一个周期函数`shouldComponentUpdate`的原因. 这个方法实际上也是在问一个问题，所以如果你想自定义或者优化你的渲染过程，你就需要通过返回一个true或者false来回答这个问题。
* 如果没有手动指定`shouldComponentUpdate`, React 会默认作出聪明的决策，多数情况下也是足够良好的.
* 首先, 这时候React会调用`componentWillUpdate`方法. 然后计算新的渲染产出把它和上一次的渲染产出进行比较.
* 如果没什么改变，那么就什么也不做.
* 如果有改变则把差异反应到浏览器上.
* 无论什么情况，尽管更新会发生在任何地方（甚至计算出来的产出是相同的）,React 最终都会调用另一个周期方法`componentDidUpdate`.

可以将`React`视为我们聘请的与浏览器通信的代理。我们不是去手动的用DOM API来操作DOM，而是更组件状态的属性值，然后让`React`代表我们去和浏览器沟通. 我相信着就是react为什么这么受欢迎的原因. 我们讨厌和浏览器先生（还说着各种带有口音的DOM方言）打交道，React志愿为我们做这些事情，还是免费的～

### 小结

* React速度很快

与其它框架相比，React采取了一种特立独行的操作DOM的方式。
它并不直接对DOM进行操作。  
它引入了一个叫做虚拟DOM的概念，安插在JavaScript逻辑和实际的DOM之间。  
这一概念提高了Web性能。在UI渲染过程中，React通过在虚拟DOM中的微操作来实对现实际DOM的局部更新。

* 跨浏览器兼容

虚拟DOM帮助我们解决了跨浏览器问题，它为我们提供了标准化的API，甚至在IE8中都是没问题的。

* 模块化

为你程序编写独立的模块化UI组件，这样当某个或某些组件出现问题是，可以方便地进行隔离。  
每个组件都可以进行独立的开发和测试，并且它们可以引入其它组件。这等同于提高了代码的可维护性。

* React与其它框架/库兼容性好

比如使用RequireJS来加载和打包，而Browserify和Webpack适用于构建大型应用。它们使得那些艰难的任务不再让人望而生畏。

**缺点：**

React本身只是一个V而已，所以如果是大型项目想要一套完整的框架的话，还需要引入Flux和routing相关的东西