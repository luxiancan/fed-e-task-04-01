# 卢显灿 ｜ Part 4 | 模块一

## 简答题

### 1、请简述 React 16 版本中初始渲染的流程

（1）jsx 转换成 react element 

- babel-react 先将 jsx 转换成 React.createElement()
- React.createElement 会 jsx 转换成 react element （react element 就是一个用来描述 react 元素的对象）

（2）render 阶段（协调层）

首先为每个 react 元素构建 fiber 对象，然后创建此 fiber 对象对应的 DOM 对象，为 fiber 对象添加 effectTag 属性（用来记录当前 Fiber 要执行的 DOM 操作），然后在render 结束后， fiber 会被保存到 fiberroot 中。

源码中详细步骤：
- 将子树渲染到容器中 (初始化 Fiber 数据结构: 创建 fiberRoot 及 rootFiber)
- 判断是否为服务器端渲染，如果不是服务器端渲染，则清空 container 容器中的节点
- 通过实例化 ReactDOMBlockingRoot 类创建 LegacyRoot ，创建 LegacyRoot 的 Fiber 数据结构
- 创建根节点 createContainer ，创建根节点对应的 fiber 对象 createFiberRoot
- 获取 container 的第一个子元素的实例对象 getPublicRootInstance
- 计算任务的过期时间，再根据任务过期时间创建 Update 任务，将任务(Update)存放于任务队列(updateQueue)中。然后判断任务是否为同步，调用同步任务入口
- 构建 workInProgress Fiber 树及 rootFiber
- 从父到子, 构建 Fiber 节点对象

（3）commit 阶段（渲染层）

commit 阶段负责根据 Fiber 节点标记 ( effectTag ) 进行相应的 DOM 操作，可以分为三个子阶段：

- before mutation 阶段（执行 DOM 操作前）
  + 处理类组件的 getSnapShotBeforeUpdate 生命周期函数
- mutation 阶段（执行 DOM 操作）
  + 根据 effectTag 执行 DOM 操作，向容器 container 中追加或插入节点
- layout 阶段（执行 DOM 操作后）
  + 调用生命周期函数和钩子函数，执行 render 传递的回调函数


### 2、为什么 React 16 版本中 render 阶段放弃了使用递归

因为如果使用递归进行 vdom 比对的话，由于递归使用 JS 自身执行栈，一旦开始就无法停止，直到任务执行完成。如果 vdom 树的层级较深， vdom 的比对就会长期占用 JS 主线程，由于 JS 是单线程的无法同时执行其他任务（界面渲染、动画执行等），在比对的过程中就无法响应用户操作，造成了页面卡顿。

所以在 React 16 的版本中，放弃了 JavaScript 递归的方式进行 virtualDOM 的比对，而是采用循环模拟递归。而且比对的过程是利用浏览器的空闲时间完成的，不会长期占用主线程，这就解决了 virtualDOM 比对造成页面卡顿的问题。

在 React 中，官方实现了自己的任务调度库 Scheduler。它可以实现在浏览器空闲时执行任务，而且还可以设置任务的优先级，高优先级任务先执行，低优先级任务后执行。 Scheduler 存储在 `packages/scheduler` 文件夹中。

React 16 后使用 fiber 架构可拆分，可中断任务
- 可重用各分阶段任务，可以设置任务优先级
- 可以在父子组件任务间前进后退切换任务
- render方法可以返回多元素（返回数组）
- 支持异常边界处理异常

> 在 window 对象中提供了 requestIdleCallback API，它可以利用浏览器的空闲时间执行任务，但是它存在一些浏览器兼容问题，而且它的触发频率也不是很稳定，所以 React 最终放弃了 requestIdleCallback 的使用，改用自己实现的 Scheduler 库。


### 3、请简述 React 16 版本中 commit 阶段的三个子阶段分别做了什么事情

commit 阶段负责根据 Fiber 节点标记 ( effectTag ) 进行相应的 DOM 操作，可以分为三个子阶段：

（1）before mutation 阶段（执行 DOM 操作前）
- 获取到 render 阶段执行完成后构建的待提交的 Fiber 对象
- 处理类组件的 getSnapShotBeforeUpdate 生命周期函数，在组件更新前捕获一些 DOM 信息

（2）mutation 阶段（执行 DOM 操作）
- 将 workInProgress Fiber 树变成 current Fiber 树
- 根据 effectTag 执行 DOM 操作 commitMutationEffects
- 挂载 DOM 元素 commitPlacement
- 获取 HostRootFiber 对象
- 向容器 container 中追加元素或插入到某一个节点的前面

（3）layout 阶段（执行 DOM 操作后）
- 调用生命周期函数和钩子函数（类组件处理生命周期函数，函数组件处理钩子函数）
- 如果调用 render 方法时传递了回调函数，则执行渲染完成之后的回调函数
- 通过遍历的方式调用 useEffect 中的回调函数
- 根据每个 effect.tag ，执行 destroy/create 操作，作用类似于 componentDidMount/componentWillUnmount

### 4、请简述 workInProgress Fiber 树存在的意义是什么

（1）双缓存技术

什么是双缓存？举个例子，使用 canvas 绘制动画时，在绘制每一帧前都会清除上一帧的画面，清除上一帧需要花费时间；如果当前帧画面计算量比较大，又需要花费比较长的时间。这就导致上一帧清除到下一帧显示中间会有较长的间隙，就会出现白屏。为了解决这个问题，我们可以在内存中绘制当前帧动画，绘制完毕后直接用当前帧替换上一帧画面，这样在帧画面替换的过程中就会节约很多时间，就不会出现白屏问题。这种在内存中构建并直接替换的技术叫做双缓存。

（2）React 16 中的双Fiber树

React 使用双缓存技术完成 Fiber 树的构建与替换，实现DOM对象的快速更新。

在 React 中最多会同时存在两棵 Fiber 树，当前屏幕中显示的内容对应的 Fiber 树叫做 current Fiber 树，当发生更新时，React 会在内存中构建一颗新的 Fiber 树叫做 workInProgress Fiber 树。在双缓存技术中，workInProgress Fiber 树就是即将要显示在页面中的 Fiber 树，当这颗 Fiber 树构建完成后，React 会使用它直接替换 current Fiber 树达到快速更新 DOM 的目的，因为 workInProgress Fiber 树是在内存中构建的所以构建它的速度是非常快的。

在 current Fiber 节点对象中有一个 alternate 属性指向对应的 workInProgress Fiber 节点对象，在 workInProgress Fiber 节点中有一个 alternate 属性也指向对应的 current Fiber 节点对象。两个 Fiber 节点通过 alternate 属性连接。

React 应用的根节点通过 current 指针在不同 Fiber 树的 rootFiber 间切换来实现 Fiber 树的切换。当 workInProgress Fiber 树构建完成交给 Renderer 渲染在页面上后，应用根节点的 current 指针指向 workInProgress Fiber 树，此时它就变为 current Fiber 树。每次状态更新都会产生新的 workInProgress Fiber 树，通过 current 与 workInProgress 的替换，完成DOM更新。

由于有两颗 Fiber 树，实现了异步中断时，更新状态的保存，中断回来以后可以拿到之前的状态。并且两者状态可以复用，节约了从头构建的时间。
