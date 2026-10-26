# 数据库与 SQLAlchemy

## 本章导读

- 关系型数据库的底层基石：B+ 树索引为什么快、事务 ACID 到底保证什么、四种隔离级别分别挡住哪些"怪现象"
- SQLAlchemy 2.0 的核心三件套：Engine、连接池、Session 各自负责什么、生命周期怎么管
- 用 `DeclarativeBase` + `Mapped` + `mapped_column` 定义类型安全的 ORM 模型
- 2.0 新风格的增删改查：`select` / `insert` / `update` / `delete`、过滤、排序分页、聚合
- 表关系建模：一对多、多对多、外键，以及 N+1 问题的成因与 `selectinload` / `joinedload` 解决方案
- 在 FastAPI 中正确集成：同步 Session 依赖注入范式与异步 `create_async_engine` 全家桶
- 用 Alembic 管理数据库结构变迁：autogenerate 的原理与局限
- 事务管理的正确姿势：`commit` / `rollback` / `begin` 上下文，以及长事务的危害
- 性能与排错：原生 SQL、EXPLAIN、慢查询、会话泄漏、时区与 utf8mb4

---

## 一、关系型数据库基础

在写任何一行 SQLAlchemy 代码之前，先把数据库本身的几个核心机制搞清楚。ORM 只是"翻译官"，真正干活的是数据库；不理解底层，遇到慢查询和并发 bug 时只能抓瞎。

### 1.1 选 MySQL 还是 PostgreSQL

两者都是成熟的开源关系型数据库。简单一句推荐：**新项目优先 PostgreSQL**（功能更全、JSONB/数组/全文检索等高级特性丰富、许可更宽松）；团队已有 MySQL 运维积累或需要对接既有生态时用 MySQL 也完全没问题。本章示例对两者通用，差异处会特别说明。

### 1.2 索引：B+ 树为什么快

没有索引时，查找一行数据只能**全表扫描**——把整张表从头到尾读一遍，10 万行就比对 10 万次，时间复杂度 O(n)。

数据库普遍采用 **B+ 树**作为索引结构。它是一棵"矮胖"的多叉平衡树：

- **每个节点是一个磁盘页**（InnoDB 默认 16KB）。一个页能存几百个索引键，所以 100 万行数据的索引通常只有 **3 层**高。查找任何一行最多读 3 次磁盘页，而不是读 100 万次。
- **只有叶子节点存数据**（或指向数据的指针），且所有叶子节点之间用**双向链表**串起来。这就是为什么 `WHERE age BETWEEN 18 AND 30` 这种**范围查询**也很快：定位到 18 所在叶子后，顺着链表一路扫到 30 即可。
- **所有叶子在同一层**，任何查询的磁盘 IO 次数相同，性能可预测。

由此推出几条实战规则：

1. **索引是有序的**。`WHERE email = 'a@b.com'` 能走索引，是因为 B+ 树按键值排序，可以快速定位。但 `WHERE email LIKE '%@b.com'`（前导通配符）破坏了有序性，只能全表扫描。
2. **联合索引遵循最左前缀**。索引 `(a, b, c)` 相当于先按 a 排序、a 相同再按 b 排序。`WHERE a = 1 AND b = 2` 能用上；只查 `b = 2` 则不行——好比查字典直接按第二个字的拼音找。
3. **索引不是免费的**。每次 INSERT/UPDATE 都要维护 B+ 树（插入节点、可能分裂页），索引越多写入越慢，还占磁盘。给高频过滤、排序、连接的列建索引，低频列不要建。
4. **主键本身就是聚簇索引**（InnoDB）：数据行直接挂在主键 B+ 树的叶子节点上。普通索引的叶子存的是主键值，用它查询要先走普通索引、再按主键回表查数据，这叫**回表**。如果普通索引已经包含查询所需的全部列，就不需要回表，这叫**覆盖索引**，是常见优化手段。

### 1.3 事务与 ACID

**事务（Transaction）**是一组"要么全部成功、要么全部不发生"的操作。经典例子：转账——A 扣 100 元、B 加 100 元，任何一步失败，整个操作必须回滚，否则钱就凭空消失或出现了。

ACID 是事务的四个性质：

- **A（Atomicity，原子性）**：事务内的操作是不可分割的整体。实现靠 **undo log**：每步修改前先记录旧值，失败时按日志逆序还原。
- **C（Consistency，一致性）**：事务把数据库从一个合法状态带到另一个合法状态，约束（唯一键、外键、CHECK）始终成立。一致性是目的，AID 是手段。
- **I（Isolation，隔离性）**：并发事务之间不能互相干扰。实现靠**锁**和 **MVCC**（多版本并发控制），隔离到什么程度由隔离级别决定（下节讲透）。
- **D（Durability，持久性）**：一旦 COMMIT 成功，数据永不丢失，哪怕数据库立刻宕机。实现靠 **redo log（WAL，预写式日志）**：先写日志到磁盘，再异步刷数据页。commit 慢往往就慢在这一步的磁盘 IO 上。

### 1.4 四种隔离级别与三种"怪现象"

并发事务交叉执行时，会出现三种经典的数据异常。先看现象，再看哪个隔离级别能挡住它。

**脏读（Dirty Read）**：读到了别的事务**还没提交**的修改。

```sql
-- 事务 A                                  -- 事务 B
BEGIN;
UPDATE account SET balance = 0 WHERE id = 1;   -- 扣钱，还没 COMMIT
                                          BEGIN;
                                          SELECT balance FROM account WHERE id = 1;
                                          -- 读到 0！但这是 A 未提交的"脏"数据
ROLLBACK;                                   -- A 撤销，真实余额还是 100
```

B 读到的"余额 0"根本不曾真实存在过。如果 B 基于这个值做决策（比如拒绝支付），就做出了错误决策。

**不可重复读（Non-repeatable Read）**：同一事务内，两次读同一条记录，结果不一样——因为期间别的事务**修改并提交**了这行。

```sql
-- 事务 A                                  -- 事务 B
BEGIN;
SELECT balance FROM account WHERE id = 1;   -- 读到 100
                                          BEGIN;
                                          UPDATE account SET balance = 50 WHERE id = 1;
                                          COMMIT;
SELECT balance FROM account WHERE id = 1;   -- 读到 50，同一事务内两次结果不同！
COMMIT;
```

**幻读（Phantom Read）**：同一事务内，两次执行**相同的范围查询**，返回的行数不一样——因为期间别的事务**插入或删除**了符合条件的行。

```sql
-- 事务 A                                  -- 事务 B
BEGIN;
SELECT COUNT(*) FROM account WHERE balance > 10;   -- 5 行
                                          INSERT INTO account VALUES (6, 999);  -- 余额 999 > 10
                                          COMMIT;
SELECT COUNT(*) FROM account WHERE balance > 10;   -- 6 行，"幻影"出现
COMMIT;
```

注意区分：不可重复读针对**已存在的行被修改**，幻读针对**符合条件的行集合变了**（新增/删除）。

SQL 标准定义了四种隔离级别，从松到严：

1. **READ UNCOMMITTED（读未提交）**：什么都不挡，脏读、不可重复读、幻读都可能发生。几乎没人用。
2. **READ COMMITTED（读已提交）**：只能读到已提交的数据，挡住**脏读**。但同一事务内每次查询都看最新已提交版本，所以仍有不可重复读和幻读。这是 **PostgreSQL 的默认级别**。
3. **REPEATABLE READ（可重复读）**：事务开始后看到的数据快照固定不变，挡住脏读和不可重复读。这是 **MySQL InnoDB 的默认级别**——而且 InnoDB 通过 MVCC 快照 + next-key lock，在这一级也基本挡住了幻读（比 SQL 标准要求更强）。
4. **SERIALIZABLE（可串行化）**：效果等同于事务一个接一个排队执行，三种异常全部杜绝。代价是并发性能最差，可能频繁锁等待和死锁。

底层的实现机制值得理解：主流数据库用 **MVCC（多版本并发控制）**——每行数据保留多个历史版本，读操作根据自己的事务 ID 选择合适的版本去读，**读写互不阻塞**。READ COMMITTED 每次查询生成一个新快照；REPEATABLE READ 在事务第一次查询时生成快照，整个事务复用。这就是"可重复读"的实现原理，也解释了为什么隔离级别越高、数据库要保留的旧版本越多、开销越大。

实践建议：绝大多数应用用默认级别即可，把"余额不能扣成负数"这类硬约束交给**行锁（`SELECT ... FOR UPDATE`）**、唯一约束或应用层校验，而不是盲目调高隔离级别。

---

## 二、SQLAlchemy 2.0 核心概念

SQLAlchemy 分两层：**Core**（SQL 表达式语言，接近写 SQL）和 **ORM**（把表映射为 Python 类）。2.0 版本统一了两层的查询风格，全部以 `select()` 函数为入口，并原生支持 asyncio。本章全部使用 2.0 新风格。

安装：

```bash
pip install sqlalchemy
# MySQL 驱动（同步）
pip install pymysql
# PostgreSQL 驱动（同步）
pip install psycopg2-binary
# 异步驱动见第六节
```

### 2.1 Engine：数据库的入口

`Engine` 是 SQLAlchemy 与数据库对话的起点，封装了**方言（dialect）**和**连接池**。

```python
from sqlalchemy import create_engine

# URL 格式：方言+驱动://用户名:密码@主机:端口/库名
engine = create_engine(
    "mysql+pymysql://root:123456@localhost:3306/demo?charset=utf8mb4",
    echo=True,        # 打印所有执行的 SQL，学习时打开，生产环境关掉
    pool_size=5,      # 连接池常驻连接数
    max_overflow=10,  # 超出 pool_size 后最多再临时创建 10 个连接
    pool_timeout=30,  # 从池中取连接最多等 30 秒，超时抛异常
    pool_recycle=3600,  # 连接最多存活 1 小时，防止 MySQL 的 wait_timeout 掐断空闲连接
)
```

`create_engine` 只是创建了一个"工厂"，**此时并没有真正连接数据库**，第一次执行 SQL 时才建立连接。

### 2.2 连接池原理

每次执行 SQL 都新建一个 TCP 连接 + 数据库端做一次认证握手，开销是毫秒级的，高并发下完全不可接受。**连接池**的思路是：提前建好一批连接放着，用完不关闭，归还池子供下次复用。

工作流程：

1. 应用调用 `engine.connect()` 或创建 Session → 池里有空闲连接就直接拿走，没有且未超上限就新建，已达上限就排队等待（最多 `pool_timeout` 秒）。
2. 用完后连接被**归还给池子**而不是关闭，TCP 连接保持存活，供下一个请求零成本复用。
3. `pool_recycle` 定期强制重建连接，避免拿到已被服务端单方面断开的"死连接"（MySQL 默认 8 小时踢掉空闲连接）。

要点：**连接池是进程级的**。Web 服务每个工作进程各自持有一个池，池大小要根据"进程数 × pool_size"来规划，别超过数据库的 `max_connections`。

### 2.3 Session：ORM 的工作单元

如果说 Engine 管"怎么连"，**Session 管"怎么用对象操作数据"**。它是 ORM 层的核心，实现了**工作单元（Unit of Work）**和**身份映射（Identity Map）**两个模式：

- **身份映射**：同一个 Session 内，主键相同的记录永远对应**同一个 Python 对象**。查询两次 `id=1` 的用户，得到的是内存里的同一实例，不会产生不一致的副本。
- **工作单元**：你对对象的修改（`user.name = "x"`、添加 `session.add(obj)`）先记在 Session 的"账本"里，直到 `session.commit()` 时才一次性生成 SQL、包在一个事务里发给数据库。`flush()` 则是"提前把账本同步到数据库但不提交"，ORM 内部会在需要时自动 flush（比如查询前，保证你能查到刚 add 的对象）。

Session 的生命周期范式——**请求级 Session**：

```python
from sqlalchemy.orm import Session

with Session(engine) as session:   # 从池取连接，开始一个工作单元
    user = session.get(User, 1)    # 按主键查询
    user.name = "张三"              # 只是改 Python 对象的属性，还没发 SQL
    session.commit()               # 此时生成 UPDATE，提交事务
# with 块结束：session.close() 自动调用，连接归还连接池
```

关键纪律：

- **Session 不是数据库连接**，它按需从池里借连接，`close()` 时归还。
- **Session 不是线程安全的**，绝不能在多线程/多协程间共享同一个 Session 实例。
- **用完必须关闭**。Web 应用里一个请求用一个 Session，请求结束立即关闭（后面 FastAPI 章节用依赖注入自动化这一步）。
- 抛异常后 Session 进入"半死不活"状态，必须 `rollback()` 后再用，或者干脆关闭换新。

---

## 三、ORM 模型定义

### 3.1 DeclarativeBase 与现代声明式基类

SQLAlchemy 2.0 用 `DeclarativeBase` 定义模型基类，配合 `Mapped` 类型注解，让模型字段同时被 **mypy/IDE 理解**，这是 2.0 相对 1.x 最大的体验升级。

```python
from datetime import datetime

from sqlalchemy import String, ForeignKey, Text, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    """所有模型的公共基类"""


class User(Base):
    __tablename__ = "user"   # 表名

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(50))          # VARCHAR(50)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    age: Mapped[int | None]                                 # 可空列：类型写 int | None
    bio: Mapped[str | None] = mapped_column(Text)           # TEXT，不限长度文本
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

建表：

```python
Base.metadata.create_all(engine)   # 根据模型生成 CREATE TABLE（学习/原型阶段用，正式项目用 Alembic）
```

### 3.2 Mapped 与可空性的推导

`Mapped` 注解里的类型直接决定列的行为，这是 2.0 的精妙之处：

- `Mapped[str]` → `NOT NULL`；`Mapped[str | None]` → 可空（`nullable=True`）。
- `Mapped[int]` → 整数列；`Mapped[datetime]` → 日期时间列。SQLAlchemy 根据 Python 类型自动推断数据库类型，推断不满意时用 `mapped_column(String(50))` 显式指定。
- `Mapped[list["Post"]]`（配合 `relationship`）→ 这不是列，是一对多关系。

类型与 SQL 严格对应，写错类型时 mypy 会直接报错，很多低级 bug 在写代码阶段就被消灭了。

### 3.3 常用字段类型

```python
from decimal import Decimal
from sqlalchemy import String, Text, Boolean, Integer, BigInteger, Numeric, DateTime, JSON

class Product(Base):
    __tablename__ = "product"

    id: Mapped[int] = mapped_column(primary_key=True)
    sku: Mapped[str] = mapped_column(String(64), unique=True)   # 定长编码用 String
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text)        # 长文本用 Text
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))       # 金额必须 Numeric/Decimal，禁用 float！
    stock: Mapped[int] = mapped_column(default=0)
    views: Mapped[int] = mapped_column(BigInteger)               # 大数值用 BigInteger
    is_active: Mapped[bool] = mapped_column(default=True)
    attrs: Mapped[dict | None] = mapped_column(JSON)             # JSON 列（MySQL 5.7+/PG 均支持）
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

金额为什么禁用 `float`？浮点数用二进制近似表示十进制小数，`0.1 + 0.2 != 0.3`，对账时会出几分钱的差错。`Numeric(10, 2)` 映射到 Python 的 `Decimal`，是精确计算。

### 3.4 默认值：default vs server_default

两者都叫"默认值"，执行位置完全不同：

```python
from sqlalchemy import func

class Article(Base):
    __tablename__ = "article"

    id: Mapped[int] = mapped_column(primary_key=True)
    # default：由 Python 端生成。INSERT 语句里带上这个值；可传可调用对象（函数）
    status: Mapped[str] = mapped_column(String(20), default="draft")
    uuid: Mapped[str] = mapped_column(String(36), default=lambda: str(uuid4()))
    # server_default：由数据库生成。体现在建表语句的 DEFAULT 子句里
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    # onupdate：Python 端，每次 UPDATE 时自动刷新（常用于"更新时间"列）
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )
```

怎么选：

- 数据库原生的时间戳用 `server_default=func.now()`——以数据库服务器时钟为准，多实例部署时不会因为应用服务器时钟漂移而错乱。
- 需要应用层可见/可控的值（如 UUID、业务状态）用 `default`。
- 注意 `server_default` 的值在 INSERT 后 Python 对象上初始是 `None`，flush 之后 ORM 会自动回填。

---

## 四、增删改查

2.0 风格下，一切查询从 `select()` 开始，一切变更通过 `insert()` / `update()` / `delete()` 构造器或 ORM 对象操作完成。

```python
from sqlalchemy import create_engine, select, insert, update, delete, func
from sqlalchemy.orm import Session

engine = create_engine("mysql+pymysql://root:123456@localhost:3306/demo?charset=utf8mb4")
```

### 4.1 增（INSERT）

ORM 方式——创建对象，`add` 进 Session：

```python
with Session(engine) as session:
    user = User(name="张三", email="zhangsan@example.com", age=28)
    session.add(user)
    session.commit()
    print(user.id)  # commit 后 ORM 自动回填自增主键
```

批量插入用 `add_all` 或 2.0 的 `insert` 构造器（性能更高，适合大批量）：

```python
with Session(engine) as session:
    session.add_all([
        User(name="李四", email="lisi@example.com", age=25),
        User(name="王五", email="wangwu@example.com", age=32),
    ])
    session.commit()

# 大批量数据：绕过 ORM 对象，直接生成一条 INSERT ... VALUES (...), (...), ...
with Session(engine) as session:
    session.execute(
        insert(User),
        [{"name": f"user{i}", "email": f"u{i}@example.com"} for i in range(1000)],
    )
    session.commit()
```

### 4.2 查（SELECT）

```python
with Session(engine) as session:
    # 按主键查：最快，走身份映射缓存
    user = session.get(User, 1)

    # 查整行（返回 User 对象）
    stmt = select(User).where(User.age >= 18)
    users = session.execute(stmt).scalars().all()   # .scalars() 取出第一列（即 User 实体）

    # 只查部分列（返回命名元组 Row）
    stmt = select(User.name, User.age)
    for row in session.execute(stmt):
        print(row.name, row.age)

    # 查单个：一个结果返回对象，没有返回 None
    user = session.execute(
        select(User).where(User.email == "zhangsan@example.com")
    ).scalar_one_or_none()
    # scalar_one()：期望恰好一个，多了少了都抛异常
```

**where 过滤**的常用写法：

```python
from sqlalchemy import and_, or_, not_

select(User).where(User.age > 18)                                # 比较
select(User).where(User.name.in_(["张三", "李四"]))               # IN
select(User).where(User.name.like("张%"))                         # 模糊匹配（前缀匹配可走索引）
select(User).where(User.age.between(18, 30))                     # BETWEEN
select(User).where(User.bio.is_(None))                           # IS NULL（务必用 is_ 不是 ==）
select(User).where(User.age > 18, User.age < 60)                 # 多个条件默认 AND
select(User).where(or_(User.age < 18, User.name == "管理员"))     # OR
```

**排序与分页**：

```python
page, size = 2, 20
stmt = (
    select(User)
    .order_by(User.created_at.desc(), User.id)   # 多列排序；加 id 兜底保证顺序稳定
    .offset((page - 1) * size)
    .limit(size)
)
users = session.execute(stmt).scalars().all()

# 通常同时查总数用于分页器
total = session.execute(select(func.count()).select_from(User)).scalar_one()
```

深分页（`OFFSET 100000`）时数据库要扫过前 10 万行再丢弃，非常慢。大数据量翻页改用**游标分页**：`WHERE id > :last_seen_id ORDER BY id LIMIT 20`。

**聚合与分组**：

```python
stmt = (
    select(User.age, func.count().label("cnt"))
    .group_by(User.age)
    .having(func.count() > 1)
)
for age, cnt in session.execute(stmt):
    print(age, cnt)
```

### 4.3 改（UPDATE）

ORM 方式：改对象属性，commit 时自动生成 UPDATE（只更新脏字段）：

```python
with Session(engine) as session:
    user = session.get(User, 1)
    user.age = 29
    session.commit()
```

批量更新用 `update` 构造器（一条 SQL 更新多行，比逐条 ORM 快几个数量级）：

```python
with Session(engine) as session:
    session.execute(
        update(User).where(User.age < 18).values(is_active=False)
    )
    session.commit()
```

### 4.4 删（DELETE）

```python
# ORM 方式
with Session(engine) as session:
    user = session.get(User, 1)
    session.delete(user)
    session.commit()

# 批量删除
with Session(engine) as session:
    session.execute(delete(User).where(User.is_active.is_(False)))
    session.commit()
```

---

## 五、表关系

### 5.1 一对多与外键

一个用户有多篇文章。外键永远建在"多"的一方：

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))

    # 关系：不是数据库列，是 ORM 层的导航属性
    posts: Mapped[list["Post"]] = relationship(back_populates="author",
                                               cascade="all, delete-orphan")


class Post(Base):
    __tablename__ = "post"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    # 外键：真正的数据库列，指向 user.id
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id", ondelete="CASCADE"))

    author: Mapped["User"] = relationship(back_populates="posts")
```

要点：

- `ForeignKey("user.id")` 是**数据库层**的约束（建表时生成 FOREIGN KEY）；`relationship` 是 **ORM 层**的对象导航，两者互补。只写 relationship 不写 ForeignKey，SQLAlchemy 不知道按哪列关联。
- `back_populates` 让两侧关系保持同步：改了 `post.author`，`user.posts` 列表自动更新。
- `cascade="all, delete-orphan"`：删除用户时，他的文章也一并从 ORM 层删除；`ondelete="CASCADE"` 则是数据库层的级联删除。两者选一个即可，数据库层的更可靠（对绕过 ORM 的写入同样生效）。
- 注意 Mapped 类型：`Mapped[list["Post"]]` 是一对多（集合），`Mapped["User"]` 是多对一（单个对象）。SQLAlchemy 靠这个区分关系方向。

### 5.2 多对多

文章和标签：一篇文章多个标签，一个标签多篇文章。多对多需要**中间表（关联表）**，中间表只存两边外键，通常直接用 `Table` 定义，不必建 ORM 类：

```python
from sqlalchemy import Table, Column

post_tag = Table(
    "post_tag",
    Base.metadata,
    Column("post_id", ForeignKey("post.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", ForeignKey("tag.id", ondelete="CASCADE"), primary_key=True),
)


class Tag(Base):
    __tablename__ = "tag"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)

    posts: Mapped[list["Post"]] = relationship(secondary=post_tag, back_populates="tags")


# 在 Post 类里加：
class Post(Base):
    __tablename__ = "post"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id", ondelete="CASCADE"))
    author: Mapped["User"] = relationship(back_populates="posts")
    tags: Mapped[list["Tag"]] = relationship(secondary=post_tag, back_populates="posts")
```

用法：

```python
post.tags.append(tag)          # 建立关联：自动向中间表 INSERT
post.tags.remove(tag)          # 解除关联
```

如果中间表本身要带额外字段（比如"打标签的时间"），就不能用 `Table` 了，要把中间表建成独立的 ORM 模型，拆成两个一对多。

### 5.3 lazy vs eager 与 N+1 问题

`relationship` 默认 **lazy loading**：访问 `user.posts` 的那一刻才发一条 SELECT 去查文章。写起来方便，但藏着 ORM 最经典的性能陷阱——**N+1 问题**：

```python
users = session.execute(select(User)).scalars().all()   # 第 1 条 SQL：查出 100 个用户
for user in users:
    print(user.posts)   # 每个用户触发 1 条 SQL → 共 100 条！总计 101 条
```

1 条主查询 + N 条关联查询，1000 个用户就是 1001 条 SQL，数据库连接来回几百次，页面慢如蜗牛。

解决方案是 **eager loading（预加载）**，用 `selectinload` 或 `joinedload` 一次把关联数据查回来：

```python
from sqlalchemy.orm import selectinload, joinedload

# 方案一：selectinload（推荐默认使用）
# 两条 SQL：先查 user，再用 IN 一次性查所有 post
# SELECT * FROM user;
# SELECT * FROM post WHERE post.author_id IN (1, 2, 3, ...);
stmt = select(User).options(selectinload(User.posts))

# 方案二：joinedload
# 一条 SQL：LEFT OUTER JOIN 把两表拼起来
# SELECT user.*, post.* FROM user LEFT OUTER JOIN post ON ...
stmt = select(User).options(joinedload(User.posts))
```

怎么选：

- **一对多 / 多对多默认用 `selectinload`**。JOIN 会让主表行数随关联数膨胀（笛卡尔积放大），一个用户 10 篇文章，结果集就重复 10 行用户数据，白白传输。
- **多对一可用 `joinedload`**（如 `joinedload(Post.author)`）：关联目标是单行，不会放大结果集，一次 JOIN 比两次往返更高效。
- 嵌套预加载：`selectinload(User.posts).selectinload(Post.tags)`。
- 如果你确定某个字段在大多数场景都会用到，也可以在模型上把加载策略设为默认值：`relationship(lazy="selectin")`（即 `Mapped[...] = relationship(back_populates=..., lazy="selectin")`）。

---

## 六、在 FastAPI 中集成

### 6.1 同步范式：Session 依赖注入

核心思想：用一个**生成器依赖**，请求开始时创建 Session，请求结束（无论成败）保证关闭。

```python
from collections.abc import Generator

from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session, sessionmaker

engine = create_engine(
    "mysql+pymysql://root:123456@localhost:3306/demo?charset=utf8mb4",
    pool_size=5, max_overflow=10, pool_recycle=3600,
)
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)

app = FastAPI()


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db            # 请求处理期间使用
    finally:
        db.close()          # 请求结束，连接归还池子——finally 保证异常时也执行


@app.get("/users/{user_id}")
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = db.get(User, user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="用户不存在")
    return {"id": user.id, "name": user.name}


@app.get("/users")
def list_users(db: Session = Depends(get_db)):
    stmt = select(User).order_by(User.id).limit(100)
    return db.execute(stmt).scalars().all()
```

`expire_on_commit=False` 很重要：默认 commit 后所有对象属性过期，访问时会重新查数据库——在返回响应阶段突然发 SQL 既慢又容易踩坑（Session 已关闭时报错）。

### 6.2 异步范式：async engine + asyncpg / aiomysql

高并发 IO 密集型服务（Web 接口、爬虫网关）适合异步。要点：**每一层都得是异步的**——引擎、Session、驱动，一个环节同步就堵住整个事件循环。

安装异步驱动（二选一）：

```bash
pip install sqlalchemy[asyncio] asyncpg    # PostgreSQL
pip install sqlalchemy[asyncio] aiomysql   # MySQL
```

配置与依赖注入：

```python
from collections.abc import AsyncGenerator

from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

# 注意 URL 里的驱动名：postgresql+asyncpg / mysql+aiomysql
async_engine = create_async_engine(
    "postgresql+asyncpg://postgres:123456@localhost:5432/demo",
    pool_size=5, max_overflow=10,
)
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine, expire_on_commit=False
)

app = FastAPI()


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:   # async with 保证关闭
        yield session


@app.get("/users/{user_id}")
async def read_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)           # ORM 方法前加 await
    if user is None:
        raise HTTPException(status_code=404, detail="用户不存在")
    return {"id": user.id, "name": user.name}


@app.post("/users")
async def create_user(name: str, email: str, db: AsyncSession = Depends(get_db)):
    user = User(name=name, email=email)
    db.add(user)
    await db.commit()                            # commit 也要 await
    await db.refresh(user)                       # 重新加载，拿到自增 id 等数据库生成值
    return {"id": user.id}
```

异步模式的三个常见坑：

1. **lazy loading 会报错**。`user.posts` 这种隐式发 SQL 的访问在异步环境下无法 await，会抛 `MissingGreenlet`。所以异步代码里关联数据一律用 `selectinload` 预加载，或 `await session.run_sync(...)` 兜底。
2. **所有数据库调用都要 `await`**：`execute`、`commit`、`rollback`、`refresh`、`get`（`AsyncSession.get` 是协程）。
3. 模型定义完全复用——`Base`/`Mapped`/`relationship` 与同步代码一模一样，变的只是 Engine、Session 和调用方式。

---

## 七、Alembic 数据库迁移

`Base.metadata.create_all()` 只能建**新表**，不会修改已存在的表——加一列、改个类型它都无能为力。正式项目必须用迁移工具 **Alembic**（SQLAlchemy 官方出品）来管理表结构的版本演进。

### 7.1 初始化与配置

```bash
pip install alembic
alembic init alembic
```

生成 `alembic/` 目录和 `alembic.ini`。改两处配置：

```ini
# alembic.ini：数据库连接串（也可以从环境变量读）
sqlalchemy.url = mysql+pymysql://root:123456@localhost:3306/demo?charset=utf8mb4
```

```python
# alembic/env.py：让 Alembic 能找到你的模型元数据
from myapp.models import Base   # 导入你的 Base（确保所有模型都被 import 过）

target_metadata = Base.metadata   # 原来是 None，改成这个
```

注意：Alembic 靠 `Base.metadata` 收集表定义，**所有模型文件必须在 env.py 中被导入**，否则 Alembic 看不到这些表，autogenerate 会误以为要 DROP TABLE——这是新手最常见的事故来源。

### 7.2 autogenerate 的原理

```bash
# 修改模型后，生成迁移脚本
alembic revision --autogenerate -m "add user.age column"
```

autogenerate 的工作机制：**把数据库当前的真实结构（连库反射）与代码里 `Base.metadata` 描述的目标结构做 diff**，差异部分自动生成 `upgrade()` / `downgrade()` 代码。

```python
# alembic/versions/xxxx_add_user_age_column.py
def upgrade() -> None:
    op.add_column("user", sa.Column("age", sa.Integer(), nullable=True))

def downgrade() -> None:
    op.drop_column("user", "age")
```

**一定要人工 review 生成的脚本**。autogenerate 对"重命名列"这类操作无法区分（它会生成"删旧列 + 加新列"，数据会丢），对索引、约束、函数默认值等的识别也有盲区。

### 7.3 执行与回滚

```bash
alembic upgrade head      # 升级到最新版本
alembic upgrade +1        # 只升一个版本
alembic downgrade -1      # 回滚一个版本
alembic downgrade base    # 全部回滚（危险）
alembic current           # 查看数据库当前版本
alembic history           # 查看迁移历史
```

Alembic 在数据库里维护一张 `alembic_version` 表记录当前版本号，每次 upgrade/downgrade 按脚本链逐个执行。

### 7.4 常见迁移场景

```python
def upgrade() -> None:
    # 加列（生产环境大表加列注意锁表，MySQL 8.0 支持 INSTANT ADD COLUMN）
    op.add_column("user", sa.Column("nickname", sa.String(50), nullable=True))

    # 删列
    op.drop_column("user", "old_field")

    # 改类型
    op.alter_column("user", "name", type_=sa.String(100))

    # 建索引 / 唯一约束
    op.create_index("ix_user_email", "user", ["email"], unique=True)

    # 重命名列（正确姿势，别让 autogenerate 帮你删了重建）
    op.alter_column("user", "name", new_column_name="username")

    # 建表
    op.create_table(
        "login_log",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("user_id", sa.Integer, sa.ForeignKey("user.id")),
        sa.Column("created_at", sa.DateTime, server_default=sa.func.now()),
    )

    # 数据迁移：结构变更顺带改数据
    op.execute("UPDATE user SET nickname = name WHERE nickname IS NULL")
```

部署流程建议：代码发布前先在 CI 跑 `alembic upgrade head`（或让部署脚本执行）；`downgrade` 只作为开发期的后悔药，生产回滚优先用"向前迁移"（写新脚本修回去）。

---

## 八、事务管理

### 8.1 commit 与 rollback

```python
with Session(engine) as session:
    try:
        order = Order(amount=100, user_id=1)
        session.add(order)
        user = session.get(User, 1)
        user.balance -= 100          # 同一事务内的另一个修改
        session.commit()             # 两条 SQL 一起提交，要么都成功要么都失败
    except Exception:
        session.rollback()           # 任何一步出错：整体回滚
        raise
```

要点：`commit()` 之前，所有修改只存在于事务内，对其他连接不可见（READ COMMITTED 级别下）；`rollback()` 把事务内已 flush 的所有改动撤销。

### 8.2 begin 上下文管理器

`session.begin()` 把"事务边界 + 异常回滚"写成一个块，更简洁也更不容易忘：

```python
with Session(engine) as session:
    with session.begin():    # 进入：开启事务；正常退出：commit；抛异常：自动 rollback
        session.add(Order(amount=100, user_id=1))
        user = session.get(User, 1)
        user.balance -= 100
    # 这里事务已提交
```

Engine 层也有对应的 `with engine.begin() as conn:`，适合写 Core 风格 SQL。嵌套 `session.begin()` 会得到**保存点（SAVEPOINT）**，内层失败只回滚到保存点，外层可以继续——处理"批量导入时单行失败不拖垮整批"的场景很有用。

### 8.3 长事务的危害

事务不是开得越久越安全，恰恰相反：

- **占用连接不归还**：一个请求事务开着 10 秒，连接池就少一个连接 10 秒。高并发下池子瞬间被榨干，其他请求只能排队等超时。
- **锁持有时间拉长**：事务里 `UPDATE` 过的行会被加锁直到 commit，其他事务改这些行就得阻塞等待，严重时演变成大面积锁等待甚至死锁。
- **MVCC 版本无法清理**：REPEATABLE READ 下，长事务的快照会阻止数据库回收旧版本数据（MySQL undo log、PostgreSQL 的死元组），导致表膨胀、vacuum 失效。
- **主从延迟放大**：大事务在从库回放时会让复制延迟飙升。

纪律：**事务要短而快**。先在事务外把数据查好、算好，开事务只做"写"这一下；绝不在事务里做网络请求、文件 IO、sleep。

---

## 九、性能与排错

### 9.1 执行原生 SQL

ORM 覆盖不了所有场景（复杂报表、数据库特有语法），2.0 用 `text()` 执行原生 SQL，**务必用绑定参数，绝不拼字符串**（防 SQL 注入）：

```python
from sqlalchemy import text

with Session(engine) as session:
    rows = session.execute(
        text("SELECT * FROM user WHERE age > :min_age AND name LIKE :kw"),
        {"min_age": 18, "kw": "张%"},
    ).all()

    # 结果直接映射回 ORM 对象也行
    users = session.execute(
        select(User).from_statement(
            text("SELECT * FROM user WHERE age > :min_age")
        ).params(min_age=18)
    ).scalars().all()
```

### 9.2 EXPLAIN 分析慢查询

SQL 慢，先看执行计划再猜原因。`echo=True` 或日志里拿到 SQL 后，拿到客户端执行：

```sql
EXPLAIN SELECT * FROM user WHERE email LIKE '%@gmail.com';
```

重点看几列（MySQL 方言）：

- `type`：访问方式。`ALL`（全表扫描，危险信号）→ `index` → `range` → `ref` → `eq_ref` → `const`（最优）。
- `key`：实际用上了哪个索引；为 NULL 说明没走索引。
- `rows`：预估扫描行数，和表总行数接近说明索引没生效。

PostgreSQL 用 `EXPLAIN ANALYZE`（会真实执行并给出实际耗时），看 `Seq Scan`（全表扫描）、`Index Scan`、`rows` 估算偏差。

### 9.3 慢查询常见原因清单

1. **没走索引**：前导通配符 `LIKE '%x'`、对索引列做运算/函数（`WHERE DATE(created_at) = '2024-01-01'`）、类型隐式转换（字符串列传数字）。
2. **N+1 问题**：第五节，用 `selectinload` 解决。
3. **深分页**：`OFFSET` 过大，改游标分页。
4. **一次查太多**：`SELECT *` 查几百列、不分页全表拉取。只查需要的列。
5. **连接池打满**：现象是接口不查库时也慢、日志大量 `QueuePool timeout`。查是否有长事务/Session 泄漏。
6. **锁竞争**：多个事务改同一批行互相等待。用 `SHOW ENGINE INNODB STATUS`（MySQL）或 `pg_locks`（PG）排查。
7. **数据库服务器本身**：缓冲池太小、磁盘 IO 饱和、跨机房网络延迟。

### 9.4 会话泄漏

症状：跑着跑着连接池耗尽，报 `QueuePool limit ... overflow ... timeout exceeded`。根因几乎都是 **Session 没有关闭**：

```python
# 错误：session 创建后从不关闭，连接被永久占用
def get_user(user_id):
    session = SessionLocal()
    return session.get(User, user_id)

# 正确：上下文管理器或 try/finally 保证 close
def get_user(user_id):
    with SessionLocal() as session:
        return session.get(User, user_id)
```

Web 框架里统一用依赖注入（第六节）从根上杜绝。排查时可临时加 `pool_timeout=5` 让泄漏快速暴露，或开启 `echo_pool=True` 观察连接的借出/归还。

### 9.5 时区问题

经典坑：数据库里存的时间和你以为的差了 8 小时。

- **MySQL 的 `DATETIME` 不带时区**，存的是什么字面值读出来就是什么。`func.now()` 用的是数据库服务器时区——服务器是 UTC、应用是东八区，就差 8 小时。
- 推荐做法：**全链路统一 UTC**。应用里用 `datetime.now(timezone.utc)`，数据库服务器时区也设为 UTC，只在展示层转换为本地时区。
- 需要带时区的类型：SQLAlchemy 用 `DateTime(timezone=True)`，对应 PostgreSQL 的 `TIMESTAMPTZ`（真时区感知）；MySQL 对应 `TIMESTAMP`（内部转 UTC 存储，读时按会话时区转回）。

```python
from datetime import datetime, timezone
from sqlalchemy import DateTime

created_at: Mapped[datetime] = mapped_column(
    DateTime(timezone=True),
    default=lambda: datetime.now(timezone.utc),
)
```

### 9.6 MySQL 字符集：必须 utf8mb4

MySQL 的 `utf8` 是"假 UTF-8"——最多 3 字节，存不下 emoji（4 字节），插入 😂 会报 `Incorrect string value` 或存成问号。`utf8mb4` 才是真正的 UTF-8（最多 4 字节）。

三层都要设：

```sql
-- 建库
CREATE DATABASE demo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

```python
# 连接 URL
engine = create_engine("mysql+pymysql://root:123456@localhost:3306/demo?charset=utf8mb4")
```

历史遗留库用 `ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4;` 修复。PostgreSQL 默认 UTF-8，无此问题。

---

## 小结

- 数据库根基：B+ 树靠"矮胖多叉 + 叶子链表"把百万级查询压到几次磁盘 IO；事务靠 undo log 保证原子性、redo log 保证持久性、MVCC + 锁实现隔离性。脏读/不可重复读/幻读分别对应"读了未提交的"、"同行两次读不一样"、"范围查询行数变了"，隔离级别 READ COMMITTED → REPEATABLE READ → SERIALIZABLE 逐级设防。
- SQLAlchemy 2.0：`Engine` 管连接池（进程级、用完归还），`Session` 是工作单元（身份映射 + 延迟写），必须"一请求一 Session、用完即关"。
- 模型定义：`DeclarativeBase` + `Mapped[...]` + `mapped_column`，Python 类型注解即表结构；`default` 在 Python 端生效，`server_default` 由数据库生成。
- 查询：`select(...).where().order_by().limit().offset()` 一条链，聚合用 `func`；批量写用 `insert/update/delete` 构造器。
- 关系：外键在"多"的一方，`relationship` 做对象导航；N+1 问题用 `selectinload`（一对多）和 `joinedload`（多对一）预加载解决。
- FastAPI：同步用 `get_db` 生成器依赖；异步全链路换成 `create_async_engine` / `AsyncSession`，异步代码禁用 lazy loading。
- Alembic：`revision --autogenerate` 做的是"库 vs 代码"的结构 diff，生成脚本必须人工 review；`upgrade head` 前进、`downgrade` 回退。
- 事务：`session.begin()` 块自动 commit/rollback；事务要短，绝不在事务内做 IO。
- 排错：慢查询先 EXPLAIN，常见问题按"索引失效 → N+1 → 深分页 → 连接池 → 锁"的顺序排查；时区全链路 UTC；MySQL 一律 utf8mb4。

## 常见问题

**1. `default` 和 `server_default` 有什么区别？**

`default` 由 SQLAlchemy 在 Python 端生成值并拼进 INSERT 语句，可以是常量或函数（如 UUID）；`server_default` 写进建表语句的 DEFAULT 子句，由数据库在插入时生成（如 `NOW()`）。时间戳建议用 `server_default=func.now()` 以数据库时钟为准；需要应用层拿到值做逻辑（如主键 UUID）用 `default`。

**2. 为什么查询时报 "DetachedInstanceError"？**

Session 关闭之后，再去访问对象上尚未加载的属性（lazy loading 需要发 SQL，但 Session 已经没了）。三种解法：在 Session 生命周期内就把需要的关联数据用 `selectinload` 加载好；`sessionmaker(expire_on_commit=False)` 避免 commit 后属性过期；或者把对象数据在 Session 内复制成 DTO/字典再传出。

**3. 什么是 N+1 问题，怎么发现和解决？**

查 N 个主对象后逐个访问其关联属性，产生 1 条主查询 + N 条关联查询。开发时开 `echo=True`，看到一个接口刷屏大量相似 SQL 基本就是它。用 `selectinload`（集合关系）或 `joinedload`（单对象关系）预加载，把 N 条合并成 1-2 条。

**4. 同步和异步 SQLAlchemy 怎么选？**

看你的场景和团队。异步（`AsyncSession` + asyncpg）在 IO 密集、并发连接多的场景吞吐更高，但所有调用要 `await`、不能 lazy loading、调试链路更复杂；同步代码简单直白，配合 FastAPI 的线程池对大多数业务完全够用。不确定就先用同步，性能瓶颈真出现了再迁移异步——模型层不用改。

**5. 什么时候该放弃 ORM 直接写 SQL？**

复杂报表/多维聚合、需要数据库特有功能（窗口函数优化、递归 CTE、`ON DUPLICATE KEY UPDATE`）、大批量数据导入导出、以及被证明是性能瓶颈的热点查询。SQLAlchemy 里两者可以混用：`text()` 执行原生 SQL 是正式特性不是作弊。原则：先用 ORM 保证开发效率， profiling 定位到瓶颈后再针对性地下沉到原生 SQL。
