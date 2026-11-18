# Python 深挖：GIL、协程与对象模型的底层细节

## GIL 的 5ms 是怎么来的，能不能改

CPython 的字节码解释器在主循环里检查一个计数器，每执行若干条指令就尝试释放 GIL。这个“若干”由 `sys.getswitchinterval()` 控制，**默认 0.005 秒**。

- `sys.setswitchinterval(0.001)` 会让切换更频繁：I/O 密集场景响应更快，但切换开销上升
- 调大：吞吐略升，但可能出现“一个线程霸占 CPU 5ms 以上”的延迟毛刺

**关键点**：GIL 的释放不是“时间片用完强制抢占”，而是**线程主动检查**。所以一个执行长 C 函数的线程（不释放 GIL）会让所有 Python 线程一起等。

## 为什么 `+=` 不是原子操作

`counter += 1` 编译成四条字节码：

```
LOAD_FAST counter
LOAD_CONST 1
BINARY_ADD
STORE_FAST counter
```

GIL 只保证“单条字节码不被打断”，但四条之间可以切线程。于是两个线程各读到同一个旧值，各自加一，最后只加了一次。

所以“有 GIL 就不用加锁”是错的——GIL 保护的是解释器内部状态，不是你的业务不变量。

## asyncio 的底层到底是什么

事件循环本质是一个 `selector`（Linux 上是 epoll，Windows 上是 IOCP/proactor）：

1. 把所有待监听的 fd 注册到 selector
2. `select()` 阻塞，直到有 fd 就绪或超时
3. 就绪的 fd 对应的回调被放进就绪队列
4. 依次执行回调（协程的 `__next__` 被调用，从 await 处恢复）
5. 同时执行到期的时间事件（`call_later`）

所以 `await asyncio.sleep(1)` 不是睡眠，是“注册一个 1 秒后的定时器，然后把自己挂起”。

**追问：为什么一个阻塞调用能卡死整个循环？**
因为它占着唯一的线程。循环在此期间无法回到 `select()`，所有其他协程都得不到调度。

## 生成器和协程的差别：`send` 与 `throw`

两者机制相同（都是保存执行现场的函数），区别在用法：

- 生成器：`next()` 驱动，`yield` 往外送值，用途是“惰性序列”
- 协程（`async def`）：`send(None)` 驱动，`yield` 处等待的是 future，用途是“挂起与恢复”

`yield from` / `await` 都做了同一件事：**把 send/throw/close 透传给子迭代器**，并返回它的终值。这就是“协程可以嵌套”的实现基础。

**再追问：`throw` 用来干什么？**
向挂起处注入异常。`task.cancel()` 就是往协程里 throw 一个 `CancelledError`。

## 描述符的 `__set_name__` 解决什么问题

`__set_name__(self, owner, name)` 在类创建时被调用，让描述符知道“自己被赋给了哪个属性名”。

没有它，一个描述符实例被多个属性复用时，无法知道自己对应哪个字段——ORM 的字段、序列化器都要靠它。

**property 是最小的数据描述符**：只有 `__get__`/`__set__`/`__delete__`，没有 `__set_name__`（它的名字由 `property()` 调用时的 fget 名字决定）。

## dict 为什么能保持插入顺序

CPython 3.6+ 的 compact dict：一个**稀疏的索引数组** + 一个**紧凑的条目数组**。

- 条目数组按插入顺序追加 `(hash, key, value)`
- 索引数组用 hash 的低位映射到条目下标（开放寻址）

好处：遍历按插入顺序、内存更紧凑（不需要每格都留空）、删除只是把索引置空（条目留着，靠 resize 回收）。

## 追问链（三层）

**Q：GIL 存在，为什么 numpy 能多线程加速？**
A：numpy 的核心循环是 C 实现，且在执行长计算时会**主动释放 GIL**，所以多个线程能真正并行。同理 hashlib、zlib 的某些操作也会释放。

**追问：那为什么自己写的 C 扩展不一定能加速？**
A：要看它有没有在长循环里释放 GIL。不释放的话，多线程只是轮流跑，没有并行收益。

**再追问：怎么判断某个库会不会释放 GIL？**
A：查文档，或者实测——跑一个纯 CPU 任务，看多线程相比单线程有没有加速。

**Q：为什么 asyncio 里不能直接调 `requests.get`？**
A：requests 是同步阻塞的，会占住事件循环线程，所有协程停摆。要么换 httpx/aiohttp，要么用 `to_thread` 丢到线程池。

**Q：`sys.getrefcount` 为什么总比预期多 1？**
A：因为把对象作为参数传给 `getrefcount` 本身会创建一次临时引用。
