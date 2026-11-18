# Django / Flask 与 ORM：N+1、事务与 WSGI

## WSGI 与 ASGI

- **WSGI**：同步协议，一个请求占一个线程/进程。Django（传统）、Flask 都是。
- **ASGI**：异步协议，支持 WebSocket、长连接、高并发 I/O。Django 3+ 和 FastAPI 支持。

选型不是"ASGI 更先进"，而是"你的瓶颈在不在 I/O 等待"。CPU 密集或简单 CRUD，WSGI + 多 worker 完全够用。

## Django ORM 的五个必知点

**1. QuerySet 是惰性的。** 定义时不查库，遍历/切片/`len()` 时才执行。所以"循环里查库"极易变成 N+1。

**2. `select_related` vs `prefetch_related`**

- `select_related`：SQL JOIN，适合**外键/一对一**（单值关系）。
- `prefetch_related`：额外一次查询再在 Python 里拼装，适合**多对多/反向外键**（集合关系）。
- 用错不会报错，只会慢。判断依据是关系是"一个"还是"一堆"。

**3. 事务**

- `transaction.atomic()` 块内要么全成要么全滚。
- 注意：**异常被吞掉事务不会回滚**，必须让异常抛出，或者手动 `set_rollback()`。
- 长事务会持锁，别把网络请求放进事务里。

**4. 并发更新**

`select_for_update()` 加行锁，防"读-改-写"丢更新。要配合事务使用，且注意锁顺序避免死锁。

**5. 批量操作**

`bulk_create` / `bulk_update` 比循环 `save()` 快一个量级，但不触发 `save()` 信号、不调用 `full_clean()`。批量导入场景要接受这个代价。

## 迁移（migrations）

- 迁移文件要进版本控制，且**不要手改已应用的迁移**。
- 大表加字段/加索引会锁表，生产要评估：加索引用 `CONCURRENTLY`（PostgreSQL）、加字段用 nullable 或给默认值。
- 数据迁移和结构迁移分开写，便于回滚。

## Flask 的上下文

- `request`：当前请求
- `g`：请求级临时存储
- `current_app`：应用实例

它们靠 **contextvars**（3.7+）实现，所以异步环境下也能正确隔离。老版本用线程本地存储，在 async 里会串。

Blueprint 做模块化路由；扩展生态（SQLAlchemy、Flask-Login）比 Django 松散但灵活。

## 追问链（三层）

**Q：N+1 怎么发现？**
A：开发期用 `django-debug-toolbar` 看每页 SQL 条数；或者开慢查询日志；或者用 `assertNumQueries` 写测试。生产上用 APM（SkyWalking/DataDog）看 SQL 分布。

**追问：`prefetch_related` 一定比循环查快吗？**
A：通常快很多（1 次 vs N 次），但它会把所有关联对象加载进内存。数据量大时要配合 `Prefetch` 对象加过滤，否则内存会炸。

**再追问：什么时候干脆不查关联？**
A：只需要计数时用 `annotate(Count(...))`，只需要判断存在时用 `exists()`，别把对象加载出来。

**Q：`transaction.atomic` 嵌套会怎样？**
A：内层是 savepoint，内层回滚不影响外层已做的操作（除非外层也回滚）。但 Django 默认的 autocommit 下，异常必须抛出才会回滚。

**Q：Django 和 FastAPI 怎么选？**
A：要 Admin、完整 ORM、成熟生态、快速交付 → Django。要异步、类型驱动、轻量 API → FastAPI。两者都能做 API，差别在"你愿意为哪些能力付复杂度"。
