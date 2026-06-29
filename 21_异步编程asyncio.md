# 21 · 异步编程 asyncio ⭐

> 你懂 Netty 的事件循环、Reactor 的背压、`CompletableFuture` 的编排、虚拟线程（Project Loom）的"廉价线程"。asyncio 想解决的是同一个问题——**一个线程内扛住成千上万个并发 I/O 连接（C10k）**——但走的是和 Loom 截然不同的路：不是"让线程变廉价让你随便阻塞"，而是"**单线程协作式调度 + 协程主动让出**"。本章把 asyncio 当成"Python 版的 Reactor + CompletableFuture"来拆，深挖事件循环源码级机制、协程对象与生成器栈帧的血缘（承接 [16 迭代器与生成器](16_迭代器与生成器.md) 的 `send`）、以及为什么一个 `time.sleep` 就能让整个服务"假死"。看懂这章，你就理解了 FastAPI、aiohttp、asyncpg 底下那台永不停转的引擎。

---

## ① Java 对照表

| 概念 | Java 世界 | Python asyncio | 关键差异 |
|------|----------|----------------|---------|
| 事件循环 | Netty `EventLoop` / Reactor 的 `Scheduler` | `asyncio` 事件循环（`SelectorEventLoop`） | 都基于 epoll/kqueue 轮询 fd；asyncio 是**进程内唯一主循环**，应用代码也跑在上面 |
| 协程 | 虚拟线程（Loom）/ `CompletableFuture` 链 | `async def` 协程对象 | 虚拟线程**被调度器抢占式挂起**；Python 协程**只能在 `await` 处主动让出** |
| 暂停等待 | `future.get()`（**阻塞**载体线程，Loom 下挂起虚拟线程） | `await awaitable`（**不阻塞**线程，让出事件循环） | `await` 形似 `.get()` 但语义相反：它把线程让给别人 |
| 任务句柄 | `Future<T>` / `CompletableFuture<T>` | `asyncio.Task` / `asyncio.Future` | Task = 被事件循环驱动的协程；Future = 结果占位符 |
| 并发聚合 | `CompletableFuture.allOf(...)` | `asyncio.gather(*coros)` | gather 收集所有结果为列表，保序 |
| 任意先到 | `CompletableFuture.anyOf` | `asyncio.wait(..., FIRST_COMPLETED)` / `as_completed` | as_completed 按**完成顺序**逐个 yield |
| 结构化并发 | `StructuredTaskScope`（Loom，预览） | `asyncio.TaskGroup`（3.11+） | 都保证"作用域退出前所有子任务收敛" |
| 超时 | `orTimeout` / `Future.get(timeout)` | `asyncio.timeout()` / `wait_for` | 超时触发**取消**，抛 `TimeoutError` |
| 取消 | `Future.cancel(true)` + `InterruptedException` | `Task.cancel()` + `CancelledError` | 取消通过在 `await` 点**注入异常**实现 |
| 线程本地 | `ThreadLocal<T>` | `contextvars.ContextVar` | ThreadLocal 在协程下**串味**；contextvars 随协程上下文走 |
| 阻塞调用兜底 | 直接在虚拟线程里调阻塞 API（Loom 替你挂起） | `await loop.run_in_executor(...)` 丢线程池 | asyncio **不会**替你把阻塞变非阻塞，必须手动外包 |
| 锁 | `synchronized` / `ReentrantLock` | `asyncio.Lock` | asyncio 锁是**协作式**、非线程安全、`async with` 获取 |
| 生态 | Netty / WebFlux / R2DBC / Reactor Netty | aiohttp / httpx / asyncpg / uvicorn | 全套"异步驱动"才能发挥威力 |

> 一句话：**Java 用"让线程变廉价（Loom）或回调链（Reactor）"来扛 I/O 并发；Python 用"单线程协作式协程"扛**。前者你几乎察觉不到调度，后者你必须亲手在每个 I/O 点写 `await`、并保证全链路不阻塞。

---

## ② 设计理念（为什么 Python 这么设计）

### 2.1 要解决的问题：C10k 与线程的成本

后端的经典痛点：一台机器要同时维持上万条 I/O 连接（长轮询、WebSocket、微服务扇出调用）。这些连接**绝大多数时间在等**——等数据库、等下游 HTTP、等 socket 可读。CPU 几乎闲着，瓶颈是"同时等待的连接数"。

传统的"一连接一线程"模型在这里很贵（这点 Java 老手最有体会，也正是 Loom 诞生的理由）：

- 每个 OS 线程有 MB 级栈内存，上万线程光栈就吃几个 GB。
- 内核线程上下文切换有成本（寄存器保存、TLB、调度器开销）。
- 大量线程争抢锁、缓存抖动。

Java 给了两条答卷：**Reactor/Netty**（少量事件循环线程 + 回调，但代码被"回调地狱"和背压污染）和 **Loom 虚拟线程**（保留同步阻塞写法，运行时把阻塞点变成挂起）。Python 的答卷是 **asyncio**：单线程（或少量线程各跑一个循环）+ 协作式协程。

### 2.2 协作式 vs 抢占式：这是和 Java 线程最根本的分野

这是全章最重要的心智模型，务必钉死：

- **Java 线程 / 虚拟线程是抢占式的**：调度器可以在**任意指令边界**挂起你的线程去跑别人。你写 `a++; b++;` 中间可能被切走。所以你需要 `synchronized`、`volatile`、`AtomicInteger` 来防数据竞争。
- **asyncio 协程是协作式的**：事件循环**只能在你 `await` 的那一刻**拿回控制权。两个 `await` 之间的代码是**原子的**——绝不会有另一个协程在中途插进来。

后果是双刃剑：

✅ **好处**：协程之间不存在"指令级数据竞争"。一段不含 `await` 的代码块就是临界区，很多时候你根本不需要锁。心智负担比多线程小一个量级。

❌ **代价**：**一旦某个协程不 `await`（比如调了阻塞的 `time.sleep` 或纯 CPU 死循环），它就霸占了整个事件循环，所有其它协程全部饿死。** 没有抢占来救你。这就是 ⑤ 里最致命的翻车点。Loom 不会这样——虚拟线程被阻塞了调度器会切走别人；asyncio 会**整个服务假死**。

```
抢占式（Java 线程/Loom）：          协作式（asyncio 协程）：
 调度器随时夺权                      只在 await 处交权
 ┌──┐ ┌──┐ ┌──┐                     ┌─────await─┐ ┌──await──┐
 │T1│↔│T2│↔│T1│  内核强制切换         │  协程A    │→│  协程B   │  协程主动让出
 └──┘ └──┘ └──┘                     └───────────┘ └─────────┘
 需要锁防竞争                         两个 await 间天然是临界区
```

### 2.3 和 GIL 的关系：async 不是为了 CPU 并行

Java 老手常把"异步"和"多核并行"混为一谈，这在 Python 里是大错。回顾 [05 CPython内部](05_CPython内部.md)：CPython 有 **GIL**，同一时刻只有一个线程执行 Python 字节码。

- **asyncio 默认是单线程的**——它根本不碰多核。它的并发来自"一个线程在多个 I/O 等待间快速切换"，而不是"多个核同时算"。
- 所以 asyncio 对 **I/O 密集**任务是利器（等待期间切去干别的），对 **CPU 密集**任务**毫无帮助**（没有真正的并行，反而把唯一的线程占满）。
- CPU 密集要并行，仍然只能靠 [20 并发与GIL](20_并发与GIL.md) 讲的多进程，或下沉到释放 GIL 的 C 扩展（numpy）。

> 记住这张分工表：**CPU 密集 → 多进程；I/O 密集且并发量大 → asyncio；I/O 密集但量不大或要复用阻塞库 → 多线程**。asyncio 与 GIL 不矛盾，因为它压根不追求并行——它在 GIL 允许的"单线程"框架内，把 I/O 等待的空隙榨干。

### 2.4 为什么是"协程"而不是"回调"

Netty/早期 Node 的回调写法（`then().then().catch()`）能实现同样的事件循环并发，但代码会碎成一地回调，控制流、异常、循环都难写。Python 选择把这套机制藏在 `async`/`await` 语法糖背后，让你**用看起来同步的顺序代码写异步逻辑**：

```python
# 回调风格（概念演示，不是 Python 惯用法）：
fetch(url, on_done=lambda data: parse(data, on_done=lambda r: save(r)))

# asyncio 风格：看起来同步，实则每个 await 都在让出循环
async def handle(url):
    data = await fetch(url)      # 让出，等 I/O，回来
    result = await parse(data)
    await save(result)
```

这正是 [16 迭代器与生成器](16_迭代器与生成器.md) 里 `yield`/`send` 埋下的种子开花结果：协程就是"能在中途暂停、把栈帧保存下来、之后再恢复"的函数。`await` 在底层就是"暂停我、把控制权还给事件循环、就绪后 `send` 回来"。Python 复用了生成器那套**栈帧冻结/解冻**机制来实现协程——这是 ③ 的核心。

---

## ③ 实现细节（底层怎么实现的）★本章重点★

### 3.1 协程对象：`async def` 产出的"可暂停对象"

先看一个 Java 老手最容易栽的事实：**调用一个 `async def` 函数，不会执行它的函数体，而是立刻返回一个协程对象。**

```python
import asyncio

async def fetch_user(uid: int) -> dict:
    print("  [函数体真正开始执行]")
    await asyncio.sleep(0.1)          # 模拟一次 I/O 等待
    return {"id": uid, "name": f"user{uid}"}

coro = fetch_user(1)                   # 注意：这里没有打印 [函数体...]！
print(type(coro))                      # 输出: <class 'coroutine'>
print(coro)                            # 输出: <coroutine object fetch_user at 0x...>

result = asyncio.run(coro)             # 直到这里，函数体才被驱动执行
print(result)                          # 先打印 [函数体真正开始执行]，再输出: {'id': 1, 'name': 'user1'}
```

这和 [16 章](16_迭代器与生成器.md) 3.3 里"调用生成器函数只造对象、函数体到第一次 `next` 才执行"是**同一个机制**。事实上，协程对象和生成器是近亲：

```python
import inspect

async def demo():
    await asyncio.sleep(0)

c = demo()
print(inspect.iscoroutine(c))          # 输出: True
print(c.cr_frame is not None)          # 输出: True —— 协程持有一个帧对象（和生成器的 gi_frame 同源）
print(inspect.getcoroutinestate(c))    # 输出: CORO_CREATED —— 还没开始跑
c.close()                              # 收尾，避免 "coroutine was never awaited" 警告
```

`async def` 编译出的 code object 带 `CO_COROUTINE` 标志（对照生成器的 `CO_GENERATOR`）。协程对象内部和生成器一样**持有一个可冻结/解冻的栈帧**（`cr_frame`、`cr_lasti`），局部变量跨越 `await` 暂停而存活。**协程 = 复用了生成器栈帧机制、但用 `await` 而非 `yield` 作为暂停点的对象。**

### 3.2 awaitable 协议：`await` 到底在等什么

`await x` 要求 `x` 是一个 **awaitable**——实现了 `__await__()` 方法、返回一个迭代器的对象。三类 awaitable：

1. **协程对象**（`async def` 的产物）——最常见。
2. **Task / Future**——事件循环的调度单元（见 3.4）。
3. 实现了 `__await__` 的自定义对象（库作者才需要）。

`await coro` 在底层做的事，本质就是 [16 章](16_迭代器与生成器.md) 3.5 的 `yield from`：把驱动权委托给被等待对象，建立一条贯通的双向管道。真正"把控制权交还事件循环"的，是管道最底层那个 Future 的 `__await__`，它会 `yield` 出自己。看一个去掉所有语法糖的最小模型：

```python
# Future.__await__ 的精神内核（极简版）：
class MiniFuture:
    def __init__(self):
        self._done = False
        self._result = None
    def __await__(self):
        if not self._done:
            yield self           # ★关键：yield 把控制权交回事件循环，并把自己交出去登记
        return self._result      # 被恢复后，返回结果
```

整条链是：你的协程 `await` 一个库协程，库协程 `await` 一个 Future，**Future 的 `__await__` 用 `yield self` 把控制权一路冒泡回事件循环**。事件循环拿到这个 Future，登记"它就绪时该恢复谁"。这就是为什么 16 章说"`send` 是协程的雏形"——事件循环驱动协程的方式，就是对最外层协程反复调用 `.send(result)`。

### 3.3 事件循环：selectors + 回调队列 + 单线程轮询

这是和 Netty `EventLoop` 最直接的对应。asyncio 的 `SelectorEventLoop` 内部就是一个 `while True` 的轮询循环，三大件：

1. **`selectors`（epoll/kqueue 的封装）**：登记"我关心哪些 fd 的哪些事件（可读/可写）"，一次系统调用阻塞等待，返回**已就绪**的 fd 集合。Linux 上是 `epoll`、macOS/BSD 上是 `kqueue`、Windows 上有 `IocpProactor`——和 Netty 选择底层多路复用器是同一套思路。
2. **就绪回调队列（`_ready`）**：所有"该立刻执行"的回调（`call_soon` 排进来的），事件循环每轮把它们挨个跑完。
3. **定时器堆（`_scheduled`，一个最小堆）**：`call_later`/`asyncio.sleep` 排进来的延时回调，按到期时间用堆排序。

事件循环一轮（`_run_once`）的伪代码，把 Netty 的 `select → processSelectedKeys → runAllTasks` 翻译成 Python：

```python
# asyncio 事件循环单轮的精神模型（简化自 CPython Lib/asyncio/base_events.py）
def _run_once(self):
    # ① 算出最近一个定时器还有多久到期，作为 select 的超时
    timeout = compute_timeout(self._scheduled)        # 没有就绪回调时才会真的阻塞

    # ② 一次系统调用：epoll/kqueue 阻塞等待，直到有 fd 就绪或超时
    event_list = self._selector.select(timeout)       # ★唯一会"阻塞"的地方

    # ③ 把就绪的 fd 对应的回调放进 _ready 队列
    for key, mask in event_list:
        self._ready.append(key.data.callback)

    # ④ 把到期的定时器回调移进 _ready
    now = self.time()
    while self._scheduled and self._scheduled[0].when <= now:
        handle = heapq.heappop(self._scheduled)
        self._ready.append(handle.callback)

    # ⑤ 依次执行 _ready 里的所有回调（这里会驱动协程往前 .send 一步）
    for _ in range(len(self._ready)):
        callback = self._ready.popleft()
        callback()                                    # ← 协程在这里被恢复，跑到下一个 await
```

关键认知：

- **整个循环是单线程的**。第 ⑤ 步执行回调（也就是恢复协程、跑你的业务代码）和第 ②③④ 步轮询 I/O **在同一个线程里轮流进行**。
- **唯一真正"阻塞等待"的是第 ② 步的 `selector.select()`**——而且它是在"没活干"时才阻塞，一旦有 fd 就绪或定时器到期立刻返回。
- 你的业务代码（第 ⑤ 步的 `callback()`）**必须很快返回到循环**（即很快遇到下一个 `await`），否则第 ②③④ 步根本没机会跑，所有 I/O 都被晾着——这就是"阻塞调用卡死事件循环"的源码级解释。

### 3.4 Task：包装协程并驱动它的 Future

协程对象本身是"死"的——它只是个能被 `send` 的栈帧，不会自己往前跑。**让协程动起来需要有人反复 `send` 它，这个"驱动者"就是 `Task`。**

`Task` 是 `Future` 的子类。它做两件事：

1. 把一个协程包起来，**在创建时就用 `loop.call_soon` 把"驱动这个协程一步"安排进事件循环**。
2. 每当协程 `await` 的东西就绪，Task 的 `__step` 方法被回调，它对协程 `.send(result)`，协程往前跑到下一个 `await`，交出下一个要等的 Future，Task 给那个 Future 注册"就绪时再回调我 `__step`"的回调。如此循环，直到协程 `return`（Task 标记完成、存结果）或抛异常（Task 存异常）。

```python
import asyncio

async def work(name, delay):
    await asyncio.sleep(delay)
    return f"{name} done"

async def main():
    # create_task：把协程包成 Task，立刻"登记"到事件循环开始驱动（并发起点！）
    t1 = asyncio.create_task(work("A", 0.2))
    t2 = asyncio.create_task(work("B", 0.1))
    print(type(t1))                    # 输出: <class '_asyncio.Task'>
    print(t1.done())                   # 输出: False —— 刚创建，还没跑完
    # await 一个 Task：等它驱动完，拿结果
    r1 = await t1
    r2 = await t2
    print(r1, r2)                      # 输出: A done B done

asyncio.run(main())
```

**协程 vs Task 的关键区别（高频考点）**：

- `await some_coro()`：**串行**——当前协程被挂起，直到 `some_coro` 整个跑完才继续。没有并发。
- `asyncio.create_task(some_coro())`：**立刻并发**——Task 一创建就被排进循环开始推进，你可以先去干别的，之后再 `await` 它收结果。

这对应 Java：`await coro` 像 `future.get()` 立刻等；`create_task` 像先 `executor.submit()` 拿到 Future、稍后再 `get()`。**并发的起点是 `create_task`（或 gather/TaskGroup），不是 `async def`。**

### 3.5 为什么一个阻塞调用就卡死整个循环

把 3.2~3.4 串起来就能精确解释这个 Java 老手最易犯的致命错误。事件循环靠"协程频繁 `await` 让出"来轮转。如果某个协程里调用了**同步阻塞**的函数：

```python
import asyncio, time

async def bad():
    print("开始")
    time.sleep(3)            # ★同步阻塞 3 秒！这 3 秒线程被死死占住
    print("结束")

async def heartbeat():
    while True:
        print("  ❤️ 心跳")
        await asyncio.sleep(0.5)

async def main():
    asyncio.create_task(heartbeat())
    await bad()
    # 运行结果：打印"开始"后，整整 3 秒一片死寂，心跳完全停摆，
    # 3 秒后才打印"结束"，然后心跳才恢复。

asyncio.run(main())
```

源码级解释：`time.sleep(3)` 是个 C 层的阻塞系统调用，它**不会 `yield` 回事件循环**。于是 3.3 伪代码里第 ⑤ 步的某个 `callback()` 进去后 3 秒不返回，循环根本走不到第 ② 步去 `select`，也走不到处理心跳定时器——**整个引擎卡死**。`heartbeat` 的 `asyncio.sleep` 定时器虽然到期了，但没人去 `_run_once` 处理它。

对比 Loom：虚拟线程里调 `Thread.sleep(3)`，Loom 运行时会把这个虚拟线程从载体线程上**卸载**，载体线程去跑别的虚拟线程——所以 Loom 下你随便阻塞都没事。**asyncio 没有这个魔法**，它无法把同步阻塞自动变成"让出"。正确做法只有两条：换成 `await asyncio.sleep`（异步版本会 `yield`），或把阻塞调用丢进线程池（3.6 的 `run_in_executor`）。

> 规则刻进 DNA：**asyncio 协程里，任何可能阻塞的操作都必须是 `await` 一个异步实现，绝不能调同步阻塞函数。** `time.sleep`→`asyncio.sleep`，`requests`→`httpx.AsyncClient`/`aiohttp`，`psycopg2`→`asyncpg`，文件大读写→丢线程池。

### 3.6 把阻塞调用外包：`run_in_executor`

现实里总有绕不开的同步库（老的 SDK、`requests`、CPU 小任务、同步文件 I/O）。asyncio 的官方逃生舱是**把它丢进一个线程池执行，并用一个 Future 在事件循环侧等待结果**——这样事件循环本身不阻塞（它只是 `await` 一个会在别的线程完成时就绪的 Future）。

```python
import asyncio, time
from concurrent.futures import ThreadPoolExecutor

def blocking_io(n):                    # 一个同步阻塞函数（假装是老 SDK）
    time.sleep(1)
    return n * n

async def main():
    loop = asyncio.get_running_loop()
    # run_in_executor(None, ...) 用默认线程池跑阻塞函数，返回一个可 await 的 Future
    results = await asyncio.gather(
        loop.run_in_executor(None, blocking_io, 2),
        loop.run_in_executor(None, blocking_io, 3),
        loop.run_in_executor(None, blocking_io, 4),
    )
    print(results)                     # 约 1 秒后输出: [4, 9, 16] —— 三个阻塞调用在线程池里并发

# 3.9+ 更推荐的等价写法：asyncio.to_thread(blocking_io, 2)
asyncio.run(main())
```

`asyncio.to_thread(func, *args)`（3.9+）是 `run_in_executor(None, ...)` 的语法糖，更易读。注意：这只是把阻塞**挪到别的线程**让事件循环喘口气，受 GIL 限制它**对 CPU 密集任务没有并行收益**——CPU 密集仍要 `ProcessPoolExecutor`（`run_in_executor` 也能接进程池）。

### 3.7 contextvars：协程世界的"ThreadLocal"

这是 Java 老手会重重踩中的暗坑。在 Spring 里你习惯用 `ThreadLocal` 存"当前请求的 traceId / 用户 / 事务上下文"，靠的是"一个请求 = 一个线程"的前提。**但在 asyncio 里，成千上万个请求（协程）跑在同一个线程上**，`ThreadLocal`（Python 的 `threading.local`）会让所有协程共享同一份数据——traceId 互相串味，灾难性的并发 bug。

正确工具是标准库 `contextvars.ContextVar`（PEP 567，3.7+）。它的语义是"**上下文随协程走，不随线程走**"：

```python
import asyncio
import contextvars

request_id = contextvars.ContextVar("request_id", default="-")

async def handler(rid):
    request_id.set(rid)                # 设置当前协程上下文的值
    await asyncio.sleep(0.1)           # 期间事件循环切去跑别的协程
    # 恢复后，读到的仍是本协程自己 set 的值，没有被别的协程污染
    print(f"处理完成, request_id={request_id.get()}")

async def main():
    # 并发跑 3 个 handler，它们交错执行但各自的 request_id 互不干扰
    await asyncio.gather(handler("req-A"), handler("req-B"), handler("req-C"))

asyncio.run(main())
# 输出（顺序可能不同，但每条 id 正确，绝不串味）:
#   处理完成, request_id=req-A
#   处理完成, request_id=req-B
#   处理完成, request_id=req-C
```

底层机制：事件循环在**切换/恢复协程时会保存和恢复 Context**。每个 Task 创建时会 `copy_context()` 复制一份当时的上下文快照，Task 运行时在自己的 Context 副本里跑——所以一个协程 `set` 的值不会泄漏到另一个协程。这正是 ThreadLocal 给不了的：ThreadLocal 绑定线程，而一个线程上交错跑着无数协程。

> 实战：日志 traceId、租户隔离、`opentelemetry` 的 span 传播、数据库会话，在异步服务里都用 `contextvars` 而非 `ThreadLocal`。框架（FastAPI 的中间件、starlette）内部正是这么传请求上下文的。

---

## ④ 使用方法与最佳实践

### 4.1 基本骨架：`async def` / `await` / `asyncio.run`

```python
import asyncio

async def main():                      # 协程函数：async def
    print("hello")
    await asyncio.sleep(1)             # await 一个异步操作，让出循环 1 秒
    print("world")

asyncio.run(main())                    # ★唯一的入口：创建事件循环、跑 main()、收尾关闭
```

- `asyncio.run()` 是程序的异步入口，负责**创建事件循环、运行顶层协程、最后关闭循环**。**一个程序通常只调一次**（在 `if __name__ == "__main__":` 里）。
- 不能在已经运行的事件循环里再调 `asyncio.run()`（会报 `RuntimeError`）——Jupyter/已在异步框架里时用 `await` 而非 `run`。

### 4.2 并发的正确姿势：gather

`asyncio.gather` 是最常用的"扇出并发"，对应 `CompletableFuture.allOf` + 收集结果：

```python
import asyncio, time

async def fetch(name, delay):
    await asyncio.sleep(delay)         # 用 sleep 模拟网络 I/O，保证示例可跑
    return f"{name}: {delay}s"

async def main():
    start = time.perf_counter()
    # gather 把多个协程并发跑，结果按传入顺序返回（保序！）
    results = await asyncio.gather(
        fetch("A", 0.3),
        fetch("B", 0.1),
        fetch("C", 0.2),
    )
    print(results)                     # 输出: ['A: 0.3s', 'B: 0.1s', 'C: 0.2s'] —— 保序
    print(f"耗时 {time.perf_counter() - start:.2f}s")  # 输出: 约 0.30s（最长那个），而非 0.6s 串行

asyncio.run(main())
```

注意 `gather` 的结果**按参数顺序**而非完成顺序排列。异常处理：默认任一子任务抛异常，`gather` 立刻把该异常向上抛（其余任务**不会被自动取消**，除非用 TaskGroup）；想"收集异常而不中断"，用 `return_exceptions=True`，异常会作为结果元素返回。

### 4.3 TaskGroup：结构化并发（3.11+，首选）

3.11 引入的 `asyncio.TaskGroup` 是现在**推荐的并发写法**，对应 Loom 的 `StructuredTaskScope`。它保证：**作用域退出时所有子任务都已完成；任一任务失败，其余任务被自动取消**，异常以 `ExceptionGroup` 聚合抛出。比裸 `gather` 更安全（不会泄漏游离任务）。

```python
import asyncio

async def fetch(name, delay):
    await asyncio.sleep(delay)
    return f"{name} ok"

async def main():
    results = []
    async with asyncio.TaskGroup() as tg:      # 3.11+
        t1 = tg.create_task(fetch("A", 0.2))
        t2 = tg.create_task(fetch("B", 0.1))
        # 退出 async with 时，自动等待 t1、t2 全部完成
    print(t1.result(), t2.result())            # 输出: A ok B ok

asyncio.run(main())
```

如果 `t1` 抛异常，`t2` 会被自动取消，整体抛 `ExceptionGroup`（用 `except*` 语法捕获，3.11+）。**生产代码优先 TaskGroup，需要简单聚合且能容忍其默认语义时才用 gather。**

### 4.4 as_completed：按完成顺序处理

要"谁先回来先处理谁"（流式出结果、做超时竞速），用 `as_completed`：

```python
import asyncio

async def fetch(name, delay):
    await asyncio.sleep(delay)
    return name

async def main():
    coros = [fetch("慢", 0.3), fetch("快", 0.1), fetch("中", 0.2)]
    for earliest in asyncio.as_completed(coros):
        result = await earliest        # 按完成先后 yield，先拿到"快"
        print("先到:", result)         # 输出顺序: 快 → 中 → 慢

asyncio.run(main())
```

### 4.5 超时与取消

```python
import asyncio

async def slow_op():
    await asyncio.sleep(10)
    return "done"

async def main():
    # 方式一：asyncio.timeout 上下文管理器（3.11+，推荐）
    try:
        async with asyncio.timeout(0.5):       # 0.5 秒内没完成就取消并抛 TimeoutError
            await slow_op()
    except TimeoutError:
        print("超时了！")                       # 输出: 超时了！

    # 方式二：asyncio.wait_for（各版本通用）
    try:
        await asyncio.wait_for(slow_op(), timeout=0.5)
    except TimeoutError:
        print("也超时了！")                     # 输出: 也超时了！

asyncio.run(main())
```

**取消**：`Task.cancel()` 会在该任务下一个 `await` 点注入一个 `CancelledError`。协程可以 `try/except` 捕获它做清理，但**最佳实践是清理后重新抛出**（吞掉 `CancelledError` 会破坏取消语义，让上层以为任务还活着）：

```python
import asyncio

async def worker():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("收到取消，清理资源中...")        # 做清理
        raise                                  # ★必须重新抛出，让取消正常传播

async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(0.1)
    task.cancel()                              # 请求取消
    try:
        await task
    except asyncio.CancelledError:
        print("任务已取消")

asyncio.run(main())
# 输出: 收到取消，清理资源中... / 任务已取消
```

`CancelledError` 在 3.8+ 直接继承 `BaseException`（不是 `Exception`），所以 `except Exception` **抓不到它**——这是有意设计，防止你用宽泛的 `except Exception` 误吞取消信号。

### 4.6 async with / async for

`with`/`for` 的异步版本，用于**进入/退出或迭代时本身需要 `await`** 的场景（见 [15 异常与上下文管理器](15_异常与上下文管理器.md) 的同步版）：

```python
import asyncio

# async with：异步上下文管理器（__aenter__ / __aexit__ 是协程）
class AsyncConnection:
    async def __aenter__(self):
        await asyncio.sleep(0.01)      # 模拟异步建连
        print("连接已建立")
        return self
    async def __aexit__(self, *exc):
        await asyncio.sleep(0.01)      # 模拟异步关闭
        print("连接已关闭")
    async def query(self, sql):
        await asyncio.sleep(0.01)
        return [{"row": 1}]

# async for：异步迭代器（__anext__ 是协程，常见于流式读取 / 数据库游标 / 分页）
async def stream_rows(n):
    for i in range(n):
        await asyncio.sleep(0.01)      # 模拟逐行从网络拉取
        yield {"id": i}                # async def + yield = 异步生成器

async def main():
    async with AsyncConnection() as conn:      # 进入/退出都会 await
        rows = await conn.query("SELECT 1")
        print(rows)                            # 输出: [{'row': 1}]
    async for row in stream_rows(3):           # 每次取下一行都会 await
        print(row)                             # 输出: {'id': 0} / {'id': 1} / {'id': 2}

asyncio.run(main())
```

`httpx`/`aiohttp` 的客户端、`asyncpg` 的连接池、流式响应，全都用 `async with` / `async for`——这是异步生态的标准接口形态。

### 4.7 异步同步原语

asyncio 提供和 `threading` 同名的同步原语，但它们是**协作式、非线程安全**的，专给协程用，且都通过 `async with` / `await` 获取：

```python
import asyncio

async def main():
    # Lock：协程间互斥（注意：async with 获取）
    lock = asyncio.Lock()
    async with lock:
        pass                           # 临界区

    # Semaphore：限并发数——异步场景超高频（限制同时打到下游的请求数）
    sem = asyncio.Semaphore(10)        # 最多 10 个并发
    async def fetch_one(i):
        async with sem:                # 第 11 个会在这里等，直到有名额释放
            await asyncio.sleep(0.1)
            return i
    results = await asyncio.gather(*(fetch_one(i) for i in range(100)))
    print(len(results))                # 输出: 100，但任意时刻最多 10 个在并发

    # Event：协程间信号；Queue：生产者-消费者（异步管道，背压控制）
    queue = asyncio.Queue(maxsize=5)   # maxsize 提供背压：满了 put 会等待

asyncio.run(main())
```

> **`Semaphore` 限流是异步后端最实用的模式**：扇出调用下游时，不加限制地 `gather` 一万个请求会瞬间打爆下游或耗尽连接。用 `Semaphore(N)` 把并发钉在 N 以内。`asyncio.Queue` 则是协程版的生产者-消费者，`maxsize` 天然提供背压（对应 Reactor 的 backpressure）。

### 4.8 何时该用 async、何时别用

| 场景 | 用 asyncio？ | 说明 |
|------|------------|------|
| 高并发 I/O（API 网关、爬虫、微服务扇出、WebSocket、长连接） | ✅ 强烈推荐 | 这是 asyncio 的主场，单线程扛上万连接 |
| CPU 密集（图像处理、加密、纯计算） | ❌ 别用 | 没有并行收益，反而卡死循环；用多进程 |
| 简单脚本 / 少量 I/O（几个请求） | ⚠️ 没必要 | 引入 async 的复杂度不值；同步 `requests` + 线程池足够 |
| 已有全套异步驱动（httpx/asyncpg/aioredis） | ✅ 推荐 | 生态齐了才能发挥威力 |
| 关键依赖只有同步库（某老 SDK、同步 ORM） | ⚠️ 谨慎 | 全靠 `run_in_executor` 兜底会很别扭，可能不如直接多线程 |
| 混合 CPU + I/O | ✅ + 进程池 | I/O 用 async，CPU 部分 `run_in_executor(ProcessPool)` 外包 |

> **黄金判据**：你的瓶颈是不是"大量并发等待 I/O"？是 → asyncio。是 CPU？→ 多进程。量不大或没有异步驱动？→ 老老实实同步 + 线程池，别为了异步而异步。

### 4.9 生态速览：换上异步驱动才有意义

asyncio 只是引擎，发挥威力要配套的异步库（同步库会卡死循环！）：

- **HTTP 客户端**：`httpx`（同时支持同步/异步，API 像 requests，推荐）、`aiohttp`（同时是客户端 + 服务端框架）。**别用 `requests`**（同步阻塞）。
- **数据库**：`asyncpg`（PostgreSQL，极快）、`aiomysql`、`asyncmy`；ORM 用 SQLAlchemy 2.0 的 async 引擎。见 [25 数据库与ORM](25_数据库与ORM.md)。
- **Redis**：`redis.asyncio`（redis-py 内置）。
- **Web 框架**：FastAPI / Starlette（原生异步）、`uvicorn`（ASGI 服务器，底层就是 asyncio/uvloop 跑你的协程）。见 [24 HTTP与Web框架](24_HTTP与Web框架.md)。
- **更快的事件循环**：`uvloop`（用 Cython 包装 libuv，替换默认循环，提速 2-4 倍）。uvicorn 默认就用它。

```python
# httpx 异步客户端：标准的异步 HTTP 写法
import asyncio
import httpx                            # pip install httpx

async def fetch(client, url):
    resp = await client.get(url)        # await：发请求期间让出循环
    return resp.status_code

async def main():
    async with httpx.AsyncClient() as client:   # async with 管理连接池
        codes = await asyncio.gather(
            fetch(client, "https://example.com"),
            fetch(client, "https://example.org"),
        )
        print(codes)                    # 输出: [200, 200]（需联网；离线时用 asyncio.sleep 模拟）

# asyncio.run(main())                   # 取消注释并联网运行
```

---

## ⑤ ⚠️ 易错点（Java 思维翻车）

| Java 习惯 | 翻车现场 | 正确做法 / 认知 |
|----------|---------|----------------|
| 以为 Loom 会替我挂起阻塞调用 | 在协程里调 `time.sleep`/`requests.get`/同步 DB，**整个事件循环假死**，所有请求一起卡 | 换异步实现（`asyncio.sleep`/`httpx`/`asyncpg`），实在不行 `await asyncio.to_thread(...)` 外包 |
| 以为调用 `async def` 就执行了 | `fetch_user(1)` 后啥也没发生，只得到 `<coroutine object>`，还报 `coroutine was never awaited` | 协程对象要被 `await` 或 `create_task` 驱动才执行 |
| 漏写 `await` | `data = fetch()` 后 `data` 是协程对象不是结果，后续 `data["x"]` 直接 `TypeError` | I/O 调用前**必须 `await`**；类型检查器/`-W` 警告能帮你抓 |
| 把 CPU 密集塞进 async | `async def` 里跑大循环/加密，并发量上去后延迟暴涨 | asyncio 不并行 CPU；CPU 任务丢 `ProcessPoolExecutor` |
| 忘了 `asyncio.run` | 写完一堆 `async def` 直接调，程序"什么都没发生"就退出 | 顶层必须 `asyncio.run(main())` 启动事件循环 |
| 用 `ThreadLocal` 传 traceId/用户上下文 | 万级协程共享一个线程，上下文**互相串味**，traceId 错乱 | 用 `contextvars.ContextVar`，随协程上下文走 |
| 串行 `await` 当成并发 | `await a(); await b()` 以为并发，其实**串行**，耗时是两者之和 | 要并发用 `gather`/`TaskGroup`/`create_task` |
| 用 `except Exception` 想兜底取消 | `CancelledError` 继承 `BaseException`，被宽 `except Exception` 漏掉或被误吞 | 单独 `except asyncio.CancelledError`，清理后 `raise` 重新抛出 |
| 裸 `gather` 不管异常 | 一个子任务异常被忽视（`return_exceptions=True` 时）或其它任务泄漏不取消 | 生产用 `TaskGroup`（自动取消 + `ExceptionGroup`）；或显式处理 gather 异常 |
| 不限并发地 `gather` 一万个请求 | 瞬间打爆下游 / 耗尽文件描述符与连接池 | 用 `asyncio.Semaphore(N)` 限流 |
| 同步/异步代码混用（"函数颜色"问题） | 同步函数里想调异步函数，只能拿到协程对象没法 await | 异步会"传染"调用链；要么全异步，要么用 `asyncio.run`/`run_in_executor` 搭桥 |
| `create_task` 后不保存引用 | Task 被 GC 回收，任务"神秘消失"或丢异常 | 持有 Task 引用（存进集合/用 TaskGroup），别让它游离 |

> **"函数颜色"（function coloring）**这个概念值得 Java 老手专门记一下：`async` 函数和普通函数是"两种颜色"，异步会沿调用链向上传染——要 `await` 一个异步函数，调用者也得是 `async`。这正是 Loom 想消除的痛点（虚拟线程下同步异步同色）。Python 没有 Loom，所以你必须接受"异步传染"这个事实，在设计时就决定哪些层是异步的。

---

## ⑥ 练习题

### 基础理解

1. 用自己的话解释"协作式调度"和"抢占式调度"的区别，并回答：为什么在 asyncio 协程里两个 `await` 之间的代码块**天然是临界区**、通常不需要加锁？这和 Java 多线程需要 `synchronized` 的根本差异在哪？

2. 下面三种写法，哪些是**并发**、哪些是**串行**？分别说明总耗时（每个 `work` 内部 `await asyncio.sleep(1)`）：
```python
# (a)
await work(1); await work(2); await work(3)
# (b)
await asyncio.gather(work(1), work(2), work(3))
# (c)
t1 = asyncio.create_task(work(1)); t2 = asyncio.create_task(work(2))
await asyncio.sleep(2); await t1; await t2
```

### 找坑改错

3. 下面这段"并发抓取用户资料"的异步代码有**三个坑**（一个让根本没并发/没执行、一个会卡死事件循环、一个上下文传递错误），找出来并改正：
```python
import asyncio, time, requests
import threading

current_user = threading.local()      # 存当前请求的用户

async def fetch_profile(uid):
    current_user.uid = uid
    time.sleep(1)                                  # 等待"网络"
    resp = requests.get(f"https://api/{uid}")      # 同步 HTTP
    return f"user {current_user.uid}: {resp.status_code}"

async def main():
    coros = [fetch_profile(i) for i in range(5)]
    results = [fetch_profile(i) for i in range(5)]  # 想并发跑 5 个
    print(results)

asyncio.run(main())
```

4. 下面的取消处理"吞掉"了取消信号，导致上层 `wait_for` 超时后任务仍在后台跑、`TaskGroup` 无法收敛。指出问题并修正：
```python
async def worker():
    try:
        await asyncio.sleep(100)
    except asyncio.CancelledError:
        print("被取消了")
        return "cleaned"          # 吞掉了 CancelledError
```

### 综合实战

5. 写一个异步函数 `fetch_all(urls)`，**并发**抓取一批 URL 并对比串行耗时，要求：① 用 `asyncio.Semaphore` 把并发数限制在 5；② 单个请求超过 2 秒算超时，记为失败但不影响其它；③ 用 `asyncio.gather(..., return_exceptions=True)` 或 `TaskGroup` 收集结果；④ 打印"成功 N 个、失败 M 个、总耗时"。为保证离线可跑，用 `await asyncio.sleep(随机延时)` 模拟网络请求（其中个别延时 > 2 秒以触发超时）。给出完整可运行代码。

---

### 参考答案

**1.** 抢占式（Java 线程/虚拟线程）：调度器可在**任意指令边界**强行挂起当前执行流去跑别人，所以 `count++` 这种非原子操作中间可能被切走，必须用 `synchronized`/原子类保护共享状态。协作式（asyncio）：事件循环**只能在协程显式 `await` 的那一刻**拿回控制权，两个 `await` 之间的代码不可能被另一个协程打断——它是事实上的临界区，所以读改写共享数据（只要中间不 `await`）天然安全，无需锁。根本差异：抢占由内核/运行时主导且不可预测，协作由协程主动让出且让出点在代码里可见。（注意：跨 `await` 的"读—await—写"仍可能被插入，那种情况才需要 `asyncio.Lock`。）

**2.**
- **(a) 串行**，总耗时约 **3 秒**：每个 `await` 都等当前 `work` 整个跑完才继续。
- **(b) 并发**，总耗时约 **1 秒**：`gather` 把三个协程同时驱动，一起等各自的 1 秒。
- **(c) 并发**，总耗时约 **2 秒**：两个 `create_task` 一创建就开始并发推进（各 1 秒，约 1 秒后都已完成），随后 `await asyncio.sleep(2)` 自己又等了 2 秒，`await t1/t2` 此时立即返回——所以总耗时由那句 `sleep(2)` 主导，约 2 秒。

**3.** 三个坑：
- **坑一（没执行/没并发）**：`results = [fetch_profile(i) for i in range(5)]` 只是创建了 5 个**协程对象**，从未 `await`，函数体一行都没跑（`coros` 那行同理是废代码）。要并发必须 `await asyncio.gather(*(fetch_profile(i) for i in range(5)))`。
- **坑二（卡死循环）**：`time.sleep(1)` 和 `requests.get`（同步阻塞）会霸占事件循环，5 个"并发"实际被串成串行且期间整个循环假死。换成 `await asyncio.sleep(1)` 和 `httpx.AsyncClient().get(...)`（异步）。
- **坑三（上下文串味）**：`threading.local()` 在单线程多协程下被所有协程共享，`current_user.uid` 会互相覆盖。改用 `contextvars.ContextVar`。
修正版：
```python
import asyncio
import contextvars
import httpx

current_user = contextvars.ContextVar("current_user")

async def fetch_profile(client, uid):
    current_user.set(uid)
    await asyncio.sleep(0.01)                       # 模拟，或真实 await
    resp = await client.get(f"https://httpbin.org/status/200")
    return f"user {current_user.get()}: {resp.status_code}"

async def main():
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(
            *(fetch_profile(client, i) for i in range(5))
        )
    print(results)

asyncio.run(main())
```

**4.** 问题：`except asyncio.CancelledError` 捕获后只 `return` 而不重新 `raise`，相当于**把取消信号吞了**。对外界（`cancel()` 的发起者、`wait_for`、`TaskGroup`）来说，这个任务表现为"正常完成并返回了 'cleaned'"，而不是"已被取消"——于是取消语义被破坏，结构化并发无法正确收敛、`wait_for` 也可能误判。修正：清理后**必须 `raise` 重新抛出 `CancelledError`**：
```python
async def worker():
    try:
        await asyncio.sleep(100)
    except asyncio.CancelledError:
        print("被取消了，清理中...")
        # ... 释放资源 ...
        raise                          # ★重新抛出，让取消正常传播
```

**5.** 参考实现：
```python
import asyncio
import random
import time

async def fetch_one(sem, url):
    async with sem:                              # ① 限流：最多 5 个并发
        delay = random.uniform(0.5, 3.0)         # 模拟网络耗时，部分会 > 2s
        async with asyncio.timeout(2.0):         # ② 单请求 2 秒超时（3.11+）
            await asyncio.sleep(delay)            # 模拟请求；真实场景是 await client.get(url)
            return f"{url} -> 200 ({delay:.2f}s)"

async def fetch_all(urls):
    sem = asyncio.Semaphore(5)
    start = time.perf_counter()
    # ③ return_exceptions=True：超时/失败作为结果元素返回，不中断其它任务
    results = await asyncio.gather(
        *(fetch_one(sem, u) for u in urls),
        return_exceptions=True,
    )
    elapsed = time.perf_counter() - start

    ok = [r for r in results if not isinstance(r, Exception)]
    failed = [r for r in results if isinstance(r, Exception)]
    # ④ 统计
    print(f"成功 {len(ok)} 个，失败 {len(failed)} 个，总耗时 {elapsed:.2f}s")
    for r in ok:
        print("  OK  ", r)
    for r in failed:
        print("  FAIL", type(r).__name__)        # 超时显示 TimeoutError
    return results

async def main():
    urls = [f"https://api.example.com/item/{i}" for i in range(20)]

    # 串行基线对比（仅演示差距，真实别这么干）
    start = time.perf_counter()
    for u in urls[:5]:
        try:
            async with asyncio.timeout(2.0):
                await asyncio.sleep(random.uniform(0.5, 1.5))
        except TimeoutError:
            pass
    print(f"[串行抓 5 个基线] 约 {time.perf_counter() - start:.2f}s\n")

    await fetch_all(urls)

asyncio.run(main())
# 典型输出（随机）:
# [串行抓 5 个基线] 约 4.83s
# 成功 16 个，失败 4 个，总耗时 2.41s     ← 20 个并发（限流 5），远快于串行
#   OK   https://api.example.com/item/3 -> 200 (0.71s)
#   ...
#   FAIL TimeoutError
```
要点：① `Semaphore(5)` 把同时在飞的请求钉在 5 个以内，保护下游；② `asyncio.timeout(2.0)` 给每个请求独立超时，超时触发取消并抛 `TimeoutError`；③ `return_exceptions=True` 让单个失败不拖垮整体（用 `TaskGroup` 则需配合 `except*` 处理 `ExceptionGroup`，语义是"一个失败全组取消"，按需求取舍）；④ 总耗时 ≈ 最慢批次而非所有请求之和，这就是异步并发的价值。对照 Java：等价于 `Semaphore` 限流 + `CompletableFuture.allOf` + 每个 future `orTimeout(2s)` + `exceptionally` 兜底。

---

下一章：[22 文件IO与系统交互](22_文件IO与系统交互.md) — 同步文件 I/O、路径与 `pathlib`、子进程、为什么文件 I/O 在 asyncio 里要丢线程池。
