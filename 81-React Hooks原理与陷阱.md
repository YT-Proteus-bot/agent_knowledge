# React Hooks：原理、闭包陷阱与依赖数组

## Hooks 存在哪

每个 Fiber 节点有一个 `memoizedState` 链表，按**调用顺序**存每个 Hook 的状态。所以：

- 不能在条件/循环里调 Hook（顺序会错位，React 会报错）
- 这也是"Hooks 规则"的根因，不是风格问题

## useState

- 惰性初始化：`useState(() => expensive())`，只在首次执行
- 函数式更新：`setCount(c => c + 1)`，避免拿到旧值
- 批处理：同一事件里多次 set 只渲染一次；React 18 起所有场景都自动批处理

## 闭包陷阱（最常考）

```js
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000)
  return () => clearInterval(id)
}, [])   // count 被闭包固定在 0
```

因为 effect 只在挂载时执行一次，回调里捕获的是当时的 `count`。两种解法：

1. 用函数式更新 `setCount(c => c + 1)`（不依赖外部值）
2. 把 `count` 放进依赖数组（每次变化重建定时器）

## useEffect 的依赖数组

- `[]`：只在挂载/卸载执行
- `[a, b]`：任一变化就执行
- 不写：每次渲染都执行（很少需要）

依赖比较用 `Object.is`。所以对象/函数每次渲染都是新引用，会导致 effect 频繁执行——这才是 `useMemo`/`useCallback` 的主要用途（不是为了"性能"，是为了**稳定引用**）。

清理函数在下次执行前和卸载时调用。异步 effect 不能直接 `async`，要在内部定义 async 函数再调用。

## useMemo / useCallback

- `useMemo(fn, deps)`：缓存计算结果
- `useCallback(fn, deps)`：缓存函数引用（等价于 `useMemo(() => fn, deps)`）

**不要默认到处加**。它们本身有成本（比较依赖、占内存）。真正的用途：

1. 传给 `React.memo` 子组件的 props 需要稳定引用
2. 作为其他 Hook 的依赖
3. 计算真的很贵

## useRef

- 返回 `{ current }`，改它不触发渲染
- 用途：存 DOM 引用、存定时器 id、存"上一次的值"、存不需要触发渲染的变量

## 自定义 Hook

以 `use` 开头、内部调用其他 Hook 的函数。本质是逻辑复用，不是状态共享——每个调用点有独立状态。

## 追问链（三层）

**Q：为什么 Hook 不能写在 if 里？**
A：因为状态是按调用顺序存在链表上的。条件调用会让后续 Hook 的位置错位，React 无法对应。这不是限制，是实现的必然。

**追问：那 Vue 的 Composition API 为什么可以条件调用？**
A：Vue 的响应式是基于 Proxy 的自动依赖收集，每个 ref 独立存在，不依赖"调用顺序"。这是两套完全不同的机制。

**再追问：React 有没有办法绕开？**
A：没有官方办法。所有"条件逻辑"要写在 Hook 内部（用参数控制），而不是条件调用 Hook。

**Q：`useEffect` 和 `useLayoutEffect` 什么时候用哪个？**
A：需要读 DOM 布局、或者要避免视觉闪烁（比如测量后立刻改样式）用 `useLayoutEffect`；其余都用 `useEffect`。SSR 环境下 `useLayoutEffect` 会警告（服务端没有 DOM）。

**Q：为什么 setState 之后拿不到最新值？**
A：因为 state 更新是异步批处理的，当前闭包里的变量还是旧值。要拿最新值用函数式更新，或者在 `useEffect` 里读。

**Q：`React.memo` 和 `useMemo` 有什么区别？**
A：`React.memo` 包组件，浅比较 props 决定是否重渲染；`useMemo` 缓存一个值。前者防"组件重渲染"，后者防"重复计算"。
