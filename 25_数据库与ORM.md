# 25 · 数据库与 ORM ⭐

> 你在 Java 里把数据库这层摸得门儿清：底层 JDBC（`DriverManager` → `Connection` → `PreparedStatement` → `ResultSet`），上面盖 HikariCP 连接池，再往上要么 MyBatis 手写 SQL、要么 Hibernate/JPA 全自动 ORM，schema 变更交给 Flyway/Liquibase。Python 这套**结构惊人地对称**：DB-API 2.0 就是"Python 版 JDBC 规范"，SQLAlchemy 就是"既是 MyBatis 又是 Hibernate 的二合一"，Alembic 就是 Flyway。但**对称之下全是差异**：DB-API 用 `%s`/`?` 占位符而非 `?` 一种；SQLAlchemy 故意分裂成 "Core（SQL 表达式）+ ORM（对象映射）"两层让你按需下沉；Session 的"工作单元"比 JPA 的 `EntityManager` 更显式、事务边界全靠你用 `with` 划清楚；而 N+1、连接泄漏、detached 对象访问惰性属性这些 Hibernate 老坑，在 Python 里**一个不少地复刻了**。更妙的是：上一章 [19 描述符与元类](19_描述符与元类.md) 拆过的"字段=描述符、模型类=元类"，正是 SQLAlchemy 模型的底层实现——这一章你将亲眼看到那套机制在工业级框架里怎么落地。读完你不仅会用，还能看懂 `User.name == "Alice"` 凭什么能拼出 SQL。

---

## ① Java 对照表

| 概念 | Java 世界 | Python 世界 | 一句话差异 |
|------|----------|------------|-----------|
| 驱动规范 | JDBC（`java.sql.*` 接口） | **DB-API 2.0**（PEP 249，约定俗成的规范） | DB-API 是"鸭子规范"，没有 `interface` 强制，靠模块函数/对象约定 |
| 连接 | `Connection` | `connection` 对象 | 都管事务边界；Python 的 `connection` 自带 `commit()`/`rollback()` |
| 语句执行器 | `Statement`/`PreparedStatement` | `cursor` 对象 | Python 没有独立的"预编译语句对象"，参数化直接在 `cursor.execute` 做 |
| 结果集 | `ResultSet`（游标式 `next()`） | `cursor` 的 `fetchone/fetchmany/fetchall`，或直接迭代 | cursor 既是执行器又是结果游标，一个对象两用 |
| 占位符 | `?`（唯一） | `?` 或 `%s` 或 `:name`，**因驱动而异**（`paramstyle`） | Python 各驱动占位符不统一，是 DB-API 的著名坑 |
| 连接池 | HikariCP / Druid（独立组件） | SQLAlchemy `QueuePool`（引擎内置）/ asyncpg 自带池 | Python 连接池通常长在 ORM/引擎里，不是独立库 |
| 全自动 ORM | Hibernate / JPA | **SQLAlchemy ORM** | 都做对象-关系映射、脏检查、工作单元 |
| SQL 映射器 | MyBatis（写 SQL，映射结果） | **SQLAlchemy Core**（SQL 表达式构造器）/ 原生 SQL | Core 用 Python 拼 SQL，类型安全；MyBatis 用 XML/注解 |
| 会话 / 持久化上下文 | JPA `EntityManager` | SQLAlchemy `Session` | 都管身份映射 + 工作单元，但 Session 事务边界更显式 |
| 工作单元 | Hibernate 的 `flush`/脏检查 | Session 的 `flush`/`commit` | 概念同源（都源自 Fowler 的 Unit of Work 模式） |
| 一级缓存 / 身份映射 | persistence context 的 identity map | Session 的 identity map | 同主键在一个会话里永远是同一对象 |
| 迁移工具 | Flyway / Liquibase | **Alembic**（SQLAlchemy 官方） | Alembic 迁移脚本是 Python 而非 SQL/XML |
| 实体状态 | transient/persistent/detached/removed（JPA） | transient/pending/persistent/deleted/detached | 状态机几乎一一对应 |
| 惰性加载 | `@OneToMany(fetch=LAZY)` + 代理 | `lazy="select"`（默认）+ `selectinload`/`joinedload` | N+1 病根同源，解法名字不同 |
| 异步驱动 | R2DBC（响应式） | asyncpg / async SQLAlchemy | Python 异步走 asyncio（[21 异步编程asyncio](21_异步编程asyncio.md)） |

> 一句话定位：**DB-API 2.0 ≈ JDBC 规范；SQLAlchemy = MyBatis（Core）+ Hibernate（ORM）合体；Session ≈ EntityManager；Alembic ≈ Flyway。** 整层架构高度同构，但占位符不统一、事务边界更显式、ORM 魔法的实现细节（描述符+元类）对你这个看过 19 章的人是透明的。

---

## ② 设计理念（为什么 Python 这么设计）

### 2.1 DB-API 2.0：一份"君子协定"式的统一驱动接口

JDBC 是**强约束**的：`java.sql.Connection`、`Statement`、`ResultSet` 都是 JVM 标准库里的 `interface`，每个数据库厂商必须 `implements` 它们，编译期就保证你换数据库时上层代码不用改。这是 Java"接口优先 + 编译期契约"哲学的典型产物。

Python 的 **DB-API 2.0**（PEP 249）想达成同样的目标——"换数据库不换上层代码"——但用的是 Python 式的**弱约束**：它**不是一组 `abc.ABC` 抽象基类**，而是一份**文档约定**。规范说："凡是数据库驱动模块，都应该提供一个 `connect()` 函数返回 connection 对象，connection 对象应该有 `cursor()`/`commit()`/`rollback()`/`close()` 方法，cursor 对象应该有 `execute()`/`fetchone()`/`fetchall()` 方法……"。驱动作者**自觉遵守**这份约定，但没有任何东西在编译期/导入期强制检查。

```python
# DB-API 的"统一"体现在：换驱动只改 import 和 connect 参数，CRUD 代码几乎不动
import sqlite3
conn = sqlite3.connect(":memory:")          # SQLite

# import psycopg
# conn = psycopg.connect("postgresql://...")  # PostgreSQL —— 下面的 cursor/execute 写法一致

cur = conn.cursor()
cur.execute("SELECT 1")
print(cur.fetchone())                        # 输出: (1,)
```

这正是 [01 设计哲学与心智模型](01_设计哲学与心智模型.md) 讲的"我们都是成年人"+ 鸭子类型哲学的延伸：**不靠 `interface` 强制，靠社区共识 + 文档约定**。代价是规范留了太多"可选项"和"实现自由"——最著名的就是 §3.2 要讲的**占位符风格 `paramstyle` 五花八门**（`qmark`/`format`/`pyformat`/`named`/`numeric`），这是 JDBC 单一 `?` 永远不会有的混乱。好处是驱动作者负担轻、生态繁荣，而 SQLAlchemy 这样的上层库可以在 DB-API 之上再抹平这些差异。

### 2.2 SQLAlchemy 的"两层"设计：Core + ORM，按需下沉

Java 世界里 MyBatis 和 Hibernate 是**两个对立的库**，你得在项目初期二选一：要么手写 SQL（MyBatis，控制力强、样板多），要么全自动映射（Hibernate，省事、但魔法多、性能黑盒）。

SQLAlchemy 的设计哲学是**把这两者做成同一个库的上下两层**，让你在**同一个项目、甚至同一个事务里自由切换**：

```
┌─────────────────────────────────────────────┐
│  ORM 层：对象 ↔ 行映射、Session、工作单元      │  ← 像 Hibernate
│  User(name="Alice"); session.add(u)          │
├─────────────────────────────────────────────┤
│  Core 层：SQL 表达式语言（Table/Column/select）│  ← 像 MyBatis 但类型安全
│  select(users).where(users.c.age > 18)        │
├─────────────────────────────────────────────┤
│  Engine + 连接池 + DBAPI 方言（dialect）       │  ← 抹平各驱动差异
├─────────────────────────────────────────────┤
│  DB-API 2.0 驱动（sqlite3 / psycopg / ...）    │  ← 像 JDBC 驱动
└─────────────────────────────────────────────┘
```

关键设计取舍：**ORM 是构建在 Core 之上的，不是另起炉灶。** ORM 的查询最终都翻译成 Core 的 SQL 表达式，再由 Core 编译成带占位符的 SQL 字符串交给 DB-API。这意味着：

- 90% 的 CRUD 用 ORM 写得很爽（对象进对象出）。
- 碰到复杂报表、批量更新、`WITH` 递归 CTE、窗口函数这种 ORM 表达不动或会生成低效 SQL 的场景，**直接下沉到 Core 写 SQL 表达式**，甚至 `text()` 写裸 SQL——而且能和 ORM 共用同一个连接、同一个事务。
- 你不必像"MyBatis vs Hibernate"那样在项目级别二选一，而是在**语句级别**选。

这是 SQLAlchemy 作者 Mike Bayer 的核心理念：**"不要把 SQL 藏起来，要让 SQL 触手可及。"** Hibernate 倾向于让你忘掉 SQL（直到性能出问题时你痛苦地发现根本不知道它生成了什么），SQLAlchemy 始终把 SQL 表达式作为一等公民摆在台面上。

### 2.3 工作单元（Unit of Work）+ 身份映射（Identity Map）

这两个模式都来自 Martin Fowler 的《企业应用架构模式》，Hibernate 和 SQLAlchemy ORM 都实现了它们——所以你对 JPA 的直觉大部分能迁移过来，但有几处关键差异值得先建立心智。

**工作单元**：你在 Session 里增删改对象时，**SQLAlchemy 不会立刻发 SQL**。它把"哪些对象是新增的、哪些被改了、哪些要删"记在 Session 里，攒成一个"待办清单"，直到 `flush`（或 `commit` 触发的 flush）那一刻，**一次性按依赖顺序生成所有 INSERT/UPDATE/DELETE 并发出去**。这和 Hibernate 的 flush 完全同源。好处：减少往返、能批量、能正确排序（先插父再插子以满足外键）。

**身份映射**：在**同一个 Session 内**，同一个主键的行**永远映射到内存里同一个 Python 对象**。

```python
u1 = session.get(User, 1)
u2 = session.get(User, 1)
assert u1 is u2          # True —— 同主键同对象，第二次 get 直接命中身份映射，不查库
```

这对应 JPA persistence context 的一级缓存。它保证了你在一个会话里对同一行的所有引用是一致的（改了一处，所有引用都看到）。

**和 JPA 最大的理念差异：事务边界更显式。** JPA 里 `@Transactional` 注解 + 容器管理事务，事务边界藏在框架里；SQLAlchemy 推崇**用 `with` 显式划定 Session 和事务的生命周期**（呼应 [15 异常与上下文管理器](15_异常与上下文管理器.md) 的事务上下文管理器模式）。你能一眼看到"事务从哪开始、到哪提交、异常时在哪回滚"，没有隐式魔法。这是 Python "显式优于隐式"哲学在持久层的体现。

---

## ③ 实现细节（底层怎么实现的）★本章重点★

### 3.1 DB-API 2.0 的对象模型：connection 与 cursor 的分工

DB-API 的核心就两个对象，对照 JDBC 看分工：

| DB-API 对象 | 职责 | JDBC 对应 | 关键差异 |
|------------|------|----------|---------|
| `connection` | 物理连接 + **事务边界**（`commit`/`rollback`/`close`） | `Connection` | 一致 |
| `cursor` | **执行语句** + **持有结果游标**（`execute`/`fetch*`） | `Statement` + `ResultSet` 合体 | JDBC 分两个对象，DB-API 合成一个 |

一次完整的查询生命周期：

```python
import sqlite3

conn = sqlite3.connect(":memory:")           # ① 建立连接
cur = conn.cursor()                          # ② 从连接拿一个 cursor

cur.execute("CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT, age INTEGER)")
cur.execute("INSERT INTO users(name, age) VALUES(?, ?)", ("Alice", 30))  # ③ 执行 + 参数
conn.commit()                                # ④ 提交事务（关键！见 §3.5）

cur.execute("SELECT id, name, age FROM users")
print(cur.description[0][0])                  # 输出: id —— description 描述结果列元数据
print(cur.fetchone())                         # 输出: (1, 'Alice', 30) —— 一行是个元组
print(cur.fetchall())                         # 输出: [] —— fetchone 已消费，游标后移，没了

cur.close()                                  # ⑤ 关闭 cursor
conn.close()                                 # ⑥ 关闭连接
```

几个 Java 老手会愣一下的点：

- **一行结果默认是元组**（`(1, 'Alice', 30)`），按列**位置**访问，不是按名字。想按名字取要设置 `conn.row_factory = sqlite3.Row`（见 §4.1），对应 JDBC 的 `rs.getString("name")`。
- **`cursor` 是有状态游标**：`fetchone()` 取一行并后移，`fetchmany(n)` 取 n 行，`fetchall()` 取剩余全部。取完就空了，再 fetch 返回 `None`/空列表。这正是 `ResultSet.next()` 的语义，只是 API 形态不同。
- **`description`** 是结果列的元数据（7 元组：name, type_code, ...），对应 JDBC 的 `ResultSetMetaData`。
- cursor 本身**可迭代**：`for row in cur:` 等价于反复 `fetchone()`，且是**流式**的（不会一次性把整个结果集塞进内存）——这点对大结果集很重要。

### 3.2 参数化查询：`paramstyle` 与防注入的底层机制

这是 DB-API 最容易让 Java 老手栽跟头的地方，也是安全的命门。

**第一个坑：占位符风格不统一。** DB-API 规定每个驱动模块要声明一个 `paramstyle` 属性，但**允许五种风格**：

```python
import sqlite3
print(sqlite3.paramstyle)        # 输出: qmark —— 用 ?

# import psycopg
# print(psycopg.paramstyle)      # pyformat —— 用 %s 和 %(name)s
```

| paramstyle | 占位符写法 | 典型驱动 |
|-----------|-----------|---------|
| `qmark` | `WHERE id = ?` | sqlite3 |
| `numeric` | `WHERE id = :1` | 部分 Oracle 驱动 |
| `named` | `WHERE id = :id` | sqlite3（也支持）、oracledb |
| `format` | `WHERE id = %s` | 旧 MySQLdb |
| `pyformat` | `WHERE id = %s` / `%(id)s` | psycopg、PyMySQL |

> 对照 JDBC：JDBC 只有 `?` 一种，换数据库占位符不用动。DB-API 这里**漏了一手**——换驱动可能要改占位符。这正是 §2.1 说的"弱约束规范"的代价，也是用 SQLAlchemy（它会按方言自动选对占位符）的一个理由。

**`%s` 不是字符串格式化！** psycopg 的 `%s` 看着像 Python 的 `%` 格式化，但**绝对不能**用 `%` 去拼。看清楚这个致命区别：

```python
# ✗✗✗ 灾难：用 Python 字符串格式化拼 SQL —— SQL 注入大门敞开
name = "Alice'; DROP TABLE users; --"
cur.execute("SELECT * FROM users WHERE name = '%s'" % name)   # 字符串先被拼好才传给 execute

# ✓✓✓ 正确：把参数作为 execute 的第二个实参，让驱动去做参数绑定
cur.execute("SELECT * FROM users WHERE name = ?", (name,))    # sqlite3
# cur.execute("SELECT * FROM users WHERE name = %s", (name,)) # psycopg —— %s 是占位符不是格式化
```

**底层防注入机制**：当你把参数交给 `execute` 的第二个实参时，**SQL 文本和参数值是分两条路走的**：

- SQL 文本（带占位符）走"语句"通道，被数据库**解析、编译成执行计划**；
- 参数值走"数据"通道，作为**已编译语句的绑定变量**填入。

数据库**先编译 SQL 结构、再填数据**，所以参数里哪怕含 `'; DROP TABLE`，也只会被当成一个**字符串字面值**去和 `name` 列比较，永远不可能改变 SQL 的语法结构。这就是参数化（prepared statement / bound parameters）防注入的本质，和 JDBC `PreparedStatement.setString` 完全同源。

而字符串拼接（`% name`、f-string、`+`）是在 SQL 抵达数据库**之前**就把恶意内容焊进了语句文本，数据库解析时把 `DROP TABLE` 当成了真正的语句——注入成功。

> 记死一条铁律：**SQL 的参数永远走 `execute` 的第二个实参，永远不要用任何字符串操作（`%`、`+`、f-string、`.format`）把变量塞进 SQL 文本。** 这条在 ORM 里同样成立（§3.7 会看到 ORM 怎么保证这点）。

`executemany` 批量绑定（对应 JDBC 的 `addBatch`/`executeBatch`）：

```python
rows = [("Bob", 25), ("Carol", 35), ("Dave", 40)]
cur.executemany("INSERT INTO users(name, age) VALUES(?, ?)", rows)   # 一次绑定多组参数
conn.commit()
```

### 3.3 SQLAlchemy ORM 工作单元：Session 如何跟踪对象状态

现在进入本章最硬核的部分——ORM 的工作单元到底怎么运转。先安装：

```bash
pip install "sqlalchemy>=2.0"
```

**对象的五种状态**（对照 JPA 的实体生命周期）。一个映射对象在 Session 眼里随时处于下面某个状态：

```
                 add()                  flush()                expunge()/session 关闭
  Transient ──────────────▶ Pending ──────────────▶ Persistent ──────────────▶ Detached
  （游离，刚 new            （已登记进 Session，      （已落库/已被 Session     （曾持久化，但脱离了
    出来，Session            但还没 INSERT，          管理，有主键，在            Session，身份映射
    不认识）                 工作单元待办里）          身份映射里）               里没它了）
                                                          │ delete()
                                                          ▼
                                                       Deleted ──flush──▶ （已发 DELETE）
```

亲手观察状态流转：

```python
from sqlalchemy import create_engine, String, Integer
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session
from sqlalchemy import inspect as sa_inspect

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    age: Mapped[int] = mapped_column(Integer)

engine = create_engine("sqlite://", echo=False)   # echo=True 可看到生成的 SQL
Base.metadata.create_all(engine)

with Session(engine) as session:
    u = User(name="Alice", age=30)
    print(sa_inspect(u).transient)     # 输出: True  —— 刚 new，游离态

    session.add(u)
    print(sa_inspect(u).pending)       # 输出: True  —— 已登记，待 flush
    print(u.id)                        # 输出: None  —— 还没 INSERT，没主键！

    session.flush()                    # 工作单元执行：发出 INSERT
    print(sa_inspect(u).persistent)    # 输出: True  —— 已落库
    print(u.id)                        # 输出: 1     —— 数据库生成的主键回填了

    session.commit()                   # 提交事务

# 离开 with，session 关闭，u 变成 detached
print(sa_inspect(u).detached)          # 输出: True
```

**关键认知**：`session.add(u)` **不发 SQL**，只是把 `u` 登记进 Session 的"新增清单"（pending）。真正发 INSERT 是在 `flush` 时。`flush` 可能被你显式调用，也可能被这些操作**自动触发**：`commit()` 之前会自动 flush；执行查询前（默认 `autoflush=True`）也会先 flush 把挂起的改动落库，保证查询看到最新状态。

**脏检查（dirty tracking）怎么实现的——呼应 19 章。** 你改一个已持久化对象的属性，Session 怎么知道它"脏"了？答案就是 [19 描述符与元类](19_描述符与元类.md) §4.5 讲过的 **`InstrumentedAttribute` 描述符**。SQLAlchemy 的元类（`DeclarativeBase` 背后）在类创建时，把每个 `mapped_column` 字段替换成一个**数据描述符**，它的 `__set__` 不只是存值，还会**把旧值记进一个历史、把对象标记进 Session 的 dirty 集合**：

```python
with Session(engine) as session:
    u = session.get(User, 1)
    print(u in session.dirty)          # 输出: False —— 还没改
    u.age = 31                         # 触发 InstrumentedAttribute.__set__ → 记录变更
    print(u in session.dirty)          # 输出: True  —— 描述符把它标记为脏
    session.commit()                   # flush 时只对脏对象生成 UPDATE，且只更新变了的列
```

这就是 19 章的描述符机制在工业框架里的真实落地：**字段是描述符**（拦截读写、做脏标记），**模型类由元类构造**（类创建时把声明式字段收集成表元数据 `__table__`、装上描述符）。你写 `u.age = 31` 看起来平平无奇，背后是描述符在替你记账。

**flush 时的依赖排序。** 工作单元 flush 时不是按你 add 的顺序发 SQL，而是**先做拓扑排序**：先 INSERT 被依赖的父行（满足外键约束），再 INSERT 子行；DELETE 则反过来。这正是为什么你可以先 `add(parent)` 再 `add(child)`、甚至顺序无所谓，SQLAlchemy 会算出正确的 INSERT 顺序。这是手写 SQL 时你得自己操心的事，工作单元替你做了。

### 3.4 身份映射：同主键同对象

```python
with Session(engine) as session:
    u1 = session.get(User, 1)          # 查库
    u2 = session.get(User, 1)          # 不查库！直接从身份映射返回同一对象
    print(u1 is u2)                    # 输出: True —— 同一个 Python 对象

    # 通过查询拿到的同主键对象，也是身份映射里那一个
    from sqlalchemy import select
    u3 = session.scalars(select(User).where(User.name == "Alice")).first()
    print(u1 is u3)                    # 输出: True —— 仍是同一对象
```

身份映射（identity map）是一个 `{(类, 主键): 对象}` 的字典（弱引用），挂在 Session 上。它保证：

1. **一致性**：一个会话里同一行只有一个对象，不会出现"两个 User(id=1) 各自被改、互相覆盖"。
2. **少查库**：`session.get(User, 1)` 命中身份映射时直接返回，省一次往返（对应 JPA 一级缓存）。

注意它是**会话级**的（对应 JPA persistence context 是 EntityManager 级）。换一个 Session，身份映射就是空的，会重新查库。它**不是**跨请求的二级缓存——别指望它当 Redis 用。

### 3.5 连接池：复用物理连接

建立一个数据库 TCP 连接 + 认证 + TLS 握手是昂贵的（毫秒级，对照 HikariCP 存在的全部理由）。SQLAlchemy 的 `Engine` **内置连接池**，默认 `QueuePool`：

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://user:pwd@localhost/mydb",
    pool_size=5,           # 池中常驻连接数（对应 HikariCP minimumIdle/maximumPoolSize 思路）
    max_overflow=10,       # 高峰允许临时超出的连接数（5+10=15 上限）
    pool_timeout=30,       # 等不到空闲连接的超时（秒），超时抛 TimeoutError
    pool_recycle=1800,     # 连接存活上限（秒），超过则丢弃重建（防 DB 端掐断空闲连接）
    pool_pre_ping=True,    # 取连接前先 ping 一下，剔除已失效的死连接（对应 HikariCP connectionTestQuery）
)
```

工作机制（和 HikariCP 几乎一一对应）：

- 你 `engine.connect()` 或 ORM 开 Session 时，从池里**借**一个物理连接；
- 用完（连接对象 `close()`）时，物理连接**不真的关闭**，而是**还回池里**复用；
- 池满且都在用时，新请求等待 `pool_timeout`，等不到就抛错；
- `pool_recycle` 防止"数据库端早把空闲连接掐了，你还拿着用导致 `connection closed` 错误"——这是生产环境的高频坑（MySQL `wait_timeout`、云数据库的空闲回收）。

> 关键差异：HikariCP 是**独立组件**，你单独配它再喂给 ORM。SQLAlchemy 的池**长在 Engine 里**，配置即连接串参数。`Engine` 是**进程级单例**——整个应用建一个 Engine，所有 Session 共享它的池。**别每次请求都 `create_engine`**（那等于每次重建连接池，§5 的坑）。

SQLite 的 `:memory:` 和文件库默认用 `SingletonThreadPool`/`StaticPool`，因为 SQLite 是嵌入式的、"连接"很廉价，语义不同——但生产用 Postgres/MySQL 时 `QueuePool` 是默认。

### 3.6 事务：DB-API 默认就在事务里

**最反 Java 直觉的一点：DB-API 连接默认开启事务，且不自动提交。** JDBC 默认 `autoCommit=true`（每条语句自己一个事务），你要手动 `setAutoCommit(false)` 才进显式事务。DB-API **反过来**——PEP 249 规定连接默认**不** autocommit，你执行的语句都在一个隐式事务里，**必须显式 `conn.commit()` 才落库**，否则连接关闭时改动全部丢失（回滚）。

```python
import sqlite3
conn = sqlite3.connect("test.db")
cur = conn.cursor()
cur.execute("INSERT INTO users(name, age) VALUES(?, ?)", ("Eve", 28))
# 忘记 conn.commit()！
conn.close()
# 重新打开查 —— Eve 根本不在！因为没 commit，关闭时事务回滚了
```

这是从 Java 转过来最高频的"数据莫名其妙没保存"事故。规则：**写操作之后必须 `commit()`，出错时 `rollback()`。** 标准模式是用 `with`（对应 [15 章](15_异常与上下文管理器.md) §5 综合实战的事务上下文管理器）：

```python
import sqlite3
# sqlite3 的 connection 本身是上下文管理器：with 块正常结束自动 commit，异常自动 rollback
conn = sqlite3.connect("test.db")
with conn:                                  # 注意：with conn 管的是事务，不是关闭连接！
    conn.execute("INSERT INTO users(name, age) VALUES(?, ?)", ("Frank", 33))
    # 正常退出 → 自动 commit；抛异常 → 自动 rollback
conn.close()                                # 关闭连接要单独做
```

> 小陷阱：`with conn:` 在 sqlite3 里管的是**事务边界**（commit/rollback），**不**负责 `close()` 连接——这和 `with open(...)` 退出就关文件不一样。容易看走眼。

### 3.7 ORM 查询如何编译成 SQL：`User.name == "Alice"` 的魔法

为什么 `User.name == "Alice"` 不是返回 `True`/`False`，而是返回一个能塞进 `where()` 的"SQL 条件对象"？答案又是 [19 描述符与元类](19_描述符与元类.md)。

`User.name` 这种**类级**属性访问（不是实例 `u.name`），触发的是 `InstrumentedAttribute` 描述符的 `__get__`，且此时 `obj is None`（19 章 §3.4 的细节：通过类访问描述符，`__get__` 的 `obj` 参数是 `None`）。SQLAlchemy 在 `obj is None` 时返回一个**特殊的 SQL 表达式对象**（`InstrumentedAttribute` 本身就支持比较运算符重载）。这个对象重载了 `__eq__`/`__gt__`/`__lt__` 等：

```python
from sqlalchemy import select

# 类级访问 User.name 返回 SQL 表达式对象，不是字符串
expr = User.name == "Alice"
print(type(expr))                  # 输出: <class 'sqlalchemy.sql.elements.BinaryExpression'>
print(str(expr))                   # 输出: users.name = :name_1  ← 它知道怎么变成带占位符的 SQL！

stmt = select(User).where(User.age > 18)
print(str(stmt))
# 输出（节选）: SELECT users.id, users.name, users.age FROM users WHERE users.age > :age_1
```

注意 `:age_1` 是个**占位符**——这就是 §3.2 的防注入在 ORM 层的保证：**ORM 永远生成参数化 SQL，你的值 `18` 是绑定参数，不是拼进字符串的。** 用 ORM 写查询天然免疫 SQL 注入（除非你手贱用 `text()` 又去拼字符串）。

一句话串起 19 章：**`u.name`（实例级）走描述符 `__get__` 返回值、`u.name = x` 走 `__set__` 做脏标记；`User.name`（类级）走描述符 `__get__` 返回 SQL 表达式对象。同一个描述符，靠 `obj` 是不是 `None` 区分两种行为。** 这就是 ORM"魔法"的全部秘密，没有编译器黑魔法。

### 3.8 惰性加载与 N+1：成因的源码级解剖

这是 Hibernate 和 SQLAlchemy 共有的头号性能杀手，根因完全相同。先建一对多关系：

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    books: Mapped[list["Book"]] = relationship(back_populates="author")

class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(100))
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped["Author"] = relationship(back_populates="books")
```

`relationship` 默认是**惰性加载（`lazy="select"`）**：`author.books` 这个属性，在你**第一次访问它**之前不会查库；一旦访问，**临时发一条 `SELECT * FROM books WHERE author_id = ?`**。这又是描述符——`books` 是个 relationship 描述符，它的 `__get__` 在被访问时才触发查询。

N+1 就这么诞生：

```python
# ✗ N+1 翻车现场
with Session(engine) as session:
    authors = session.scalars(select(Author)).all()    # ① 1 条 SQL：查所有作者
    for author in authors:                               # 假设有 N 个作者
        print(author.name, len(author.books))            # ② 每次访问 .books 各发 1 条 SQL
    # 总共 1 + N 条 SQL！作者越多，往返越多，延迟线性爆炸
```

**源码级成因**：第 ① 步只查了 `authors` 表，`author.books` 此刻是"未加载"状态。循环里每访问一个 `author.books`，relationship 描述符发现没加载过，就**当场补一条 SELECT**。N 个作者 = N 条额外 SELECT，加上最初那条 = N+1。这和 Hibernate 的 `LAZY` 集合在循环里被逐个触发是**一模一样的机制**。

**解法：预加载（eager loading）。** 告诉 SQLAlchemy"我接下来要用 books，请提前一次性load 好"。两种主力策略：

```python
from sqlalchemy.orm import selectinload, joinedload

# 方案 A：selectinload —— 用 2 条 SQL（推荐用于一对多/多对多）
with Session(engine) as session:
    authors = session.scalars(
        select(Author).options(selectinload(Author.books))
    ).all()
    # SQL ①: SELECT * FROM authors
    # SQL ②: SELECT * FROM books WHERE author_id IN (1,2,3,...)  ← 一条 IN 查询捞回所有子行
    for author in authors:
        print(author.name, len(author.books))   # 不再发 SQL，books 已在内存
    # 总共永远是 2 条 SQL，与作者数无关

# 方案 B：joinedload —— 用 1 条 SQL（LEFT JOIN，适合多对一/一对一）
with Session(engine) as session:
    books = session.scalars(
        select(Book).options(joinedload(Book.author))
    ).all()
    # SQL: SELECT ... FROM books LEFT OUTER JOIN authors ON books.author_id = authors.id
    for book in books:
        print(book.title, book.author.name)      # author 已随 JOIN 一起load，不发 SQL
```

**`selectinload` vs `joinedload` 怎么选**（实战经验）：

| 策略 | SQL 形态 | 适合 | 坑 |
|------|---------|------|-----|
| `selectinload` | 2 条（主查询 + `WHERE IN` 子查询） | **一对多 / 多对多**（集合） | 几乎总是一对多的最优解 |
| `joinedload` | 1 条 `LEFT JOIN` | **多对一 / 一对一**（标量） | 一对多时 JOIN 会让父行**笛卡尔重复**，传输放大、还要去重 |

对照 Hibernate：`selectinload` ≈ `@BatchSize` / `JOIN FETCH` 的子查询变体，`joinedload` ≈ `JOIN FETCH`。**经验法则：集合关系（一对多）用 `selectinload`，标量关系（多对一）用 `joinedload`。**

### 3.9 同步驱动 vs asyncpg 异步驱动

回顾 [21 异步编程asyncio](21_异步编程asyncio.md) §3.5 的铁律：**在 asyncio 协程里调用同步阻塞的数据库驱动会卡死整个事件循环。** psycopg2/sqlite3 这些同步驱动的 `execute` 会阻塞线程直到数据库返回，期间事件循环转不动，所有协程饿死。

所以异步后端必须用**异步驱动**，它们把"等数据库响应"做成 `await` 让出点：

- **asyncpg**：纯异步的 PostgreSQL 驱动，自己实现了 Postgres 二进制协议（不走 DB-API，API 是 async 的），以速度著称。
- **async SQLAlchemy 2.0**：在 ORM 之上提供 `AsyncEngine`/`AsyncSession`，底层接 asyncpg。

```bash
pip install asyncpg sqlalchemy
```

asyncpg 裸用（注意 API 形态和 DB-API 不同——它是为 asyncio 重新设计的，占位符用 `$1`/`$2`）：

```python
import asyncio
import asyncpg     # pip install asyncpg

async def main():
    conn = await asyncpg.connect("postgresql://user:pwd@localhost/mydb")
    # asyncpg 用 $1/$2 编号占位符（PostgreSQL 原生风格），不是 ? 或 %s
    rows = await conn.fetch("SELECT id, name FROM users WHERE age > $1", 18)
    for r in rows:
        print(r["id"], r["name"])    # asyncpg 的 Record 支持按列名访问
    await conn.close()

# asyncio.run(main())   # 需要真实 Postgres
```

它的并发威力（呼应 21 章 §4.7 的限流模式）：用连接池 + `asyncio.gather` 并发打多条查询，事件循环在等每条 SQL 时切去发别的，吞吐远超同步串行：

```python
async def main():
    pool = await asyncpg.create_pool("postgresql://...", min_size=5, max_size=20)
    async with pool.acquire() as conn:           # 从异步池借连接
        await conn.fetch("SELECT 1")
    # 并发跑多条查询，互相不阻塞
    async def q(i):
        async with pool.acquire() as conn:
            return await conn.fetchval("SELECT $1::int * 2", i)
    results = await asyncio.gather(*(q(i) for i in range(100)))
    await pool.close()
```

async SQLAlchemy（同样的 ORM，只是 `await`）：

```python
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy import select

async def main():
    engine = create_async_engine("postgresql+asyncpg://user:pwd@localhost/mydb")
    async with AsyncSession(engine) as session:          # async with！
        result = await session.scalars(select(User).where(User.age > 18))
        for u in result:
            print(u.name)
        await session.commit()
    await engine.dispose()

# asyncio.run(main())
```

> 异步 ORM 的一个额外约束：**惰性加载在 async 下会出问题**——`author.books` 的惰性触发需要发 SQL，但 async 下属性访问 `__get__` 不能 `await`，会抛错。所以**异步 SQLAlchemy 里必须用 `selectinload`/`joinedload` 显式预加载**，不能依赖惰性加载。这把 §3.8 的 N+1 解法从"性能优化"升级成了"async 下的硬性要求"。

---

## ④ 使用方法与最佳实践

### 4.1 sqlite3 快速上手（可直接跑，零依赖）

sqlite3 是标准库内置的，无需 pip，适合本地开发、测试、原型。完整 CRUD：

```python
import sqlite3

conn = sqlite3.connect(":memory:")           # 内存库；换成文件路径即持久化
conn.row_factory = sqlite3.Row               # ★让行支持按列名访问（像 ResultSet.getString("name")）

conn.execute("""
    CREATE TABLE orders(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        customer TEXT NOT NULL,
        amount REAL NOT NULL
    )
""")

# Create —— 永远参数化
conn.execute("INSERT INTO orders(customer, amount) VALUES(?, ?)", ("Alice", 99.5))
conn.executemany(
    "INSERT INTO orders(customer, amount) VALUES(?, ?)",
    [("Bob", 150.0), ("Carol", 30.0)],
)
conn.commit()

# Read —— row_factory=Row 后可按名取
for row in conn.execute("SELECT id, customer, amount FROM orders WHERE amount > ?", (50,)):
    print(row["id"], row["customer"], row["amount"])
# 输出:
# 1 Alice 99.5
# 2 Bob 150.0

# Update / Delete
conn.execute("UPDATE orders SET amount = amount * ? WHERE customer = ?", (1.1, "Alice"))
conn.execute("DELETE FROM orders WHERE amount < ?", (50,))
conn.commit()

print(conn.execute("SELECT COUNT(*) FROM orders").fetchone()[0])   # 输出: 2
conn.close()
```

### 4.2 SQLAlchemy 2.0 风格定义模型

2.0 风格用类型注解 + `Mapped[]` + `mapped_column`（比 1.x 的 `Column` 更类型安全，对照 14 章的类型注解理念）：

```python
from datetime import datetime
from sqlalchemy import String, ForeignKey, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)        # 类型来自注解，无需重复写 Integer
    name: Mapped[str] = mapped_column(String(50), unique=True)
    bio: Mapped[str | None] = mapped_column(default=None)     # Mapped[str | None] → 列可空
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    # 一对多：一个作者多本书
    books: Mapped[list["Book"]] = relationship(back_populates="author", cascade="all, delete-orphan")

class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(100))
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped["Author"] = relationship(back_populates="books")  # 多对一反向

    def __repr__(self) -> str:
        return f"Book(id={self.id!r}, title={self.title!r})"
```

要点：`Mapped[int]` 注解既给类型检查器看（[14 类型系统与注解](14_类型系统与注解.md)），也被 SQLAlchemy 元类读取来推断列类型；`Mapped[str | None]` 表示可空列；`relationship` 的 `back_populates` 让双向关系同步；`cascade="all, delete-orphan"` 让删作者级联删书（对应 JPA 的 `CascadeType.ALL` + `orphanRemoval`）。

### 4.3 用 `select()` 查询（2.0 统一风格）

2.0 废弃了 1.x 的 `session.query()`，统一用 `select()`：

```python
from sqlalchemy import create_engine, select, func
from sqlalchemy.orm import Session

engine = create_engine("sqlite://", echo=False)
Base.metadata.create_all(engine)

with Session(engine) as session:
    session.add_all([
        Author(name="Rowling", books=[Book(title="HP1"), Book(title="HP2")]),
        Author(name="Tolkien", books=[Book(title="LOTR")]),
    ])
    session.commit()

    # 基本查询：scalars 返回对象，first/all/one 取结果
    rowling = session.scalars(select(Author).where(Author.name == "Rowling")).one()
    print(rowling.name, len(rowling.books))      # 输出: Rowling 2

    # 条件、排序、分页
    stmt = (
        select(Author)
        .where(Author.name.like("%o%"))           # 列方法：like/in_/is_/between...
        .order_by(Author.name)
        .limit(10).offset(0)
    )
    for a in session.scalars(stmt):
        print(a.name)                             # 输出: Rowling \n Tolkien

    # 聚合 + JOIN + 分组（返回行元组而非对象，用 session.execute）
    stmt = (
        select(Author.name, func.count(Book.id).label("book_count"))
        .join(Book, Book.author_id == Author.id)
        .group_by(Author.id)
        .having(func.count(Book.id) >= 1)
    )
    for name, cnt in session.execute(stmt):
        print(f"{name}: {cnt} 本")                # 输出: Rowling: 2 本 \n Tolkien: 1 本
```

`scalars()` 用于"查一个实体、返回对象列表"；`execute()` 用于"查多列/聚合、返回 `Row` 元组"。

### 4.4 Session 用 `with` 管理事务边界

最佳实践模式（呼应 [15 章](15_异常与上下文管理器.md) 的事务上下文管理器）：

```python
from sqlalchemy.orm import Session

# 模式一：Session + 手动 commit（异常时 with 退出自动 rollback + 关闭）
with Session(engine) as session:
    session.add(Author(name="Asimov"))
    session.commit()                  # 显式提交；若这之前抛异常，with 退出会 rollback

# 模式二：session.begin() 自动管理事务（推荐，最像 @Transactional）
with Session(engine) as session:
    with session.begin():             # 进入开事务，正常退出自动 commit，异常自动 rollback
        session.add(Author(name="Clarke"))
        session.add(Author(name="Herbert"))
    # 这里事务已提交

# 模式三：合并写法
with Session(engine) as session, session.begin():
    session.add(Author(name="Le Guin"))
    # 正常 → commit；异常 → rollback；无论如何 → 关闭 session
```

> 推荐**模式二/三**：`session.begin()` 把"提交/回滚"交给上下文管理器，你不会忘 commit、也不会忘 rollback，事务边界一目了然——这正是 SQLAlchemy "显式事务边界"理念的落地。Web 框架里通常每个请求开一个 Session（用依赖注入，见 [24 HTTP与Web框架](24_HTTP与Web框架.md)），请求结束关闭。

### 4.5 连接池配置（生产）

```python
engine = create_engine(
    "postgresql+psycopg://user:pwd@db.internal:5432/app",
    pool_size=10,            # 常驻连接，按 (CPU核数 × 2 ~ 并发请求数) 估
    max_overflow=20,         # 峰值溢出
    pool_pre_ping=True,      # ★生产必开：取连接前 ping，自动剔除被 DB 掐断的死连接
    pool_recycle=1800,       # ★云数据库必设：小于 DB 的空闲超时，主动回收
    echo_pool=False,         # 调试连接池时设 True 看借还日志
)
# Engine 是进程级单例：建一次，全程序共用。绝不要每请求 create_engine！
```

> 经验：`pool_size + max_overflow` 别超过数据库的 `max_connections` 除以应用实例数。多个 worker 进程（gunicorn/uvicorn 多 worker）时，每个进程都有自己的池，总连接数 = 进程数 × (pool_size + max_overflow)，很容易把数据库连接数打爆——这是生产事故高发区。

### 4.6 Alembic 迁移流程（对照 Flyway）

Alembic 是 SQLAlchemy 官方迁移工具，对应 Flyway/Liquibase，但**迁移脚本是 Python**（能写复杂数据迁移逻辑，比纯 SQL 灵活）。

```bash
pip install alembic

alembic init migrations          # ① 初始化，生成 migrations/ 目录和 alembic.ini

# ② 编辑 migrations/env.py，把 target_metadata 指向你的 Base.metadata：
#    from myapp.models import Base
#    target_metadata = Base.metadata
#    并设置 sqlalchemy.url（或从环境变量读）

# ③ 自动生成迁移脚本（对比模型与数据库当前 schema 的差异）
alembic revision --autogenerate -m "create authors and books"

# ④ 应用迁移到最新版本（对应 Flyway migrate）
alembic upgrade head

# 回滚一步 / 到指定版本（Flyway 社区版没有的下迁能力）
alembic downgrade -1
alembic history          # 查看迁移版本链（对应 flyway info）
alembic current          # 当前数据库在哪个版本
```

自动生成的迁移脚本长这样（你应当 review，autogenerate 不是万能的——它检测不到列改名、某些约束变更）：

```python
# migrations/versions/xxxx_create_authors_and_books.py
def upgrade() -> None:
    op.create_table(
        "authors",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("name", sa.String(50), nullable=False),
    )
    op.create_index("ix_authors_name", "authors", ["name"], unique=True)

def downgrade() -> None:
    op.drop_index("ix_authors_name", "authors")
    op.drop_table("authors")
```

> 对照 Flyway：Flyway 版本号在文件名（`V1__xxx.sql`）、强制顺序；Alembic 用 `down_revision` 在脚本里串成**链表**（甚至支持分支合并），更灵活但也更容易在团队协作时产生"多个 head"冲突，需要 `alembic merge`。**铁律：迁移脚本进版本库、autogenerate 后必须人工 review、生产环境前在 staging 跑一遍。**

### 4.7 Core 写复杂 SQL（下沉到 MyBatis 那一层）

ORM 表达不动的复杂查询，下沉到 Core 用 `Table`/`select` 拼，或直接 `text()` 写裸 SQL——都能和 ORM 共用 Session/事务：

```python
from sqlalchemy import text

with Session(engine) as session:
    # 方式一：text() 写裸 SQL，但参数仍然走绑定（:threshold 占位符，不是字符串拼接！）
    result = session.execute(
        text("""
            SELECT a.name, COUNT(b.id) AS cnt
            FROM authors a LEFT JOIN books b ON b.author_id = a.id
            GROUP BY a.id
            HAVING COUNT(b.id) > :threshold
        """),
        {"threshold": 1},              # 参数化，安全
    )
    for name, cnt in result:
        print(name, cnt)
```

`text()` 里依然用 `:name` 占位符 + 参数字典，**绝不拼字符串**——防注入这条在 Core 层同样不能破。Core 的 SQL 表达式构造器（`select(table.c.x).where(...)`）则提供类型安全、可组合、自动选方言占位符的中间方案，是"既想要 SQL 控制力、又不想手拼字符串"时的首选。

### 4.8 避免 N+1 的实战清单

```python
# 1. 一对多集合 → selectinload
session.scalars(select(Author).options(selectinload(Author.books)))

# 2. 多对一标量 → joinedload
session.scalars(select(Book).options(joinedload(Book.author)))

# 3. 多层嵌套 → 链式
from sqlalchemy.orm import selectinload
session.scalars(
    select(Author).options(
        selectinload(Author.books).selectinload(Book.reviews)   # 作者→书→书评，3 条 SQL
    )
)

# 4. 只要部分字段，别拉整个对象 → 直接 select 列
session.execute(select(Author.name, Book.title).join(Book))     # 不触发 ORM 对象，更轻

# 5. 模型层全局默认预加载（谨慎，可能过度加载）
# books: Mapped[list["Book"]] = relationship(lazy="selectin")
```

> 排查 N+1 的方法：开 `echo=True` 看实际发出的 SQL 条数，或用 `sqlalchemy` 的事件钩子统计查询次数。生产用 APM（如 OpenTelemetry 的 SQLAlchemy instrumentation，见 [30 部署与可观测性](30_部署与可观测性.md)）监控单请求 SQL 数。

---

## ⑤ ⚠️ 易错点（Java 思维翻车）

| Java 习惯 | 翻车现场 | 正确做法 / 认知 |
|----------|---------|----------------|
| 用 `StringBuilder` 拼 SQL 是日常 | f-string / `%` / `+` 把变量拼进 SQL 文本，**SQL 注入门户大开**，`'; DROP TABLE--` 直接生效 | **参数永远走 `execute` 第二个实参**（`?`/`%s`/`:name` 占位符 + 参数序列），任何字符串操作拼 SQL 都是漏洞 |
| JDBC 默认 autoCommit=true | DB-API **默认在事务里且不自动提交**，写完忘 `commit()`，连接一关数据全没，"莫名其妙没保存" | 写操作后 `conn.commit()`；或用 `with conn`（sqlite3）/`session.begin()` 自动提交 |
| `EntityManager` 注入到处用、跨方法传 | **Session 跨线程/跨请求共享**：一个全局 Session 被多线程用，状态错乱、身份映射污染、`commit` 互相干扰 | **Session 非线程安全**，每线程/每请求一个 Session，用完即弃（web 框架靠依赖注入做 per-request session） |
| Hibernate 循环里访问 `LAZY` 集合 | `for a in authors: a.books` 触发 **N+1**，作者多了延迟爆炸 | 一对多 `selectinload`、多对一 `joinedload` 预加载；async 下更是**必须**预加载 |
| 信任连接池自动归还 | 手动 `engine.connect()` 后忘 `close()`，连接**泄漏不归还池**，池耗尽后所有请求卡死在 `pool_timeout` | 用 `with engine.connect()` / `with Session(engine)`，让上下文管理器保证归还 |
| 在 service 方法返回实体给上层用 | Session 关闭后对象变 **detached**，上层访问 `author.books`（惰性属性）抛 `DetachedInstanceError` | 在 Session 还活着时就预加载好需要的关系，或返回 DTO/dict（见 [26 日期时间序列化pydantic](26_日期时间序列化pydantic.md)） |
| 每次用 ORM 都 new 一个 EntityManagerFactory | **每请求 `create_engine`**，等于每次重建整个连接池，连接数暴涨、性能崩 | `Engine` 进程级单例，建一次全程序共享 |
| 把异步当"更快的同步" | 在 asyncio 里用 **同步驱动**（psycopg2/sqlite3）或同步 Session，阻塞事件循环，整个服务假死 | 异步用 asyncpg + `AsyncSession`，全链路 `await`（[21 异步编程asyncio](21_异步编程asyncio.md)） |
| 以为 SQLite 能扛并发写 | 多线程/多进程并发写 SQLite，频繁 `database is locked`（SQLite **写是全库串行锁**） | SQLite 适合开发/测试/单写者；生产高并发写用 Postgres/MySQL |
| 占位符照搬 `?` | 换到 psycopg 还写 `?`，报语法错（psycopg 是 `%s`） | 看驱动的 `paramstyle`；或用 SQLAlchemy 让它按方言自动选 |
| 改完对象忘了 flush 就查 | 关掉 `autoflush` 后改了对象直接查，查到的是旧数据（改动还没落库） | 保留默认 `autoflush=True`，或查询前手动 `session.flush()` |

补充演示——**最隐蔽的 `DetachedInstanceError`**（Hibernate 老手的肌肉记忆陷阱）：

```python
def get_author(author_id):
    with Session(engine) as session:
        return session.get(Author, author_id)     # 返回时 session 即将关闭

author = get_author(1)
# session 已关闭，author 是 detached 状态
print(author.name)            # OK —— 标量属性在 expire 前已加载，能读
print(author.books)           # ✗ DetachedInstanceError！books 是惰性关系，
                              #    需要发 SQL 但 session 没了，无法加载
```

修复——在 Session 存活期间就预加载需要的关系，或转成纯数据：

```python
def get_author(author_id):
    with Session(engine) as session:
        author = session.scalars(
            select(Author)
            .options(selectinload(Author.books))   # ★在 session 内就把 books 加载好
            .where(Author.id == author_id)
        ).one()
        # 此时 author.books 已在内存，detach 后仍可访问
        return author

author = get_author(1)
print(author.books)           # OK —— 已预加载，不需要再发 SQL
```

> Java 老手特别容易栽这个：Hibernate 里 `LazyInitializationException` 是同一个病（session/EntityManager 关闭后碰 lazy 属性）。心法一致：**要么在会话内加载完所有要用的关系，要么在会话内就转成 DTO**——别把"半加载的持久态对象"漏到会话外面去用。

---

## ⑥ 练习题

### 基础理解

**1.** DB-API 2.0 和 JDBC 都想做到"换数据库不换上层代码"，但实现方式根本不同。说清楚：① DB-API 用什么手段达成统一（对比 JDBC 的 `interface`）？② 这种手段带来的一个具体代价是什么（提示：占位符）？③ 为什么说 `connection` 管事务、`cursor` 管执行+结果，它和 JDBC 的对象划分差在哪？

**2.** 解释 SQLAlchemy ORM 的"工作单元"：`session.add(u)` 之后、`session.flush()` 之前，对象处于什么状态、数据库里有没有这行、`u.id` 是不是 None？flush 那一刻发生了什么？再解释"身份映射"保证了什么、它是会话级还是全局级。

### 找坑改错

**3.**（SQL 注入 + 事务）下面这段"按用户名查订单并标记已读"的代码有**三处**严重问题（一处安全漏洞、一处数据不落库、一处资源泄漏），逐一指出并改正：

```python
import sqlite3

def get_orders(username):
    conn = sqlite3.connect("app.db")
    cur = conn.cursor()
    # 拼接用户名查询
    cur.execute("SELECT * FROM orders WHERE customer = '" + username + "'")   # (a)
    rows = cur.fetchall()
    cur.execute("UPDATE orders SET read = 1 WHERE customer = '" + username + "'")  # (b)
    return rows                                                               # (c)
```

**4.**（N+1）下面这段"统计每个作者的藏书数"在生产日志里发现发了几百条 SQL。指出 N+1 的成因，给出**两种**不同的修复（一种用预加载、一种用聚合查询），并说明各自适合的场景：

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

def report(engine):
    with Session(engine) as session:
        authors = session.scalars(select(Author)).all()
        for a in authors:
            print(f"{a.name}: {len(a.books)} 本")    # 每次 a.books 触发一条 SQL
```

### 综合实战

**5.** 用 SQLAlchemy 2.0 建一个简易博客模型并完成一组操作，要求：
- 定义 `User`（id, name 唯一）和 `Post`（id, title, content, author_id 外键, created_at），一对多（一个 user 多篇 post），双向 `relationship`；
- 用 `Session` + `session.begin()` 做事务管理；
- 写一个 `create_user_with_posts(engine, name, titles)`：在一个事务里创建用户和若干文章，演示工作单元（一次 commit 落库所有行）；
- 写一个 `list_users_with_post_count(engine)`：**用预加载避免 N+1**，返回每个用户及其文章数，并打印实际发出的 SQL 条数概念（说明为什么是 2 条而非 1+N）；
- 写一个 `get_user_detail(engine, user_id)`：返回用户及其所有文章，**保证返回后在 session 外访问 `user.posts` 不报 `DetachedInstanceError`**；
- 用 `sqlite://`（内存库）让代码可直接运行，打印验证输出。

附：说明这个设计里"字段=描述符、模型类=元类构造"是怎么呼应 [19 描述符与元类](19_描述符与元类.md) 的，以及 `User.name == x` 为什么能拼成 SQL。

---

### 参考答案

**1.** ① DB-API 不用 `interface` 强制，而是一份**文档约定**（PEP 249）：规范说"驱动模块该有 `connect()`、connection 该有 `cursor()`/`commit()`、cursor 该有 `execute()`/`fetchone()`……"，驱动作者**自觉遵守**，靠鸭子类型而非编译期契约——这是 Python"我们都是成年人"哲学的体现。② 代价：规范留了太多自由，最典型的是**占位符风格 `paramstyle` 不统一**（sqlite3 用 `?`、psycopg 用 `%s`、oracledb 用 `:name`），换驱动可能要改占位符，而 JDBC 永远是 `?`。③ JDBC 把"执行语句"（`Statement`/`PreparedStatement`）和"结果集"（`ResultSet`）分成两个对象；DB-API 把这两者**合成一个 `cursor`**——cursor 既负责 `execute` 执行，又作为结果游标 `fetchone/fetchall`。`connection` 两边都管物理连接 + 事务边界（commit/rollback）。

**2.** `session.add(u)` 后、flush 前：对象处于 **pending（待定）** 状态——已登记进 Session 的"新增清单"（工作单元的待办），但**数据库里还没有这行**，INSERT 还没发出，所以 `u.id` 仍是 **None**（主键由数据库在 INSERT 时生成）。`flush()` 那一刻：工作单元执行，按依赖拓扑排序生成所有挂起的 INSERT/UPDATE/DELETE 并发给数据库，`u` 被 INSERT、数据库生成的主键**回填**到 `u.id`，对象转为 **persistent（持久）** 状态。注意 flush 不等于 commit——flush 只是把 SQL 发出去（事务仍可回滚），commit 才真正提交事务。"身份映射"保证：**在同一个 Session 内，同一主键的行永远映射到内存里同一个 Python 对象**（`session.get(User,1) is session.get(User,1)` 为 True），从而保证一致性 + 减少查库（命中即返回，不再发 SQL）。它是**会话级**的（对应 JPA persistence context），换一个 Session 就是空的，**不是全局/跨请求**的二级缓存。

**3.** 三处问题：
- **(a) SQL 注入**：用 `+` 把 `username` 拼进 SQL 文本，`username` 若是 `x' OR '1'='1` 就能拖走全表，`x'; DROP TABLE orders; --` 能删表。必须参数化：`cur.execute("SELECT * FROM orders WHERE customer = ?", (username,))`。
- **(b) 同样的注入 + 数据不落库**：UPDATE 也拼了字符串（注入），且**整个函数没有 `conn.commit()`**——DB-API 默认在事务里不自动提交，UPDATE 改动在连接关闭时会回滚，"标记已读"根本没生效。要参数化 + commit。
- **(c) 资源泄漏**：`conn` 从头到尾**没有 `close()`**（也没 `with`），连接泄漏；返回后无人关闭。应该用 `with` 管理。

修正版：
```python
import sqlite3

def get_orders(username):
    with sqlite3.connect("app.db") as conn:         # with conn 管事务（commit/rollback）
        conn.row_factory = sqlite3.Row
        rows = conn.execute(
            "SELECT * FROM orders WHERE customer = ?", (username,)    # 参数化
        ).fetchall()
        conn.execute(
            "UPDATE orders SET read = 1 WHERE customer = ?", (username,)
        )
        # with conn 正常退出 → 自动 commit；异常 → rollback
        result = [dict(r) for r in rows]            # 转成普通 dict，脱离游标也能用
    conn.close()                                    # 注意 with conn 不关连接，单独关
    return result
```
（注：sqlite3 的 `with conn` 只管事务边界、不关连接，所以仍需 `conn.close()`；更稳的写法是用 `contextlib.closing(conn)` 再嵌 `with conn`。）

**4.** **成因**：`relationship` 默认惰性加载（`lazy="select"`）。第一条 SQL 只查了 `authors`，循环里每访问一次 `a.books`，relationship 描述符发现未加载就**当场补发**一条 `SELECT * FROM books WHERE author_id = ?`。N 个作者 = 1（查作者）+ N（逐个查书）= **N+1 条**。

修复一（**预加载**，要拿到对象本身/还要用 books 里的数据时）：
```python
from sqlalchemy.orm import selectinload
def report(engine):
    with Session(engine) as session:
        authors = session.scalars(
            select(Author).options(selectinload(Author.books))   # 一对多用 selectinload
        ).all()
        # 固定 2 条 SQL：SELECT authors + SELECT books WHERE author_id IN (...)
        for a in authors:
            print(f"{a.name}: {len(a.books)} 本")                 # books 已在内存，不发 SQL
```

修复二（**聚合查询**，只要计数、不需要 book 对象时——最省）：
```python
from sqlalchemy import func
def report(engine):
    with Session(engine) as session:
        stmt = (
            select(Author.name, func.count(Book.id))
            .outerjoin(Book, Book.author_id == Author.id)
            .group_by(Author.id)
        )
        for name, cnt in session.execute(stmt):     # 1 条 SQL，数据库侧 COUNT
            print(f"{name}: {cnt} 本")
```
场景区别：**预加载**适合"既要计数、又要遍历每本书的内容"（books 对象要在内存里用）；**聚合查询**适合"只要数量"——直接让数据库 `COUNT`，1 条 SQL、不把 book 行拉到 Python，最高效。原则：只要计数就别拉整张子表。

**5.**
```python
from datetime import datetime
from sqlalchemy import create_engine, String, Text, ForeignKey, func, select
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship, Session, selectinload,
)

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    posts: Mapped[list["Post"]] = relationship(
        back_populates="author", cascade="all, delete-orphan"
    )

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(100))
    content: Mapped[str] = mapped_column(Text, default="")
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    author: Mapped["User"] = relationship(back_populates="posts")

    def __repr__(self) -> str:
        return f"Post(title={self.title!r})"


def create_user_with_posts(engine, name, titles):
    with Session(engine) as session, session.begin():        # 一个事务
        user = User(name=name, posts=[Post(title=t) for t in titles])
        session.add(user)                                     # pending：还没发 SQL
        # 退出 session.begin() 时一次性 flush + commit：
        # 工作单元先 INSERT user 拿到 id，再按外键依赖 INSERT 所有 post
    # 一次 commit 落库了 1 个 user + len(titles) 个 post


def list_users_with_post_count(engine):
    with Session(engine) as session:
        users = session.scalars(
            select(User).options(selectinload(User.posts))    # ★预加载，避免 N+1
        ).all()
        # 固定 2 条 SQL：① SELECT users ② SELECT posts WHERE user_id IN (...)
        # 而不是 1（查 users）+ N（逐个查 posts）
        return [(u.name, len(u.posts)) for u in users]


def get_user_detail(engine, user_id):
    with Session(engine) as session:
        user = session.scalars(
            select(User).options(selectinload(User.posts))    # 会话内加载好 posts
            .where(User.id == user_id)
        ).one()
        return user            # detached 后 posts 已在内存，访问不报错


if __name__ == "__main__":
    engine = create_engine("sqlite://", echo=False)
    Base.metadata.create_all(engine)

    create_user_with_posts(engine, "Alice", ["Hello World", "On Python"])
    create_user_with_posts(engine, "Bob", ["First Post"])

    print(list_users_with_post_count(engine))
    # 输出: [('Alice', 2), ('Bob', 1)]

    detail = get_user_detail(engine, 1)
    print(detail.name, [p.title for p in detail.posts])   # session 外访问 posts，不报错
    # 输出: Alice ['Hello World', 'On Python']
```

设计说明（呼应 [19 描述符与元类](19_描述符与元类.md)）：

- **字段 = 描述符**：每个 `mapped_column`/`relationship` 在类创建时被 SQLAlchemy 装成 `InstrumentedAttribute`（**数据描述符**）。`user.name = x` 走它的 `__set__`，不仅存值还**记入 Session 的脏集合**（脏检查的实现）；`post.author` 走 relationship 描述符的 `__get__`，惰性触发查询（N+1 的来源）。这正是 19 章 §4.5 讲的工业级描述符应用。
- **模型类 = 元类构造**：`DeclarativeBase` 背后有元类，在 `class User(Base)` 执行的那一刻**遍历类体**，读取 `Mapped[]` 注解推断列类型、把声明式字段收集成 `User.__table__` 表元数据、给每个字段装上描述符——这就是 19 章"模型类由元类在创建时收集字段、织入框架能力"的真实落地。
- **`User.name == x` 为什么能拼 SQL**：`User.name` 是**类级**属性访问，触发描述符 `__get__` 且 `obj is None`（19 章 §3.4），此时返回一个重载了 `__eq__` 的 **SQL 表达式对象**（`BinaryExpression`），它 `str()` 出来是 `users.name = :name_1`——一个**带占位符的参数化片段**。这既解释了 ORM 查询的"魔法"，也保证了 ORM 天然防注入（值永远是绑定参数，不拼进字符串）。

理解了这 60 行，你再去读 SQLAlchemy 的 `InstrumentedAttribute`、声明式元类源码，会发现它就是 19 章那套"描述符 + 类创建时收集字段"心智模型的工程化放大版。

---

下一章：[26 日期时间序列化pydantic](26_日期时间序列化pydantic.md) — `datetime`/时区的坑、JSON 序列化、以及 pydantic 怎么用同一套"描述符 + 元类"做数据校验与 ORM 对象到 DTO 的转换（正好接上本章"返回 DTO 避免 detached 坑"的伏笔）。相关回顾：[15 异常与上下文管理器](15_异常与上下文管理器.md)（事务上下文管理器）、[19 描述符与元类](19_描述符与元类.md)（ORM 字段/模型的底层机制）、[21 异步编程asyncio](21_异步编程asyncio.md)（asyncpg 与异步 Session）。
