# 给 Java 工程师的 Python 深入学习手册

> 目标读者：**Java 功底扎实，但 Python 近乎零基础**的开发者。
> 设计原则：**不从零教语言，而是处处用 Java 当跳板**，把篇幅花在"形似神不同"的地方——那正是 Java 老手最容易翻车的地方。

---

## 怎么用这套资料

每一章固定五个区块，建议按区块的目的来读：

| 区块 | 给你的作用 |
|------|-----------|
| ① **Java 对照表** | 5 秒建立映射，知道"我在 Java 里的 X 对应 Python 的什么" |
| ② **核心讲解** | 概念与底层原理，重点讲和 Java 不同的部分 |
| ③ **最佳实践** | 地道（Pythonic）写法、工程约定 |
| ④ ⚠️ **易错点** | "Java 习惯 → 翻车现场 → 正确写法"，每条都是真实陷阱 |
| ⑤ **练习题** | 基础理解 / 找坑改错 / 综合实战，三档，附答案讲解 |

**学习建议**：每章读完，至少把 ④ 和 ⑤ 亲手敲一遍。Python 的坑几乎都"看懂了但还是会犯"，必须用手记住。

---

## 章节地图与阅读顺序（24 章）

> 标 ⭐ 的是 Java 老手最易忽视、但最该重点啃的章。

### 第一部分 · 地基与运行时（建立心智模型 + 理解底层）
- **[01 设计哲学与心智模型](01_设计哲学与心智模型.md)** — Python 之禅、动态强类型、鸭子类型、一切皆对象、EAFP、GIL 概览。
- **[02 环境与工具链](02_环境与工具链.md)** — 解释器、虚拟环境、`uv`/pip、`pyproject.toml`。对照 Maven/Gradle/classpath。
- **[03 变量与对象模型](03_变量与对象模型.md)** — ⭐引用语义、可变/不可变、`is` vs `==`、整数缓存、深浅拷贝。Java 老手坑的总源头。
- **[04 运行时与内存模型](04_运行时与内存模型.md)** — ⭐**垃圾回收**：引用计数 + 循环 GC，对照 JVM 分代追踪式 GC。`__del__`、`weakref`、`gc` 模块、循环引用。
- **[05 CPython 内部](05_CPython内部.md)** — ⭐对照 JVM：字节码与 `dis`、`ceval` 求值循环、`PyObject` 结构、interning、GIL 机制、为什么慢、C 扩展与 FFI。

### 第二部分 · 数据与基础类型
- **[06 内置数据结构](06_内置数据结构.md)** — list/dict/set/tuple 对照集合框架，底层实现（动态数组/compact dict）与复杂度。
- **[07 collections 模块](07_collections模块.md)** — ⭐defaultdict/Counter/deque/namedtuple/heapq/bisect/array，对照 Guava/JUC。
- **[08 数字与算术语义](08_数字与算术语义.md)** — ⭐整数**任意精度永不溢出**、`/` vs `//`、`bool` 是 `int`、`Decimal`/`Fraction`。Java 老手必踩。
- **[09 字符串、字节与编码](09_字符串字节与编码.md)** — ⭐`str` vs `bytes`、Unicode、PEP 393、encode/decode、f-string。对照 `String`/`char`/`byte[]`/`Charset`。
- **[10 正则表达式 re](10_正则表达式.md)** — 回溯式引擎、灾难性回溯/ReDoS、命名组、`re.compile` 缓存。对照 `java.util.regex`。

### 第三部分 · 语法与控制流
- **[11 控制流与推导式](11_控制流与推导式.md)** — 真值判断、`for-else`、推导式、`match` 模式匹配（对照 Java 21 switch）。
- **[12 函数](12_函数.md)** — 可变默认参数陷阱、`*args/**kwargs`、闭包、作用域 LEGB、`global`/`nonlocal`、一等函数。
- **[13 类与对象](13_类与对象.md)** — `self`、`@property`、魔术方法/运算符重载、`super()`/MRO、`__slots__`、`enum`、ABC、"没有真正的 private"。

### 第四部分 · 进阶机制
- **[14 类型系统与注解](14_类型系统与注解.md)** — type hints、`mypy`、`dataclass`（对照 record/Lombok）、泛型、`Protocol`。
- **[15 异常与上下文管理器](15_异常与上下文管理器.md)** — `with` vs try-with-resources、EAFP、无 checked exception、异常链。
- **[16 迭代器与生成器](16_迭代器与生成器.md)** — 对照 Stream/Iterator，`yield`、惰性求值、`itertools`、`functools` 函数式工具。
- **[17 装饰器](17_装饰器.md)** — 对照注解 + AOP，本质是高阶函数。
- **[18 模块包与导入](18_模块包与导入.md)** — import 机制、循环导入、`__init__.py`、`if __name__ == "__main__"`。
- **[19 描述符与元类](19_描述符与元类.md)** — ⭐**完整讲透**：`@property` 底层、ORM/pydantic 的魔法、元类与类创建，对照 JVM 反射/字节码增强/注解处理器。

### 第五部分 · 并发与异步
- **[20 并发与 GIL](20_并发与GIL.md)** — threading、multiprocessing、`concurrent.futures`、GIL 实战影响、free-threaded Python。对照 Java 线程。
- **[21 异步编程 asyncio](21_异步编程asyncio.md)** — ⭐async/await、事件循环源码级机制、协程 vs 线程、`contextvars`。对照 Java 虚拟线程/Reactor。后端命脉。

### 第六部分 · IO、网络与系统
- **[22 文件 IO 与系统交互](22_文件IO与系统交互.md)** — ⭐`open`/`pathlib`/`os`/`shutil`/`subprocess`、流与缓冲。对照 `java.nio.file`/`ProcessBuilder`。
- **[23 网络编程](23_网络编程.md)** — ⭐socket → TCP/UDP → HTTP → WSGI/ASGI 全栈底座。对照 Socket/Netty/Servlet。
- **[24 HTTP 与 Web 框架](24_HTTP与Web框架.md)** — ⭐requests/httpx 客户端、FastAPI/Django/Flask 服务端。对照 OkHttp/Spring MVC。

### 第七部分 · 数据持久化与生态
- **[25 数据库与 ORM](25_数据库与ORM.md)** — ⭐SQLAlchemy、psycopg/asyncpg、连接池、工作单元模式、Alembic 迁移。对照 JDBC/Hibernate/Flyway。
- **[26 日期时间·序列化·pydantic](26_日期时间序列化pydantic.md)** — datetime/时区、json/pickle、pydantic（FastAPI 核心）。对照 java.time/Jackson。
- **[27 标准库与数据·AI 生态](27_标准库与数据AI生态.md)** — 标准库地图 + numpy/pandas/PyTorch 生态导览。

### 第八部分 · 工程化与交付
- **[28 测试](28_测试.md)** — ⭐pytest、fixture、mock、parametrize、coverage、hypothesis。对照 JUnit/Mockito。
- **[29 工程化最佳实践](29_工程化最佳实践.md)** — ruff/black、打包发布、配置与密钥管理、CLI（typer/click）、logging、安全/加密。
- **[30 部署与可观测性](30_部署与可观测性.md)** — ⭐uvicorn/gunicorn、Docker、结构化日志、metrics、OpenTelemetry trace。对照 Tomcat/Spring Boot/Micrometer。

### 第九部分 · 巩固
- **[31 易错点专题：Java 思维翻车合集](31_易错点专题.md)** — 全书坑点汇总速查表。
- **[32 综合实战](32_综合实战.md)** — 串联多主题的后端 API + 数据管道实战项目。

---

## Java → Python 极速对照表（贴墙速查）

| 概念 | Java | Python |
|------|------|--------|
| 类型系统 | 静态强类型，编译期检查 | **动态**强类型，运行期检查（可选 type hints） |
| 相等比较 | `equals()` 比值，`==` 比引用 | **`==` 比值（调 `__eq__`）**，`is` 比引用 |
| null | `null` | `None`（单例，用 `is None` 判断） |
| 变量声明 | `int x = 1;` | `x = 1`（无类型声明，名字绑定到对象） |
| 集合 | `List<String>`, `Map<K,V>` | `list`, `dict`（无泛型擦除问题，运行期不检查元素类型） |
| 接口 | `interface` + `implements` | 鸭子类型 / `Protocol`（结构化子类型） |
| 私有成员 | `private` 强制 | `_name`（约定）、`__name`（名称改写，非强制） |
| 不可变 | `final` | 看对象类型：`tuple/str/int` 不可变，`list/dict` 可变 |
| 常量 | `static final` | 全大写约定 `MAX_SIZE = 100`（无强制） |
| 包管理 | Maven/Gradle + `pom.xml` | pip/uv + `pyproject.toml` |
| 依赖隔离 | 每项目独立 classpath | **虚拟环境**（venv）每项目独立解释器环境 |
| 入口 | `public static void main` | `if __name__ == "__main__":` |
| 三元 | `cond ? a : b` | `a if cond else b` |
| Stream | `stream().map().filter()` | 推导式 / 生成器 / `itertools` |
| 异常风格 | LBYL（先检查再做） | **EAFP**（先做，错了再抓异常） |
| 并发并行 | 线程真并行 | **GIL：多线程不能 CPU 并行**，靠多进程或 asyncio |
| 构建对象 | `new Foo()` | `Foo()`（无 `new` 关键字） |
| 重载 | 方法重载（按签名） | **无重载**，靠默认参数 / `*args` / 单分派 |
| 字符串拼接 | `"a" + b` | `f"a{b}"`（f-string，首选） |

---

## 环境约定

- 本手册示例针对 **Python 3.12+**（你机器上是 3.9.6，第 02 章会教你升级）。
- 涉及 3.10/3.11/3.12 新增语法处会标注最低版本。
- 所有示例都可直接运行；建议边读边在 REPL 或 `.py` 文件里敲。
