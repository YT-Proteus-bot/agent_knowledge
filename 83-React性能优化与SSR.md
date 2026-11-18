# React 性能优化与 SSR/RSC

## 先搞清"为什么会重渲染"

React 的默认行为：**父组件重渲染，所有子组件都重渲染**（除非被 memo 挡住）。三个触发源：

1. 自身 state 变化
2. 父组件重渲染
3. 订阅的 Context/store 变化

优化的第一步不是加 memo，是**减少不必要的 state**（能从 props 派生的就别存）。

## 四件套

- `React.memo(Comp)`：浅比较 props，相同就跳过渲染。对对象/函数 props 无效（引用每次都变）。
- `useMemo`：缓存计算结果。
- `useCallback`：缓存函数引用。
- `key`：列表 diff 的依据，用稳定 id，不要用 index。

判断该不该加：用 React DevTools Profiler 看"为什么渲染"。如果 props 没变却渲染了，才需要 memo。

## 大列表与长任务

- **虚拟列表**（react-window / react-virtualized）：只渲染可视区。
- **分页/无限滚动**：减少单次渲染量。
- **`startTransition`**：把"结果列表更新"标为非紧急，让输入框先响应。
- **Web Worker**：把重计算移出主线程（注意不能直接传 DOM）。

## 代码分割

- `React.lazy(() => import('./X'))` + `<Suspense fallback>`
- 路由级分割（每个页面一个 chunk）
- 预加载：`import(/* webpackPrefetch */ './X')` 或 `<link rel="prefetch">`

## SSR: 流程与坑

1. 服务端 `renderToString`（或 `renderToPipeableStream` 流式）生成 HTML
2. 浏览器先展示静态 HTML（首屏可见，但不可交互）
3. 加载 JS，`hydrateRoot` 接管事件与状态

**hydration mismatch**：服务端和客户端渲染结果不一致时，React 会警告并回退到客户端重渲染（丢失 SSR 收益）。

常见原因：`Date.now()`、`Math.random()`、`window`/`localStorage`、以及"服务端和客户端请求了不同的数据"。

解法：把客户端专属逻辑放 `useEffect`，或者用 `suppressHydrationWarning`（仅限文本、慎用）。

## RSC（React Server Components）

思路：组件在服务端渲染，**不把它的 JS 发到客户端**。只有需要交互的部分标 `"use client"`。

收益：bundle 变小、能直接访问数据库/文件系统。
代价：心智模型变复杂（server/client 边界、序列化限制、不能传函数）。

这是 Next.js App Router 的核心，也是 React 近几年的主要方向。

## 追问链（三层）

**Q：`React.memo` 为什么经常没用？**
A：因为 props 里的对象/函数每次渲染都是新引用，浅比较必然不等。要配合 `useMemo`/`useCallback` 稳定引用，否则 memo 形同虚设。

**追问：那不是到处都要包 useCallback？**
A：不是。只有当函数被传给 memo 组件、或者作为其他 Hook 的依赖时才需要。乱加反而增加开销和心智负担。

**再追问：有没有更省心的办法？**
A：React Compiler（自动 memo）方向就是解决这个——把"该不该 memo"交给编译器。但在它普及前，还是要理解原理。

**Q：SSR 一定比 CSR 快吗？**
A：首屏内容更快可见（HTML 直出），但 TTFB 可能更慢、hydration 期间不可交互。内容型页面收益大，重交互后台收益小。

**Q：为什么 hydration 不能直接复用 DOM 就完事？**
A：因为服务端只有 HTML，没有事件监听和组件状态。hydration 要把 Fiber 树和 DOM 对应起来、绑定事件、恢复状态，这本身有成本。

**Q：RSC 和 SSR 是一回事吗？**
A：不是。SSR 是"服务端渲染 HTML 再 hydration"，组件代码仍要发到客户端；RSC 是"组件根本不在客户端运行"，不发它的 JS。两者可以叠加。
