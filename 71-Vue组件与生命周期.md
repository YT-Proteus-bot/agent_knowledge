# Vue 组件、生命周期与 Composition API

## 两种写法

- **Options API**：`data / methods / computed / watch / mounted`，按选项组织。
- **Composition API**：`setup()` 里用 `ref / computed / watch / onMounted`，按逻辑组织。

组合式的好处不是"新"，是**逻辑可以抽成函数复用**（composable），而不是靠 mixin（mixin 有命名冲突和来源不清的问题）。

## setup 的执行时机

`setup` 在 `beforeCreate` 之前执行（此时 `this` 不可用），返回的对象暴露给模板。`<script setup>` 是编译糖：顶层变量自动暴露，少写 return。

## 生命周期对照

| Options | Composition |
|---|---|
| beforeCreate / created | setup（不需要，逻辑直接写） |
| beforeMount | onBeforeMount |
| mounted | onMounted |
| beforeUpdate | onBeforeUpdate |
| updated | onUpdated |
| beforeUnmount | onBeforeUnmount |
| unmounted | onUnmounted |

注意：`onMounted` 里能拿到 DOM，但**子组件不一定全部挂载完**（父的 mounted 在子的 mounted 之后）。

## 组件通信的六种方式

1. **props / emit**：父子，最常用。
2. **v-model**：本质是 `:modelValue` + `@update:modelValue`，Vue 3 支持多个 v-model（`v-model:title`）。
3. **provide / inject**：跨层级，避免 props 逐层透传。注入的是响应式对象时要小心解构。
4. **attrs**：透传未声明的属性（`v-bind="$attrs"`），`inheritAttrs: false` 可关闭自动透传。
5. **slots**：内容分发，作用域插槽能把子组件数据传给父。
6. **全局状态（Pinia）**：跨页面/跨模块。

选择原则：能 props 就 props，跨三层以上再 provide/inject，全局共享才上 store。

## 插槽与动态组件

- 默认插槽、具名插槽、作用域插槽（`<slot :item="item">`）。
- `<component :is="comp">` 动态切换组件。
- `<KeepAlive>` 缓存组件实例（保留表单状态），配 `include/exclude` 控制。

## 响应式丢失的三种情况

1. 解构 `reactive` 对象（用 `toRefs`）
2. 把 `ref` 当普通值传递（传 `.value` 就丢了）
3. 替换整个 `reactive` 对象（`state = {...}` 会断开代理，应该逐个赋值或用 `ref`）

## 追问链（三层）

**Q：为什么 `onMounted` 里拿不到子组件的数据？**
A：挂载顺序是子先父后，但父的 `onMounted` 执行时子组件可能还在异步加载（`<Suspense>` 或异步组件）。要等 `nextTick` 或 `Suspense` 的 resolve。

**追问：`nextTick` 到底等什么？**
A：等 Vue 的更新队列（微任务）flush 完，DOM 已经 patch 完。它不是"等浏览器渲染"，只保证 DOM 树更新了。

**再追问：那要等布局完成呢？**
A：用 `requestAnimationFrame` 或 `ResizeObserver`。`nextTick` 之后样式可能还没算，读 `offsetHeight` 会强制同步布局（有性能代价）。

**Q：provide/inject 的值改了，子组件会更新吗？**
A：如果 provide 的是 `ref` 或 `reactive`，会；如果 provide 的是普通值，不会。常见错误是 `provide('x', count)` 而不是 `provide('x', count)` 里传 ref 本身——传 ref 是对的，传 `.value` 就错了。

**Q：mixin 和 composable 的本质区别？**
A：mixin 是"把选项合并进组件"，来源不清晰、可能冲突；composable 是"普通函数返回状态和方法"，调用点明确、可以组合、可以传参。
