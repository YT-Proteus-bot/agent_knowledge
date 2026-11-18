# asyncio：事件循环、协程与常见陷阱

## 核心概念

- **协程（coroutine）**：`async def` 定义的函数，调用它不会执行，只返回协程对象。
- **事件循环**：单线程的调度器，维护就绪队列和 I/O 多路复用（epoll/kqueue/IOCP）。
- **Task**：把协程包装成可调度的任务，`asyncio.create_task()`。
- **Future**：一个"未来会有结果"的占位符，Task 是它的子类。

`await` 的语义是：**把控制权交回事件循环**，等这个 awaitable 完成后再继续。所以 await 不是"阻塞等待"，是"挂起自己"。

## 一次 await 发生了什么

1. 当前协程挂起，保存执行位置
2. 事件循环去跑别的就绪任务
3. 被 await 的对象完成（I/O 就绪或 Future 置结果）
4. 回调把协程重新放回就绪队列
5. 协程从挂起点继续

## 五个最常见的坑：async 里调阻塞函数会卡死事件循环

**1. 在 async 函数里调阻塞代码。**
`time.sleep(1)`、`requests.get()`、同步数据库驱动——这些会卡住整个事件循环，所有并发请求一起变慢。必须换成 `asyncio.sleep` / `aiohttp` / 异步驱动，或者 `to_thread`。

**2. 忘记 await。**
`asyncio.create_task` 之外的协程调用不 await 就不会执行，而且不报错（只会警告 "coroutine was never awaited"）。

**3. 任务没被持有，被 GC 回收。**
`create_task` 的返回值要存起来（或用 `TaskGroup`），否则事件循环只持弱引用，任务可能中途消失。

**4. 取消没有被正确传播。**
`task.cancel()` 会在下一个 await 点抛 `CancelledError`。如果代码里 `except Exception` 吞掉了它，取消就失效。正确做法是 `except asyncio.CancelledError: raise`。

**5. 没有背压。**
一次 `gather` 一万个请求，会把连接池和内存打爆。要用 `Semaphore` 限流，或者分批。

## 并发组合子

- `asyncio.gather(*aws)`：并发跑，全部完成返回列表；默认一个报错就抛出（`return_exceptions=True` 可收集异常）。
- `asyncio.wait(aws, return_when=...)`：更细的控制（FIRST_COMPLETED / FIRST_EXCEPTION / ALL_COMPLETED）。
- `asyncio.as_completed`：谁先完成先处理。
- `TaskGroup`（3.11+）：结构化并发，退出时保证所有子任务都结束，异常聚合。**推荐用它替代 gather**。

## 超时与限流

- `asyncio.wait_for(coro, timeout)`：超时抛 `TimeoutError`。
- `asyncio.timeout()`（3.11+）：上下文管理器写法，更清晰。
- `Semaphore(n)`：限制并发数，配合 `async with`。
- 连接池（如 httpx 的 limits）：比信号量更贴近真实瓶颈。

## 与线程/进程的关系

- 协程解决"大量并发 I/O"，不解决 CPU 密集。
- 协程里要跑 CPU 密集任务：`run_in_executor(ProcessPoolExecutor())`。
- 协程与线程可以共存，但共享状态要加锁（协程之间是协作式切换，通常单线程内不需要锁，但 await 点之间仍可能交错）。

## 追问链（三层）

**Q：asyncio 为什么比线程快？**
A：不是"更快"，是"更省"。切换在用户态（一次函数调用），线程切换要进内核；协程栈只有 KB 级。所以同样内存能撑更多并发。

**追问：协程是并行吗？**
A：不是，是并发。单线程内交替执行，同一时刻只有一个在跑。要并行得用进程或多线程。

**再追问：那 asyncio 有什么用？**
A：在 I/O 等待期间让出 CPU 去处理别的请求。它的价值是"用单线程撑住高并发 I/O"，不是"算得更快"。

**Q：为什么不能在所有地方都用 async？**
A：因为它是"传染"的：一个函数变 async，调用它的所有函数都要变。而且生态里大量库还是同步的。所以现实是同步/异步混合，边界处用线程池桥接。

**Q：事件循环里能不能跑 CPU 密集任务？**
A：不能，会阻塞所有任务。要么丢进程池，要么把任务拆小并用 `await asyncio.sleep(0)` 主动让出（治标不治本）。
