## **React**

React 框架是 Meta 公司前身 Facebook 于 2013 年开源的，声明式、组件化的前端开发框架

### **React的核心概念**

- 声明式
- 组件化
- 单向数据流
- Hooks

### **React生态**

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

在 Web 领域，**类 HTML 语法**天生就更受欢迎。JSX 提供的类 HTML/XML 的语法会让声明代码更加直观，在 IDE 的支持下，语法高亮更醒目，比起纯 JS 也更容易维护。相比 JSX 带来的开发效率的提升，编译 JSX 的成本基本可以忽略不计

相比Struts2 用 XML 定义了一套名为标签库的 DS（Domain-Specific Language，领域特定语言），JSX 则直接利用了 JS 语句。很明显，JS 表达式能做的，JSX 都能做，不需要开发者再去学习一套新的 DSL

因为 JSX 作为语法糖足够“甜”，我们才能得到这样的结论：JSX 是前端视图领域“最 JS”的声明式语法，它为 React 的推广和流行起了至关重要的作用

#### **声明式 vs 命令式**

![declarative_vs_Imperative.png](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E5%89%8D%E7%AB%AF/resources/declarative_vs_Imperative.png)

React是声明式的前端技术，这一点首先就体现在创建组件的视图上，无论是使用 JSX 语法还是直接利用React.createElement() 函数，都是在描述开发者期待的视图状态。开发者只需关心渲染结果，而 React 框架内部会实现具体的渲染过程，最终调用浏览器 DOM API

目前的三大主流前端框架，React、Vue、Angular 都是声明式的

#### **JSX基本语法**

JSX = JavaScript XML，即**在 JS 语言里加入类 XML 的语法扩展**

- **X部分**：包括标签的命名规则，支持的元素类型、子元素类型；
- **JS部分**：即 JSX 中都有哪里可以加入 JS 表达式、规则是什么；

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

#### **JSX命名规则**

1. 定义 React 组件时，组件本身采用的变量名或者函数名，需要以大写字母开头

```
function MyApp() {
//_______^
  return (<div></div>);
}
const KanbanCard = () => (
//____^
  <div></div>
);
```

2. 在 JSX 中编写标签时，HTML 元素名称均为小写字母，自定义组件首字母务必大写

```
   <h1>我的看板</h1>
<!-- ^________全小写 -->
    <img src={logo} className="App-logo" alt="logo" />
<!-- ^^^______全小写 -->
    <button onClick={handleAdd} disabled={showAdd}>添加新卡片</button>
<!-- ^^^^^^___全小写 -->


    <KanbanCard />
<!-- ^_____首字母大写 -->
```

3. props 属性名称，在 React 中使用驼峰命名（camelCase），且区分大小写，比如在 ```<FileCard filename="文件名" fileName="另一个文件名" />```中，你可以同时传两个字母相同但大小写不同的属性 ，这与传统的 HTML 属性不同

#### **JSX元素类型**

JSX 产生的每个节点都称作 React 元素，它是 React 应用的最小单元。React 元素有三种基本类型：

1. React 封装的 DOM 元素，如 ```<div></div>、 <img />```，这部分元素会最终被渲染为真实的 DOM；
2. React 组件渲染的元素，如```<KanbanCard />``` ，这部分元素会调用对应组件的渲染方法；
3. React Fragment 元素，```<React.Fragment></React.Fragment>```或者简写成 ```<>```，这一元素没有业务意义，也不会产生额外的 DOM，主要用来将多个子元素分组;

不同类型元素的 props 有所区别:

1. React 封装的 DOM 元素将浏览器 DOM 整体做了一次面向 React 的标准化;
2. React 组件渲染的元素，JSX 中的 props 应该与自定义组件定义中的 props 对应起来；如果没有特别处理，没有对应的 props 会被忽略掉;
3. Fragment 元素，没有 props;

#### **JSX 子元素类型**

JSX 元素可以指定子元素，子元素不一定是子组件，子组件一定是子元素，子元素的类型包括：

1. 字符串，最终会被渲染成 HTML 标签里的字符串；
2. 另一段 JSX，会嵌套渲染；
3. JS 表达式，会在渲染过程中执行，并让返回值参与到渲染过程中；
4. 布尔值、null 值、undefined 值，不会被渲染出来；
5. 以上各种类型组成的数组；

#### **JSX 中的 JS 表达式**

在 JSX 中可以插入 JS 表达式，特征是用大括号 { } 包起来，主要有两个地方：

1. 作为 props 值，如 ```<button disabled={showAdd}>```添加新卡片```</button>```；
2. 作为 JSX 元素的子元素，如```<div className="card-title">{title}</div>```;

这些表达式可以简单到原始数据类型 {true} 、{123} ，也可以复杂到一大串 Lambda 组成的函数表达式 ```{ todoList.filter(card => card.title.startsWith('TODO:')).map(props => <KanbanCard {...props} />) }``` ，只要确保最终的返回值符合 props 值或者 JSX 子元素的要求，就是有效的表达式

JSX 是声明式的，所以它的内部不应该出现命令式的语句，如 ```if ... else ...```

有个 props 表达式的特殊用法：属性展开，``` <KanbanCard {...props} /> ```利用 JS ... 语法把 props 这个对象中的所有属性都传给 KanbanCard 组件

#### **JSX 与 React 组件的关系**

- JSX 就是 React？
  - 不是。JSX 只是 React 其中一个 API createElement 函数的语法糖
- JSX 就是 React 组件？
  - 不是。JSX 是 React 组件渲染方法返回值的一部分，React 组件还有其他的功能
- JSX 就是另一种 HTML？
  - 不是。JSX 本质还是 JS，只是在最终渲染时才创建修改 DOM
- JSX 既能声明视图，又能混入 JS 表达式，那是不是可以把所有逻辑都写在 JSX 里？
  - 可以是可以，但毕竟不能在 JSX 里使用命令式语句，能做的事情很有限

运用好 JSX，可以很大程度提高React 开发效率和效果

### **React组件化**

组件化开发已经成为前端开发的主流趋势，市面上大部分前端框架都包含组件概念，有些框架里叫 Component，有些叫 Widget。React 更是把组件作为前端应用的核心

#### **为什么要组件化**

在前端领域，**组件是对视图以及与视图相关的逻辑、数据、交互等的封装**。如果没有组件这层封装，这些代码将有可能四散在各个地方，低内聚，也不一定能低耦合，这种代码往往难写、难读、难维护、难扩展

#### **为什么要有组件层次结构？**

组件层次结构（Hierarchy）在面向对象编程里也有 这个概念，一般是指父类子类之间的继承关系

React 并没有用类继承的方式扩展现有组件（类组件继承 React.Component类，但类组件之间没有继承关系），所以在 React 中提到 Hierarchy，一般都是指组件与组件间的层次结构

组件层次结构可以帮助我们在设计开发组件过程中，将前端应用需要承担的业务和技术复杂度分摊到多个组件中去，并把这些组件拼装在一起

React 组件层次结构从一个根部组件开始，一层层加入子组件，最终形成一棵组件树

#### **组件如何拆分？**

组件拆分并无唯一标准。拆分时需要理解业务和交互，设计组件层次结构（Hierarchy），以关注点分离（Separation Of Concern）原则检验每次拆分。另外也要避免一个误区：组件确实是代码复用的手段之一，但并不是每个组件都需要复用

就拆分方向而言，一般面对中小型应用，更倾向于从上到下拆分，先定义最大粒度的组件，然后逐渐缩小粒度；面对大型应用，则更倾向于从下往上拆分，先从较小粒度的组件开始。无论从哪个方向拆分组件，都尽量遵守以下基本原则：

1. 单一职责（Single Responsibility）原则；
2. 关注点分离（Separation of Concern）原则；
3. 一次且仅一次（DRY, Don’t Repeat Yourself）原则；
4. 简约（KISS，Keep It Simple & Stupid）原则;

#### **React组件的渲染机制**

#### **虚拟DOM**

虚拟 DOM（Virtual DOM）是相对于 HTML DOM（Document Object Model，文档对象模型）更轻量的 JS 模型。在 React、Vue.js、Elm 这样的声明式前端框架中，都包含了虚拟 DOM

React 提供了包括 JSX 语法在内的声明组件 API，在运行时，开发者声明的组件会渲染成虚拟 DOM，虚拟 DOM 再由 React 框架渲染成真实的 DOM；虚拟 DOM 的变动，最终会自动体现在真实 DOM 上；真实 DOM 上的交互，也会由 React 框架抽象成虚拟 DOM 上的副作用（Side-effect），与开发者编写的交互逻辑关联起来

理想状态下，开发者在开发 React 应用时，可以完全不去接触真实 DOM（但现实世界中这种情况很少见），一定程度上隐藏了 Web 原生技术的细节，有助于提高开发效率

如下图所示，左半边展示了 React 面向开发者的 API，右半边则是 React 内部实现对 DOM API 的封装，渲染面向用户的页面:

![virtual-dom](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E5%89%8D%E7%AB%AF/resources/virtual_dom.png)

虚拟 DOM 最重要的作用，是作为 React 面向开发者的 API 与 React 内部实现对接的桥梁。**React API 整体都是声明式的，而 DOM API 是命令式的**。我们知道，No Magic（没有魔法），开发者用 API 声明的 React 组件，最终成为页面上的动态 DOM 元素，必然在 React 框架内部有着一系列命令式的实现，负责最终调用浏览器 DOM API

如果没有虚拟 DOM 这个中间模型，那么 React API 就需要直接对接 DOM API，耦合程度提高，React 概念和 API 设计也会受制于浏览器，React Native 这样对接多端的愿景也无从实现了

### **真实DOM有何问题**

React 的设计哲学 UI=f(state) ，理论上来说，对于给定的 f() 和状态数据，一定可以重现一模一样的 UI；这也意味着，只要状态数据有变化，f()就需要重新执行，整个 UI 需要重新渲染

操作真实 DOM 是比较耗费资源的，无脑地大量调用 DOM API 绘制页面，页面很容易就卡了，对于浏览器网页中的应用，假如达到60FPS，意味着 1000ms ÷ 60 ≈ 16ms 之内至少需要执行完一次 f() ，否则会掉帧，显示和交互都会卡顿

这时就需要 React 提供一系列算法和过程，过滤掉没有必要的 DOM API 调用，最终把f() 的成本降下来。虚拟 DOM 就是这些算法过程的中间模型，它远比 DOM API 轻量，跟最终的 DOM API 分摊成本后，可以保证 React 组件的渲染效率

协调

React 组件会渲染出一棵元素树。因为开发者使用的是 React 的声明式 API，在此基础上，每次有 props、state 等数据变动时，组件会渲染出新的元素树，React 框架会与之前的树做 Diffing 对比，将元素的变动最终体现在浏览器页面的 DOM 中。这一过程就称为协调（Reconciliation）

### **Diffing 算法**

React 框架后续的版本中在不断优化 Diffing 算法。近四年算法细节变了不少，但基本逻辑还是能归纳出以下几点：

1. 从根元素开始，React 将递归对比两棵树的根元素和子元素；
2. 对比不同类型的元素，如对比 HTML 元素和 React 组件元素，React 会直接清理旧的元素和它的子树，然后建立新的树；
3. 对比同为 HTML 元素，但 Tag 不同的元素，如从```<a>``` 变成```<div>```，React 会直接清理旧的元素和子树，然后建立新的树；
4. 对比同为 React 组件元素，但组件类或组件函数不同的元素，如从 KanbanNewCard 变成 KanbanCard ，React 会卸载旧的元素和子树，然后挂载新的元素树；
5. 对比 Tag 相同的 HTML 元素，如```<input type="text" value="old" />```和 ```<input type="text" value="new" />```，React 将会保留该元素，并记录有改变的属性，在这个例子里就是 value 的值从 "old" 变成了 "new" ；
6. 对比组件类或组件函数相同的组件元素，如```<KanbanCard title="老卡片" />``` 和 ```<KanbanCard title="新卡片" />```React 会保留组件实例，更新 props，并触发组件的生命周期方法或者 Hooks;

### **什么情况下触发协调？**

只要props、state和context这三种数据之一发生了变化，React 就会对当前组件触发协调过程，最终按照 Diffing 结果更改页面，注意以下几个知识点：

- 一个组件的 props 应该由父组件传进来，props 数据的变动也应该由父组件负责
- state 则是活跃在组件内部
- 至于 context ，在组件外面的 Context.Provider 提供数据，组件内部则可以消费 context 数据
-props 和 state 都是不可变的（Immutable）

### **Fiber 协调引擎**

在 React 中最贴近虚拟 DOM 的，是在 Fiber 协调引擎中的核心模型 FiberNode

在 React 的早期版本，协调是一个同步过程，这意味着当虚拟 DOM 足够复杂，或者元素渲染时产生的各种计算足够重，协调过程本身就可能超过 16ms，严重的会导致页面卡顿

从 React v16 开始，协调从之前的同步改成了异步过程，这主要得益于新的 Fiber 协调引擎

FiberNode 依靠对元素到子元素的双向链表、子元素到子元素的单向链表实现了一棵树，这棵树可以随时暂停并恢复渲染，触发组件生命周期等副作用（Side-effect），并将中间结果分散保存在每一个节点上，不会 block 浏览器中的其他工作

![fiber-node](https://github.com/xiaoyuge/Tech-Notes/blob/main/%E5%89%8D%E7%AB%AF/resources/fiber_node.jpg)

总结一下
React 组件的渲染机制，就是**虚拟 DOM 是真实 DOM 的抽象**，React 开发者通过 JSX 等 API 声明组件树，React 内部则会生成对应的虚拟 DOM；组件 props、state、context 有变化时会触发协调过程，通过 Diffing 算法比对新旧虚拟 DOM，**最终只针对差异部分调用 DOM API 改变页面**

### **Hooks**

Hooks 是 React 实现组件逻辑的重要方式，可以用来操作 state，定义副作用，更支持开发者自定义 Hooks。Hooks 借鉴自函数式编程，但同时在使用上也有一些限制

我们来借助函数式编程中**纯函数**和**副作用**这两个概念，来理解什么是 Hooks

当一个函数满足如下条件时，就可以被认为是纯函数：

1. 函数无论被调用多少次，**只要参数相同，返回值就一定相同**，这一过程不受外部状态或者 IO 操作的影响；
2. 函数被调用时**不会产生副作用（Side Effect）**，即不会修改传入的引用参数，不会修改外部状态，不会触发 IO 操作，也不会调用其他会产生副作用的函数；

比如下面的js代码就是一个最简单的纯函数：

```
const func = (a, b) => {
  return a + b;
};
```

相似的，下面的 React 函数组件，对于给定的 props a和b，每次渲染时都会返回相同的无序列表元素

```

const Component = ({ a, b }) => {
  return (
    <ul>
      <li>{a}</li>
      <li>{b}</li>
    </ul>
  );
};
```

虽然 React 官方并没有类似的提法，但我们姑且可以把这样用纯函数的方式编写的 React 组件称作“纯函数组件”。编写纯函数组件，可以最直观地展示输入的 props 与输出的渲染元素之间的关系，非常利于开发者把握组件的层次结构和样式

但是，这样的纯函数组件除了 props、JSX 外，几乎不能使用 React 组件的所有其他特性——对于纯函数组件来说，这些其他特性全部都是外部状态或副作用

反过来说，若想让函数组件使用这些其他特性，只要让它以某种方式，显式地访问函数的外部状态（应限制在 React 框架的范围以内，所以对 React 而言是内部状态），或者执行副作用就好了

**Hooks就是这样一套为函数组件设计的，用于访问 React 内部状态或执行副作用操作，以函数形式存在的 React API**

作为例子，我们在上面的“纯函数组件”代码中加入 Hooks。useState 这一 Hook 会读取或存储组件的 state，加入它，让函数组件具有了操作 state 的能力：

```react
const Component = ({ a, b }) => {
  const [m, setM] = useState(a); // 一个Hook
  const [n, setN] = useState(b); // 另一个Hook
  return (
    <ul>
      <li>{m}<button onClick={() => setM(m + 1)}>+</button></li>
      <li>{n}<button onClick={() => setN(n + 1)}>+</button></li>
    </ul>
  );
};
```

在上面的函数中调用 useState ，意味着函数将访问函数本身以外、React 以内的状态，这就让函数产生了副作用，导致函数不再是纯函数，也意味着函数组件不再是“纯函数组件”

但我们从来没有强求过组件函数必须是纯函数，加入 Hooks 的函数组件不再纯粹，但更强大，变得可以使用包含 state 在内的、React 的大部分特性

### **Reac Hook有哪些**

React v18.2.0 提供的基础 Hooks 包括三个：

1. useState
2. useEffect
3. useContext

其他 Hooks，有些是上面基础 Hooks 的变体，有些虽然用途不同，但与基础 Hooks 共享底层实现。包括十个：

1. useReducer
2. useMemo
3. useCallback
4. useRef
5. useImperativeHandle
6. useLayoutEffect
7. useDebugValue
8. useDeferredValue
9. useTransition
10. useId

此外还有为第三方库作者提供的 useSyncExternalStore 和 useInsertionEffect

## **To Be Continued....**
