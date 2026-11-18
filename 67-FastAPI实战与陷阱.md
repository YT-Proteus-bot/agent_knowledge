# FastAPI：依赖注入、async 陷阱与生产部署

## 它是什么

FastAPI = Starlette（ASGI 框架，路由/中间件/WebSocket）+ Pydantic（校验/序列化）+ 类型驱动的自动文档。它本身很薄，性能主要来自 ASGI 和 Pydantic v2 的 Rust 核心。

## 依赖注入（Depends）

这是 FastAPI 最有设计感的部分：

- `def get_db(): ...`，路由参数写 `db=Depends(get_db)`，框架负责调用并注入。
- **同一请求内默认缓存**：多个依赖引用同一个 `get_db`，只会执行一次（可用 `use_cache=False` 关闭）。
- **yield 依赖**：`yield` 之前是 setup，之后是 teardown（异常也会执行），适合数据库会话、临时资源。
- **类依赖**：`Depends(SomeClass)` 会自动用构造函数的类型注解做校验。
- **全局依赖**：`app = FastAPI(dependencies=[Depends(verify_token)])`，适合鉴权。

依赖的边界要清楚：它做的是"参数解析 + 资源准备 + 鉴权"，不要在里面写业务逻辑。

## async def 与 def 的区别（最大的坑）

- `async def` 路由：在事件循环里直接跑。**里面有任何阻塞调用都会卡死整个循环**。
- `def` 路由：FastAPI 会把它丢到线程池跑，不阻塞循环。

所以：

- 用同步库（requests、同步 ORM）→ 写 `def`，别写 `async def`
- 用异步库（httpx、asyncpg）→ 写 `async def`
- **绝对不要在 `async def` 里调 `time.sleep` / `requests.get` / 同步数据库**

这一条在真实项目里造成的事故最多，面试也最爱问。

## 校验与序列化

- 请求体用 Pydantic 模型，自动校验并返回 422。
- `response_model` 控制输出字段（防止泄露内部字段），也影响性能（要做序列化）。
- 路径/查询参数用 `Path()` / `Query()` 加约束（`ge`、`le`、`max_length`）。
- `model_config = ConfigDict(extra="forbid")` 拒绝多余字段，对外 API 建议开。

## 中间件 vs 依赖

- **中间件**：包住整个请求，能改请求/响应，适合日志、CORS、耗时统计、trace id。
- **依赖**：拿到的是解析后的参数，适合鉴权、资源注入。

中间件不能轻易拿请求体（会消费流），这是常见限制。

## 后台任务与队列

- `BackgroundTasks`：请求返回后在同一进程里跑，适合发邮件这种轻量、可丢的任务。
- 真正的异步任务用 Celery/RQ/Arq：要持久化、可重试、可观测，就别用 BackgroundTasks。

## 生命周期

用 `lifespan`（async context manager）替代已废弃的 `@app.on_event("startup")`：启动时建连接池，关闭时释放。注意多 worker 下每个 worker 都会执行一次。

## 部署

- `uvicorn app:app --workers N`（或 gunicorn + uvicorn worker）。
- workers 数通常 `2 × CPU 核数 + 1`，但要结合内存和是否 CPU 密集。
- 前置 Nginx 做 TLS、静态文件、限流。
- 健康检查分 liveness / readiness：readiness 要检查依赖（数据库、缓存）。

## 追问链（三层）

**Q：FastAPI 为什么快？**
A：三个原因：ASGI 异步、Pydantic v2 的 Rust 核心做校验、以及 Starlette 本身很薄。但"快"是相对的——如果路由里跑同步阻塞代码，再快也没用。

**追问：那它比 Flask 快多少？**
A：纯框架开销上快一个量级（尤其在高并发 I/O 场景），但真实业务里瓶颈通常在数据库和下游服务。框架选型不要只比 benchmark。

**再追问：什么时候 Flask/Django 更合适？**
A：团队熟、生态成熟、需要 Django 的 ORM/Admin 时。FastAPI 的优势在"类型驱动 + 异步 + 自动文档"，不在"一定更快"。

**Q：依赖注入会不会拖慢性能？**
A：有一点开销（解析签名、调用依赖），但很薄。真正的性能问题是依赖里做了慢操作（比如每次请求新建数据库连接）。用连接池和缓存解决。

**Q：怎么在 FastAPI 里做限流？**
A：简单场景用中间件 + Redis 计数（滑动窗口/令牌桶）；复杂场景交给网关（Nginx/Kong）或专门的限流库。应用层限流只适合兜底。
