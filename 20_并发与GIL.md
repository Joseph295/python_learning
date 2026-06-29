# 20 · 并发与 GIL ⭐

> 你在 Java 里靠 `Thread`、`ExecutorService`、`ForkJoinPool`、`ConcurrentHashMap` 把多核榨干。到了 Python，同样的"多开几个线程"思路会让你撞上一堵叫 **GIL** 的墙：八核机器、八个线程算数，CPU 只跑满一核。本章在 [05 CPython内部](05_CPython内部.md) 讲过的 GIL 机制之上讲实战——`threading`、`queue`、`multiprocessing`、`concurrent.futures` 各自的定位，以及那条 Java 老手必须重写的肌肉记忆：**Python 里"多线程做 I/O、多进程做 CPU"**。本章不碰 asyncio，那是 [21 异步编程asyncio](21_异步编程asyncio.md) 的事。

---

## ① Java 对照表

| 概念 | Java | Python | 一句话差异 |
|------|------|--------|-----------|
| 线程对象 | `Thread` / `Runnable` | `threading.Thread` | 都是**真 OS 线程**；但 Python 线程被 GIL 串行化字节码执行 |
| 互斥锁 | `synchronized` / `ReentrantLock` | `threading.Lock` | Python 的 `Lock` **不可重入**；要重入用 `RLock` |
| 可重入锁 | `ReentrantLock`（默认可重入） | `threading.RLock` | 对应 Java 锁的默认语义 |
| 条件变量 | `Object.wait/notify` / `Condition` | `threading.Condition` | 几乎一一对应，但要配 `while` 防虚假唤醒 |
| 闭锁/信号 | `CountDownLatch` / 标志位 | `threading.Event` | 一次性/可重置的"开关" |
| 信号量 | `Semaphore` | `threading.Semaphore` | 语义相同，限流常用 |
| 阻塞队列 | `BlockingQueue` | `queue.Queue` | 生产者-消费者标配；线程安全 |
| 线程局部 | `ThreadLocal` | `threading.local` | 语义相同 |
| 线程池/执行器 | `ExecutorService` | `concurrent.futures.ThreadPoolExecutor` | 高层抽象，几乎照搬 |
| 分治并行池 | `ForkJoinPool` | `ProcessPoolExecutor` | Python 靠**多进程**绕过 GIL 实现真并行 |
| 异步结果 | `Future` | `concurrent.futures.Future` | API 神似（`result()`/`done()`/`cancel()`） |
| 完成即取 | `CompletionService` | `as_completed()` | 谁先完成先拿谁 |
| 全局并行锁 | 无（JVM 真并行） | **GIL** | Python 并发模型的核心约束 |
| 虚拟线程 | `Thread.ofVirtual()`（JDK21） | asyncio（[21 章](21_异步编程asyncio.md)）/ 未来 free-threaded | 海量轻量并发的对应物 |
| 进程 | `ProcessBuilder` | `multiprocessing.Process` | Python 把"多进程"当一等并发手段 |

> 映射几乎是逐条对得上的——`concurrent.futures` 这套高层 API 像是照着 `java.util.concurrent` 设计的。**真正的鸿沟只有一个：GIL**。它让"线程"在 Python 里只对 I/O 有用，CPU 并行要靠进程。记住这一点，下面全是细节。

---

## ② 设计理念（为什么 Python 这么设计）

### 2.1 GIL 是什么、为什么存在（呼应 05 章）

[05 CPython内部](05_CPython内部.md) 已经讲过结论，这里点到为止再往实战推进：

**GIL（Global Interpreter Lock）是一把保护解释器内部状态的全局互斥锁，同一时刻只有持有它的线程能执行 Python 字节码。**

它存在的根本动机是**保护引用计数的原子性**。[04 运行时与内存模型](04_运行时与内存模型.md) 讲过，CPython 用引用计数管理内存，每个对象头里有 `ob_refcnt`。每次变量赋值、传参、离开作用域都要 `+1/-1`。这个 `+1/-1` 在机器层面是"读-改-写"三步，**不是原子操作**。如果两个线程同时给同一个对象的引用计数 `+1`，可能丢更新——计数算少了就提前释放、悬空指针、解释器崩溃。

JVM 怎么没这问题？因为 JVM 用**追踪式 GC**，不靠每次访问都改计数；对象的存活靠周期性可达性分析判断，天然不需要在每次引用时同步。CPython 选了引用计数（实时回收、实现简单、对 C 扩展友好），代价就是必须保护那个计数器。最省事的保护方式：**一把大锁锁住整个解释器**。

于是 GIL 成了一个工程取舍：
- **好处**：单线程极快（无需对每个对象上锁）、C 扩展好写（默认串行，不用考虑并发）、引用计数实现简单。
- **代价**：纯 Python 代码无法利用多核做 CPU 并行。

### 2.2 为什么是"线程做 I/O、进程做 CPU"

GIL 的关键细节（[05 章 2.5](05_CPython内部.md)）：**线程在执行阻塞 I/O 时会主动释放 GIL**。读文件、网络收发、`time.sleep`——这些都是在等外部设备，等的时候持着 GIL 毫无意义，于是 CPython 在进入这类系统调用前释放 GIL，让别的线程跑。

这就决定了分工：

```
I/O 密集（爬虫、调 API、读数据库）：
    线程 A 等网络  ─释放GIL─→ 线程 B 跑  ─释放GIL─→ 线程 C 跑 ...
    多线程有效！等待时间被重叠掉了。

CPU 密集（纯 Python 算数、加密、压缩）：
    线程全程持 GIL 跑字节码，谁也不释放
    → 多线程被串行化，加不了速，还多了切换开销 → 用多进程
```

多进程为什么能绕开？**每个进程有自己独立的 Python 解释器、自己独立的 GIL**。8 个进程 = 8 把 GIL = 8 核真并行。代价是进程比线程重、数据要跨进程传（序列化开销，见 3.3）。

### 2.3 concurrent.futures：统一的高层抽象

Python 早期只有 `threading` 和 `multiprocessing` 两套底层 API，写法繁琐。3.2 引入 `concurrent.futures`，提供和 Java `ExecutorService` 几乎一样的高层模型：**你提交任务（`submit`）拿回 `Future`，或批量 `map`，由 Executor 管理工作线程/进程池**。

它最大的价值是**线程池和进程池共享同一套 API**——`ThreadPoolExecutor` 和 `ProcessPoolExecutor` 接口完全一致。这意味着你可以先用线程池写，发现是 CPU 密集就改一个类名换成进程池，业务代码不动。这正是"先用统一抽象、按需切换执行后端"的设计哲学，和 Java `Executors.newFixedThreadPool` vs `ForkJoinPool` 一脉相承。

> **心智模型**：日常并发优先用 `concurrent.futures`（高层、好用、不易错）。只有在需要精细控制（自定义同步原语、生产者-消费者拓扑、共享内存）时才下沉到 `threading` / `multiprocessing`。

---

## ③ 实现细节（底层怎么实现的）★本章重点★

### 3.1 Python 线程是真 OS 线程，只是被 GIL 串行化

一个常见误解：以为 Python 线程是"绿色线程"或协程。**不是**。`threading.Thread` 底层是 `pthread_create`（Unix）/ `CreateThread`（Windows），是货真价实的内核线程，由 OS 调度器调度。

那 GIL 怎么"串行化"它们？机制是这样的：

```
线程要执行字节码 → 必须先 acquire GIL
执行一段时间后（或遇到 I/O）→ release GIL
其它等待的线程之一被唤醒 → acquire GIL → 继续
```

"执行一段时间"具体是多久？由 **switch interval** 控制，默认 5ms：

```python
import sys
print(sys.getswitchinterval())   # 输出: 0.005  （5 毫秒）
```

注意这里和老版本（3.2 之前）的区别：老 GIL 是"每执行 N 条字节码切一次"，会导致 CPU 任务饿死 I/O 任务。3.2 后改成**基于时间**的 GIL：持锁线程跑满 5ms 后，会设置一个标志请求释放，等待线程超时后会强制要求切换。这缓解了但没消除"GIL 竞争"开销——多个 CPU 线程互相抢 GIL，切换本身就是纯损耗，所以双线程 CPU 任务常常**比单线程还慢**（[05 章 2.5](05_CPython内部.md) 的实验）。

哪些操作会释放 GIL（让多线程真正并发）：
- **阻塞 I/O**：socket 收发、文件读写、`time.sleep()`。
- **部分 C 扩展主动释放**：numpy 的大数组运算、`hashlib` 的哈希、zlib 压缩，在进入纯 C 计算前用 `Py_BEGIN_ALLOW_THREADS` 宏释放 GIL。这就是为什么 numpy 多线程能加速，而纯 Python 循环不能。

```python
# 验证 I/O 会释放 GIL：多线程 sleep 几乎不累加耗时
import threading, time

def io_task():
    time.sleep(1)            # 模拟 I/O，期间释放 GIL

start = time.perf_counter()
threads = [threading.Thread(target=io_task) for _ in range(8)]
for t in threads: t.start()
for t in threads: t.join()
print(f"8 个线程各 sleep 1 秒，总耗时: {time.perf_counter() - start:.2f}s")
# 输出: 8 个线程各 sleep 1 秒，总耗时: 1.00s   ← 不是 8 秒！并发了
```

### 3.2 GIL 不保证你的代码线程安全——`+=` 不是原子的

Java 老手最容易翻的车：**"既然有 GIL 全局锁，那我的共享变量是不是天生线程安全？"** 不是。

GIL 保证的是**单条字节码指令**执行时不被打断，以及解释器内部状态（引用计数等）的一致性。但你的一行 Python 往往编译成**多条**字节码，GIL 可能在指令之间切走。

```python
import dis
def incr(counter):
    counter.value += 1

dis.dis(incr)
```

`counter.value += 1` 反汇编（简化）：
```
  LOAD_FAST    counter
  LOAD_ATTR    value        # ① 读 counter.value
  LOAD_CONST   1
  BINARY_OP    +=           # ② 加 1
  STORE_ATTR   value        # ③ 写回 counter.value
```

读、加、写是**三条独立指令**。GIL 完全可能在 ① 之后、③ 之前切到另一个线程，两个线程读到同样的旧值、各自 +1、各自写回——丢了一次更新。这就是经典的竞态条件，和 Java 里 `count++` 非原子是**一模一样**的坑。

```python
import threading

class Counter:
    def __init__(self): self.value = 0

counter = Counter()
def worker():
    for _ in range(100_000):
        counter.value += 1     # 非原子！

threads = [threading.Thread(target=worker) for _ in range(8)]
for t in threads: t.start()
for t in threads: t.join()
print(counter.value)   # 期望 800000，实际常常更小，如 743912（每次运行不同）
```

> **结论**：GIL ≠ 免锁。共享可变状态该加锁还得加锁（用 `Lock`），或者干脆别共享（用 `queue.Queue` 传消息）。有些操作恰好是单条字节码（如对 `list.append`、`dict[k]=v` 这类内置容器的单一方法调用）在 CPython 里**碰巧**原子，但**别依赖这个**——它是实现细节，不是语言保证，free-threaded 构建下更不能赌。

### 3.3 multiprocessing：独立解释器 + pickle 序列化传数据

多进程是 Python 实现 CPU 并行的正路，但它的代价全藏在"进程间怎么传数据"里。理解这点，你才能解释一连串看似奇怪的报错和开销。

**创建方式：fork vs spawn**

```python
import multiprocessing as mp
print(mp.get_start_method())   # Linux: 'fork'；macOS(3.8+)/Windows: 'spawn'
```

- **fork（Linux 默认）**：直接复制父进程的整个内存空间（写时复制 COW）。子进程一出生就拥有父进程的所有变量、已导入的模块。快，但危险（见 3.4）。
- **spawn（macOS/Windows 默认）**：从零启动一个全新 Python 解释器，**重新导入**你的主模块，只把必要的东西序列化传过去。干净，但慢，且带来 `if __name__ == "__main__"` 的坑（见 ⑤）。

> macOS 在 3.8 起把默认从 fork 改成 spawn，因为 fork 在 macOS 上和系统库（如 Grand Central Dispatch）冲突会崩溃。所以**跨平台代码要按 spawn 的规则写**。

**数据怎么传：pickle + 管道**

进程内存不共享。你传给 `Process(target=f, args=(x,))` 的参数 `x`、`f` 的返回值，都要先用 **pickle 序列化成字节**，通过操作系统管道（pipe）送到对方进程，再反序列化还原。

这解释了三件事：

1. **为什么参数和返回值必须可 pickle**。lambda、本地函数、打开的文件句柄、数据库连接、线程锁——这些都不能 pickle，传过去直接报错：
   ```python
   import multiprocessing as mp
   with mp.Pool(2) as p:
       p.map(lambda x: x*2, [1,2,3])   # 报错: Can't pickle <lambda>
   ```
   必须用模块级具名函数。

2. **为什么有开销**。每次传参/取返回值都是一次"序列化 → 拷贝 → 反序列化"。传一个大 DataFrame 给子进程可能比计算本身还慢。所以多进程适合**计算重、数据传输轻**的任务（CPU 密集），不适合频繁传大数据。

3. **为什么子进程改了变量父进程看不到**。它们是各自独立的内存副本，不是共享。

**真正共享数据的几种手段**（按从轻到重）：

| 手段 | 适用 | 说明 |
|------|------|------|
| `Value` / `Array` | 共享简单标量/定长数组 | 基于共享内存，带锁，最轻 |
| `shared_memory.SharedMemory`（3.8+） | 大块二进制 / numpy 数组零拷贝 | 真共享内存，避免序列化大数组 |
| `Manager()` | 共享 dict/list 等复杂结构 | 起一个**服务进程**代管对象，其它进程通过代理 + IPC 访问，**每次访问都有 IPC 开销**，慢 |
| `Queue` / `Pipe` | 传消息 | 底层也是 pickle + 管道 |

```python
from multiprocessing import shared_memory
import numpy as np

# 父进程创建共享内存放一个大数组，子进程零拷贝读取（无 pickle 大数组）
arr = np.arange(1_000_000, dtype=np.float64)
shm = shared_memory.SharedMemory(create=True, size=arr.nbytes)
buf = np.ndarray(arr.shape, dtype=arr.dtype, buffer=shm.buf)
buf[:] = arr[:]                       # 写进共享内存
# 把 shm.name（一个字符串）传给子进程，子进程 attach 同名共享内存即可读到
print(shm.name)                       # 输出: 类似 'psm_a1b2c3'
shm.close(); shm.unlink()             # 用完要 unlink，否则泄漏
```

### 3.4 fork 与全局状态/锁的坑

fork 只复制**调用 fork 的那个线程**，但复制**整个内存**——包括其它线程持有的锁的状态。如果父进程某个线程正持着一把锁时 fork，子进程里那把锁是"已锁定"状态却**永远没有线程会去释放它**（持锁的线程没被复制过来）→ 子进程一去 acquire 就**死锁**。

这就是 fork 模式下"父进程用了多线程 + 锁，再 fork 子进程"经典死锁的根源，也是 macOS 改用 spawn 的原因之一。实战规避：
- 跨平台统一用 spawn：`mp.set_start_method("spawn")`。
- 别在已经起了线程的进程里 fork。
- 子进程里需要的资源（DB 连接、日志 handler）在子进程内**重新初始化**，别指望继承父进程的。

### 3.5 Pool 与 Executor 的工作分发

`multiprocessing.Pool` 和 `concurrent.futures` 的 Executor 内部都是**任务队列 + worker 池**模型：

```
       submit/map
你的代码 ───────────→ [任务队列] ──→ worker1 (线程或进程)
                                  ──→ worker2
                                  ──→ worker3
        Future ←──── [结果队列] ←──  worker 完成回填结果
```

- **ThreadPoolExecutor**：worker 是线程，任务队列是进程内的 `queue.Queue`，零序列化（同进程共享对象）。
- **ProcessPoolExecutor**：worker 是进程，任务和结果都要 pickle 跨进程传。提交的函数、参数必须可 pickle。

`map` 默认按提交顺序返回结果（即使后面的先算完也要等前面的）；`as_completed` 则是**谁先完成先 yield 谁**（对应 Java `CompletionService`）。批量任务可以用 `chunksize` 参数减少调度次数（把多个小任务打包成一批发给 worker，省 IPC）。

### 3.6 free-threaded Python（3.13t，no-GIL）

[05 章 2.5](05_CPython内部.md) 提到的"未来"已经落地：**Python 3.13 起提供官方的 free-threaded 构建**（PEP 703），二进制带 `t` 后缀（如 `python3.13t`），编译时 `--disable-gil`。它**去掉了 GIL**，让多线程能真正利用多核跑 Python 字节码。

它怎么在没有 GIL 的情况下保住引用计数安全？几项核心技术：
- **偏向引用计数（biased reference counting）**：区分"对象创建线程"和"其它线程"。创建线程对自己对象的计数操作走快速非原子路径（绝大多数对象只被一个线程碰），跨线程访问才走原子路径。
- **不朽对象（immortal objects，PEP 683）**：`None`、`True`、小整数、内置类型等永生对象的引用计数被冻结，根本不再增减，省掉大量同步。
- **细粒度锁**：给容器（dict/list）等加各自的锁，替代一把大 GIL。
- **延迟引用计数 + 线程安全的内存分配器（mimalloc）**。

代价与现状（截至 3.13/3.14）：
- 单线程性能有回退（早期约 5~10%，在持续优化）。
- **C 扩展要适配**才能在 free-threaded 下安全运行（很多还没适配，加载时可能重新启用 GIL）。
- 仍是**实验性、默认不启用**，需专门构建/下载。

```python
# 在 free-threaded 构建上可以检测 GIL 是否启用（3.13+）
import sys
if hasattr(sys, "_is_gil_enabled"):
    print(sys._is_gil_enabled())   # free-threaded 构建上可能输出 False
```

> **对 Java 老手的意义**：free-threaded 成熟后，Python 的多线程会越来越像 JVM——线程能用满多核、`threading` + 锁的并发模型直接可用于 CPU 并行。但"现在"（2026 年），生产代码仍要按"有 GIL"来设计：**CPU 并行用进程**。

---

## ④ 使用方法与最佳实践

### 4.1 ThreadPoolExecutor 跑 I/O 并发（首选写法）

I/O 密集（爬虫、批量调 HTTP API、并发查多个数据库）的标准解法。三种用法：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import urllib.request, time

URLS = ["https://httpbin.org/delay/1"] * 8   # 每个请求服务端 sleep 1 秒

def fetch(url: str) -> int:
    with urllib.request.urlopen(url, timeout=10) as resp:
        return len(resp.read())

# 用法 A：map —— 顺序对应输入，最简洁
start = time.perf_counter()
with ThreadPoolExecutor(max_workers=8) as pool:
    sizes = list(pool.map(fetch, URLS))
print(f"map: {len(sizes)} 个请求, 耗时 {time.perf_counter()-start:.1f}s")
# 输出: map: 8 个请求, 耗时 1.x s   ← 8 个串行要 8 秒，并发只要 1 秒多

# 用法 B：submit + as_completed —— 谁先回来先处理
with ThreadPoolExecutor(max_workers=8) as pool:
    futures = {pool.submit(fetch, u): u for u in URLS}
    for fut in as_completed(futures):           # 完成顺序，非提交顺序
        url = futures[fut]
        try:
            print(f"  {url} -> {fut.result()} bytes")
        except Exception as e:
            print(f"  {url} 失败: {e}")          # 异常在 result() 时重新抛出
```

要点：
- `with` 语句退出时自动 `shutdown(wait=True)`，等所有任务结束（像 `ExecutorService.close()`，JDK19+）。
- `max_workers` 对 I/O 可以设得比 CPU 核数大很多（线程大部分时间在等 I/O）。
- **异常不会丢**——它被存进 `Future`，你调 `fut.result()` 时重新抛出（见 3.2 的对比：裸 `Thread` 的异常会被默默吞掉）。

### 4.2 ProcessPoolExecutor 跑 CPU 并行

把 `ThreadPoolExecutor` 换成 `ProcessPoolExecutor`，业务代码几乎不动，就能吃满多核：

```python
from concurrent.futures import ProcessPoolExecutor
import time, os

def is_prime(n: int) -> bool:                 # CPU 密集：纯 Python 算数
    if n < 2: return False
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0: return False
    return True

def count_primes(lo_hi: tuple[int, int]) -> int:
    lo, hi = lo_hi
    return sum(is_prime(n) for n in range(lo, hi))

if __name__ == "__main__":                    # spawn 下必须有！见 ⑤
    chunks = [(i*250_000, (i+1)*250_000) for i in range(8)]

    start = time.perf_counter()
    with ProcessPoolExecutor(max_workers=os.cpu_count()) as pool:
        total = sum(pool.map(count_primes, chunks))
    print(f"进程池: {total} 个素数, 耗时 {time.perf_counter()-start:.1f}s")

    start = time.perf_counter()
    total = sum(count_primes(c) for c in chunks)   # 单进程对照
    print(f"单进程: {total} 个素数, 耗时 {time.perf_counter()-start:.1f}s")
    # 典型输出（8 核机器）:
    # 进程池: 144684 个素数, 耗时 1.8s
    # 单进程: 144684 个素数, 耗时 6.5s   ← 进程池快约 3-4 倍（受序列化/启动开销影响达不到满 8 倍）
```

要点：
- `max_workers` 设成 `os.cpu_count()`（CPU 密集，多了只会争抢）。
- 任务函数和参数必须**模块级、可 pickle**。
- 适合"计算重、传数据轻"。如果每个任务要传/收大对象，序列化会吃掉并行收益。

### 4.3 queue.Queue 做生产者-消费者

对应 Java `BlockingQueue`。线程安全，内部已加好锁，你不用自己同步：

```python
import threading, queue, time

q: queue.Queue[int | None] = queue.Queue(maxsize=10)   # 有界队列，满了 put 会阻塞
SENTINEL = None                                        # 毒丸：通知消费者结束

def producer():
    for i in range(20):
        q.put(i)                       # 队列满时阻塞（背压）
    q.put(SENTINEL)                    # 放一个结束信号

def consumer():
    while True:
        item = q.get()                 # 队列空时阻塞
        if item is SENTINEL:
            q.task_done()
            break
        print(f"处理 {item}", end=" ")
        q.task_done()                  # 配合 q.join() 统计完成

t_prod = threading.Thread(target=producer)
t_cons = threading.Thread(target=consumer)
t_prod.start(); t_cons.start()
t_prod.join(); t_cons.join()
print("\n全部处理完")
```

要点：
- `maxsize` 提供**背压**：生产快于消费时，`put` 阻塞，防止内存爆炸（对照 Java `ArrayBlockingQueue`）。
- 多消费者时，每个消费者都要收到一个 sentinel，或用 `q.join()` + `task_done()` 协调。
- 进程间用 `multiprocessing.Queue`（接口相似，但走 pickle + 管道）。

### 4.4 Lock 保护共享状态

需要共享可变状态时（计数器、缓存），用 `Lock`：

```python
import threading

class SafeCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def incr(self):
        with self._lock:               # 等价 Java synchronized 块；退出自动释放
            self._value += 1            # 临界区：读-改-写现在是原子的

    @property
    def value(self): return self._value

counter = SafeCounter()
threads = [threading.Thread(
    target=lambda: [counter.incr() for _ in range(100_000)]) for _ in range(8)]
for t in threads: t.start()
for t in threads: t.join()
print(counter.value)                   # 输出: 800000   ← 这次准确
```

- `Lock` **不可重入**：同一线程二次 `acquire` 会**死锁**。需要在持锁时再调用也要锁的方法，用 `RLock`（可重入，对应 Java `ReentrantLock` 默认语义）。
- `Condition` 用于"等某个条件成立"（配 `while` 防虚假唤醒）；`Event` 是一次性/可重置开关（像 `CountDownLatch`）；`Semaphore` 限制并发数（限流）。

### 4.5 决策表：线程 vs 进程 vs asyncio

| 场景 | 选择 | 理由 |
|------|------|------|
| I/O 密集，几十~几百并发（爬虫、批量调 API） | **线程池** `ThreadPoolExecutor` | I/O 时释放 GIL，写法最简单 |
| I/O 密集，**海量**连接（万级，如长连接网关） | **asyncio**（[21 章](21_异步编程asyncio.md)） | 单线程事件循环，无线程栈开销 |
| CPU 密集（计算、加密、图像/数据处理） | **进程池** `ProcessPoolExecutor` | 每进程独立 GIL，真多核并行 |
| CPU 密集但能向量化（数值） | **numpy** / **C 扩展** | 下沉到 C，C 层释放 GIL，常比多进程还省心 |
| 混合（少量 CPU + 大量 I/O） | 线程/async 为主，CPU 部分丢进程池 | 各取所长 |
| 任务间要频繁共享大量可变状态 | **线程**（共享内存）或重新设计 | 进程间共享贵；优先消息传递避免共享 |

### 4.6 避免共享可变状态

并发最佳实践的"上位思想"：**能不共享就不共享**。

- 优先**消息传递**（`queue.Queue` 在线程间，`multiprocessing.Queue` 在进程间）而非共享变量 + 锁——这也是 Go channel、Actor 模型的思路。
- 让任务函数**纯函数化**：输入参数、返回结果，不碰外部可变状态。`map` 模式天然如此。
- 需要线程独立状态（如每线程一个 DB 连接、requests session）用 `threading.local`（对应 `ThreadLocal`）：

```python
import threading
_local = threading.local()
def get_session():
    if not hasattr(_local, "session"):
        _local.session = make_session()    # 每个线程一份，互不干扰
    return _local.session
```

---

## ⑤ ⚠️ 易错点（Java 思维翻车）

| Java 习惯 | 翻车现场 | 正确做法 / 认知 |
|----------|---------|----------------|
| 开多线程跑 CPU 密集期待加速 | 8 线程算数，CPU 只满一核，甚至比单线程慢 | GIL 串行化字节码；CPU 并行用 `ProcessPoolExecutor`（[05 章](05_CPython内部.md)） |
| 以为有 GIL 就不用加锁 | 8 线程对计数器 `+=`，结果比预期小 | `x += 1` 是读-改-写三条字节码，非原子；共享可变状态要 `Lock` |
| 给进程池传 lambda / 本地函数 / 连接对象 | `Can't pickle <lambda>` / `TypeError` | 进程间靠 pickle 传数据；用模块级具名函数，传可序列化的值 |
| 多进程代码不写 `if __name__=="__main__"` | spawn 下子进程重导入主模块 → **无限递归创建进程** / `RuntimeError` | 把启动代码放进 `if __name__ == "__main__":`（spawn 必须） |
| fork 前已起线程并持锁 | 子进程死锁（持锁线程没被复制） | 跨平台用 spawn；子进程内重建资源（DB/日志/连接） |
| 裸 `Thread(target=f).start()`，f 里抛异常 | 异常被**默默吞掉**，主线程毫无察觉 | 用 `Executor`，异常存进 `Future`，`result()` 时重抛；或自己 try/except + 日志 |
| `Lock` 当 `ReentrantLock` 用，持锁时再 acquire | 同一线程二次 acquire **死锁** | Python `Lock` 不可重入；需要重入用 `RLock` |
| 给 `ProcessPoolExecutor` 频繁传大 DataFrame | 序列化开销超过计算，越并行越慢 | 多进程适合"计算重传数据轻"；大数组用 `shared_memory` 零拷贝 |
| 用 `Manager().dict()` 当高性能共享 map | 每次读写都走 IPC，慢得离谱 | Manager 是代理 + 进程间通信；高频共享改用共享内存或重新设计 |
| `time.sleep` 在 `if cond` 里等条件（轮询） | 忙等浪费 CPU / 响应迟钝 | 用 `Condition`/`Event` 等待通知，对应 `wait/notify` |

---

## ⑥ 练习题

### 基础理解

1. 一个 Java 同事说："Python 的 `threading.Thread` 既然是真 OS 线程，那我开 8 个线程跑 CPU 计算就能用满 8 核。" 解释他错在哪，并说明：(a) 什么样的任务多线程**能**加速；(b) 要让 CPU 密集任务用满多核，该用什么。

2. 为什么给 `ProcessPoolExecutor.submit` 传一个 `lambda` 会报 `Can't pickle` 错误，而 `ThreadPoolExecutor` 不会？从两者底层数据传递机制的差异回答。

### 找坑改错

3. 下面这段"Java 味"的 Python 想用 8 个线程把一个大列表的每个元素平方后求和加速，但既慢又偶尔结果不对。指出**两个**问题并改对：

```python
import threading

total = 0
data = list(range(1, 1_000_001))

def work(chunk):
    global total
    for n in chunk:
        total += n * n          # 想并行累加

threads = []
size = len(data) // 8
for i in range(8):
    chunk = data[i*size:(i+1)*size]
    t = threading.Thread(target=work, args=(chunk,))
    threads.append(t)
    t.start()
for t in threads:
    t.join()
print(total)
```

4. 下面的多进程代码在 macOS / Windows 上运行直接报错（甚至疯狂创建进程）。指出根因并修正：

```python
from concurrent.futures import ProcessPoolExecutor

def square(x):
    return x * x

results = list(ProcessPoolExecutor().map(square, range(10)))
print(results)
```

### 综合实战

5. 你要给一个后端服务写一个批处理工具：输入 200 个商品 ID，对每个 ID 先**调一次外部 HTTP API** 拿原始数据（I/O 密集，每次约 200ms），再对返回的数据做一次**CPU 密集的打分计算**（纯 Python，每个约 100ms）。请设计并发方案：用什么跑 I/O 部分、什么跑 CPU 部分、为什么，并写出关键代码骨架（可伪代码）。说明如果直接用单一 `ThreadPoolExecutor` 把两步都放进去会有什么问题。

---

### 参考答案

**1.** 错在把"真 OS 线程"等同于"能并行跑字节码"。Python 线程虽是 OS 线程，但执行 Python 字节码必须先拿 GIL，同一时刻只有一个线程在跑字节码，CPU 计算被串行化（还多了抢锁切换开销，常比单线程慢）。
- (a) **I/O 密集**任务能加速：线程阻塞在 socket/文件/`sleep` 时**释放 GIL**，其它线程趁机跑，等待时间被重叠。释放 GIL 的 C 扩展计算（如 numpy 大数组）也能并行。
- (b) 用 **`ProcessPoolExecutor` / `multiprocessing`**，每个进程有独立解释器和独立 GIL，真多核并行；或把计算下沉到会释放 GIL 的 C 库（numpy）。

**2.** `ThreadPoolExecutor` 的 worker 是**同进程内的线程**，共享同一内存空间，函数对象直接引用即可，无需序列化。`ProcessPoolExecutor` 的 worker 是**独立进程**，内存不共享，提交的函数和参数必须 **pickle 序列化**后经管道传到子进程再反序列化。`lambda`（以及本地函数、闭包）不可 pickle，所以报错。改用模块级具名函数即可。

**3.** 两个问题：
- **GIL + CPU 密集**：纯 Python 平方求和是 CPU 密集，多线程被 GIL 串行化，不但不加速，线程切换还让它更慢。应改用 `ProcessPoolExecutor`（或直接 numpy 向量化）。
- **`total += ...` 竞态**：`total += n*n` 是读-改-写非原子操作，多线程同时改 `global total` 会丢更新，结果偏小。
- 改法（用进程池 + 纯函数，避免共享）：
  ```python
  from concurrent.futures import ProcessPoolExecutor

  def work(chunk):
      return sum(n * n for n in chunk)   # 纯函数，返回局部和，不碰全局

  if __name__ == "__main__":
      data = list(range(1, 1_000_001))
      size = len(data) // 8
      chunks = [data[i*size:(i+1)*size] for i in range(8)]
      with ProcessPoolExecutor(max_workers=8) as pool:
          total = sum(pool.map(work, chunks))
      print(total)
  ```
  （注：此例数据量下其实单线程 `sum(n*n for n in data)` 更快——多进程序列化大 chunk 有开销；真要快用 numpy。重点是消除竞态 + 理解 CPU 密集不该用线程。）

**4.** 根因：没有 `if __name__ == "__main__":` 守护。macOS/Windows 默认 **spawn** 启动方式，子进程会**重新导入主模块**来获取 `square` 等定义；若模块顶层就有创建进程池的代码，重导入时又会执行它 → 递归创建进程，CPython 检测到会抛 `RuntimeError`（"An attempt has been made to start a new process before the current process has finished its bootstrapping phase..."）。修正：
```python
from concurrent.futures import ProcessPoolExecutor

def square(x):
    return x * x

if __name__ == "__main__":                  # 守护：只在主进程执行
    with ProcessPoolExecutor() as pool:
        results = list(pool.map(square, range(10)))
    print(results)                          # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

**5.** 方案：**I/O 部分用线程池，CPU 部分用进程池**，分两段流水。
- I/O（调 API）释放 GIL，线程池高并发，`max_workers` 可设几十；200 个 200ms 的请求并发后总耗时约几秒。
- CPU（打分）受 GIL 限制，线程池跑它不会并行；交给进程池吃满多核。
- 若**只**用一个 `ThreadPoolExecutor` 把两步都塞进去：I/O 部分并发良好，但 CPU 打分部分被 GIL 串行化，200×100ms ≈ 20 秒全压在一个核上，成为瓶颈，多核闲置。

关键骨架：
```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def fetch(item_id):            # I/O 密集
    return http_get(f"/api/items/{item_id}")     # 约 200ms，释放 GIL

def score(raw):                # CPU 密集，纯 Python，模块级可 pickle
    return compute_score(raw)  # 约 100ms

if __name__ == "__main__":
    ids = load_ids()           # 200 个
    # 第一段：线程池并发抓取
    with ThreadPoolExecutor(max_workers=32) as io_pool:
        raws = list(io_pool.map(fetch, ids))
    # 第二段：进程池并行打分
    with ProcessPoolExecutor(max_workers=8) as cpu_pool:
        scores = list(cpu_pool.map(score, raws))
    save(ids, scores)
```
进阶可做成流水线（抓到一个就丢进进程池打分，用 `as_completed` 衔接），让 I/O 和 CPU 重叠，进一步缩短总时长。

---

下一章：[21 异步编程asyncio](21_异步编程asyncio.md) —— 单线程事件循环、`async`/`await`、协程，对照虚拟线程，讲海量 I/O 并发的另一条路。
