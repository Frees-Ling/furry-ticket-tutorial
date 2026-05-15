# 第2章：SQL/MySQL 完全教程 + 数据库设计

## 本章学习目标

本章将从零开始学习 SQL 和 MySQL 8.0 的全部核心知识，最终设计出票务系统的完整数据库 Schema。

**学习路径：**

1. ✅ SQL 概述与 MySQL 环境准备
2. ✅ DDL 数据定义语言（完整语法/数据类型/约束）
3. ✅ DML 数据操作语言（INSERT/SELECT/UPDATE/DELETE 完整语法图）
4. ✅ JOIN 连接查询（集合论基础 + 全部连接类型 + 连接算法）
5. ✅ 子查询与集合操作（EXISTS/IN/ANY/ALL + UNION/INTERSECT/EXCEPT）
6. ✅ 数据库规范化理论（1NF/2NF/3NF/BCNF + 函数依赖 + Armstrong 公理 + 数学证明）
7. ✅ MySQL 存储引擎深度解析（InnoDB 架构）
8. ✅ 索引完全教程（B+Tree/EXPLAIN/复合索引/覆盖索引）
9. ✅ 事务与隔离级别（ACID/MVCC/隔离性证明）
10. ✅ 票务系统数据库设计（四表 Schema + 3NF 证明）

---

## 1. SQL 概述与 MySQL 环境准备

### 1.1 什么是 SQL？

SQL（Structured Query Language，结构化查询语言）是操作关系型数据库的标准语言。它诞生于 1974 年由 IBM 的 Donald D. Chamberlin 和 Raymond F. Boyce 设计，历经数十年发展已成为数据库领域的通用语言。

SQL 按功能分为六大部分：

| 子语言 | 全称 | 功能 |
| -------- | ------ | ------ |
| **DDL** | Data Definition Language | 定义数据库结构：CREATE、DROP、ALTER |
| **DML** | Data Manipulation Language | 操作数据：INSERT、SELECT、UPDATE、DELETE |
| **DCL** | Data Control Language | 权限控制：GRANT、REVOKE |
| **TCL** | Transaction Control Language | 事务控制：BEGIN、COMMIT、ROLLBACK、SAVEPOINT |
| **DQL** | Data Query Language | 查询数据（通常归入 DML） |
| **S-CL** | Session Control Language | 会话控制：ALTER SESSION |

### 1.2 SQL 语法规则

```sql
-- SQL 关键字不区分大小写（但习惯上关键字大写）
-- 字符串用单引号 'text'
-- 标识符（表名/列名）可用反引号 ` 包裹
-- 分号 ; 结束一条语句

-- 注释风格：
# 这是单行注释（MySQL 特有）
-- 这也是单行注释（SQL 标准）
/* 多行
   注释 */
```text

### 1.3 连接 MySQL

在 Ch01 中我们已经通过 Docker Compose 启动了 MySQL 容器：

```bash
# 查看容器状态
docker ps

# 进入 MySQL 容器内部执行 SQL
docker exec -it ticket-mysql mysql -uroot -proot123 ticket_dev

# 或者用 Navicat/MySQL Workbench 图形化连接
# Host: localhost, Port: 3306, User: root, Password: root123
```

进入 MySQL 后看到 `mysql>` 提示符即表示连接成功。

```sql
-- 查看当前版本
SELECT VERSION();

-- 查看当前数据库
SELECT DATABASE();

-- 列出所有数据库
SHOW DATABASES;
```text

---

## 2. DDL 数据定义语言（完整语法）

### 2.1 数据库操作

```sql
-- 创建数据库
CREATE DATABASE ticket_dev
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- 查看创建数据库的语句
SHOW CREATE DATABASE ticket_dev;

-- 修改数据库
ALTER DATABASE ticket_dev
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

-- 删除数据库（危险！不可恢复，除非有备份）
DROP DATABASE ticket_dev;

-- 使用数据库
USE ticket_dev;
```

**字符集说明：**

- `utf8mb4`：真正的 UTF-8 编码，每字符最多 4 字节，支持 Emoji 😊 和生僻汉字
- `utf8mb4_unicode_ci`：基于 Unicode 标准排序规则，支持多语言正确排序

> **为什么不用 `utf8`？** MySQL 的 `utf8` 是假的 UTF-8，每个字符最多只支持 3 字节，无法存储 Emoji 和一些 CJK 扩展汉字。**务必使用 `utf8mb4`。**

### 2.2 MySQL 全部数据类型详解

#### 2.2.1 数值类型

| 类型 | 存储空间 | 最小值 | 最大值 | 用途 |
| ------ | --------- | -------- | -------- | ------ |
| `TINYINT` | 1 字节 | -128 | 127 | 状态码、布尔值 |
| `SMALLINT` | 2 字节 | -32768 | 32767 | 小范围统计 |
| `MEDIUMINT` | 3 字节 | -8388608 | 8388607 | 中范围计数 |
| `INT` | 4 字节 | -2147483648 | 2147483647 | 常规整数（用户ID、订单号） |
| `BIGINT` | 8 字节 | -2^63 | 2^63-1 | 大整数（雪花ID、流水号） |
| `DECIMAL(M,D)` | 变长 | - | - | **精确小数（金额必须用此类型）** |
| `FLOAT` | 4 字节 | - | - | 近似小数（科学计算） |
| `DOUBLE` | 8 字节 | - | - | 双精度小数 |

**DECIMAL 详解：**

```sql
-- DECIMAL(10,2) 表示总共 10 位数，其中 2 位小数
-- 即：12345678.99（8位整数 + 2位小数）
-- 金额场景必须使用 DECIMAL，绝不能使用 FLOAT/DOUBLE！

-- 原因：FLOAT 是 IEEE 754 近似存储
-- 0.1 + 0.2 = 0.30000000000000004（浮点精度误差）
-- DECIMAL 是定点数，精确存储
-- 0.1 + 0.2 = 0.30（精确）
```text

**UNSIGNED 属性：**

```sql
-- UNSIGNED 表示无符号，只能存非负数
-- 例如年龄用 TINYINT UNSIGNED（0-255）
age TINYINT UNSIGNED

-- MySQL 8.0.17 起不建议对 INT 使用 UNSIGNED
-- 推荐用 CHECK 约束替代
age TINYINT CHECK (age >= 0 AND age <= 150)
```

**ZEROFILL 已弃用：** MySQL 8.0.17 起标记废弃，不再使用。

#### 2.2.2 字符串类型

| 类型 | 最大长度 | 存储 | 用途 |
| ------ | --------- | ------ | ------ |
| `CHAR(N)` | 255 字符 | 固定长度，不足补空格 | 定长数据：手机号、身份证 |
| `VARCHAR(N)` | 65535 字节 | 变长，额外1-2字节记录长度 | 变长文本：用户名、邮箱 |
| `TINYTEXT` | 255 字节 | 变长 | 短文本 |
| `TEXT` | 65535 字节 | 变长 | 长文本 |
| `MEDIUMTEXT` | 16MB | 变长 | 中长文本 |
| `LONGTEXT` | 4GB | 变长 | 超大文本 |
| `BINARY(N)` | 255 字节 | 定长二进制 | 二进制数据 |
| `VARBINARY(N)` | 65535 字节 | 变长二进制 | 变长二进制 |
| `BLOB` | 65535 字节 | 二进制大对象 | 图片、文件 |
| `ENUM('a','b','c')` | 65535 个值 | 1-2 字节 | 枚举单选 |
| `SET('a','b','c')` | 64 个值 | 1-8 字节 | 集合多选 |

**VARCHAR vs CHAR 选择原则：**

```sql
-- CHAR 适合定长数据
phone CHAR(11)       -- 手机号固定11位
id_card CHAR(18)     -- 身份证固定18位

-- VARCHAR 适合变长数据
username VARCHAR(50) -- 用户名长度不定
email VARCHAR(255)   -- 邮箱长度不定
```text

**TEXT 限制：**

- TEXT 列不能有默认值
- TEXT 列不能在索引中指定前缀长度之外的全文搜索
- 排序时使用 max_sort_length 指定的前 N 字节

#### 2.2.3 日期时间类型

| 类型 | 存储空间 | 范围 | 精度 | 用途 |
| ------ | --------- | ------ | ------ | ------ |
| `DATE` | 3 字节 | 1000-01-01 ~ 9999-12-31 | 天 | 生日、日期 |
| `TIME` | 3 字节 | -838:59:59 ~ 838:59:59 | 秒(可到微秒) | 时间段 |
| `DATETIME` | 5 字节 | 1000-01-01 ~ 9999-12-31 | 秒(可到微秒) | **通用日期时间** |
| `TIMESTAMP` | 4 字节 | 1970-01-01 ~ 2038-01-19 | 秒(可到微秒) | 自动记录修改时间 |
| `YEAR` | 1 字节 | 1901 ~ 2155 | 年 | 年份 |

**DATETIME vs TIMESTAMP 选择：**

```sql
-- DATETIME：推荐，范围大，不受时区影响
created_at DATETIME(3)  -- (3)表示毫秒精度

-- TIMESTAMP：受 time_zone 影响，自动转换为 UTC
-- 2038 年问题（Year 2038 Problem）：会溢出！
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

-- 最佳实践：统一使用 DATETIME(3)
created_at DATETIME(3) DEFAULT CURRENT_TIMESTAMP(3),
updated_at DATETIME(3) DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
```

#### 2.2.4 JSON 类型（MySQL 5.7+）

```sql
-- JSON 列自动验证 JSON 格式合法性
extra_info JSON

-- 插入
INSERT INTO table VALUES ('{"key": "value", "tags": ["a", "b"]}');

-- 查询 JSON 字段
SELECT extra_info->'$.key' FROM table;          -- 返回 "value"（带引号）
SELECT JSON_EXTRACT(extra_info, '$.key') FROM table;
SELECT extra_info->>'$.key' FROM table;         -- 返回 value（去引号）

-- JSON 作为虚拟列建立索引
ALTER TABLE table ADD COLUMN key_virtual VARCHAR(50)
    GENERATED ALWAYS AS (extra_info->>'$.key') STORED;
CREATE INDEX idx_key ON table(key_virtual);
```text

#### 2.2.5 布尔类型

```sql
-- MySQL 没有真正的 BOOL 类型，TINYINT(1) 的别名
is_active BOOL     -- 等价于 TINYINT(1)
is_deleted BOOLEAN -- 等价于 TINYINT(1)
-- 存储时：0=false，非0=true
```

### 2.3 约束（Constraints）完整语法

```sql
CREATE TABLE 表名 (
    列名 数据类型 [约束] [DEFAULT 默认值] [COMMENT '注释'],
    ...
    [表级约束]
);

-- ========== 约束类型 ==========

-- 1. PRIMARY KEY（主键）：唯一标识一行，自动 NOT NULL + UNIQUE
id BIGINT PRIMARY KEY AUTO_INCREMENT,       -- 列级定义
-- 或表级定义：
PRIMARY KEY (id),

-- 2. FOREIGN KEY（外键）：引用另一张表的行
user_id BIGINT,
FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE    -- 级联删除：用户删除时，关联记录也删除
    ON UPDATE CASCADE,   -- 级联更新：用户ID更新时，关联记录也更新

-- ON DELETE 选项：
--   CASCADE：级联删除
--   SET NULL：外键列设为 NULL
--   RESTRICT：拒绝删除（默认，MySQL 8.0 默认行为）
--   NO ACTION：同 RESTRICT

-- 3. UNIQUE（唯一）：值不能重复
email VARCHAR(255) UNIQUE,
-- 复合唯一：
UNIQUE KEY uk_user_event (user_id, event_id),

-- 4. NOT NULL：值不能为空
username VARCHAR(50) NOT NULL,

-- 5. DEFAULT：默认值
status VARCHAR(20) DEFAULT 'pending',
created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

-- 6. CHECK（MySQL 8.0.16+ 真正生效）：值必须满足条件
price DECIMAL(10,2) CHECK (price >= 0),
status VARCHAR(20) CHECK (status IN ('pending', 'paid', 'cancelled')),

-- 7. AUTO_INCREMENT：自增（仅整数类型）
id BIGINT AUTO_INCREMENT,
-- 自增起始值和步长：
AUTO_INCREMENT = 1000,
-- 或全局设置：
-- SET @@auto_increment_increment = 2;  -- 步长
-- SET @@auto_increment_offset = 1;     -- 起始
```text

**索引键（索引）：**

```sql
-- 索引是在 CREATE TABLE 时一并定义的索引
INDEX idx_username (username),           -- 普通索引
UNIQUE KEY uk_email (email),             -- 唯一索引
FULLTEXT KEY ft_content (content),       -- 全文索引
INDEX idx_user_status (user_id, status), -- 复合索引
```

### 2.4 完整 CREATE TABLE 语法

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] 表名 (
    列定义 [, 列定义 ...]
    [, 表级约束 ...]
    [, INDEX 索引定义 ...]
) [ENGINE = InnoDB]
  [AUTO_INCREMENT = N]
  [DEFAULT CHARSET = utf8mb4]
  [COLLATE = utf8mb4_unicode_ci]
  [COMMENT = '表注释'];
```text

### 2.5 修改表结构（ALTER TABLE）

```sql
-- 添加列
ALTER TABLE users ADD COLUMN avatar VARCHAR(500) AFTER username;

-- 修改列类型
ALTER TABLE users MODIFY COLUMN username VARCHAR(100) NOT NULL;

-- 重命名列
ALTER TABLE users CHANGE COLUMN username user_name VARCHAR(50);

-- 删除列
ALTER TABLE users DROP COLUMN avatar;

-- 添加索引
ALTER TABLE users ADD INDEX idx_email (email);

-- 删除索引
ALTER TABLE users DROP INDEX idx_email;

-- 添加外键
ALTER TABLE orders ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id);

-- 删除外键
ALTER TABLE orders DROP FOREIGN KEY fk_orders_user;

-- 重命名表
ALTER TABLE users RENAME TO members;
RENAME TABLE members TO users;  -- 另一种写法
```

### 2.6 TRUNCATE vs DELETE

```sql
-- TRUNCATE：删除所有行，DDL，不可回滚，重置 AUTO_INCREMENT
TRUNCATE TABLE temp_orders;

-- DELETE：逐行删除，DML，可回滚，不重置 AUTO_INCREMENT
DELETE FROM temp_orders WHERE status = 'cancelled';
```text

| 对比项 | TRUNCATE | DELETE |
| -------- | ---------- | -------- |
| 类型 | DDL（数据定义语言） | DML（数据操作语言） |
| 速度 | 极快（直接删除表重建） | 慢（逐行加锁删除） |
| 回滚 | 不可回滚 | 在事务内可回滚 |
| WHERE | 不支持 | 支持条件删除 |
| AUTO_INCREMENT | 重置 | 不重置 |
| 触发器 | 不触发 | 触发 DELETE 触发器 |

---

### 2.7 视图（VIEW）

视图（View）是基于 SQL 查询的虚拟表，不存储数据本身，只保存查询定义。每次使用视图时实时执行底层查询。

#### 2.7.1 创建与使用视图

```sql
-- 语法
CREATE [OR REPLACE] VIEW 视图名 [(列名列表)] AS
SELECT 查询
[WITH CHECK OPTION];

-- 示例：活动概览视图（隐藏敏感字段，封装复杂 JOIN）
CREATE VIEW event_summary AS
SELECT
    e.id,
    e.title,
    e.category,
    e.total_stock,
    e.sold_count,
    (e.total_stock - e.sold_count) AS available_stock,
    e.price,
    e.status,
    u.username AS creator_name
FROM events e
JOIN users u ON e.created_by = u.id
WHERE e.status != 'cancelled';

-- 使用视图（像普通表一样查询）
SELECT * FROM event_summary WHERE category = 'concert';
SELECT * FROM event_summary WHERE available_stock > 0 ORDER BY price DESC;

-- OR REPLACE：修改视图定义
CREATE OR REPLACE VIEW event_summary AS
SELECT
    e.id, e.title, e.category,
    e.total_stock, e.sold_count,
    (e.total_stock - e.sold_count) AS available_stock,
    e.price, e.status, e.start_date,
    u.username AS creator_name
FROM events e
JOIN users u ON e.created_by = u.id;
```

#### 2.7.2 视图的价值

```text
1. 简化查询：将复杂 JOIN/聚合封装为虚拟表
2. 安全性：只暴露需要的列，隐藏 password_hash 等敏感字段
3. 逻辑层：应用层不依赖底层表结构，表结构调整时只需更新视图
4. 一致性：相同查询逻辑在不同地方复用，避免重复 SQL
```

```sql
-- 安全视图：对外暴露不包含敏感信息的用户数据
CREATE VIEW user_public AS
SELECT id, username, email, role, created_at
FROM users;
-- password_hash、phone 等字段不被暴露
```text

#### 2.7.3 可更新视图

```sql
-- 简单视图（基于单表、不含聚合/分组）支持 INSERT/UPDATE/DELETE
CREATE VIEW active_events AS
SELECT id, title, price, total_stock, status
FROM events
WHERE status IN ('draft', 'published');

-- 通过视图更新数据（实际修改底层 events 表）
UPDATE active_events SET price = 299 WHERE id = 1;

-- WITH CHECK OPTION：防止更新后数据不再满足视图条件
CREATE OR REPLACE VIEW active_events AS
SELECT id, title, price, total_stock, status
FROM events
WHERE status IN ('draft', 'published')
WITH CHECK OPTION;

-- 下面这条会报错！插入的 status='cancelled' 不符合视图的 WHERE 条件
INSERT INTO active_events (title, price, total_stock, status)
VALUES ('隐藏活动', 99, 50, 'cancelled');
-- ERROR 1369 (HY000): CHECK OPTION failed
```

#### 2.7.4 视图的限制与注意事项

```sql
-- 不可更新的视图类型：
--   - 包含 GROUP BY / HAVING / 聚合函数
--   - 包含 DISTINCT / UNION / UNION ALL
--   - 包含窗口函数
--   - FROM 子句包含不可更新视图

-- 性能：视图每次查询都执行底层 SQL，复杂视图可能很慢
-- MySQL 不支持物化视图（Materialized View），
-- 可用 表 + 事件/定时任务 手动刷新模拟

-- 嵌套视图：视图基于视图时，性能影响叠加，需注意
```text

#### 2.7.5 视图管理

```sql
-- 查看所有视图
SHOW FULL TABLES WHERE Table_type = 'VIEW';

-- 查看视图定义
SHOW CREATE VIEW event_summary;

-- 删除视图
DROP VIEW IF EXISTS event_summary;

-- 修改视图（等价于 CREATE OR REPLACE）
ALTER VIEW event_summary AS
SELECT id, title, status FROM events;
```

**票务系统实践：** 使用视图封装订单统计，报表查询不再写复杂 JOIN：

```sql
CREATE VIEW order_daily_stats AS
SELECT
    DATE_FORMAT(o.created_at, '%Y-%m-%d') AS 统计日期,
    e.title AS 活动名称,
    COUNT(*) AS 订单数,
    SUM(o.quantity) AS 出票数,
    SUM(o.total_amount) AS 总金额,
    AVG(o.total_amount) AS 客单价
FROM orders o
JOIN events e ON o.event_id = e.id
WHERE o.status = 'paid'
GROUP BY 统计日期, e.title;

-- 报表查询：一行 SQL 搞定
SELECT * FROM order_daily_stats
WHERE 统计日期 BETWEEN '2024-01-01' AND '2024-01-31'
ORDER BY 统计日期, 总金额 DESC;
```text

---

## 3. DML 数据操作语言（完整语法）

### 3.1 INSERT — 插入数据

```sql
-- 标准语法（推荐）
INSERT INTO users (username, email, password_hash)
VALUES ('张三', 'zhangsan@example.com', 'hashed_pwd_123');

-- 一次插入多行
INSERT INTO users (username, email, password_hash) VALUES
    ('李四', 'lisi@example.com', 'hashed_pwd_456'),
    ('王五', 'wangwu@example.com', 'hashed_pwd_789');

-- SET 语法
INSERT INTO users SET username = '赵六', email = 'zhao@example.com';

-- 从 SELECT 查询结果插入
INSERT INTO users_backup (username, email, password_hash)
SELECT username, email, password_hash FROM users WHERE deleted = 0;

-- ON DUPLICATE KEY UPDATE（唯一键冲突时更新）
INSERT INTO users (id, username, email)
VALUES (1, '张三', 'newemail@example.com')
ON DUPLICATE KEY UPDATE
    email = VALUES(email),           -- 发生冲突时更新 email
    updated_at = CURRENT_TIMESTAMP;

-- REPLACE（有则替换，无则插入）——注意：先 DELETE 再 INSERT，会重置 AUTO_INCREMENT
REPLACE INTO users (id, username, email)
VALUES (1, '张三', 'replace@example.com');

-- INSERT IGNORE（忽略冲突，不报错）
INSERT IGNORE INTO users (id, username) VALUES (1, '冲突会忽略');
```

### 3.2 SELECT — 查询数据（完整语法图）

```text
SELECT
    [DISTINCT | ALL]
    选择列表
    [FROM 表名 [JOIN 子句]]
    [WHERE 条件]
    [GROUP BY {列名 | 表达式} [WITH ROLLUP]]
    [HAVING 分组条件]         -- 必须与 GROUP BY 一起用
    [WINDOW 窗口定义]
    [ORDER BY 列名 [ASC | DESC]]
    [LIMIT {行数 | OFFSET 偏移量}]
    [FOR UPDATE | FOR SHARE]  -- 锁定读
```

#### 3.2.1 SELECT 基础

```sql
-- 查询所有列（生产环境尽量避免——可能导致索引失效和网络开销）
SELECT * FROM users;

-- 查询指定列
SELECT id, username, email FROM users;

-- 别名（AS 可省略）
SELECT id AS 用户ID, username AS 用户名 FROM users;
SELECT id 用户ID, username 用户名 FROM users;

-- 常量列
SELECT '票务系统' AS 系统名称, username FROM users;

-- 去重
SELECT DISTINCT status FROM orders;

-- 计算列
SELECT price * quantity AS total_price FROM order_items;

-- 函数
SELECT NOW() AS 当前时间;
SELECT COUNT(*) AS 总数 FROM users;
```text

#### 3.2.2 WHERE 子句 — 条件过滤

```sql
-- 比较运算符：=, !=, <>, >, <, >=, <=
SELECT * FROM orders WHERE total_amount >= 100;

-- NULL 判断（不能用 = NULL，要使用 IS NULL）
SELECT * FROM users WHERE email IS NULL;
SELECT * FROM users WHERE email IS NOT NULL;

-- 范围：BETWEEN（闭区间）
SELECT * FROM events WHERE price BETWEEN 100 AND 500;
-- 等价于：price >= 100 AND price <= 500

-- 集合：IN
SELECT * FROM orders WHERE status IN ('pending', 'paid');

-- 模式匹配：LIKE
-- % 匹配任意多个字符
-- _ 匹配单个字符
SELECT * FROM users WHERE username LIKE '张%';   -- 以"张"开头的
SELECT * FROM users WHERE email LIKE '%@example.com';  -- 特定邮箱域名
SELECT * FROM users WHERE username LIKE '张_';    -- "张"后面一个字符

-- 注意：LIKE 以 % 开头时无法使用索引（全表扫描）

-- 正则表达式：REGEXP（MySQL 8.0+）
SELECT * FROM users WHERE email REGEXP '^[a-z]+@example\.com$';

-- 逻辑运算：AND, OR, NOT
SELECT * FROM orders
WHERE status = 'paid'
  AND total_amount > 100
  AND created_at >= '2024-01-01';

-- 优先级：NOT > AND > OR，建议用括号明确
SELECT * FROM orders
WHERE (status = 'paid' OR status = 'pending')
  AND total_amount > 100;

-- 组合：IN + 子查询
SELECT * FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE is_vip = 1
);
```

#### 3.2.3 GROUP BY — 分组聚合

```sql
-- 按状态分组统计订单数
SELECT status, COUNT(*) AS order_count, SUM(total_amount) AS total
FROM orders
GROUP BY status;

-- 统计每个用户的总消费
SELECT user_id, COUNT(*) AS order_count, SUM(total_amount) AS total_spent
FROM orders
WHERE status = 'paid'
GROUP BY user_id
ORDER BY total_spent DESC;

-- GROUP BY 多个列
SELECT event_id, ticket_type, COUNT(*) AS sold_count
FROM tickets
GROUP BY event_id, ticket_type;

-- WITH ROLLUP：在分组基础上增加总计行
SELECT status, COUNT(*) AS cnt
FROM orders
GROUP BY status WITH ROLLUP;
-- 结果：
-- pending    5
-- paid      10
-- cancelled  2
-- NULL      17   ← 总计行
```text

#### 3.2.4 HAVING — 过滤分组

```sql
-- HAVING 在 GROUP BY 之后过滤（WHERE 在 GROUP BY 之前过滤）
-- 查出订单数超过 5 的用户
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING order_count >= 5;

-- 查出总消费超过 1000 的 VIP 用户（WHERE 过滤分组前数据）
SELECT user_id, COUNT(*) AS order_count, SUM(total_amount) AS total_spent
FROM orders
WHERE status = 'paid'     -- 先过滤：只算已支付的
GROUP BY user_id
HAVING total_spent > 1000;  -- 后过滤：总消费超过1000
```

#### 3.2.5 窗口函数（MySQL 8.0+）

窗口函数在不合并行的情况下执行聚合计算，是 SQL 进阶核心技能。

```sql
-- 窗口函数语法：
-- 函数() OVER (PARTITION BY 分组列 ORDER BY 排序列 窗口帧)

-- ROW_NUMBER：为每组内的行分配连续编号（1, 2, 3...）
SELECT
    username,
    total_amount,
    ROW_NUMBER() OVER (ORDER BY total_amount DESC) AS 排名
FROM orders;

-- RANK：排名相同则并列，后续跳过（1, 1, 3...）
SELECT
    username,
    total_amount,
    RANK() OVER (ORDER BY total_amount DESC) AS 排名
FROM orders;

-- DENSE_RANK：排名相同则并列，后续不跳过（1, 1, 2...）
SELECT
    username,
    total_amount,
    DENSE_RANK() OVER (ORDER BY total_amount DESC) AS 密集排名
FROM orders;

-- 分组内的排名：每个用户的订单按金额排名
SELECT
    user_id,
    total_amount,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total_amount DESC) AS 用户内排名
FROM orders;

-- NTILE：将数据分为 N 组
SELECT
    total_amount,
    NTILE(4) OVER (ORDER BY total_amount DESC) AS 四分位
FROM orders;

-- LAG/LEAD：访问前/后行
SELECT
    total_amount,
    LAG(total_amount, 1) OVER (ORDER BY created_at) AS 上一单金额,
    total_amount - LAG(total_amount, 1) OVER (ORDER BY created_at) AS 差额
FROM orders
WHERE user_id = 1;

-- FIRST_VALUE/LAST_VALUE：组的第一个/最后一个值
SELECT
    user_id,
    total_amount,
    FIRST_VALUE(total_amount) OVER (PARTITION BY user_id ORDER BY created_at) AS 首单金额,
    LAST_VALUE(total_amount) OVER (
        PARTITION BY user_id ORDER BY created_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS 末单金额
FROM orders;

-- 窗口帧详解
-- ROWS BETWEEN ... AND ... 可以指定：
--   UNBOUNDED PRECEDING   — 分组第一行
--   N PRECEDING           — 前 N 行
--   CURRENT ROW           — 当前行
--   N FOLLOWING           — 后 N 行
--   UNBOUNDED FOLLOWING   — 分组最后一行

-- 移动平均：前3单加上当前单的平均金额
SELECT
    total_amount,
    AVG(total_amount) OVER (
        ORDER BY created_at
        ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
    ) AS moving_avg_4
FROM orders
WHERE user_id = 1;
```text

#### 3.2.6 ORDER BY — 排序

```sql
-- ASC（默认升序），DESC（降序）
SELECT * FROM events ORDER BY price ASC;
SELECT * FROM events ORDER BY price DESC;

-- 多列排序
SELECT * FROM events
ORDER BY category ASC, price DESC;

-- 使用字段序号
SELECT username, email FROM users ORDER BY 2;  -- 按第2列(email)排序

-- NULL 排序
-- MySQL 中 NULL 默认视为最小值（ASC 时排第一）
-- NULLS FIRST / NULLS LAST（MySQL 8.0+）
SELECT * FROM users ORDER BY email ASC NULLS LAST;
```

#### 3.2.7 LIMIT — 分页

```sql
-- 取前10行
SELECT * FROM events LIMIT 10;

-- 分页：跳过前20条，取10条（第3页，每页10条）
SELECT * FROM events LIMIT 10 OFFSET 20;

-- 简写语法：LIMIT 偏移量, 行数
SELECT * FROM events LIMIT 20, 10;

-- 注意：OFFSET 越大越慢！因为数据库仍然要扫描并丢弃前面的行
-- 大数据量分页推荐使用游标分页（Keyset Pagination）：
SELECT * FROM events
WHERE id > 1000   -- 上一页的最后一个 ID
ORDER BY id
LIMIT 10;
```text

#### 3.2.8 FOR UPDATE — 锁定读

```sql
-- 悲观锁：锁定选中的行，其他事务无法修改直到事务结束
BEGIN;
SELECT * FROM inventory
WHERE event_id = 1
FOR UPDATE;  -- 加行级排他锁
-- 执行扣减库存...
UPDATE inventory SET stock = stock - 1 WHERE event_id = 1;
COMMIT;  -- 释放锁

-- FOR SHARE：共享锁，其他事务可读但不能修改（MySQL 8.0+，代替 LOCK IN SHARE MODE）
SELECT * FROM inventory
WHERE event_id = 1
FOR SHARE;

-- 跳过锁定行（Skip Locked，MySQL 8.0+）
-- 适合任务队列场景：多个 worker 竞争任务
BEGIN;
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY priority DESC
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- 只获取未被其他事务锁定的行
```

### 3.3 UPDATE — 更新数据

```sql
-- 标准语法
UPDATE users SET email = 'new@example.com' WHERE id = 1;

-- 更新多列
UPDATE orders SET
    status = 'confirmed',
    confirmed_at = CURRENT_TIMESTAMP
WHERE id = 1001;

-- 多表更新（基于另一张表的值）
UPDATE orders o
JOIN users u ON o.user_id = u.id
SET o.status = 'cancelled'
WHERE u.is_banned = 1;

-- 使用子查询更新
UPDATE events
SET sold_count = (
    SELECT COUNT(*) FROM tickets WHERE event_id = events.id
)
WHERE id = 1;

-- 注意：不加 WHERE 会更新全表！
UPDATE users SET status = 'active';  -- 所有用户都变成 active
```text

### 3.4 DELETE — 删除数据

```sql
-- 标准语法
DELETE FROM orders WHERE id = 1001;

-- 多表删除
DELETE o FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.is_banned = 1;

-- 使用子查询删除
DELETE FROM tickets
WHERE order_id IN (
    SELECT id FROM orders WHERE status = 'cancelled'
);

-- 注意：不加 WHERE 会清空全表！
-- DELETE FROM users;  -- 危险！
```

---

### 3.5 条件表达式（CASE WHEN）

CASE WHEN 是 SQL 中的条件表达式，相当于编程语言中的 if/else，可在 SELECT、WHERE、ORDER BY、UPDATE 等语句中使用。

#### 3.5.1 两种语法形式

```sql
-- 形式1：简单 CASE（等值比较）
CASE 表达式
    WHEN 值1 THEN 结果1
    WHEN 值2 THEN 结果2
    ELSE 默认结果
END

-- 形式2：搜索 CASE（条件比较，更灵活）
CASE
    WHEN 条件1 THEN 结果1
    WHEN 条件2 THEN 结果2
    ELSE 默认结果
END
```text

#### 3.5.2 在 SELECT 中做值转换

```sql
-- 将状态码转为中文（形式1：等值比较）
SELECT
    id,
    order_no,
    CASE status
        WHEN 'pending_payment' THEN '待支付'
        WHEN 'paid'           THEN '已支付'
        WHEN 'cancelled'      THEN '已取消'
        WHEN 'refunded'       THEN '已退款'
        ELSE '未知状态'
    END AS 状态名称
FROM orders;

-- 按金额分等级（形式2：条件比较）
SELECT
    id,
    total_amount,
    CASE
        WHEN total_amount >= 1000 THEN '大额'
        WHEN total_amount >= 100  THEN '中额'
        WHEN total_amount > 0     THEN '小额'
        ELSE '免费/异常'
    END AS 订单等级
FROM orders
WHERE created_at >= '2024-01-01';
```

#### 3.5.3 在 GROUP BY 中做分组

```sql
-- 按价格区间统计活动数量
SELECT
    CASE
        WHEN price < 100   THEN '低价票 (<100)'
        WHEN price < 500   THEN '中价票 (100-500)'
        ELSE '高价票 (>500)'
    END AS 价格区间,
    COUNT(*) AS 活动数量,
    ROUND(AVG(price), 2) AS 平均价格
FROM events
WHERE status = 'published'
GROUP BY 价格区间
ORDER BY 价格区间;
```text

#### 3.5.4 在 ORDER BY 中自定义排序

```sql
-- 自定义状态排序：已支付排最前，已取消排最后
SELECT id, status, total_amount, created_at
FROM orders
WHERE event_id = 1
ORDER BY
    CASE status
        WHEN 'paid'             THEN 1
        WHEN 'pending_payment'  THEN 2
        WHEN 'refunded'         THEN 3
        WHEN 'cancelled'        THEN 4
    END,
    created_at DESC;
```

#### 3.5.5 在 UPDATE 中做条件更新

```sql
-- 根据不同条件批量更新订单状态
UPDATE orders
SET status = CASE
    WHEN paid_at IS NOT NULL AND status = 'paid'      THEN 'completed'
    WHEN created_at < NOW() - INTERVAL 30 MINUTE
         AND status = 'pending_payment'                THEN 'cancelled'
    ELSE status
END
WHERE event_id = 1;

-- 票务系统实践：不同分类活动不同调价幅度
UPDATE events
SET price = CASE category
    WHEN 'concert' THEN ROUND(price * 1.10, 2)
    WHEN 'sports'  THEN ROUND(price * 0.95, 2)
    WHEN 'theater' THEN ROUND(price * 1.05, 2)
    ELSE price
END
WHERE status = 'published';
```text

#### 3.5.6 CASE WHEN 与聚合函数结合（行转列）

```sql
-- 行转列（Pivot）：一行展示多个状态的统计
SELECT
    COUNT(*) AS 总订单,
    SUM(CASE WHEN status = 'paid'             THEN 1 ELSE 0 END) AS 已支付,
    SUM(CASE WHEN status = 'pending_payment'  THEN 1 ELSE 0 END) AS 待支付,
    SUM(CASE WHEN status = 'cancelled'        THEN 1 ELSE 0 END) AS 已取消,
    SUM(CASE WHEN status = 'refunded'         THEN 1 ELSE 0 END) AS 已退款
FROM orders
WHERE created_at >= '2024-01-01';

-- 按用户统计
SELECT
    user_id,
    COUNT(*) AS 总订单,
    SUM(CASE WHEN status = 'paid' THEN total_amount ELSE 0 END) AS 已支付金额,
    COUNT(CASE WHEN status = 'pending_payment' THEN 1 END) AS 待支付笔数
FROM orders
GROUP BY user_id;

-- 周度销售统计（按星期几做列）
SELECT
    DATE_FORMAT(created_at, '%Y-%u') AS 周数,
    SUM(CASE WHEN DAYOFWEEK(created_at) = 1 THEN total_amount ELSE 0 END) AS 周日,
    SUM(CASE WHEN DAYOFWEEK(created_at) = 2 THEN total_amount ELSE 0 END) AS 周一,
    SUM(CASE WHEN DAYOFWEEK(created_at) = 3 THEN total_amount ELSE 0 END) AS 周二,
    SUM(CASE WHEN DAYOFWEEK(created_at) = 4 THEN total_amount ELSE 0 END) AS 周三,
    SUM(CASE WHEN DAYOFWEEK(created_at) = 5 THEN total_amount ELSE 0 END) AS 周四,
    SUM(CASE WHEN DAYOFWEEK(created_at) = 6 THEN total_amount ELSE 0 END) AS 周五,
    SUM(CASE WHEN DAYOFWEEK(created_at) = 7 THEN total_amount ELSE 0 END) AS 周六
FROM orders
WHERE status = 'paid'
GROUP BY 周数
ORDER BY 周数;
```

#### 3.5.7 CASE WHEN 的 NULL 处理

```sql
-- 简单 CASE 做等值比较：NULL = NULL 不成立
SELECT
    CASE status
        WHEN NULL THEN '无状态'   -- 永远不会匹配！
        ELSE status
    END
FROM orders;

-- 正确写法：使用搜索 CASE 或 IS NULL
SELECT
    CASE
        WHEN status IS NULL THEN '无状态'
        ELSE status
    END
FROM orders;
```text

---

## 4. JOIN 连接查询

### 4.1 集合论基础

关系型数据库的本质是集合论的应用。在理解 JOIN 之前，需要掌握三个基本概念：

**笛卡尔积（Cartesian Product）：**

- 集合 A = {a, b}，集合 B = {1, 2, 3}
- A × B = {(a,1), (a,2), (a,3), (b,1), (b,2), (b,3)}
- 共 2 × 3 = 6 个元素

SQL 中不带 WHERE 的 JOIN 就是笛卡尔积（交叉连接）：

```sql
SELECT * FROM users, orders;  -- 隐式 CROSS JOIN
```

**关系代数（Relational Algebra）：**
E. F. Codd 在 1970 年提出的理论基础，8 种基本运算：

| 运算 | 符号 | SQL 对应 |
| ------ | ------ | --------- |
| 选择（Selection） | σ | WHERE |
| 投影（Projection） | π | SELECT 列 |
| 连接（Join） | ⋈ | JOIN |
| 重命名（Rename） | ρ | AS |
| 并（Union） | ∪ | UNION |
| 差（Difference） | − | EXCEPT |
| 交（Intersection） | ∩ | INTERSECT |
| 笛卡尔积（Product） | × | CROSS JOIN |

**Venn 图理解 JOIN：**

```text
左表 A               右表 B
┌──────────┐        ┌──────────┐
│  id name │        │  id order│
│  1 张三  │        │  1 O001  │
│  2 李四  │        │  3 O002  │
│  3 王五  │        │  4 O003  │
└──────────┘        └──────────┘

INNER JOIN → 只取交集：id=1(A)+id=3(A), id=1(B)+id=3(B)
LEFT JOIN  → A 全部 + 匹配的 B
RIGHT JOIN → B 全部 + 匹配的 A
FULL JOIN  → A 全部 + B 全部（MySQL 不支持，用 UNION 模拟）
```

### 4.2 全部连接类型

```sql
-- ========== 准备测试数据 ==========
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);
INSERT INTO students VALUES (1, '张三'), (2, '李四'), (3, '王五');

CREATE TABLE scores (
    student_id INT,
    score INT,
    subject VARCHAR(50)
);
INSERT INTO scores VALUES (1, 90, '数学'), (2, 85, '英语'), (4, 95, '物理');
```text

#### 4.2.1 INNER JOIN（内连接）

```sql
-- 只返回两表匹配的行
SELECT s.name, sc.score, sc.subject
FROM students s
INNER JOIN scores sc ON s.id = sc.student_id;

-- 结果：张三(90)、李四(85)
-- 王五没有成绩→不出现；物理成绩没有对应学生→不出现
```

#### 4.2.2 LEFT JOIN（左外连接）

```sql
-- 返回左表所有行，右表无匹配则填充 NULL
SELECT s.name, sc.score, sc.subject
FROM students s
LEFT JOIN scores sc ON s.id = sc.student_id;

-- 结果：张三(90)、李四(85)、王五(NULL)
```text

#### 4.2.3 RIGHT JOIN（右外连接）

```sql
-- 返回右表所有行，左表无匹配则填充 NULL
SELECT s.name, sc.score, sc.subject
FROM students s
RIGHT JOIN scores sc ON s.id = sc.student_id;

-- 结果：张三(90)、李四(85)、NULL(95)
```

#### 4.2.4 FULL OUTER JOIN（全外连接）

```sql
-- MySQL 不支持 FULL OUTER JOIN，使用 UNION 模拟
SELECT s.name, sc.score, sc.subject
FROM students s
LEFT JOIN scores sc ON s.id = sc.student_id
UNION
SELECT s.name, sc.score, sc.subject
FROM students s
RIGHT JOIN scores sc ON s.id = sc.student_id;

-- 结果：张三(90)、李四(85)、王五(NULL)、NULL(95)
```text

#### 4.2.5 CROSS JOIN（交叉连接）

```sql
-- 返回笛卡尔积：左表 N 行 × 右表 M 行
SELECT s.name, sc.subject
FROM students s
CROSS JOIN scores sc;
-- 3 × 3 = 9 行
```

#### 4.2.6 SELF JOIN（自连接）

```sql
-- 表与自身连接，必须使用别名
-- 场景：员工表，找上级
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    manager_id INT
);

SELECT e1.name AS 员工, e2.name AS 上级
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id;
```text

#### 4.2.7 NATURAL JOIN（自然连接）

```sql
-- 自动按同名列连接（不推荐——隐式依赖列名，容易出错）
SELECT * FROM students NATURAL JOIN scores;
-- 等价于 ON students.id = scores.student_id
-- 如果还有同名列如 name，也会自动连接！
```

#### 4.2.8 STRAIGHT_JOIN

```sql
-- 强制左表为驱动表（强制固定连接顺序）
-- 某些优化器选择错误时使用
SELECT * FROM students s
STRAIGHT_JOIN scores sc ON s.id = sc.student_id;
```text

### 4.3 连接算法（MySQL 内部实现）

理解连接算法有助于写出高效的查询。

#### 4.3.1 Nested Loop Join（NLJ）

最简单但最慢的算法，适用于小表：

```

for each row r in R:          -- 外层循环（驱动表）
    for each row s in S:      -- 内层循环（被驱动表）
        if r.id = s.r_id:
            output (r, s)

```text

时间复杂度：O(|R| × |S|)

如果 R 有 10000 行，S 有 10000 行，需要 1 亿次比较。

**优化：** 内层表使用索引，复杂度降为 O(|R| × log|S|)

#### 4.3.2 Block Nested Loop Join（BNL）

MySQL 优化：一次读取一批（block）驱动表行到 join buffer：

```

for each block of R (size = join_buffer_size):
    for each row s in S:
        for each row r in block:
            if r.id = s.r_id:
                output (r, s)

```text

减少内层表的扫描次数，时间复杂度相同但 I/O 减少。

#### 4.3.3 Hash Join（MySQL 8.0.18+）

适用于等值连接且被驱动表无索引，大表连接性能最好：

```

1. 构建阶段：扫描小表 R，构建哈希表（key = 连接列，value = 行）
2. 探测阶段：扫描大表 S，每行用连接列值查哈希表
   如果命中，输出匹配行

```text

时间复杂度：O(|R| + |S|)，空间复杂度：O(min(|R|, |S|))

Hash Join 是星型连接和报表查询的利器。

---

## 5. 子查询与集合操作

### 5.1 子查询类型

#### 5.1.1 标量子查询（返回单个值）

```sql
-- 查询每个订单的金额与平均金额的差异
SELECT
    id,
    total_amount,
    (SELECT AVG(total_amount) FROM orders) AS avg_amount,
    total_amount - (SELECT AVG(total_amount) FROM orders) AS diff
FROM orders;
```

#### 5.1.2 行子查询（返回一行）

```sql
-- 查找与"张三"同城市且同年龄的用户
SELECT * FROM users
WHERE (city, age) = (
    SELECT city, age FROM users WHERE username = '张三'
);
```text

#### 5.1.3 表子查询（FROM 子句中）

```sql
-- 临时表的常见用法
SELECT 用户, 平均消费
FROM (
    SELECT
        user_id AS 用户,
        AVG(total_amount) AS 平均消费
    FROM orders
    GROUP BY user_id
) AS 统计
WHERE 平均消费 > 500;
```

#### 5.1.4 关联子查询

```sql
-- 查每个用户的最新订单
SELECT * FROM orders o
WHERE created_at = (
    SELECT MAX(created_at)
    FROM orders
    WHERE user_id = o.user_id  -- 关联外部查询
);
```text

#### 5.1.5 LATERAL（MySQL 8.0.14+）

```sql
-- LATERAL 子查询可以引用前面的 FROM 项
SELECT
    u.id,
    u.username,
    最近订单.*
FROM users u
LEFT JOIN LATERAL (
    SELECT id, total_amount, created_at
    FROM orders
    WHERE user_id = u.id
    ORDER BY created_at DESC
    LIMIT 3
) AS 最近订单 ON TRUE;
```

### 5.2 EXISTS / IN / ANY / ALL

```sql
-- EXISTS：存在性测试（相关子查询）
-- 查找下过订单的用户（比 JOIN ... DISTINCT 更高效）
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- NOT EXISTS：查找从未下过订单的用户
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- IN：值是否在集合中
SELECT * FROM users WHERE id IN (1, 2, 3);
SELECT * FROM users WHERE id IN (
    SELECT user_id FROM orders WHERE total_amount > 500
);

-- 注意：IN 遇到 NULL 会出问题！
SELECT * FROM users WHERE id NOT IN (
    SELECT user_id FROM orders WHERE total_amount > 500
);
-- 如果子查询返回结果包含 NULL（某个 user_id 为 NULL）
-- NOT IN (...) 会变成 ... != 1 AND ... != 2 AND ... != NULL
-- ... != NULL 结果为 UNKNOWN
-- 整行被过滤！查不出任何结果！
-- 安全写法：NOT IN (SELECT user_id FROM orders WHERE total_amount > 500 AND user_id IS NOT NULL)
-- 或使用 NOT EXISTS 替代

-- ANY/SOME：任意一个满足
SELECT * FROM products
WHERE price > ANY (SELECT price FROM competitor_products);

-- ALL：所有都满足
SELECT * FROM products
WHERE price > ALL (SELECT price FROM competitor_products);
```text

### 5.3 集合操作

```sql
-- UNION：合并两个查询结果（自动去重）
SELECT name FROM vip_users
UNION
SELECT name FROM regular_users;

-- UNION ALL：合并两个查询结果（保留重复，更快）
SELECT name FROM vip_users
UNION ALL
SELECT name FROM regular_users;

-- INTERSECT：交集（MySQL 8.0+）
SELECT id FROM orders WHERE status = 'paid'
INTERSECT
SELECT order_id FROM tickets WHERE used = 1;

-- EXCEPT：差集（MySQL 8.0+）
SELECT id FROM orders WHERE status = 'paid'
EXCEPT
SELECT order_id FROM tickets WHERE used = 0;
-- 已支付但票还没被使用的订单

-- 用 IN 模拟 INTERSECT
SELECT DISTINCT id FROM orders
WHERE status = 'paid'
  AND id IN (SELECT order_id FROM tickets WHERE used = 1);

-- 用 NOT EXISTS 模拟 EXCEPT
SELECT DISTINCT id FROM orders o
WHERE status = 'paid'
  AND NOT EXISTS (
    SELECT 1 FROM tickets t
    WHERE t.order_id = o.id AND t.used = 0
);
```

### 5.4 NULL 陷阱全面分析

NULL 是 SQL 中最容易出错的概念。以下是完整规则：

```text
1. NULL ≠ NULL
   SELECT NULL = NULL;    → NULL (不是 TRUE!)
   SELECT NULL <> NULL;   → NULL (不是 FALSE!)
   SELECT NULL IS NULL;   → TRUE
   SELECT NULL IS NOT NULL; → FALSE

2. NULL 参与运算结果为 NULL
   SELECT 1 + NULL;       → NULL
   SELECT CONCAT('a', NULL); → NULL (注意!)
   SELECT CONCAT_WS(',', 'a', NULL, 'b'); → 'a,b' (CONCAT_WS 会跳过 NULL)

3. NULL 在聚合函数中被忽略
   SELECT COUNT(*) FROM t;   → 计数含 NULL 的行
   SELECT COUNT(col) FROM t; → 只计数 col 非 NULL 的行
   SELECT AVG(col) → SUM/COUNT，NULL 行不计入分母

4. NULL 在排序中的位置
   ORDER BY col ASC  → NULL 在最前（MySQL 默认）
   ORDER BY col DESC → NULL 在最后（MySQL 默认）

5. 三值逻辑
   AND: TRUE  AND TRUE  = TRUE
        TRUE  AND FALSE = FALSE
        TRUE  AND NULL  = NULL
        FALSE AND NULL  = FALSE  ← 短路！
        NULL  AND NULL  = NULL

   OR:  TRUE  OR TRUE  = TRUE
        TRUE  OR FALSE = TRUE
        TRUE  OR NULL  = TRUE   ← 短路！
        FALSE OR NULL  = NULL
        NULL  OR NULL  = NULL

   NOT: NOT TRUE  = FALSE
        NOT FALSE = TRUE
        NOT NULL  = NULL
```

### 5.4.1 NULL 处理函数

```sql
-- COALESCE：返回第一个非 NULL 参数
SELECT COALESCE(NULL, NULL, 'default', '备用');  -- 'default'
SELECT COALESCE(phone, email, '无联系方式') FROM users;

-- 票务系统实践：订单展示时处理 NULL
SELECT
    order_no,
    COALESCE(payment_method, '未支付') AS 支付方式,
    COALESCE(paid_at, 'N/A') AS 支付时间,
    COALESCE(remark, '') AS 备注
FROM orders;

-- IFNULL：MySQL 特有的两参数版本
SELECT IFNULL(NULL, '默认值');  -- '默认值'
SELECT IFNULL(phone, '未填写') FROM users;

-- NULLIF：两值相等则返回 NULL，否则返回第一个值
SELECT NULLIF('a', 'a');  -- NULL
SELECT NULLIF('a', 'b');  -- 'a'

-- 用途：防止除零错误
SELECT total_amount / NULLIF(quantity, 0) FROM orders;

-- IF 函数：MySQL 的三目运算符
SELECT IF(1 > 0, '真', '假');  -- '真'
SELECT
    IF(used = 1, '已使用', '未使用') AS 状态,
    COUNT(*) AS 数量
FROM tickets
GROUP BY used;

-- 对比：
-- COALESCE(value1, value2, ...)  标准 SQL，N 个参数
-- IFNULL(value, default)         MySQL 扩展，2 个参数
-- IF(条件, 真值, 假值)            MySQL 三目运算符
```text

### 5.5 CTE（公用表表达式 / WITH 子句）

CTE（Common Table Expression）是 MySQL 8.0 引入的功能，用于在单个查询中定义临时结果集，可被后续查询多次引用。CTE 在当前语句执行结束后自动释放。

#### 5.5.1 基础 CTE

```sql
-- 语法
WITH cte_name AS (
    SELECT 查询
)
SELECT * FROM cte_name;

-- 示例：找出每个分类中价格最高的活动
WITH ranked_events AS (
    SELECT
        category,
        title,
        price,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM events
    WHERE status = 'published'
)
SELECT category, title, price
FROM ranked_events
WHERE rn = 1;
```

#### 5.5.2 CTE vs 子查询

```sql
-- 子查询方式：可读性差，多次引用需重复编写
SELECT * FROM (
    SELECT category, title, price,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM events
    WHERE status = 'published'
) AS sub
WHERE rn = 1;

-- CTE 方式：逻辑清晰，可多次引用
WITH ranked AS (
    SELECT category, title, price,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM events
    WHERE status = 'published'
)
SELECT * FROM ranked WHERE rn = 1
UNION ALL
SELECT * FROM ranked WHERE rn = 2;
```text

#### 5.5.3 多 CTE

```sql
WITH
    user_orders AS (
        SELECT user_id, COUNT(*) AS order_count, SUM(total_amount) AS total_spent
        FROM orders
        GROUP BY user_id
    ),
    vip_users AS (
        SELECT user_id FROM user_orders WHERE total_spent > 5000
    )
SELECT u.username, uo.order_count, uo.total_spent
FROM users u
JOIN user_orders uo ON u.id = uo.user_id
JOIN vip_users vu ON u.id = vu.user_id
ORDER BY uo.total_spent DESC;
```

#### 5.5.4 递归 CTE（RECURSIVE）

递归 CTE 是处理树形/层级数据的利器，如组织结构、分类树、日期序列。

```sql
-- 语法
WITH RECURSIVE cte_name AS (
    SELECT ...           -- anchor：递归起始行
    UNION ALL
    SELECT ... FROM cte_name WHERE ...  -- 递归：引用自身
)
SELECT * FROM cte_name;

-- 示例1：生成数字序列（1 到 10）
WITH RECURSIVE numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 10
)
SELECT * FROM numbers;
-- 结果：1, 2, 3, 4, 5, 6, 7, 8, 9, 10

-- 示例2：生成日期序列
WITH RECURSIVE dates AS (
    SELECT '2024-01-01' AS dt
    UNION ALL
    SELECT dt + INTERVAL 1 DAY FROM dates WHERE dt < '2024-01-31'
)
SELECT * FROM dates;

-- 票务系统实践：补全无订单的月份（避免 LEFT JOIN 缺失）
WITH RECURSIVE months AS (
    SELECT DATE_FORMAT(MIN(created_at), '%Y-%m-01') AS month_start
    FROM orders
    UNION ALL
    SELECT month_start + INTERVAL 1 MONTH
    FROM months
    WHERE month_start < (SELECT DATE_FORMAT(MAX(created_at), '%Y-%m-01') FROM orders)
)
SELECT
    m.month_start AS 月份,
    COUNT(o.id) AS 订单数,
    COALESCE(SUM(o.total_amount), 0) AS 总收入
FROM months m
LEFT JOIN orders o
    ON DATE_FORMAT(o.created_at, '%Y-%m-01') = m.month_start
   AND o.status = 'paid'
GROUP BY m.month_start
ORDER BY m.month_start;

-- 示例3：组织树遍历
CREATE TABLE departments (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    parent_id INT
);

INSERT INTO departments VALUES
    (1, '总公司', NULL),
    (2, '技术部', 1),
    (3, '市场部', 1),
    (4, '前端组', 2),
    (5, '后端组', 2);

WITH RECURSIVE dept_tree AS (
    SELECT id, name, parent_id, 1 AS level
    FROM departments
    WHERE id = 2
    UNION ALL
    SELECT d.id, d.name, d.parent_id, dt.level + 1
    FROM departments d
    JOIN dept_tree dt ON d.parent_id = dt.id
)
SELECT * FROM dept_tree ORDER BY level, id;

-- 递归限制：MySQL 默认最大递归深度 1000
-- SET cte_max_recursion_depth = 10000; 可调整
```text

#### 5.5.5 CTE 最佳实践

```

✅ 适合的场景：

- 复杂查询分步推理（CTE 像函数/变量的作用）
- 同一子查询在语句中多次引用时
- 递归遍历树形结构（组织架构、评论回复链）
- 生成序列数据（数字、日期、时间）

❌ 不适合的场景：

- 简单查询（用子查询即可）
- 需要在多个语句间共享数据（用视图）
- 超大数据集（CTE 没有索引，每次引用都重新执行）

```text

```sql
-- 复杂报表：CTE 让查询像管道一样清晰
WITH
    valid_orders AS (
        SELECT * FROM orders
        WHERE status = 'paid' AND created_at >= '2024-01-01'
    ),
    daily_sales AS (
        SELECT event_id, DATE(created_at) AS sale_date,
               SUM(total_amount) AS revenue, SUM(quantity) AS tickets_sold
        FROM valid_orders
        GROUP BY event_id, DATE(created_at)
    ),
    sales_rank AS (
        SELECT *, ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY revenue DESC) AS rn
        FROM daily_sales
    )
SELECT
    e.title AS 活动名称,
    sr.sale_date AS 最佳销售日,
    sr.revenue AS 当日收入,
    sr.tickets_sold AS 当日销量
FROM sales_rank sr
JOIN events e ON sr.event_id = e.id
WHERE sr.rn = 1
ORDER BY sr.revenue DESC;
```

---

## 6. 数据库规范化理论（含数学证明）

规范化（Normalization）是数据库设计的核心理论，目标是最小化数据冗余和避免更新异常。

### 6.1 为什么要规范化？

考虑一个非规范化的订单表：

```sql
CREATE TABLE orders_bad (
    order_id INT,
    customer_name VARCHAR(50),
    customer_phone VARCHAR(20),
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    quantity INT,
    order_date DATE
);
```text

**更新异常：**

- 如果一个客户改了手机号，需要更新多行（**更新异常**）
- 插入一个新客户但没有订单时，需要 NULL 占位（**插入异常**）
- 删除一个订单同时删除了客户信息（**删除异常**）
- 相同数据重复存储（**数据冗余**）

### 6.2 函数依赖（Functional Dependency）

**定义：** 设 R 是一个关系模式，X 和 Y 是属性子集。对于 R 的任意两个元组 t1 和 t2：
如果 t1[X] = t2[X] 必然推出 t1[Y] = t2[Y]，则称 X → Y（X 函数决定 Y 或 Y 函数依赖于 X）。

```

例：在 users 表中
    id → username      （id 决定用户名）
    id → email         （id 决定邮箱）
    username → email   （不成立：不同用户可能同名吗？假设 UNIQUE 则成立）

注意：函数依赖是 语义/业务规则，不是数学定律

```text

#### 6.2.1 Armstrong 公理（1974）

Armstrong 公理是推导函数依赖的完备公理系统：

```

设 R 是关系模式，X、Y、Z 是属性子集：

1. 自反律（Reflexivity）：如果 Y ⊆ X，则 X → Y
   例：{id, name} → {id}（显然成立）

2. 增广律（Augmentation）：如果 X → Y，则 XZ → YZ
   例：id → name，则 {id, age} → {name, age}

3. 传递律（Transitivity）：如果 X → Y 且 Y → Z，则 X → Z
   例：id → email，email → phone，则 id → phone

```text

**Armstrong 公理的推论：**

```

1. 合并律（Union）：如果 X → Y 且 X → Z，则 X → YZ
   例：id → name 且 id → email，则 id → {name, email}

2. 分解律（Decomposition）：如果 X → YZ，则 X → Y 且 X → Z
   例：id → {name, email}，则 id → name 且 id → email

3. 伪传递律（Pseudo-Transitivity）：如果 X → Y 且 WY → Z，则 WX → Z

```text

#### 6.2.2 闭包（Closure）计算算法

属性集 X 关于函数依赖集 F 的闭包 X⁺ 是：从 X 通过 F 能推导出的所有属性的集合。

**计算算法：**

```

输入：属性集 X，函数依赖集 F
输出：X⁺

1. result = X
2. while result 发生变化：
     for each 函数依赖 Y → Z in F:
         if Y ⊆ result:
             result = result ∪ Z
3. return result

```text

**示例：**

```

R = (A, B, C, D, E, F)
F = {A → BC, B → E, CD → EF, E → D}

计算 A⁺:

1. result = {A}
2. A → BC: A ⊆ {A}, 所以 result = {A, B, C}
3. B → E: B ⊆ {A, B, C}, 所以 result = {A, B, C, E}
4. E → D: E ⊆ {A, B, C, E}, 所以 result = {A, B, C, D, E}
5. CD → EF: CD ⊆ {A, B, C, D, E}, 所以 result = {A, B, C, D, E, F}
6. 没有更多变化，返回 {A, B, C, D, E, F}

所以 A 是候选键（闭包包含全部属性）！

```text

**判断 X 是否为超键：** X⁺ 包含 R 的所有属性。

### 6.3 第一范式（1NF）

**定义：** 关系模式 R 满足 1NF 当且仅当每个属性的值都是**原子值**（不可再分）。

```sql
-- 违反 1NF：电话号码列存储了多个值
CREATE TABLE users_not_1nf (
    id INT,
    name VARCHAR(50),
    phones VARCHAR(100)  -- "13800138000,13900139000"  ← 多值！
);

-- 违反 1NF：存在嵌套结构
CREATE TABLE orders_not_1nf (
    id INT,
    items JSON  -- JSON 数组中的每个元素不是原子的
);

-- 满足 1NF：拆分为单独的电话表
CREATE TABLE users_1nf (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE phones_1nf (
    user_id INT,
    phone VARCHAR(20),
    PRIMARY KEY (user_id, phone)
);
```

**注意：** 现代数据库实践中，JSON 类型有时可以接受（权衡了灵活性和规范性），但在严格的关系设计理论中违反 1NF。

### 6.4 第二范式（2NF）

**定义：** 满足 1NF，且每个**非主属性**都**完全函数依赖于**候选键（不存在部分依赖）。

**部分依赖：** 候选键是组合键时，非主属性只依赖于组合键的一部分。

```sql
-- 违反 2NF：候选键是 (student_id, course_id)
-- student_name 只依赖于 student_id（部分依赖）
-- course_name 只依赖于 course_id（部分依赖）
CREATE TABLE grade_not_2nf (
    student_id INT,
    course_id INT,
    student_name VARCHAR(50),  -- 部分依赖：只依赖于 student_id
    course_name VARCHAR(100),  -- 部分依赖：只依赖于 course_id
    score INT,                 -- 完全依赖：依赖于整个 (student_id, course_id)
    PRIMARY KEY (student_id, course_id)
);

-- 满足 2NF：拆成三张表
CREATE TABLE students_2nf (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE courses_2nf (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE grade_2nf (
    student_id INT,
    course_id INT,
    score INT,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students_2nf(id),
    FOREIGN KEY (course_id) REFERENCES courses_2nf(id)
);
```text

**判定算法：**

```

1. 找出所有候选键
2. 对于每个候选键，确定是完全依赖还是部分依赖
3. 如果存在部分依赖（部分键 → 非主属性），违反 2NF
4. 通过分解消除部分依赖

```text

### 6.5 第三范式（3NF）

**定义：** 满足 2NF，且每个非主属性**不传递依赖于**候选键。

**传递依赖：** X → Y 且 Y → Z，且 Y 不是超键，则 X → Z 是传递依赖。

```sql
-- 违反 3NF：传递依赖 id → city_id → city_name
CREATE TABLE users_not_3nf (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    city_id INT,
    city_name VARCHAR(100)  -- 传递依赖：id → city_id → city_name
    -- city_name 依赖于 city_id，而不是直接依赖于 id
);

-- 满足 3NF：拆表
CREATE TABLE cities_3nf (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE users_3nf (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    city_id INT,
    FOREIGN KEY (city_id) REFERENCES cities_3nf(id)
);
```

### 6.6 BCNF（Boyce-Codd 范式）

**定义：** 对于 R 上的每一个函数依赖 X → Y（Y ⊈ X），X 必须是 R 的一个超键。

BCNF 比 3NF 更严格——3NF 允许 X 不是超键，只要 Y 是主属性即可。

```sql
-- 违反 BCNF：一个教授只能在一个系，但一个系可以有多个教授
-- 候选键：(student, professor)
-- FD: professor → department（教授决定系）
-- 但 professor 不是超键！
CREATE TABLE enrollment_bcnf (
    student VARCHAR(50),
    professor VARCHAR(50),
    department VARCHAR(50),
    PRIMARY KEY (student, professor)
    -- FD: professor → department（非超键 → 非主属性）
);

-- 满足 BCNF：分解
CREATE TABLE professors_bcnf (
    name VARCHAR(50) PRIMARY KEY,
    department VARCHAR(50)
);

CREATE TABLE enrollment_bcnf_ok (
    student VARCHAR(50),
    professor VARCHAR(50),
    PRIMARY KEY (student, professor),
    FOREIGN KEY (professor) REFERENCES professors_bcnf(name)
);
```text

### 6.7 无损分解算法

将违反范式的表分解为符合范式的多个表时，必须保证**无损连接**（分解后的表通过 JOIN 能还原出原始数据）。

**判定无损分解的 Chase 算法：**

```

输入：R = (A1, A2, ..., An)，分解 ρ = {R1, R2, ..., Rk}，函数依赖集 F
输出：ρ 是否为无损分解

1. 创建 k 行 n 列的表格，每个 Ri 对应一行，每个 Aj 对应一列
2. 如果 Aj ∈ Ri，填充 a_j，否则填充 b_ij
3. repeat:
     对于每个 FD X → Y in F:
         找到所有在 X 上值一致的行
         使它们在 Y 上的值也一致（遵循符号偏序：a_j > b_ij）
4. 直到某行全部变为 a_1, a_2, ..., a_n（表示该行能完全重建原始元组）
5. 如果出现全 a 行，则是无损分解

```text

**简化的二元分解判定：**

将 R 分解为 {R1, R2}，则是无损分解当且仅当：

```

(R1 ∩ R2) → R1 或 (R1 ∩ R2) → R2
直观理解：R1 ∩ R2 是 R1 的超键，或 R1 ∩ R2 是 R2 的超键。

```text

**示例：**

```

R = (A, B, C)
F = {A → B}
分解为 R1 = (A, B), R2 = (A, C)

R1 ∩ R2 = {A}
A → R1? A → B，是的，A 是 R1 的超键。所以是无损分解！

```text

### 6.8 规范化总结

| 范式 | 条件 | 解决 |
| ------ | ------ | ------ |
| 1NF | 属性值原子化 | 拆表/去 JSON |
| 2NF | 无部分依赖 | 消除组合键的部分依赖 |
| 3NF | 无传递依赖 | 消除非主属性对非超键的依赖 |
| BCNF | 所有 FD 的左边都是超键 | 分解所有违反 BCNF 的 FD |

**实际项目中的规范化程度选择：**

- **OLTP（在线交易）系统**：3NF 足够，BCNF 通常更好
- **OLAP（分析报表）系统**：适当**反规范化**（故意增加冗余）以减少 JOIN 次数
- **日常业务系统**：3NF 是最佳平衡点

---

## 7. MySQL 存储引擎深度解析

### 7.1 存储引擎概述

MySQL 的插件式存储引擎架构是其最大特色之一：

```sql
-- 查看当前默认存储引擎
SELECT @@default_storage_engine;

-- 查看表使用的存储引擎
SHOW TABLE STATUS WHERE Name = 'users';

-- 创建表时指定存储引擎
CREATE TABLE example (
    id INT PRIMARY KEY
) ENGINE = InnoDB;

-- 修改已有表的存储引擎
ALTER TABLE example ENGINE = InnoDB;
```

### 7.2 InnoDB 架构

InnoDB 是 MySQL 8.0 的默认存储引擎（从 MySQL 5.5 开始），也是**唯一支持事务**的引擎。

#### 7.2.1 核心架构图

```text
                    MySQL Server (SQL Layer)
                          │
                          ▼
                     ┌─────────────┐
                     │  InnoDB     │
                     │  (存储引擎) │
                     └─────────────┘
                     ┌─────────────┐
          Buffer     │  Buffer Pool │
          Pool       │ (数据页缓存) │
                     │ LRU 列表管理 │
                     └─────────────┘
                     ┌─────────────┐
                     │  Change     │
                     │  Buffer     │
                     │ (变更缓冲)  │
                     └─────────────┘
                     ┌─────────────┐
                     │  Redo Log   │
                     │  Buffer     │
                     └─────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌────────┐      ┌────────────┐     ┌────────────┐
   │ 数据   │      │  Redo     │     │  Undo      │
   │ 文件   │      │  Log 文件 │     │  表空间    │
   │ .ibd   │      │  ib_logfile│    │  undo_001  │
   └────────┘      └────────────┘     └────────────┘
```

#### 7.2.2 关键特性

**1. MVCC（多版本并发控制）**

- 每行记录维护多个版本（通过 undo log 实现）
- 读操作不阻塞写操作，写操作不阻塞读操作
- 为不同的隔离级别创建不同的 read view

**2. 聚簇索引（Clustered Index）**

- 数据物理存储顺序与主键顺序相同
- 叶子节点 = 完整的数据行
- 二级索引叶子节点 = 主键值

**3. 行级锁（Row-Level Lock）**

- 只锁定需要操作的行，并发度远高于表锁
- 通过索引实现行锁——如果查询不走索引，会升级为表锁！
- 支持 Gap Lock（间隙锁）防止幻读

**4. 外键约束**

- 唯一支持外键引用的 MySQL 存储引擎

**5. Doublewrite Buffer（双写缓冲）**

- 解决部分页面写入（partial page write）问题
- 写数据页前先写 doublewrite buffer
- 崩溃恢复时可以从 doublewrite 恢复损坏的页面

**6. Change Buffer（变更缓冲）**

- 对二级索引的修改先缓存在内存中
- 等到页面被读取到 buffer pool 时再合并
- 减少离散 I/O，提高修改性能

**7. 自适应哈希索引（Adaptive Hash Index）**

- InnoDB 自动为频繁访问的索引页建立哈希索引
- 将 B+Tree 的 O(log n) 查找降为 O(1)

### 7.3 其他存储引擎对比

| 特性 | InnoDB | MyISAM | MEMORY | CSV |
| ------ | -------- | -------- | -------- | ----- |
| 事务 | ✅ | ❌ | ❌ | ❌ |
| 行锁 | ✅ | ❌（表锁） | ❌（表锁） | ❌（表锁） |
| 外键 | ✅ | ❌ | ❌ | ❌ |
| 全文索引 | ✅（8.0+） | ✅ | ❌ | ❌ |
| 聚簇索引 | ✅ | ❌ | ❌ | ❌ |
| 崩溃恢复 | ✅ | ❌ | 重启后清空 | ❌ |
| 支持 MVCC | ✅ | ❌ | ❌ | ❌ |
| 数据压缩 | ✅ | ✅ | ❌ | ❌ |
| 磁盘空间 | 大 | 小 | N/A | 小 |

**生产选择：** 一律使用 InnoDB。MyISAM 只应存在于 MySQL 系统表。

---

## 8. 索引完全教程

### 8.1 B+Tree 数据结构

B+Tree 是 InnoDB 索引的核心数据结构。

#### 8.1.1 结构定义

B+Tree 是一棵**平衡多路搜索树**，具有以下特性：

```text
1. 所有数据存储在叶子节点
2. 内部节点只存储键值和指针（用于路由）
3. 叶子节点之间用双向链表连接
4. 所有叶子节点在同一层（高度平衡）

                    ┌──────────┐
                    │  根节点   │
                    │  50   150 │        ← 内部节点，只有键值+指针
                    └────┬─────┘
                   /     |      \
          ┌───────┐ ┌────┴───┐ ┌┴────────┐
          │ 30    │ │ 50  100│ │ 150  200│     ← 内部节点
          └───┬───┘ └───┬────┘ └────┬────┘
         /    |        /    \        |     \
   ┌────┐ ┌───┴──┐ ┌──┴──┐ ┌──┴──┐ ┌──┴──┐ ┌──┴───┐
   │10,20│ │30,40│ │50,55│ │80,90│ │100, │ │150,  │ ← 叶子节点
   │     │ │     │ │     │ │     │ │120  │ │160   │    含完整数据
   └────┘ └─────┘ └─────┘ └─────┘ └─────┘ └──────┘
   ←───────────────── 双向链表 ──────────────────→
```

#### 8.1.2 B+Tree vs B-Tree vs 二叉树

| 特性 | B+Tree | B-Tree | 二叉搜索树 |
| ------ | -------- | -------- | ----------- |
| 数据位置 | 叶子节点 | 所有节点 | 所有节点 |
| 叶子节点连接 | ✅ 双向链表 | ❌ | ❌ |
| 范围查询 | 快（链表遍历） | 慢（多次回溯） | 慢 |
| 树高（百万数据） | 3-4 层 | 3-4 层 | ~20 层 |
| 磁盘 I/O 次数 | ~3 次 | ~3 次 | ~20 次 |

#### 8.1.3 Fanout（扇出）和树高计算

**扇出（Fanout）：** 每个内部节点能存储的最大键值数。

```text
InnoDB 数据页大小：16KB（默认）

假设键值类型为 BIGINT（8 字节）+ 指针（6 字节）
每个索引条目 = 14 字节

每个节点的扇出 = 16KB / 14B ≈ 1170

树高计算：
- 叶子节点能存储的行数取决于行大小
- 假设行大小 1KB，每个叶子节点存储 16 行

3 层 B+Tree 能存储的行数：
第 1 层（根）：1 节点 × 1170 条目 = 1170
第 2 层：1170 节点 × 1170 条目 = 1,368,900
第 3 层（叶子）：1,368,900 节点 × 16 行 = ≈ 2100 万行

3 层树 → 3 次磁盘 I/O（根页常驻内存，实际只需 2 次）
```

**数学证明：** 对于 N 行数据，二叉搜索树的树高为 log₂(N)，B+Tree 的树高为 log_fanout(N)。当 N = 10⁷ 时：

- 二叉搜索树：log₂(10⁷) ≈ 24 次 I/O
- B+Tree（fanout=1170）：log₁₁₇₀(10⁷) ≈ 3 次 I/O（实际 2 次）

#### 8.1.4 聚簇索引 vs 二级索引

```sql
-- 聚簇索引（主键索引）
-- 叶子节点 = 完整的行数据
-- 表数据物理有序
-- 一张表只能有一个聚簇索引

-- 二级索引（辅助索引）
-- 叶子节点 = 主键值
-- 通过二级索引查询需要"回表"（先查二级索引，找到主键，再查聚簇索引）
-- 一张表可以有多个二级索引

-- 回表演示
CREATE TABLE users (
    id BIGINT PRIMARY KEY,      -- 聚簇索引
    username VARCHAR(50),
    email VARCHAR(100),
    INDEX idx_email (email)     -- 二级索引
);

-- 查询：SELECT * FROM users WHERE email = 'test@example.com'
-- 1. 通过 idx_email 找到对应的 id
-- 2. 通过 id 回表查询完整行数据

-- 覆盖索引：索引已包含需要查询的列，无需回表
-- SELECT email FROM users WHERE email = 'test@example.com'
-- 只需访问 idx_email，不需要回表
```text

### 8.2 索引类型

```sql
-- 1. 普通索引（INDEX）
CREATE INDEX idx_username ON users(username);

-- 2. 唯一索引（UNIQUE INDEX）
CREATE UNIQUE INDEX idx_email ON users(email);

-- 3. 主键索引（PRIMARY KEY）
ALTER TABLE users ADD PRIMARY KEY (id);

-- 4. 复合索引（多列索引）
-- 最左前缀原则：索引 (a, b, c) 能加速 a, a+b, a+b+c 的查询
-- 不能加速 b, c, b+c 的查询
CREATE INDEX idx_user_status ON orders(user_id, status, created_at);

-- 5. 全文索引（FULLTEXT INDEX）
CREATE FULLTEXT INDEX ft_content ON articles(title, content);
-- 使用 MATCH ... AGAINST 语法
SELECT * FROM articles WHERE MATCH(title, content) AGAINST('票务系统');

-- 6. 空间索引（SPATIAL INDEX）
CREATE SPATIAL INDEX sp_location ON venues(location);
```

### 8.3 EXPLAIN 解读

```sql
-- 查看查询执行计划
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com\G
```text

**EXPLAIN 输出字段详解：**

| 字段 | 值 | 含义 |
| ------ | ----- | ------ |
| **id** | 数字 | 查询的 SELECT 标识号，越大越先执行 |
| **select_type** | SIMPLE | 简单查询（无子查询/UNION） |
| | PRIMARY | 最外层查询 |
| | SUBQUERY | 子查询 |
| | DERIVED | 派生表（FROM 子查询） |
| | UNION | UNION 中的第二个或后续查询 |
| **table** | 表名 | 当前行对应的表 |
| **partitions** | NULL/p# | 匹配的分区 |
| **type** | **system** | 表只有一行（系统表，最佳） |
| | **const** | 主键或唯一索引的等值查找（最佳） |
| | **eq_ref** | 唯一索引扫描，每个驱动表行返回一行（极佳） |
| | **ref** | 非唯一索引等值扫描（良好） |
| | **range** | 索引范围扫描（BETWEEN、>、<、IN）（良好） |
| | **index** | 索引全扫描（比全表快但需要回表）（一般） |
| | **ALL** | 全表扫描（最差，需要尽量避免） |
| **possible_keys** | 索引名 | 可能使用的索引 |
| **key** | 索引名 | 实际使用的索引 |
| **key_len** | 字节数 | 索引使用的字节数（越长越精确） |
| **ref** | const/列 | 索引列匹配的值 |
| **rows** | 估算行数 | MySQL 估算的需要扫描的行数 |
| **filtered** | 百分比 | 经过表条件过滤后剩余的比例 |
| **Extra** | Using index | 覆盖索引，不需要回表（最佳） |
| | Using where | 使用 WHERE 过滤 |
| | Using index condition | 索引条件下推（ICP 优化） |
| | **Using filesort** | 需要额外排序（需要优化！） |
| | **Using temporary** | 需要临时表（需要优化！） |
| | Using join buffer | 连接未使用索引 |

**最需要关注的字段：** `type`、`key`、`rows`、`Extra`

**判断索引是否高效的速查表：**

```sql
-- 优秀：
type: const/eq_ref          -- 唯一索引/主键查找
Extra: Using index          -- 覆盖索引

-- 良好：
type: ref/range             -- 索引范围查找

-- 警戒：
type: index                 -- 索引全扫描
Extra: Using filesort       -- 需要优化 ORDER BY

-- 危险：
type: ALL                   -- 全表扫描
Extra: Using temporary      -- 需要优化 GROUP BY/DISTINCT
```

### 8.4 复合索引的最左前缀原则

```sql
-- 创建复合索引
CREATE INDEX idx_a_b_c ON t(a, b, c);

-- 能用索引的查询：
WHERE a = 1                 -- ✅ 最左前缀：a
WHERE a = 1 AND b = 2       -- ✅ 最左前缀：a, b
WHERE a = 1 AND b = 2 AND c = 3  -- ✅ 全用
WHERE a = 1 ORDER BY b      -- ✅ 排序也能用索引
WHERE a > 1 AND b = 2      -- ❌ b 用不了（a 是范围查询）

-- 不能用索引的查询：
WHERE b = 2                 -- ❌ 没从最左列开始
WHERE c = 3                 -- ❌ 没从最左列开始
WHERE b = 2 AND c = 3       -- ❌ 跳过了 a
```text

**为什么叫"最左前缀"？** 复合索引等价于按 (a, b, c) 的顺序排序，只有严格按顺序使用才能利用排序关系。

### 8.5 索引最佳实践

```sql
-- 1. 为 WHERE 中的过滤列建索引
-- 2. 为 ORDER BY 的排序列建索引（避免 filesort）
-- 3. 为 JOIN 的连接列建索引
-- 4. 利用覆盖索引减少回表
-- 5. 复合索引将选择性高的列放前面

-- 选择性 = COUNT(DISTINCT col) / COUNT(*)
-- 选择性越高，索引越有效
-- 例如：ID 的选择性 = 1（每条记录唯一）→ 最适合放最前面
-- 例如：性别选择性 ≈ 0.5 → 不适合单独索引

-- 6. 不要过度索引
-- 每个索引都是写操作的负担（INSERT/UPDATE/DELETE 需要更新索引）
-- 索引占用磁盘空间

-- 7. 使用 SHOW INDEX 查看表索引
SHOW INDEX FROM users;
```

---

## 9. 事务与隔离级别

### 9.1 什么是事务？

事务（Transaction）是一组 SQL 操作的**逻辑单元**，全部成功或全部失败。

### 9.2 ACID 属性

| 属性 | 全称 | 含义 | 实现机制 |
| ------ | ------ | ------ | --------- |
| **A** | Atomicity（原子性） | 事务不可分割，要么全做要么全不做 | undo log（回滚日志） |
| **C** | Consistency（一致性） | 事务前后数据满足所有约束 | 应用层 + 数据库约束 |
| **I** | Isolation（隔离性） | 并发事务互不干扰 | MVCC + 锁 |
| **D** | Durability（持久性） | 提交的事务永久保存 | redo log（重做日志） |

#### ACID 详解

**原子性（Atomicity）证明：**

```text
如果事务中途崩溃，InnoDB 通过 undo log 回滚已执行的修改。

undolog 结构：
TRX_ID | 表空间ID | 页号 | 行号 | 旧值
-------|----------|------|------|------
1001   | 10       | 5    | 3    | 100

回滚时，InnoDB 读取 undo log，将数据恢复到修改前的状态。
```

**一致性（Consistency）保证：**

- 应用层通过业务逻辑确保数据正确
- 数据库层通过约束（PRIMARY KEY、UNIQUE、FOREIGN KEY、CHECK）确保数据合法性

**持久性（Durability）证明：**

```text
WAL（Write-Ahead Logging，预写日志）原理：
1. 事务提交前，先将修改写入 redo log（顺序写，快）
2. 再将数据页写入磁盘（随机写，慢）
3. 崩溃恢复时：已写入 redo log 但未写入数据页的修改，会被重放（replay）

redo log 采用组提交（group commit）优化：
多个事务的 redo log 可以合并一次 fsync。
顺序 I/O 速度 ≈ 500MB/s，随机 I/O ≈ 10MB/s
redo log 的 fsync 是提交延迟的主要来源
```

### 9.3 事务控制语句

```sql
-- 自动提交模式（默认）
-- 每条语句自动提交
SELECT @@autocommit;  -- 默认 1

-- 开启事务
BEGIN;           -- 或 START TRANSACTION;

-- 执行操作
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- 提交事务
COMMIT;

-- 回滚事务
ROLLBACK;

-- 保存点（部分回滚）
SAVEPOINT sp1;
UPDATE ...;
ROLLBACK TO SAVEPOINT sp1;  -- 回滚到 sp1，保留之前的修改
RELEASE SAVEPOINT sp1;       -- 释放保存点
```text

### 9.4 四种隔离级别

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置隔离级别（会话级别）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置隔离级别（全局级别）
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| --------- | ------ | ----------- | ------ |
| READ UNCOMMITTED | ✅ 可能 | ✅ 可能 | ✅ 可能 |
| READ COMMITTED | ❌ 不会 | ✅ 可能 | ✅ 可能 |
| REPEATABLE READ | ❌ 不会 | ❌ 不会 | ✅ 可能（InnoDB 不会） |
| SERIALIZABLE | ❌ 不会 | ❌ 不会 | ❌ 不会 |

**MySQL InnoDB 默认隔离级别：REPEATABLE READ**

#### 9.4.1 脏读（Dirty Read）

```sql
-- 事务 A 修改未提交，事务 B 读取到了修改
-- 如果事务 A 回滚，事务 B 读到了"脏数据"

-- 时间线：
-- T1: 事务A: UPDATE users SET name='新名字' WHERE id=1;
-- T2: 事务B: SELECT name FROM users WHERE id=1; -- 读到"新名字"（脏数据！）
-- T3: 事务A: ROLLBACK; -- 回滚了
-- T4: 事务B: 基于读取的"新名字"做了其他操作（错误！）

-- READ UNCOMMITTED 级别时会发生
-- READ COMMITTED 及以上级别不会
```text

#### 9.4.2 不可重复读（Non-Repeatable Read）

```sql
-- 同一事务内两次读取同一数据，结果不一致

-- 时间线：
-- T1: 事务A: SELECT name FROM users WHERE id=1; -- 读到"张三"
-- T2: 事务B: UPDATE users SET name='李四' WHERE id=1; COMMIT;
-- T3: 事务A: SELECT name FROM users WHERE id=1; -- 读到"李四"（不一致！）
-- T4: 事务A: 基于两次不同的读取做操作（可能有问题）

-- READ COMMITTED 级别时会发生
-- REPEATABLE READ 级别不会
```

#### 9.4.3 幻读（Phantom Read）

```sql
-- 同一事务内两次查询返回不同的行集合

-- 时间线：
-- T1: 事务A: SELECT * FROM orders WHERE total_amount > 100; -- 返回 3 行
-- T2: 事务B: INSERT INTO orders(total_amount) VALUES(200); COMMIT;
-- T3: 事务A: SELECT * FROM orders WHERE total_amount > 100; -- 返回 4 行！（幻觉行）
-- 同一个事务内，同样的查询条件，返回了不同的行数

-- 其他隔离级别可能发生
-- InnoDB 通过 Gap Lock 在 REPEATABLE READ 级别消除了幻读
```text

### 9.5 MVCC 实现原理

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 InnoDB 实现高并发的核心技术。

#### 9.5.1 核心组件

**1. 隐藏列**
每行记录有三个隐藏列：

```

DB_TRX_ID：最后修改该行的事务 ID
DB_ROLL_PTR：回滚指针，指向 undo log 中的旧版本
DB_ROW_ID：行 ID（没有主键时自动生成）

```text

**2. Undo Log**

| 事务 ID | 版本链（DB_ROLL_PTR 连接） |
| --------- | -------------------------- |
| 100 | 行当前值 ← trx_id=100 |
| 90 | 旧版本 ← trx_id=90 |
| 80 | 更旧版本 ← trx_id=80 |

**3. Read View（读视图）**
包含三个关键信息：

```

low_limit_id：         下一个待分配的事务 ID（当前上限）
up_limit_id：          ReadView 创建时活跃的最小事务 ID
active_trx_ids：       ReadView 创建时所有活跃事务 ID 列表
creator_trx_id：       创建 ReadView 的事务自己的 ID

```text

#### 9.5.2 可见性判断算法

```

判断一个版本（由 DB_TRX_ID 标识）对当前 ReadView 是否可见：

1. 如果 DB_TRX_ID < up_limit_id：
   → 可见（事务已提交）
2. 如果 DB_TRX_ID >= low_limit_id：
   → 不可见（事务在 ReadView 创建之后开始）
3. 如果 DB_TRX_ID in active_trx_ids：
   → 不可见（事务未提交）
4. 其他情况：
   → 可见（事务已提交且在 ReadView 创建范围之外）

```text

#### 9.5.3 REPEATABLE READ 实现

```

在 REPEATABLE READ 级别，Read View 在事务的第一次 SELECT 时创建，整个事务期间复用。

事务 A（ID=100）：
SELECT * FROM users WHERE id = 1;
→ 创建 Read View: {up=100, low=200, active=[100]}

                        修改历史
                  ┌─────────────────┐
  事务B(101)      │ DB_TRX_ID = 101 │  → 不可见（101 > up_limit_id=100）
  修改并提交      └─────────────────┘
                  ┌─────────────────┐
  事务C(102)      │ DB_TRX_ID = 102 │  → 不可见（102 > up_limit_id=100）
  修改并提交      └─────────────────┘
                  ┌─────────────────┐
  事务A(100)      │ DB_TRX_ID = 0   │  → 可见（0 < up_limit_id=100）
  修改前          └─────────────────┘
                  （原始值）

事务 A 始终看到创建 Read View 时刻的数据快照。
这就是 REPEATABLE READ 的实现原理。

```text

#### 9.5.4 READ COMMITTED 实现

```

在 READ COMMITTED 级别，每条 SELECT 语句都创建新的 Read View。

事务 A（ID=100）：
SELECT * FROM users WHERE id = 1;
→ 创建 Read View: {up=100, low=101, active=[100]}
→ 事务B(101) 已提交，101 < low_limit_id，可见
→ 读到事务B的修改

第二条 SELECT：
→ 创建新的 Read View: {up=100, low=102, active=[100]}
→ 事务C(102) 已提交，102 < low_limit_id，可见
→ 读到事务C的修改

事务 A 每次 SELECT 都看到最新已提交的数据。
这就是 READ COMMITTED 的特性（不可重复读的根本原因）。

```text

### 9.6 隔离性证明：SERIALIZABLE 级别

SERIALIZABLE 是最高隔离级别，等价于所有事务串行执行。

**实现方式：** 所有 SELECT 隐式转为 `SELECT ... FOR SHARE`，读取的行加上共享锁，事务结束才释放。写操作互斥。

**证明（反证法）：**

```

假设有两个并发事务 T1 和 T2，在 SERIALIZABLE 级别下产生了与串行执行不等价的结果。

由于所有读取都加锁，T1 读取某行时 T2 不能修改该行。
T1 释放锁后 T2 才能获取锁。
所以所有操作的执行顺序必然等价于按加锁顺序串行执行。

与假设矛盾，故 SERIALIZABLE 等价于串行化执行。

```text

---

## 10. 票务系统数据库设计

### 10.1 需求分析

票务系统的核心业务需求：

```

1. 用户管理：注册、登录、信息管理
2. 活动管理：创建活动、设置场次、票价和库存
3. 订单管理：用户下单、取消订单
4. 票务管理：出票、检票
5. 支付管理：支付状态跟踪

核心约束：每张票只能被一个用户购买，不能超卖。

```text

### 10.2 概念设计（ER 图）

```

┌──────────┐      ┌──────────┐      ┌──────────┐
│  User    │      │  Event   │      │  Order   │
├──────────┤      ├──────────┤      ├──────────┤
│ id (PK)  │1──┐  │ id (PK)  │1──┐  │ id (PK)  │
│ username │   │  │ title    │   │  │ user_id  │─┐
│ email    │   └──│ user_id  │   └──│ event_id │ │
│ password │      │ date     │      │ status   │ │
│ role     │      │ venue    │      │ total    │ │
└──────────┘      │ price    │      │ created  │ │
                  │ stock    │      └──────────┘ │
                  └──────────┘                   │
                        │                        │
                        │ 1                      │
                        │                        │
                   ┌────┴──────┐                 │
                   │  Ticket   │                 │
                   ├───────────┤                 │
                   │ id (PK)   │                 │
                   │ order_id  │◄────────────────┘
                   │ event_id  │
                   │ seat_no   │
                   │ used      │
                   └───────────┘

```text

### 10.3 逻辑设计（四表 Schema）

```sql
-- ========== 1. 用户表（users）==========
-- 存储用户基本信息、认证信息和角色
-- 用户与订单：一对多
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '用户ID',
    username VARCHAR(50) NOT NULL COMMENT '用户名',
    password_hash VARCHAR(255) NOT NULL COMMENT '密码哈希(bcrypt)',
    email VARCHAR(255) NOT NULL COMMENT '邮箱',
    phone VARCHAR(20) DEFAULT NULL COMMENT '手机号',
    avatar VARCHAR(500) DEFAULT NULL COMMENT '头像URL',
    role ENUM('user', 'admin') NOT NULL DEFAULT 'user' COMMENT '角色',
    is_active BOOLEAN NOT NULL DEFAULT TRUE COMMENT '是否激活',
    last_login_at DATETIME(3) DEFAULT NULL COMMENT '最后登录时间',
    created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    UNIQUE KEY uk_username (username),
    UNIQUE KEY uk_email (email),
    INDEX idx_role (role),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';

-- ========== 2. 活动表（events）==========
-- 存储活动的详细信息、票价和库存
-- 一个活动由管理员创建，属于一个管理员用户
-- 一个活动有多个场次（不同日期/时间可视为不同活动或不同行）
CREATE TABLE events (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '活动ID',
    title VARCHAR(200) NOT NULL COMMENT '活动标题',
    description TEXT COMMENT '活动描述',
    venue VARCHAR(200) NOT NULL COMMENT '场馆地址',
    start_date DATETIME(3) NOT NULL COMMENT '开始时间',
    end_date DATETIME(3) NOT NULL COMMENT '结束时间',
    category VARCHAR(50) DEFAULT 'other' COMMENT '活动分类(concert/sports/theater/other)',
    cover_image VARCHAR(500) DEFAULT NULL COMMENT '封面图片',
    total_stock INT NOT NULL DEFAULT 0 COMMENT '总库存',
    sold_count INT NOT NULL DEFAULT 0 COMMENT '已售数量',
    price DECIMAL(10,2) NOT NULL COMMENT '统一票价',
    status ENUM('draft', 'published', 'sold_out', 'cancelled', 'finished')
        NOT NULL DEFAULT 'draft' COMMENT '状态',
    created_by BIGINT NOT NULL COMMENT '创建者(管理员ID)',
    created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    INDEX idx_category (category),
    INDEX idx_status (status),
    INDEX idx_start_date (start_date),
    INDEX idx_created_by (created_by),
    INDEX idx_status_date (status, start_date),
    CONSTRAINT fk_events_creator FOREIGN KEY (created_by) REFERENCES users(id)
        ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='活动表';

-- ========== 3. 订单表（orders）==========
-- 存储用户的购票订单
-- 订单状态流转：pending_payment → paid → completed
--                               ↘ cancelled
--                                 pending_payment → cancelled
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '订单ID',
    order_no VARCHAR(32) NOT NULL COMMENT '订单号(业务唯一)',
    user_id BIGINT NOT NULL COMMENT '用户ID',
    event_id BIGINT NOT NULL COMMENT '活动ID',
    quantity INT NOT NULL COMMENT '购买数量',
    total_amount DECIMAL(10,2) NOT NULL COMMENT '总金额',
    status ENUM('pending_payment', 'paid', 'completed', 'cancelled', 'refunded')
        NOT NULL DEFAULT 'pending_payment' COMMENT '订单状态',
    payment_method VARCHAR(20) DEFAULT NULL COMMENT '支付方式(alipay/wechat)',
    trade_no VARCHAR(64) DEFAULT NULL COMMENT '支付宝/微信交易号',
    paid_at DATETIME(3) DEFAULT NULL COMMENT '支付时间',
    cancelled_at DATETIME(3) DEFAULT NULL COMMENT '取消时间',
    remark VARCHAR(500) DEFAULT NULL COMMENT '订单备注',
    version INT NOT NULL DEFAULT 1 COMMENT '乐观锁版本号',
    created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    UNIQUE KEY uk_order_no (order_no),
    INDEX idx_user_id (user_id),
    INDEX idx_event_id (event_id),
    INDEX idx_status (status),
    INDEX idx_user_status (user_id, status),
    INDEX idx_created_at (created_at),
    CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT fk_orders_event FOREIGN KEY (event_id) REFERENCES events(id)
        ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单表';

-- ========== 4. 票务表（tickets）==========
-- 每张票对应一个座位，精确追踪每张票的状态
-- 存储购票后的加密验证信息，供检票使用
CREATE TABLE tickets (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '票ID',
    order_id BIGINT NOT NULL COMMENT '订单ID',
    event_id BIGINT NOT NULL COMMENT '活动ID',
    user_id BIGINT NOT NULL COMMENT '用户ID',
    ticket_no VARCHAR(32) NOT NULL COMMENT '票号(唯一)',
    seat_info VARCHAR(50) DEFAULT 'stand' COMMENT '座位信息(区域-排-号)',
    price DECIMAL(10,2) NOT NULL COMMENT '该票售价',
    qr_code VARCHAR(255) DEFAULT NULL COMMENT '二维码数据(加密)',
    used BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否已使用(检票)',
    used_at DATETIME(3) DEFAULT NULL COMMENT '检票时间',
    created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '创建时间',
    updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3) COMMENT '更新时间',
    UNIQUE KEY uk_ticket_no (ticket_no),
    INDEX idx_order_id (order_id),
    INDEX idx_event_id (event_id),
    INDEX idx_user_id (user_id),
    INDEX idx_used (used),
    CONSTRAINT fk_tickets_order FOREIGN KEY (order_id) REFERENCES orders(id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT fk_tickets_event FOREIGN KEY (event_id) REFERENCES events(id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT fk_tickets_user FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='票务表';
```

### 10.4 3NF 证明

现在证明上述 Schema 满足 3NF。

**第一步：确定候选键**

```text
users:    id → (所有属性)，候选键 = {id}
events:   id → (所有属性)，候选键 = {id}
orders:   order_no → (所有属性)，候选键 = {id, order_no}
tickets:  ticket_no → (所有属性)，候选键 = {id, ticket_no}
```

**第二步：检查函数依赖**

```text
users 表：
  主属性：id
  函数依赖：id → (所有非主属性)
  ✅ 所有非主属性完全依赖于候选键（无部分依赖 → 满足 2NF）
  ✅ 不存在传递依赖 X → Y → Z 其中 Y 不是超键（满足 3NF）

events 表：
  主属性：id
  函数依赖：id → (所有非主属性)
  ✅ 满足 2NF
  ✅ 满足 3NF

orders 表：
  主属性：id, order_no
  潜在依赖：order_no → user_id, event_id, quantity, ...（业务规则：订单号唯一）
  由于 order_no 是超键，order_no → 任何属性都符合 BCNF/3NF
  ✅ 满足 2NF（非主属性完全依赖于 {id} 或 {order_no}）
  ✅ 满足 3NF（无传递依赖）

tickets 表：
  主属性：id, ticket_no
  潜在依赖：ticket_no → (所有非主属性)
  如 orders 表类似，ticket_no 是超键
  ✅ 满足 2NF 和 3NF
```

**第三步：验证外键**

所有外键引用其他表的候选键（主键）：

```text
events.created_by → users.id
orders.user_id → users.id
orders.event_id → events.id
tickets.order_id → orders.id
tickets.event_id → events.id
tickets.user_id → users.id
```

✅ 全部符合参照完整性约束。

**结论：票务系统数据库 Schema 满足 3NF（实际上满足 BCNF）。**

### 10.5 运行 DDL

```bash
# 进入 MySQL 容器
docker exec -it ticket-mysql mysql -uroot -proot123 ticket_dev

# 或者将 DDL 保存为文件执行
docker exec -i ticket-mysql mysql -uroot -proot123 ticket_dev < app/models/schema.sql
```text

验证表创建成功：

```sql
-- 查看所有表
SHOW TABLES;

-- 查看表结构
DESC users;
SHOW CREATE TABLE users;

-- 查看索引
SHOW INDEX FROM orders;
```

---

## 11. 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `ERROR 1064 (42000)` SQL 语法错误 | 关键字错误或括号不匹配 | 检查逗号、括号、引号，用 IDE 语法高亮 |
| `ERROR 1452 (23000)` 外键约束失败 | 引用的父键不存在 | 先插入父表数据，或检查 DELETE 时被引用 |
| `ERROR 1175 (HY000)` 安全更新模式 | 非主键约束的 WHERE 不允许 UPDATE/DELETE | SET SQL_SAFE_UPDATES=0; 或使用主键条件 |
| `ERROR 1213 (40001)` 死锁 | 并发事务相互等待 | 代码中捕获重试，保持一致的加锁顺序 |
| LIKE '%keyword%' 查询极慢 | 前导通配符导致无法使用索引 | 使用全文索引（FULLTEXT）或搜索引擎 |
| 分页 OFFSET 越大越慢 | 数据库需要扫描并丢弃前面的行 | 使用游标分页（WHERE id > last_id） |
| `ERROR 1366 (HY000)` 中文插入乱码 | 字符集不是 utf8mb4 | 检查连接字符集 `SET NAMES utf8mb4` |
| `ERROR 1062 (23000)` 唯一键冲突 | 插入重复值 | 使用 INSERT ... ON DUPLICATE KEY UPDATE |

---

---

## 📁 项目代码参考

| 模型 | 文件 |
| ------ | ------ |
| 用户表 | `app/models/user.py` — 含 `is_banned_or_inactive` 属性 |
| 活动表 | `app/models/event.py` — 含 `version` 乐观锁字段 |
| 订单表 | `app/models/order.py` — 含状态机 `pending→paid→cancelled` |
| 票务表 | `app/models/ticket.py` |
| 登录历史 | `app/models/login_history.py` |
| 声明基类 | `app/models/base.py` — SQLAlchemy `DeclarativeBase` |

**MySQL 窗口函数实践：** 在查询活动列表时，窗口函数可用于统计每个分类的活动数量：

```sql
SELECT category, COUNT(*) AS cnt,
       RANK() OVER (ORDER BY COUNT(*) DESC) AS rank
FROM events GROUP BY category;
```text

## 12. 本章总结

✅ **已掌握：**

- SQL 完整语法：DDL/DML 全部命令、全部数据类型选择依据
- 全部 JOIN 类型（INNER/LEFT/RIGHT/FULL/CROSS/SELF/NATURAL）
- 子查询与集合操作（IN/EXISTS/ANY/ALL + UNION/INTERSECT/EXCEPT）
- NULL 陷阱全面理解
- 数据库规范化理论：1NF/2NF/3NF/BCNF 定义、判定算法、Armstrong 公理、无损分解证明
- MySQL InnoDB 架构：MVCC、聚簇索引、行锁、事务
- B+Tree 索引原理：扇出计算、树高证明、EXPLAIN 解读、复合索引最左前缀
- ACID 属性与隔离级别：脏读/不可重复读/幻读、MVCC 可见性算法
- 票务系统数据库设计：四表 Schema + 3NF 证明

✅ **项目里程碑：**

- [ ] 用户表（users）创建成功
- [ ] 活动表（events）创建成功
- [ ] 订单表（orders）创建成功
- [ ] 票务表（tickets）创建成功
- [ ] 所有外键约束生效
- [ ] 3NF 验证通过

**练习：**

1. 在 MySQL 中执行本章的 DDL 创建所有表
2. 插入 3 个用户、2 个活动、5 个订单和 10 张票的测试数据
3. 编写 SQL 查询：查询某个用户的所有已支付订单及其关联的票
4. 使用 EXPLAIN 分析查询的执行计划
5. 尝试修改某个用户信息并提交事务，观察隔离级别的影响

**下一章预告：** 第3章将使用 SQLAlchemy 2.0 完成 Python ORM 层，定义 Python 模型的 ORM 映射，使用 Alembic 管理数据库迁移，编写种子数据脚本。
