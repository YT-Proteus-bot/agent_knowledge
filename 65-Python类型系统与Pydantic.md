# Python 类型系统与 Pydantic

## typing 的常用工具

- `list[int]` / `dict[str, int]`：3.9+ 内置泛型，不用 `List`
- `Optional[X]` = `X | None`：注意它不等于"有默认值"
- `Union[A, B]` = `A | B`（3.10+）
- `TypeVar`：泛型参数；`Generic[T]` 定义泛型类
- `Protocol`：结构化子类型（鸭子类型的静态版），只要方法签名匹配就算实现，不需要继承
- `Literal["a","b"]`：枚举字面量，比 str 更安全
- `TypedDict`：给 dict 加键的类型，适合 JSON 结构
- `ParamSpec`：给装饰器保留被装饰函数的签名
- `Annotated[int, "范围 0-100"]`：附加元信息，FastAPI/Pydantic 大量使用

## 类型检查是"开发期工具"，不是运行时

Python 的注解默认**不执行**，`mypy` / `pyright` 在 CI 里检查。所以：

- 注解写错不会在运行时炸，只会被工具发现
- 想运行时校验，要用 Pydantic、`typeguard` 这类库
- `from __future__ import annotations` 让注解变成字符串（延迟求值），解决前向引用问题

## dataclass / attrs / Pydantic 怎么选

| 库 | 运行时校验 | 性能 | 适合 |
|---|---|---|---|
| dataclass | 无 | 快 | 内部数据结构 |
| attrs | 可选 | 快 | 需要 slots/validators |
| Pydantic | 有 | v2 用 Rust 核心，很快 | 边界输入（API、配置、消息） |

判断标准：**数据来自外部（不可信）就用 Pydantic；内部构造用 dataclass**。

## Pydantic v2 的关键点

- v2 核心用 Rust 重写（pydantic-core），校验速度比 v1 快数倍。
- `model_validate(dict)` / `model_dump()` 替代 v1 的 `parse_obj` / `dict()`。
- `field_validator` / `model_validator` 替代 v1 的 `validator` / `root_validator`。
- `model_config = ConfigDict(extra="forbid")`：拒绝多余字段（对外 API 建议开）。
- `TypeAdapter`：给非 BaseModel 的类型（如 `list[int]`）做校验。
- 严格模式 `strict=True`：不做隐式转换（"123" 不会变成 123）。

## 常见设计取舍

- **边界校验 vs 内部断言**：只在系统边界（HTTP、消息、配置）做重校验，内部传已校验的对象，避免重复开销。
- **Optional 滥用**：`Optional` 会让调用方到处判空。能不给默认值就别给，让缺字段直接报错。
- **类型即文档**：函数签名写清楚比写 docstring 更不容易过期。

## 追问链（三层）

**Q：Python 是动态语言，写类型有什么意义？**
A：三个收益：IDE 补全和重构、CI 里提前发现低级错误、以及作为接口契约。代价是要维护注解，且运行时不保证。

**追问：那运行时怎么保证？**
A：在边界用 Pydantic 校验一次，内部信任。这就是"边界校验、内部信任"的原则。

**再追问：为什么不全程校验？**
A：性能开销大，而且重复校验没有新增信息。校验一次后，内部类型错误应该靠测试和代码审查。

**Q：Protocol 和 ABC 的区别？**
A：ABC 是名义子类型（必须显式继承），Protocol 是结构子类型（鸭子类型）。对接第三方库、不想改对方代码时用 Protocol。

**Q：TypeVar 约束和 bound 的区别？**
A：`TypeVar("T", A, B)` 是枚举允许的类型；`TypeVar("T", bound=Base)` 是要求是某类的子类。前者限制更死，后者更灵活。
