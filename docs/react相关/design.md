## 理念

> `React`是用`JavaScript`构建**快速响应**的大型 Web 应用程序的首选方式

## 设计

### React15 架构

#### 架构设计

React15 架构分两层

1. Reconciler（协调器）—— 负责找出变化的组件
2. Renderer（渲染器）—— 负责将变化的组件渲染到页面上

#### 架构缺点

仅能进行同步更新，同步更新的方式是通过递归，即开始就无法停止。  
当层级较深，执行时间比较长的话，就与**快速响应**这个理念无法吻合了， 在大项目中是比较常见的。  
因为浏览器刷新的频率是 60hz, 页面更新一次的时间为大概 16ms，如果执行的 task 超过 16ms, 画面不流畅的话就会让用户感觉卡顿。

### React16 架构

#### 架构设计

1. Scheduler（调度器）—— 调度任务的优先级，高优任务优先进入 Reconciler
2. Reconciler（协调器）—— 负责找出变化的组件
3. Renderer（渲染器）—— 负责将变化的组件渲染到页面上

#### Scheduler（调度器)

**什么是`Scheduler`?**  
Scheduler 是和 react 一个层级的 monorepo 库，独立于 react.  
它实现了类似`requestIdleCallback`函数的功能，在浏览器空闲时去提供多种优先级的任务调度设置  
通过`Scheduler`·实现异步可中断的更新，更加贴合**快速响应**的理念

#### Fiber

##### 什么是`Fiber`

`Fiber`意为纤程  
`Fiber 架构`为 React 内部实现的一套**状态更新机制**。支持任务不同优先级，可中断与恢复，并且恢复后可以复用之前的中间状态  
`Fiber节点`是在 React 中最小粒度的执行单元，可以将 fiber 理解为是 React 的虚拟 DOM.

##### `Fiber节点`结构

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // 作为静态数据结构的属性
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null;

  // 用于连接其他Fiber节点形成Fiber树
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  // 作为动态的工作单元的属性
  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;

  this.effectTag = NoEffect;
  this.nextEffect = null;

  this.firstEffect = null;
  this.lastEffect = null;

  // 调度优先级相关
  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  // 指向该fiber在另一次更新时对应的fiber
  this.alternate = null;
}
```

##### Fiber 工作原理

**双缓存 Fiber 树：**  
React 内部维护的两个 Fiber 树的引用，一个是`current Fiber树`，为当前正在渲染的节点树。  
另外一个是`workInProgress Fiber树`。  
为什么需要双缓存树呢？  
其实对于更新来说单缓存树就够了，双缓存树是服务于`concurrent mode`的.  
正是因为双缓存树才能保证了高优先级可中断更新的机制

当`workInProgress Fiber树`构建完成交给 Renderer 渲染在页面上后，应用根节点的 current 指针指向`workInProgress Fiber树`，此时`workInProgress Fiber树`就变为`current Fiber树`。

每次状态更新都会产生新的`workInProgress Fiber树`，通过`current`与`workInProgress`的替换，完成 DOM 更新

```js
currentFiber.alternate === workInProgressFiber;
workInProgressFiber.alternate === currentFiber;
```

在构建 `workInProgress` 的过程中，如果有更高优先级的更新产生， React 会停止 `workInProgress fiber树` 的构建，然后开始处理更高优先级的更新，重新构建`workInProgress fiber树`。等更高优先级的更新处理完毕之后，才会处理原来被中断的更新。

#### Hooks

**什么是`Hooks`?**  
`Hooks`翻译过来就是钩子，对于计算器而言就是响应和处理事件。  
在 React 中的体现为不同的函数， 这些函数返回或执行的内容与更新机制存在关联  
比如`useState`返回 state 和更新视图的 setState 函数，以及 useEffect 去关联到在更新时需要执行的副作用函数。  
React 团队核心成员`Sebastian Markbåge`也就是发明者说：

> 我们在 React 中做的就是践行代数效应（Algebraic Effects）。

**什么是`代数效应`?**

> 代数效应是函数式编程中的一个概念，用于将副作用从函数调用中分离。  
> 具体体现为: 函数式编程中的单子`Monad`(https://zhuanlan.zhihu.com/p/306339035)  
> 而 Hooks 中的 useEffect 正是很好的体现了这个思想

### diff 算法

diff 算法是用于对比新旧两个虚拟 DOM 树差异的算法，如果不满足 diff 的要去，则旧 DOM 将无法被复用并且返回新的 Fiber 节点

在虚拟 DOM 需要比较时，React 会对新旧两树进行深度优先遍历.  
如果是暴力遍历的话，时间复杂度会很高，
那么就需要进行一些条件设定，基于可服用的前提去做一个判断。  
那么 diff 算法条件设定的策略有这么三个：

1. React 只会比较同层级的节点，不会跨层级比较。这个策略可以避免不必要的比较和更新。
2. React 会比较节点的 key 值，如果 key 相同，则认为是同一个节点，不需要更新。这个策略可以提高比较的效率。
3. React 会比较组件的类型(Tag)，如果类型相同，则认为是同一个组件，不需要更新。这个策略可以避免不必要的比较和更新。
4. React 会比较列表中的节点，找出新增、删除和移动的节点。这个策略可以避免不必要的 DOM 操作，提高性能。
