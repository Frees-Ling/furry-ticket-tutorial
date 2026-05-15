# 第 29 章：基础概念速查手册

> **难度等级：** 初级
> **前置知识：** 无
> **学习目标：** 快速掌握 Python/SQL/Git/HTTP 核心基础概念，为后续章节扫清障碍

---

## 目录

1. [Python 基础速查](#1-python-基础速查)
2. [SQL 基础速查](#2-sql-基础速查)
3. [Git 基础速查](#3-git-基础速查)
4. [HTTP/REST 基础](#4-httprest-基础)

---

## 1. Python 基础速查

### 1.1 变量与基本类型

```python
# ===== 基本类型 =====
name: str = "张三"          # 字符串（类型注解可选）
age: int = 25               # 整数
price: float = 99.99        # 浮点数
is_active: bool = True      # 布尔值
data: bytes = b"hello"      # 字节串
nothing = None              # 空值

# ===== 字符串操作 =====
s = "Hello, 世界"
s[0]                        # → 'H'（索引）
s[0:5]                      # → 'Hello'（切片）
f"姓名: {name}, 年龄: {age}"  # f-string 格式化
s.split(",")                # → ['Hello', ' 世界']
s.replace("Hello", "Hi")    # → 'Hi, 世界'
",".join(["a", "b", "c"])   # → 'a,b,c'

# ===== 集合类型 =====
nums_list = [1, 2, 3]                    # 列表（有序，可变）
nums_tuple = (1, 2, 3)                   # 元组（有序，不可变）
nums_set = {1, 2, 3}                     # 集合（无序，去重）
user_dict = {"name": "张三", "age": 25}  # 字典（键值对）

# 推导式
squares = [x**2 for x in range(10)]        # → [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
even_squares = [x**2 for x in range(10) if x % 2 == 0]  # 带条件
square_dict = {x: x**2 for x in range(5)}  # 字典推导式
```text

### 1.2 函数

```python
# ===== 基础函数 =====
def greet(name: str) -> str:
    """文档字符串：说明函数功能"""
    return f"你好, {name}"

# ===== 默认参数 =====
def create_user(name: str, age: int = 18, role: str = "user") -> dict:
    return {"name": name, "age": age, "role": role}

# ===== 可变参数 =====
def sum_all(*args: int) -> int:       # 任意数量位置参数
    return sum(args)

def log_event(event: str, **kwargs):  # 任意数量关键字参数
    print(f"[{event}] {kwargs}")

# ===== 匿名函数 =====
double = lambda x: x * 2
sorted([(1, "b"), (2, "a")], key=lambda x: x[1])  # 按第二个元素排序

# ===== 装饰器（高阶函数） =====
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        import time
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} 耗时: {time.time() - start:.3f}s")
        return result
    return wrapper

@timer
def slow_function():
    import time
    time.sleep(1)
```

### 1.3 类与面向对象

```python
# ===== 基础类 =====
class User:
    """用户类"""
    # 类变量（所有实例共享）
    total_users: int = 0

    def __init__(self, username: str, email: str):
        """构造函数"""
        self.username = username          # 实例变量
        self.email = email
        User.total_users += 1

    def __repr__(self) -> str:
        """字符串表示（调试用）"""
        return f"User({self.username})"

    # 实例方法
    def display(self) -> str:
        return f"用户名: {self.username}, 邮箱: {self.email}"

    # 类方法
    @classmethod
    def create_admin(cls) -> "User":
        return cls("admin", "admin@example.com")

    # 静态方法
    @staticmethod
    def validate_email(email: str) -> bool:
        return "@" in email

# ===== 继承 =====
class AdminUser(User):
    def __init__(self, username: str, email: str, level: int = 1):
        super().__init__(username, email)  # 调用父类构造
        self.level = level

    def display(self) -> str:              # 方法重写
        return f"管理员: {self.username}, 级别: {self.level}"

# ===== 数据类（Python 3.7+） =====
from dataclasses import dataclass

@dataclass
class Product:
    name: str
    price: float
    quantity: int = 0

    @property
    def total_value(self) -> float:
        return self.price * self.quantity
```text

### 1.4 async/await 异步编程

```python
import asyncio

# ===== 异步函数定义 =====
async def fetch_data(url: str) -> str:
    """异步函数：用 async def 声明"""
    await asyncio.sleep(1)  # 模拟网络请求
    return f"从 {url} 获取的数据"

# ===== 并发执行 =====
async def main():
    # 顺序执行（总耗时 ~2s）
    # result1 = await fetch_data("url1")
    # result2 = await fetch_data("url2")

    # 并发执行（总耗时 ~1s）
    task1 = asyncio.create_task(fetch_data("url1"))
    task2 = asyncio.create_task(fetch_data("url2"))
    result1 = await task1
    result2 = await task2

    # 批量并发
    urls = [f"url{i}" for i in range(10)]
    tasks = [fetch_data(url) for url in urls]
    results = await asyncio.gather(*tasks)  # 全部完成

    # 超时控制
    try:
        result = await asyncio.wait_for(
            fetch_data("slow-url"), timeout=5.0
        )
    except asyncio.TimeoutError:
        print("请求超时")

# 运行入口
# asyncio.run(main())
```

### 1.5 类型注解（Type Hints）

```python
from typing import Optional, Union, List, Dict, Tuple, Any, Callable

# 基本类型
name: str = "张三"
count: int | float = 42  # Python 3.10+ 联合类型

# 容器类型
names: list[str] = ["张三", "李四"]
scores: dict[str, int] = {"张三": 95}
mixed: tuple[int, str, bool] = (1, "a", True)

# 可选类型（等同于 Union[X, None]）
age: Optional[int] = None
age: int | None = None       # Python 3.10+ 写法

# 联合类型
value: Union[int, str] = 42
value: int | str = "hello"   # Python 3.10+

# 可调用类型
handler: Callable[[int, str], bool]  # 参数 (int, str) → 返回 bool

# 任意类型
anything: Any = "可以是任何类型"

# ===== FastAPI 中的类型注解 =====
# from fastapi import Query, Path, Body

# @app.get("/items/{item_id}")
# async def get_item(
#     item_id: int = Path(..., gt=0),       # 路径参数，大于0
#     q: str | None = Query(None, max_length=50),  # 查询参数
# ):
```text

### 1.6 常用标准库

```python
import os            # 操作系统接口：os.path, os.environ
import sys           # Python 解释器：sys.path, sys.argv
import json          # JSON 编解码：json.dumps, json.loads
import re            # 正则表达式：re.search, re.match, re.sub
import datetime      # 日期时间：datetime.now(), timedelta
import math          # 数学函数：math.sqrt, math.ceil, math.floor
import random        # 随机数：random.randint, random.choice
import hashlib       # 哈希：hashlib.sha256, hashlib.md5
import uuid          # UUID 生成：uuid.uuid4()
import logging       # 日志：logging.getLogger, logging.info
import time          # 时间函数：time.time(), time.sleep
```

---

## 2. SQL 基础速查

### 2.1 数据库基本概念

```text
数据库（Database）     → 项目的持久化存储
表（Table）            → 数据的结构化集合
行（Row / Record）     → 一条完整的数据
列（Column / Field）   → 一个数据字段
主键（Primary Key）    → 唯一标识一行记录的字段
外键（Foreign Key）    → 引用其他表的主键
索引（Index）          → 加速查询的数据结构
```

### 2.2 DDL：数据定义语言

```sql
-- ===== 创建表 =====
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- 自增主键
    username VARCHAR(50) NOT NULL UNIQUE,   -- 非空唯一
    email VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'user',        -- 默认值
    is_active BOOLEAN DEFAULT TRUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email),                -- 索引
    UNIQUE INDEX idx_username (username)    -- 唯一索引
);

-- ===== 修改表 =====
ALTER TABLE users ADD COLUMN phone VARCHAR(20);     -- 添加列
ALTER TABLE users MODIFY COLUMN phone VARCHAR(30);  -- 修改列类型
ALTER TABLE users DROP COLUMN phone;                -- 删除列
ALTER TABLE users ADD INDEX idx_role (role);        -- 添加索引

-- ===== 外键约束 =====
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)  -- 外键
);
```text

### 2.3 DML：数据操作语言

```sql
-- ===== INSERT（插入） =====
INSERT INTO users (username, email, password_hash, role)
VALUES ('zhangsan', 'zhangsan@example.com', 'hashed_pwd', 'user');

INSERT INTO users (username, email, password_hash)
VALUES
    ('lisi', 'lisi@example.com', 'hash1'),
    ('wangwu', 'wangwu@example.com', 'hash2');

-- ===== SELECT（查询） =====
-- 基本查询
SELECT * FROM users;                        -- 全表查询
SELECT id, username, email FROM users;       -- 指定列
SELECT * FROM users WHERE id = 1;            -- 条件查询
SELECT * FROM users WHERE role = 'admin' AND is_active = TRUE;

-- 排序与分页
SELECT * FROM events ORDER BY created_at DESC LIMIT 10;
SELECT * FROM events ORDER BY created_at DESC LIMIT 10 OFFSET 20;  -- 第2页
SELECT * FROM events ORDER BY created_at DESC LIMIT 20, 10;        -- MySQL 简写

-- 聚合函数
SELECT COUNT(*) FROM users;                  -- 计数
SELECT MAX(price) FROM events;               -- 最大值
SELECT MIN(price) FROM events;               -- 最小值
SELECT AVG(price) FROM events;               -- 平均值
SELECT SUM(quantity) FROM orders;            -- 求和

-- 分组
SELECT role, COUNT(*) as user_count
FROM users
GROUP BY role
HAVING COUNT(*) > 1;                        -- 分组后过滤

-- LIKE 模糊查询
SELECT * FROM events WHERE title LIKE '%演唱会%';

-- BETWEEN 范围查询
SELECT * FROM orders
WHERE total_amount BETWEEN 100 AND 500;

-- IN 集合查询
SELECT * FROM users WHERE role IN ('admin', 'manager');

-- ===== UPDATE（更新） =====
UPDATE users SET email = 'new@example.com' WHERE id = 1;
UPDATE orders SET status = 'paid', paid_at = NOW() WHERE order_no = 'TKT202401010001';

-- ===== DELETE（删除） =====
DELETE FROM users WHERE id = 100;            -- 删除单行
TRUNCATE TABLE logs;                         -- 清空表（速度快，但不可回滚）
```

### 2.4 JOIN：表连接

```sql
-- ===== INNER JOIN（内连接） =====
-- 只返回两表中匹配的行
SELECT u.username, o.order_no, o.total_amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- ===== LEFT JOIN（左连接） =====
-- 返回左表所有行，右表无匹配则为 NULL
SELECT e.title, o.order_no, o.status
FROM events e
LEFT JOIN orders o ON e.id = o.event_id;

-- ===== 多表连接 =====
SELECT u.username, e.title, o.total_amount, o.status
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN events e ON o.event_id = e.id
WHERE o.status = 'paid';

-- ===== 自连接 =====
-- 查找同一活动的所有订单
SELECT a.order_no, b.order_no
FROM orders a
JOIN orders b ON a.event_id = b.event_id AND a.id < b.id;
```text

### 2.5 子查询

```sql
-- ===== WHERE 子查询 =====
SELECT * FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE role = 'vip'
);

-- ===== EXISTS 子查询 =====
SELECT * FROM events e
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.event_id = e.id AND o.status = 'paid'
);

-- ===== FROM 子查询（派生表） =====
SELECT event_id, paid_count
FROM (
    SELECT event_id, COUNT(*) as paid_count
    FROM orders
    WHERE status = 'paid'
    GROUP BY event_id
) AS stats
WHERE paid_count > 10;
```

### 2.6 索引原理

```text
索引类型：
  B+ Tree 索引    → InnoDB 默认，适合范围查询
  Hash 索引       → 精确匹配快，不支持范围
  全文索引         → 全文搜索，如 MATCH AGAINST
  复合索引         → 多列组合，最左前缀原则

索引设计原则：
  1. 为 WHERE 条件列建索引
  2. 为 JOIN 连接列建索引
  3. 为 ORDER BY 列建索引（减少 filesort）
  4. 选择性高的列更适合索引（性别选择性差，不推荐）
  5. 复合索引遵循最左前缀：INDEX(a,b,c) 有效于 a, a+b, a+b+c
  6. 不要过度索引（写性能下降，占用空间）

EXPLAIN 分析查询：
  EXPLAIN SELECT * FROM orders WHERE user_id = 1;
  → type: ref          （索引查找）
  → rows: 10           （扫描行数）
  → Extra: Using index （索引覆盖）

索引失效场景：
  - 对索引列使用函数：WHERE YEAR(created_at) = 2024
  - 隐式类型转换：WHERE phone = 13800138000（phone 是字符串）
  - LIKE 以 % 开头：WHERE name LIKE '%张'
  - OR 条件中有非索引列
```

### 2.7 事务与隔离级别

```sql
-- ===== 事务控制 =====
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;     -- 提交
-- ROLLBACK; -- 回滚

-- ===== 隔离级别 =====
-- READ UNCOMMITTED   → 脏读、不可重复读、幻读（几乎不用）
-- READ COMMITTED     → 无脏读，有不可重复读/幻读（PostgreSQL 默认）
-- REPEATABLE READ    → 无脏读/不可重复读，有幻读（MySQL InnoDB 默认）
-- SERIALIZABLE       → 全部避免，性能最差
```text

---

## 3. Git 基础速查

### 3.1 常用命令

```bash
# ===== 配置 =====
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --global core.editor vim

# ===== 基本操作 =====
git init                                # 初始化仓库
git clone <url>                         # 克隆远程仓库
git status                              # 查看状态
git add <file>                          # 暂存文件
git add .                               # 暂存所有变更
git commit -m "提交信息"                 # 提交
git commit -am "提交信息"                # 暂存+提交（仅跟踪过的文件）

# ===== 分支操作 =====
git branch                              # 列出本地分支
git branch <branch-name>                # 创建分支
git checkout <branch-name>              # 切换分支
git checkout -b <branch-name>           # 创建并切换
git merge <branch-name>                 # 合并分支到当前
git branch -d <branch-name>             # 删除分支

# ===== 远程操作 =====
git remote -v                           # 查看远程仓库
git pull origin main                    # 拉取远程更新
git push origin main                    # 推送到远程
git push -u origin <branch>             # 推送并建立跟踪
git fetch origin                        # 获取远程信息（不合并）

# ===== 查看历史 =====
git log                                 # 提交历史
git log --oneline                       # 简洁历史
git log --graph --oneline --all         # 分支图
git diff                                # 工作区 vs 暂存区
git diff --staged                       # 暂存区 vs 上次提交
git show <commit-hash>                  # 查看某次提交详情

# ===== 撤销操作 =====
git restore <file>                      # 撤销工作区修改
git restore --staged <file>             # 取消暂存
git reset --soft HEAD~1                 # 撤销上一次提交（保留修改）
git reset --hard HEAD~1                 # 撤销上一次提交（丢弃修改）
git revert <commit-hash>                # 安全的撤销（创建新提交）
```

### 3.2 分支策略

```text
主分支模型（Git Flow 简化版）：

  main（生产分支）
    ↑ 合并发布
  develop（开发主分支）
    ↑ 合并完成的功能
  feature/xxx（功能分支）

工作流程：
  1. 从 main 创建功能分支：git checkout -b feature/add-login
  2. 开发并在功能分支上多次提交
  3. 合并回 main：git checkout main && git merge feature/add-login
  4. 推送：git push origin main

提交信息规范（Conventional Commits）：
  feat: 新功能
  fix: 修复 bug
  docs: 文档
  style: 代码格式
  refactor: 重构
  test: 测试
  chore: 构建/工具

  示例：feat: add user registration API
```

### 3.3 解决冲突

```bash
# 合并时出现冲突：
# <<<<<<< HEAD
# 当前分支的内容
# =======
# 合并分支的内容
# >>>>>>> feature-branch

# 手动编辑后：
git add <resolved-file>
git commit                         # 完成合并

# 取消合并（冲突太多时）：
git merge --abort
```text

---

## 4. HTTP/REST 基础

### 4.1 HTTP 请求方法

| 方法 | 用途 | 幂等 | 安全 | 请求体 | 响应体 |
| ------ | ------ | ------ | ------ | -------- | -------- |
| **GET** | 获取资源 | ✅ | ✅ | 无 | 资源 |
| **POST** | 创建资源 | ❌ | ❌ | 创建数据 | 新资源 |
| **PUT** | 整体更新/替换 | ✅ | ❌ | 完整资源 | 更新后资源 |
| **PATCH** | 部分更新 | ❌ | ❌ | 部分字段 | 更新后资源 |
| **DELETE** | 删除资源 | ✅ | ❌ | 无 | 删除结果 |
| **HEAD** | 获取响应头 | ✅ | ✅ | 无 | 无 |
| **OPTIONS** | 查询支持方法 | ✅ | ✅ | 无 | 允许的方法 |

**幂等**：同一操作执行多次和一次结果相同。
**安全**：操作不会修改资源状态。

### 4.2 HTTP 状态码

```

1xx 信息
  100 Continue        → 继续发送请求体

2xx 成功
  200 OK              → 请求成功
  201 Created         → 创建成功（POST）
  204 No Content      → 成功无返回体（DELETE）

3xx 重定向
  301 Moved Permanently  → 永久重定向
  302 Found              → 临时重定向
  304 Not Modified       → 缓存未修改

4xx 客户端错误
  400 Bad Request      → 请求参数错误
  401 Unauthorized     → 未认证（需要登录）
  403 Forbidden        → 无权限
  404 Not Found        → 资源不存在
  409 Conflict         → 资源冲突（如重复注册）
  422 Unprocessable Entity → 校验失败
  429 Too Many Requests   → 请求频率超限

5xx 服务器错误
  500 Internal Server Error → 服务器内部错误
  502 Bad Gateway           → 网关错误
  503 Service Unavailable   → 服务暂时不可用
  504 Gateway Timeout       → 网关超时

```text

### 4.3 RESTful 设计原则

```

URL 设计规范：

- 使用名词复数表示资源：/users, /events, /orders
- 使用 HTTP 方法表示操作：GET/POST/PUT/DELETE
- 层级关系用路径表示：/events/{id}/tickets
- 过滤/排序/分页用查询参数：?page=1&size=20&sort=created_at

示例：

  GET    /api/v1/events              # 活动列表
  POST   /api/v1/events              # 创建活动
  GET    /api/v1/events/{id}         # 活动详情
  PUT    /api/v1/events/{id}         # 更新活动
  DELETE /api/v1/events/{id}         # 删除活动

  GET    /api/v1/events/{id}/tickets # 活动门票列表
  POST   /api/v1/orders              # 创建订单
  GET    /api/v1/orders              # 订单列表
  GET    /api/v1/orders/{id}         # 订单详情

响应格式：
  {
    "code": 200,
    "message": "success",
    "data": { ... },
    "meta": {
      "page": 1,
      "size": 20,
      "total": 100
    }
  }

错误响应：
  {
    "detail": "订单不存在",
    "error_code": "ORDER_NOT_FOUND"
  }

```text

### 4.4 HTTP Headers 常见用法

```

请求头：
  Authorization: Bearer <token>       # JWT 认证
  Content-Type: application/json       # JSON 请求体
  Content-Type: multipart/form-data    # 文件上传
  Accept: application/json             # 期望响应格式
  X-Request-ID: uuid                   # 请求追踪 ID

响应头：
  Content-Type: application/json       # 响应格式
  Cache-Control: max-age=3600          # 缓存控制
  RateLimit-Remaining: 95              # 速率限制剩余次数
  X-Request-ID: uuid                   # 请求追踪 ID

```text

---

## 5. 原理深入

### 5.1 Python async/await 原理

```

事件循环（Event Loop）是异步编程的核心：

- 单线程并发执行多个协程
- 遇到 await 时挂起当前协程，切换到其他协程
- I/O 操作不阻塞线程，由操作系统回调通知完成

协程 vs 线程：
  协程：用户态切换，极轻量（KB级），十万级并发
  线程：内核态切换，较重（MB级），千级并发

await 的本质：
  await = "我在这里等待结果，但不阻塞线程，先去执行别的任务"

```text

### 5.2 B+ Tree 索引原理

```

B+ Tree 特点：

- 只有叶子节点存储数据
- 非叶子节点只存储索引键
- 叶子节点形成有序链表（范围查询高效）
- 树高度通常 3-4 层（百万级数据仅需 3-4 次 I/O）

聚簇索引 vs 非聚簇索引：
  InnoDB 中，主键是聚簇索引：

- 聚簇索引：叶子节点直接存储行数据（主键查找一次 I/O）
- 非聚簇索引（二级索引）：叶子节点存储主键值（回表查询）
- 覆盖索引：索引中包含所有需要字段，无需回表

```text

---

## 学习路径参考

| 知识点 | 对应章节 | 重要性 |
| -------- | --------- | -------- |
| Python 基础 | 第 29 章（本章） | ⭐⭐⭐⭐⭐ |
| SQL 基础 | 第 02 章 | ⭐⭐⭐⭐⭐ |
| Git 基础 | 第 01 章环境搭建 | ⭐⭐⭐⭐ |
| HTTP/REST | 第 04、05 章 | ⭐⭐⭐⭐⭐ |
| FastAPI | 第 04 章 | ⭐⭐⭐⭐⭐ |
| SQLAlchemy | 第 03 章 | ⭐⭐⭐⭐⭐ |
| Docker | 第 14 章 | ⭐⭐⭐⭐ |
