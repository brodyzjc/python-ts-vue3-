# Redis 实战：从数据结构与原理到缓存三大经典问题

> **本章导读**
>
> - Redis 为什么快：内存操作、单线程模型、IO 多路复用的底层机制
> - 用 Docker 安装 Redis，掌握 redis-cli 常用命令，理解 `KEYS *` 为什么危险
> - 五大数据结构（string / hash / list / set / zset）与它们对应的真实业务场景
> - 高级结构：bitmap 签到、HyperLogLog 统计 UV、Stream 消息流
> - 用 redis-py 操作 Redis：连接池、同步与 asyncio 用法、FastAPI 集成
> - Cache Aside 旁路缓存模式：读流程、写流程、一致性分析
> - 缓存穿透、击穿、雪崩三大经典问题的成因与解决方案代码
> - 分布式锁：SET NX EX、误删问题、Lua 脚本原子释放、看门狗机制
> - 典型应用：短信验证码、登录 token 与踢人下线、秒杀限流
> - 持久化（RDB / AOF）、过期策略与内存淘汰策略

Redis 是后端开发中出镜率最高的中间件之一：缓存、计数器、排行榜、分布式锁、限流、消息队列……几乎每个高并发系统里都有它的身影。但很多人对 Redis 的理解停留在"一个很快的 key-value 缓存"上，面试时被追问"为什么快""缓存和数据库怎么保证一致性""分布式锁怎么防误删"就答不上来。本章的目标是把这些问题从原理层面讲透，并给出可以直接运行的 Python 代码。

---

## 一、Redis 是什么、为什么快

### 1.1 Redis 是什么

Redis（**RE**mote **DI**ctionary **S**erver）是一个开源的、基于内存的数据结构服务器。把它理解成"一个远程的、跨进程共享的超级字典"最贴切：

- **key-value 存储**：所有数据以键值对形式存放，key 是字符串，value 可以是 string、hash、list、set、zset 等多种数据结构。
- **基于内存**：数据主要存放在内存中，读写不经过磁盘，这是它快的根本原因。
- **支持持久化**：可以把内存数据定期（RDB）或逐命令（AOF）刷到磁盘，重启后不丢数据。
- **单线程执行命令**：同一时间只执行一条命令，天然避免了并发竞争。
- **丰富的附加能力**：过期时间、发布订阅、Lua 脚本、事务、Stream 消息流等。

和 MySQL 这类关系型数据库相比，Redis 不擅长复杂查询（没有 join、没有 SQL），它擅长的是**根据 key 以 O(1) 或 O(log n) 的复杂度快速存取**。

### 1.2 快的原因一：纯内存操作

机械硬盘的随机读写延迟大约在毫秒级（0.1~10ms），SSD 在几十到几百微秒，而内存的访问延迟只有 **100 纳秒左右**——相差 4 到 5 个数量级。Redis 的核心数据全部放在内存里，一次 GET 就是一次哈希表查找加内存读取，单节点轻松支撑 10 万级 QPS。

需要澄清一个常见误解：说"Redis 是内存数据库所以快"只对了一半。把 MySQL 的数据全部 load 进内存（开足够大的 buffer pool），它的单点查询依然远慢于 Redis——因为还有下面两个原因。

### 1.3 快的原因二：单线程避免了锁竞争与上下文切换

Redis 处理客户端命令的主流程是**单线程**的（这里的"单线程"特指命令执行，6.0 之后网络 IO 可以多线程，后面细说）。单线程带来了两个巨大的好处：

1. **没有锁竞争**。多线程程序访问共享数据时必须加锁，加锁意味着开销：抢不到锁的线程要阻塞、唤醒，还可能出现死锁。Redis 同一时刻只有一条命令在执行，所有命令天然原子，`INCR` 不需要任何锁就是线程安全的。
2. **没有上下文切换**。线程切换需要保存/恢复寄存器、刷新 CPU 缓存（cache line 失效），线程越多切换越频繁，大量 CPU 时间浪费在"切换"而不是"干活"上。单线程把 CPU 全部用在执行命令上。

**那单线程为什么不慢？** 因为 Redis 的瓶颈根本不在 CPU。对于内存操作来说，CPU 早就"过剩"了——真正的瓶颈是**内存带宽和网络 IO**。多线程对内存操作几乎没有加速效果，反而引入竞争。官方也一直强调：Redis 的 CPU 极少成为瓶颈。如果将来某天 CPU 真的不够用了，水平方向加实例（集群分片）比垂直方向开多线程简单得多。

注意区分：这里的单线程是指**执行命令**的线程。Redis 后台还有其他线程做耗时的杂活，比如 `UNLINK` 异步删除大 key、AOF 刷盘、RDB 快照等，避免阻塞主线程。

### 1.4 快的原因三：IO 多路复用

这是理解 Redis 的关键，也是面试高频考点。我们把问题拆开看：

**单线程要同时服务上万个客户端连接，怎么做到不被某一个慢连接拖死？**

如果用最原始的"一个连接一个线程"模型，一万个连接就要一万个线程，光线程切换就把 CPU 耗尽了。如果用单线程 + 阻塞式 IO：

```text
read(conn_A)   ← 阻塞在这里等 A 的数据，B、C 的数据已经到了也只能干等
```

这就是"阻塞"的含义：调用 `read()` 时如果数据没准备好，当前线程就被挂起，直到数据到达。单线程 + 阻塞 IO 意味着任何一个没数据的连接都会卡住整个服务。

**IO 多路复用**（I/O Multiplexing）解决的就是这个问题：把"等待数据"这件事委托给操作系统。Linux 上的 `epoll`（Redis 默认使用）允许一个线程**同时监听成千上万个 socket**，然后告诉你"哪些 socket 现在有数据可读了"。主线程只去处理这些就绪的连接：

```text
epoll_wait() 返回就绪列表: [conn_B, conn_C]
→ 依次 read(conn_B)、处理、write 回包
→ 依次 read(conn_C)、处理、write 回包
→ 再次 epoll_wait()
```

整个流程是一个**事件循环（event loop）**：没有一条命令在执行时线程是"白等"的，要么在处理就绪的事件，要么在 `epoll_wait` 里睡眠（不占 CPU）。因为每条命令都是纯内存操作，耗时极短（微秒级），所以事件循环转得飞快，单线程就能撑起十万级 QPS。

一句话总结这套模型：**单线程负责执行（保证原子性、无锁），epoll 负责等数据（一个线程监听所有连接），两者配合让 CPU 一刻不停地处理真正的请求**。

补充：Redis 6.0 引入了**多线程 IO**——把 socket 的读取解析和回包写出让多个 IO 线程并行做（网络读写其实占了大量 CPU 时间），但**命令执行依然是单线程**的，原子性保证不变。默认未开启，需要配置 `io-threads` 生效。

### 1.5 与 Memcached 的一行区别（非对比章节，仅澄清）

Redis 不只是缓存：它有丰富的数据结构、持久化、Lua 脚本、过期策略，因此能胜任锁、队列、排行榜等 Memcached 做不了的事。本章后续内容全部围绕这些能力展开。

---

## 二、安装与上手

### 2.1 用 Docker 安装（推荐）

本地学习最省心的方式是 Docker，一条命令搞定，不污染宿主机：

```bash
# 拉取并启动一个 Redis 容器，命名为 my-redis，映射 6379 端口
docker run -d --name my-redis -p 6379:6379 redis

# 设置密码并指定持久化目录的完整写法（学习用可以省略）
docker run -d --name my-redis -p 6379:6379 \
  -v /opt/redis/data:/data \
  redis redis-server --requirepass "你的密码" --appendonly yes
```

进入容器使用命令行客户端：

```bash
docker exec -it my-redis redis-cli
# 如果设置了密码
docker exec -it my-redis redis-cli -a "你的密码"
```

看到 `127.0.0.1:6379>` 提示符就说明安装成功。Windows 用户也可以用 WSL2 或直接装 Memurai（Redis 的 Windows 移植版），但 Docker 方案各平台行为一致，推荐统一使用。

### 2.2 redis-cli 常用命令速览

```bash
# 连通性测试，回复 PONG 即正常
127.0.0.1:6379> PING
PONG

# 最基本的读写
127.0.0.1:6379> SET name "kimi"
OK
127.0.0.1:6379> GET name
"kimi"

# 设置时附带过期时间（秒），10 秒后自动删除
127.0.0.1:6379> SET code "886543" EX 10
OK
# 查看剩余存活时间，-2 表示已不存在
127.0.0.1:6379> TTL code
(integer) 7

# 判断 key 是否存在
127.0.0.1:6379> EXISTS name
(integer) 1

# 删除 key
127.0.0.1:6379> DEL name
(integer) 1

# 查看 key 的数据类型
127.0.0.1:6379> SET age 18
OK
127.0.0.1:6379> TYPE age
string

# 查看服务器信息（内存、连接数、持久化状态等，排查问题必用）
127.0.0.1:6379> INFO memory
```

### 2.3 `KEYS *` 为什么危险

排查问题时很多人会随手敲：

```bash
127.0.0.1:6379> KEYS *
```

它能列出所有 key，但**在生产环境这是一条可能引发事故的命令**。原因回到第一节的原理：Redis 执行命令是单线程的，`KEYS` 需要对全量 key 做一次**全表扫描**，时间复杂度 O(N)。生产库里有几千万个 key 时，这条命令会独占主线程几秒钟甚至更久——期间所有其他请求全部排队等待，从外部看就是"Redis 卡死了"，缓存超时大面积打穿到数据库，可能引发雪崩。

正确的替代方案是 `SCAN`：游标式增量遍历，每次只扫一小批，不阻塞主线程：

```bash
# 从游标 0 开始，匹配 user: 前缀，每次最多扫 100 个（COUNT 是提示值非精确值）
127.0.0.1:6379> SCAN 0 MATCH user:* COUNT 100
1) "173"              # 下次迭代要传入的游标，返回 0 表示遍历结束
2) 1) "user:1001"
   2) "user:1002"
```

注意 `SCAN` 是分页的：返回的第一个元素是**下一次的游标**，需要循环调用直到游标回到 0 才算遍历完成。同理，hash 用 `HSCAN`、set 用 `SSCAN`、zset 用 `ZSCAN`。另外几个同样有阻塞风险的命令也要警惕：`SMEMBERS`（大集合全量返回）、`HGETALL`（大 hash）、`FLUSHALL`（清空整个库）。

---

## 三、五大数据结构与真实业务场景

选择数据结构的标准是：**你的业务操作能不能映射成这个结构的一条 O(1)/O(log n) 命令**。映射得越直接，代码越简洁、性能越好。以下 Python 示例统一使用 `redis-py`（安装：`pip install redis`），连接细节在第五节展开，这里先用最简单的直连：

```python
import redis

r = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)
```

`decode_responses=True` 让返回值从 bytes 自动解码为 str，业务代码里几乎必加。

### 3.1 string：缓存、计数器、分布式锁

string 是最基础的类型，一个 key 对应一个字符串值（最大 512MB，可以存 JSON、序列化对象甚至图片二进制）。

**场景一：缓存对象**。把数据库查出来的对象序列化成 JSON 存进去：

```python
import json

user = {"id": 1001, "name": "张三", "vip_level": 3}
# 缓存 1 小时
r.set("user:1001", json.dumps(user, ensure_ascii=False), ex=3600)

# 读取
raw = r.get("user:1001")
if raw:
    user = json.loads(raw)
```

**场景二：计数器**。`INCR` 是原子操作——这是单线程模型送你的礼物，不需要任何锁：

```python
# 文章阅读量
r.incr("article:42:views")          # +1，key 不存在时从 0 开始
r.incrby("article:42:views", 10)    # +10

# 库存扣减（下单前）
remaining = r.decr("sku:99:stock")
if remaining < 0:
    r.incr("sku:99:stock")          # 库存不足，加回去
    print("已售罄")
```

因为 `DECR` 是原子的，一千个并发请求同时扣库存也不会出现超扣到负数还浑然不觉的情况——每个请求拿到的都是唯一的递减结果。

**场景三：分布式锁**。`SET key value NX EX seconds` 一条命令完成"不存在才设置 + 设置过期时间"，是分布式锁的基础，第八节详细展开。

```python
# NX：key 不存在才设置成功；EX：10 秒后自动过期（防死锁）
ok = r.set("lock:order:88", "持有者标识", nx=True, ex=10)
if ok:
    try:
        ...  # 临界区业务逻辑
    finally:
        r.delete("lock:order:88")   # 简陋版释放，正确姿势见第八节
```

### 3.2 hash：对象存储

存一个对象的多个字段时，把对象 JSON 塞进 string 有两个缺点：改一个字段要读出整个 JSON、反序列化、改完再整体写回；并发改不同字段还会互相覆盖。hash 让对象的不同字段可以**独立读写**：

```python
# 存对象（HSET 支持一次设置多个字段）
r.hset("user:1001", mapping={"name": "张三", "age": 30, "city": "北京"})

# 只改一个字段，不影响其他字段
r.hset("user:1001", "city", "上海")

# 读单个字段 / 读全部字段
r.hget("user:1001", "name")        # '张三'
r.hgetall("user:1001")             # {'name': '张三', 'age': '30', 'city': '上海'}

# 字段级计数：购物车商品数量 +1
r.hincrby("cart:1001", "sku:99", 1)
```

典型场景：用户信息、商品详情、购物车（key 是 `cart:用户id`，field 是商品 id，value 是数量）。注意 hash 的 field 不能单独设置过期时间，过期只能作用在整个 key 上，这是它和 string 方案的重要取舍。

### 3.3 list：简单队列

list 是双向链表，支持两端推拉，天然是个队列：

```python
# 生产者：从左端推入任务
r.lpush("queue:email", json.dumps({"to": "a@b.com", "subject": "欢迎注册"}))

# 消费者：从右端阻塞式取出（BRPOP 的 timeout=0 表示永久等待）
# 没有任务时连接挂起等待，有任务立即返回，不需要轮询
while True:
    key, payload = r.brpop("queue:email", timeout=5)
    if key is None:
        continue  # 超时，继续等
    task = json.loads(payload)
    send_email(task["to"], task["subject"])
```

`LPUSH` + `BRPOP` 构成一个简单的生产者-消费者队列，足够支撑低峰期的异步任务（发邮件、写日志）。但它没有"消费确认"机制——`BRPOP` 弹出后任务就没了，消费者宕机任务就丢了。要求可靠投递的场景应该用 Stream（见 4.3）或专业消息队列。

另一个高频用法是**固定长度的最新列表**，比如"最新评论前 50 条"：

```python
r.lpush("article:42:latest_comments", comment_json)
r.ltrim("article:42:latest_comments", 0, 49)   # 只保留前 50 个
r.lrange("article:42:latest_comments", 0, -1)  # 读取全部
```

### 3.4 set：去重与共同关注

set 是无序不重复集合，去重是它的本能：

```python
# 场景一：抽奖去重 / 活动参与名单（一个人只能参与一次）
r.sadd("activity:7:joined", "user:1001")
added = r.sadd("activity:7:joined", "user:1001")  # 返回 0，重复参与
if added == 0:
    print("您已经参与过了")

# 场景二：标签系统
r.sadd("article:42:tags", "python", "redis", "后端")

# 场景三：共同关注（集合运算）
r.sadd("user:1001:follows", "user:2", "user:3", "user:5")
r.sadd("user:1002:follows", "user:3", "user:5", "user:8")
r.sinter("user:1001:follows", "user:1002:follows")   # {'user:3', 'user:5'}
r.sdiff("user:1001:follows", "user:1002:follows")    # {'user:2'}：我关注而对方没关注的
```

`SINTER`（交集）、`SUNION`（并集）、`SDIFF`（差集）让"共同好友""可能认识的人"这类社交功能一行命令就能实现。注意：对大集合做交集运算是 O(N) 级别的阻塞操作，生产环境对大 key 慎用，或者用 `SINTERSTORE` 把结果存到新 key。

### 3.5 zset：排行榜与延迟队列

zset（sorted set）= set + 每个成员带一个分数（score），内部按分数排序，底层是跳表（skiplist）+ 哈希表，范围查询 O(log n + M)。凡是"带分数的排序"需求都该想到它。

**场景一：排行榜**。

```python
# 用户积分变动（ZINCRBY 原子加分）
r.zincrby("rank:game:7", 35, "user:1001")
r.zincrby("rank:game:7", 20, "user:1002")

# 查询前 10 名（WITHSCORES 把分数一起带回来）
top10 = r.zrevrange("rank:game:7", 0, 9, withscores=True)
for name, score in top10:
    print(f"{name}: {score:.0f} 分")

# 查某个用户的名次（0 基，第一名返回 0）
rank = r.zrevrank("rank:game:7", "user:1001")
score = r.zscore("rank:game:7", "user:1001")
```

实时更新、实时查询，不需要像数据库那样 `ORDER BY ... LIMIT` 全表排序，这就是"选对数据结构"的威力。

**场景二：延迟队列**。把"执行时间戳"作为 score，消费者轮询取出到期的任务：

```python
import time

# 下单后 30 分钟不支付就自动关单：score = 应执行的时间戳
deadline = time.time() + 30 * 60
r.zadd("delay:close_order", {"order:886": deadline})

# 消费者：循环取出"现在已经到期"的任务
while True:
    now = time.time()
    # ZRANGEBYSCORE 取 score ≤ now 的任务
    due = r.zrangebyscore("delay:close_order", "-inf", now, start=0, num=10)
    for member in due:
        # ZREM 成功说明抢到了这个任务（多消费者防重）
        if r.zrem("delay:close_order", member) == 1:
            close_order(member)
    time.sleep(1)
```

用 `ZREM` 的返回值做"抢占"是关键细节：多个消费者可能同时 `ZRANGEBYSCORE` 到同一个任务，但只有 `ZREM` 返回 1 的那个真正执行，避免重复处理。

---

## 四、高级结构简介

### 4.1 bitmap：一亿用户的签到系统

bitmap 不是独立的数据类型，而是 string 之上的位操作。一个 key 的 value 可以被当成 bit 数组，`SETBIT key offset 1` 把第 offset 位设为 1。

签到场景的经典设计：key 用 `sign:用户id:年月`，offset 用"日期-1"：

```python
# user:1001 在 2024 年 6 月 15 日签到
r.setbit("sign:1001:202406", 14, 1)

# 检查 6 月 15 日是否签到
signed = r.getbit("sign:1001:202406", 14)

# 统计当月签到天数（BITCOUNT）
days = r.bitcount("sign:1001:202406")
```

算一笔账就知道为什么用 bitmap：一个月最多 31 天，每个用户每月的签到记录只占 **31 bit ≈ 4 字节**。一亿用户存一个月的签到只需要约 400MB；如果换成 set（每个签到记录存一个成员），同样的数据要几十 GB。日活统计同理：key 是 `uv:20240615`，offset 是用户 id，上线就 `SETBIT uv:20240615 用户id 1`，`BITCOUNT` 就是当天 UV。

### 4.2 HyperLogLog：海量 UV 统计

bitmap 统计 UV 的内存占用和用户 id 的最大值成正比，如果用户 id 稀疏（比如用 UUID），bitmap 就不划算了。HyperLogLog（HLL）用概率算法做**基数统计**（估算不重复元素个数），**固定只占 12KB 内存，标准误差约 0.81%**：

```python
# 每来一个访问者就 PFADD
r.pfadd("uv:20240615", "user:1001")
r.pfadd("uv:20240615", "user:1002")
r.pfadd("uv:20240615", "user:1001")   # 重复添加不影响结果

# 估算当天 UV
uv = r.pfcount("uv:20240615")

# 可以合并多天的统计：一周 UV（注意是估算一周去重总数）
r.pfmerge("uv:week24", "uv:20240615", "uv:20240616")
```

取舍很明确：**用极小的固定内存换不到 1% 的误差**。页面访问量、搜索词去重这类"量级对就行"的场景非常合适；但它是估算值且只能增不能删，需要精确计数（如财务数据）不能用。

### 4.3 Stream：Redis 的轻量级消息队列

Redis 5.0 引入的 Stream 是专门为消息队列设计的结构，补上了 list 队列的短板：消息持久化、消费确认（ACK）、消费者组（Consumer Group）。核心概念：

- 消息有唯一 ID（`毫秒时间戳-序号`，如 `1686900000000-0`），追加后不可变。
- **消费者组**：一条消息只被组内一个消费者处理，组间互相独立（都收到）。
- 消费者处理完后发 `XACK` 确认；没确认的消息留在 **pending 列表**，宕机恢复后可以用 `XCLAIM` 把别人没确认的消息抢过来重试——这就实现了"至少一次"投递。

```python
# 生产者：追加一条消息，MAXLEN ~ 10000 表示近似修剪到 1 万条防无限增长
r.xadd("stream:orders", {"order_id": "886", "amount": "99.5"}, maxlen=10000, approximate=True)

# 创建消费者组（mkstream=True 表示流不存在则创建）
r.xgroup_create("stream:orders", "group:pay", id="0", mkstream=True)

# 消费者：以 "consumer-1" 身份从 group:pay 组读取新消息
while True:
    # ">" 表示只读"从未分配给任何消费者"的新消息
    entries = r.xreadgroup("group:pay", "consumer-1",
                           {"stream:orders": ">"}, count=10, block=5000)
    for stream, messages in entries:
        for msg_id, fields in messages:
            try:
                process_order(fields)
                r.xack("stream:orders", "group:pay", msg_id)  # 确认
            except Exception:
                pass  # 不 ACK，消息留在 pending 列表，稍后可 XAUTOCLAIM 重试
```

Stream 让一个 Redis 实例同时胜任缓存和轻量 MQ。但要说清楚边界：它没有专业 MQ（如 RocketMQ/Kafka）的堆积能力、重试策略和监控体系，消息量大或可靠性要求高的核心链路仍应使用专业消息队列。

---

## 五、Python 操作 Redis

### 5.1 连接池：不要每次请求都新建连接

TCP 连接的建立（三次握手）是有成本的，如果每个业务请求都 `Redis()` 新建连接、用完关闭，高并发下会浪费大量时间在握手上。**连接池**预先维护一批长连接，用的时候借、用完还，是所有数据库类客户端的标准做法。

`redis-py` 的好消息是：`redis.Redis` 实例**内部自带连接池**，默认 `max_connections=2**31`（几乎不限）。所以正确的用法很简单——**全局只创建一个 `Redis` 实例，到处复用它**：

```python
# redis_client.py：项目里唯一的 Redis 客户端
import redis

pool = redis.ConnectionPool(
    host="localhost",
    port=6379,
    db=0,
    password=None,
    decode_responses=True,
    max_connections=50,      # 池上限：并发峰值超过它时新请求会阻塞等待
    socket_timeout=5,        # 读写超时，防止连接被慢命令拖死
    socket_connect_timeout=2,
    retry_on_timeout=True,
)

r = redis.Redis(connection_pool=pool)
```

`redis.Redis` 是线程安全的：每个方法调用时自动从池里取连接、用完归还，多线程共享一个实例没有问题。**反模式**是在每个函数里 `redis.Redis(...)`——那会绕过池的好处（实际上 `Redis()` 不传 pool 也会复用模块级默认池机制，但显式创建 ConnectionPool 才能控制参数）。

### 5.2 同步用法与 pipeline

常规操作前面已经见过。这里补一个性能要点：**pipeline（管道）**。如果你有 1000 个 key 要写，循环调用 `r.set()` 意味着 1000 次"请求→等响应"的网络往返（RTT）；pipeline 把 1000 条命令一次性发出去、一次性收所有响应，把 RTT 从 1000 次压缩到 1 次：

```python
# 慢：1000 次网络往返
for i in range(1000):
    r.set(f"key:{i}", i)

# 快：1 次网络往返
pipe = r.pipeline(transaction=False)
for i in range(1000):
    pipe.set(f"key:{i}", i)
pipe.execute()
```

`transaction=False` 表示只要"打包发送"不要事务语义（Redis 事务本身很弱，见常见问题），纯为提速时建议关掉。注意 pipeline 会把所有响应攒在内存里，批量太大（如百万级）应分批。

### 5.3 asyncio 用法

FastAPI、Sanic 这类异步框架里，如果用同步版 `redis.Redis`，每次 Redis 调用都会**阻塞事件循环**——整个应用的并发能力瞬间崩塌。异步环境必须用 `redis.asyncio`：

```python
import asyncio
from redis import asyncio as aioredis   # redis-py 4.2+ 内置 asyncio 支持

async def main():
    r = aioredis.Redis(host="localhost", port=6379, decode_responses=True)
    await r.set("greeting", "hello asyncio")
    print(await r.get("greeting"))

    # 多个独立操作用 gather 并发发出，总耗时≈最慢的一个
    await asyncio.gather(
        r.set("a", 1),
        r.set("b", 2),
        r.set("c", 3),
    )
    await r.aclose()   # 关闭连接池

asyncio.run(main())
```

### 5.4 在 FastAPI 中集成

原则就两条：**整个应用共享一个客户端**（随应用生命周期创建/销毁），**通过依赖注入提供给路由**：

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, Request
from redis import asyncio as aioredis


@asynccontextmanager
async def lifespan(app: FastAPI):
    # 应用启动时创建一次连接池
    app.state.redis = aioredis.Redis(
        host="localhost", port=6379, decode_responses=True, max_connections=50
    )
    yield
    # 应用关闭时释放
    await app.state.redis.aclose()

app = FastAPI(lifespan=lifespan)


async def get_redis(request: Request) -> aioredis.Redis:
    return request.app.state.redis


@app.get("/users/{user_id}")
async def get_user(user_id: int, r: aioredis.Redis = Depends(get_redis)):
    cached = await r.get(f"user:{user_id}")
    if cached:
        return {"source": "cache", "data": cached}
    # ... 缓存未命中，回源数据库（见第六节）
    return {"source": "db"}
```

`lifespan` 是 FastAPI 推荐的资源管理方式：连接池在启动时建好、关闭时优雅释放，避免了"第一个请求到来才建连"的冷启动延迟。

---

## 六、缓存实战：Cache Aside 旁路缓存模式

### 6.1 为什么需要缓存模式

把 Redis 当缓存用，看起来就是"先查缓存，没有再查库"。但魔鬼在细节里：**写数据的时候，缓存怎么办？** 先更新数据库还是先删缓存？并发下会不会出现缓存和数据库长期不一致？这些问题的标准答案就是 Cache Aside（旁路缓存）模式——它是互联网应用使用最广泛的缓存模式。

### 6.2 读流程（cache miss 回源）

```python
import json

def get_user(user_id: int) -> dict | None:
    key = f"user:{user_id}"
    cached = r.get(key)
    if cached is not None:
        return json.loads(cached)              # 1. 缓存命中，直接返回

    user = db_query_user(user_id)              # 2. 未命中，查数据库
    if user is None:
        return None
    # 3. 回填缓存，设置过期时间（兜底防脏数据永生）
    r.set(key, json.dumps(user), ex=3600)
    return user
```

要点：

- **必须设置过期时间**。过期时间是最后的保险丝：即使写流程出了纰漏导致缓存变脏，脏数据最多存活一个 TTL。
- 回填发生在"读"的路径上，所以叫 lazy loading（懒加载），只缓存真正被访问的数据。

### 6.3 写流程：为什么是"更新数据库 + 删除缓存"

写流程有两个经典的错误方案，理解了它们错在哪，就理解了正确方案为什么正确。

**错误方案一：更新数据库的同时"更新缓存"。** 两个线程并发写同一个用户：

```text
线程 A：更新 DB（name=张三）
线程 B：更新 DB（name=李四）
线程 B：更新缓存（name=李四）
线程 A：更新缓存（name=张三）   ← 网络延迟让 A 的缓存更新后到
结果：DB 里是"李四"，缓存里是"张三"，脏数据一直活到过期
```

并发下"更新缓存"的执行顺序无法保证，缓存极易变成旧值。而**删除**缓存没有这个问题——删掉之后下次读会回源拿到最新值。

**错误方案二：先删缓存，再更新数据库。**

```text
线程 A（写）：删除缓存 user:1001
线程 B（读）：缓存未命中 → 查 DB 拿到旧值 → 回填缓存（旧值）
线程 A（写）：更新 DB 为新值
结果：DB 是新值，缓存是旧值，脏数据又活到过期
```

A 删缓存和 B 回填之间没有任何同步机制，读的回填很容易插进来。

**正确方案：先更新数据库，再删除缓存。**

```python
def update_user(user_id: int, data: dict):
    db_update_user(user_id, data)   # 1. 先更新数据库
    r.delete(f"user:{user_id}")     # 2. 再删除缓存（不是更新！）
```

它也不是数学意义上的绝对一致，存在一个极窄的窗口：缓存恰好过期的瞬间，一个读请求回源拿到旧值并回填，而此刻另一个写请求刚更新完 DB、还没删缓存——旧值被回填后又被删掉的前提是删除在回填之后。这个窗口要求"读回填"慢于"写+删"，而实际中写操作（带事务、索引维护）通常远慢于读，所以概率极低。工程上还常用**延迟双删**进一步压缩风险：

```python
def update_user_safe(user_id: int, data: dict):
    db_update_user(user_id, data)
    r.delete(f"user:{user_id}")
    time.sleep(0.5)                 # 等业务读流程走完
    r.delete(f"user:{user_id}")     # 第二次删除，清掉窗口期回填的脏数据
```

### 6.4 一致性问题的终极取舍

只要数据同时存在于两个系统（DB 和缓存），又没有分布式事务把它们绑在一起，**强一致性在理论上就不可达**，工程上只能追求**最终一致性**。可选的加固手段按复杂度递增：

1. **过期时间兜底**（必备，成本为零）。
2. **延迟双删**（如上，压缩不一致窗口）。
3. **binlog 订阅**：用 Canal 之类的组件监听 MySQL binlog，数据库一旦变更就异步删缓存/推消息。删除动作从业务代码里剥离，不怕业务忘了删，也不怕删失败——Canal 侧可以重试。
4. **重试机制**：删除缓存失败时把 key 投递到消息队列，异步重试删除，避免"DB 改了缓存没删"。

对一致性要求极高的场景（如余额），更诚实的做法是**不缓存，直读数据库**——缓存是为"读多写少、可容忍毫秒~秒级延迟"的数据设计的。

---

## 七、三大缓存经典问题

### 7.1 缓存穿透：查询一个根本不存在的数据

**成因**：恶意请求（或爬虫 bug）反复查询数据库里不存在的 id（如负数、超大 id）。因为数据不存在，缓存永远写不进去，每个请求都打穿到数据库。流量大时数据库会被拖垮——这本质是一种攻击面。

**方案一：缓存空值**。数据库查不到也把"空"缓存起来，下次同样的请求直接被缓存挡掉：

```python
def get_product(product_id: int) -> dict | None:
    key = f"product:{product_id}"
    cached = r.get(key)
    if cached is not None:
        return None if cached == "NULL" else json.loads(cached)

    product = db_query_product(product_id)
    if product is None:
        # 空值也要缓存，但 TTL 设短一些（防真实数据上线后长时间被"空"挡住）
        r.set(key, "NULL", ex=60)
        return None
    r.set(key, json.dumps(product), ex=3600)
    return product
```

缺点是会缓存大量无意义的空 key（攻击者换着 id 打，每个 id 一个空 key），所以 TTL 一定要短。

**方案二：布隆过滤器（Bloom Filter）**。在缓存之前再加一道概率型屏障，一句话判断"这个 id 可能存在 / 一定不存在"：

```python
# 使用 RedisBloom 插件提供的布隆过滤器
# 初始化：把所有真实存在的商品 id 加入过滤器（数据变更时同步维护）
# redis-cli: BF.ADD product:bf 1001

def get_product_bf(product_id: int) -> dict | None:
    # 布隆过滤器说"一定不存在"，直接返回，连缓存都不用查
    if not r.execute_command("BF.EXISTS", "product:bf", product_id):
        return None
    return get_product(product_id)   # 走正常缓存流程
```

布隆过滤器的特性要记牢：**说"不存在"就一定不存在（零漏判），说"存在"可能不存在（有误判率）**——误判的后果只是多查一次缓存/数据库，无害；而且它内存占用极小、不支持删除（误判率与容量在初始化时权衡）。两个方案可以叠加：布隆过滤器挡住绝大多数非法 id，缓存空值兜住漏网的。

### 7.2 缓存击穿：热点 key 在过期瞬间被打爆

**成因**：某个**超级热点 key**（如秒杀商品、头条新闻）恰好在流量洪峰时过期。这一瞬间，几万个并发请求同时缓存未命中，同时回源数据库、同时回填——数据库瞬间承受同一条 SQL 的几万次并发，可能直接宕机。和穿透的区别：穿透查的是"不存在的数据"，击穿是"存在但缓存恰好失效的热点数据"。

**方案一：互斥锁（mutex）**。只允许第一个发现未命中的请求回源，其他请求等它回填完再读缓存：

```python
import time
import json

def get_hot_product(product_id: int) -> dict | None:
    key = f"product:{product_id}"
    cached = r.get(key)
    if cached is not None:
        return json.loads(cached)

    lock_key = f"lock:rebuild:{product_id}"
    # 尝试获取重建锁：NX 保证同一时刻只有一个线程拿到，EX 防持锁者宕机死锁
    if r.set(lock_key, "1", nx=True, ex=10):
        try:
            # 双重检查：拿锁期间可能已有别人重建过
            cached = r.get(key)
            if cached is not None:
                return json.loads(cached)
            product = db_query_product(product_id)   # 全系统只有一个请求在查库
            if product:
                r.set(key, json.dumps(product), ex=3600)
            return product
        finally:
            r.delete(lock_key)
    else:
        # 没抢到锁：睡一小会儿重试，等持锁者回填缓存
        time.sleep(0.05)
        return get_hot_product(product_id)
```

几万个并发里只有 1 个打到数据库，其余自旋等待后从缓存读。注意三个细节：锁必须带过期时间（防死锁）、拿到锁后要**双重检查**缓存、重试要有间隔。

**方案二：逻辑过期**。不给 key 设 TTL，把过期时间存进 value 里；发现"逻辑过期"后不阻塞等待，而是**异步**派一个线程去重建，其他请求先返回旧数据：

```python
def get_hot_product_logical(product_id: int) -> dict | None:
    key = f"product:{product_id}"
    cached = r.get(key)
    if cached is None:
        return None                       # 从未缓存过，说明是冷数据，不回源

    wrapper = json.loads(cached)
    if wrapper["expire_at"] > time.time():
        return wrapper["data"]            # 逻辑未过期，直接返回

    # 逻辑已过期：抢到锁的线程去后台重建，没抢到的先返回旧数据
    lock_key = f"lock:rebuild:{product_id}"
    if r.set(lock_key, "1", nx=True, ex=10):
        threading.Thread(target=rebuild, args=(product_id, lock_key),
                         daemon=True).start()
    return wrapper["data"]                # 返回旧值，可用性优先

def rebuild(product_id: int, lock_key: str):
    try:
        product = db_query_product(product_id)
        wrapper = {"data": product, "expire_at": time.time() + 3600}
        r.set(f"product:{product_id}", json.dumps(wrapper))  # 注意：不设 TTL
    finally:
        r.delete(lock_key)
```

取舍很清晰：互斥锁保证数据新鲜但让请求排队（可能超时），逻辑过期保证高可用但短时间内返回旧数据。秒杀类场景常用逻辑过期，对一致性敏感的场景用互斥锁。

### 7.3 缓存雪崩：大量 key 同时过期

**成因**：两种典型情况——其一，系统启动/活动开始前批量预热缓存，所有 key 设了**相同的 TTL**（比如统一 2 小时），到期时刻集体失效，流量洪峰全部打到数据库；其二，Redis 实例宕机，所有缓存瞬间消失。

**解决方案**：

1. **随机过期时间**（针对批量同时过期）：在基础 TTL 上加随机抖动，让过期时间自然散开：

```python
import random

def cache_product(product: dict):
    # 基础 1 小时 + 随机 0~300 秒，过期时刻从"一个点"摊平成"一个区间"
    ttl = 3600 + random.randint(0, 300)
    r.set(f"product:{product['id']}", json.dumps(product), ex=ttl)
```

2. **服务降级与熔断**（针对 Redis 宕机）：缓存不可用时，用限流器（如令牌桶）放少量请求进数据库，其余直接返回降级数据或友好错误，宁可部分不可用也不能让数据库被打死。
3. **Redis 高可用**：主从 + 哨兵自动故障转移，或直接用集群模式，从架构上消除单点宕机。
4. **多级缓存**：本地缓存（如进程内 dict / Caffeine 式结构）挡一层，Redis 全挂时还能扛住一部分热点流量。

---

## 八、分布式锁

### 8.1 为什么单机锁不够

一个服务部署了 4 个实例（4 个进程，可能在 4 台机器上），它们都要操作同一笔订单。`threading.Lock` 只能锁住**当前进程内**的线程，对其他进程完全无效。分布式锁需要一个**所有实例都能访问的第三方存储**来充当"锁仲裁者"，Redis 因为快且命令原子，成为最常用的选择。

### 8.2 加锁：SET NX EX 必须是一条命令

加锁的两个要求：**互斥**（只有一个人能拿到）和**防死锁**（持锁者宕机后锁能自动释放）。错误的写法是分两步：

```python
# 错误示范：SETNX 成功之后、EXPIRE 之前进程宕机，锁永远不释放 → 死锁
r.setnx("lock:order:88", "1")
r.expire("lock:order:88", 10)
```

正确做法是用一条命令原子完成，`SET key value NX EX seconds`：

```python
# NX：key 不存在才设置（互斥）；EX：10 秒自动过期（防死锁）
ok = r.set("lock:order:88", "唯一标识", nx=True, ex=10)
if not ok:
    raise RuntimeError("获取锁失败，稍后重试")
```

原子性由单线程模型保证：这条命令要么整体执行成功，要么整体不执行，不存在"设置了值但没来得及设过期"的中间态。

### 8.3 误删问题：A 的锁被 B 删了

设想这个时序：

```text
线程 A：加锁成功（TTL 10s），开始处理业务
        业务执行了 15s（超过 TTL）→ 锁已自动过期
线程 B：加锁成功，开始处理同一笔订单    ← 互斥已被打破
线程 A：业务做完，执行 DEL lock:order:88 → 删掉的是 B 的锁！
线程 C：趁虚而入又拿到锁……互斥彻底失效
```

这就是经典的**误删问题**。解法分两步：**加锁时存入唯一标识**（如 UUID），**释放时先核对标识，是自己的锁才删**。

### 8.4 为什么需要 Lua 脚本原子释放

"核对标识再删除"拆成两条命令仍有漏洞：

```python
# 错误示范：GET 判断通过、DEL 还没执行的瞬间锁刚好过期被 B 拿走，
# 接下来 DEL 又删了 B 的锁
if r.get("lock:order:88") == my_token:
    r.delete("lock:order:88")
```

GET 和 DEL 之间隔着一次网络往返，什么事情都可能发生。必须把"判断 + 删除"合成**一次原子操作**——这正是 Lua 脚本的用途：Redis 会把整个脚本当作一条命令在单线程里执行完，中间不会插入任何其他命令：

```python
UNLOCK_SCRIPT = """
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
"""

def release_lock(r, lock_key: str, token: str) -> bool:
    # eval(script, 1, key, token)：1 表示 KEYS 数组里有 1 个元素
    return r.eval(UNLOCK_SCRIPT, 1, lock_key, token) == 1
```

完整的安全用锁模板：

```python
import uuid

def with_lock(r, lock_key: str, ttl: int, business_fn):
    token = uuid.uuid4().hex
    ok = r.set(lock_key, token, nx=True, ex=ttl)
    if not ok:
        raise RuntimeError("系统繁忙，请稍后重试")
    try:
        return business_fn()
    finally:
        release_lock(r, lock_key, token)   # 只删自己的锁
```

### 8.5 看门狗：锁的自动续期

还有一个隐患：TTL 设多少合适？设短了（10s），业务没执行完锁就过期，互斥失效；设长了（10min），持锁者宕机后其他实例要等 10 分钟。这个两难的标准答案是**看门狗（watchdog）机制**：

- 加锁时给较短的 TTL（如 10s）。
- 同时启动一个后台线程（看门狗），每隔 TTL/3 的时间检查一次："我还持有这把锁吗？业务跑完没有？"如果业务还在跑，就执行 `EXPIRE` 把锁续期 10s。
- 业务正常结束 → 停掉看门狗，Lua 脚本释放锁；进程宕机 → 看门狗线程一起死掉，不再续期，锁 10s 后自然过期，无死锁。

著名 Java 客户端 Redisson 的分布式锁就是这么实现的（默认 30s 锁、每 10s 续期）。Python 生态没有完全等价的成熟实现，生产环境的常见选择是：用 `python-redis-lock`（自带自动续期）或者自己实现一个简化版看门狗：

```python
import threading, time

class RedisLock:
    def __init__(self, r, key: str, ttl: int = 10):
        self.r, self.key, self.ttl = r, key, ttl
        self.token = uuid.uuid4().hex
        self._stop = threading.Event()
        self._dog: threading.Thread | None = None

    def acquire(self) -> bool:
        if not self.r.set(self.key, self.token, nx=True, ex=self.ttl):
            return False
        self._dog = threading.Thread(target=self._watchdog, daemon=True)
        self._dog.start()
        return True

    def _watchdog(self):
        # 每 ttl/3 秒续期一次，直到 release() 被调用
        while not self._stop.wait(self.ttl / 3):
            self.r.expire(self.key, self.ttl)

    def release(self):
        self._stop.set()
        release_lock(self.r, self.key, self.token)
```

注意续期也应校验 token（防给别人的锁续期），生产代码中把 `EXPIRE` 换成带判断的 Lua 脚本即可，思路相同。

---

## 九、典型应用

### 9.1 短信验证码

需求拆解：发送验证码（5 分钟有效）、校验验证码、防刷（1 分钟只能发 1 条）。全用 string + TTL 实现：

```python
import random

CODE_TTL = 300          # 验证码 5 分钟有效
SEND_INTERVAL = 60      # 60 秒内不允许重发

def send_code(phone: str) -> str:
    limit_key = f"sms:limit:{phone}"
    # 用 NX 做发送频率限制：60 秒内已存在则拒绝
    if not r.set(limit_key, "1", nx=True, ex=SEND_INTERVAL):
        raise RuntimeError("发送太频繁，请稍后再试")

    code = f"{random.randint(0, 999999):06d}"
    r.set(f"sms:code:{phone}", code, ex=CODE_TTL)
    # send_sms(phone, code)  # 调用短信服务商
    return code

def verify_code(phone: str, user_input: str) -> bool:
    key = f"sms:code:{phone}"
    real = r.get(key)
    if real is not None and real == user_input:
        r.delete(key)       # 一次性：验证成功后立即作废
        return True
    return False
```

两个细节：频率限制用 `SET NX EX` 天然实现（key 存在即表示还在冷却期）；验证码**校验成功即删**，防止同一个码被重复使用。

### 9.2 登录 token 存储与踢人下线

JWT 本身无状态，但也因此**无法主动失效**——改密码、封号时旧 token 在过期前依然有效。把会话状态放进 Redis 补上了这个短板：

```python
import secrets

SESSION_TTL = 7 * 24 * 3600    # 会话 7 天

def login(user_id: int) -> str:
    token = secrets.token_hex(32)
    # token → user_id，用于鉴权反查
    r.set(f"session:{token}", user_id, ex=SESSION_TTL)
    # user_id → token，用于"单点登录"：记录该用户当前唯一的会话
    r.set(f"session:uid:{user_id}", token, ex=SESSION_TTL)
    return token

def get_current_user(token: str) -> int | None:
    uid = r.get(f"session:{token}")
    return int(uid) if uid else None

def kick_user(user_id: int) -> None:
    """踢人下线：删除该用户的会话"""
    old_token = r.get(f"session:uid:{user_id}")
    if old_token:
        r.delete(f"session:{old_token}")
    r.delete(f"session:uid:{user_id}")

def login_exclusive(user_id: int) -> str:
    """单设备登录：新登录踢掉旧会话"""
    kick_user(user_id)
    return login(user_id)
```

"踢人下线""单设备登录（后登录挤掉先登录）""修改密码后所有会话失效"这类需求，都只是"删掉 Redis 里的 key"而已——这就是服务端可控会话的价值。请求到来时先查 `session:{token}`，存在则顺便 `EXPIRE` 续期，就实现了"活跃自动续期、闲置自动过期"。

### 9.3 秒杀限流

限流保护的是下游：把超出系统承载能力的请求在最上层挡掉。用 Redis 做**固定窗口限流**只要两行：

```python
def allow_request(user_id: int, limit: int = 10, window: int = 60) -> bool:
    """每个用户每分钟最多 10 个请求"""
    key = f"rate:{user_id}:{int(time.time() // window)}"   # key 里带窗口编号
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)    # 窗口内第一次访问才设过期，避免 key 泄漏
    return count <= limit
```

`INCR` 的原子性保证了一千个并发请求拿到的是一千个不同的递增值，限流计数精确无竞争。固定窗口在窗口边界有"双倍流量"缺陷（两个窗口交界处可能通过 2 倍请求），要求更平滑时用滑动窗口（zset 记录每次请求时间戳）或令牌桶（Lua 脚本实现）。

秒杀场景的完整链路一般是：限流挡掉超额请求 → 库存预热到 Redis，用 `DECR` 原子扣减（超卖防护第一道）→ 扣减成功的请求投递到消息队列 → 消费者异步落库下单（数据库只承受真实成交量级）。Redis 在前面扛住了洪峰的 99%，数据库只见涓涓细流。

---

## 十、持久化：RDB 与 AOF

Redis 是内存数据库，断电即失。持久化解决"重启后数据还在"的问题，有两种机制，可单独或同时开启。

### 10.1 RDB：定时快照

RDB（Redis Database）在**满足条件时把某一时刻的全量内存数据生成二进制快照**写入磁盘（`dump.rdb`）。触发方式：

- 配置规则：`save 900 1`（900 秒内有 1 次写就触发）、`save 300 100` 等；
- 手动执行 `SAVE`（**阻塞主线程**，生产禁用）或 `BGSAVE`（后台执行）；
- 主从全量同步、正常关闭 Redis 时也会触发。

`BGSAVE` 的原理值得讲清楚：主线程 `fork()` 出一个子进程。**fork 的瞬间子进程与父进程共享同一份内存页**（操作系统的写时复制 COW 机制），子进程把这份"冻结的"内存慢慢写入磁盘；期间父线程继续处理请求，一旦有写操作修改了某内存页，操作系统先把该页**复制一份**再给父进程改——子进程看到的快照始终一致。所以 RDB 期间内存占用最坏可能翻倍（所有页都被改过），内存要留足余量。

**优点**：二进制快照紧凑、体积小，恢复时直接 load 进内存，**重启恢复速度远快于 AOF**；对线上性能影响小（只有 fork 的一瞬间）。
**缺点**：快照是"某一时刻"的，两次快照之间的数据变更在宕机时**全部丢失**——可能丢几分钟的数据。

### 10.2 AOF：追加命令日志

AOF（Append Only File）把**每一条写命令**按文本协议追加到日志文件末尾，重启时重新执行一遍所有命令来恢复数据——类似 MySQL 的 binlog。

刷盘策略 `appendfsync` 是三档取舍：

- `always`：每条命令都 fsync 落盘。最安全（最多丢一条），最慢（每次写都等磁盘）。
- `everysec`（默认）：每秒 fsync 一次。宕机最多丢 1 秒数据，性能接近纯内存。**绝大多数场景的选择**。
- `no`：交给操作系统决定（通常 30 秒一次）。最快但最不安全。

AOF 文件只增不减会无限膨胀，所以 Redis 提供 **AOF 重写**：`BGREWRITEAOF` fork 子进程，根据当前内存数据**生成一份达到同样状态的最短命令序列**（例如 `INCR` 执行了一万次，重写后就是一条 `SET key 10000`），替换旧文件。重写期间的新命令会同时写入 AOF 重写缓冲区，结束后追加进新文件，保证不丢。

**优点**：数据安全性远高于 RDB（everysec 最多丢 1 秒）。
**缺点**：文件体积通常比 RDB 大；恢复时要逐条重放命令，**重启慢**；极端情况下 AOF 文件可能损坏（可用 `redis-check-aof` 修复截断）。

### 10.3 生产环境的取舍

- **只做缓存（丢了能重建）**：可以两个都不开，性能最好。
- **一般业务（缓存 + 少量状态）**：RDB + AOF（everysec）同时开。Redis 重启时**优先加载 AOF**（因为它更完整），RDB 作为 AOF 文件损坏时的备份和快速全量恢复手段。
- **金融级数据**：AOF everysec 是底线，配合主从把数据冗余到别的机器——持久化防的是"重启丢数据"，防不了"机器硬盘报废"，真正的数据安全靠的是副本。

---

## 十一、过期策略与内存淘汰策略

### 11.1 过期 key 是怎么被删除的

给 key 设了 TTL，到期后 Redis 什么时候删它？直觉上的"定时器到点立刻删"并不可行——为几百万个 key 维护定时器，到期事件风暴会把 CPU 拖垮。Redis 实际采用两种策略的组合：

**惰性删除（lazy expiration）**：访问 key 的时候才检查是否过期，过期则删除并返回空。CPU 友好（只为被访问的 key 付出检查成本），但对内存不友好——一个过期后再也没人访问的 key 会一直占着内存，形同泄漏。

**定期删除（active expiration）**：Redis 每秒钟跑若干次**过期扫描**：从设置了过期时间的 key 的字典里**随机抽取一小批**（默认 20 个），删除其中已过期的；如果过期比例超过阈值（25%），就再抽一批继续删，直到比例降下来或时间片用完。"随机抽样 + 比例控制"保证这个后台任务不会阻塞主线程太久。

两种策略配合的效果：热 key 靠惰性删除即访问即清，冷 key 靠定期扫描慢慢收。**这是"用概率换性能"的典型工程权衡**——不保证每个过期 key 立刻消失，但保证过期 key 不会无限堆积、清理成本不会失控。

### 11.2 lazy free：异步删除

删除本身也可能阻塞：`DEL` 一个有几百万成员的集合，主线程要释放所有成员的内存，耗时可能上百毫秒——期间所有请求被卡住。Redis 4.0 引入 **lazy free（惰性释放）**：`UNLINK key` 只做"把 key 从字典里摘掉"这个 O(1) 操作，真正的内存释放交给后台线程慢慢做。业务代码里删除大 key 应优先用 `UNLINK`：

```python
r.unlink("big:set:xxx")   # 对大 key 用 UNLINK 代替 DEL
```

同时配置 `lazyfree-lazy-expire yes` 等选项，让过期删除、淘汰删除也走异步路径。

### 11.3 内存淘汰策略：内存满了怎么办

`maxmemory` 限定了 Redis 的内存上限。达到上限后又有新写请求时，由 `maxmemory-policy` 决定淘汰谁：

| 策略 | 行为 |
| --- | --- |
| `noeviction`（默认） | 不淘汰，写请求直接报错（读正常） |
| `allkeys-lru` | 从**所有 key** 中淘汰最近最少使用的 |
| `allkeys-lfu` | 从所有 key 中淘汰使用频率最低的（4.0+） |
| `volatile-lru` | 只在**设了过期时间**的 key 中按 LRU 淘汰 |
| `volatile-lfu` | 只在设了过期时间的 key 中按 LFU 淘汰 |
| `allkeys-random` / `volatile-random` | 随机淘汰 |
| `volatile-ttl` | 在设了过期时间的 key 中优先淘汰 TTL 最短的 |

**LRU vs LFU 的区别**：LRU（Least Recently Used）按"最近一次访问时间"淘汰，容易被一次性流量污染——某个冷 key 刚被偶然访问一次，反而比每天都被访问、但碰巧几分钟没访问的热 key 活得更久。LFU（Least Frequently Used）按"访问频率"淘汰，能识别真正的长期热点。Redis 的 LFU 用概率计数器（8 bit 计数 + 随时间衰减）实现，只花极小代价就逼近了精确频率统计。

需要注意 Redis 的 LRU/LFU 都是**近似实现**：维护一个候选池，淘汰时随机抽样若干 key 取最冷者，而非精确排序——又是"用概率换性能"，这在工程上已足够好。

**选型建议**：

- **纯缓存场景**（所有数据都能从 DB 重建）：`allkeys-lru` 或 `allkeys-lfu`，内存满了自动挤掉冷数据，热数据始终在位。
- **缓存 + 持久数据混合**：`volatile-lru` 并确保持久数据**不设 TTL**——但风险是若忘了给缓存设 TTL，缓存会挤占内存且永不淘汰，最终写请求报错。混合部署更稳妥的做法本来就是拆实例。
- **`noeviction` 适合把 Redis 当数据库用的场景**：宁可写失败也不丢数据，配合监控在内存逼近上限时告警扩容。

---

## 小结

- Redis 快是三个因素叠加：**纯内存操作**（纳秒级访问）、**单线程执行**（无锁、无上下文切换，命令天然原子）、**IO 多路复用**（epoll 让一个线程服务上万连接，事件循环永不空转）。瓶颈在内存和网络而非 CPU，所以单线程不慢。
- 五大数据结构选型口诀：缓存/计数/锁用 **string**，对象字段独立读写用 **hash**，简单队列和最新列表用 **list**，去重和共同好友用 **set**，排行榜和延迟队列用 **zset**。
- 生产禁忌：`KEYS *`、对大 key 用 `DEL`/`HGETALL`/`SMEMBERS`。改用 `SCAN` 家族和 `UNLINK`。
- Python 侧要点：全局共享一个连接池；异步框架必须用 `redis.asyncio`；批量操作用 pipeline；FastAPI 用 lifespan 管理连接池 + Depends 注入。
- 缓存模式记结论：**读走 cache miss 回源回填，写先更 DB 再删缓存**，TTL 兜底，强一致不可达只能追最终一致。
- 三大问题对症药：**穿透**→缓存空值/布隆过滤器；**击穿**（热点过期）→互斥锁或逻辑过期；**雪崩**（批量过期/宕机）→TTL 加随机抖动 + 熔断降级 + 高可用。
- 分布式锁四要素：`SET NX EX` 原子加锁、唯一标识防误删、Lua 脚本原子释放、看门狗自动续期。
- 持久化：RDB 恢复快但可能丢几分钟，AOF（everysec）最多丢 1 秒但恢复慢，生产通常混开、重启优先加载 AOF。
- 内存管理：过期删除靠"惰性 + 定期抽样"；大 key 删除用 `UNLINK`；淘汰策略缓存场景选 `allkeys-lru/lfu`，LFU 比 LRU 更能识别长期热点。

## 常见问题

**1. Redis 单线程为什么还能这么快？多路复用到底是什么？**

瓶颈不在 CPU 而在内存和网络 IO，内存操作的微秒级耗时让单线程足以消化十万级 QPS；单线程反而省掉了锁竞争和上下文切换的开销。IO 多路复用（Linux 下为 epoll）是把"等待上万个连接的数据"委托给操作系统，一次系统调用拿到所有就绪连接，主线程只处理有数据的连接，从不空等——单线程因此能同时服务海量连接。

**2. 为什么更新缓存的标准做法是"先更新数据库，再删除缓存"，而不是"更新缓存"或"先删缓存"？**

并发下两个写线程的"更新缓存"顺序无法保证，后写 DB 的线程可能后刷缓存，留下旧值；"先删缓存再写库"会被读线程插队——读线程在删缓存后回源拿到旧值并回填。正确方案不一致的窗口极窄（要求读回填恰好慢于整个写+删，而写通常远慢于读），配合延迟双删和 TTL 兜底，工程上足够安全。

**3. 布隆过滤器为什么能防穿透？它说"存在"的数据真的存在吗？**

布隆过滤器是位数组 + 多个哈希函数：添加元素时把多个哈希位置 1，查询时这些位全是 1 才判"可能存在"。位被其他元素置 1 会造成**误判**（说不存在的元素可能存在），所以"存在"只是可能；但只要有一位是 0 就**一定不存在**——用"一定不存在"的结论在缓存之前直接拦截非法 id，零漏判，这正是防穿透需要的性质。它不支持删除（位被共享），元素变更需要重建。

**4. 分布式锁为什么要用 Lua 脚本释放？看门狗解决什么问题？**

释放锁需要"先 GET 判断标识、再 DEL"两步，两步之间锁可能恰好过期并被他人获取，DEL 就会误删别人的锁；Lua 脚本被 Redis 当作一条命令在单线程内原子执行，消除了这个间隙。看门狗解决 TTL 设置的两难：设短了业务没完锁就丢，设长了持锁者宕机后长期死锁。看门狗让持锁线程定期续期（正常执行时锁不过期），宕机后看门狗随之消亡、锁自然过期，兼顾两者。

**5. RDB 和 AOF 同时开启时，重启加载哪个？为什么 AOF 恢复更慢？**

优先加载 AOF，因为 AOF（everysec 最多丢 1 秒）的数据完整性高于 RDB（可能丢数分钟），且 RDB 文件可能是更旧的快照。AOF 恢复慢是因为它本质是**命令重放**：文件里有多少条写命令就要重新执行多少条；而 RDB 是二进制内存镜像，直接反序列化 load 进内存即可。AOF 重写通过把中间状态压缩成最短命令序列来缓解文件膨胀和重放耗时。

**6. 缓存和数据库的双写一致性，能不能做到强一致？**

不能（在不引入分布式事务的前提下）。两个独立系统之间的写操作总有先后，任一时刻都可能存在"一个已更新、一个未更新"的中间态，而任何"补偿动作"（删缓存、发消息）本身也可能失败。工程上只能追求最终一致：正确顺序（先 DB 后删缓存）+ TTL 兜底 + 失败重试/binlog 订阅。对余额这类强一致数据，正确选择是不走缓存、直读数据库或加数据库行锁。
