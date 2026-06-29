# 24 · HTTP 客户端与 Web 框架 ⭐

> 这一章是你从"会写 Python 语法"跨到"用 Python 做后端"的分水岭。你在 Java 世界里有一整套肌肉记忆：发 HTTP 用 `RestTemplate`/`OkHttp`/`HttpClient`，写服务用 `@RestController` + Spring Boot，校验用 `@Valid` + Jackson，依赖注入靠 `@Autowired`。换到 Python，对应物分别是 **requests/httpx**（客户端）和 **FastAPI/Django/Flask**（服务端三巨头）。形似的地方一带而过，真正要花篇幅的是两个"形似神不同"的硬核点：① **httpx 的 Session/连接池**和 OkHttp 连接池机制几乎一样，但超时和复用的默认行为坑得多；② **FastAPI 看起来是"注解驱动"的 Spring WebFlux，其实底层是装饰器立即注册 + 读取类型注解交给 pydantic 校验 + Depends 函数式依赖注入 + 跑在 ASGI 上**——这条链路把前面 [14 类型系统与注解](14_类型系统与注解.md)、[17 装饰器](17_装饰器.md)、[19 描述符与元类](19_描述符与元类.md)、[21 异步编程asyncio](21_异步编程asyncio.md)、[23 网络编程](23_网络编程.md) 全部串了起来。看懂它，你就懂了"现代 Python Web 框架为什么长这样"。

---

## ① Java 对照表

| 维度 / 概念 | Java 生态 | Python 生态 | 关键差异 |
|------------|-----------|-------------|---------|
| 同步 HTTP 客户端 | `RestTemplate` / Apache `HttpClient` | `requests` | requests "为人类设计"，API 极简 |
| 现代 HTTP 客户端（同步+异步+HTTP/2） | `java.net.http.HttpClient`(11+) / `OkHttp` | `httpx` | httpx = requests 的 API + async + HTTP/2 |
| 连接池 / 复用 | `OkHttpClient` 单例、`PoolingHttpClientConnectionManager` | `requests.Session` / `httpx.Client` | 都靠 keep-alive 复用 TCP，**Python 默认不复用要显式建 Session** |
| 现代异步 Web 框架 | Spring WebFlux（响应式 + 注解） | **FastAPI** | 类型注解驱动校验 + 自动 OpenAPI 文档 |
| 全家桶 Web 框架 | **Spring Boot**（全家桶 + JPA） | **Django**（ORM + Admin + Auth + 模板） | 都是"batteries included" |
| 轻量 Web 框架 | Spring MVC 精简 / Spark Java | **Flask**（微核 + 插件） | 微框架，自己拼组件 |
| 路由声明 | `@GetMapping("/x")` 注解 | `@app.get("/x")` **装饰器** | 注解=元数据待扫描；装饰器=导入时**立即注册** |
| 请求体校验 | `@Valid` + Bean Validation(JSR-380) + Jackson | **pydantic** `BaseModel` | 类型注解即 schema，运行时校验 + 序列化 |
| 依赖注入 | `@Autowired` / `@Inject`（容器扫描装配） | `Depends(...)`（**函数式**，按需调用） | Spring 启动时建容器；FastAPI 每请求按需求值 |
| 中间件 / 拦截 | `Filter` / `HandlerInterceptor` / AOP | 中间件（middleware）/ 依赖 | Filter 链 ↔ 中间件链（洋葱模型） |
| 运行时容器 | Tomcat / Netty（内嵌） | **uvicorn / gunicorn**（ASGI/WSGI server） | 框架不含 server，分开部署 |
| 服务端协议规范 | Servlet API（同步）/ Servlet 3.1 async | **WSGI**（同步）/ **ASGI**（异步） | 见 [23 网络编程](23_网络编程.md) |
| 路径变量 / 查询参数 / 请求体 | `@PathVariable` / `@RequestParam` / `@RequestBody` | 函数参数 + 类型注解自动区分 | FastAPI 靠"参数类型 + 默认值"推断归属 |

> 一句话定调：在 Java 里 HTTP 客户端和 Web 框架是两套庞大的、显式配置的体系；在 Python 里它们追求**最小心智负担**——客户端 `requests.get(url)` 一行能跑，服务端 FastAPI 用你**本来就要写的类型注解**顺手做了校验和文档。Python 把"约定"和"类型"用到了极致，省掉了 Spring 那一大堆 XML/注解配置。

---

## ② 设计理念（为什么 Python 这么设计）

### 2.1 requests：「为人类设计的 HTTP」（HTTP for Humans）

Python 标准库本来就有 HTTP 客户端 `urllib.request`，但它的 API 是出了名的反人类——发一个带 header 的 POST 要建 `Request` 对象、`opener`、`handler`，编码 body、catch 一堆异常。这就像 Java 里直接用 `HttpURLConnection`：能用，但没人愿意用。

Kenneth Reitz 在 2011 年写了 `requests`，口号就是 **"HTTP for Humans"**。它的设计哲学是把"人类最常做的事"变成一行：

```python
import requests
r = requests.get("https://httpbin.org/get", params={"q": "python"}, timeout=5)
print(r.status_code)   # 200
print(r.json())        # 自动解析 JSON 成 dict
```

对照 Java 的 `RestTemplate`，理念是一致的（都在原始 API 上包一层人性化封装），但 requests 更激进：状态码、编码、JSON、cookie、重定向、连接池**全部有合理默认值**，你只在需要时覆盖。这正是 [01 设计哲学与心智模型](01_设计哲学与心智模型.md) 里"简单优于复杂"的体现。代价是 requests **只支持同步**——这在 asyncio 时代成了硬伤，于是有了 httpx。

### 2.2 FastAPI：「类型即契约」（Type Hints as a Contract）

FastAPI（2018，作者 Sebastián Ramírez）抓住了一个别人没用好的东西：**Python 的类型注解本来是"只给工具看、运行时不强制"的**（[14 类型系统与注解](14_类型系统与注解.md) 讲过）。FastAPI 把它升级成**运行时契约**——你写：

```python
@app.post("/orders")
async def create_order(order: OrderIn) -> OrderOut:
    ...
```

这一个签名同时做了五件事：① 声明请求体的形状（`OrderIn`）；② 运行时**自动校验**进来的 JSON 是否符合（不符合自动返回 422）；③ 声明响应形状（`OrderOut`）并据此序列化、过滤多余字段；④ 自动生成 **OpenAPI/Swagger 文档**；⑤ 给 IDE/mypy 提供静态检查。

在 Spring 里，这些要靠 `@RequestBody @Valid OrderIn`（Jackson 反序列化 + Bean Validation 校验）+ `@ResponseBody`（Jackson 序列化）+ springdoc（生成文档）多套机制拼起来。FastAPI 的洞见是：**这些信息其实都能从同一个类型注解推导出来，何必让你写三遍？** 这就是"类型即契约（DRY 到极致）"——一处声明，校验、序列化、文档、静态检查全有了。

### 2.3 三巨头的取舍：batteries included vs 微核 vs 现代异步

| 框架 | 哲学 | 类比 | 适合 |
|------|------|------|------|
| **Django** | "batteries included"——ORM、Admin 后台、Auth、模板、表单、迁移全自带 | Spring Boot 全家桶 | 内容驱动的大型站点、后台系统、团队规范统一 |
| **Flask** | "微核 + 插件"——核心只有路由和请求/响应，其余自己挑库 | Spring MVC 精简版 / 自己拼装 | 小服务、需要完全掌控技术栈、渐进式 |
| **FastAPI** | "现代异步 + 类型驱动"——async 原生、pydantic 校验、自动文档 | Spring WebFlux（响应式 + 注解） | 高并发 API、对接前端/移动端、AI 模型服务 |

这三种哲学的张力和 Java 世界一模一样：Django 像 Spring Boot 那样"约定大于配置、给你铺好路"，上手即生产但灵活性受框架约束；Flask 像"裸 Spring MVC + 你自己选 Hibernate/MyBatis/Jackson"，自由但要操心更多；FastAPI 是后来者，专攻"类型安全的异步 API"这个现代场景。**没有最好，只有最匹配**——选型决策见 §4.6。

---

## ③ 实现细节（底层怎么实现的）★本章重点★

### 3.1 Session 与连接池：keep-alive 复用 TCP（对照 OkHttp 连接池）

这是客户端部分最值钱的底层知识。先看一个 Java 老手最容易踩的对比：

```python
import requests

# 写法 A：每次 requests.get 都新建一个临时 Session → 每次都重新握手
for i in range(100):
    requests.get("https://httpbin.org/get")   # ❌ 100 次 TCP 握手 + 100 次 TLS 握手

# 写法 B：复用同一个 Session → 连接池复用，握手一次
with requests.Session() as session:
    for i in range(100):
        session.get("https://httpbin.org/get")  # ✓ keep-alive 复用底层 TCP 连接
```

**底层发生了什么。** requests 的连接池由它依赖的 `urllib3` 实现。一个 `Session` 内部持有一个 `PoolManager`，按 `(scheme, host, port)` 为 key 维护一组 `HTTPConnectionPool`，每个 pool 是一个连接队列。请求完成后，只要响应带 `Connection: keep-alive`（HTTP/1.1 默认），底层 TCP socket **不关闭，放回池里**，下次同主机请求直接取出复用，省掉：

- **TCP 三次握手**（1 个 RTT）
- **TLS 握手**（HTTPS 下 1~2 个 RTT，还有证书验证、密钥协商的 CPU 开销）

对高频调用同一服务的场景，复用连接能把延迟和 CPU 砍掉一大截。这和 **OkHttp 的 `ConnectionPool`** 是同一个机制：OkHttp 默认全局连接池，`OkHttpClient` 应当单例复用；requests/httpx 则要你**显式建 `Session`/`Client` 并复用**。

> 关键差异（Java 翻车点预告）：在 Java 里你早就知道"`OkHttpClient`/`RestTemplate` 要做成单例 bean"，不会每次 new。但 Python 里 `requests.get()` 这种"模块级函数"用着太顺手，新手往往在循环里直接调，**每次都隐式新建一次性 Session**，连接池形同虚设。规模一上来，连接耗尽、延迟飙升。

连接池的容量参数（urllib3 层）：

```python
import requests
from requests.adapters import HTTPAdapter

session = requests.Session()
adapter = HTTPAdapter(
    pool_connections=10,   # 缓存多少个不同 host 的连接池
    pool_maxsize=50,       # 单个 host 池里最多保留多少连接（类比 OkHttp maxIdleConnections）
    max_retries=3,         # 见 3.2
)
session.mount("https://", adapter)
session.mount("http://", adapter)
```

httpx 的 `Client`/`AsyncClient` 同理，连接池由 `httpcore` 实现，参数用 `httpx.Limits`：

```python
import httpx
limits = httpx.Limits(max_connections=100, max_keepalive_connections=20)
client = httpx.Client(limits=limits, timeout=10.0)
```

### 3.2 超时与重试：requests 默认「永不超时」是头号陷阱

requests/httpx 的 `timeout` **默认是 `None`，即无限等待**。这意味着如果对端 TCP 连上了但迟迟不返回数据（半死的服务、网络黑洞），你的线程/协程会**永远挂起**。在 Java 里 `RestTemplate` 不配 `setConnectTimeout`/`setReadTimeout` 也是同样的雷，但 OkHttp 默认有 10 秒读超时，相对安全。Python 这边**没有任何默认超时，必须手动设**。

`timeout` 实际包含两段（httpx 分得更细）：

| 阶段 | requests | httpx |
|------|----------|-------|
| connect（建立连接） | `timeout=(connect, read)` 元组的第 1 项 | `httpx.Timeout(connect=...)` |
| read（等待响应数据） | 元组第 2 项 | `read=...` |
| write / pool（等池里腾出连接） | 不细分 | `write=...` / `pool=...` |

```python
# requests：分别设连接超时和读超时
requests.get(url, timeout=(3.05, 10))   # 连接 3.05s，读 10s

# httpx：更细粒度
client = httpx.Client(timeout=httpx.Timeout(10.0, connect=5.0))
```

**重试**在 requests 里通过 urllib3 的 `Retry` 配置，能做指数退避、限定状态码：

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=3,
    backoff_factor=0.5,                      # 退避：0.5, 1.0, 2.0 秒（指数）
    status_forcelist=[500, 502, 503, 504],   # 只对这些状态码重试
    allowed_methods=["GET", "POST"],
)
session = requests.Session()
session.mount("https://", HTTPAdapter(max_retries=retry))
# 之后 session.get(...) 自动按策略重试，类比 OkHttp 的 retryOnConnectionFailure + 拦截器
```

httpx 内置重试较弱，生产中常配合 `tenacity`（呼应 [17 装饰器](17_装饰器.md) 末尾讲的 retry 装饰器骨架）或自己用 transport 实现。

### 3.3 FastAPI 是怎么工作的 ★全章最硬核★

这一节把前面五章串成一条链。我们追踪一个请求 `POST /items` 从启动到响应的完整路径。

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel, Field

app = FastAPI()

class ItemIn(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    price: float = Field(gt=0)
    tags: list[str] = []

def get_db():                       # 一个"依赖"
    db = {"connected": True}
    try:
        yield db
    finally:
        pass                        # 这里会关闭连接

@app.post("/items")
async def create_item(item: ItemIn, db=Depends(get_db)):
    return {"saved": item.name, "db": db["connected"]}
```

**第 1 步：装饰器在导入时立即注册路由（呼应 17 章）。**
`@app.post("/items")` **不是注解，是带参数的装饰器**。`app.post("/items")` 先返回一个真正的装饰器，它在模块被 import、执行到 `def create_item` 那一刻**当场**把这个函数连同它的路径、方法、参数签名登记进 `app.router.routes` 这张路由表里，然后原样返回函数。这与 Spring 的 `@PostMapping`（注解=元数据，等容器启动反射扫描）机制相反——FastAPI **没有反射扫描阶段，导入即注册**。所以你必须确保含路由的模块被 import 到，路由才存在。

**第 2 步：启动时读取类型注解，为每个路由「编译」出参数解析方案（呼应 14/19 章）。**
注册路由时，FastAPI 用 `inspect.signature(create_item)` 拿到函数签名，**逐个参数读取它的类型注解和默认值**，据此判断每个参数的"来源"：

| 参数 | 注解/默认值 | FastAPI 推断的来源 |
|------|------------|-------------------|
| `item: ItemIn` | 类型是 pydantic `BaseModel` 子类 | **请求体**（JSON body） |
| `db=Depends(get_db)` | 默认值是 `Depends(...)` | **依赖注入**（调 `get_db`） |
| `uid: int`（若有，且在 path 里） | 基本类型 + 路径里有 `{uid}` | **路径参数** |
| `q: str = "x"`（若有，基本类型，不在 path） | 基本类型 + 有默认值 | **查询参数** |

这套"看类型注解决定行为"的玩法，正是 [14 类型系统与注解](14_类型系统与注解.md) 里说的"注解存在 `__annotations__` 里、运行时可被读取"的实战；而 `ItemIn` 这个 pydantic 模型类本身，是由 pydantic 的**元类在类创建时读取注解、把每个字段编译成校验 schema**（[19 描述符与元类](19_描述符与元类.md) §4.5 讲过的"字段是描述符、模型由元类构造"）。

**第 3 步：请求进来，pydantic 解析校验请求体。**
ASGI server（uvicorn）把 HTTP 请求交给 FastAPI，FastAPI 读出 body（JSON bytes），交给 `ItemIn` 做 `model_validate`。pydantic 逐字段校验：`name` 长度 1~50、`price` 必须 > 0、`tags` 必须是 `list[str]`。

- 全部通过 → 得到一个类型安全的 `ItemIn` 实例，注入参数 `item`。
- 任一不通过 → FastAPI **自动返回 422 Unprocessable Entity**，body 是结构化的错误详情，**你的函数体根本不会执行**。

对照 Spring：这等价于 `@RequestBody @Valid` 触发 Jackson 反序列化 + Hibernate Validator 校验，校验失败抛 `MethodArgumentNotValidException`。差别是 FastAPI 把这套写进了类型系统，**零额外注解**。

**第 4 步：依赖注入用 `Depends` 求值（呼应 Spring DI，但机制不同）。**
`db=Depends(get_db)` 告诉 FastAPI：调用 `create_item` 前，**先调用 `get_db()`**，把返回值作为 `db` 传进来。`get_db` 是个生成器函数（`yield`），FastAPI 会：进入时执行到 `yield` 取出 `db`，请求结束后执行 `yield` 之后的清理代码（关连接）——这正是 [15 异常与上下文管理器](15_异常与上下文管理器.md) 的 `yield` 式资源管理。

和 Spring DI 的关键区别：

| | Spring `@Autowired` | FastAPI `Depends` |
|---|---|---|
| 装配时机 | 应用启动时建好容器、注入单例 bean | **每个请求**按需调用依赖函数 |
| 本质 | 容器管理对象图，反射注入字段 | **函数调用**，依赖就是普通函数/可调用对象 |
| 作用域 | singleton/prototype/request 等 | 默认每请求一次（可缓存、可 `yield` 带清理） |
| 嵌套 | bean 依赖 bean | 依赖里可再 `Depends` 别的依赖，FastAPI 递归求值并去重 |

FastAPI 的 DI 是**函数式、显式、请求级**的，没有"容器"这个重型概念——依赖就是函数，依赖图就是函数调用图。这比 Spring 的反射容器轻得多，也更好测试（直接传 mock 函数）。

**第 5 步：底层跑在 ASGI / uvicorn 上（呼应 21/23 章）。**
FastAPI 自己**不监听端口**，它是一个 ASGI 应用（一个 `async def __call__(self, scope, receive, send)` 的可调用对象，见 [23 网络编程](23_网络编程.md) 的 ASGI 协议）。真正监听 socket、解析 HTTP、跑事件循环的是 **uvicorn**（基于 [21 异步编程asyncio](21_异步编程asyncio.md) 的 asyncio + uvloop）。请求流：

```
client → uvicorn(监听socket, HTTP解析, asyncio事件循环)
       → ASGI scope/receive/send
       → Starlette(FastAPI 的底层路由/中间件框架)
       → 路由匹配 → 解析参数 → pydantic 校验 → 求值 Depends
       → await create_item(...)      ← 你的协程在事件循环里跑
       → 返回值 → pydantic 序列化成 JSON → ASGI send → uvicorn → client
```

因为是 async，**单个 uvicorn worker（单线程事件循环）能并发处理成千上万个 I/O 等待中的请求**——只要你的处理函数是 `async def` 且用 `await` 做 I/O（数据库、调外部 API），事件循环就能在等待时切去处理别的请求。这就是 FastAPI 高并发的来源，和 Spring WebFlux 的 Reactor 事件循环异曲同工。**但有个致命前提**：你函数体里**不能有阻塞调用**（见 §3.5、⑤）。

### 3.4 Django 的工作机制：WSGI + 中间件链 + ORM

Django 是**同步**框架（4.0+ 起逐步支持 async view，但生态主体仍同步），底层是 **WSGI**（[23 网络编程](23_网络编程.md)），由 gunicorn/uWSGI 跑。一个请求的路径：

```
client → gunicorn(WSGI server, 多 worker 进程/线程)
       → WSGI app(environ, start_response)
       → 中间件链(洋葱模型，见下) → URLconf 路由匹配
       → View 函数/类 → ORM 查库 → 模板渲染/序列化 → Response
       → 反向穿过中间件链 → WSGI server → client
```

**中间件链 = Filter 链的洋葱模型。** Django 的 `MIDDLEWARE` 是一个有序列表，每个中间件像 Spring 的 `Filter`/`HandlerInterceptor` 一样包住内层，请求**自上而下**进、响应**自下而上**出：

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",      # 最外层
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    # ... 你的自定义中间件
]
```

这就是 [17 装饰器](17_装饰器.md) §3.5 讲的"装饰器堆叠的洋葱模型"在框架层的放大版——外层中间件先碰到请求、最后处理响应。顺序有语义：认证中间件必须在用它的中间件之前。

**ORM 是 Django 的灵魂**（详见 [25 数据库与ORM](25_数据库与ORM.md)）。`models.Model` 由元类 `ModelBase` 构造（[19 描述符与元类](19_描述符与元类.md) §4.5），字段是描述符。它对标 JPA/Hibernate，自带迁移（`makemigrations`/`migrate`）、查询构造器（`User.objects.filter(...)`）、关系映射。同样有 **N+1 查询陷阱**（见 ⑤），解法是 `select_related`（JOIN）/`prefetch_related`（额外查询批量取），对应 JPA 的 `fetch join`/`@EntityGraph`。

### 3.5 同步框架 vs 异步框架：并发模型的根本差异（呼应 20/21 章）

这是选型和踩坑的理论基础，必须讲透。

| | 同步（Django/Flask + WSGI） | 异步（FastAPI + ASGI） |
|---|---|---|
| 并发单位 | **线程/进程**（每请求占一个 worker） | **协程**（单线程事件循环里成千上万个） |
| 阻塞 I/O 时 | 该 worker **整个挂起**，但 OS 调度别的线程 | `await` 让出，事件循环切去处理别的请求 |
| 并发上限 | 受 worker 数限制（线程开销大，几十~几百） | 受内存/连接限制（协程极轻，上万） |
| CPU 密集任务 | 多进程能真并行（绕过 GIL） | **会卡死事件循环**，必须丢线程池/进程池 |
| 心智模型 | 像 Spring MVC（Servlet 线程池） | 像 Spring WebFlux（Reactor 事件循环） |
| GIL 影响 | I/O 时释放 GIL，多线程对 I/O 有效 | 单线程，本就不靠多线程 |

关键认知（[20 并发与GIL](20_并发与GIL.md)、[21 异步编程asyncio](21_异步编程asyncio.md)）：

- **同步框架**靠"多 worker + 每 worker 阻塞"扛并发，简单直接，I/O 等待时该 worker 闲置但不影响别的 worker。瓶颈是 worker 数（线程/进程开销）。
- **异步框架**靠"单线程事件循环 + 协程协作"，I/O 等待时 `await` 让出控制权，一个线程喂饱海量并发连接。**但这建立在"绝不阻塞事件循环"之上**——任何同步阻塞调用（同步的 `requests.get`、`time.sleep`、CPU 密集计算）都会**冻结整个事件循环**，所有并发请求一起卡住。这就是为什么"在 FastAPI 里用同步 requests"是头号事故（见 ⑤），正解是用 `httpx.AsyncClient` + `await`。

---

## ④ 使用方法与最佳实践

> 以下示例需安装：`pip install requests httpx "fastapi[standard]" uvicorn`。公共测试 API 用 `https://httpbin.org`（回显请求）。

### 4.1 requests：发 GET / POST（最常用形态）

```python
import requests

# GET + 查询参数 + 超时（永远设 timeout！）
r = requests.get(
    "https://httpbin.org/get",
    params={"page": 1, "size": 20},          # → ?page=1&size=20
    headers={"Authorization": "Bearer xxx"},
    timeout=(3.05, 10),                       # (连接, 读)
)
r.raise_for_status()                          # 4xx/5xx 抛 HTTPError（类比 OkHttp 自己判 isSuccessful）
print(r.status_code)                          # 200
print(r.json()["args"])                       # {'page': '1', 'size': '20'}

# POST JSON（json= 自动序列化并设 Content-Type: application/json）
r = requests.post(
    "https://httpbin.org/post",
    json={"name": "Alice", "age": 30},        # 用 json= 而非 data=
    timeout=5,
)
print(r.json()["json"])                       # {'name': 'Alice', 'age': 30}

# POST 表单（data= 是 application/x-www-form-urlencoded）
requests.post("https://httpbin.org/post", data={"k": "v"}, timeout=5)
```

要点：`json=` 自动 JSON 序列化 + 设 header；`data=` 是表单编码；`raise_for_status()` 把 4xx/5xx 变异常（默认 requests **不会**因状态码抛异常，要主动调）。

### 4.2 Session：复用连接（生产必须）

```python
import requests

# 把 Session 当成长生命周期对象（类似 OkHttpClient 单例），复用连接池 + 共享 header/cookie
session = requests.Session()
session.headers.update({"User-Agent": "my-service/1.0"})   # 所有请求共享
session.params = {}                                         # 共享默认查询参数

for page in range(1, 6):
    r = session.get("https://httpbin.org/get",
                    params={"page": page}, timeout=5)        # 复用底层 TCP 连接
    # ... 处理
session.close()   # 或用 with requests.Session() as session: 自动关
```

最佳实践：**模块级或应用级建一个 Session 长期复用**（像 Spring 里的单例 `RestTemplate` bean），不要在循环/函数里反复 `requests.get()`。

### 4.3 httpx：requests 的 API + 异步能力

httpx 的同步 API 几乎和 requests 一字不差，可平滑迁移：

```python
import httpx

# 同步，和 requests 几乎一样
with httpx.Client(timeout=10.0) as client:
    r = client.get("https://httpbin.org/get", params={"q": "x"})
    print(r.json()["args"])     # {'q': 'x'}
```

**异步客户端**——这是 httpx 存在的核心理由（requests 做不到），在 async 上下文里并发发请求：

```python
import asyncio
import httpx

async def fetch(client: httpx.AsyncClient, url: str) -> int:
    r = await client.get(url, timeout=10.0)     # await，不阻塞事件循环
    return r.status_code

async def main():
    async with httpx.AsyncClient() as client:   # 复用连接池
        # 并发发 5 个请求，总耗时 ≈ 最慢的那个，而非串行求和
        urls = ["https://httpbin.org/get"] * 5
        results = await asyncio.gather(*(fetch(client, u) for u in urls))
        print(results)          # [200, 200, 200, 200, 200]

asyncio.run(main())
```

这就是 [21 异步编程asyncio](21_异步编程asyncio.md) 的 `asyncio.gather` 并发模式 + HTTP 客户端的结合：5 个请求并发飞出，事件循环在等响应时互相穿插。在 FastAPI 的 async 端点里调外部服务，**必须**用这种 `AsyncClient` + `await`，而不是同步 requests。

### 4.4 FastAPI 最小完整应用（路由 / pydantic / 参数 / DI / 异常）

```python
# main.py  —  运行： uvicorn main:app --reload
from fastapi import FastAPI, Depends, HTTPException, Query
from pydantic import BaseModel, Field

app = FastAPI(title="Item API")

# ---- pydantic 模型：请求体 / 响应体的契约 ----
class ItemIn(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    price: float = Field(gt=0, description="必须为正")
    tags: list[str] = []

class ItemOut(ItemIn):
    id: int

# ---- 假装的"数据库" + 依赖 ----
_DB: dict[int, dict] = {}
_seq = 0

def get_db() -> dict[int, dict]:
    return _DB                       # 真实场景这里 yield 一个连接，finally 关闭

# ---- 路由 ----
@app.get("/items/{item_id}")                       # 路径参数 item_id
async def read_item(
    item_id: int,                                  # 路径参数：类型在 path 里 → URL 取
    verbose: bool = Query(False),                  # 查询参数：有默认值、基本类型 → ?verbose=true
    db: dict = Depends(get_db),                    # 依赖注入
) -> ItemOut:
    if item_id not in db:
        raise HTTPException(status_code=404, detail="item not found")   # 自动转 404 JSON
    data = db[item_id]
    return data if not verbose else {**data, "tags": data["tags"]}

@app.post("/items", status_code=201)
async def create_item(item: ItemIn, db: dict = Depends(get_db)) -> ItemOut:
    global _seq
    _seq += 1
    record = {"id": _seq, **item.model_dump()}     # pydantic v2: model_dump()
    db[_seq] = record
    return record                                  # 自动按 ItemOut 序列化

# 启动后访问 http://127.0.0.1:8000/docs 自动有交互式 Swagger 文档
```

参数归属的判定规则（FastAPI 的核心约定，对照 Spring 的 `@PathVariable`/`@RequestParam`/`@RequestBody`）：

| 想要 | 写法 | Spring 对应 |
|------|------|------------|
| 路径参数 | 名字出现在路径 `{item_id}` 里 + 函数参数同名 | `@PathVariable` |
| 查询参数 | 基本类型（`int/str/bool`），**不在路径里**，通常给默认值 | `@RequestParam` |
| 请求体 | 类型是 pydantic `BaseModel` 子类 | `@RequestBody @Valid` |
| 依赖注入 | 默认值是 `Depends(...)` | `@Autowired` |

混淆这四者是新手最常见的困惑（见 ⑤）。记忆法：**基本类型→在路径里就是 path、否则是 query；BaseModel→body；Depends→注入**。

测试一下校验：向 `/items` POST `{"name": "", "price": -1}`，pydantic 会因 `name` 空、`price<=0` 自动返回 **422**，body 列出每个字段的错误，**你的函数体不执行**。

### 4.5 中间件与依赖（横切关注点）

FastAPI 中间件（对标 Spring `Filter`/`Interceptor`，洋葱模型）：

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)          # 调用内层（路由处理）
    response.headers["X-Process-Time"] = f"{time.perf_counter() - start:.4f}"
    return response
```

但 FastAPI 更推荐用**依赖**做横切（权限、限流、取当前用户），因为依赖能拿到解析后的参数、能 `yield` 清理、能复用：

```python
from fastapi import Header

async def require_token(authorization: str = Header(...)):
    if authorization != "Bearer secret":
        raise HTTPException(401, "invalid token")
    return authorization.removeprefix("Bearer ")

@app.get("/admin", dependencies=[Depends(require_token)])   # 路由级依赖，相当于权限拦截器
async def admin_panel():
    return {"ok": True}
```

### 4.6 选型：FastAPI vs Django vs Flask

| 需求场景 | 首选 | 理由 |
|---------|------|------|
| 高并发 API / 对接前端/移动端 / AI 模型服务 | **FastAPI** | async 高并发、类型校验、自动文档、轻 |
| 大型内容站 / 后台管理 / 团队要统一规范 | **Django** | 自带 Admin、ORM、Auth、迁移，开箱即用 |
| 小服务 / 想自己掌控每个组件 / 渐进式 | **Flask** | 微核，自由组合，学习曲线平缓 |
| 已重度用 SQLAlchemy + 想要 async | FastAPI（+ SQLAlchemy 2.0 async） | 二者配合好 |
| 需要服务端渲染模板的传统 Web | Django / Flask | 模板生态成熟（FastAPI 偏 API） |

经验法则：**做"返回 JSON 的 API 后端"——FastAPI 几乎是当下默认选择**；要"全功能网站含后台"——Django；要"极简或特殊定制"——Flask。这正对应你在 Java 里"WebFlux 做响应式 API / Spring Boot 做全栈 / 裸 Spring MVC 做轻量"的取舍。

### 4.7 用 uvicorn 跑（开发与生产）

```bash
# 开发：单进程 + 自动重载
uvicorn main:app --reload --host 127.0.0.1 --port 8000

# 生产：多 worker（用 gunicorn 管理 uvicorn worker）
gunicorn main:app -k uvicorn.workers.UvicornWorker -w 4 --bind 0.0.0.0:8000
#   -w 4 = 4 个进程（绕过 GIL 用满多核），每个进程内是单线程 asyncio 事件循环
```

心智模型：**uvicorn = Tomcat/Netty 的角色（HTTP server + 协议解析 + 事件循环）**，FastAPI 应用挂在它上面。Django/Flask 同理用 gunicorn（WSGI worker）。worker 数 ≈ CPU 核数用来吃满多核（绕过 GIL），每个 async worker 内部靠事件循环扛单进程内的高并发。详见 [30 部署与可观测性](30_部署与可观测性.md)。

---

## ⑤ ⚠️ 易错点（Java 思维翻车）

| Java 习惯 | 翻车现场 | 正确做法 / 认知 |
|----------|---------|----------------|
| 以为客户端默认有读超时（OkHttp 默认 10s） | requests/httpx **默认 `timeout=None` 永不超时**，对端半死时线程/协程永久挂起，连接池耗尽，服务雪崩 | **每个请求都显式设 `timeout`**，区分 connect/read；生产用 Session 级默认超时 |
| 像随手 new 一样在循环里 `requests.get()` | 每次隐式新建一次性 Session，**连接池失效**，每请求重新 TCP+TLS 握手，延迟与 CPU 暴涨 | 建一个长生命周期 `requests.Session()`/`httpx.Client()` 复用（类比 `OkHttpClient` 单例 bean） |
| 在 async 框架里用同步 `requests`/`time.sleep`/重 CPU 计算 | 同步阻塞调用**冻结整个事件循环**，所有并发请求一起卡死——比 Spring MVC 还糟 | async 端点里 I/O 用 `await httpx.AsyncClient()`；不得不跑阻塞代码用 `await run_in_executor`/`asyncio.to_thread`（[21 异步编程asyncio](21_异步编程asyncio.md)） |
| 把 `@app.get` 当 `@GetMapping` 注解理解 | 以为有"扫描阶段"，结果含路由的模块没被 import，路由神秘消失 | 它是**装饰器，导入即注册**（[17 装饰器](17_装饰器.md)）；确保路由模块被加载 |
| 混淆 path / query / body 参数 | 想要请求体却写成基本类型（变 query 参数）；或基本类型想当 body | 记规则：**基本类型在 path→path，否则→query；pydantic 模型→body；Depends→注入** |
| 以为状态码 4xx/5xx 会自动抛异常 | requests 默认**不抛**，`r.json()` 在错误响应上拿到错误体却以为成功 | 主动 `r.raise_for_status()`，或检查 `r.status_code` |
| 把 FastAPI 端点写成阻塞的 `def`（同步） | FastAPI 会把同步 `def` 丢线程池，看似没事，但里面再调阻塞 I/O 时并发能力远不如 async；或误以为 `async def` 里同步阻塞也没事 | I/O 密集端点用 `async def` + `await` 异步库；纯阻塞且无法改写的才用同步 `def`（FastAPI 自动丢线程池） |
| Django ORM 当 JPA 随便点关系属性 | 循环里 `for u in users: u.profile.x` 触发 **N+1 查询**，列表页慢成狗 | `select_related`（JOIN，1对1/外键）/ `prefetch_related`（多对多/反向，批量），对应 JPA `fetch join`（[25 数据库与ORM](25_数据库与ORM.md)） |
| pydantic 校验当成可选的"建议" | 以为像 Java 注解不强制（[14 章](14_类型系统与注解.md)），不传必填字段也想过 | FastAPI 里 pydantic **运行时强制校验**，不符合直接 422，函数体不执行——类型在这里是**真契约** |

---

## ⑥ 练习题

### 基础理解

**1.** 解释为什么下面两段代码性能差距巨大（假设循环 1000 次请求同一主机），底层连接发生了什么不同？

```python
# A
import requests
for _ in range(1000):
    requests.get("https://api.example.com/ping", timeout=5)

# B
import requests
with requests.Session() as s:
    for _ in range(1000):
        s.get("https://api.example.com/ping", timeout=5)
```

**2.** 在 FastAPI 里，下面这个端点的三个参数 `user_id`、`q`、`payload` 分别会从 HTTP 请求的哪个部分取值？FastAPI 是依据什么判断的？

```python
from pydantic import BaseModel
class Payload(BaseModel):
    title: str

@app.put("/users/{user_id}/posts")
async def f(user_id: int, payload: Payload, q: str = "default"):
    ...
```

### 找坑改错

**3.** 一位 Java 同事把 Spring WebFlux 的写法搬到 FastAPI，写了下面这个"调用上游服务的网关端点"。压测时发现 QPS 上不去、并发请求互相拖慢，几乎退化成串行。指出**根因**并改对：

```python
import requests
from fastapi import FastAPI

app = FastAPI()

@app.get("/proxy")
async def proxy():
    # 调用上游服务拿数据
    r = requests.get("https://upstream.example.com/data", timeout=10)
    return r.json()
```

**4.** 下面的 FastAPI 端点想"创建用户并校验邮箱格式、年龄非负"，但作者用 Java 思维手写校验，既啰嗦又漏洞百出（非字符串、缺字段会直接 500）。用 pydantic 重写成地道形态：

```python
from fastapi import FastAPI, HTTPException
app = FastAPI()

@app.post("/users")
async def create_user(data: dict):           # 直接收 dict
    if "email" not in data or "@" not in data["email"]:
        raise HTTPException(400, "bad email")
    if "age" not in data or data["age"] < 0:
        raise HTTPException(400, "bad age")
    return {"email": data["email"], "age": data["age"]}
```

### 综合实战

**5.** 用 FastAPI 写一个内存版的「图书」CRUD，要求：
- pydantic 模型 `BookIn`（`title` 非空且 ≤100 字符、`author` 非空、`year` 在 1450~2100 之间）和 `BookOut`（在 `BookIn` 基础上加 `id: int`）。
- 端点：`POST /books` 创建（返回 201 + `BookOut`）、`GET /books/{book_id}` 查询（不存在返回 404）、`GET /books` 列表（支持 `author` 查询参数过滤）、`DELETE /books/{book_id}`（不存在 404，成功 204）。
- 用一个 `Depends` 依赖提供"数据库"（内存 dict）。
- 写出可直接 `uvicorn` 跑的完整代码。

---

### 参考答案

**1.** 差距来自**连接复用**。A 每次调用模块级 `requests.get()`，内部**临时新建一个 `Session`、用完即弃**，于是每次请求都要：DNS（可能缓存）→ **TCP 三次握手** → **TLS 握手**（HTTPS 下证书验证 + 密钥协商，最贵）→ 发请求 → 收响应 → **关闭连接**。1000 次就是 1000 轮完整握手。B 复用同一个 `Session`，其内部 `urllib3` 连接池按 `(scheme,host,port)` 缓存 keep-alive 的 TCP 连接，第一次握手后连接放回池，后续 999 次**直接复用**已建立的 TCP/TLS 连接，省掉绝大部分握手开销，延迟和 CPU 大幅下降。这与 Java 里"`OkHttpClient` 单例复用 `ConnectionPool`"是同一道理——区别是 Python 要你**显式建 Session**。

**2.**
- `user_id` → **路径参数**：它是基本类型 `int`，且名字 `user_id` 出现在路由路径 `/users/{user_id}/posts` 里，FastAPI 从 URL 路径段取值。
- `payload` → **请求体（JSON body）**：它的类型是 pydantic `BaseModel` 子类 `Payload`，FastAPI 据此判定它是请求体，读 body 的 JSON 并用 pydantic 校验。
- `q` → **查询参数**：它是基本类型 `str`、有默认值 `"default"`、且**不在路径里**，FastAPI 判定为查询参数（`?q=...`），不传则用默认值。
- **判断依据**：FastAPI 在注册路由时用 `inspect.signature` 读取每个参数的**类型注解 + 默认值 + 是否出现在路径模板里**，按规则推断来源（基本类型且在 path→path；基本类型不在 path→query；BaseModel→body；`Depends`→依赖）。

**3.** **根因**：在 `async def` 端点里调用了**同步阻塞**的 `requests.get()`。FastAPI 的 async 端点跑在**单线程事件循环**上，`requests.get` 不会 `await` 让出，它会**阻塞整个事件循环直到上游返回**——这期间该 worker 无法处理任何其他并发请求，于是高并发退化成串行（比同步框架还糟，因为同步框架至少有多 worker/线程）。正解是改用 `httpx.AsyncClient` + `await`，让 I/O 等待时事件循环能切去服务别的请求：

```python
import httpx
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.client = httpx.AsyncClient(timeout=10.0)   # 应用级复用连接池
    yield
    await app.state.client.aclose()

app = FastAPI(lifespan=lifespan)

@app.get("/proxy")
async def proxy():
    r = await app.state.client.get("https://upstream.example.com/data")  # await，不阻塞
    r.raise_for_status()
    return r.json()
```

（若实在要调用某个只有同步版的阻塞库，用 `await asyncio.to_thread(blocking_call, ...)` 把它丢到线程池，避免冻结事件循环。）

**4.** 用 pydantic 模型把校验交给类型系统，函数体只剩业务：

```python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr, Field

app = FastAPI()

class UserIn(BaseModel):
    email: EmailStr                       # 自动校验邮箱格式（需 pip install "pydantic[email]")
    age: int = Field(ge=0)                # 非负

class UserOut(BaseModel):
    email: EmailStr
    age: int

@app.post("/users", status_code=201)
async def create_user(user: UserIn) -> UserOut:
    return user                           # 校验已由 pydantic 完成；不合法自动 422
```

改进点：① 缺字段、类型错、邮箱非法、年龄为负**全部由 pydantic 自动返回 422 + 结构化错误**，不再 500 也不用手写 `if`；② `EmailStr` 比 `"@" in email` 严谨得多；③ 类型注解即文档，`/docs` 自动展示约束；④ 这才是 [14 类型系统与注解](14_类型系统与注解.md) 说的"类型在 FastAPI 里是真契约"的体现。

**5.** 完整可运行的图书 CRUD：

```python
# books.py  —  运行： uvicorn books:app --reload
from fastapi import FastAPI, Depends, HTTPException, Query, status
from pydantic import BaseModel, Field

app = FastAPI(title="Books API")

# ---- 模型 ----
class BookIn(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    author: str = Field(min_length=1)
    year: int = Field(ge=1450, le=2100)

class BookOut(BookIn):
    id: int

# ---- "数据库" 依赖 ----
_DB: dict[int, dict] = {}
_seq = 0

def get_db() -> dict[int, dict]:
    return _DB

# ---- CRUD ----
@app.post("/books", status_code=status.HTTP_201_CREATED)
async def create_book(book: BookIn, db: dict = Depends(get_db)) -> BookOut:
    global _seq
    _seq += 1
    record = {"id": _seq, **book.model_dump()}
    db[_seq] = record
    return record

@app.get("/books/{book_id}")
async def get_book(book_id: int, db: dict = Depends(get_db)) -> BookOut:
    if book_id not in db:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "book not found")
    return db[book_id]

@app.get("/books")
async def list_books(
    author: str | None = Query(None),        # 可选查询参数，?author=...
    db: dict = Depends(get_db),
) -> list[BookOut]:
    books = list(db.values())
    if author is not None:
        books = [b for b in books if b["author"] == author]
    return books

@app.delete("/books/{book_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_book(book_id: int, db: dict = Depends(get_db)) -> None:
    if book_id not in db:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "book not found")
    del db[book_id]
    # 204 不返回 body
```

要点回顾：① `BookIn` 把所有约束（长度、年份范围）声明在类型里，pydantic 自动校验、非法 422；② `BookOut(BookIn)` 继承复用字段并加 `id`，响应按它序列化；③ `Depends(get_db)` 把"数据库"做成依赖，便于测试时替换成 mock；④ 路径参数 `book_id: int`、查询参数 `author`、请求体 `BookIn` 三种参数来源各司其职；⑤ `status_code` 显式声明 201/204，符合 REST 语义；⑥ 启动后 `/docs` 自动生成交互式文档，这是 FastAPI "类型即契约"白送的回报。生产中把内存 dict 换成真正的数据库会话（[25 数据库与ORM](25_数据库与ORM.md)），`get_db` 改成 `yield` 连接并在 `finally` 关闭。

---

下一章：[25 数据库与ORM](25_数据库与ORM.md) — DB-API、SQLAlchemy（Core/ORM、连接池、async）、Django ORM 对照 JPA/Hibernate/MyBatis，以及 N+1、事务、连接池的底层。
