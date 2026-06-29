# 26 · 日期时间·序列化·pydantic ⭐

> 这一章把三件后端天天打交道、又最容易翻车的事讲透：**时间**、**序列化**、**运行时校验**。你在 Java 里有 `java.time`（`LocalDateTime`/`ZonedDateTime`/`Instant`/`Duration`）这套设计精良的现代时间 API，有 Jackson `ObjectMapper` 做 JSON、有 `Serializable` 做二进制、有 Bean Validation（`@NotNull`/`@Size`）做校验。来到 Python，对应物分别是 `datetime` 模块、`json`/`pickle` 模块、以及第三方的 `pydantic`。本章重点讲三个"形似神不同"的坑：① Python 的 `datetime` 有 **naive（无时区）vs aware（带时区）** 的分裂，两者相减直接抛异常，是分布式系统最隐蔽的 bug 源；② `json` 是跨语言文本格式、`pickle` 是 Python 私有二进制——**`pickle` 反序列化不可信数据等于远程代码执行（RCE）**，和 Java 反序列化漏洞同源；③ [14 章](14_类型系统与注解.md) 留的尾巴——"注解运行时不强制"——由 **pydantic v2**（Rust 核心）补上，它用你写的类型注解做真正的运行时校验。

---

## ① Java 对照表

| 概念 | Java (`java.time` / Jackson) | Python | 关键差异 |
|------|------|--------|---------|
| 本地日期时间（无时区） | `LocalDateTime` | `datetime`（**naive**，`tzinfo is None`） | 都是"墙上时间"，不指向时间轴上唯一一点 |
| 带时区日期时间 | `ZonedDateTime` | `datetime`（**aware**，带 `tzinfo`） | Python 用同一个 `datetime` 类，靠 `tzinfo` 字段区分 naive/aware |
| 时间轴上的瞬间（UTC） | `Instant` | aware `datetime`（`tzinfo=timezone.utc`） | Java 用独立类型强制区分；Python 全揉进 `datetime` |
| 时区标识 | `ZoneId.of("Asia/Shanghai")` | `ZoneInfo("Asia/Shanghai")`（3.9+） | 都读 IANA tz 数据库 |
| 固定偏移 | `ZoneOffset.ofHours(8)` | `timezone(timedelta(hours=8))` | 偏移 ≠ 时区（偏移不含夏令时规则） |
| 时间段 | `Duration`（秒/纳秒）/ `Period`（年月日） | `timedelta`（天/秒/微秒） | Python 只有一个 `timedelta`，精度到微秒（非纳秒） |
| 只有日期 | `LocalDate` | `date` | 一致 |
| 只有时间 | `LocalTime` | `time` | 一致 |
| 格式化/解析 | `DateTimeFormatter` | `strftime`/`strptime` / `isoformat`/`fromisoformat` | Python 的格式串是 C `strftime` 风格 |
| JSON 序列化 | Jackson `ObjectMapper` | `json` 模块 / pydantic | `json` 不认 `datetime`/`Decimal`，要 `default` 钩子 |
| Java 原生序列化 | `Serializable` + `ObjectOutputStream` | `pickle` | **都有反序列化 RCE 风险**，绝不读不可信数据 |
| Bean Validation | `@NotNull` `@Size` `@Email` + Hibernate Validator | **pydantic** `BaseModel` | Java 靠注解处理器 + 反射；pydantic 靠类型注解 + Rust 核心 |
| DTO / POJO 校验 | Spring `@Valid @RequestBody` | pydantic 模型（FastAPI 自动校验） | 见 [24 HTTP与Web框架](24_HTTP与Web框架.md) |
| 配置绑定 | `@ConfigurationProperties` | `pydantic-settings` `BaseSettings` | 从环境变量/`.env` 读配置并校验 |

> 一句话先记住三条铁律：**存时间一律用 aware/UTC**；**跨进程/跨语言传数据用 `json`，绝不用 `pickle` 读外部数据**；**外部输入的运行时校验交给 pydantic，注解本身不校验**。

---

## ② 设计理念（为什么 Python 这么设计）

### 2.1 `datetime` 的"naive vs aware 分裂"——一个历史包袱

Java 的 `java.time`（JSR-310，Java 8）是 Joda-Time 作者重新设计的，从类型上就把概念分干净了：

- `LocalDateTime` = 墙上时间，**没有**时区，"2026-06-28 12:00" 这串数字，不知道是北京还是纽约的 12 点。
- `ZonedDateTime` = 墙上时间 + 时区规则，指向时间轴上**唯一**一点。
- `Instant` = UTC 时间轴上的一个瞬间（纪元秒 + 纳秒）。

**类型系统逼你选**：你拿到一个 `LocalDateTime`，编译器就提醒你"这玩意没时区，别拿去比较绝对先后"。

Python 的 `datetime` 比 `java.time` 早十几年（2002 年左右成型），设计上**只有一个 `datetime` 类**，靠一个可选字段 `tzinfo` 区分两种语义：

- `tzinfo is None` → **naive**（朴素的），等价于 Java 的 `LocalDateTime`。
- `tzinfo` 是某个 `tzinfo` 子类实例 → **aware**（有意识的），等价于 `ZonedDateTime`/`Instant`。

```python
from datetime import datetime, timezone

naive = datetime(2026, 6, 28, 12, 0)                       # naive：墙上时间
aware = datetime(2026, 6, 28, 12, 0, tzinfo=timezone.utc)  # aware：UTC 时间轴上一点
print(naive.tzinfo)   # 输出: None
print(aware.tzinfo)   # 输出: UTC
```

**致命取舍**：因为是同一个类，Python **没法用类型系统拦住你**把 naive 和 aware 混用。两者相减、比较会在运行时抛 `TypeError`（见 ③ 3.2），但前提是你真的混用了——很多代码在测试时全用 naive 跑得好好的，一接入真实多时区数据才爆。这正是 Java 工程师最需要改肌肉记忆的地方：**Java 靠类型分，Python 靠纪律分**。

> 业界共识（也是本章反复强调的）：**程序内部一律用 aware + UTC，只在最外层显示给用户时才转本地时区**。这和 [09 章](09_字符串字节与编码.md) 的"边界处 encode/decode、内部全程文本"是同一种"把脏活锁在边界"的哲学。

### 2.2 `json` vs `pickle`：跨语言文本 vs Python 私有二进制

这两个模块解决的问题看似都是"把对象变成可存储/可传输的形式"，但定位天差地别，对照 Java 帮你秒懂：

| 维度 | `json`（≈ Jackson） | `pickle`（≈ Java `Serializable`） |
|------|------|------|
| 格式 | 文本（UTF-8），人类可读 | 二进制，私有协议 |
| 跨语言 | ✅ 任何语言都能解析 | ❌ **只有 Python 能反序列化** |
| 能存什么 | 只有 6 种 JSON 类型（对象/数组/字符串/数/布尔/null） | **任意 Python 对象图**（类实例、函数引用、闭包……） |
| 安全性 | 安全（纯数据） | **危险：反序列化可执行任意代码** |
| 典型用途 | HTTP API、配置、跨服务通信 | 进程间缓存、`multiprocessing` 传参、模型权重（内部可信场景） |

设计理念：**`json` 是"通用数据交换格式"**，所以它只支持所有语言都有的最小公共类型集，连 `datetime` 和 `Decimal` 都不认（JSON 标准里没有这两个概念，要你显式约定怎么编码）。**`pickle` 是"Python 进程的内存快照"**，所以它能把对象图原封不动序列化——代价是它必须信任数据来源，因为重建对象图的过程可以触发任意代码（见 ③ 3.5）。

> 对照 Java：Jackson 序列化 JSON 是跨语言的、安全的；Java 原生 `Serializable` + `ObjectInputStream` 是 Java 私有的、并且是著名的 RCE 漏洞温床（Log4Shell 之前，Java 反序列化漏洞统治了好几年的 CVE 榜）。**`pickle` 之于 Python，正是 `Serializable` 之于 Java**——同样的能力，同样的雷。

### 2.3 pydantic：用类型注解做运行时校验，补上 [14 章](14_类型系统与注解.md) 的缺口

[14 章](14_类型系统与注解.md) 反复强调：**Python 的类型注解运行时完全不强制**，`def f(x: int)` 传 str 照样跑。这在内部代码里没问题（靠 mypy 静态把关），但在**系统边界**（HTTP 请求体、外部 API 响应、配置文件、消息队列）就要命了——外部数据不可信，你必须在运行时**真的校验**：字段在不在、类型对不对、能不能转换、范围合不合法。

Java 这块是 Bean Validation（`@NotNull`/`@Size`/`@Email`）+ Hibernate Validator，靠注解 + 反射在运行时校验。Python 标准库**没有**对应物——`@dataclass` 只生成 `__init__`，不校验类型（[14 章](14_类型系统与注解.md) 3.6 讲过它只是 `exec` 拼接赋值语句）。

**pydantic 填了这个洞**。它的核心思想极其优雅：**复用你已经在写的类型注解，把它们变成运行时校验规则**。

```python
# pip install pydantic
from pydantic import BaseModel

class Order(BaseModel):
    id: int
    amount: float
    currency: str

# 注解不再只是"给工具看"——pydantic 在实例化时真的校验并转换
o = Order(id="42", amount="9.9", currency="USD")   # 字符串被自动转成 int/float
print(o.id, type(o.id))        # 输出: 42 <class 'int'>
print(o.amount)                # 输出: 9.9

Order(id="not-a-number", amount=1, currency="USD")
# 抛 pydantic.ValidationError：id 不能解析为 int
```

理念对比：

| | `@dataclass`（[14 章](14_类型系统与注解.md)） | pydantic `BaseModel` |
|---|---|---|
| 注解的作用 | 纯元数据，运行时不看 | **运行时校验 + 强制转换的规则** |
| 类型不符 | 静默接受 | 抛 `ValidationError` |
| 用途 | 内部可信数据容器 | **系统边界的输入校验** |
| 性能 | 零开销 | 校验有开销，但核心是 Rust（见 ③ 3.6） |

> 选型口诀：**内部传值、确定类型对 → `dataclass`；边界校验外部输入 → pydantic**。FastAPI（[24 章](24_HTTP与Web框架.md)）正是把 pydantic 模型当请求/响应 DTO，自动校验，这是 Python 后端的事实标准。

---

## ③ 实现细节（底层怎么实现的）★本章重点★

### 3.1 `datetime` 五个类的内部构成

`datetime` 模块里五个核心类，都是不可变（immutable）、可哈希的值对象：

| 类 | 存什么 | Java 对应 |
|----|--------|----------|
| `date` | year, month, day | `LocalDate` |
| `time` | hour, minute, second, microsecond, **tzinfo** | `LocalTime` |
| `datetime` | date + time 全部字段 + **tzinfo** | `LocalDateTime`/`ZonedDateTime` |
| `timedelta` | **只存 3 个数**：days, seconds, microseconds | `Duration` |
| `tzinfo` | 抽象基类，子类有 `timezone`、`ZoneInfo` | `ZoneId`/`ZoneOffset` |

`timedelta` 的内部规范化是个有意思的细节——无论你怎么构造，它只归一化存 3 个字段，并保证 `0 <= seconds < 86400`、`0 <= microseconds < 1e6`：

```python
from datetime import timedelta

td = timedelta(weeks=1, hours=25, minutes=70, seconds=5)
print(td)                                  # 输出: 8 days, 2:10:05
print(td.days, td.seconds, td.microseconds)  # 输出: 8 7805 0
# weeks=1(7天) + 25h = 8天1h；70min=1h10min；累计 8天 2:10:05
print(td.total_seconds())                  # 输出: 699005.0   折算成总秒数（float）
```

> 对照 Java `Duration`：`java.time.Duration` 内部存 `seconds`(long) + `nanos`(int)，精度到**纳秒**；Python `timedelta` 精度只到**微秒**（microseconds）。从 Java 高精度时间戳往 Python 搬数据，纳秒部分会丢精度，这是个真实的互操作坑。

### 3.2 naive vs aware：相减/比较的坑（深挖）

aware 与 naive **不能混合运算**，因为语义上无法定义——一个有时区锚点、一个没有，强行相减等于问"北京时间的 12 点减去某个不知道时区的 12 点等于多少"，无解：

```python
from datetime import datetime, timezone

naive = datetime(2026, 6, 28, 12, 0)
aware = datetime(2026, 6, 28, 12, 0, tzinfo=timezone.utc)

aware - naive
# 抛 TypeError: can't subtract offset-naive and offset-aware datetimes
naive < aware
# 抛 TypeError: can't compare offset-naive and offset-aware datetimes
```

但**两个 naive 之间、两个 aware 之间**都能算。最阴险的坑在这里——**两个 naive 相减不报错，但结果可能是错的**，因为它们可能其实代表不同时区的时间：

```python
from datetime import datetime
from zoneinfo import ZoneInfo

# 数据库里存的"北京时间"和"纽约时间"，都被读成了 naive
beijing_naive = datetime(2026, 6, 28, 12, 0)   # 实际是北京 12:00
ny_naive      = datetime(2026, 6, 28, 12, 0)   # 实际是纽约 12:00
print(ny_naive - beijing_naive)                # 输出: 0:00:00  ← 错！实际差 12 小时

# 正确做法：都带上时区（aware），相减自动按 UTC 对齐
beijing = datetime(2026, 6, 28, 12, 0, tzinfo=ZoneInfo("Asia/Shanghai"))
ny      = datetime(2026, 6, 28, 12, 0, tzinfo=ZoneInfo("America/New_York"))
print(ny - beijing)                            # 输出: 12:00:00  ← 对！
```

**底层机制**：aware 之间相减时，CPython 先把两边各自的 `utcoffset()` 减掉、换算到 UTC 时间轴再做差；naive 之间相减则直接把两个"墙上时间"的字段值相减——它**假设**两者在同一时区，而这个假设往往是错的。Java 用 `LocalDateTime`（不能直接转 `Instant`）从类型上拦你；Python 让你"能算但算错"。

### 3.3 `zoneinfo`：读 IANA 时区库（3.9+，PEP 615）

3.9 之前 Python 标准库没有像样的时区数据库，得装第三方 `pytz`（而且 `pytz` 的 API 反人类，要 `localize()`）。3.9 起 `zoneinfo` 进标准库，直接读操作系统的 IANA tz 数据库（Linux/Mac 在 `/usr/share/zoneinfo`），Windows 上没有系统库则从 PyPI 的 `tzdata` 包读：

```python
from zoneinfo import ZoneInfo
from datetime import datetime

shanghai = ZoneInfo("Asia/Shanghai")
dt = datetime(2026, 6, 28, 20, 0, tzinfo=shanghai)
print(dt.isoformat())      # 输出: 2026-06-28T20:00:00+08:00
print(dt.utcoffset())      # 输出: 8:00:00

# 转到另一个时区：同一时刻的不同墙上时间（astimezone）
ny = dt.astimezone(ZoneInfo("America/New_York"))
print(ny.isoformat())      # 输出: 2026-06-28T08:00:00-04:00  （同一瞬间，纽约的表示）
```

**`ZoneInfo` 比固定偏移强在哪**：它含**夏令时（DST）规则**，能算出"某个具体日期该用哪个偏移"。`timezone(timedelta(hours=8))` 是死偏移，永远 +8；`ZoneInfo("America/New_York")` 则会在 3 月/11 月自动切换 -5/-4：

```python
from datetime import datetime
from zoneinfo import ZoneInfo

ny = ZoneInfo("America/New_York")
winter = datetime(2026, 1, 15, 12, 0, tzinfo=ny)
summer = datetime(2026, 7, 15, 12, 0, tzinfo=ny)
print(winter.utcoffset())  # 输出: -1 day, 19:00:00  即 -5 小时（冬令时 EST）
print(summer.utcoffset())  # 输出: -1 day, 20:00:00  即 -4 小时（夏令时 EDT）
```

> 对照 Java：`ZoneId.of("America/New_York")` 同样读 IANA 库、含 DST 规则，`ZoneOffset.ofHours(-5)` 才是死偏移。`ZoneInfo` ≈ `ZoneId`，`timezone(timedelta(...))` ≈ `ZoneOffset`。**用时区名（`ZoneInfo`）而非死偏移**，否则夏令时一来全错。

DST 转换还有两个"魔鬼时刻"，对照 Java `ZonedDateTime` 的 gap/overlap 处理：

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

ny = ZoneInfo("America/New_York")
# 2026-03-08 02:30 在纽约"不存在"（钟从 02:00 直接跳到 03:00），fold 处理重叠
# 跨 DST 做加法：墙上时间 +1 天，但真实流逝可能不是 24 小时
start = datetime(2026, 3, 7, 12, 0, tzinfo=ny)
naive_plus = start + timedelta(days=1)             # 朴素加 1 天（墙上）
print(naive_plus.isoformat())    # 2026-03-08T12:00:00-04:00  偏移已从 -05 变 -04
# 真实经过的绝对时长：换算 UTC 看
elapsed = naive_plus.astimezone(ZoneInfo("UTC")) - start.astimezone(ZoneInfo("UTC"))
print(elapsed)                   # 输出: 23:00:00  ← 这一天只过了 23 小时！
```

`fold` 属性（PEP 495）用于消解"秋季回拨"时一个墙上时间出现两次的歧义，`fold=0` 是第一次、`fold=1` 是第二次出现——这是 Python 解决 DST 重叠的独特设计，Java 用 `withEarlierOffsetAtOverlap()`/`withLaterOffsetAtOverlap()`。

### 3.4 `json` 的编解码：为什么 `datetime` 不能直接 `dumps`

`json` 模块内部是 `JSONEncoder`（Python 对象 → JSON 文本）和 `JSONDecoder`（JSON 文本 → Python 对象）两个类。`json.dumps` 只是 `JSONEncoder().encode()` 的便捷封装。

**默认编码表**（写死在 C 加速器 `_json` 和纯 Python 回退里）：

| Python 类型 | JSON |
|---|---|
| `dict` | object `{}` |
| `list`, `tuple` | array `[]` |
| `str` | string |
| `int`, `float` | number |
| `True`/`False` | `true`/`false` |
| `None` | `null` |

**仅此 6 类**。`datetime`、`Decimal`、`set`、自定义类——统统不认，因为 **JSON 标准里没有这些概念**：

```python
import json
from datetime import datetime, timezone

json.dumps({"t": datetime(2026, 6, 28, 12, 0, tzinfo=timezone.utc)})
# 抛 TypeError: Object of type datetime is not JSON serializable
```

解决靠 `default` 钩子——遇到不认识的对象时回调它，你返回一个"可序列化的替代物"：

```python
import json
from datetime import datetime, timezone
from decimal import Decimal

def encode_extra(o):
    if isinstance(o, datetime):
        return o.isoformat()           # datetime → ISO8601 字符串
    if isinstance(o, Decimal):
        return float(o)                # Decimal → float（注意精度，见下）
    raise TypeError(f"{type(o).__name__} 不可序列化")

data = {"t": datetime(2026, 6, 28, 12, 0, tzinfo=timezone.utc),
        "price": Decimal("9.99"), "name": "café"}
print(json.dumps(data, default=encode_extra, ensure_ascii=False))
# 输出: {"t": "2026-06-28T12:00:00+00:00", "price": 9.99, "name": "café"}
```

反方向用 `object_hook`——每解析出一个 JSON object（dict）就回调它，让你把字符串转回 `datetime`：

```python
import json
from datetime import datetime

def decode_extra(d):
    if "created" in d:
        d["created"] = datetime.fromisoformat(d["created"])
    return d

s = '{"id": 1, "created": "2026-06-28T12:00:00+00:00"}'
print(json.loads(s, object_hook=decode_extra))
# 输出: {'id': 1, 'created': datetime.datetime(2026, 6, 28, 12, 0, tzinfo=...utc)}
```

> 对照 Jackson：`default`/`object_hook` 就是 Python 版的 `JsonSerializer`/`JsonDeserializer`（或 `@JsonFormat`）。Jackson 的 `JavaTimeModule` 默认就懂 `LocalDateTime`，Python 的 `json` 啥都不懂、要你手写——**这也是为什么后端直接用 pydantic**（它内置 `datetime`/`Decimal`/`UUID` 等的序列化，见 ④）。

`ensure_ascii=False` 是中文场景必加项：默认 `ensure_ascii=True` 会把非 ASCII 转成 `\uXXXX` 转义（见上面 `café` 那种），加 `False` 才输出真正的 UTF-8 字符（呼应 [09 章](09_字符串字节与编码.md) 的编码哲学）。

### 3.5 `pickle` 协议与 `__reduce__`：RCE 风险（★安全重点★）

`pickle` 把 Python 对象图序列化成字节流。它有 6 个协议版本（0–5），数字越大越紧凑高效，协议 0 是古早的 ASCII 格式，**协议 5（3.8+ 起为最高）**支持 out-of-band 大数据（如 NumPy 数组零拷贝）：

```python
import pickle

class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y

p = Point(1, 2)
blob = pickle.dumps(p)                  # 对象 → bytes
print(type(blob))                       # 输出: <class 'bytes'>
print(pickle.HIGHEST_PROTOCOL)          # 输出: 5
p2 = pickle.loads(blob)                 # bytes → 重建对象
print(p2.x, p2.y)                       # 输出: 1 2
```

**底层机制 `__reduce__`**：pickle 不是简单 dump 字段，而是问对象"你怎么重建自己"。每个对象的 `__reduce__()`（或 `__reduce_ex__`）返回一个元组 `(callable, args, ...)`，意思是"反序列化时，调用 `callable(*args)` 就能把我造回来"。`pickle.loads` 时就**真的去调用这个 callable**——这正是 RCE 的根：

```python
import pickle

class Evil:
    def __reduce__(self):
        # 返回 (要调用的函数, 参数元组)。loads 时会执行 函数(*参数)
        import os
        return (os.system, ("echo 你的机器现在执行了攻击者的命令",))

payload = pickle.dumps(Evil())
# 受害者一旦 loads 这段不可信数据 ——
pickle.loads(payload)
# 控制台打印: 你的机器现在执行了攻击者的命令
# 真实攻击里这里会是 rm -rf、反弹 shell、下载木马……
```

**这就是为什么文档头一句话是：绝不要 unpickle 来自不可信来源的数据。** 一个精心构造的 pickle 字节流 = 任意代码执行。攻击面包括：用 pickle 做的缓存/session（如果攻击者能写入）、`pickle` 序列化的网络消息、把 pickle 文件当下载内容、`pandas.read_pickle` 读网上的 `.pkl`、joblib 加载来路不明的模型权重……

> **对照 Java 反序列化漏洞**：Java `ObjectInputStream.readObject()` 同样会在反序列化时调用对象的 `readObject`/构造逻辑，配合 classpath 上的 "gadget chain"（如 Commons-Collections）可构造 RCE——这是 2015 年起统治 CVE 榜的一类漏洞。**`pickle.loads` 和 `ObjectInputStream.readObject` 是同一类雷**。Java 的缓解是 `ObjectInputFilter` 白名单 + 尽量改用 JSON；Python 的缓解就一条：**信不过的数据，永远别 pickle.loads，改用 JSON**。

防御要点：
- 跨服务/跨网络/面向用户的数据，**一律 `json`**，不用 `pickle`。
- pickle **只用于完全可信的内部场景**：`multiprocessing` 进程间传参（同一份代码）、本地缓存（自己写自己读）。
- 真要传 pickle 又怕篡改，加 HMAC 签名校验来源（但仍不如直接别用）。
- 别用 pickle 做长期存储格式：它绑定 Python 版本和类定义，类改了就 loads 不回来（脆弱），这点也不如 JSON。

### 3.6 pydantic v2 的 Rust 核心（pydantic-core）：为什么快

pydantic v1 是纯 Python 实现，校验逻辑全是 Python 代码，复杂模型校验慢。**pydantic v2（2023 发布）做了彻底重写：把校验核心抽成独立的 Rust crate `pydantic-core`**，Python 层只负责"把你的模型定义编译成一份校验器 schema"，真正逐字段校验的热循环跑在 Rust 里。

工作流程（理解这个才知道它和 dataclass 的本质区别）：

```
你写的 BaseModel 子类
   │  ① 类创建时（元类 ModelMetaclass）：
   │     读取所有字段的类型注解 + Field() 约束
   ▼
   构建一份 "core schema"（描述每个字段怎么校验/转换的数据结构）
   │  ② 把 schema 交给 pydantic-core（Rust）编译成一个 SchemaValidator
   ▼
   SchemaValidator（Rust 对象，挂在模型类上）
   │  ③ 每次 Model(**data) 或 model_validate(data)：
   │     数据进入 Rust 热循环，逐字段校验+强制转换
   ▼
   成功 → 填好字段的模型实例 / 失败 → ValidationError（含每个错误的路径和原因）
```

关键点：

- **校验器只在类创建时构建一次**（编译期一样的一次性成本），之后每次实例化都复用，热路径在 Rust，所以比 v1 快 5–50 倍。
- **和 `dataclass` 的本质区别**：`dataclass` 用 `exec` 拼接的 `__init__` 只做 `self.x = x` 赋值（[14 章](14_类型系统与注解.md) 3.6），**零校验**；pydantic 的 `__init__` 走 Rust 校验器，**真的检查并转换**每个字段。这就是 ②2.3 那张表的底层原因。
- pydantic 用**元类**（[19 章](19_描述符与元类.md) 的机制）在类创建时介入——和 `@dataclass` 用装饰器在类创建后介入异曲同工，但 pydantic 干的活重得多（构建 Rust schema）。

```python
# pip install pydantic
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str

# 校验器在 class User 定义时就已构建（Rust 对象）
print(type(User.__pydantic_validator__))
# 输出: <class 'pydantic_core._pydantic_core.SchemaValidator'>  ← Rust 对象
```

> v1 vs v2 是 pydantic 史上最大断裂，API 大量改名（见 ⑤ 的对照表）。新项目**一律 v2**；遇到老代码报 `BaseSettings` 找不到、`.dict()` 等，多半是 v1/v2 混淆。

---

## ④ 使用方法与最佳实践

### 4.1 `datetime` 基础运算

```python
from datetime import datetime, date, timedelta, timezone
from zoneinfo import ZoneInfo

# ✅ 取当前时间：永远带时区！用 now(tz)，不要裸 now()/已废弃的 utcnow()
now = datetime.now(timezone.utc)              # aware，UTC
local_now = datetime.now(ZoneInfo("Asia/Shanghai"))  # aware，本地

# 日期算术：datetime ± timedelta → datetime
deadline = now + timedelta(days=7, hours=12)
print((deadline - now).total_seconds())       # 输出: 648000.0  （7.5 天的秒数）

# 比较（同为 aware 才行）
print(deadline > now)                          # 输出: True

# 取日期部分 / 组合
today = date.today()
print(today.weekday())                         # 0=周一 ... 6=周日
print(today.isoweekday())                      # 1=周一 ... 7=周日

# 替换字段（不可变，replace 返回新对象，类似 java.time 的 withXxx）
midnight = now.replace(hour=0, minute=0, second=0, microsecond=0)
```

> **`datetime.utcnow()` 已在 3.12 弃用**（它返回的是 naive，是无数 bug 的源头！），一律用 `datetime.now(timezone.utc)` 得到 aware。

### 4.2 时区处理：内部 UTC，边界转换

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

# 入口：把用户提交的本地时间（带其时区）→ 统一转 UTC 存储
user_input = datetime(2026, 6, 28, 20, 0, tzinfo=ZoneInfo("Asia/Shanghai"))
stored = user_input.astimezone(timezone.utc)
print(stored.isoformat())     # 输出: 2026-06-28T12:00:00+00:00  ← 库里存这个

# 出口：从 UTC 取出 → 转用户所在时区展示
display = stored.astimezone(ZoneInfo("America/New_York"))
print(display.isoformat())    # 输出: 2026-06-28T08:00:00-04:00

# 给 naive 补时区（确知它是哪个时区时）用 replace，而非 astimezone！
naive = datetime(2026, 6, 28, 20, 0)
correct = naive.replace(tzinfo=ZoneInfo("Asia/Shanghai"))   # "贴标签"
# astimezone 会假设 naive 是本地时区再转换，含义不同，别混用
```

> 黄金法则：**存储和传输用 UTC（aware），仅在 UI 展示层 `astimezone` 到用户时区**。数据库存 UTC、日志记 UTC、API 传 ISO8601 带偏移——全链路无歧义。

### 4.3 ISO8601 解析与格式化

```python
from datetime import datetime, timezone

dt = datetime(2026, 6, 28, 12, 30, 45, tzinfo=timezone.utc)

# ISO8601（API 传输首选，机器友好、带时区）
print(dt.isoformat())                       # 输出: 2026-06-28T12:30:45+00:00
parsed = datetime.fromisoformat("2026-06-28T12:30:45+00:00")
print(parsed == dt)                         # 输出: True
# 3.11+ fromisoformat 增强，能解析末尾 'Z'（UTC 的 Zulu 写法）
print(datetime.fromisoformat("2026-06-28T12:30:45Z"))  # 3.11+ 可用

# strftime/strptime：自定义格式（对照 Java DateTimeFormatter）
print(dt.strftime("%Y-%m-%d %H:%M:%S %Z"))  # 输出: 2026-06-28 12:30:45 UTC
custom = datetime.strptime("2026/06/28 12:30", "%Y/%m/%d %H:%M")
print(custom)                               # 输出: 2026-06-28 12:30:00  （注意是 naive！）
```

常用格式码：`%Y`(四位年) `%m`(月) `%d`(日) `%H`(24时) `%M`(分) `%S`(秒) `%z`(±0800) `%Z`(时区名) `%f`(微秒) `%A`(星期全名) `%j`(年内第几天)。

> **优先 `isoformat`/`fromisoformat`**（标准、无歧义、带时区）；只在对接固定格式的老系统时才用 `strftime`/`strptime`。注意 `strptime` 解析出的若格式串没含时区信息，结果是 **naive**——这是个常见的 aware/naive 退化点。

### 4.4 `json`：序列化最佳实践

```python
import json
from datetime import datetime, timezone
from decimal import Decimal

payload = {
    "order_id": 1001,
    "created": datetime(2026, 6, 28, 12, 0, tzinfo=timezone.utc),
    "amount": Decimal("99.90"),
    "customer": "张三",
}

def default(o):
    if isinstance(o, datetime):
        return o.isoformat()
    if isinstance(o, Decimal):
        return str(o)          # ← 用 str 而非 float！避免二进制浮点精度丢失（见 08 章）
    raise TypeError(f"{type(o).__name__} not serializable")

# 序列化：ensure_ascii=False 输出真中文；indent 美化；default 处理特殊类型
text = json.dumps(payload, default=default, ensure_ascii=False, indent=2)
print(text)
# {
#   "order_id": 1001,
#   "created": "2026-06-28T12:00:00+00:00",
#   "amount": "99.90",
#   "customer": "张三"
# }

# 反序列化：parse_float 控制数字解析（金融场景用 Decimal 而非 float）
back = json.loads(text, parse_float=Decimal)
print(type(back["order_id"]), type(back["amount"]))
# 输出: <class 'int'> <class 'str'>   （amount 此处是字符串，需自行转 Decimal）
```

要点：
- **`ensure_ascii=False`**：中文/emoji 直接输出 UTF-8，别让满屏 `\uXXXX`。
- **`Decimal` 转 JSON 用 `str`**：JSON number 没有 Decimal 概念，转 `float` 会引入二进制浮点误差（[08 章](08_数字与算术语义.md) 详述），金额一律用字符串传输。
- 读文件：`json.load(f)`；写文件：`json.dump(obj, f)`（无 `s`，直接读写文件对象）。
- 大文件/流式：标准库 `json` 不支持流式解析，海量数据考虑 `ijson` 或按行存的 JSONL。

### 4.5 `pickle`：仅限可信内部场景

```python
import pickle

cache = {"weights": [1.5, 2.3], "version": 3}

# ✅ 正确用途：本地缓存、进程间传对象（自己写自己读，来源可信）
with open("cache.pkl", "wb") as f:        # 注意是二进制模式 "wb"
    pickle.dump(cache, f)
with open("cache.pkl", "rb") as f:
    restored = pickle.load(f)
print(restored)                            # 输出: {'weights': [1.5, 2.3], 'version': 3}

# ❌ 绝不这样：loads 网络来的/用户上传的/缓存可被篡改的数据
# data = requests.get(url).content
# obj = pickle.loads(data)        # ← 潜在 RCE！
```

控制哪些字段被 pickle（如排除连接、文件句柄等不可序列化或不该存的状态）用 `__getstate__`/`__setstate__`：

```python
class Service:
    def __init__(self):
        self.config = {"retries": 3}
        self.conn = "<live db connection>"     # 不该被 pickle 的运行时资源

    def __getstate__(self):
        state = self.__dict__.copy()
        del state["conn"]                       # pickle 时排除连接
        return state

    def __setstate__(self, state):
        self.__dict__.update(state)
        self.conn = None                        # 反序列化后重新建立
```

> 这对照 Java 的 `transient` 关键字 + `readObject`/`writeObject`——同样的"排除瞬态字段、反序列化后重建"模式。

### 4.6 pydantic v2：定义、校验、序列化

```python
# pip install pydantic
from datetime import datetime
from decimal import Decimal
from typing import Literal
from pydantic import BaseModel, Field, EmailStr, field_validator

class OrderItem(BaseModel):
    sku: str
    qty: int = Field(gt=0, le=999)             # 约束：>0 且 <=999（对照 @Min/@Max）

class CreateOrder(BaseModel):
    order_id: int
    amount: Decimal = Field(gt=0, decimal_places=2)
    status: Literal["pending", "paid", "shipped"] = "pending"   # 枚举式约束
    items: list[OrderItem] = Field(default_factory=list)
    created: datetime                          # 自动解析 ISO8601 字符串 → datetime
    note: str | None = None                    # 可空字段

    @field_validator("order_id")
    @classmethod
    def must_be_positive(cls, v: int) -> int:  # 自定义校验（对照自定义 Validator）
        if v <= 0:
            raise ValueError("order_id 必须为正")
        return v

# —— 校验 + 解析：从外部 dict（如 HTTP body 解析出的）构造 ——
raw = {
    "order_id": "1001",                        # 字符串，会被转成 int
    "amount": "99.90",
    "status": "paid",
    "items": [{"sku": "A1", "qty": 2}],
    "created": "2026-06-28T12:00:00+00:00",    # ISO 字符串 → datetime
}
order = CreateOrder.model_validate(raw)        # v2 用 model_validate（v1 是 parse_obj）
print(order.order_id, type(order.order_id))    # 输出: 1001 <class 'int'>
print(order.created)                           # 输出: 2026-06-28 12:00:00+00:00
print(order.amount)                            # 输出: 99.90  （Decimal，未丢精度）

# —— 序列化：模型 → dict / JSON ——
print(order.model_dump())                      # → Python dict（v1 是 .dict()）
print(order.model_dump(mode="json"))           # datetime/Decimal 转成 JSON 友好的字符串
print(order.model_dump_json())                 # → JSON 字符串（v1 是 .json()）
# model_dump_json 直接产出: {"order_id":1001,"amount":"99.90","status":"paid",...}
```

校验失败时 `ValidationError` 带**结构化错误**（每个错误有字段路径、类型、消息），非常适合直接返回给前端：

```python
from pydantic import ValidationError

try:
    CreateOrder.model_validate({"order_id": "x", "amount": -5, "created": "bad"})
except ValidationError as e:
    print(e.error_count())          # 输出: 3
    for err in e.errors():
        print(err["loc"], err["type"])
    # ('order_id',) int_parsing
    # ('amount',) greater_than
    # ('created',) datetime_from_date_parsing
```

> 对照 Bean Validation：`Field(gt=0)` ≈ `@Positive`，`Field(max_length=10)` ≈ `@Size(max=10)`，`EmailStr` ≈ `@Email`，`@field_validator` ≈ 自定义 `ConstraintValidator`。但 pydantic 更进一步——它不只校验，还**强制类型转换**（`"1001"` → `1001`），这是 Bean Validation 不做的（Spring 那层的转换由 `HttpMessageConverter` 做，pydantic 把校验+转换合一）。

### 4.7 pydantic-settings：读环境变量配置

```python
# pip install pydantic-settings
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_", env_file=".env")

    db_url: str                                # 必填，缺了就 ValidationError
    pool_size: int = 10                        # 有默认值
    debug: bool = False                        # "true"/"1"/"yes" 自动转 bool
    timeout: float = Field(default=30.0, gt=0)

# 从环境变量读：APP_DB_URL, APP_POOL_SIZE, APP_DEBUG ...
# 自动按类型转换 + 校验
settings = Settings()                          # 实例化即读取并校验环境
print(settings.db_url, settings.pool_size, settings.debug)
```

> 对照 Spring `@ConfigurationProperties` + `application.yml`：`BaseSettings` 自动从环境变量/`.env` 文件读、按声明类型转换并校验，缺必填项启动即失败（fail-fast）。这是 Python 后端管理配置的标准做法，比手写 `os.environ.get(...) + int(...)` 安全得多（见 [02 环境与工具链](02_环境与工具链.md) 关于环境隔离，以及 [22 文件IO与系统交互](22_文件IO与系统交互.md) 的 `os.environ`）。

---

## ⑤ ⚠️ 易错点（Java 思维翻车）

| Java 习惯 | 翻车现场 | 正确做法 / 认知 |
|----------|---------|----------------|
| 以为时间对象都自带时区（像 `ZonedDateTime`） | `datetime(2026,6,28,12,0)` 默认是 **naive**，无时区 | 显式 `tzinfo=`，存储一律 aware/UTC |
| 用 `datetime.now()` 取当前时间 | 得到 naive 本地时间，跨时区比较/存储全乱 | 用 `datetime.now(timezone.utc)`（aware） |
| 用 `datetime.utcnow()`（看着像对的） | 它返回 **naive**！是无数 bug 源，3.12 已弃用 | `datetime.now(timezone.utc)` 才是 aware UTC |
| 拿 naive 和 aware 直接比较/相减 | `TypeError: can't compare offset-naive and offset-aware` | 全程统一为 aware；naive 补时区用 `replace(tzinfo=...)` |
| 两个 naive 相减以为结果对 | 不报错但**算错**（假设同时区，实则不同） | 都带时区再算，aware 相减自动按 UTC 对齐 |
| 用固定偏移当时区 | `timezone(timedelta(hours=-5))` 不含 DST，夏令时全错 | 用 `ZoneInfo("America/New_York")`（含 DST 规则） |
| 直接 `json.dumps(datetime对象)` | `TypeError: ... not JSON serializable` | 给 `default=` 钩子转 ISO 字符串，或用 pydantic |
| `json.dumps` 中文 | 默认输出 `张三` 一堆转义 | 加 `ensure_ascii=False` |
| 金额用 float 走 JSON | 二进制浮点误差，`0.1+0.2!=0.3` | `Decimal` + 用 `str` 序列化（见 [08 章](08_数字与算术语义.md)） |
| 把 `pickle` 当通用序列化（像 Jackson 那样到处用） | `pickle.loads` 不可信数据 → **RCE**，和 Java 反序列化漏洞同源 | 跨服务/对外一律 `json`；pickle 仅可信内部 |
| 混淆 `json` 和 `pickle` 用途 | 用 pickle 存长期数据 → 换 Python 版本/改类就读不回 | json=跨语言文本存档；pickle=临时进程内缓存 |
| 以为类型注解会运行时校验（[14 章](14_类型系统与注解.md) 老坑） | `dataclass` 收下任何类型，外部脏数据潜入 | 边界用 pydantic `BaseModel` 做真校验 |
| 用 pydantic v1 的 API | `.dict()`/`.json()`/`parse_obj`/`BaseSettings` 从 pydantic 导入 → 报错或弃用警告 | v2：`model_dump()`/`model_dump_json()`/`model_validate()`；`BaseSettings` 改从 `pydantic_settings` 导 |
| 以为 `strptime` 出来是 aware | 格式串无 `%z` 时解析结果是 **naive** | 解析后 `replace(tzinfo=...)` 或确保格式含时区 |

一段典型的"Java 味 Python"反面教材：

```python
# ❌ Java 脑回路：以为 now() 带时区、注解会校验、pickle 像 Jackson 一样安全通用
import pickle
from datetime import datetime, timedelta

class BookingService:
    def create(self, start: datetime, hours: int) -> dict:
        # 1) 用 now() —— 拿到 naive 本地时间
        created = datetime.now()
        # 2) start 来自用户、可能是 aware；naive - aware 会炸
        duration = start - created                  # 可能 TypeError！
        end = start + timedelta(hours=hours)
        return {"start": start, "end": end, "created": created}

    def save_to_cache(self, data: bytes):
        # 3) 把外部传来的字节直接 unpickle —— RCE 大门
        obj = pickle.loads(data)
        return obj
```

问题：(1) `datetime.now()` 是 naive，与可能为 aware 的 `start` 相减抛 `TypeError`；(2) 返回的 dict 里塞了 `datetime` 对象，下一步 `json.dumps` 必炸；(3) `pickle.loads` 不可信的 `data` 是教科书级 RCE 漏洞。修正：`created = datetime.now(timezone.utc)`，统一 aware；用 pydantic 模型定义请求并 `model_dump_json()` 序列化；缓存改用 json 或确保数据完全可信。

---

## ⑥ 练习题

### 基础理解

1. 用一句话分别说清：(a) naive datetime 和 aware datetime 的区别，各自对应 `java.time` 的哪个类；(b) 为什么 `datetime.now()` 在后端代码里几乎总是错的，该用什么代替。

2. `json` 和 `pickle` 都能"把对象变成可存储的形式"，请从**跨语言能力**和**安全性**两个维度说清它们的根本区别，并各举一个**该用它**的典型场景。

### 找坑改错

3. 下面这段"订单超时检测"在测试环境（数据全是 naive）跑得好好的，一上线接入带时区的真实数据就频繁抛异常或算错时长。指出**两个**问题并改写：

```python
from datetime import datetime, timedelta

def is_expired(order_created):                  # order_created 来自 DB，可能 aware
    now = datetime.now()                        # ①
    age = now - order_created                   # ②
    return age > timedelta(hours=24)
```

4. 一个同事写了"通用对象缓存"，号称"比 JSON 强，能存任何 Python 对象，还快"，并把它用在了**接收下游服务推送的消息**上。指出这里最严重的安全问题，并说明该怎么改：

```python
import pickle

def load_message(raw: bytes):
    # 下游服务通过 MQ 推送的二进制消息
    return pickle.loads(raw)                     # "什么对象都能还原，多方便"
```

### 综合实战

5. 你要给一个"创建支付"的 API 写请求体校验。需求：请求 JSON 含 `payment_id`(int，>0)、`amount`(金额，两位小数，>0，**不能用 float**)、`currency`(只能是 `"CNY"`/`"USD"`/`"EUR"`)、`created_at`(ISO8601 时间字符串，**必须是 aware**)、可选的 `memo`(字符串，最长 200)。请：(a) 用 pydantic v2 定义这个模型，加上所有约束；(b) 写出从一段 JSON 字符串校验构造、再序列化回 JSON 的完整流程；(c) 说明如果改用 `@dataclass` 而非 pydantic，会**丢掉**哪些运行时保障（点名第 14 章的结论）。

---

### 参考答案

**1.** (a) naive datetime 的 `tzinfo is None`，是"墙上时间"、不指向时间轴上唯一一点，对应 `LocalDateTime`；aware datetime 带 `tzinfo`、锚定到具体时区/UTC，对应 `ZonedDateTime`（UTC 的 aware 则对应 `Instant`）。(b) `datetime.now()` 返回 **naive 本地时间**，既丢了时区信息、又无法和 aware 数据比较/相减（抛 `TypeError`），跨时区或存库后必出错；应用 `datetime.now(timezone.utc)` 得到 aware 的 UTC 时间。

**2.** **跨语言**：`json` 是文本格式，任何语言都能解析，适合跨服务/对外 API；`pickle` 是 Python 私有二进制，只有 Python 能反序列化。**安全性**：`json` 是纯数据，安全；`pickle.loads` 在重建对象时会执行对象 `__reduce__` 指定的 callable，**反序列化不可信数据 = 任意代码执行（RCE）**，与 Java `ObjectInputStream` 漏洞同源。典型场景：`json` 用于 HTTP API 请求/响应、跨服务通信、配置文件；`pickle` 仅用于完全可信的内部场景，如 `multiprocessing` 进程间传参、自己写自己读的本地缓存。

**3.** 两个问题：
- **①**：`datetime.now()` 是 naive，而 `order_created` 来自 DB 可能是 aware——naive 与 aware 相减直接 `TypeError`；即便都 naive，本地时间与可能不同时区的存储时间相减也算错。应改为 `datetime.now(timezone.utc)`。
- **②**：相减的前提是两边时区语义一致。要保证 `order_created` 也是 aware（若 DB 读出来是 naive 且已知存的是 UTC，需 `replace(tzinfo=timezone.utc)` 补上），再相减才正确。

```python
from datetime import datetime, timedelta, timezone

def is_expired(order_created):
    now = datetime.now(timezone.utc)                     # aware UTC
    if order_created.tzinfo is None:                     # 容错：若 DB 读出是 naive
        order_created = order_created.replace(tzinfo=timezone.utc)  # 已知存的是 UTC
    age = now - order_created                            # 两边都 aware，按 UTC 对齐
    return age > timedelta(hours=24)
```

**4.** 最严重的问题：`pickle.loads` 用在**外部（下游服务、MQ）推送的不可信数据**上——攻击者（或被攻陷的下游、可被篡改的消息通道）只需构造一段恶意 pickle 字节流（在 `__reduce__` 里返回 `(os.system, ("rm -rf ...",))` 之类），消费端一 `loads` 就会**执行任意代码（RCE）**，与 Java 反序列化漏洞同源。改法：跨服务消息**一律用 `json`**（纯数据、跨语言、安全），消息格式约定好字段，必要时再用 pydantic 校验解析结果。若因历史原因必须传 pickle，至少要保证通道完全可信并加 HMAC 签名校验来源——但首选永远是别用 pickle 读外部数据。

**5.**

```python
# pip install pydantic
from datetime import datetime
from decimal import Decimal
from typing import Literal
from pydantic import BaseModel, Field, field_validator, ValidationError

class CreatePayment(BaseModel):
    payment_id: int = Field(gt=0)
    amount: Decimal = Field(gt=0, decimal_places=2)         # Decimal 而非 float，两位小数
    currency: Literal["CNY", "USD", "EUR"]                  # 枚举式约束
    created_at: datetime                                    # 自动解析 ISO8601
    memo: str | None = Field(default=None, max_length=200)  # 可选，最长 200

    @field_validator("created_at")
    @classmethod
    def must_be_aware(cls, v: datetime) -> datetime:        # 强制 aware
        if v.tzinfo is None:
            raise ValueError("created_at 必须带时区（aware）")
        return v

# (b) 完整流程：JSON 字符串 → 校验构造 → 序列化回 JSON
incoming = '''
{"payment_id": 1001, "amount": "99.90", "currency": "USD",
 "created_at": "2026-06-28T12:00:00+00:00", "memo": "首单"}
'''
try:
    payment = CreatePayment.model_validate_json(incoming)   # 直接从 JSON 字符串校验
    print(payment.amount, type(payment.amount))   # 输出: 99.90 <class 'decimal.Decimal'>
    print(payment.created_at.tzinfo)              # 输出: UTC
    out = payment.model_dump_json()               # 序列化回 JSON
    print(out)
    # {"payment_id":1001,"amount":"99.90","currency":"USD","created_at":"2026-06-28T12:00:00Z","memo":"首单"}
except ValidationError as e:
    print(e.errors())                             # 结构化错误，可直接返回前端
```

(c) 若改用 `@dataclass`，会**丢掉所有运行时校验**（[14 章](14_类型系统与注解.md) 的结论：注解运行时不强制）：不会检查 `payment_id` 真的是正整数、`amount` 真的两位小数且为正、`currency` 真的在三个字面量内、`created_at` 真的是 aware datetime——`dataclass` 只生成 `__init__/__repr__/__eq__` 且只做字段赋值，传 `currency="JPY"`、`amount=-5`、naive 时间统统照单全收，脏数据潜入下游才暴露。同时还会丢掉**类型强制转换**（`"1001"`→`1001`、ISO 字符串→`datetime`、`"99.90"`→`Decimal`）和**结构化错误报告**。要在 API 边界做真校验，必须用 pydantic（或等价的运行时校验库），这正是 FastAPI（[24 HTTP与Web框架](24_HTTP与Web框架.md)）选 pydantic 当请求体 DTO 的原因。

---

下一章：[27 标准库与数据AI生态](27_标准库与数据AI生态.md) — 盘点 Python "自带电池"的标准库精华与数据/AI 生态（NumPy/pandas 入口），看 Python 为什么是数据时代的胶水语言。
