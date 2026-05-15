# 工具使用指南

> 本文档详细说明项目中使用的每个工具的配置、用法和最佳实践。

---

## 目录

1. [Ruff — Python 代码检查与格式化](#1-ruff--python-代码检查与格式化)
2. [Pyright — Python 静态类型检查](#2-pyright--python-静态类型检查)
3. [Pytest — 测试框架](#3-pytest--测试框架)
4. [Alembic — 数据库迁移](#4-alembic--数据库迁移)
5. [Docker Compose — 容器编排](#5-docker-compose--容器编排)
6. [ARQ — 异步任务队列](#6-arq--异步任务队列)
7. [Locust — 负载测试](#7-locust--负载测试)
8. [pip-audit / Safety — 依赖安全扫描](#8-pip-audit--safety--依赖安全扫描)
9. [Bandit — 静态安全分析](#9-bandit--静态安全分析)
10. [Radon — 代码复杂度度量](#10-radon--代码复杂度度量)
11. [Pre-commit — Git 钩子管理](#11-pre-commit--git-钩子管理)

---

## 1. Ruff — Python 代码检查与格式化

### 1.1 简介

Ruff 是一个极快的 Python lint 工具，用 Rust 编写，速度比 Flake8 快 10-100 倍。它兼容 Flake8 的大部分规则，并内置了 isort（导入排序）功能。

### 1.2 配置

项目配置在 `pyproject.toml`：

```toml
[tool.ruff]
line-length = 100                     # 最大行长度
target-version = "py311"              # 目标 Python 版本

[tool.ruff.lint]
select = ["ALL"]                      # 启用所有规则
ignore = [
    "D100", "D101", "D102", "D103", "D104", "D105", "D107",  # 文档字符串
    "S101",                           # assert 使用（测试中必要）
    "ANN101", "ANN102",               # 类型注解中的 self/cls
    "PLR2004",                        # 魔法数字
    "SLF001",                         # 访问私有属性
    "COM812",                         # 尾部逗号
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "D", "SLF", "PLR2004"]
"alembic/**" = ["D"]
```text

### 1.3 常用命令

```bash
# 检查所有文件
ruff check .

# 自动修复（安全修复）
ruff check --fix .

# 自动修复（包含非安全修复）
ruff check --unsafe-fixes --fix .

# 显示规则统计
ruff check --statistics .

# 查看特定规则的详细说明
ruff rule B008

# 仅检查特定规则
ruff check --select B008 .

# 检查后格式化
ruff format .
```

### 1.4 关键规则说明

| 规则 | 含义 | 危害等级 | 说明 |
| ------ | ------ | --------- | ------ |
| `B008` | 函数调用在默认参数中 | 高 | FastAPI 的 `Depends()` 会触发此规则，属误报 |
| `BLE001` | 盲捕获异常 | 高 | 应用 `except Exception` 时不指定具体异常 |
| `EM101` | 字符串字面量在异常中 | 中 | 建议使用变量 |
| `S105` | 硬编码密码 | **安全** | 代码中疑似出现明文密码 |
| `S311` | 非密码学安全随机数 | 中 | 使用 `secrets` 模块替代 `random` |
| `FAST002` | FastAPI 依赖未注解 | 高 | 依赖注入需要类型注解 |
| `ANN201` | 缺少返回类型注解 | 低 | 公共函数缺少 `-> ...` |
| `C901` | 函数复杂度过高 | 中 | McCabe 圈复杂度 > 10 |

### 1.5 pre-commit 集成

```yaml
# .pre-commit-config.yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.9.0
  hooks:
    - id: ruff
      args: [--fix]
    - id: ruff-format
```text

---

## 2. Pyright — Python 静态类型检查

### 2.1 简介

Pyright 是 Microsoft 开发的 Python 静态类型检查器，用 TypeScript 编写。它速度极快，能发现类型不匹配、None 值访问、参数类型错误等问题。

### 2.2 配置

项目配置在 `pyrightconfig.json`：

```json
{
    "include": ["app", "tests"],
    "exclude": ["**/node_modules", "**/__pycache__", ".venv", "alembic"],
    "typeCheckingMode": "standard",
    "reportMissingImports": true,
    "reportMissingTypeStubs": false,
    "pythonVersion": "3.10"
}
```

### 2.3 常用命令

```bash
# 检查整个项目
pyright .

# 检查特定目录
pyright app/services/

# 检查特定文件
pyright app/services/inventory_service.py

# 输出统计信息
pyright --stats .
```text

### 2.4 常见错误码及处理

| 错误码 | 含义 | 处理方式 |
| -------- | ------ | --------- |
| `reportAttributeAccessIssue` | 属性访问错误 | 第三方库类型桩缺失时加 `# type: ignore[attr-defined]` |
| `reportCallIssue` | 函数调用错误 | 参数数量不匹配时加 `# type: ignore[call-arg]` |
| `reportArgumentType` | 参数类型不匹配 | 修复类型或加 `# type: ignore[arg-type]` |
| `reportOptionalMemberAccess` | 可能为 None 的访问 | 添加 None 检查或类型守卫 |

### 2.5 type: ignore 使用规范

```python
# 第三方库类型桩缺失
result = await self.redis.evalsha(self._sha, len(keys), *keys, *args)  # type: ignore[arg-type]

# 原因不明的类型错误
public_key.verify(  # type: ignore[attr-defined]
    signature, sign_str.encode("utf-8"),
    padding.PKCS1v15(),  # type: ignore[call-arg]
    hashes.SHA256(),  # type: ignore[call-arg]
)
```

**原则**：尽量使用精确的 `# type: ignore[代码]`，而非不加参数的 `# type: ignore`，避免隐藏真正的类型错误。

---

## 3. Pytest — 测试框架

### 3.1 配置

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"           # 自动识别 async def 测试
testpaths = ["tests"]           # 测试目录
python_files = ["test_*.py"]    # 测试文件匹配模式
markers = [
    "integration: 标记集成测试，需要真实的 MySQL 和 Redis 服务",
]
```text

### 3.2 AsyncIO 模式

`asyncio_mode = "auto"` 让 pytest 自动识别异步测试函数，无需手动添加 `@pytest.mark.asyncio`（但仍建议添加，提高可读性）：

```python
# asyncio_mode = "auto" 时，以下两种方式都有效：
async def test_something():  # 自动识别为异步测试
    ...

@pytest.mark.asyncio
async def test_something_else():  # 显式标记
    ...
```

### 3.3 Fixture 详解

**Mock 型 Fixture（conftest.py）**：

```python
@pytest.fixture
def client():
    """创建 Mock HTTP 客户端"""
    from httpx import ASGITransport, AsyncClient
    from app.main import app
    transport = ASGITransport(app=app)
    return AsyncClient(transport=transport, base_url="http://localhost")
```text

**集成测试 Fixture（conftest_db.py）**：

```python
@pytest_asyncio.fixture(scope="session")
async def db_engine():
    """真实的数据库引擎（session 级别，复用连接）"""
    engine = create_async_engine(settings.DATABASE_URL, ...)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()
```

### 3.4 Mock 模式

```python
from unittest.mock import AsyncMock, MagicMock, patch

# 基本 Mock
mock_redis = AsyncMock()
mock_redis.get = AsyncMock(return_value=b"42")
mock_redis.zcard = AsyncMock(return_value=3)

# Mock 查询结果链
mock_result = MagicMock()
mock_result.scalars.return_value.all.return_value = [mock_order]
mock_session.execute = AsyncMock(return_value=mock_result)

# Mock 异步上下文管理器
mock_pipeline = AsyncMock()
mock_redis.pipeline.return_value.__aenter__.return_value = mock_pipeline
```text

### 3.5 常用命令速查

```bash
# 详细输出
pytest -v

# 简短回溯
pytest --tb=short

# 单行回溯
pytest --tb=line

# 运行特定模块
pytest tests/test_auth.py -v

# 运行特定测试
pytest tests/test_auth.py::test_register_success -v

# 按名称匹配
pytest -k "order" -v

# 覆盖率
pytest --cov=app --cov-report=term-missing -v

# 要求覆盖率不低于 80%
pytest --cov=app --cov-fail-under=80 -v

# 首次失败停止
pytest -x

# 失败时进入 PDB 调试
pytest --pdb
```

---

## 4. Alembic — 数据库迁移

### 4.1 配置

`alembic.ini`：

```ini
[alembic]
script_location = alembic
sqlalchemy.url = mysql+asyncmy://root:root123@localhost:3306/ticket_dev
```text

### 4.2 工作流程

```bash
# 1. 修改 SQLAlchemy 模型（添加/修改字段）
# 2. 生成迁移文件
alembic revision --autogenerate -m "add user phone field"

# 3. 检查生成的迁移文件
#    编辑 alembic/versions/xxxx_add_user_phone_field.py，确认无误

# 4. 应用迁移
alembic upgrade head

# 5. 查看当前版本
alembic current

# 6. 查看历史
alembic history

# 7. 回滚一步
alembic downgrade -1

# 8. 回滚到指定版本
alembic downgrade <revision_id>
```

### 4.3 迁移文件示例

```python
"""add user phone field

Revision ID: a1b2c3d4e5f6
Revises: 1234567890ab
Create Date: 2026-05-13 10:00:00.000000
"""

from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

revision: str = "a1b2c3d4e5f6"
down_revision: Union[str, None] = "1234567890ab"

def upgrade() -> None:
    op.add_column("users", sa.Column("phone", sa.String(20), nullable=True))

def downgrade() -> None:
    op.drop_column("users", "phone")
```text

### 4.4 自动生成 vs 手动编辑

自动生成可以处理大部分场景，但以下情况需要手动编辑：

1. **数据迁移**：更新已有数据的值
2. **复杂索引**：组合索引、部分索引
3. **表重命名**：自动生成会识别为"删除旧表 + 创建新表"，导致数据丢失
4. **默认值变更**：自动生成可能检测不到

### 4.5 最佳实践

1. **每次修改模型后立即生成迁移**，避免一次迁移包含多个不相关的修改
2. **始终审查自动生成的迁移文件**，确保没有误报
3. **生产环境先备份数据库**，再执行 `alembic upgrade head`
4. **不要在已发布的迁移上做修改**，应创建新的迁移来修正

---

## 5. Docker Compose — 容器编排

### 5.1 环境配置

| 文件 | 用途 | 端口暴露 | 网络 |
| ------ | ------ | --------- | ------ |
| `docker-compose.yml` | 开发环境 | 不暴露（内部网络） | 默认桥接 |
| `docker-compose.prod.yml` | 生产环境 | `127.0.0.1:3306:3306`（仅本地） | backend + frontend |

### 5.2 健康检查

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s    # 每 10 秒检查一次
  timeout: 5s      # 超时时间
  retries: 5       # 重试 5 次标记为 unhealthy
```

### 5.3 生产环境安全配置

```yaml
# 内部网络：禁止外部直接访问数据库
networks:
  backend:
    driver: bridge
    internal: true    # 关键！外部无法访问

# 端口锁定：仅监听本地回环
ports:
  - "127.0.0.1:3306:3306"  # 不对外暴露

# 资源限制
deploy:
  resources:
    limits:
      memory: 512M
      cpus: "1.0"
```text

### 5.4 常用命令

```bash
# 启动服务
docker compose up -d

# 查看日志
docker compose logs -f mysql
docker compose logs -f redis

# 重启服务
docker compose restart mysql

# 停止服务
docker compose down

# 停止并删除数据卷（危险！会丢失数据）
docker compose down -v

# 重建镜像
docker compose build

# 查看容器状态
docker compose ps
```

---

## 6. ARQ — 异步任务队列

### 6.1 简介

ARQ（Async Redis Queue）是一个轻量级的异步任务队列，基于 Redis 实现。本项目使用它来执行定时任务：自动取消超时订单。

### 6.2 Worker 配置

```python
class WorkerSettings:
    """ARQ Worker 配置"""
    functions = [
        send_order_confirmation,   # 发送订单确认通知
        release_expired_seats,     # 释放过期座位
        cancel_expired_orders,     # 取消超时订单
    ]
    keep_result = 86400           # 保留结果 24 小时
    keep_result_hours = 24
    on_startup = startup          # 启动时创建 Redis 连接
    on_shutdown = shutdown        # 关闭时清理连接
```text

### 6.3 定时任务

```python
async def cancel_expired_orders(ctx):
    """每 60 秒检查一次超时订单（由 ARQ 定时触发）"""
    redis: Redis = ctx["redis"]
    async with async_session_factory() as session:
        service = OrderService(session, redis)
        count = await service.auto_cancel_expired_orders()
        await session.commit()
        if count:
            logger.info("自动取消 %d 个超时订单", count)
    return {"cancelled": count}
```

### 6.4 启动 Worker

```bash
arq app.worker.WorkerSettings
```text

---

## 7. Locust — 负载测试

### 7.1 简介

Locust 是一个基于 Python 的负载测试工具，可以模拟大量用户并发访问系统。

### 7.2 基本用法

```bash
# Web 界面模式
locust -f locustfile.py

# 无头模式
locust -f locustfile.py --headless -u 10 -r 2 --run-time 10s
```

参数说明：

- `-u 10` — 模拟 10 个并发用户
- `-r 2` — 每秒启动 2 个用户
- `--run-time 10s` — 运行 10 秒
- `--host` — 目标主机地址

### 7.3 CI 集成

```yaml
# GitHub Actions 中使用
- run: pip install locust && locust -f locustfile.py --headless -u 10 -r 2 --run-time 10s
  env:
    DB_HOST: localhost
    DB_PASSWORD: root123
    DB_NAME: ticket_test
```text

---

## 8. pip-audit / Safety — 依赖安全扫描

### 8.1 pip-audit

```bash
# 安装
pip install pip-audit

# 扫描当前项目的依赖
pip-audit

# 扫描 requirements.txt
pip-audit -r requirements.txt

# JSON 格式输出
pip-audit --format json
```

### 8.2 Safety

```bash
# 安装
pip install safety

# 检查
safety check

# 检查特定文件
safety check -r requirements.txt

# JSON 输出
safety check --json
```text

两者都会将项目依赖与已知漏洞数据库（如 PyPI 安全公告）进行比对，发现存在已知 CVE 的依赖包。

---

## 9. Bandit — 静态安全分析

### 9.1 简介

Bandit 是 Python 代码的安全扫描工具，可以检测常见的安全漏洞模式。

### 9.2 用法

```bash
# 安装
pip install bandit

# 扫描整个 app 目录
bandit -r app/

# 输出 JSON 格式（便于 CI 处理）
bandit -r app/ -f json

# 排除特定规则
bandit -r app/ --skip B101,B102

# 只报告高严重性问题
bandit -r app/ --severity high
```

### 9.3 可检测的问题类型

| 问题 ID | 描述 | 项目中可能的位置 |
| --------- | ------ | ---------------- |
| B101 | 使用 assert | 测试文件（已忽略） |
| B105 | 硬编码密码 | 配置中的密码字符串 |
| B311 | 非密码学随机数 | 订单号生成（已使用 secrets） |
| B501 | request_with_no_cert_validation | HTTP 请求 |
| B608 | 可能的 SQL 注入 | LIKE 查询（已手动转义） |

---

## 10. Radon — 代码复杂度度量

### 10.1 简介

Radon 可以计算代码的各种复杂度指标，帮助识别需要重构的复杂代码。

### 10.2 常用命令

```bash
# 安装
pip install radon

# 计算 McCabe 圈复杂度
radon cc app/ -s

# 按复杂度排序
radon cc app/ -s --order SCORE

# 计算可维护性指数
radon mi app/

# HTML 报告
radon html app/
```text

### 10.3 复杂度解读

| McCabe 圈复杂度 | 评级 | 说明 |
| ---------------- | ------ | ------ |
| 1-5 | A | 代码简单，易于维护 |
| 6-10 | B | 代码较复杂，需要关注 |
| 11-20 | C | 代码复杂，建议重构 |
| 21-30 | D | 非常复杂，应重构 |
| 30+ | F | 极其复杂，必须重构 |

---

## 11. Pre-commit — Git 钩子管理

### 11.1 简介

Pre-commit 可以在 `git commit` 之前自动运行代码检查，确保提交的代码符合质量标准。

### 11.2 配置

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.15.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
        args: [--ignore-missing-imports]
        exclude: alembic/

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace      # 删除行尾空格
      - id: end-of-file-fixer        # 确保文件末尾有空行
      - id: check-yaml               # YAML 文件语法检查
      - id: check-added-large-files  # 检查大文件（默认 > 500KB）
      - id: check-merge-conflict     # 检查未解决的合并冲突
      - id: detect-private-key       # 检查泄露的私钥
```

### 11.3 安装

```bash
pip install pre-commit
pre-commit install
```text

安装后，每次 `git commit` 都会自动运行配置的钩子。

### 11.4 常用命令

```bash
# 手动运行所有钩子
pre-commit run --all-files

# 运行特定钩子
pre-commit run ruff --all-files

# 更新钩子版本
pre-commit autoupdate

# 跳过钩子（紧急情况）
git commit --no-verify
```
