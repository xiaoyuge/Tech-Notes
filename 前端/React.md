## **React**

React 框架是 Meta 公司前身 Facebook 于 2013 年开源的，声明式、组件化的前端开发框架

### **React的核心概念**：
- 声明式
- 组件化
- 单向数据流
- Hooks

### **React生态**：
- **类MVC框架**：早期的 React 侧重于视图，并没有内置 dispatch 和 reducer 相关的接口，但开源社区为 React 设计的应用状态管理框架如雨后春笋，百家争鸣，如 FB 的 Flux、开源的 Redux、MobX 等等；
- **服务器通信**：浏览器标准的fetch API，以及更强的 React Query 框架；
- **表单处理**：Formik 框架、React Hook Form 框架；
- **前端路由**：事实标准的 react-router 框架；
- **组件样式**：诸多 CSS-in-JS 框架，如 emotion ；
- **打包编译工具**：Webpack、Babel，以及包含了前者的 Create-React-App 脚手架；
- **自动化测试框架**：Jest、React Testing Library

### **声明式语法：JSX**
React 是一套声明式的、组件化的前端框架，声明（动词）组件是 React 前端开发工作最重要的组成部分。在声明组件的代码中使用了 JSX 语法，JSX 不是 HTML，也不是组件的全部

#### **JSX是语法糖**
**组件是 React 开发的基本单位**。在组件中，需要被渲染的内容是用 React.createElement(component, props, ...children) 声明的，而 **JSX 正是createElement函数的语法糖**。浏览器本身不支持 JSX，所以在应用发布上线前，JSX 源码需要工具编译成由若干createElement函数组成的 JS 代码，然后才能在浏览器中正常执行

为什么不直接手写js而使用JSX的原因？
直接手写js最显著的好处就是，这部分代码不需要针对 JSX 做编译，直接可以作用于浏览器。但是，当元素或者元素的嵌套层级比较多时，JS 代码的右括号会越来越多，阅读和修改会越来越费劲

在 Web 领域，类 HTML 语法天生就更受欢迎。JSX 提供的类 HTML/XML 的语法会让声明代码更加直观，在 IDE 的支持下，语法高亮更醒目，比起纯 JS 也更容易维护。相比 JSX 带来的开发效率的提升，编译 JSX 的成本基本可以忽略不计

因为 JSX 作为语法糖足够“甜”，我们才能得到这样的结论：JSX 是前端视图领域“最 JS”的声明式语法，它为 React 的推广和流行起了至关重要的作用

#### **声明式 vs 命令式**

![declarative_vs_Imperative.png](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E5%89%8D%E7%AB%AF/resources/declarative_vs_Imperative.png)

React是声明式的前端技术，这一点首先就体现在创建组件的视图上，无论是使用 JSX 语法还是直接利用React.createElement() 函数，都是在描述开发者期待的视图状态。开发者只需关心渲染结果，而 React 框架内部会实现具体的渲染过程，最终调用浏览器 DOM API

目前的三大主流前端框架，React、Vue、Angular 都是声明式的

#### ***JSX基本语法*
JSX = JavaScript XML，即在 JS 语言里加入类 XML 的语法扩展
- X部分：包括标签的命名规则，支持的元素类型、子元素类型；
- JS部分：即 JSX 中都有哪里可以加入 JS 表达式、规则是什么；

我们来看一下 JSX 各个组成部分与React.createElement() 函数各参数的对应关系，代码如下
```
React.createElement(type)
React.createElement(type, props)
React.createElement(type, props, ...children)
```
其中 type 参数是必须的，props 可选，当参数数量大于等于 3 时，可以有一个或多个 children

比如：
```
   <li className="kanban-card">
<!-- ^^ ^^^^^^^^^ ^^^^^^^^^^^^^
  type  props-key  props-value                                  -->
      <div className="card-title">{title}</div>   <!-- children -->
      <div className="card-status">{status}</div> <!-- ____|    -->
    </li>
```

把 children 中的一个成员单独来看，也是对应一条createElement() 语句:
```
   <div className="card-title">{title}</div>
<!-- ^^^ ^^^^^^^^^ ^^^^^^^^^^^^ ^^^^^^^
   type  props-key props-value   children -->
```

#### **JSX元素类型**
JSX 产生的每个节点都称作 React 元素，它是 React 应用的最小单元。React 元素有三种基本类型：
1. React 封装的 DOM 元素，如 ```<div></div>、 <img /> ```，这部分元素会最终被渲染为真实的 DOM；
2. React 组件渲染的元素，如```<KanbanCard />``` ，这部分元素会调用对应组件的渲染方法；
3. React Fragment 元素，```<React.Fragment></React.Fragment>```或者简写成 ```<>```，这一元素没有业务意义，也不会产生额外的 DOM，主要用来将多个子元素分组;

不同类型元素的 props 有所区别:
1. React 封装的 DOM 元素将浏览器 DOM 整体做了一次面向 React 的标准化;
2. React 组件渲染的元素，JSX 中的 props 应该与自定义组件定义中的 props 对应起来；如果没有特别处理，没有对应的 props 会被忽略掉;
3.  Fragment 元素，没有 props;

#### **JSX 子元素类型**
JSX 元素可以指定子元素，子元素不一定是子组件，子组件一定是子元素，子元素的类型包括：
1. 字符串，最终会被渲染成 HTML 标签里的字符串；
2. 另一段 JSX，会嵌套渲染；
3. JS 表达式，会在渲染过程中执行，并让返回值参与到渲染过程中；
4. 布尔值、null 值、undefined 值，不会被渲染出来；
5. 以上各种类型组成的数组；

#### JSX 中的 JS 表达式**
在 JSX 中可以插入 JS 表达式，特征是用大括号 { } 包起来，主要有两个地方：
1. 作为 props 值，如 ```<button disabled={showAdd}>```添加新卡片```</button>```；
2. 作为 JSX 元素的子元素，如```<div className="card-title">{title}</div>```;

这些表达式可以简单到原始数据类型 {true} 、{123} ，也可以复杂到一大串 Lambda 组成的函数表达式 ```{ todoList.filter(card => card.title.startsWith('TODO:')).map(props => <KanbanCard {...props} />) }``` ，只要确保最终的返回值符合 props 值或者 JSX 子元素的要求，就是有效的表达式

JSX 是声明式的，所以它的内部不应该出现命令式的语句，如 ```if ... else ...```

有个 props 表达式的特殊用法：属性展开，``` <KanbanCard {...props} /> ```利用 JS ... 语法把 props 这个对象中的所有属性都传给 KanbanCard 组件

#### **JSX 与 React 组件的关系**
- JSX 就是 React？
  * 不是。JSX 只是 React 其中一个 API createElement 函数的语法糖
- JSX 就是 React 组件？
  * 不是。JSX 是 React 组件渲染方法返回值的一部分，React 组件还有其他的功能
- JSX 就是另一种 HTML？
  * 不是。JSX 本质还是 JS，只是在最终渲染时才创建修改 DOM
- JSX 既能声明视图，又能混入 JS 表达式，那是不是可以把所有逻辑都写在 JSX 里？
  * 可以是可以，但毕竟不能在 JSX 里使用命令式语句，能做的事情很有限

运用好 JSX，可以很大程度提高React 开发效率和效果

## **to be continued...**




