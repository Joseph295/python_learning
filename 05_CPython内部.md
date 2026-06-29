# 05 · CPython 内部 ⭐

> 你懂 JVM：class 文件、字节码、解释器 + JIT、对象头、栈帧。本章把 CPython 当成"另一种 JVM"来拆解——源码怎么变字节码、`ceval` 求值循环怎么跑、`PyObject` 长什么样、GIL 到底锁的是什么、为什么 Python 比 Java 慢一个数量级、以及怎么用 C 把热点补回来。看懂这章，前面 GC、GIL、性能的结论你就都"知其所以然"了。

---

## ① Java 对照表

| 概念 | JVM | CPython |
|------|-----|---------|
| 官方实现 | HotSpot | **CPython**（C 写的，你装的就是它） |
| 编译产物 | `.class` 字节码文件 | `.pyc`（缓存的字节码，在 `__pycache__/`） |
| 编译时机 | `javac` 提前编译 | **导入/运行时**即时编译成字节码 |
| 执行方式 | 解释 + **JIT** 编译成机器码 | **纯解释**（CPython 无 JIT；3.13 起有实验性 JIT） |
| 字节码查看 | `javap -c` | `dis` 模块 |
| 对象头 | mark word + klass 指针 | `PyObject`：refcount + 类型指针 |
| 栈帧 | JVM 栈帧 | `PyFrameObject`（堆上分配） |
| 全局锁 | 无（真多线程） | **GIL**（全局解释器锁） |
| 替代实现 | GraalVM 等 | PyPy（带 JIT）、Jython、GraalPy |
| 原生互操作 | JNI | C API / ctypes / cffi / Cython |

---

## ② 设计理念（为什么这么设计）

你熟悉的 JVM 是"javac 提前编译 + 运行时 JIT 成机器码 + 无全局锁的真多线程"。CPython 几乎每一条都反着来，背后是一组刻意的取舍：

- **运行时编译字节码 + 纯解释执行（无 JIT）**：CPython 不做 JIT，每条字节码都走解释循环。这换来的是实现简单、可移植、启动快、与 C 扩展边界清晰——代价是纯 Python 计算比 Java 慢一个数量级。设计者赌的是：Python 的定位是**胶水语言**，热点会下沉到 C 库（numpy/数据库驱动），解释器本身不在关键路径上。
- **GIL（全局解释器锁）**：CPython 用一把全局锁保证"同一时刻只有一个线程执行字节码"。动机是**简单**——引用计数的 +1/-1 不是原子操作，多线程并发改会损坏内存；与其给每个对象加细粒度锁（复杂、慢、易错），不如用一把大锁一劳永逸。代价是多线程无法做 CPU 并行（见 ③ 的 2.5）。这是"实现简单性"压倒"多核并行"的典型取舍。
- **一切下沉到 C**：正因为解释器慢、又有 GIL，Python 生态演化出"逻辑用 Python 写、计算交给释放 GIL 的 C 扩展"的分工。理解这个定位，才能解释为什么"Python 很慢"在 Web 和数据科学领域都不成立（见 ⑥ 练习 5）。

> 一句话：CPython 拿"解释执行 + GIL"换"实现简单、可移植、易与 C 协作"，把性能问题外包给 C 扩展和未来的解释器优化。这套取舍在它的目标场景（胶水、脚本、I/O 密集后端、数据调度）里是划算的。

---

## ③ 实现细节（底层怎么实现的）

### 2.1 从源码到执行：四个阶段

```
源码 .py
  │  ① 词法+语法分析 → AST
  ▼
AST（抽象语法树）
  │  ② 编译器 compile()
  ▼
字节码（bytecode，存进 code object，缓存为 .pyc）
  │  ③ ceval 求值循环逐条解释
  ▼
执行结果
```

和 JVM 的关键差异：
- JVM 是 `javac` **提前**把源码编成 `.class`，运行时 JVM 再 JIT 成机器码。
- CPython 在**运行/导入时**才把 `.py` 编成字节码（编一次，缓存进 `__pycache__/*.pyc`，下次源码没改就直接用），然后**纯解释执行字节码，不编机器码**（标准 CPython 无 JIT）。这是 Python 慢的根本原因之一。

`__pycache__/` 里的 `.pyc` 就像 Java 的 `.class`，是字节码缓存。删掉它无害，下次自动重建。

### 2.2 字节码与 `dis`（对照 `javap -c`）

每个函数有一个 **code object**（`func.__code__`），里面装着字节码、常量表、变量名表等。用标准库 `dis` 反汇编看：

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
```

输出（简化）：
```
  RESUME                   0
  LOAD_FAST                0 (a)      # 把局部变量 a 压栈
  LOAD_FAST                1 (b)      # 把局部变量 b 压栈
  BINARY_OP                0 (+)      # 弹两个、相加、结果压栈
  RETURN_VALUE                        # 返回栈顶
```

> CPython 是**栈式虚拟机**（和 JVM 一样基于操作数栈，不是寄存器机）。`LOAD_FAST` 压栈、`BINARY_OP` 运算、`RETURN_VALUE` 返回——和 JVM 的 `iload`/`iadd`/`ireturn` 一一对得上。差别在于 JVM 字节码是强类型的（`iadd` 专管 int），而 CPython 的 `BINARY_OP +` 是**多态**的：运行时才看操作数类型去找 `__add__`，这正是动态类型的代价。

实战价值：想知道两种写法谁更省指令、`a += 1` 和 `a = a + 1` 编出来一不一样、某个语法糖背后发生了什么——`dis` 一看便知。

### 2.3 PyObject：万物的对象头

CPython 里**每个对象**在 C 层都是一个 `PyObject`，头部至少有两个字段：

```c
typedef struct {
    Py_ssize_t ob_refcnt;     // 引用计数（第 04 章的主角）
    PyTypeObject *ob_type;    // 指向类型对象的指针（决定行为）
} PyObject;
```

- `ob_refcnt` 就是上一章的引用计数。
- `ob_type` 指向类型对象，所有方法/运算都通过它分派——这就是"一切皆对象"和鸭子类型的 C 层根基。

后果（Java 老手务必理解）：**Python 没有基本类型**。Java 的 `int` 是 4 字节裸值放栈上；Python 的 `int 5` 是一个堆上的 `PyLongObject`，带对象头 + 实际数值，**一个小整数对象要 28 字节**。这就是为什么：
- Python 数值运算慢（每次都要拆装对象、查类型、分派方法）。
- 一个 `list` 存的是**指针**，不是连续的值（对照 Java 的 `int[]` 是连续内存）——所以数值计算要快必须用 `numpy`（底层连续的 C 数组）。

### 2.4 名字查找：LOAD_FAST vs LOAD_GLOBAL（性能相关）

字节码里你会看到不同的加载指令，速度差很多：
- `LOAD_FAST`：局部变量，**数组索引**访问，最快。
- `LOAD_GLOBAL`：全局/内置名字，要查模块 dict（和内置 dict），**慢**。

```python
# 热循环里反复 LOAD_GLOBAL 查 len 会慢；把它绑成局部变量能加速
def hot_loop(data):
    _len = len            # 提成局部，循环里变 LOAD_FAST
    total = 0
    for d in data:
        total += _len(d)
    return total
```

这是 Python 经典微优化，背后就是字节码层面 FAST vs GLOBAL 的区别。平时不必这么写，但热点优化时是真实手段。

### 2.5 GIL：到底锁的是什么

**GIL（Global Interpreter Lock）= 一把保护解释器内部状态的全局互斥锁。同一时刻只有持有 GIL 的线程能执行 Python 字节码。**

为什么存在？因为引用计数（`ob_refcnt` 的 +1/-1）不是原子的，多线程同时改会导致计数错乱、内存崩坏。GIL 用"一把大锁"简单粗暴地保证了引用计数和解释器状态的线程安全——代价是牺牲了多核并行。

机制要点：
- 线程执行一段字节码后会**周期性释放 GIL**（按时间片，默认约 5ms，`sys.getswitchinterval()`），让别的线程有机会跑。
- **执行阻塞式 I/O（读文件、网络、sleep）时会主动释放 GIL** → 所以**多线程对 I/O 密集任务有效**（一个线程等 I/O 时别的线程能跑）。
- **C 扩展做纯计算时可以主动释放 GIL**（numpy 大量这么做）→ 所以 numpy 的计算能并行。
- 但**纯 Python 的 CPU 密集任务**全程持 GIL → 多线程不能加速，只能多进程。

```python
# 验证：CPU 密集任务多线程不加速
import threading, time

def count(n):
    while n > 0:
        n -= 1

# 单线程
start = time.perf_counter()
count(50_000_000); count(50_000_000)
print("串行:", time.perf_counter() - start)

# 双线程（受 GIL 限制，几乎不快甚至更慢）
start = time.perf_counter()
t1 = threading.Thread(target=count, args=(50_000_000,))
t2 = threading.Thread(target=count, args=(50_000_000,))
t1.start(); t2.start(); t1.join(); t2.join()
print("双线程:", time.perf_counter() - start)
```

> **前沿**：Python 3.13 引入了实验性的 **free-threaded（no-GIL）** 构建，去掉 GIL、让多线程真并行（用更细粒度的锁 + 偏向引用计数等技术保护对象）。还在实验阶段、默认不开、生态适配中。这是 Python 并发模型未来几年最大的变化，值得关注。第 18 章细谈并发取舍。

### 2.6 为什么 Python 慢（以及怎么补救）

把前面串起来，Python 比 Java 慢 10~100 倍的原因：
1. **纯解释、无 JIT**：每条字节码都要解释循环分派，没有 JVM 那样把热点编成机器码。
2. **动态类型分派**：每个 `+`、每次属性访问都要运行时查类型、找方法，无法像 Java 那样编译期定死。
3. **一切皆堆对象**：没有栈上基本类型，数值运算不断拆装对象、查类型。
4. **GIL**：多核并行受限。

补救手段（这正是 Python 在数据/AI 领域反而快的秘密——**把热点下沉到 C**）：
- **numpy/pandas**：数组运算在连续内存的 C 层做，一次调用处理整个数组（向量化），且常释放 GIL。
- **C 扩展 / Cython / Rust（PyO3）**：把热点函数用原生语言写，Python 只做胶水。
- **PyPy**：带 JIT 的替代解释器，纯 Python 代码常快几倍。
- **3.11+ 的解释器加速**（自适应特化字节码 Specializing Adaptive Interpreter，PEP 659）和 **3.13 实验性 JIT**：CPython 自身在变快。

> 心智模型：**Python 的定位是"胶水语言"**——用它表达逻辑和调度，把计算密集部分交给 C 层的库。"Python 慢"在 Web/脚本/调度场景根本不是瓶颈（瓶颈是 I/O 和数据库），在数值场景靠 numpy 绕开。理解这个分工，就不会用错语言。

### 2.7 C 扩展与 FFI（对照 JNI）

Python 调原生代码的几条路，按从底到高：
- **C API**：直接用 CPython 的 C 接口写扩展模块，最底层、最快、最繁琐（numpy 这类库走这条）。
- **ctypes**（标准库）：运行时加载 `.so`/`.dll` 调函数，无需编译，类比 JNA。
- **cffi**：更友好的 C 接口绑定。
- **Cython**：写类 Python 的语法、编译成 C 扩展，渐进加类型提速。
- **PyO3 / Rust**：现代项目流行用 Rust 写扩展（pydantic v2、polars、ruff、uv 都是 Rust）。

对照 JNI：思路一样（跨语言调用、要管内存和引用计数），但 Python 的 C API 要求你在 C 层手动维护 `Py_INCREF`/`Py_DECREF`（引用计数），这是写扩展最易错的地方。

---

## ④ 使用方法与最佳实践

1. **用 `dis` 验证猜想**，而不是凭感觉判断哪种写法快。微优化前先看字节码。
2. **先 profile 再优化**（`cProfile`、`timeit`，第 22 章）。Python 性能问题常在意想不到的地方。
3. **CPU 密集 → numpy 向量化 / 多进程 / C 扩展**；别指望多线程或微优化语法。
4. **I/O 密集 → 多线程或 asyncio**（GIL 在 I/O 时释放，并发有效）。
5. 把 Python 当**胶水**：逻辑用 Python 写得清楚，热点交给成熟 C 库，别用纯 Python 撸数值循环。
6. 关注版本：3.11/3.12/3.13 每代解释器都在显著提速，升级常是"免费"的性能收益。

---

## ⑤ ⚠️ 易错点（Java 思维翻车）

| Java 习惯 | 翻车现场 | 正确认知 |
|----------|---------|---------|
| 以为有 JIT 会把热点优化掉 | 纯 Python 数值循环慢得离谱还不理解 | CPython 无 JIT，每条字节码都在解释 |
| 以为 `int` 是栈上裸值 | 困惑于 100 万个数的 list 占内存巨大 | 每个 int 是 28 字节堆对象，list 存指针 |
| 以为多线程能用满多核 | 开 8 线程算数，CPU 只跑满一核 | GIL：CPU 密集用多进程/numpy |
| 以为 GIL 让 I/O 也串行 | 不敢用多线程做爬虫/网络 | I/O 时 GIL 释放，多线程对 I/O 有效 |
| 用纯 Python 写矩阵运算 | 慢到不可用 | 用 numpy 向量化，下沉到 C |
| 凭感觉做语法微优化 | 优化错地方，没效果 | 先 profile，再用 dis 验证 |
| 把 `.pyc`/`__pycache__` 当源码或提交 git | 仓库脏、误解为产物 | 它是字节码缓存，gitignore 掉 |

---

## ⑥ 练习题

### 基础理解
1. 用 `dis` 分别反汇编 `a + b` 和 `a += b`（在函数里），观察并解释指令差异。
2. 解释"为什么纯 Python 的 CPU 密集多线程不能加速，但用 numpy 做大数组运算的多线程却可能加速"——从 GIL 释放的角度回答。

### 找坑改错 / 实验
3. 下面函数对一个一千万元素的列表逐个平方求和，很慢。从本章原理解释慢的根因，并给出至少两种提速方向（不要求写完整代码）：
```python
def sum_squares(nums):     # nums 是 Python list，长度 1e7
    total = 0
    for n in nums:
        total += n * n
    return total
```
4. 一个 Java 同事坚持"Python 多开几个线程就能把 8 核 CPU 跑满做并行计算"。用一个最小实验（threading 跑 CPU 密集函数计时）证明他错了，并说明正确做法。

### 综合思考
5. 结合本章，解释为什么"Python 很慢"这个说法在 Web 后端和数据科学两个领域都**不太成立**，分别说明真正的性能由什么决定、Python 的角色是什么。

---

### 参考答案

**1.** `a + b`：`LOAD_FAST a` / `LOAD_FAST b` / `BINARY_OP +` / 结果留栈（若 return 则 `RETURN_VALUE`）。`a += b`：同样 `LOAD_FAST`×2 + `BINARY_OP +=`（就地加，`BINARY_OP` 带 inplace 变体）+ `STORE_FAST a`（把结果存回 a）。差异在于 `+=` 多一步 `STORE_FAST`，且对可变对象会调用就地操作（如 list 的 `__iadd__`）。

**2.** 纯 Python CPU 密集任务全程持有 GIL 执行字节码，多线程被 GIL 串行化，加不了速。numpy 的大数组运算在 C 层执行，运算期间**主动释放 GIL**，此时其它线程（包括另一个 numpy 计算）可并行推进，所以可能加速。

**3.** 慢的根因：①纯解释执行的 Python 循环，每次迭代都解释多条字节码；②每个 `n`、`n*n`、`total` 都是堆上 int 对象，`*`、`+=` 都要运行时类型分派、对象拆装；③list 存指针，访问要解引用。提速方向：
- **numpy 向量化**：`np.asarray(nums); arr.dot(arr)` 或 `(arr*arr).sum()`，整个运算在 C 层连续内存上做，快几十倍。
- **多进程**分块求和（CPU 密集，绕开 GIL），再汇总。
- **Cython / Numba** 把循环编成机器码。
- 内置 `sum(n*n for n in nums)` 略快于手写循环（少些字节码），但数量级提升仍要靠 numpy。

**4.** 实验：见 2.5 的代码——用 `threading` 跑两个 5000 万次递减循环，对比串行与双线程耗时，会发现双线程**不比串行快**（甚至更慢，因 GIL 切换开销）。正确做法：改用 `multiprocessing`/`concurrent.futures.ProcessPoolExecutor`，每个进程独立 GIL 真并行；或把计算交给释放 GIL 的库（numpy）。

**5.** Web 后端：瓶颈通常是数据库查询、网络 I/O、外部服务，而非 CPU；请求处理时间里 Python 解释开销占比很小，且 I/O 期间 GIL 释放、可用多线程/asyncio 高并发。所以"语言慢"基本不影响吞吐，Python 的角色是高效表达业务逻辑 + 调度 I/O。数据科学：真正的计算（矩阵、统计、训练）全在 numpy/pandas/PyTorch 的 C/CUDA 层，Python 只负责编排，"慢"的解释器循环根本不参与热点。两个领域里 Python 都是**胶水/调度层**，性能由底层 I/O 或 C/GPU 决定。

---

下一章：[06 内置数据结构](06_内置数据结构.md) — list/dict/set/tuple 对照集合框架，含底层实现与复杂度。
