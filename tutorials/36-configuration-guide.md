# 配置指南

## 默认账号

系统启动时通过 `app/services/seed_service.py` 自动创建以下测试账号：

### 超级管理员（ID=1）

| 字段 | 值 | 说明 |
| ------ | ----- | ------ |
| 用户名 | `admin` | 可在 `.env` 中通过 `DEFAULT_ADMIN_USERNAME` 修改 |
| 密码 | `Admin@12345` | 可在 `.env` 中通过 `DEFAULT_ADMIN_PASSWORD` 修改（≥8位） |
| 邮箱 | `admin@ticket-sell.com` | 可在 `.env` 中通过 `DEFAULT_ADMIN_EMAIL` 修改 |
| 角色 | `super_admin` | 超级管理员，拥有全部权限（包括日志查看、角色管理等） |
| 用户 ID | 1 | 固定为 1 |

> **安全警告**：生产环境务必修改默认密码！默认密码 `Admin@12345` 仅用于开发环境。

### 普通用户（ID=2）— 已实名认证

| 字段 | 值 |
| ------ | ----- |
| 用户名 | `ceshiyonghu` |
| 密码 | `Test@123456` |
| 手机号 | `13800000002` |
| 真实姓名 | 张三 |
| 身份证号 | `110101199001011234` |
| 实名认证 | ✅ 已完成 |
| 角色 | `user` |

### 游客用户（ID=3）— 仅手机验证

| 字段 | 值 |
| ------ | ----- |
| 用户名 | `youke` |
| 密码 | `Youke@123456` |
| 手机号 | `13800000003` |
| 实名认证 | ❌ 未认证 |
| 角色 | `user` |

### 一般管理员（ID=4）

| 字段 | 值 |
| ------ | ----- |
| 用户名 | `guanliyuan` |
| 密码 | `Admin@123456` |
| 手机号 | `13800000004` |
| 角色 | `admin`（可管理用户/活动/订单，无日志查看权限） |

---

## 数据库

## 数据库

### 存储位置

数据库文件在 Docker volume `ticket-sell_mysql_data` 中持久化：

```bash
# 查看物理存储路径（Windows Docker Desktop）
docker volume inspect ticket-sell_mysql_data
# 输出示例: "Mountpoint": "/var/lib/docker/volumes/ticket-sell_mysql_data/_data"

# Windows 实际路径（WSL2 后端，在文件资源管理器地址栏输入）
# \\wsl$\docker-desktop-data\data\docker\volumes\ticket-sell_mysql_data\_data

# 备份数据库
docker exec ticket-mysql mysqldump -u root -proot123 ticket_dev > backup.sql

# 恢复数据库
docker exec -i ticket-mysql mysql -u root -proot123 ticket_dev < backup.sql
```text

### 连接信息

| 参数 | 值 |
| ------ | ----- |
| 主机 | `localhost` |
| 端口 | `3306` |
| 用户名 | `root` |
| 密码 | `root123`（docker-compose.yml 默认值，可通过 `DB_PASSWORD` 环境变量覆盖） |
| 数据库名 | `ticket_dev`（可通过 `DB_NAME` 环境变量覆盖） |
| 连接串 | `mysql+asyncmy://root:root123@localhost:3306/ticket_dev` |

### 查看原始数据

#### 方式一：Adminer 网页端（推荐）

启动后访问 **<http://localhost:8080>** 直接通过浏览器管理数据库：

| 登录字段 | 填写值 |
| --------- | -------- |
| 系统 | `MySQL` |
| 服务器 | `mysql`（Docker 内部主机名） |
| 用户名 | `root` |
| 密码 | `root123` |
| 数据库 | `ticket_dev` |

登录后可浏览所有表：`users`、`events`、`orders`、`tickets`、`login_history`、`coupons`、`operation_logs`

#### 方式二：命令行

```bash
# 进入 MySQL 命令行
docker exec -it ticket-mysql mysql -u root -proot123 ticket_dev

# 直接查询
docker exec ticket-mysql mysql -u root -proot123 -e "USE ticket_dev; SELECT * FROM users;"

# 查看所有表
docker exec ticket-mysql mysql -u root -proot123 -e "USE ticket_dev; SHOW TABLES;"

# 查看表结构
docker exec ticket-mysql mysql -u root -proot123 -e "USE ticket_dev; DESCRIBE users;"
```

---

## Redis

| 参数 | 值 |
| ------ | ----- |
| 主机 | `localhost` |
| 端口 | `6379` |
| 密码 | `redispass123`（可通过 `REDIS_PASSWORD` 环境变量覆盖） |

```bash
# 进入 Redis 命令行
docker exec -it ticket-redis redis-cli -a redispass123

# 查看所有 key
docker exec ticket-redis redis-cli -a redispass123 KEYS '*'

# 查看库存
docker exec ticket-redis redis-cli -a redispass123 GET "stock:event:{event_id}"
```text

---

## 配置文件

### `.env` 文件（本地开发）

项目根目录的 `.env` 文件覆盖默认配置。关键变量：

```ini
# Security - 必须修改！
SECRET_KEY=dev-secret-key-change-in-production        # JWT 签名密钥（至少32字符）

# Database - 默认连接 Docker MySQL
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=root123
DB_NAME=ticket_dev

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redispass123

# JWT
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# 默认管理员（生产环境务必修改密码！）
DEFAULT_ADMIN_USERNAME=admin
DEFAULT_ADMIN_PASSWORD=Admin@12345
DEFAULT_ADMIN_EMAIL=admin@ticket-sell.com

# 实名认证（开发环境 Mock）
REALNAME_MOCK_ENABLED=true
SMS_MOCK_ENABLED=true

# 支付宝（沙箱环境）
ALIPAY_APP_ID=your-alipay-app-id
ALIPAY_APP_PRIVATE_KEY_PATH=
ALIPAY_PUBLIC_KEY_PATH=
ALIPAY_NOTIFY_URL=https://your-domain.com/api/v1/payments/notify

# 速率限制
RATE_LIMIT_AUTH_MAX=20
RATE_LIMIT_API_MAX=100
RATE_LIMIT_WINDOW=60

# 密码策略
PASSWORD_MIN_LENGTH=8
PASSWORD_REQUIRE_UPPER=true
PASSWORD_REQUIRE_LOWER=true
PASSWORD_REQUIRE_DIGIT=true
PASSWORD_REQUIRE_SPECIAL=true
```

### `alembic.ini`（数据库迁移）

```ini
sqlalchemy.url = mysql+asyncmy://root:root123@localhost:3306/ticket_dev
```text

---

## 关键配置类说明

配置类在 `app/config.py` 的 `Settings` 中定义，按功能分组：

| 配置组 | 对应字段 | 说明 |
| -------- | --------- | ------ |
| **应用** | `APP_NAME`, `ENV`, `DEBUG` | 应用基本设置 |
| **JWT** | `SECRET_KEY`, `ACCESS_TOKEN_EXPIRE_MINUTES`, `REFRESH_TOKEN_EXPIRE_DAYS` | 令牌签名与过期 |
| **密码策略** | `PASSWORD_MIN_LENGTH`, `PASSWORD_REQUIRE_*` | 注册密码强度要求 |
| **登录安全** | `LOGIN_MAX_ATTEMPTS`, `LOGIN_LOCKOUT_MINUTES` | 账户锁定策略 |
| **令牌安全** | `REFRESH_TOKEN_ROTATION`, `JWT_ISSUER` | 令牌轮换与签发者 |
| **数据库** | `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` | MySQL 连接 |
| **Redis** | `REDIS_HOST`, `REDIS_PORT`, `REDIS_DB`, `REDIS_PASSWORD` | Redis 连接 |
| **实名认证** | `REALNAME_MOCK_ENABLED`, `REALNAME_API_URL` | 第三方实名认证 |
| **短信** | `SMS_MOCK_ENABLED`, `SMS_API_URL`, `SMS_CODE_TTL` | 短信验证码 |
| **默认管理员** | `DEFAULT_ADMIN_USERNAME`, `DEFAULT_ADMIN_PASSWORD`, `DEFAULT_ADMIN_EMAIL` | 超级管理员（ID=1） |
| **加密密钥** | `ENCRYPTION_KEY` | AES-256 身份证加密（base64），不设置则明文存储 |
| **支付宝** | `ALIPAY_APP_ID`, `ALIPAY_APP_PRIVATE_KEY_PATH` | 支付集成 |

---

## 快速参考命令

```bash
# Adminer 网页数据库管理（推荐）
# 启动后访问 http://localhost:8080，选择 MySQL，服务器填 mysql

# 查看数据库所有表
docker exec ticket-mysql mysql -u root -proot123 -e "USE ticket_dev; SHOW TABLES;"

# 查看用户表
docker exec ticket-mysql mysql -u root -proot123 -e "USE ticket_dev; SELECT id, username, email, role, is_active, is_banned FROM users;"

# 查看 Redis 库存
docker exec ticket-redis redis-cli -a redispass123 KEYS 'stock:*'

# 运行迁移
alembic upgrade head

# 生成新迁移（模型变更后）
alembic revision --autogenerate -m "description"

# 启动应用
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```
