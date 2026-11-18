# React 渲染模型与 Fiber

## JSX 到 DOM 的三步

1. **编译**：JSX → `React.createElement(type, props, children)`（React 17+ 用自动 runtime，不再需要 import React）
2. **生成元素**：得到 React element —— 一个普通对象 `{ type, key, props }`，很轻，不是 DOM
3. **渲染**：React 根据元素树构建/更新 Fiber 树，最后把变更提交到 DOM

React element 是"描述"，Fiber 是"工作单元"，DOM 是"结果"。三者别混。

## Fiber 是什么

Fiber 是 React 16 引入的**可中断的渲染架构**。每个 Fiber 节点是一个工作单元，用链表连接：

- `child`：第一个子节点
- `sibling`：下一个兄弟
- `return`：父节点

用链表而不是树递归，是为了**能暂停和恢复**：遍历到一半可以停下来处理更高优先级的更新，之后再从断点继续。

## 双缓存

React 同时维护两棵树：

- `current`：当前屏幕上的
- `workInProgress`：正在构建的

两者通过 `alternate` 互相引用。构建完成后一次性切换 `root.current`，所以用户永远看不到"半成品"。

## 两个阶段

- **Render 阶段（可中断）**：构建 workInProgress 树，计算差异。不碰 DOM，所以可以暂停、丢弃、重做。
- **Commit 阶段（同步）**：把变更一次性写到 DOM（`mutation`），然后执行 `useLayoutEffect`，再 `useEffect`。

这个划分解释了：为什么 render 阶段不能有副作用（可能被多次执行/丢弃），而 commit 阶段可以。

## 优先级与并发

React 18 用 **lane** 模型表示优先级（位运算），不同更新走不同通道。两个常用 API：

- `startTransition`：把更新标记为"非紧急"，可被打断（比如搜索框输入时更新结果列表）。
- `useDeferredValue`：给某个值一个"延迟版本"，让渲染先用旧值。

典型场景：输入框输入触发大列表过滤。输入是紧急的，列表更新是可延迟的。

## 自动批处理

React 18 之前只有事件处理函数里的 setState 会批处理，setTimeout/Promise 里的不批。18 之后**所有更新都自动批处理**（同一轮事件循环里合并成一次渲染）。

想强制同步刷新可以用 `flushSync`，但会破坏批处理，慎用。

## StrictMode 为什么执行两次

开发模式下，StrictMode 会故意 double-invoke render、effect（mount → unmount → mount），用来暴露副作用没清理干净的问题。生产不执行。看到"接口被调两次"先确认是不是 StrictMode。

## 追问链（三层）

**Q：为什么 React 需要 Fiber，Vue 不需要？**
A：Vue 的响应式能精确知道"哪个组件变了"，更新是细粒度的；React 是"状态变了就重渲染组件树"，需要一个可中断的调度器来避免长任务卡顿。目标不同。

**追问：Fiber 解决了什么问题？**
A：老版本递归渲染一旦开始就不能停，大组件树会阻塞主线程几百毫秒，输入卡顿。Fiber 把渲染拆成小单元，可以按优先级让路。

**再追问：那是不是所有更新都该可中断？**
A：不是。commit 必须同步，否则用户会看到半更新的 UI。而且频繁中断也有调度开销，所以 React 用 lane 区分紧急程度。

**Q：render 阶段为什么不能有副作用？**
A：因为可能被多次执行（并发、StrictMode）或被丢弃。副作用应该放 `useEffect`（commit 后）或 `useLayoutEffect`（DOM 更新后、绘制前）。

**Q：`useLayoutEffect` 和 `useEffect` 的区别？**
A：`useLayoutEffect` 在 DOM 变更后、浏览器绘制前同步执行，会阻塞绘制；`useEffect` 在绘制后异步执行。需要读布局（`offsetHeight`）或避免闪烁时用前者。
