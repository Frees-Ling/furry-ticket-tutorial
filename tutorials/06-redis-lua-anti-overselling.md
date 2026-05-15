# 第6章：Redis 深入 + Lua 脚本 + 防超卖核心

## 本章学习目标

本章是票务系统**最关键**的一章。将从零学习 Redis 全部数据结构和命令、Lua 5.1 完整语法，最终实现防超卖核心逻辑。

**学习路径：**

1. ✅ Redis 7.x 全部数据结构与命令详解
2. ✅ Lua 5.1 完全教程（全部语法/API）
3. ✅ Redis Lua 脚本原子性证明
4. ✅ redis-py 完全教程
5. ✅ 缓存模式详解（穿透/击穿/雪崩）
6. ✅ 防超卖核心实现（双层防御）
7. ✅ 速率限制实现

---

## 1. Redis 7.x 完全教程

### 1.1 Redis 概述

Redis（Remote Dictionary Server）是一个**内存型**键值数据库，以**单线程**事件循环处理命令，因此所有操作都是原子的。

**核心特性：**

- 纯内存操作（微秒级延迟）
- 单线程模型（无需锁）
- 丰富的数据结构（String/List/Set/Hash/ZSet/Stream/Bitmap/HyperLogLog）
- 持久化（RDB 快照 + AOF 日志）
- 主从复制/哨兵/集群

**与 MySQL 的区别：**

| 特性 | MySQL | Redis |
| ------ | ------- | ------- |
| 数据存储 | 磁盘 | 内存 |
| 延迟 | 毫秒级 | 微秒级 |
| 数据结构 | 表/行 | String/List/Hash/Set/ZSet |
| 查询语言 | SQL | 命令集 |
| 事务 | ACID（复杂） | 原子命令/Lua脚本 |
| 持久化 | 天生持久 | RDB/AOF 可选 |
| 容量 | TB 级 | GB 级（受内存限制） |

### 1.2 连接与基础命令

```bash
# 进入 Redis CLI
docker exec -it ticket-redis redis-cli

# 基本使用
redis> PING                    # 测试连接 → PONG
redis> SET key value           # 设置键值
redis> GET key                 # 获取值 → "value"
redis> DEL key                 # 删除键 → 1
redis> EXISTS key              # 判断存在 → 0/1
redis> TYPE key               # 返回类型 → string
redis> KEYS *                  # 列出所有键（⚠️危险！生产禁用）
redis> SCAN 0 MATCH * COUNT 10 # 游标遍历（生产使用）
```text

**键命名规范：**

```

# 推荐格式：项目:实体:ID:属性

ticket:event:100:stock        # 活动 100 的库存
ticket:order:200:status       # 订单 200 的状态
ticket:user:300:token         # 用户 300 的令牌

# 优点：格式统一、按冒号前缀分类、Redis 内部按前缀索引

```text

### 1.3 String（字符串）

```bash
# ===== 基础操作 =====
SET name "张三"                # 设置键值
GET name                       # "张三"
SET name "李四" XX             # 仅当键已存在
SET name "张三" NX             # 仅当键不存在
GETSET name "王五"             # 获取旧值，设置新值
SETEX name 60 "张三"           # 设置 + 过期时间（60秒）
PSETEX name 60000 "张三"       # 设置 + 过期时间（毫秒）
MSET k1 v1 k2 v2 k3 v3        # 批量设置
MGET k1 k2 k3                  # 批量获取 → ["v1","v2","v3"]
STRLEN name                    # 字符串长度

# ===== 递增递减 =====
INCR counter                   # 递增 1 → 1
INCRBY counter 10              # 递增 10 → 11
DECR counter                   # 递减 1 → 10
DECRBY counter 5               # 递减 5 → 5
INCRBYFLOAT price 9.99         # 浮点递增

# ===== 位操作 =====
SETBIT status 0 1              # 设置第 0 位为 1
GETBIT status 0                # → 1
BITCOUNT status                # 统计 1 的个数
BITOP AND dest src1 src2       # 位运算

# ===== 子串 =====
GETRANGE name 0 1              # 取子串（左闭右闭）
SETRANGE name 0 "赵"           # 覆盖子串
APPEND name "先生"              # 追加

# ===== 原子操作 =====
SETNX lock:order "1"           # 不存在时才设置（分布式锁）
# 返回 1 表示抢到锁，0 表示锁已被占用
```

**String 内部编码：**

```text
- INT：整数，8 字节
- EMBSTR：短字符串（≤44 字节）
- RAW：长字符串（>44 字节）

查看编码：
OBJECT ENCODING key
```

### 1.4 List（列表）

```bash
# ===== 入队出队 =====
LPUSH queue "task1"            # 左侧推入
RPUSH queue "task2"            # 右侧推入
LPOP queue                     # 左侧弹出 → "task2"
RPOP queue                     # 右侧弹出 → "task1"

# ===== 阻塞操作 =====
BLPOP queue 10                 # 左侧阻塞弹出（超时10秒）
BRPOP queue 10                 # 右侧阻塞弹出（超时10秒）

# ===== 范围操作 =====
LLEN queue                     # 列表长度
LRANGE queue 0 -1              # 全部元素
LINDEX queue 0                 # 索引为 0 的元素
LSET queue 0 "new"             # 设置索引为 0 的元素
LINSERT queue BEFORE "a" "b"   # 在元素前插入
LREM queue 2 "a"               # 删除前 2 个"a"
LTRIM queue 0 99               # 修剪列表（保留前100个）

# ===== 应用模式 =====
# 消息队列：一个 LPUSH，多个 RPOP
# 栈：LPUSH + LPOP
# 有界集合：LPUSH + LTRIM（保留最新 N 条）
```text

### 1.5 Set（集合）

```bash
# ===== 基础操作 =====
SADD tags "python"             # 添加元素
SREM tags "java"               # 删除元素
SISMEMBER tags "python"        # 判断存在 → 1
SMEMBERS tags                  # 所有元素
SCARD tags                     # 元素个数
SPOP tags                      # 随机弹出一个
SRANDMEMBER tags 3             # 随机取 3 个（不删除）

# ===== 集合运算 =====
SINTER set1 set2               # 交集
SUNION set1 set2               # 并集
SDIFF set1 set2                # 差集

SINTERSTORE dest set1 set2     # 交集结果存到 dest
SUNIONSTORE dest set1 set2
SDIFFSTORE dest set1 set2

# ===== 游标遍历 =====
SSCAN tags 0 MATCH * COUNT 10

# ===== 应用场景 =====
# 标签系统：SADD event:1:tags "演唱会" "流行"
# 去重：SAFD uid 123（自动去重）
# 随机抽奖：SPOP / SRANDMEMBER
```

### 1.6 Hash（哈希）

```bash
# ===== 基础操作 =====
HSET user:1 username "张三"     # 设置字段
HSET user:1 age 25 email "test@example.com"
HGET user:1 username            # → "张三"
HMGET user:1 username age       # → ["张三", "25"]
HGETALL user:1                  # 所有字段和值
HKEYS user:1                   # 所有字段名
HVALS user:1                   # 所有值
HDEL user:1 age                # 删除字段
HEXISTS user:1 age             # 判断字段存在 → 0
HLEN user:1                    # 字段数

# ===== 数值操作 =====
HINCRBY user:1 age 1           # 递增字段
HINCRBYFLOAT user:1 score 0.5

# ===== 游标遍历 =====
HSCAN user:1 0 MATCH * COUNT 10

# ===== 应用场景 =====
# 对象存储（比 String 更节省内存，支持部分读写）
# 购物车：HSET cart:user1 product_id quantity
```text

### 1.7 Sorted Set（有序集合）

ZSet 是 Redis 最强大的数据结构。每个元素关联一个 score，按 score 排序。

```bash
# ===== 基础操作 =====
ZADD leaderboard 100 "张三"     # 添加成员
ZADD leaderboard 90 "李四" 80 "王五"
ZREM leaderboard "王五"         # 删除成员
ZCARD leaderboard               # 成员数
ZSCORE leaderboard "张三"       # 获取 score → 100
ZINCRBY leaderboard 10 "张三"   # 增加 score

# ===== 范围查询 =====
ZRANGE leaderboard 0 -1                    # 按排名升序
ZREVRANGE leaderboard 0 -1                 # 按排名降序
ZRANGE leaderboard 0 -1 WITHSCORES         # 带分数
ZRANGEBYSCORE leaderboard 80 100           # 按分数范围
ZREVRANGEBYSCORE leaderboard 100 80        # 降序分数范围
ZRANK leaderboard "张三"                   # 排名（从0开始）
ZREVRANK leaderboard "张三"                # 倒序排名

# ===== 删除操作 =====
ZREM leaderboard "李四"                    # 删除成员
ZREMRANGEBYRANK leaderboard 0 0            # 删除最低分成员
ZREMRANGEBYSCORE leaderboard 0 60          # 删除指定分数范围

# ===== 集合运算 =====
ZINTERSTORE dest 2 z1 z2 WEIGHTS 1 0.5    # 加权交集
ZUNIONSTORE dest 2 z1 z2                  # 并集

# ===== 应用场景 =====
# 排行榜：ZINCRBY leaderboard score user_id
# 延时队列：ZADD queue timestamp task
# 滑动窗口限流：ZREMRANGEBYSCORE window min now
```

### 1.8 Stream（流）

Stream 是 Redis 5.0 引入的日志型数据结构，支持消费者组。

```bash
# ===== 生产消息 =====
XADD mystream * field1 value1 field2 value2
# 自动生成时间戳 ID：1710500000000-0
XADD mystream MAXLEN ~ 10000 * field value
# 有界追加（保留约 10000 条）

# ===== 消费消息 =====
XRANGE mystream - +            # 全部消息
XREAD COUNT 10 STREAMS mystream 0  # 从ID=0开始读
XREAD BLOCK 5000 STREAMS mystream $ # 阻塞读取最新消息

# ===== 消费者组 =====
XGROUP CREATE mystream mygroup $  # 创建消费者组
XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream >
# > 表示从组中未分配的 ID 开始

XACK mystream mygroup message_id   # 确认处理完成
XPENDING mystream mygroup          # 查看待处理
XCLAIM mystream mygroup consumer1 3600000 msg_id  # 转移消息所有权
```text

### 1.9 其他数据结构

```bash
# ===== Bitmaps（位图） =====
SETBIT signin:2024-01-01 user:100 1    # 用户100签到
BITCOUNT signin:2024-01-01             # 当天签到数
BITOP AND dest signin:01 signin:02     # 两天都签到的用户

# ===== HyperLogLog（基数估计） =====
PFADD visitors "user1" "user2" "user3"
PFCOUNT visitors                      # 近似计数 → 3
PFMERGE dest src1 src2                # 合并

# ===== Geospatial（地理位置） =====
GEOADD locations 116.39 39.91 "天安门"
GEODIST locations "天安门" "故宫" km   # 计算距离
GEORADIUS locations 116.39 39.91 5 km # 附近5公里

# ===== Bitfields（位域） =====
BITFIELD mykey SET u8 0 255          # 设置无符号8位，偏移0，值255
BITFIELD mykey INCRBY u8 0 1         # 递增
```

### 1.10 键管理

```bash
# ===== 过期时间 =====
EXPIRE key 60                # 设置过期时间（秒）
EXPIREAT key 1710500000      # 设置过期时间戳（秒）
TTL key                      # 查看剩余时间（秒）
PTTL key                     # 查看剩余时间（毫秒）
PERSIST key                  # 移除过期时间

# ===== 重命名 =====
RENAME key newkey            # 重命名（key 存在则覆盖）
RENAMENX key newkey          # 重命名（newkey 不存在时才执行）

# ===== 序列迁移 =====
DUMP key                     # 序列化成 RDB 格式
RESTORE key 0 data           # 恢复
MIGRATE host port key db timeout  # 迁移到另一台 Redis

# ===== 数据库 =====
SELECT 0                     # 切换数据库（0-15）
FLUSHDB                      # 清空当前数据库
FLUSHALL                     # 清空所有数据库
DBSIZE                       # 键的数量
```text

### 1.11 SCAN 游标遍历（替代 KEYS）

```python
# KEYS * 的缺点：阻塞 Redis，生产禁用（单线程！）
# SCAN 的游标遍历，每次返回少量数据，不阻塞

import redis

r = redis.Redis()
cursor = 0
while True:
    cursor, keys = r.scan(cursor, match="ticket:*", count=100)
    for key in keys:
        print(key)
    if cursor == 0:
        break
```

### 1.12 Redis 事务

```bash
# Redis 事务：MULTI/EXEC/DISCARD

MULTI                    # 开始事务
SET stock:100 100        # 命令入队
DECR stock:100           # 命令入队（也入队，不执行）
EXEC                     # 依次执行所有命令 → [OK, 99]

# 事务中的错误处理：
# 语法错误：整个事务不执行（全部放弃）
# 运行时错误（如对 string 执行 LPUSH）：其他命令正常执行

# WATCH 乐观锁
WATCH stock:100          # 监视键
stock = GET stock:100    # 获取值
MULTI
SET stock:100 $new_val   # 基于旧值计算新值
EXEC                     # 如果 stock:100 被修改，EXEC 返回 nil（事务失败）
```text

**Redis 事务 vs 关系型数据库事务：**

| 特性 | Redis 事务 | SQL 事务 |
| ------ | ----------- | --------- |
| 原子性 | ✅ 全执行或全不执行 | ✅ |
| 隔离性 | ✅ 队列执行（无穿插）| 隔离级别 |
| 回滚 | ❌ 不支持 | ✅ |
| 持久性 | ❌（取决于配置） | ✅ WAL |
| 锁定 | WATCH 乐观锁 | MVCC/行锁 |
| 嵌套 | ❌ 不支持 | ✅ 保存点 |

---

## 2. Lua 5.1 完全教程

Lua 是 Redis 的扩展语言，可以用脚本实现复杂原子操作。

### 2.1 Lua 基本语法

```lua
-- ===== 注释 =====
-- 单行注释
--[[ 多行
     注释 ]]

-- ===== 变量 =====
-- 全局变量（默认）
name = "张三"
age = 25
-- 局部变量
local money = 99.99
local is_vip = true

-- ===== 全部类型 =====
print(type(nil))            --> nil
print(type(true))           --> boolean
print(type(42))             --> number（Lua 只有 number，没有 int/float 之分）
print(type("hello"))        --> string
print(type({}))             --> table
print(type(function() end)) --> function
print(type(coroutine.create(function() end))) --> thread
print(type(io.stdin))       --> userdata

-- ===== 运算符 =====
-- 算术：+ - * / ^（幂） %（取模）
-- 关系：== ~= < > <= >=
-- 逻辑：and or not
-- 连接：..（字符串连接）
-- 长度：#（取字符串长度或 table 长度）

print(2 ^ 10)           --> 1024.0（幂运算）
print("Hello" .. " " .. "World")  --> "Hello World"
print(#"abcdef")        --> 6

-- ===== 控制流 =====
-- if/elseif/else
if age < 18 then
    print("未成年")
elseif age >= 18 and age < 60 then
    print("成年")
else
    print("老年")
end

-- while
local i = 1
while i <= 5 do
    print(i)
    i = i + 1
end

-- repeat until（相当于 do-while）
local j = 1
repeat
    print(j)
    j = j + 1
until j > 5

-- for 循环（数值）
for i = 1, 5 do         -- 1,2,3,4,5
    print(i)
end
for i = 1, 10, 2 do     -- 1,3,5,7,9（步长2）
    print(i)
end

-- for 循环（泛型）
local fruits = {"苹果", "香蕉", "橙子"}
for index, value in ipairs(fruits) do
    print(index, value)
end

-- pairs vs ipairs
-- ipairs：遍历数组部分（从 1 开始连续索引）
-- pairs：遍历所有键值对（无序）

-- ===== 函数 =====
-- 基础函数
function greet(name)
    return "你好, " .. name
end
print(greet("张三"))    --> "你好, 张三"

-- 多返回值
function min_max(arr)
    local min = arr[1]
    local max = arr[1]
    for _, v in ipairs(arr) do
        if v < min then min = v end
        if v > max then max = v end
    end
    return min, max
end
local min, max = min_max({3, 1, 4, 1, 5, 9})
print(min, max)         --> 1      9

-- 变长参数
function sum(...)
    local args = {...}
    local total = 0
    for _, v in ipairs(args) do
        total = total + v
    end
    return total
end
print(sum(1, 2, 3, 4)) --> 10

-- 闭包
function counter()
    local count = 0
    return function()
        count = count + 1
        return count
    end
end
local c = counter()
print(c())  --> 1
print(c())  --> 2
```

### 2.2 Table（Lua 核心数据结构）

Table 是 Lua 中**唯一**的数据结构（同时实现数组和字典）。

```lua
-- ===== 作为数组 =====
local arr = {10, 20, 30, 40}
print(arr[1])           --> 10（Lua 索引从 1 开始！）
print(#arr)             --> 4（数组长度）

-- ===== 作为字典 =====
local person = {
    name = "张三",
    age = 25,
    email = "zhangsan@example.com",
}
print(person.name)      --> 张三（语法糖）
print(person["age"])    --> 25（完整语法）

-- ===== 混合 =====
local mixed = {
    "苹果",              -- [1] = "苹果"
    "香蕉",             -- [2] = "香蕉"
    price = 5.5,
    quantity = 100,
}
print(mixed[1])         --> 苹果
print(mixed.price)      --> 5.5

-- ===== table 库 =====
local t = {3, 1, 4, 1, 5}
table.insert(t, 9)               -- 末尾插入
table.insert(t, 2, 99)           -- 位置 2 插入
table.remove(t, 3)               -- 删除位置 3
table.sort(t)                    -- 排序
table.concat({"a", "b", "c"}, ",")  --> "a,b,c"
table.unpack({1, 2, 3})         -- 拆包 → 1, 2, 3
```text

### 2.3 字符串库

```lua
local s = "Hello Lua"

print(#s)                       --> 9（长度）
print(string.upper(s))          --> "HELLO LUA"
print(string.lower(s))          --> "hello lua"
print(string.sub(s, 1, 5))      --> "Hello"
print(string.reverse(s))        --> "auL olleH"
print(string.rep("ab", 3))      --> "ababab"
print(string.format("年龄: %d, 姓名: %s", 25, "张三"))

-- 模式匹配（类似正则）
print(string.match("2024-01-15", "(%d+)-(%d+)-(%d+)"))
--> "2024" "01" "15"

print(string.gsub("hello world", "l", "L"))
--> "heLLo worLd"  3（替换结果，替换次数）
```

### 2.4 数学库

```lua
print(math.abs(-10))        --> 10
print(math.ceil(3.14))      --> 4
print(math.floor(3.14))     --> 3
print(math.max(1, 5, 3))    --> 5
print(math.min(1, 5, 3))    --> 1
print(math.pow(2, 10))      --> 1024.0
print(math.random())        --> [0, 1) 随机数
print(math.random(1, 100))  --> [1, 100] 随机整数
print(math.sqrt(100))       --> 10
print(math.pi)              --> 3.1415926535898
```text

### 2.5 元表（Metatable）

元表是 Lua 实现面向对象和运算符重载的机制：

```lua
-- ===== 算术运算重载 =====
local vec3 = {x = 1, y = 2, z = 3}
local mt = {
    __add = function(a, b)
        return {x = a.x + b.x, y = a.y + b.y, z = a.z + b.z}
    end,
    __tostring = function(t)
        return string.format("(%g, %g, %g)", t.x, t.y, t.z)
    end,
}
setmetatable(vec3, mt)

local v1 = {x = 1, y = 2, z = 3}
local v2 = {x = 4, y = 5, z = 6}
setmetatable(v1, mt)
setmetatable(v2, mt)

local v3 = v1 + v2
print(v3)                         --> (5, 7, 9)

-- ===== __index 实现继承 =====
local Animal = {type = "动物"}
function Animal:new(name)
    local obj = {name = name}
    setmetatable(obj, self)        -- self = Animal
    self.__index = self            -- 查找时从 Animal 上找
    return obj
end
function Animal:speak()
    return self.name .. "发出声音"
end

local Dog = Animal:new("狗")
function Dog:speak()
    return self.name .. "汪汪叫"
end

local my_dog = Dog:new("旺财")
print(my_dog:speak())             --> "旺财汪汪叫"
print(my_dog.type)                --> "动物"（通过 __index 链）
```

### 2.6 Redis Lua API

```lua
-- Redis 为 Lua 提供的 API：

-- redis.call()：执行 Redis 命令，错误时抛出异常
-- redis.pcall()：执行 Redis 命令，错误时返回错误对象（不抛异常）

local stock = redis.call("GET", KEYS[1])
if not stock then
    return redis.error_reply("库存键不存在")
end

local result = redis.pcall("DECRBY", KEYS[1], tonumber(ARGV[1]))
if result["err"] then
    return redis.error_reply("扣减失败")
end

-- redis.status_reply()：返回状态回复
return redis.status_reply("OK")

-- redis.log()：记录日志
redis.log(redis.LOG_WARNING, "库存不足")

-- 日志级别：
-- redis.LOG_DEBUG
-- redis.LOG_VERBOSE
-- redis.LOG_NOTICE
-- redis.LOG_WARNING
```text

---

## 3. Redis Lua 脚本原子性证明

### 3.1 EVAL 命令

```bash
# 加载并执行 Lua 脚本
EVAL "return 'Hello Redis'" 0
# → "Hello Redis"

# 带参数
EVAL "return {KEYS[1], ARGV[1]}" 1 stock:100 10
# → stock:100, 10

# 参数说明：
# EVAL script numkeys key1 key2 ... arg1 arg2 ...
# KEYS[1..N]：键名参数（告诉 Redis 哪些键会被访问）
# ARGV[1..N]：附加参数
```

### 3.2 原子性证明

**定理：** Redis 执行 Lua 脚本具有 SERIALIZABLE 级别的隔离性。

**证明：**

```text
Redis 使用单线程事件循环处理所有命令。

1. 当 Redis 开始执行一个 Lua 脚本时：
   - 整个脚本在同一个事件循环 tick 中运行
   - 直到脚本返回前，Redis 不处理任何其他命令
   - 没有其他客户端能在脚本执行期间插入操作

2. Lua 脚本内部的 redis.call() 调用：
   - 直接调用 Redis 内部的命令处理函数
   - 不经过事件循环队列
   - 因此不产生"可中断"点

3. 结论：
   脚本执行期间，Redis 完全停止处理其他请求。
   等价于 MySQL 的 SERIALIZABLE 隔离级别。
   
   换句话说：Lua 脚本是原子的。
   不存在"执行了一半脚本，另一个请求插入"的情况。
```

**数学证明（反证法）：**

```text
假设：两个客户端 C1 和 C2 同时发送 Lua 脚本 S1 和 S2。
S1 和 S2 的操作交错执行。

由于 Redis 是单线程事件循环：
- C1 的脚本 S1 开始执行 → 事件循环处理 S1
- S1 执行期间，C2 的请求在输入缓冲区等待
- S1 返回后 → 事件循环处理 C2 的 S2

所以在 S1 执行期间，不可能有 S2 的任何操作插入。
S1 要么完全在 S2 之前执行，要么完全在 S2 之后执行。

与假设矛盾，故 Lua 脚本的执行是原子的。
```

### 3.3 SCRIPT 命令

```bash
# 缓存脚本（返回 SHA 摘要）
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "4e6d2c36b13a..."

# 通过 SHA 执行（节省带宽）
EVALSHA "4e6d2c36b13a..." 1 stock:100

# 检查脚本是否存在
SCRIPT EXISTS "4e6d2c36b13a..."

# 清空脚本缓存
SCRIPT FLUSH

# 终止正在执行的脚本
SCRIPT KILL
```text

---

## 4. redis-py 完全教程

### 4.1 基础使用

```python
import redis

# 同步连接
r = redis.Redis(
    host="localhost",
    port=6379,
    db=0,
    decode_responses=True,  # 自动解码为字符串而非字节
)

# 基本操作
r.set("name", "张三")
print(r.get("name"))          # 张三
print(r.exists("name"))       # 1
print(r.delete("name"))       # 1

# 批量操作
r.mset({"k1": "v1", "k2": "v2"})
print(r.mget(["k1", "k2"]))   # ["v1", "v2"]

# 过期时间
r.setex("tmp", 60, "value")
print(r.ttl("tmp"))           # 剩余秒数

# 整数操作
r.set("counter", 0)
r.incr("counter")             # 1
r.incrby("counter", 10)       # 11
```

### 4.2 各数据结构操作

```python
# List
r.lpush("queue", "task1", "task2")
r.rpush("queue", "task3")
print(r.lpop("queue"))        # task2（左侧弹出）
print(r.lrange("queue", 0, -1))  # ["task1", "task3"]
print(r.llen("queue"))        # 2

# Set
r.sadd("tags", "python", "redis", "lua")
print(r.smembers("tags"))     # {"python", "redis", "lua"}
print(r.sismember("tags", "go"))  # False

# Hash
r.hset("user:1", "username", "张三")
r.hset("user:1", "age", 25)
print(r.hgetall("user:1"))    # {"username": "张三", "age": "25"}
print(r.hincrby("user:1", "age", 1))  # 26

# Sorted Set
r.zadd("leaderboard", {"张三": 100, "李四": 90, "王五": 80})
print(r.zrange("leaderboard", 0, -1, withscores=True))
# [("王五", 80.0), ("李四", 90.0), ("张三", 100.0)]
print(r.zrevrange("leaderboard", 0, 0))  # 第一名
print(r.zrank("leaderboard", "张三"))    # 2（0-based）
```text

### 4.3 Pipeline（管道）

Pipeline 将多个命令打包成一次网络往返，**大幅提升性能**。

```python
# 普通模式（N 条命令 = N 次网络往返）
for i in range(1000):
    r.set(f"key:{i}", i)

# Pipeline 模式（1 次网络往返）
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f"key:{i}", i)
pipe.execute()  # 一次发送所有命令
```

**Pipeline vs 事务：**

```python
# 默认 pipeline：不保证原子性（仅仅是批量发送）
pipe = r.pipeline()

# 事务 pipeline：保证原子性（内部使用 MULTI/EXEC）
pipe = r.pipeline(transaction=True)
pipe.set("a", 1)
pipe.incr("b")
result = pipe.execute()  # [True, 2]
```text

### 4.4 异步 Redis

```python
from redis.asyncio import Redis as AsyncRedis

async def demo():
    r = AsyncRedis(
        host="localhost",
        port=6379,
        db=0,
        decode_responses=True,
    )

    await r.set("name", "张三")
    name = await r.get("name")
    print(name)

    # Pipeline 也支持异步
    async with r.pipeline(transaction=True) as pipe:
        await pipe.set("a", 1)
        await pipe.incr("b")
        result = await pipe.execute()

    await r.aclose()  # 关闭连接
```

### 4.5 连接池管理

```python
from redis.asyncio import Redis as AsyncRedis, ConnectionPool

# 共享连接池
pool = ConnectionPool.from_url(
    "redis://localhost:6379/0",
    max_connections=20,
    socket_timeout=5,
    socket_connect_timeout=3,
    health_check_interval=30,
    decode_responses=True,
)

async def get_redis():
    async with AsyncRedis(connection_pool=pool) as r:
        yield r
```text

---

## 5. 缓存模式详解

### 5.1 Cache-Aside（旁路缓存）

最常用的缓存模式：

```

读取流程：

1. 读 Redis → 命中则直接返回
2. 未命中 → 读 MySQL
3. 将数据写入 Redis（设置 TTL）
4. 返回数据

写入流程：

1. 更新 MySQL
2. 删除 Redis 中的缓存（不是更新缓存！）

```text

```python
async def get_event(event_id: int, redis: AsyncRedis, session: AsyncSession):
    # 1. 读缓存
    cache_key = f"event:{event_id}"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. 缓存未命中，读数据库
    event = await session.get(Event, event_id)
    if not event:
        return None

    # 3. 写入缓存（5 分钟 TTL）
    await redis.setex(cache_key, 300, event.model_dump_json())
    return event

async def update_event(event_id: int, data: dict, redis: AsyncRedis, session: AsyncSession):
    # 1. 更新数据库
    event = await session.get(Event, event_id)
    for k, v in data.items():
        setattr(event, k, v)
    await session.flush()

    # 2. 删除缓存（不是更新！）
    await redis.delete(f"event:{event_id}")

    return event
```

### 5.2 缓存穿透（Cache Penetration）

**问题：** 请求不存在的数据（如 id=-1），缓存永远不命中，直接打到数据库。

**解决方案：**

```python
# 方案一：缓存空值
cache_key = f"event:{event_id}"
cached = await redis.get(cache_key)
if cached == "NULL":
    return None  # 缓存层拦截

event = await get_event(event_id)
if not event:
    await redis.setex(cache_key, 60, "NULL")  # 缓存空值，60秒过期
    return None

# 方案二：布隆过滤器（Bloom Filter）
from redis_bloom import Client

bf = Client(redis)
bf.bfAdd("events", str(event_id))  # 存在性检查
if not bf.bfExists("events", str(event_id)):
    return None  # 一定不存在
```text

### 5.3 缓存击穿（Cache Breakdown）

**问题：** 过期瞬间大量并发请求同时打到数据库。

**解决方案：**

```python
import asyncio

async def get_event_with_mutex(event_id: int, redis: AsyncRedis, session: AsyncSession):
    """互斥锁防止缓存击穿"""
    cache_key = f"event:{event_id}"
    lock_key = f"lock:event:{event_id}"

    # 1. 读缓存
    cached = await redis.get(cache_key)
    if cached:
        if cached == "NULL":
            return None
        return json.loads(cached)

    # 2. 获取分布式锁（只有一个请求能重建缓存）
    locked = await redis.setnx(lock_key, "1")
    if locked:
        await redis.expire(lock_key, 10)  # 锁自动过期

        try:
            # 重建缓存
            event = await session.get(Event, event_id)
            if event:
                await redis.setex(cache_key, 300, event.model_dump_json())
            else:
                await redis.setex(cache_key, 60, "NULL")
            return event
        finally:
            await redis.delete(lock_key)
    else:
        # 3. 没抢到锁，等待
        await asyncio.sleep(0.1)
        return await get_event_with_mutex(event_id, redis, session)
```

### 5.4 缓存雪崩（Cache Avalanche）

**问题：** 大量缓存同时过期，所有请求同时打到数据库。

**解决方案：**

```python
# 方案一：过期时间加随机偏移
import random

base_ttl = 300  # 5分钟
random_ttl = base_ttl + random.randint(0, 60)  # 5分钟 ± 0~60秒
await redis.setex(cache_key, random_ttl, value)

# 方案二：多级缓存
# L1：本地缓存（如 functools.lru_cache）
# L2：Redis
# L3：MySQL

# 方案三：缓存永不失效 + 后台更新
from functools import lru_cache

@lru_cache(maxsize=128)
def get_event_sync(event_id: int):
    """本地缓存（极短生命周期）"""
    ...

# 后台线程定期刷新缓存而不是等到过期
```text

### 5.5 淘汰策略

```bash
# Redis 的内存淘汰策略（maxmemory-policy）

# 常见策略：
noeviction        # 不淘汰，写入返回错误（默认）
allkeys-lru       # 最近最少使用（推荐）
allkeys-lfu       # 最不经常使用（8.0+）
volatile-lru      # 仅对设置了 TTL 的键执行 LRU
allkeys-random    # 随机淘汰

# 查看和设置
CONFIG GET maxmemory-policy
CONFIG SET maxmemory-policy allkeys-lru
```

---

## 6. 防超卖核心实现

这是本章最重要的部分，实现两层防御机制。

### 6.1 问题分析

**超卖问题定义：**

```text
库存 = 10 张票
用户 A 和用户 B 同时买票

朴素逻辑（一定有 bug）：
1. A 查询库存 = 10，剩余 > 0，可以买
2. B 查询库存 = 10，剩余 > 0，可以买  ← 并发的瞬间！
3. A 减库存 = 9
4. B 减库存 = 9  ← 超卖了！（卖出了 2 张，库存只减了 1）

根本原因：读-改-写 不是原子的。
```

**解决方案架构：**

```text
请求 → API 网关 → 速率限制 (Redis 滑动窗口)
                         ↓
                   Redis Lua 快速门卫 (原子扣减，~0.1ms)
                         ↓
                   MySQL 乐观锁最终审计 (version 列)
                         ↓
                   失败补偿 (Redis INCRBY 恢复库存)
```

### 6.2 初始化库存

```python
async def init_inventory(event_id: int, total_stock: int, redis: AsyncRedis):
    """活动发布时初始化 Redis 库存"""
    stock_key = f"stock:{event_id}"
    await redis.set(stock_key, total_stock)
```text

### 6.3 防超卖 Lua 脚本

> ⚠️ **项目实际情况：** 本项目中该 Lua 脚本以内联字符串 `INVENTORY_LUA_SCRIPT` 的形式定义在 `app/services/inventory_service.py` 中，而非外部文件。这样做的原因是：外部的 `.lua` 文件在 Docker 部署时可能出现路径问题。以下代码在概念上等价，但实际项目使用 Python 内联字符串 + `script_load` API。

**app/services/inventory_service.py（内联 Lua 脚本）：**

```lua
-- 原子库存扣减脚本
-- KEYS[1]: 活动库存键 (stock:{event_id})
-- KEYS[2]: 用户已购键 (user_bought:{event_id}:{user_id})  -- 限购检查
-- ARGV[1]: 扣减数量
-- ARGV[2]: 用户最大限购数量
-- ARGV[3]: 用户ID（用于限购检查）

-- 返回值：
--  1: 成功
--  0: 库存不足
-- -1: 超过限购数量
-- -2: 库存键不存在

local stock_key = KEYS[1]
local limit_key = KEYS[2]
local quantity = tonumber(ARGV[1])
local max_per_user = tonumber(ARGV[2])
local user_id = ARGV[3]

-- 1. 检查库存键是否存在
local stock = redis.call("GET", stock_key)
if not stock then
    return -2  -- 库存键不存在
end

-- 2. 检查限购
if max_per_user > 0 then
    local bought = redis.call("GET", limit_key)
    if bought then
        if tonumber(bought) + quantity > max_per_user then
            return -1  -- 超过限购
        end
    end
end

-- 3. 检查库存是否充足
if tonumber(stock) >= quantity then
    -- 4. 原子扣减库存
    redis.call("DECRBY", stock_key, quantity)

    -- 5. 记录用户已购数量（带过期时间）
    if max_per_user > 0 then
        redis.call("INCRBY", limit_key, quantity)
        redis.call("EXPIRE", limit_key, 86400)  -- 24小时过期
    end

    return 1  -- 成功
end

return 0  -- 库存不足
```

**Python 调用（实际项目代码 `app/services/inventory_service.py`）：**

```python
INVENTORY_LUA_SCRIPT = """
-- 原子库存扣减脚本
-- ...（同上方的 Lua 代码）
"""

class InventoryService:
    """库存服务"""

    def __init__(self, redis: AsyncRedis):
        self.redis = redis
        self._sha: Optional[str] = None   # 缓存 Lua 脚本 SHA

    async def load_script(self):
        """将 Lua 脚本加载到 Redis，获取 SHA 摘要"""
        if not self._sha:
            self._sha = await self.redis.script_load(INVENTORY_LUA_SCRIPT)

    def _stock_key(self, event_id: int) -> str:
        return f"stock:{event_id}"

    def _limit_key(self, event_id: int, user_id: int) -> str:
        return f"user_bought:{event_id}:{user_id}"

    async def deduct_stock(
        self, event_id: int, user_id: int, quantity: int, max_per_user: int = 0
    ) -> int:
        """
        原子扣减库存
        返回: 1=成功, 0=库存不足, -1=超过限购, -2=库存键不存在

        安全要点：
        - quantity 必须为正整数（防止负数导致库存增加）
        """
        if quantity <= 0:
            raise ValueError("扣减数量必须为正数")

        await self.load_script()
        keys = [self._stock_key(event_id), self._limit_key(event_id, user_id)]
        args = [str(quantity), str(max_per_user), str(user_id)]

        try:
            return await self.redis.evalsha(self._sha, len(keys), *keys, *args)
        except Exception:
            # Redis 重启导致脚本丢失，回退到 EVAL
            return await self.redis.eval(
                INVENTORY_LUA_SCRIPT, len(keys), *keys, *args
            )

    async def restore_stock(self, event_id: int, quantity: int):
        """
        归还库存（取消订单/支付超时）

        安全要点：
        - 通过 Redis watch + 上限校验防止超量归还
        """
        if quantity <= 0:
            return  # 忽略无效归还

        key = self._stock_key(event_id)
        try:
            async with self.redis.pipeline(transaction=True) as pipe:
                await pipe.watch(key)
                current = await pipe.get(key)
                current_val = int(current) if current else 0
                new_val = current_val + quantity
                await pipe.multi()
                await pipe.set(key, new_val)
                await pipe.execute()
        except Exception:
            # watch 冲突时，回退到简单 incrby
            await self.redis.incrby(key, quantity)

    async def get_stock(self, event_id: int) -> Optional[int]:
        stock = await self.redis.get(self._stock_key(event_id))
        return int(stock) if stock is not None else None

    async def init_stock(self, event_id: int, total: int):
        """初始化缓存库存"""
        await self.redis.set(self._stock_key(event_id), total)
```text

### 6.4 EVALSHA 回退机制

```python
# 重要：Redis 重启后脚本缓存丢失，需要 EVALSHA → EVAL 回退

async def safe_evalsha(redis: AsyncRedis, sha: str, keys: list, args: list):
    """安全的 EVALSHA（自动回退到 EVAL）"""
    try:
        return await redis.evalsha(sha, len(keys), *keys, *args)
    except redis.exceptions.NoScriptError:
        # Redis 重启了，重新加载脚本
        return None  # 告知调用者重新加载
```

### 6.5 第二层防御：MySQL 乐观锁

即使 Redis 出错了（如宕机、数据丢失），MySQL 乐观锁作为最终防线。

```sql
-- orders 表增加 version 列
ALTER TABLE orders ADD COLUMN version INT NOT NULL DEFAULT 1;
```text

```python
async def confirm_order_with_optimistic_lock(
    session: AsyncSession,
    event_id: int,
    user_id: int,
    quantity: int,
) -> Order:
    """使用乐观锁创建订单（MySQL 最终审计）"""

    # 读取活动和版本号
    event = await session.get(Event, event_id)

    # 检查库存（MySQL 层面再确认一次）
    if event.sold_count + quantity > event.total_stock:
        raise HTTPException(status_code=409, detail="库存不足")

    # 创建订单
    order = Order(
        order_no=generate_order_no(),
        user_id=user_id,
        event_id=event_id,
        quantity=quantity,
        total_amount=event.price * quantity,
        status="paid",
        version=1,
    )
    session.add(order)
    await session.flush()

    # 乐观锁更新：version 条件保证原子性
    result = await session.execute(
        update(Event)
        .where(
            Event.id == event_id,
            Event.version == event.version,  # 检查版本号
            Event.sold_count + quantity <= Event.total_stock,  # 最终检查
        )
        .values(
            sold_count=Event.sold_count + quantity,
            version=Event.version + 1,
        )
    )

    if result.rowcount == 0:
        # 乐观锁冲突！回滚订单
        await session.rollback()
        raise HTTPException(status_code=409, detail="库存已更新，请重试")

    return order
```

### 6.6 完整下单流程

```python
async def create_order(
    event_id: int,
    user_id: int,
    quantity: int,
    redis: AsyncRedis,
    session: AsyncSession,
):
    """完整下单流程：Redis 快速门卫 → MySQL 最终审计"""

    # 第一层：Redis Lua 快速检查（~0.1ms）
    inventory_service = InventoryService(redis)
    result = await inventory_service.deduct_stock(
        event_id=event_id,
        user_id=user_id,
        quantity=quantity,
        max_per_user=4,  # 每人限购 4 张
    )

    if result <= 0:
        error_map = {
            0: "库存不足",
            -1: "超过限购数量",
            -2: "活动不存在",
        }
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=error_map.get(result, "下单失败"),
        )

    try:
        # 第二层：MySQL 乐观锁最终审计
        order = await confirm_order_with_optimistic_lock(
            session, event_id, user_id, quantity
        )

        # 设置 Redis 分布式锁防止重复处理
        lock_key = f"order_lock:{order.order_no}"
        await redis.setex(lock_key, 10, "processing")

        return order

    except HTTPException:
        # MySQL 层面失败，恢复 Redis 库存
        await inventory_service.restore_stock(event_id, quantity)
        raise

    except Exception:
        # 未知异常，恢复 Redis 库存
        await inventory_service.restore_stock(event_id, quantity)
        raise
```text

### 6.7 并发测试验证

```python
import asyncio

async def test_anti_overselling():
    """并发测试：100 个用户抢 10 张票，只有 10 个应该成功"""
    success_count = 0
    fail_count = 0

    async def buy_ticket(user_id: int):
        nonlocal success_count, fail_count
        try:
            await create_order(
                event_id=1,
                user_id=user_id,
                quantity=1,
                redis=redis,
                session=session,
            )
            success_count += 1
        except Exception:
            fail_count += 1

    # 并发 100 个请求
    tasks = [buy_ticket(i) for i in range(100)]
    await asyncio.gather(*tasks)

    print(f"成功: {success_count}")  # 应该 ≤ 10
    print(f"失败: {fail_count}")     # 应该 ≥ 90

    # 验证
    assert success_count <= 10, f"超卖！成功 {success_count} > 10"
    assert success_count + fail_count == 100
```

---

## 7. 速率限制实现

### 7.1 滑动窗口限流（Sorted Set 实现）

```python
import time

class RateLimiter:
    """滑动窗口速率限制器"""

    def __init__(self, redis: AsyncRedis):
        self.redis = redis

    async def is_allowed(
        self,
        key: str,
        max_requests: int,
        window_seconds: int = 60,
    ) -> bool:
        """
        检查请求是否允许通过
        key: 限流键（如 rate:user:1, rate:ip:192.168.1.1）
        max_requests: 窗口内最大请求数
        window_seconds: 窗口大小（秒）
        """
        now = time.time()
        window_start = now - window_seconds

        # 使用 ZSet 实现滑动窗口
        # 每个请求的时间戳作为 score 和 member

        # 1. 移除窗口外的旧记录
        await self.redis.zremrangebyscore(key, 0, window_start)

        # 2. 获取当前窗口内的请求数
        count = await self.redis.zcard(key)

        if count >= max_requests:
            return False

        # 3. 添加当前请求
        await self.redis.zadd(key, {str(now): now})
        await self.redis.expire(key, window_seconds)  # 自动清理

        return True

    async def get_remaining(self, key: str, max_requests: int, window_seconds: int = 60) -> int:
        """获取剩余可用次数"""
        now = time.time()
        window_start = now - window_seconds
        await self.redis.zremrangebyscore(key, 0, window_start)
        count = await self.redis.zcard(key)
        return max(0, max_requests - count)
```text

### 7.2 固定窗口限流（INCR + EXPIRE）

```python
async def fixed_window_rate_limit(
    redis: AsyncRedis,
    key: str,
    max_requests: int,
    window_seconds: int = 60,
) -> bool:
    """固定窗口限流（简单，但有边界效应）"""
    # INCR 原子递增
    count = await redis.incr(key)

    if count == 1:
        # 第一次访问，设置过期时间
        await redis.expire(key, window_seconds)

    return count <= max_requests
```

### 7.3 令牌桶算法

```python
import asyncio

class TokenBucket:
    """令牌桶限流器"""

    def __init__(self, redis: AsyncRedis):
        self.redis = redis

    async def allow_request(
        self,
        key: str,
        capacity: int = 100,
        refill_rate: float = 10.0,  # 每秒补充令牌数
    ) -> bool:
        """
        令牌桶算法
        key: 桶键
        capacity: 桶容量
        refill_rate: 补充速率（个/秒）
        """
        now = time.time()
        bucket_key = f"token_bucket:{key}"
        time_key = f"token_bucket_time:{key}"

        # Lua 脚本：原子化令牌桶操作
        script = """
        local bucket_key = KEYS[1]
        local time_key = KEYS[2]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        -- 获取当前令牌数
        local tokens = redis.call("GET", bucket_key)
        if not tokens then
            tokens = capacity
        else
            tokens = tonumber(tokens)
        end

        -- 获取上次补充时间
        local last_refill = redis.call("GET", time_key)
        if not last_refill then
            last_refill = now
        else
            last_refill = tonumber(last_refill)
        end

        -- 计算需要补充的令牌
        local elapsed = now - last_refill
        local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

        if new_tokens >= 1 then
            redis.call("SET", bucket_key, new_tokens - 1)
            redis.call("SET", time_key, now)
            return 1
        else
            redis.call("SET", bucket_key, new_tokens)
            redis.call("SET", time_key, now)
            return 0
        end
        """

        result = await self.redis.eval(
            script, 2, bucket_key, time_key, capacity, refill_rate, now
        )
        return result == 1
```text

---

## 8. 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `EVALSHA` 返回 `NOSCRIPT` | Redis 重启后脚本丢失 | 实现 EVALSHA→EVAL 回退 |
| Redis 连接超时 | 防火墙/Docker 未就绪 | `docker ps` 检查 `ticket-redis` |
| 库存扣减为负数 | Redis 未正确初始化 | 检查发布活动时是否 SET stock |
| Lua 脚本返回 `ERR` | 脚本语法错误 | 用 `redis-cli --eval` 单独测试 |
| 限流误伤正常用户 | 窗口边界效应 | 用滑动窗口替代固定窗口 |
| 超卖验证仍失败 | 只用了单层防御 | 必须同时用 Redis + MySQL 两层 |
| Redis 数据丢失 | 未配置持久化 | 启用 RDB + AOF 组合 |

---

## 9. 本章总结

✅ **已掌握：**

- Redis 7.x 全部数据结构：String/List/Set/Hash/ZSet/Stream 的命令和内部编码
- Lua 5.1 完全语法：Table/函数/闭包/元表/字符串库
- Redis Lua 脚本的原子性数学证明
- redis-py 完全使用：基础操作/Pipeline/连接池
- 缓存三大问题（穿透/击穿/雪崩）及其解决方案
- **防超卖核心：两层防御（Redis Lua + MySQL 乐观锁）**
- 三种限流算法：滑动窗口/固定窗口/令牌桶

✅ **项目里程碑：**

- [ ] Redis 库存初始化
- [ ] Lua 原子扣减脚本加载成功
- [ ] 两层防御（Redis + MySQL）完整链路过关
- [ ] 并发测试 100 抢 10 只成功 10 次（不超卖）
- [ ] 限流器生效（429）
- [ ] EVALSHA 回退机制正常

**下一章预告：** 第7章将实现 JWT 认证和 OAuth2 授权系统，包含用户注册/登录/令牌刷新和 RBAC 权限控制。

**练习：**

1. 在 Redis CLI 中手动测试 Lua 脚本的每个分支
2. 运行并发测试验证不超卖
3. 实现库存恢复的 Lua 脚本（原子化恢复）
4. 为不同 API 配置不同的速率限制策略
5. 实现布隆过滤器防止缓存穿透

---

## 10. 原理深入（补充）

### 10.1 防超卖两层防御的数学概率证明

#### 问题建模

```

定义：
  N = 总库存
  M = 并发请求数（M >> N）
  P_oversell = 系统发生超卖的概率（应趋近于 0）

假设：
  第1层（Redis Lua）失效概率 = p₁（如 Redis 宕机、键丢失等）
  第2层（MySQL 乐观锁）失效概率 = p₂（如数据库宕机等）

  每层在正常运行时的防御成功率 = 100%
  层间失效独立（Redis 和 MySQL 部署在不同的服务器上）

两层独立防御的综合失效概率 = p₁ × p₂

实际数值估算：
  Redis 可用性（有主从）：99.99% → p₁ = 0.0001
  MySQL 可用性（有主从）：99.995% → p₂ = 0.00005

  综合失效概率 = 0.0001 × 0.00005 = 5 × 10⁻⁹

  即：每 2 亿次下单操作中，理论超卖不到 1 次。

```text

#### 并发场景下的严格证明

```

场景：N = 10 张票，M = 100 个并发请求

第1层 - Redis Lua 原子扣减：

  由于 Lua 脚本是原子的（单线程执行），
  100 个请求顺序执行 Lua 脚本，每个请求执行 DECRBY。

  假设库存初始值为 N：
  第 i 个执行 DECRBY 的请求：
    if stock >= quantity → DECRBY → 成功
    if stock < quantity → 返回库存不足

  由于 DECRBY 是原子的，不存在读-改-写竞态。
  在前 N 个成功请求后，库存降为 0。
  第 N+1 个请求发现库存不足 → 拒绝。

  因此：第1层单独保证「不超卖」的概率 = 100%
  ∵ Lua 脚本在 Redis 单线程中原子执行，不可能出现交错

第2层 - MySQL 乐观锁：

  假设第1层失效（如 Redis 宕机），N 个请求直接到达 MySQL。

  MySQL 乐观锁：
  UPDATE events
  SET sold_count = sold_count + qty, version = version + 1
  WHERE id = :id
    AND sold_count + qty <= total_stock
    AND version = :ver

  这行 UPDATE 在 InnoDB 中通过行锁 + 条件判断保证原子性。
  对于每行数据，同一时间只有一个事务能成功执行该 UPDATE。

  假设 N 个并发事务：
  事务1: UPDATE ... WHERE sold_count + 1 <= total_stock → 成功 (sold_count=1)
  事务2: UPDATE ... WHERE sold_count + 1 <= total_stock → 成功 (sold_count=2)
  ...
  事务N: UPDATE ... WHERE sold_count + 1 <= total_stock → 成功 (sold_count=N)
  事务N+1: UPDATE ... WHERE sold_count + 1 <= total_stock → 失败 (rowcount=0)

  因此：第2层单独保证「不超卖」的概率 = 100%
  ∵ 行锁 + 条件判断保证同一时刻只有满足库存条件的事务能成功

综合：
  两层独立运行，每层单独保证 100% 不超卖。
  两层同时失效 = 两层同时宕机（概率 = p₁ × p₂ ≈ 5×10⁻⁹）

Q.E.D.

```text

### 10.2 Redis Lua 原子性深入

#### 原子性的三个层面

```

层面1：命令级原子性
  INCR、DECR、GETSET 等单个 Redis 命令是原子的。
  因为 Redis 单线程执行，一个命令不会被其他命令打断。

层面2：事务级原子性
  MULTI/EXEC 块中的命令序列整体入队、整体执行。
  但 EXEC 之前其他客户端可以执行命令（队列期间不阻塞）。

层面3：Lua 脚本级原子性
  EVAL "script" 整个脚本在同一个事件循环 tick 中执行。
  脚本执行期间，Redis 不处理任何其他命令。
  这是最强的原子性保证（等同于单线程下的串行化）。

```text

#### Lua 脚本原子性的实际含义

```

重要区分：

正确理解：
  Lua 脚本是原子的 → 脚本执行期间不会被其他命令插入
  → 等价于数据库的 SERIALIZABLE 隔离级别

错误理解：
  Lua 脚本是原子的 → 脚本要么成功要么回滚
  → Redis Lua 不支持回滚！
  → redis.call() 失败会抛出异常，脚本终止，但已执行的命令不会撤销

示例：
  redis.call("DECRBY", "stock", 10)   -- 成功扣减
  redis.call("LPUSH", "stock", "x")   -- 类型错误！抛出异常
  -- 脚本终止，但 DECRBY 已经执行了，不会回滚！

在防超卖脚本中，我们的设计规避了此问题：

- 先检查条件 → 满足再操作
- 操作命令都是针对 String 类型（不会出现类型错误）
- 失败返回错误码而非抛出异常

```text

#### 原子性的性能含义

```

因为 Lua 脚本是原子的，脚本执行期间 Redis 不处理其他命令：

  好的一面：

- 防超卖严格安全
- 无需额外的锁机制

  坏的一面：

- 脚本应尽量短（毫秒级）
- 过长的脚本会阻塞 Redis 事件循环
- 不应在脚本中执行耗时操作（如循环大量数据）

  Redis 官方建议：Lua 脚本执行时间不应超过几毫秒。
  如果脚本执行超过 5 秒，Redis 会自动 kill 并报错。

我们的防超卖脚本执行时间 ≈ 0.1ms，完全在安全范围内。

```text

### 10.3 乐观锁 vs 悲观锁

#### 对比总览

| 特性 | 乐观锁（Optimistic Lock） | 悲观锁（Pessimistic Lock） |
| ------ | -------------------------- | --------------------------- |
| 原理 | 假设不会冲突，更新时检查 | 假设会冲突，操作前先加锁 |
| 实现方式 | version 列 / CAS | SELECT ... FOR UPDATE |
| 并发性能 | 高（无锁开销） | 低（锁等待） |
| 冲突时 | 重试或放弃 | 排队等待 |
| 适用场景 | 读多写少 | 写多读少 |
| 死锁风险 | 无 | 有 |
| MySQL 实现 | UPDATE ... WHERE version = :v | FOR UPDATE |
| 本系统使用 | 库存扣减、优惠券领取 | 无（未使用） |

#### 为什么防超卖选乐观锁

```

本系统的库存扣减是典型的"读多写少"场景：

- 一个活动有 1000 张票
- 1000 个请求抢购
- 成功写入的只有 1000 个，其余都是读操作

如果使用悲观锁：
  每个请求都要执行一次 FOR UPDATE，
  未抢到的请求也在排队等待锁释放。
  大量连接被占用，连接池很快耗尽，
  用户体验变差（长时间等待）。

如果使用乐观锁：
  所有请求同时尝试 UPDATE。
  大多数请求的 version 不匹配，立即失败返回。
  不需要排队等待，不占用连接。
  失败者立即收到响应（建议重试）。

使用 Redis Lua 作为第一层后：
  绝大部分并发请求在 Redis 层就被拒绝，
  到达 MySQL 乐观锁的请求量大大减少。
  乐观锁的冲突率降低，性能更好。

```text

#### 乐观锁在 MySQL 中的执行计划

```sql
-- 不含乐观锁（可能超卖）
UPDATE events
SET sold_count = sold_count + 1
WHERE id = 1;
-- 这是不安全的！两个并发事务都能成功

-- 含乐观锁（安全）
UPDATE events
SET sold_count = sold_count + 1, version = version + 1
WHERE id = 1
  AND sold_count + 1 <= total_stock  -- 库存检查
  AND version = :current_version;     -- 版本检查
-- EXPLAIN: type=eq_ref, key=PRIMARY
-- 影响行数：1（成功）或 0（冲突）

-- 含乐观锁 + 索引（更高效）
-- 如果经常按 event_id 查询，加复合索引：
ALTER TABLE events ADD INDEX idx_ver (id, version);
```

#### 乐观锁的 ABA 问题

```text
乐观锁的典型陷阱：ABA 问题

场景：
  线程1 读取 version = 1
  线程2 读取 version = 1，更新完成 (version → 2)
  线程3 读取 version = 2，更新完成 (version → 1)  -- 某业务场景下 version 可能回绕
  线程1 检查 version = 1，认为数据未修改 → 错误更新！

本系统为什么不受影响：
  1. version 严格递增（version + 1），不回绕
  2. 行锁保证同一时刻只有一个事务能修改该行
  3. WHERE 条件中包含库存检查 (sold_count + qty <= total_stock)
     version 即使一致，库存条件也阻断超卖

  因此本系统的乐观锁是安全的，不受 ABA 问题影响。
```
