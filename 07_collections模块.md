# 07 · collections 模块与标准数据结构 ⭐

> [06 内置数据结构](06_内置数据结构.md) 讲了 `list/dict/set/tuple` 这四件套——它们对标 Java 的 `ArrayList/HashMap/HashSet`。但 Java 的集合框架远不止这四个：你有 `ArrayDeque`、`PriorityQueue`、`record`、Guava 的 `Multiset`、原始类型数组 `int[]`……这些在 Python 里去哪了？答案是：**散落在标准库里**——`collections`、`heapq`、`bisect`、`array` 几个模块。本章把它们逐个对照 Java，深挖底层实现（deque 的分块双向链表、heapq 的数组二叉堆、namedtuple 的动态类生成），让你**用对数据结构消灭样板代码**。

---

## ① Java 对照表

| Python 工具 | 所在模块 | Java 对应物 | 一句话差异 |
|------------|---------|------------|-----------|
| `defaultdict` | `collections` | `Map.computeIfAbsent` / Guava `Multimap` | 缺失 key 自动用工厂建默认值，省掉"先查再建"样板 |
| `Counter` | `collections` | `Map<K,Long>` 计数 / Guava `Multiset` | 计数专用 dict，缺失 key 返回 0，`most_common` 直接出 Top-K |
| `deque` | `collections` | `ArrayDeque` / `LinkedList` | 两端 O(1)，可设 `maxlen` 做固定窗口；底层是分块双向链表 |
| `namedtuple` | `collections` | `record`（Java 16+）| 不可变值对象，是 `tuple` 子类，可下标可解包 |
| `OrderedDict` | `collections` | `LinkedHashMap` | 3.7 后普通 dict 已保序，它只剩少数专有方法 |
| `ChainMap` | `collections` | 多层 `Map` 叠加查找（无直接对应）| 把多个 dict 串成一个视图，按顺序查找，做配置覆盖 |
| `heapq` | `heapq` | `PriorityQueue` | **不是类是一组函数**，就地操作普通 list；**最小堆** |
| `bisect` | `bisect` | `Collections.binarySearch` + `List.add` | 在有序 list 上二分查找/插入，维护有序序列 |
| `array` | `array` | `int[]` / `double[]` 基本类型数组 | 紧凑存 C 数值，省内存；数据量大时用，否则上 numpy |

> 记忆框架：**`collections` 是"高级 Map/容器"，`heapq`+`bisect` 是"在 list 上做算法"，`array` 是"省内存的原始类型数组"。**

---

## ② 设计理念（为什么 Python 这么设计）

### 2.1 为什么这些不是内置，而是放在标准库？

Java 把 `ArrayDeque`、`PriorityQueue` 和 `ArrayList` 平等地放在 `java.util` 里，你都得 `import`。Python 的分界线不一样：**只有"语言语法层面要用到的"才是内置（built-in）**。

- `list/dict/set/tuple` 有**专用字面量语法**（`[]`、`{}`、`()`、`{k: v}`），编译器要认识它们，解释器启动就得在，所以是内置类型。
- `deque/Counter/heapq` 没有字面量、不参与语法，纯粹是"库"，于是放进标准库按需 `import`。这跟 Java 把它们都塞进 `java.util` 只是**打包粒度不同**——Python 倾向"核心语言尽量小，能力靠库扩展"（这是 Python 之禅"简洁"哲学的体现，见 [01 设计哲学](01_设计哲学与心智模型.md)）。

所以你会看到一个 Java 老手常困惑的现象：**Python 没有内置的队列/优先队列类型**。不是没有，是它们叫 `collections.deque` 和 `heapq`，得 import。

### 2.2 "用对数据结构消灭样板代码"

`collections` 模块的设计哲学，一句话概括：**把高频出现的"组合套路"固化成一个类型，让你少写胶水代码**。对照 Java 你会立刻有共鸣：

```java
// Java：按首字母分组，经典的 computeIfAbsent 样板
Map<Character, List<String>> groups = new HashMap<>();
for (String word : words) {
    groups.computeIfAbsent(word.charAt(0), k -> new ArrayList<>()).add(word);
}
```

```python
# Python：defaultdict 把"缺失就建空 list"内化掉
from collections import defaultdict
groups = defaultdict(list)
for word in words:
    groups[word[0]].append(word)   # 缺失 key 自动 = list()，直接 append
```

`computeIfAbsent` 是 Java 8 才补上的"消除样板"利器；`defaultdict` 是 Python 2.5 就有的同一思路。`Counter` 更进一步：它知道你"计数"这个意图，于是 `most_common()`、`+`/`-`/`&`/`|` 集合式运算全给你备好了，对标 Guava `Multiset` 的设计动机完全一致——**让数据结构携带领域语义，而不是让你每次手搓**。

> 心智模型：在 Java 里你习惯"`HashMap` + 一堆 if/computeIfAbsent 拼出意图"；在 Python 里先问"标准库有没有现成的语义类型"。选对了，代码量直接减半，还更不容易写错。

---

## ③ 实现细节（底层怎么实现的）★本章重点★

这一段是和普通教程拉开差距的地方。我们逐个拆开看 CPython 里它们到底长什么样。

### 3.1 `deque`：分块双向链表（block of 64）

Java 的 `ArrayDeque` 是**环形数组**（circular buffer）：一块连续数组 + head/tail 两个游标，满了就扩容到 2 倍。Python 的 `deque` 走的是**另一条路——分块双向链表**。

CPython 源码（`Modules/_collectionsmodule.c`）里，deque 由一串 **block** 组成，每个 block 是一个定长小数组：

```c
#define BLOCKLEN 64                 // 每块 64 个槽位
typedef struct BLOCK {
    struct BLOCK *leftlink;         // 指向前一块
    PyObject *data[BLOCKLEN];       // 64 个对象指针
    struct BLOCK *rightlink;        // 指向后一块
} block;
```

结构示意：

```
   leftmost                              rightmost
      ↓                                      ↓
  ┌─block─┐    ┌─block─┐    ┌─block─┐
  │ [..][x]│←→│[64个] │←→│[y][..] │
  └───────┘    └───────┘    └───────┘
   左端有空槽   中间装满       右端有空槽
```

- **两端 push/pop 是 O(1)**：在 leftmost/rightmost block 的空槽里写/读；块满了就 `malloc` 一个新 block 链上去，块空了就摘掉。**不需要像环形数组那样整体扩容搬数据**。
- **中间访问 / `insert` 是 O(n)**：要从一端沿链表走过去（CPython 做了优化，从较近的一端开始，但仍是线性）。
- 用定长 block（而不是每个元素一个链表节点）是**缓存友好**的折中：减少了指针开销和 `malloc` 次数，64 个元素挤在一块连续内存里，遍历对 CPU cache 友好。

对照总结：

| 维度 | `ArrayDeque`（环形数组） | `collections.deque`（分块双向链表） |
|------|----------------------|--------------------------------|
| 两端操作 | 摊还 O(1)，扩容时 O(n) | **稳定 O(1)，无整体扩容** |
| 随机访问 `dq[i]` | O(1)（数组下标） | **O(n)**（沿链表走） |
| 内存 | 一块连续数组，可能有空洞 | 多个 64 槽 block 链起来 |
| `maxlen` 支持 | 无 | **有**（满了自动挤掉另一端） |

> 关键认知：`deque` 是**队列/栈/滑动窗口**专用，不是给你随机下标访问的。要 `dq[i]` 频繁取中间元素，说明你该用 `list`。

### 3.2 `heapq`：普通 list 上的二叉堆（无独立类型）

Java 的 `PriorityQueue` 是一个**类**，内部藏着一个数组堆，你看不见它。Python 的 `heapq` 反过来——**它没有堆类型，只有一组函数，直接操作你给的普通 `list`**。这个 list 就是堆本身。

二叉堆用数组表示，下标关系是经典的：

```
       下标 0
      /      \
   下标1      下标2
   /   \      /
 下标3 下标4 下标5

父 i  → 左子 2i+1，右子 2i+2
子 i  → 父 (i-1)//2
堆性质（最小堆）：heap[parent] <= heap[child]
```

所以一个 `[1, 3, 2, 7, 5]` 这样的普通 list，只要满足"每个父节点 ≤ 子节点"，它就**是**一个合法的最小堆，`heapq` 的函数直接在上面 `siftup`/`siftdown`：

```python
import heapq

heap = [5, 3, 8, 1, 9, 2]
heapq.heapify(heap)          # 原地建堆，O(n)
print(heap)                  # → [1, 3, 2, 5, 9, 8]  注意只是堆序，不是全排序
heapq.heappush(heap, 0)      # O(log n)：append 到末尾再 siftdown 上浮
print(heapq.heappop(heap))   # → 0  弹出最小，O(log n)
```

CPython 的 `heappop` 实现思路（`Modules/_heapqmodule.c`）：取走 `heap[0]`（最小），把末尾元素挪到 0 号位，再 `_siftup` 下沉到正确位置——和教科书一模一样。

关键差异：

- **最小堆（min-heap）**：堆顶永远是最小值。Java 的 `PriorityQueue` 默认也是小顶（自然序），这点一致。要最大堆？**取负数**（见 ④/⑤）。
- **没有"堆类型"**：因为是裸 list，你能 `len(heap)`、`heap[0]` 偷看堆顶、遍历——但**只有下标 0 保证是最小，其余顺序无意义**，别误把 `heap` 当排序结果。
- 它是**就地（in-place）算法库**的典型 Python 风格：把数据结构和操作解耦，函数作用在通用容器上。

### 3.3 `bisect`：在有序 list 上二分（维护有序序列）

`bisect` 对标 `Collections.binarySearch`，但更进一步——它不只查，还帮你**插入并维持有序**。底层就是教科书二分查找，作用在一个**你保证有序**的普通 list 上：

```python
import bisect

scores = [60, 70, 75, 90]
i = bisect.bisect_left(scores, 75)   # → 2  第一个 >=75 的位置
bisect.insort(scores, 80)            # 二分定位 + list.insert，保持有序
print(scores)                        # → [60, 70, 75, 80, 90]
```

- `bisect_left/bisect_right`：纯二分查找，**O(log n)** 找到插入点。
- `insort`：找到点后调 `list.insert`，**插入本身是 O(n)**（list 要搬移后续元素，见 [06 内置数据结构](06_内置数据结构.md) 的 list 底层）。所以 `bisect` 适合"查多插少"或中等规模；高频插入有序数据要考虑别的结构（如平衡树，Python 标准库没有，可用第三方 `sortedcontainers`）。

### 3.4 `Counter` / `defaultdict` / `OrderedDict`：都是 dict 的子类

这三个都**继承自 `dict`**，复用了 [06](06_内置数据结构.md) 讲的开放寻址哈希表，只是加了行为：

- **`defaultdict`**：多存一个 `default_factory`（一个无参可调用）。当 `__getitem__` 触发 `KeyError` 时，CPython 调 `__missing__`，用工厂造个默认值**塞进去再返回**。注意"塞进去"——读一个不存在的 key 会**真的新增条目**（易错点，见 ⑤）。
- **`Counter`**：也是 dict 子类，键是元素、值是计数。它重写了 `__missing__` 让缺失 key 返回 `0`（**不抛 KeyError，也不新增条目**——和 defaultdict 不同）。`most_common(k)` 内部用的正是 **`heapq.nlargest`**，所以求 Top-K 是 `O(n log k)`，不是全排序。
- **`OrderedDict`**：3.7 之前 dict 不保证顺序，它靠**额外的双向链表**记录插入顺序。3.7 起普通 dict 已经保插入序（语言规范级保证），`OrderedDict` 大部分场景被取代，只在需要 `move_to_end()`、顺序敏感的 `==` 比较、`popitem(last=False)` 时才用。

```python
from collections import Counter
c = Counter("mississippi")
print(c)                 # → Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
print(c.most_common(2))  # → [('i', 4), ('s', 4)]  底层走 heapq.nlargest
print(c["xyz"])          # → 0  缺失返回 0，不报错、不新增条目
```

### 3.5 `namedtuple`：动态生成的 tuple 子类（`__slots__ = ()` 省内存）

这是最有"黑魔法"味道的一个。`namedtuple("Point", "x y")` **在运行时动态生成一个新的类**，它是 `tuple` 的子类。机制上类似 Java 注解处理器/Lombok 生成 `record` 样板，只不过 Python 是**运行期**用字符串拼出类定义再 `exec` 出来的。

关键设计：

- 它是 **`tuple` 子类**，所以实例**就是个 tuple**——可下标 `p[0]`、可解包 `x, y = p`、不可变、可哈希、能当 dict key，全继承自 tuple。额外给你 `p.x`/`p.y` 命名访问（靠 `property` + `operator.itemgetter` 实现）。
- 类上声明 **`__slots__ = ()`**：这意味着实例**不创建 `__dict__`**（普通对象那个存属性的字典，见 [04 运行时与内存模型](04_运行时与内存模型.md)）。数据全存在 tuple 自身的紧凑数组里，所以**内存和裸 tuple 几乎一样省**——这正是它能当"轻量值对象"的底气。

实测验证：

```python
from collections import namedtuple
import sys

Point = namedtuple("Point", ["x", "y"])
p = Point(1, 2)

print(Point.__slots__)                      # → ()  没有 __dict__
print([c.__name__ for c in Point.__mro__])  # → ['Point', 'tuple', 'object']
print(sys.getsizeof(p), sys.getsizeof((1, 2)))  # → 56 56  和裸 tuple 等大
print(p.x, p[0])                            # → 1 1  命名访问和下标访问都行
x, y = p                                    # 解包，因为它就是 tuple
print(p._asdict())                          # → {'x': 1, 'y': 2}
```

对比：普通 class 实例每个都背一个 `__dict__`（几十~上百字节），而 namedtuple 实例只有 tuple 的开销。这就是"百万个小值对象"时 namedtuple 比普通类省内存的原因（更极致的用 `__slots__` 类或 `dataclass(slots=True)`，[14 类型系统](14_类型系统与注解.md) 细讲）。

### 3.6 `array`：紧凑的 C 数值（对照 list 存指针）

[05 CPython 内部](05_CPython内部.md) 讲过：Python 的 `list` 存的是**指针**，每个元素指向一个堆上的 `PyObject`（一个 `int` 对象 28 字节）。`array` 模块给你**真正的 C 类型连续数组**——像 Java 的 `int[]`/`double[]`，元素是裸值，紧凑排布。

```python
import array
import sys

a = array.array("i", [1, 2, 3])   # 'i' = C int（4字节）
print(sys.getsizeof(a))            # → 76（数组对象头 + 3*4 字节连续数据）
print(sys.getsizeof([1, 2, 3]))    # → 120（list 头 + 3 个指针，还没算 int 对象本身）
```

`array` 用**类型码（typecode）**指定元素的 C 类型：

| typecode | C 类型 | 字节 | 对应 Java |
|----------|--------|------|-----------|
| `'b'`/`'B'` | signed/unsigned char | 1 | `byte` |
| `'i'`/`'I'` | int | 4 | `int` |
| `'q'`/`'Q'` | long long | 8 | `long` |
| `'f'` | float | 4 | `float` |
| `'d'` | double | 8 | `double` |

> 定位：`array` 是"省内存的同构数值序列"，但**没有向量化运算**（不能 `a * 2` 整体操作）。数据/AI 场景几乎都跳过它直接上 **numpy**（连续内存 + 向量化 + 释放 GIL，见 [05](05_CPython内部.md)）。`array` 的实际用武之地：需要大量同类型数值、又不想引入 numpy 依赖、或要和 C 库做二进制 I/O（`array.tobytes()`）时。

### 3.7 复杂度速查表

| 操作 | `list` | `deque` | `dict`/`defaultdict`/`Counter` | `heapq`(on list) | `bisect`(on list) |
|------|--------|---------|-------------------------------|------------------|-------------------|
| 头部插入/删除 | **O(n)** | **O(1)** | — | — | — |
| 尾部 append/pop | O(1)摊还 | O(1) | — | push/pop O(log n) | — |
| 随机下标 `x[i]` | O(1) | **O(n)** | — | O(1) 仅堆顶 | O(1) |
| 按 key 查 | — | — | O(1) 平均 | — | 查找 O(log n) |
| 取最小/最大 | O(n) | O(n) | — | **堆顶 O(1)** | 端点 O(1) |
| 有序插入 | — | — | — | — | 定位O(log n)+插O(n) |
| Top-K | 排序 O(n log n) | — | `most_common` O(n log k) | `nlargest` O(n log k) | — |

---

## ④ 使用方法与最佳实践

### 4.1 `defaultdict`：分组、计数、累加的样板杀手

```python
from collections import defaultdict

# 分组（对照 Java Stream.groupingBy）：把订单按用户聚合
orders = [("alice", 100), ("bob", 50), ("alice", 30)]
by_user = defaultdict(list)
for user, amount in orders:
    by_user[user].append(amount)
print(dict(by_user))     # → {'alice': [100, 30], 'bob': [50]}

# 累加：用 int 工厂（int() == 0）做求和计数器
totals = defaultdict(int)
for user, amount in orders:
    totals[user] += amount      # 缺失 key 自动从 0 开始
print(dict(totals))      # → {'alice': 130, 'bob': 50}
```

> 工厂可以是任意无参可调用：`list`、`int`、`set`、`dict`、`lambda: ...`。`defaultdict(lambda: defaultdict(list))` 可建嵌套结构。

### 4.2 `Counter`：词频、Top-K、集合式运算

```python
from collections import Counter

logs = ["GET", "POST", "GET", "GET", "DELETE", "POST"]
hits = Counter(logs)                 # 一行完成计数
print(hits.most_common(2))           # → [('GET', 3), ('POST', 2)]  Top-K

# Counter 支持多重集（multiset）运算，对照 Guava Multiset
a = Counter("aabbc")
b = Counter("abbbd")
print(a + b)   # → Counter({'b':5,'a':3,'c':1,'d':1})  计数相加
print(a - b)   # → Counter({'a':1,'c':1})  相减（只留正数）
print(a & b)   # → Counter({'a':1,'b':2})  交集取 min
print(a | b)   # → Counter({'b':3,'a':2,'c':1,'d':1})  并集取 max
```

### 4.3 `deque`：队列 / 栈 / 滑动窗口 / `maxlen`

```python
from collections import deque

# 当队列（FIFO）：左进右出 或 右进左出。绝不要用 list.pop(0)！
q = deque()
q.append("task1"); q.append("task2")
print(q.popleft())       # → task1  O(1) 出队

# maxlen：固定容量环形缓冲，满了自动挤掉另一端——天然滑动窗口/最近N条
recent = deque(maxlen=3)
for i in range(5):
    recent.append(i)
print(recent)            # → deque([2, 3, 4], maxlen=3)  只留最近 3 个

# rotate：循环移位，list 没有的便利
d = deque([1, 2, 3, 4, 5])
d.rotate(2)              # 右移 2 位
print(d)                 # → deque([4, 5, 1, 2, 3])
```

> `maxlen` 是 `ArrayDeque` 没有的杀手锏：做"最近 N 条日志"、"滑动窗口求和"、"限流时间戳缓冲"时，一个参数解决，不用手写淘汰逻辑。

### 4.4 `heapq`：Top-K、优先队列、多路归并

```python
import heapq

# Top-K：求最大 3 个，O(n log k) 不需要全排序
nums = [5, 1, 8, 3, 9, 2, 7]
print(heapq.nlargest(3, nums))    # → [9, 8, 7]
print(heapq.nsmallest(2, nums))   # → [1, 2]

# 优先队列：存 (priority, item)，按 priority 出队
pq = []
heapq.heappush(pq, (2, "中优先级"))
heapq.heappush(pq, (1, "高优先级"))
heapq.heappush(pq, (3, "低优先级"))
print(heapq.heappop(pq))          # → (1, '高优先级')  最小优先级先出

# 多路归并：合并多个已排序序列（惰性，省内存），对照归并外排
a = [1, 4, 7]; b = [2, 5, 8]; c = [3, 6, 9]
print(list(heapq.merge(a, b, c))) # → [1,2,3,4,5,6,7,8,9]
```

> 优先队列实战技巧：元组按字典序比较，若 priority 相同会去比第二个元素（可能报 TypeError，若 item 不可比较）。稳妥做法是塞入 `(priority, 自增序号, item)`，用序号打破平局。

### 4.5 `bisect`：有序插入与"区间映射"

`bisect` 有个 Java 程序员常忽略的妙用——**把连续数值映射到档位**（成绩转等级、价格转区间、限流分桶）：

```python
import bisect

# 分数 → 等级：breakpoints 是有序边界
def grade(score, breakpoints=[60, 70, 80, 90], grades="FDCBA"):
    i = bisect.bisect_right(breakpoints, score)
    return grades[i]

print([grade(s) for s in [55, 65, 75, 85, 95]])
# → ['F', 'D', 'C', 'B', 'A']   一次二分搞定区间映射，无需一串 if-elif
```

### 4.6 `namedtuple` / `typing.NamedTuple`：轻量不可变值对象

```python
from typing import NamedTuple

# 现代写法：class 风格 + 类型注解（3.6+），对照 Java record
class Order(NamedTuple):
    id: int
    user: str
    amount: float = 0.0           # 支持默认值

o = Order(1, "alice", 99.5)
print(o.user, o.amount)           # → alice 99.5
print(o._replace(amount=120.0))   # → Order(id=1, user='alice', amount=120.0)  返回新实例
# o.amount = 5                     # ✗ AttributeError：不可变
```

> 选型：纯不可变值对象、要解包/当 dict key → `NamedTuple`；需要可变字段、方法、`__post_init__` 校验 → `@dataclass`（[14 类型系统](14_类型系统与注解.md)）。`NamedTuple` ≈ Java `record`，`dataclass` ≈ 普通 POJO/Lombok。

### 4.7 `ChainMap`：配置叠加（覆盖优先级）

```python
from collections import ChainMap

defaults = {"host": "localhost", "port": 8080, "debug": False}
env      = {"port": 9090}
cli      = {"debug": True}

# 查找顺序：cli → env → defaults，前面的覆盖后面的（不复制、是视图）
config = ChainMap(cli, env, defaults)
print(config["port"])    # → 9090  来自 env，覆盖了 defaults
print(config["debug"])   # → True  来自 cli
print(config["host"])    # → localhost  回落到 defaults
```

> 对照 Spring 的多层 `PropertySource` 覆盖。`ChainMap` 是**视图不是合并**——底层 dict 改了，它跟着变；写操作只作用于第一个 dict。比 `{**defaults, **env, **cli}` 合并字典更省内存、且能动态反映底层变化。

---

## ⑤ ⚠️ 易错点（Java 思维翻车）

| Java 习惯 | 翻车现场 | 正确做法 / 认知 |
|----------|---------|----------------|
| 用 `list.pop(0)` / `list.insert(0,x)` 当队列 | 每次 O(n) 搬移整个列表，万级数据就卡死 | 队列/栈两端操作一律用 `collections.deque`（O(1)） |
| 以为读 dict 不会改它 | `defaultdict[missing]` **会新增一个条目**，遍历/`len` 后莫名多 key | 只想查不想建用 `d.get(k, default)` 或普通 dict；清楚 `d[k]` 对 defaultdict 有副作用 |
| 期待缺失 key 抛 `KeyError`（像 Java `get` 返 null 后你 NPE） | `Counter[missing]` 静默返回 `0`，逻辑里少了"不存在"分支 | 知道 Counter 缺失即 0；要判存在用 `k in counter` |
| 以为 heap 是最大堆 / 是排好序的 | `heapq` 是**最小堆**，且 list 只有 `[0]` 是最小，其余无序 | 最大堆把值**取负** push/pop（或用 `nlargest`）；别把 heap 当 sorted 结果遍历 |
| 把 namedtuple 当可变 POJO | `point.x = 5` 抛 `AttributeError`，它不可变 | 改值用 `._replace()` 返回新实例；要可变用 `dataclass` |
| 以为 `OrderedDict` 才保序 | 3.7+ 普通 dict 已保插入序，多此一举地到处用 OrderedDict | 一般 dict 即可；只在要 `move_to_end`/顺序敏感 `==` 时才用 OrderedDict |
| 优先队列直接塞自定义对象 | `heappush(pq, my_obj)` 报 `TypeError: not supported between instances` | 塞 `(priority, 序号, obj)` 元组，让堆比较 priority；序号打破平局 |
| 大量数值用 `list` 省事 | 百万级数值 list 内存爆炸（每个 int 28 字节 + 指针） | 同构数值用 `array` 省内存；要计算直接上 numpy |

---

## ⑥ 练习题

### 基础理解

**1.** 解释为什么 `collections.deque` 的两端 push/pop 是 O(1) 而中间随机访问 `dq[i]` 是 O(n)，从它的底层数据结构（分块双向链表）说明；并对比 Java `ArrayDeque`（环形数组）在这两项上的复杂度。

**2.** `defaultdict(int)` 和 `Counter` 都能计数，行为有什么关键区别？分别说明：访问一个**不存在的 key** 时各自会发生什么（返回值、是否新增条目、是否报错）。

### 找坑改错

**3.** 下面这段"Java 味"的 Python 代码想实现一个任务队列，但在生产高负载下 CPU 飙升、吞吐崩塌。指出根因并改写：

```python
class TaskQueue:
    def __init__(self):
        self.tasks = []
    def submit(self, task):
        self.tasks.append(task)
    def next(self):
        if self.tasks:
            return self.tasks.pop(0)    # 取出最早的任务
        return None
```

**4.** 下面代码想用最小堆维护"实时 Top-3 最高分玩家"，但结果完全不对（拿到的是最低分）。指出两个问题并修复：

```python
import heapq

def top3_players(scores):           # scores: list[(name, score)]
    heap = []
    for name, score in scores:
        heapq.heappush(heap, (score, name))
    return [heapq.heappop(heap) for _ in range(3)]
```

### 综合实战

**5.** （后端/数据场景）你有一个 Nginx 访问日志文件，每行末尾是请求路径。写一个函数 `top_k_paths(path, k)`，统计访问量最高的 K 个路径并返回 `[(path, count), ...]`。要求：用 `Counter` 计数、用高效方式取 Top-K（不要对全部路径排序），并说明你用的 API 的时间复杂度。

```python
# 日志行形如：'127.0.0.1 - - [28/Jun/2026:10:00:00] "GET /api/users HTTP/1.1" 200'
def top_k_paths(filepath: str, k: int) -> list[tuple[str, int]]:
    ...
```

---

### 参考答案

**1.** `deque` 由一串定长 64 槽的 block 用前后指针连成双向链表。两端各记着 leftmost/rightmost block 和块内偏移，push/pop 只是在端点 block 的空槽读写、或摘挂一个 block，恒为 **O(1)**，且不像数组那样需要整体扩容搬数据。但要访问中间第 i 个元素，得从某一端沿链表逐块走过去定位，是 **O(n)**。Java `ArrayDeque` 是环形数组：两端操作靠 head/tail 游标也是摊还 O(1)（扩容时那一次 O(n)），但随机访问 `dq[i]` 是数组下标 **O(1)**——这是两者最大的区别：deque 牺牲随机访问换取稳定的两端性能和 `maxlen` 能力。

**2.** 关键区别在"缺失 key"的副作用：
- `defaultdict(int)`：访问不存在的 key 会触发 `__missing__`，用工厂 `int()`（=0）**新建一个条目存进字典**并返回 0。即"读"会改变字典——之后 `len()` 变大、`in` 为真。
- `Counter`：访问不存在的 key 返回 `0`，但**不新增条目、不报错**（它重写 `__missing__` 只返回 0 不写入）。`len()` 不变、`k in counter` 仍为假。
- 两者都不抛 `KeyError`（普通 dict 才抛）。所以纯计数累加两者都行，但若你会"试探性地读很多可能不存在的 key"，`defaultdict` 会悄悄塞满垃圾条目，`Counter` 不会。

**3.** 根因：`self.tasks.pop(0)` 从 list **头部**删除，list 底层是连续数组，删头要把后面所有元素**整体前移**，单次 O(n)，N 个任务出队总成本 O(n²)。高负载下队列越长越慢。改用 `deque`，头部出队 O(1)：

```python
from collections import deque

class TaskQueue:
    def __init__(self):
        self.tasks = deque()
    def submit(self, task):
        self.tasks.append(task)        # 右端入队 O(1)
    def next(self):
        return self.tasks.popleft() if self.tasks else None   # 左端出队 O(1)
```

**4.** 两个问题：
1. **最小堆方向反了**：`heapq` 是最小堆，堆顶是最小 score，`heappop` 三次拿到的是**最低**的 3 个。要最高分，把 score **取负**入堆（或改用 `heapq.nlargest`）。
2. **全量入堆再弹效率低**且没考虑只要 Top-3。更地道的是直接 `nlargest`，O(n log k)。

```python
import heapq

def top3_players(scores):
    # 方案 A：nlargest，按 score 排（key 指定比较字段），O(n log k)
    return heapq.nlargest(3, scores, key=lambda ns: ns[1])

# 方案 B：若坚持手动堆，score 取负
def top3_players_heap(scores):
    heap = [(-score, name) for name, score in scores]
    heapq.heapify(heap)
    return [(name, -negscore) for negscore, name in (heapq.heappop(heap) for _ in range(3))]
```

**5.** 用 `Counter` 累计，`most_common(k)` 取 Top-K（其内部用 `heapq.nlargest`，复杂度 **O(n log k)**，n 为不同路径数，远优于全排序 O(n log n)）：

```python
import re
from collections import Counter

def top_k_paths(filepath: str, k: int) -> list[tuple[str, int]]:
    counter = Counter()
    # 匹配 "METHOD /path HTTP/x.y" 里的 path
    pattern = re.compile(r'"(?:GET|POST|PUT|DELETE|PATCH|HEAD)\s+(\S+)\s+HTTP')
    with open(filepath, encoding="utf-8") as f:   # 逐行读，省内存（对照大文件）
        for line in f:
            m = pattern.search(line)
            if m:
                counter[m.group(1)] += 1          # 缺失 key 从 0 开始累加
    return counter.most_common(k)                  # Top-K，内部 heapq.nlargest，O(n log k)

# 复杂度：构建计数 O(总行数)；most_common(k) 是 O(U log k)，U=不同路径数。
# 相比 sorted(counter.items())[:k] 的 O(U log U)，k 远小于 U 时更快。
```

要点：(1) `Counter[key] += 1` 利用缺失即 0 的特性，省掉 `if key not in`；(2) `most_common(k)` 而非排序全部，体现"用对 API 选对复杂度"；(3) 逐行迭代文件而非 `readlines()`，应对大日志（呼应 [05](05_CPython内部.md) 的内存意识）。

---

下一章：[09 字符串、字节与编码](09_字符串字节与编码.md) — `str` vs `bytes`、Unicode、encode/decode、f-string，对照 `String`/`char`/`byte[]`/`Charset`。
