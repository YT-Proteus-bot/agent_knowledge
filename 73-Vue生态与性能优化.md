# Vue 生态与性能优化：Pinia、路由、SSR

## Pinia

Vue 3 的官方状态库，替代 Vuex。

- **没有 mutations**：直接改 state，或者写 action。Vuex 的 mutation/action 二分在 Pinia 里被砍掉，因为响应式本身就能追踪变化。
- **setup store**：用 `defineStore('id', () => { const count = ref(0); ...; return {...} })`，和组件写法一致，类型推导最好。
- **模块化天然**：一个 store 一个文件，没有嵌套 modules。
- **devtools / SSR 友好**：支持时间旅行、服务端状态序列化。

什么时候该用：跨页面、跨组件共享，且生命周期比组件长。**不要把所有状态都塞 store**——组件内部状态留在组件里。

## Vue Router

- **hash 模式**：`#/path`，不需要服务端配置，但 URL 难看、不利于 SEO。
- **history 模式**：`/path`，需要服务端把所有路径回退到 `index.html`。
- **路由守卫**：`beforeEach`（全局）、`beforeEnter`（路由级）、组件内 `beforeRouteEnter`。异步守卫要 `await`，否则会放行。
- **懒加载**：`component: () => import('./X.vue')`，按路由分包。
- **keep-alive**：缓存列表页，返回时保留滚动位置和筛选条件。

## SSR 与 hydration

SSR 的流程：服务端渲染出 HTML → 浏览器先展示（可交互前）→ 客户端加载 JS 并"接管"（hydration）。

**hydration mismatch** 是最常见的坑：服务端和客户端渲染结果不一致，Vue 会丢弃服务端 DOM 重新渲染（性能损失 + 闪烁）。

常见原因：

- 用了 `Date.now()`、`Math.random()`、`window` 相关值
- 服务端和客户端的数据不一致（接口在两端各调一次）
- 条件渲染依赖 `process.client` 判断错误

解法：把"只在客户端"的逻辑放到 `onMounted`，或者用 `<ClientOnly>`。

## 性能优化清单

**渲染层**

- `v-if` / `v-show` 按切换频率选
- `v-memo`（3.2+）缓存子树，只有依赖变化才更新
- `shallowRef` / `markRaw` 减少代理开销
- 长列表用虚拟滚动（vue-virtual-scroller），不要一次渲染几千条

**加载层**

- 路由懒加载、组件异步加载
- 图片懒加载 + 响应式尺寸
- 打包分析（`rollup-plugin-visualizer`）找大依赖
- 第三方库按需引入（Element Plus / Ant Design Vue 都支持）

**运行时**

- 避免在 `computed` 里做重计算（用缓存或 worker）
- 大对象不要放 `data`/`ref`（会深度代理）
- 频繁更新用 `requestAnimationFrame` 节流

## 追问链（三层）

**Q：为什么 Pinia 去掉了 mutation？**
A：因为 mutation 的初衷是"让 devtools 能追踪状态变化"，而 Vue 3 的响应式本身就能追踪。多一层 mutation 只是增加样板代码。

**追问：那怎么保证状态变更可预测？**
A：靠约定：所有修改走 action，不在组件里直接改 store。类型系统（TypeScript）也能约束。

**再追问：什么时候不该用 Pinia？**
A：状态只在一个组件树里用时，用 `provide/inject` 或 props 更轻。全局 store 会让依赖关系变隐蔽。

**Q：SSR 一定比 CSR 快吗？**
A：首屏内容更快可见（HTML 直出），但 TTFB 可能更慢（服务端要渲染），且 hydration 期间不可交互。对内容型页面收益大，对重交互应用收益小。

**Q：虚拟列表的原理？**
A：只渲染可视区域内的项（加少量缓冲），滚动时动态替换。关键是"项高度已知或可估算"，否则要做动态测量。

**Q：Vue 项目体积大怎么查？**
A：先 `visualizer` 看依赖占比，再看是否有全量引入的 UI 库、重复的 polyfill、没被 tree-shaking 的 CJS 包。
