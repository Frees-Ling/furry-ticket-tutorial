# 第15章：性能优化 + 架构回顾 + 上线清单

## 本章学习目标

本章将对票务系统进行全面性能优化，从 SQL 查询优化到 Python 代码分析，从缓存层次架构到生产上线清单。完成本章后，你将掌握：

1. SQL 执行计划分析和慢查询优化
2. N+1 查询问题检测和解决
3. 连接池调优策略
4. Python 性能分析工具（cProfile / py-spy / snakeviz）
5. 四级缓存层次架构设计
6. B+Tree 索引原理的数学证明
7. 20+ 项生产上线检查清单

---

## 1. SQL 优化

### 1.1 EXPLAIN ANALYZE 详解

MySQL 8.0.18+ 引入了 `EXPLAIN ANALYZE`，它不仅能显示查询计划，还能输出每个步骤的实际执行时间和行数。这比传统的 `EXPLAIN FORMAT=JSON` 更有价值。

```sql
-- 基础用法
EXPLAIN ANALYZE
SELECT o.id, o.order_no, o.total_amount, o.status, e.title
FROM orders o
JOIN events e ON o.event_id = e.id
WHERE o.user_id = 1001
ORDER BY o.created_at DESC
LIMIT 20;
```text

**输出示例及逐段解读：**

```

-> Limit: 20 rows  (actual time=0.456..0.468 rows=20 loops=1)
    -> Sort: o.created_at DESC;  (actual time=0.455..0.459 rows=20 loops=1)
        -> Stream results  (actual time=0.361..0.438 rows=20 loops=1)
            -> Nested loop inner join  (actual time=0.320..0.410 rows=20 loops=1)
                -> Index lookup on o using idx_user_id (user_id=1001)  (actual time=0.150..0.180 rows=45 loops=1)
                -> Single-row index lookup on e using PRIMARY (id=o.event_id)  (actual time=0.005..0.005 rows=1 loops=45)

```text

**解读关键字段：**

| 字段 | 含义 | 判断标准 |
| ------ | ------ | ---------- |
| `actual time` | 实际执行时间（第一个值是平均，第二个是最大） | 越小越好 |
| `rows` | 实际返回的行数 | 接近预期则为好 |
| `loops` | 该步骤的执行次数 | loops=1 理想，loops > 1 说明被多次调用 |
| `Nested loop` | 嵌套循环连接 | 外层驱动，内层匹配 |
| `Index lookup` | 使用索引查找 | 好！避免全表扫描 |
| `Full table scan` | 全表扫描 | 危险信号，通常要优化 |
| `Using temporary` | 使用了临时表 | 应避免 |
| `Using filesort` | 使用了文件排序 | 应尽量优化 |

**常见查询类型及执行计划解读：**

```sql
-- === 类型1：全表扫描（坏） ===
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending_payment';
-- Extra: Using where
-- type: ALL (全表扫描)
-- rows: 100000 (预估扫描所有行)

-- === 类型2：索引范围扫描（好） ===
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1001;
-- type: ref (使用非唯一索引)
-- possible_keys: idx_user_id
-- rows: 45 (只扫描需要的行)

-- === 类型3：等值引用（最优） ===
EXPLAIN ANALYZE SELECT * FROM orders WHERE order_no = '20240501123456';
-- type: const (使用唯一索引)
-- rows: 1 (精确命中一行)
```

### 1.2 EXPLAIN 输出列详解

传统 `EXPLAIN` 的输出包含以下关键列：

```sql
EXPLAIN SELECT o.*, e.title
FROM orders o
LEFT JOIN events e ON o.event_id = e.id
WHERE o.total_amount > 100
  AND o.status = 'paid'
ORDER BY o.created_at DESC
LIMIT 10;
```text

| 列名 | 值举例 | 说明 |
| ------ | -------- | ------ |
| `id` | 1 | SELECT 的标识符，id 越大越先执行 |
| `select_type` | SIMPLE | `SIMPLE`=简单查询, `PRIMARY`=主查询, `SUBQUERY`=子查询, `DERIVED`=派生表 |
| `table` | o / e | 当前访问的表 |
| `type` | ALL / ref / range / const | 访问类型，从好到差：`const>eq_ref>ref>range>index>ALL` |
| `possible_keys` | idx_status | 可能使用的索引 |
| `key` | idx_status | 实际使用的索引 |
| `key_len` | 82 | 使用的索引长度，越短越好 |
| `rows` | 45678 | 预估扫描行数 |
| `filtered` | 10.00 | 经过 WHERE 过滤后剩余行的百分比 |
| `Extra` | Using index; Using filesort | 额外信息 |

**Extra 字段重要取值：**

| Extra 值 | 说明 | 建议 |
| ---------- | ------ | ------ |
| `Using index` | 覆盖索引（只查询索引列） | 最优 |
| `Using where` | 使用 WHERE 过滤 | 正常 |
| `Using temporary` | 使用临时表 | 应避免（GROUP BY 无索引） |
| `Using filesort` | 使用文件排序 | 应避免（ORDER BY 无索引） |
| `Using index condition` | 索引下推（ICP） | MySQL 5.6+ 优化 |
| `Using join buffer` | 使用连接缓存 | 添加索引优化 |
| `Impossible WHERE` | WHERE 条件永远不成立 | 检查 SQL 逻辑 |

### 1.3 慢查询日志配置

慢查询日志记录了执行时间超过阈值的 SQL 语句，是性能优化的第一道防线。

```mysql
-- === 运行时配置（重启失效） ===
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;          -- 超过 500ms 记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询
SET GLOBAL log_output = 'TABLE';           -- 记录到 mysql.slow_log 表

-- === 配置文件配置（永久生效） ===
-- 在 my.cnf / my.ini 的 [mysqld] 段添加：
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 0.5
log_queries_not_using_indexes = 1
log_output = FILE

-- === 查看慢查询设置 ===
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'log_queries_not_using_indexes';

-- === 从表中查询慢查询记录 ===
SELECT * FROM mysql.slow_log
ORDER BY start_time DESC
LIMIT 10\G

-- === 使用 mysqldumpslow 分析慢查询日志 ===
-- 按平均查询时间排序，查看前 10 条
mysqldumpslow -t 10 /var/log/mysql/mysql-slow.log

-- 按执行次数排序
mysqldumpslow -s c /var/log/mysql/mysql-slow.log

-- 汇总相似查询（不显示具体参数值）
mysqldumpslow -g 'SELECT.*orders' /var/log/mysql/mysql-slow.log
```

**mysqldumpslow 输出示例：**

```text
Count: 123  Time=2.34s (287s)  Lock=0.00s (0s)  Rows=45.0 (5535)
  SELECT o.*, e.title FROM orders o
  JOIN events e ON o.event_id = e.id
  WHERE o.status = 'S' AND o.total_amount > N
  ORDER BY o.created_at DESC
  LIMIT N

-- 解读：
-- Count: 123        → 该查询被执行了 123 次
-- Time=2.34s (287s) → 平均 2.34 秒，总共 287 秒
-- Rows=45.0 (5535)  → 平均返回 45 行，总共返回 5535 行
```

### 1.4 索引优化

#### 复合索引列顺序原则

复合索引（多列索引）的列顺序至关重要。它遵循**最左前缀原则**。

```sql
-- === 错误的列顺序 ===
CREATE INDEX idx_bad ON orders (status, created_at, user_id);
-- 以下查询可以用到这个索引：
WHERE status = 'paid'                     -- 可以用（命中第一列）
WHERE status = 'paid' AND created_at > '2024-01-01'  -- 可以用（命中前两列）
WHERE status = 'paid' AND user_id = 1001  -- 可以用第一列
-- 以下查询不可用：
WHERE created_at > '2024-01-01'           -- 不能用（跳过了第一列）

-- === 正确的列顺序（高选择性在前） ===
CREATE INDEX idx_optimal ON orders (user_id, status, created_at);
-- user_id 选择性高（每个用户的订单数少），放在最前面
-- status 选择性中等
-- created_at 用于排序

-- === 选择性计算 ===
-- 计算列的选择性（值越分散，选择性越高）
SELECT
    COUNT(DISTINCT user_id) AS user_cardinality,
    COUNT(DISTINCT status) AS status_cardinality,
    COUNT(*) AS total_rows,
    COUNT(DISTINCT user_id) / COUNT(*) AS user_selectivity,
    COUNT(DISTINCT status) / COUNT(*) AS status_selectivity
FROM orders;
```text

#### 覆盖索引（Covering Index）

当索引包含了查询所需的所有列时，MySQL 只需扫描索引而不需要回表查询数据行。

```sql
-- === 非覆盖索引（需要回表） ===
-- 索引：idx_user_id (user_id)
EXPLAIN SELECT * FROM orders WHERE user_id = 1001;
-- Extra: NULL  → 需要从索引找到 row_id，再回主键查数据

-- === 覆盖索引（无需回表） ===
-- 索引：idx_user_id_covering (user_id, status, total_amount, created_at)
EXPLAIN SELECT user_id, status, total_amount, created_at
FROM orders
WHERE user_id = 1001
ORDER BY created_at DESC;
-- Extra: Using index; Using index condition
-- 所有数据都在索引中，无需回表

-- === 验证覆盖索引 ===
EXPLAIN FORMAT=JSON
SELECT user_id, status, total_amount
FROM orders
WHERE user_id = 1001\G
-- 输出中查找 "using_index": true
```

#### 索引设计检查清单

```sql
-- 检查查询是否使用索引
EXPLAIN SELECT ...;

-- 查找未使用索引的查询（通过慢查询日志）
-- 查看索引使用情况
SHOW INDEX FROM orders;

-- 监控索引使用统计
SELECT
    INDEX_NAME,
    COUNT_STAR,
    COUNT_READ,
    COUNT_FETCH
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'ticket_dev'
  AND OBJECT_NAME = 'orders';
```text

### 1.5 N+1 查询问题

N+1 问题是 ORM 最常见的性能问题，指执行了 1 条查询获取 N 条主记录，然后又执行了 N 条查询获取每条记录的关联数据。

**N+1 问题图解：**

```

sqlalchemy.engine.Engine:
-- 第 1 条查询（取订单列表）
SELECT * FROM orders WHERE user_id = 1001;
-- 假设返回 100 条订单记录

-- 第 2~101 条查询（对每个订单查活动）
SELECT *FROM events WHERE id = 1;
SELECT* FROM events WHERE id = 2;
SELECT * FROM events WHERE id = 3;
...（共 100 条查询）

```text

#### 检测 N+1

方法一：开启 SQLAlchemy echo 日志

```python
# database.py 中设置 echo=True
engine = create_async_engine(
    settings.DATABASE_URL,
    echo=True,                    # 在控制台打印所有 SQL
    # ...
)
```

运行后观察日志中的 SQL 数量，如果多条相似的 SQL 依次出现，就是 N+1。

方法二：使用 `app.utils.profiling` 中的 `@timeit` 装饰器

```python
from app.utils.profiling import timeit

@timeit
async def list_orders_with_events(user_id: int):
    """N+1 问题版本"""
    result = await session.execute(
        select(Order).where(Order.user_id == user_id)
    )
    orders = result.scalars().all()

    for order in orders:                        # N 次查询
        await session.get(Event, order.event_id)
    return orders
```text

#### 解决方案

方法一：`selectinload`（推荐）

```python
from sqlalchemy.orm import selectinload

async def list_orders_with_events(user_id: int):
    """使用 selectinload 解决 N+1"""
    stmt = (
        select(Order)
        .where(Order.user_id == user_id)
        .options(selectinload(Order.event))     # 用 IN 查询一次性加载
        .order_by(Order.created_at.desc())
    )
    result = await session.execute(stmt)
    # 会生成 2 条 SQL：
    # 1. SELECT * FROM orders WHERE user_id = :user_id
    # 2. SELECT * FROM events WHERE id IN (:id1, :id2, ..., :idN)
    orders = result.scalars().all()

    # 访问 order.event 不会再产生查询
    for order in orders:
        print(order.event.title)
    return orders
```

方法二：`joinedload`（JOIN 方式）

```python
from sqlalchemy.orm import joinedload

async def list_orders_with_events(user_id: int):
    """使用 joinedload，通过 LEFT JOIN 一次性加载"""
    stmt = (
        select(Order)
        .where(Order.user_id == user_id)
        .options(joinedload(Order.event))       # LEFT JOIN events
        .order_by(Order.created_at.desc())
    )
    result = await session.execute(stmt)
    # 生成 1 条 SQL：
    # SELECT orders.*, events.*
    # FROM orders LEFT JOIN events ON orders.event_id = events.id
    orders = result.scalars().all()
    return orders
```text

**selectinload vs joinedload 对比：**

| 特性 | selectinload | joinedload |
| ------ | ------------- | ------------ |
| SQL 数量 | 2 条 | 1 条 |
| 大数据量时 | 更好（分批 IN 查询） | 可能产生笛卡尔积 |
| 对主查询影响 | 无影响 | LEFT JOIN 可能影响排序 |
| 加载多对多 | 适合 | 可能产生大量重复行 |
| 推荐场景 | 列表页、集合关联 | 一对一、详情页 |

### 1.6 连接池调优

数据库连接池管理应用与数据库之间的长连接，避免频繁创建和销毁连接的开销。

#### 连接池参数

```python
# app/database.py
engine = create_async_engine(
    settings.DATABASE_URL,

    # --- 连接池核心参数 ---
    pool_size=10,              # 连接池的固定连接数（默认 5）
                               # 公式：pool_size = (CPU核心数 * 2) + 有效磁盘数
                               # 对于 4 核服务器：pool_size = 4*2 + 2 = 10

    max_overflow=5,            # 超过 pool_size 后的最大额外连接数（默认 10）
                               # 突发流量时短期建立额外连接
                               # 总最大连接数 = pool_size + max_overflow = 15

    pool_recycle=3600,         # 连接的最大存活时间（秒），默认 -1（不过期）
                               # MySQL 的 wait_timeout 默认为 8 小时
                               # 建议设置为 3600 秒（1 小时），早于 MySQL 断开连接

    pool_pre_ping=True,        # 从池中取出连接前，先发送 SELECT 1 检测连接是否存活
                               # 如果连接已断开，透明地获取新连接

    # --- 辅助参数 ---
    echo=settings.DEBUG,       # 打印所有 SQL（生产环境设为 False）
    echo_pool=True,            # 打印连接池活动（调试用）
)
```

#### 调优策略

```python
# === 策略 1：根据并发用户数调整 ===
# 公式：pool_size = (并发用户数 * 请求耗时) / 1000
# 举例：100 并发用户，平均请求 200ms
# pool_size = (100 * 200) / 1000 = 20

# === 策略 2：根据 CPU 核心数调整 ===
# pool_size = 2 * CPU核心数
# 4 核 → pool_size = 8
# 8 核 → pool_size = 16
# 16 核 → pool_size = 32

# === 策略 3：分环境配置 ===
import os

DB_POOL_SIZE = {
    "development": 5,
    "staging": 10,
    "production": 20,
}.get(os.getenv("ENV", "development"), 5)

DB_MAX_OVERFLOW = {
    "development": 2,
    "staging": 5,
    "production": 10,
}.get(os.getenv("ENV", "development"), 2)

engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=DB_POOL_SIZE,
    max_overflow=DB_MAX_OVERFLOW,
    pool_pre_ping=True,
    pool_recycle=3600,
)
```text

#### 连接池监控

```python
# 查看连接池状态
from sqlalchemy import inspect

def check_pool_status(engine):
    """检查数据库连接池状态"""
    pool = engine.pool
    return {
        "size": pool.size(),                    # 当前连接数
        "checked_in": pool.checkedin(),         # 空闲连接数
        "checked_out": pool.overflow(),         # 已借出连接数
        "overflow": pool.overflow(),            # 额外连接数
    }

# 定期调用
import asyncio

async def monitor_pool():
    while True:
        status = check_pool_status(engine)
        print(f"连接池状态: {status}")
        await asyncio.sleep(60)  # 每分钟检查一次
```

---

## 2. Python 性能分析

### 2.1 cProfile — 确定性分析器

cProfile 是 Python 内置的性能分析工具，记录每个函数的调用次数和执行时间。

#### 基础用法

```bash
# === 方式 1：直接分析整个脚本 ===
python -m cProfile -o output.prof app/main.py

# === 方式 2：分析单个请求 ===
python -m cProfile -o profile.prof -m uvicorn app.main:app --workers 1
# 然后发送一个请求，Ctrl+C 退出，分析 profile.prof

# === 方式 3：按时间排序输出到终端 ===
python -m cProfile -s time app/main.py | head -30

# === 方式 4：按累计时间排序 ===
python -m cProfile -s cumulative app/main.py | head -30
```text

#### 代码中嵌入 cProfile

```python
import cProfile
import pstats
from pathlib import Path

def profile_request():
    """分析单个请求的性能"""
    profiler = cProfile.Profile()
    profiler.enable()

    # 执行要分析的代码
    import asyncio
    from app.services.order_service import OrderService
    # ... 执行订单查询 ...

    profiler.disable()

    # 保存结果
    profiler.dump_stats("request_profile.prof")

    # 输出统计
    stats = pstats.Stats(profiler)
    stats.sort_stats("cumulative")
    stats.print_stats(30)  # 打印前 30 条

    # 输出调用者/被调用者关系
    stats.print_callers("get_order")
    stats.print_callees("get_order")
```

#### pstats 分析

```python
import pstats

# 加载分析结果
stats = pstats.Stats("output.prof")

# 按累计时间排序
stats.sort_stats("cumulative")

# 打印前 20 条
stats.print_stats(20)

# 只显示包含 "order" 的函数的统计
stats.print_stats("order")

# 显示调用关系
stats.print_callers("get_order")
```text

**输出解读：**

```

         10023 function calls (9876 primitive calls) in 2.345 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.001    0.001    2.345    2.345 app/main.py:20(<module>)
        1    0.002    0.002    2.101    2.101 app/services/order_service.py:13(list_orders)
        5    1.234    0.247    1.234    0.247 {method 'execute' of 'AsyncSession' ...}
       45    0.567    0.013    0.567    0.013 {method 'get' of 'AsyncSession' ...}
        5    0.089    0.018    0.089    0.018 {method 'scalars' of 'Result' ...}

```text

| 列 | 含义 | 关注点 |
| ---- | ------ | -------- |
| `ncalls` | 调用次数 | 如果次数异常多，可能是 N+1 |
| `tottime` | 该函数自身耗时（不含子函数） | 纯计算密集型函数 |
| `percall`(首次) | tottime / ncalls | 单次调用自身耗时 |
| `cumtime` | 累计耗时（含子函数） | 性能瓶颈的入口 |
| `percall`(二次) | cumtime / ncalls | 单次调用累计耗时 |

### 2.2 snakeviz — 可视化分析

snakeviz 是 cProfile 输出的可视化工具，生成交互式火焰图和 ICE 图。

```bash
# 安装
pip install snakeviz

# 启动可视化服务（默认端口 8080）
snakeviz output.prof

# 指定端口
snakeviz -p 8081 output.prof

# 在浏览器中打开输出地址，会看到：
# 1. 火焰图（Flame Graph）：水平条代表函数调用，宽度代表耗时
# 2. ICE 图（Icicle）：从根向下的层级调用关系
```

**snakeviz 火焰图解读技巧：**

```text
火焰图从上往下看：

[uvicorn]                          ← 应用入口（整个请求的耗时）
├── [FastAPI.handle_request]       ← FastAPI 处理请求
│   ├── [get_order]                ← 业务逻辑
│   │   ├── [session.execute]      ← SQLAlchemy 执行查询（宽条 = 耗时多）
│   │   └── [session.get]          ← 单条查询
│   ├── [jwt.decode_token]         ← JWT 解码
│   └── [get_current_user]         ← 用户验证
└── [logging]                      ← 日志记录

技巧：
1. 找最宽的水平条 → 最耗时的函数
2. 看调用栈深度 → 是否有重复调用
3. 关注 "session.execute" 的宽度 → SQL 是否优化
4. 比较不同请求的火焰图 → 性能回归分析
```

### 2.3 py-spy — 生产安全分析器

py-spy 是采样式分析器（Sampling Profiler），**不修改目标进程的代码或执行流**，因此可以安全用于生产环境。

#### 安装

```bash
pip install py-spy
# py-spy 需要 root/sudo 权限来读取其他进程的内存
```text

#### 基础用法

```bash
# === 分析正在运行的 Python 进程 ===

# 方式 1：生成火焰图 SVG（直接杀死进程或 Ctrl+C 停止）
sudo py-spy record -o profile.svg --pid 12345

# 方式 2：指定采样频率（默认 100Hz）
sudo py-spy record -o profile.svg --pid 12345 --rate 50

# 方式 3：设置持续时间
sudo py-spy record -o profile.svg --pid 12345 --duration 30

# 方式 4：分析启动中的程序
sudo py-spy record -o profile.svg -- python app/main.py

# 方式 5：实时查看调用栈（top 风格）
sudo py-spy top --pid 12345

# 方式 6：只显示 Python 栈（不显示原生 C 栈）
sudo py-spy record -o profile.svg --pid 12345 --subprocesses
```

#### py-spy top 实时监控

```bash
# 实时显示最耗时的函数
sudo py-spy top --pid 12345

# 输出示例：
GIL: 0.0%, Active: 1, Threads: 8

  %Own   %Total  OwnTime  TotalTime  Function (filename)
  45.2%  45.2%   2.34s    2.34s      execute (sqlalchemy/engine/base.py)
  20.1%  20.1%   1.02s    1.02s      get (sqlalchemy/orm/session.py)
  15.3%  15.3%   0.78s    0.78s      select (sqlalchemy/sql/selectable.py)
   5.2%   5.2%   0.25s    0.25s      decode (jwt/algorithm.py)
```text

#### py-spy 与 cProfile 对比

| 特性 | cProfile | py-spy |
| ------ | ---------- | -------- |
| 类型 | 确定性（追踪每个调用） | 采样式（定时检查调用栈） |
| 生产安全 | 否（会拖慢程序 2-10x） | 是（开销极小，<1%） |
| 需要修改代码 | 是 | 否 |
| 支持多线程 | 是（但可能不准） | 是 |
| 输出格式 | .prof 文件 | SVG 火焰图 / Top |
| 定位 CPU 热点 | 精确 | 统计近似 |
| 定位 I/O 等待 | 不直接支持 | 可以看到等待栈 |
| 需要 root | 否 | 是（分析其他进程） |

### 2.4 @profile 装饰器方案

对于不想引入外部工具的简单场景，可以使用项目自带的 `@timeit` 装饰器。

```python
# app/utils/profiling.py
import time
import functools
from typing import Callable
from app.utils.logging_config import logger  # 可选的日志模块


def timeit(func: Callable) -> Callable:
    """装饰器：打印函数执行时间"""
    @functools.wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.perf_counter()          # 使用高精度计时器
        result = await func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"[profile] {func.__name__} took {elapsed*1000:.2f}ms")
        return result
    return wrapper


# 使用示例
from app.utils.profiling import timeit

class OrderService:
    @timeit
    async def list_orders(self, user_id: int, page: int = 1, size: int = 20):
        """带性能监控的订单列表查询"""
        # ...
        pass

# 输出示例：
# [profile] list_orders took 45.32ms
# [profile] create_order took 12.15ms
```

**进阶版装饰器（含调用次数统计和阈值告警）：**

```python
import time
import functools
from collections import defaultdict

# 函数调用统计
_call_stats = defaultdict(lambda: {"count": 0, "total_time": 0.0, "max_time": 0.0})


def monitor(threshold_ms: float = 1000):
    """性能监控装饰器，超过阈值打印警告"""
    def decorator(func):
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            start = time.perf_counter()
            result = await func(*args, **kwargs)
            elapsed = time.perf_counter() - start

            # 更新统计
            stats = _call_stats[func.__name__]
            stats["count"] += 1
            stats["total_time"] += elapsed
            stats["max_time"] = max(stats["max_time"], elapsed)
            stats["avg_time"] = stats["total_time"] / stats["count"]

            # 超过阈值时告警
            if elapsed * 1000 > threshold_ms:
                print(f"[WARN] {func.__name__} 耗时 {elapsed*1000:.2f}ms (阈值 {threshold_ms}ms)")

            return result
        return async_wrapper
    return decorator


def print_stats():
    """打印所有被监控函数的统计"""
    print("=" * 60)
    print(f"{'函数名':<30} {'调用次数':<10} {'平均耗时':<12} {'最大耗时':<12}")
    print("-" * 60)
    for name, stats in sorted(_call_stats.items()):
        print(f"{name:<30} {stats['count']:<10} "
              f"{stats['avg_time']*1000:<10.2f}ms {stats['max_time']*1000:<10.2f}ms")
    print("=" * 60)
```text

---

## 3. 缓存层次架构

### 3.1 四级缓存架构

本系统采用四层缓存架构，从 L1 到 L4，速度递减但存储容量递增：

```

┌─────────────────────────────────────────────────────────────┐
│                   四级缓存架构总览                            │
├─────────┬─────────────┬───────────┬────────────┬────────────┤
│ 层级    │ 技术方案    │ 速度      │ 存储容量   │ 适用场景   │
├─────────┼─────────────┼───────────┼────────────┼────────────┤
│ L1      │ lru_cache   │ 微秒级    │ 进程内存   │ 热点配置   │
│ L2      │ Redis       │ 毫秒级    │ 内存 ~GB   │ 会话/库存  │
│ L3      │ Nginx       │ 毫秒级    │ 内存+磁盘  │ 静态资源   │
│ L4      │ MySQL查询缓存│ 毫秒级    │ 磁盘       │ 冷数据     │
└─────────┴─────────────┴───────────┴────────────┴────────────┘

```text

### 3.2 L1：进程内缓存（lru_cache）

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def get_config(key: str) -> str:
    """获取配置项（LRU 缓存，最多缓存 128 项）"""
    # 实际从数据库读取
    return db.query(Config).filter(Config.key == key).first().value

# 首次调用：查询数据库
value1 = get_config("system_name")    # cache miss

# 再次调用：从缓存读取（微秒级）
value2 = get_config("system_name")    # cache hit

# 查看缓存统计
print(get_config.cache_info())
# CacheInfo(hits=5, misses=1, maxsize=128, currsize=1)

# 清除缓存
get_config.cache_clear()

# 限制缓存大小的策略：maxsize=None 表示无限制
# 但要注意进程内存，建议设置合理上限

# === LRU 缓存工作原理 ===
# 当缓存项超过 maxsize 时，移除最久未使用的项
# 适用于：热点数据频繁访问，冷数据自动淘汰
```

**lru_cache 性能测试：**

```python
import time
from functools import lru_cache

# 模拟耗时数据库查询
def get_from_db(key: str) -> str:
    time.sleep(0.1)  # 100ms 数据库查询
    return f"value_{key}"

# 无缓存版本
start = time.perf_counter()
for i in range(100):
    get_from_db(f"key_{i % 10}")   # 只查询 10 个键
elapsed = time.perf_counter() - start
print(f"无缓存: {elapsed*1000:.0f}ms")  # 约 10000ms (100 * 100ms)

# LRU 缓存版本
@lru_cache(maxsize=10)
def get_from_db_cached(key: str) -> str:
    time.sleep(0.1)
    return f"value_{key}"

start = time.perf_counter()
for i in range(100):
    get_from_db_cached(f"key_{i % 10}")
elapsed = time.perf_counter() - start
print(f"LRU缓存: {elapsed*1000:.0f}ms")   # 约 1000ms (10 * 100ms)

# 性能提升 10 倍！
```text

### 3.3 L2：Redis 缓存

Redis 作为分布式缓存，跨所有应用实例共享。

```python
from redis.asyncio import Redis
import json

class EventCache:
    """活动信息 Redis 缓存"""

    def __init__(self, redis: Redis):
        self.redis = redis

    async def get_event(self, event_id: int) -> dict | None:
        """获取缓存的活动信息"""
        key = f"event:{event_id}"
        cached = await self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None

    async def set_event(self, event_id: int, event_data: dict, ttl: int = 300):
        """设置活动缓存（默认 5 分钟）"""
        key = f"event:{event_id}"
        await self.redis.setex(key, ttl, json.dumps(event_data, default=str))

    async def invalidate_event(self, event_id: int):
        """使活动缓存失效"""
        key = f"event:{event_id}"
        await self.redis.delete(key)
```

**Redis 缓存更新策略：**

```text
策略 1：Cache-Aside（旁路缓存）—— 推荐
  读取：先读缓存 → 未命中则读数据库 → 写入缓存
  写入：先更新数据库 → 删除缓存（而非更新缓存）

策略 2：Read-Through（穿透读）
  读取：由缓存库自动从数据库加载
  写入：由缓存库自动更新

策略 3：Write-Behind（异步写）
  写入：先更新缓存 → 异步批量写入数据库
  注意：可能丢失数据，适用于日志类场景
```

**缓存穿透/击穿/雪崩解决方案：**

```python
# === 缓存穿透（查询不存在的数据） ===
# 问题：大量请求查询一个不存在的 key，压力集中到数据库
async def get_event_safe(event_id: int) -> dict | None:
    key = f"event:{event_id}"
    cached = await redis.get(key)
    if cached is not None:
        return json.loads(cached) if cached != "NULL" else None

    # 从数据库加载
    event = await db.get(Event, event_id)
    if event:
        await redis.setex(key, 300, json.dumps(event.to_dict()))
    else:
        # 对不存在的 key 也缓存空值（短期）
        await redis.setex(key, 60, "NULL")
    return event

# === 缓存击穿（热点 key 过期） ===
# 问题：高并发访问一个即将过期的 key，大量线程同时回源数据库
# 解决：使用互斥锁（SETNX）
async def get_hot_event(event_id: int) -> dict | None:
    key = f"event:{event_id}"
    cached = await redis.get(key)
    if cached:
        return json.loads(cached)

    # 尝试获取互斥锁（只有第一个线程能成功）
    lock_key = f"lock:event:{event_id}"
    locked = await redis.setnx(lock_key, "1")
    if locked:
        await redis.expire(lock_key, 10)  # 锁 10 秒超时
        try:
            event = await db.get(Event, event_id)
            if event:
                await redis.setex(key, 300, json.dumps(event.to_dict()))
            return event
        finally:
            await redis.delete(lock_key)
    else:
        # 其他线程等待一会儿后重试
        await asyncio.sleep(0.1)
        return await self.get_hot_event(event_id)

# === 缓存雪崩（大量 key 同时过期） ===
# 解决：在基础过期时间上增加随机值
ttl = 300 + random.randint(0, 60)  # 5分钟 + 0-60秒随机
await redis.setex(key, ttl, value)
```text

### 3.4 L3：Nginx 缓存

Nginx 在反向代理层提供缓存功能，缓存静态资源和 API 响应。

```nginx
# nginx.conf 缓存配置片段

http {
    # 定义缓存路径和参数
    proxy_cache_path /var/cache/nginx levels=1:2
                     keys_zone=ticket_cache:10m
                     max_size=1g
                     inactive=60m
                     use_temp_path=off;

    server {
        listen 80;
        server_name api.example.com;

        # --- 静态资源缓存（强缓存） ---
        location /static/ {
            root /opt/ticket-booking;
            expires 30d;                    # 浏览器缓存 30 天
            add_header Cache-Control "public, immutable";
        }

        # --- API 缓存（事件列表，5 分钟） ---
        location /api/v1/events {
            proxy_pass http://backend;

            # 启用缓存
            proxy_cache ticket_cache;
            proxy_cache_key "$scheme$request_method$host$request_uri";
            proxy_cache_valid 200 5m;       # 200 响应缓存 5 分钟
            proxy_cache_valid 404 1m;       # 404 响应缓存 1 分钟

            # 缓存绕过条件（带特定 header 时绕过）
            proxy_cache_bypass $http_cache_control;
            proxy_no_cache $http_pragma;

            # 添加缓存状态 header
            add_header X-Cache-Status $upstream_cache_status;
            # $upstream_cache_status 取值：
            #   HIT   → 命中缓存
            #   MISS  → 未命中
            #   EXPIRED → 已过期
            #   BYPASS → 已绕过
        }

        # --- 不缓存接口（订单、支付等动态数据） ---
        location /api/v1/orders {
            proxy_pass http://backend;
            proxy_no_cache 1;               # 不缓存
            proxy_cache_bypass 1;           # 绕过缓存
        }

        # --- API 缓存清理接口 ---
        location ~ /purge(/.*) {
            allow 127.0.0.1;
            deny all;
            proxy_cache_purge ticket_cache "$scheme$request_method$host$1";
        }
    }
}
```

### 3.5 L4：MySQL 查询缓存

MySQL 8.0 已弃用查询缓存（Query Cache）。替代方案是使用缓冲池和索引优化。

```my.cnf
[mysqld]
# 替代查询缓存的配置
innodb_buffer_pool_size = 1G          # 设置为物理内存的 60-80%
innodb_buffer_pool_instances = 4      # 减少锁争用（每个实例约 256MB）
innodb_old_blocks_time = 1000         # 1 秒内不被访问的页不进入 LRU 热区
innodb_log_file_size = 256M           # 增大 redo log 减少磁盘 I/O
```text

---

## 4. B+Tree 索引原理的数学证明

### 4.1 B+Tree 的基本结构

B+Tree 是 MySQL InnoDB 存储引擎默认的索引结构。理解其数学原理对优化索引设计至关重要。

```

B+Tree (阶数 m = 5 的简化示例)：

                 [50, 80, 120, 200]
                /    |    |     \    \
           [10,30] [55,70] [90,110] [130,180] [220,300]
           /   \    /   \    /   \    /    \    /    \
         [1,5] [31] [51] [71] [91] [111] [131] [181] [221]

```text

**关键特性：**

1. **非叶子节点**存储键值和子节点指针（不存储数据）
2. **叶子节点**存储键值和数据（或指向数据的指针）
3. **叶子节点之间**通过链表连接，支持范围查询
4. 所有根到叶子的路径**等高**

### 4.2 扇出（Fanout）计算

扇出指的是每个节点最多能持有的子节点数量，由页面大小和键/指针大小决定。

```

MySQL InnoDB 默认页面大小：page_size = 16384 bytes (16KB)

假设：
  键（key）大小：8 bytes（如 BIGINT）
  指针（pointer）大小：6 bytes（InnoDB 内部指针）

每个节点能存储的键值对数量：
  fanout = page_size / (key_size + pointer_size)
         = 16384 / (8 + 6)
         = 16384 / 14
         ≈ 1170

实际中，考虑节点本身的元数据（页头、页尾等）：
  可用空间 ≈ 16384 - 128 ≈ 16256 bytes
  实际 fanout ≈ 16256 / 14 ≈ 1161

简记：InnoDB 的 B+Tree 扇出约为 1000-1200

```text

### 4.3 树高计算

B+Tree 的高度决定了查询需要几次磁盘 I/O。

```

公式：高度 h = log_fanout(N)
  其中 N = 数据行数
      fanout = 每个节点的分支数

计算表：

行数 N      fanout    高度 h              I/O 次数
─────────────────────────────────────────────────
1,000       1,000     log_1000(1000) = 1   1
10,000      1,000     log_1000(10000) = 1.33  → 2
1,000,000   1,000     log_1000(1M) = 2     2
1,000,000,000 1,000   log_1000(1B) = 3     3
10,000,000  1,024     log_1024(10M)        约 2.3 → 3

数学推导：
  h = log_fanout(N)
  fanout = 1024（近似值）
  
  当 N = 10,000,000：
  h = log_1024(10,000,000)
  = ln(10,000,000) / ln(1024)
  ≈ 16.12 / 6.93
  ≈ 2.33
  
  所以树高为 3（向上取整），查询最多需要 3 次磁盘 I/O

  当 N = 1,000,000,000：
  h = log_1024(1,000,000,000)
  ≈ 20.72 / 6.93
  ≈ 2.99
  
  即使有 10 亿行数据，树高也只有 3！

```text

**数学证明总结：**

```

因为 fanout ≈ 1024，所以：
  1024^1 ≈ 1K（层高 1 可管理约 1 千条记录）
  1024^2 ≈ 1M（层高 2 可管理约 100 万条记录）
  1024^3 ≈ 1B（层高 3 可管理约 10 亿条记录）

每个层级能容纳的记录数是指数增长的。
这也是为什么 B+Tree 索引在海量数据下仍然高效的数学基础。

结论：在合理设计的表上，通过主键查询最多需要 3-4 次磁盘 I/O。

```text

### 4.4 索引优化的数学依据

```sql
-- 基于 B+Tree 原理的索引设计原则：

-- 原则 1：索引列应尽量短
-- key 越小 → fanout 越大 → 树高越低 → I/O 更少
-- BIGINT(8B) 优于 VARCHAR(255)
CREATE INDEX idx_user_id ON orders (user_id);  -- 8B per key

-- 原则 2：复合索引前缀选择性要高
-- 选择性越高 → 索引过滤的行越多 → 减少回表

-- 原则 3：覆盖索引避免回表
-- 如果查询只需要索引中的列，无需访问数据页
-- Extra: Using index

-- 原则 4：使用自增主键
-- B+Tree 的叶子节点按主键顺序排列
-- UUID 或随机主键 → 频繁的页分裂 → 索引碎片
-- 自增 BIGINT → 顺序插入 → 无页分裂

-- 页分裂的代价：
-- 当插入的键不在当前页的末尾时，需要拆分页面
-- 拆分涉及：分配新页、复制数据、更新父节点指针
-- 在高并发插入场景下，非自增主键可能导致严重的性能下降
```

---

## 5. 生产上线 Checklist（20+ 项）

### 5.1 基础设施

- [ ] **SSL/TLS 证书**：使用 Let's Encrypt 或商业证书

  ```bash
  # 申请 Let's Encrypt 免费证书
  apt install certbot python3-certbot-nginx
  certbot --nginx -d api.example.com

  # 设置自动续期
  certbot renew --dry-run
  ```

- [ ] **域名解析**：DNS A 记录指向服务器 IP，配置 CNAME
- [ ] **防火墙规则**：仅开放必要端口（80, 443, SSH）

  ```bash
  # iptables 规则示例
  iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # HTTP
  iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # HTTPS
  iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # SSH
  iptables -A INPUT -j DROP                         # 默认拒绝
  ```

### 5.2 数据库

- [ ] **定期备份策略**：

  ```bash
  # 每日全量备份（crontab）
  0 3 * * * mysqldump -u root -p ticket_prod | gzip > /backup/ticket_$(date +\%Y\%m\%d).sql.gz

  # 保留最近 30 天备份
  0 5 * * * find /backup -name "ticket_*.sql.gz" -mtime +30 -delete
  ```

- [ ] **备份恢复测试**：每月测试一次从备份恢复
- [ ] **慢查询监控**：启用慢查询日志并定期分析
- [ ] **连接数限制**：`max_connections = 200`，防止连接耗尽
- [ ] **字符集和排序规则**：统一使用 `utf8mb4` 和 `utf8mb4_unicode_ci`

### 5.3 监控和告警

- [ ] **服务器监控**：CPU、内存、磁盘、网络

  ```bash
  # 安装 Node Exporter + Prometheus + Grafana
  # 关键告警指标：
  #   CPU > 80% 持续 5 分钟
  #   磁盘使用率 > 85%
  #   内存使用率 > 90%
  ```

- [ ] **应用监控**：
  - API 响应时间 P99 < 500ms
  - API 错误率 < 1%
  - 端点健康检查（每分钟）

- [ ] **数据库监控**：
  - 活跃连接数
  - 慢查询数
  - 死锁次数
  - 复制延迟（如果有主从）

- [ ] **业务监控**：
  - 订单创建成功率
  - 支付成功率
  - 退款处理时间

### 5.4 安全性

- [ ] **安全响应头**：

  ```nginx
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-XSS-Protection "1; mode=block" always;
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
  add_header Content-Security-Policy "default-src 'self'" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  ```text

- [ ] **请求速率限制**：

  ```nginx
  # Nginx 限流
  limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
  location /api/v1/ {
      limit_req zone=api burst=20 nodelay;
  }
  ```

- [ ] **CORS 配置**：仅允许可信域名

  ```python
  # main.py
  app.add_middleware(
      CORSMiddleware,
      allow_origins=["https://example.com"],   # 生产环境不应使用 ["*"]
      allow_credentials=True,
      allow_methods=["GET", "POST", "PUT", "DELETE"],
      allow_headers=["Authorization", "Content-Type"],
  )
  ```text

- [ ] **数据库密码强度**：使用 `openssl rand -base64 32` 生成

- [ ] **密钥轮换计划**：SECRET_KEY、数据库密码每 90 天轮换
- [ ] **依赖安全扫描**：

  ```bash
  pip-audit                   # 扫描 Python 依赖漏洞
  safety check                # 另一款安全扫描器
  ```

### 5.5 日志管理

- [ ] **日志轮转配置**：

  ```bash
  # /etc/logrotate.d/ticket-app
  /opt/ticket-booking/logs/*.log {
      daily
      rotate 30
      compress
      delaycompress
      missingok
      notifempty
      copytruncate
  }
  ```

- [ ] **应用日志分级**：ERROR/WARN/INFO/DEBUG 分级别输出
- [ ] **敏感信息脱敏**：日志中不记录密码、Token、支付信息
- [ ] **集中式日志收集**：ELK (Elasticsearch + Logstash + Kibana) 或 Loki + Grafana

### 5.6 CI/CD 和运维

- [ ] **CI/CD 流水线**：
  - 测试流水线在 PR 时自动运行
  - 构建流水线在 Release 时触发
  - 部署流水线自动执行并检查

- [ ] **灰度发布策略**：新版本先部署到 staging 环境，验证通过后再上线
- [ ] **回滚方案**：保存历史版本镜像，回滚命令不超过 5 分钟

### 5.7 灾难恢复

- [ ] **灾备方案**：
  - 每天备份到异地存储（OSS/S3）
  - 主数据库故障时切换到从库
  - 关键操作有操作日志可追溯

- [ ] **SLA 定义**：
  - 服务可用性目标：99.9%（全年宕机不超过 8.76 小时）
  - RTO（恢复时间目标）：< 1 小时
  - RPO（恢复点目标）：< 15 分钟

- [ ] **紧急联系人**：至少 2 人可响应线上故障

### 5.8 性能基准

```bash
# 性能基准测试（上线前后各执行一次）

# 1. API 响应时间
locust -f tests/locustfile.py --headless -u 100 -r 10 --run-time 5m
# 预期：P95 < 500ms, P99 < 1s, 错误率 < 0.1%

# 2. 数据库查询
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1001;
# 预期：rows < 1000, type = ref 或 range

# 3. Redis 响应时间
redis-cli --latency -h localhost -p 6379
# 预期：avg < 1ms

# 4. 静态资源加载
curl -w "%{time_total}" -o /dev/null -s https://api.example.com/static/css/main.css
# 预期：< 50ms
```

---

## 6. 架构回顾

### 6.1 最终系统架构图

```text
用户 (Browser/App)
    │
    ▼
Nginx (反向代理 + SSL 终止 + 缓存 + 限流)
    │
    ▼
FastAPI (Uvicorn ASGI Server)
    ├── Middleware (CORS / Auth / Logging / Rate Limit)
    ├── Routers (Auth / Events / Orders / Payments / WebSocket)
    ├── Services (业务逻辑层)
    │   ├── OrderService      → 订单 + 库存（Redis + MySQL乐观锁）
    │   ├── PaymentService    → 支付宝支付
    │   ├── InventoryService  → Redis 原子扣减
    │   └── NotificationService → WebSocket 推送
    ├── Models (SQLAlchemy ORM)
    │   ├── User → orders / tickets
    │   ├── Event → orders / tickets
    │   ├── Order → tickets
    │   └── Ticket
    └── Schemas (Pydantic 校验)
        └── Request / Response 模型
            │
            ▼
MySQL 8.0 (主数据库)
    ├── orders (B+Tree index: user_id, order_no, status)
    ├── events (B+Tree index: status, start_date)
    ├── users (B+Tree index: username, email)
    └── tickets (B+Tree index: order_id, event_id, user_id)

Redis 7 (缓存 + 库存 + 会话)
    ├── 活动信息缓存 (event:{id})
    ├── 库存原子扣减 (stock:{event_id})
    ├── 用户限购 (purchase_limit:{user_id}:{event_id})
    ├── 支付幂等 (alipay:processed:{order_no})
    └── JWT Token 黑名单 (token:blacklist:{jti})
```

### 6.2 技术栈总结

| 层级 | 技术 | 版本 | 用途 |
| ------ | ------ | ------ | ------ |
| 语言 | Python | 3.12 | 后端开发语言 |
| Web 框架 | FastAPI | 0.111+ | 异步 API 框架 |
| 数据库 | MySQL | 8.0 | 主数据存储 |
| 缓存 | Redis | 7-alpine | 缓存、库存、限购 |
| ORM | SQLAlchemy | 2.0+ | 数据库映射 |
| 迁移 | Alembic | 1.13+ | 数据库版本管理 |
| 容器化 | Docker | 24+ | 镜像化部署 |
| 编排 | Docker Compose | 2.x | 多容器编排 |
| CI/CD | GitHub Actions | - | 自动测试构建 |
| 部署 | Linux / Windows | - | 生产运行环境 |
| 支付 | 支付宝 | alipay-sdk-python | 在线支付集成 |

---

✅ **本章项目里程碑：**

- [ ] SQL 慢查询日志已启用且分析过
- [ ] EXPLAIN 优化了所有慢查询
- [ ] N+1 问题已通过 selectinload/joinedload 解决
- [ ] 连接池参数已根据服务器配置调整
- [ ] 使用 cProfile/py-spy 分析过生产请求
- [ ] L1-L4 缓存层次已实施
- [ ] 上线清单 20+ 项全部确认
- [ ] B+Tree 索引设计有数学依据
- [ ] 性能基准测试报告完成

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/utils/profiling.py` | 性能分析工具：cProfile 封装、Prometheus 指标暴露 |
| `app/middleware/logging.py` | 请求耗时日志记录（内置性能监控） |
| `app/services/inventory_service.py` | Redis Lua 库存扣减（毫秒级操作） |
| `app/services/connection_manager.py` | WebSocket 连接管理（性能关键路径） |

**开启 Profiling：**

```python
from app.utils.profiling import profile_request

@router.get("/slow-endpoint")
@profile_request  # 自动记录函数执行时间到 Prometheus
async def slow_endpoint():
    ...
```text

## 进阶：WebSocket 实时性能优化

### 连接管理

WebSocket 连接使用 ConnectionManager 管理，基于房间（room）模式：

```

Room: seats:{event_id}    — 座位实时更新（所有用户可见）
Room: user:{user_id}      — 订单状态更新（仅对应用户）

```text

### 心跳检测

每 30 秒发送 ping 帧，60 秒无响应则断开连接，防止僵尸连接占用资源。

### 自动取消超时订单（ARQ Worker）

```

Worker 每 60 秒轮询
  ↓
SELECT * FROM orders WHERE status='pending_payment' AND expires_at < NOW()
  ↓
批量取消（RESTORE 库存 + 广播取消通知）
  ↓
返回取消数量

```text

### 支付倒计时前端实现

创建订单时返回 `expires_at = now + 15min`，前端用 `PaymentCountdown` 组件展示倒计时：

```

组件收到 expires_at
  ↓
计算剩余毫秒数
  ↓
每秒 setInterval 更新 display = MM:SS
  ↓
剩余 < 10% → 显示"即将超时"警告
  ↓
剩余 = 0 → 触发 expired 事件 → 跳转订单页

```text
