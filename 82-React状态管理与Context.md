# React 状态管理：Context 的性能陷阱与外部 store

## 状态该放哪

三层判断：

1. **只在组件内用** → `useState`
2. **兄弟组件共享、层级不深** → 状态提升到共同父组件
3. **跨层级、跨页面、生命周期长** → Context 或外部 store

很多项目一上来就上 Redux，结果是"所有状态都全局化"，依赖关系反而看不清。

## Context 的两个坑

**坑 1：value 变化会让所有 consumer 重渲染。**
Context 没有内置的 selector 机制。只要 Provider 的 value 引用变了，所有 `useContext` 的组件都会重渲染——哪怕它们只用到 value 里的一小部分。

解法：

- 拆成多个 Context（按更新频率拆）
- value 用 `useMemo` 稳定引用
- 改用支持 selector 的外部 store

**坑 2：Provider 里放内联对象。**
`<Ctx.Provider value={{a, b}}>` 每次渲染都是新对象，等于每次都在通知所有消费者。必须 `useMemo`。

## 外部 store 与 useSyncExternalStore

Redux/Zustand 这类库把状态放在 React 之外，组件用订阅的方式读。

React 18 提供 `useSyncExternalStore(subscribe, getSnapshot)`，专门用于这种"外部可变状态"。它的存在是为了解决**并发渲染下的 tearing**：如果组件树的一部分读到了新值、另一部分读到旧值，画面就不一致。这个 Hook 保证同一次渲染里读到同一个快照。

## 各方案定位

- **Redux Toolkit**：重规范、强调可预测性、devtools 强、适合大团队/复杂业务。样板代码多，但 RTK 已经削掉大半。
- **Zustand**：极轻，没有 Provider，组件用 selector 订阅，只有选中的切片变化才重渲染。中小项目首选。
- **Jotai / Recoil**：原子化，状态分散成小单元，依赖关系天然细粒度。适合状态之间关系复杂的场景。
- **MobX**：响应式，写法接近 Vue。团队熟悉响应式时上手快。

## 不可变更新

React 靠引用比较判断变化，所以必须返回新对象：

```js
setUser({ ...user, name: 'x' })          // 浅层
setList(list.map(i => i.id === id ? {...i, done: true} : i))
```

深层嵌套很痛苦，通常用 **immer** 的 `produce` 写"看起来可变"的代码，内部生成不可变副本（结构共享，只复制变了的路径）。

## 追问链（三层）

**Q：为什么 Context 不能像 Zustand 那样只订阅一部分？**
A：因为 Context 的传播机制是"value 变了就通知所有消费者"，没有依赖追踪。要做细粒度，必须在消费端加 selector，而这需要外部 store 的订阅机制。

**追问：那拆 Context 不就解决了？**
A：部分解决。拆得越细，Provider 嵌套越深、组件耦合越重。状态多、更新频繁时，拆 Context 会很痛苦，这时外部 store 更合适。

**再追问：Zustand 为什么不需要 Provider？**
A：它把 store 放在模块作用域，组件直接 `useStore(selector)` 订阅。Provider 只是为了让 SSR 时不串数据才需要（每个请求一个 store 实例）。

**Q：为什么 Redux 要求 reducer 是纯函数？**
A：因为要支持时间旅行、可重放、可测试。纯函数意味着"同样的输入必然同样的输出"，才能把 action 序列重放出来。副作用放 middleware。

**Q：什么时候不该用全局状态？**
A：数据只在局部用、或者可以派生计算出来（用 `useMemo` 而不是再存一份）。重复存储是状态不同步的主要来源。
