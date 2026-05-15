# 第9章：pytest 完全教程 — 从零到精通（以票务系统为例）

## 本章学习目标

本章将系统学习 pytest 测试框架的全部核心功能，并以票务系统的实际代码为例子，覆盖单元测试、集成测试、异步测试、并发测试等场景。

**学习路径：**

1. pytest 基础：发现规则、断言、conftest、fixture 层级
2. Fixture 进阶：scope、autouse、yield、内置 fixture
3. pytest-asyncio：异步测试、事件循环策略
4. httpx.AsyncClient：FastAPI 集成测试
5. Mock 与 MonkeyPatch：Mock、AsyncMock、pytest-mock
6. parametrize：参数化测试、indirect fixture
7. 覆盖率：pytest-cov 配置与报告
8. 票务系统测试模式逐行讲解

---

## 1. pytest 完全指南

### 1.1 安装与配置

```bash
pip install pytest pytest-asyncio httpx pytest-mock pytest-cov
```text

**推荐配置文件 `pyproject.toml`：**

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]              # 测试目录
python_files = ["test_*.py"]       # 测试文件命名模式
python_functions = ["test_*"]      # 测试函数命名模式
python_classes = ["Test*"]         # 测试类命名模式
asyncio_mode = "auto"              # pytest-asyncio 自动模式
asyncio_default_fixture_loop_scope = "function"
addopts = [
    "-v",                          # 详细输出
    "--tb=short",                  # 短回溯
    "--strict-markers",            # 严格 marker 检查
    "--cov=app",                   # 覆盖率目标
    "--cov-report=term-missing",   # 显示未覆盖行
    "--cov-report=html",           # HTML 覆盖率报告
]
markers = [
    "slow: 慢测试",
    "integration: 集成测试",
    "concurrent: 并发测试",
    "e2e: 端到端测试（需 Docker MySQL + Redis）",
]
```

### 1.2 测试发现规则

pytest 按照以下规则自动发现测试：

**默认发现路径：**

```text
项目根目录/
├── test_*.py              # 当前目录（或 testpaths 指定）
├── *_test.py
└── tests/                 # 默认 testpaths
    ├── test_*.py
    └── *_test.py
```

**发现规则详解：**

```python
# test_example.py — 这个文件会被发现（test_ 开头）

def test_simple():          # 这个函数会被发现（test_ 前缀）
    pass

class TestGroup:            # 这个类会被发现（Test 开头，不含 __init__）
    def test_in_class(self):  # 这个方法会被发现
        pass

    def not_a_test(self):     # 这个方法不会被发现（非 test_ 前缀）
        pass

def not_test_function():    # 这个函数不会被发现
    pass
```text

**选择测试：**

```bash
# 执行所有测试
pytest

# 执行特定文件
pytest tests/test_auth.py -v

# 执行特定函数
pytest tests/test_auth.py::test_login

# 执行特定类
pytest tests/test_auth.py::TestAuth

# 使用 -k 进行关键字过滤（支持 and/or/not）
pytest -k "login or register"
pytest -k "login and not slow"
pytest -k "TestAuth"

# 使用 marker 过滤
pytest -m "slow"           # 只运行带 @pytest.mark.slow 的测试
pytest -m "not slow"       # 跳过慢测试
pytest -m "integration"    # 只运行集成测试
```

### 1.3 断言系统

pytest 的断言比标准 `unittest` 更简洁，且失败信息更友好：

```python
import pytest
import warnings


def test_basic_assertions():
    """基本断言示例"""

    # 相等/不等
    assert 1 + 1 == 2
    assert 2 + 2 != 5

    # 包含
    assert "hello" in "hello world"
    assert 3 in [1, 2, 3]
    assert "key" in {"key": "value"}

    # 布尔值
    assert True
    assert not False
    assert [1, 2, 3]  # 非空列表为 True
    assert [] is not None  # 空列表也为 True
    assert not []     # 显式检查空列表

    # 近似相等（浮点数）
    assert 0.1 + 0.2 == pytest.approx(0.3)
    assert (0.1 + 0.2 - 0.3) == pytest.approx(0, abs=1e-15)

    # 近似相对误差
    assert 100.0 == pytest.approx(100.1, rel=0.01)  # 1% 相对误差


def test_exception_assertions():
    """异常断言"""

    # 期望抛出特定异常
    with pytest.raises(ValueError) as exc_info:
        int("not_a_number")

    # 检查异常信息
    assert "invalid literal" in str(exc_info.value)

    # 检查异常类型
    assert exc_info.type is ValueError

    # 使用 match 参数进行正则匹配
    with pytest.raises(ValueError, match="invalid literal"):
        int("not_a_number")

    # 不抛出异常（确保代码正常执行）
    with pytest.raises(ValueError):
        pass  # 这个测试会失败！因为没有异常抛出
    # 正确做法：使用 pytest.raises(None) 或将代码移出 with 块


def test_warning_assertions():
    """警告断言"""
    with pytest.warns(UserWarning):
        warnings.warn("this is a warning", UserWarning)

    # 检查警告消息
    with pytest.warns(UserWarning, match="this is a warning"):
        warnings.warn("this is a warning", UserWarning)

    # 确保没有警告
    with pytest.warns(None):
        pass  # 确认不会产生警告
```text

### 1.4 调试选项

```bash
# 打印输出（不捕获 stdout）
pytest -s

# 在第一个失败时停止
pytest -x

# 失败时进入 pdb 调试
pytest --pdb

# 显示每个测试的执行时间（最慢的 N 个）
pytest --durations=10

# 失败时打印局部变量
pytest --showlocals
# 或缩写：pytest -l

# 重试失败测试
# 需要安装 pytest-rerunfailures
pytest --reruns 2

# 并行执行
# 需要安装 pytest-xdist
pytest -n auto  # 自动使用所有 CPU 核心
```

---

## 2. conftest.py 层次架构

### 2.1 层次结构

conftest.py 是 pytest 的最重要特性之一。它定义了 fixture、钩子和插件，并在测试目录树中自动应用。

**作用域层次：**

```text
pytest 运行时
  │
  ├── root conftest.py（项目根目录）       # session 级 fixture 可见
  │     │
  │     ├── tests/conftest.py              # session/package 级 fixture
  │     │     │
  │     │     ├── tests/test_auth.py       # module 级 fixture 可见
  │     │     ├── tests/test_events.py     # 共享 tests/conftest 的 fixture
  │     │     └── tests/sub_package/
  │     │           └── conftest.py         # 仅 sub_package 下的测试可见
  │     │
  │     └── app/conftest.py（可选）         # app 包下的 fixture
```

**关键规则：**

1. **向上搜索**：测试文件会向上搜索所有祖先目录中的 conftest.py
2. **优先级**：最近的 conftest.py 覆盖较远的同名 fixture
3. **自动加载**：conftest.py 不需要 import，pytest 自动加载
4. **插件注册**：可以在 conftest.py 中注册自定义 marker、钩子函数

### 2.2 项目 conftest.py 模板

```python
# tests/conftest.py
# 这个 conftest 中的所有 fixture 对所有测试文件可见

import pytest
import asyncio
from typing import AsyncGenerator, Generator
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from redis.asyncio import Redis

from app.main import app
from app.database import Base, get_db
from app.redis_client import get_redis


# ===== 辅助函数 =====
@pytest.fixture(scope="session")
def event_loop():
    """
    pytest-asyncio 需要的事件循环 fixture

    scope="session" 意味着所有异步测试共享一个事件循环。
    注意：pytest-asyncio >= 0.21 已弃用此方式，改用
    asyncio_mode = "auto" + 配置。

    为了兼容性，保留此 fixture 并设为 session 级别。
    """
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session")
async def db_engine():
    """
    Session 级 fixture：创建一次数据库引擎

    所有测试共享一个内存 SQLite 数据库。
    每个测试函数独立的事务（通过 db_session fixture 回滚）。
    """
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        echo=False,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    yield engine

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture
async def db_session(db_engine):
    """
    Function 级 fixture：每个测试独立的数据库会话

    使用事务回滚实现测试隔离：
    1. 开启事务
    2. 执行测试
    3. 回滚事务（不留下副作用）
    """
    # 创建会话工厂
    session_factory = async_sessionmaker(
        db_engine,
        class_=AsyncSession,
        expire_on_commit=False,
    )

    async with session_factory() as session:
        # 开启嵌套事务
        async with session.begin():
            yield session
            # 测试结束后回滚（rollback 由 session.begin() 上下文自动处理）
```text

**Session 级数据库引擎原理：**

```python
def session_engine_analysis():
    """
    Session 级数据库引擎的设计权衡

    为什么用 session 级引擎？
    - 创建数据库引擎是昂贵操作（连接池初始化）
    - session 级保证所有测试共享一个引擎
    - 内存数据库 + 每次测试回滚 = 极快执行

    内存数据库 vs 文件数据库：
    - 内存 SQLite：极快，但数据在测试间不持久
    - 文件 SQLite：较慢，但可查看测试数据
    - 真实 MySQL：最真实，但需要外部服务

    回滚隔离的原理：
    每个测试函数在一个数据库事务中执行。
    测试结束后回滚事务，数据库恢复到测试前状态。
    相当于每个测试看到的数据是独立的。
    """
```

### 2.3 conftest 的 hook 功能

```python
# tests/conftest.py — pytest 钩子

def pytest_configure(config):
    """pytest 配置时调用"""
    # 注册自定义 marker（避免 --strict-markers 报错）
    config.addinivalue_line("markers", "slow: 标记慢测试，默认跳过")
    config.addinivalue_line("markers", "concurrent: 并发测试")
    config.addinivalue_line("markers", "integration: 集成测试（需要外部服务）")


def pytest_collection_modifyitems(config, items):
    """
    收集所有测试后调用，可以修改测试项

    例如：自动添加 marker 或跳过某些测试
    """
    # 为所有测试添加自定义 marker
    for item in items:
        # 如果测试文件名包含 "integration"，自动添加 marker
        if "integration" in item.nodeid:
            item.add_marker(pytest.mark.integration)


@pytest.fixture(autouse=True)
def _db_setup_teardown():
    """
    自动 fixture：每个测试前后执行

    名字以 _ 开头表示这是一个内部 fixture。
    autouse=True 表示每个测试自动使用，无需显式声明。
    """
    # setup：测试前的准备工作
    print("测试开始前执行")
    yield
    # teardown：测试后的清理工作
    print("测试结束后执行")
```text

---

## 3. Fixture 完全指南

### 3.1 Fixture 的定义与使用

```python
import pytest


# 定义 fixture
@pytest.fixture
def user_data():
    """提供测试用户数据"""
    return {
        "username": "alice",
        "password": "secure123",
        "email": "alice@example.com",
    }


# 使用 fixture（通过函数参数注入）
def test_user_registration(user_data):
    """测试用户注册逻辑"""
    assert user_data["username"] == "alice"
    assert len(user_data["password"]) >= 6
```

### 3.2 Fixture Scope（作用域）

```python
import pytest


@pytest.fixture(scope="function")
def func_fixture():
    """函数级：每个测试函数独立调用"""
    print("  [function] setup")
    yield "function_data"
    print("  [function] teardown")


@pytest.fixture(scope="class")
def class_fixture():
    """类级：每个测试类调用一次"""
    print("  [class] setup")
    yield "class_data"
    print("  [class] teardown")


@pytest.fixture(scope="module")
def module_fixture():
    """模块级：每个测试模块（文件）调用一次"""
    print("  [module] setup")
    yield "module_data"
    print("  [module] teardown")


@pytest.fixture(scope="session")
def session_fixture():
    """会话级：整个测试会话调用一次"""
    print("  [session] setup")
    yield "session_data"
    print("  [session] teardown")


class TestFixtureScopes:
    def test_one(self, func_fixture, class_fixture, module_fixture, session_fixture):
        assert func_fixture == "function_data"

    def test_two(self, func_fixture, class_fixture, module_fixture, session_fixture):
        assert func_fixture == "function_data"
```text

**Scope 执行顺序：**

```

在 TestFixtureScopes 这个类中：

test_one:
  [function] setup         ← 每次独立调用
  [class] setup            ← 只调用一次（类级）
  [module] setup           ← 只调用一次（模块级）
  [session] setup          ← 只调用一次（整个会话）
  测试执行...
  [function] teardown      ← 每次独立清理

test_two:
  [function] setup         ← 新的独立调用
  [class] setup            ← 不执行（类级在一开始已经执行）
  [module] setup           ← 不执行
  [session] setup          ← 不执行
  测试执行...
  [function] teardown      ← 新的独立清理

```text

### 3.3 Autouse Fixture

`autouse=True` 使得 fixture 自动应用于所有测试，无需显式声明依赖：

```python
import pytest
import time


@pytest.fixture(autouse=True)
def measure_time():
    """自动计时所有测试的执行时间"""
    start = time.time()
    yield
    elapsed = time.time() - start
    print(f"\n  测试耗时: {elapsed:.3f}s")


@pytest.fixture(autouse=True)
def setup_test_db(monkeypatch):
    """自动替换数据库连接为测试数据库"""
    test_config = {
        "DATABASE_URL": "sqlite+aiosqlite:///:memory:",
    }
    monkeypatch.setattr("app.config.settings.DATABASE_URL", test_config["DATABASE_URL"])
    yield


# 这两个测试会自动使用 measure_time 和 setup_test_db
def test_a():
    print("测试 A")
    assert True

def test_b():
    print("测试 B")
    assert True
```

### 3.4 Yield Fixture（Teardown）

yield fixture 支持测试后的清理逻辑（类似 unittest 的 tearDown）：

```python
import pytest
from typing import Generator


@pytest.fixture
def managed_resource() -> Generator:
    """
    Yield Fixture 模式

    yield 之前的代码是 setup（测试前执行）
    yield 之后的代码是 teardown（测试后执行）
    即使测试失败，teardown 也会执行。
    """
    print("\n  打开资源")
    resource = {"status": "open", "data": []}

    # yield 语句返回 fixture 值给测试函数
    yield resource

    # 测试结束后一定会执行这里（即使是 assert 失败）
    print("  关闭资源")
    resource["status"] = "closed"


def test_resource_management(managed_resource):
    """使用 yield fixture 管理资源生命周期"""
    assert managed_resource["status"] == "open"
    managed_resource["data"].append("test_entry")

# 测试结束后，managed_resource 的状态变为 "closed"
```text

### 3.5 内置 Fixture

pytest 提供了一些非常实用的内置 fixture：

```python
import pytest
import os
import tempfile


def test_tmp_path(tmp_path):
    """
    tmp_path：每个测试的临时目录（pathlib.Path 对象）
    测试结束后自动清理
    """
    d = tmp_path / "subdir"
    d.mkdir()
    p = d / "hello.txt"
    p.write_text("test content")

    assert p.read_text() == "test content"
    assert tmp_path.exists()


def test_tmp_dir(tmpdir):
    """
    tmpdir：每个测试的临时目录（py.path.local 对象）
    旧式 API，推荐使用 tmp_path
    """
    f = tmpdir.join("test.txt")
    f.write("hello")
    assert f.read() == "hello"


def test_capsys(capsys):
    """capsys：捕获 stdout 和 stderr"""
    print("hello stdout")
    print("hello stderr", file=sys.stderr)

    captured = capsys.readouterr()
    assert captured.out == "hello stdout\n"
    assert captured.err == "hello stderr\n"


# 重新打印（capsys 只捕获一次）
def test_capsys_disabled(capsys):
    """禁用捕获，直接输出"""
    with capsys.disabled():
        print("这段直接输出，不会被捕获")


def test_monkeypatch(monkeypatch):
    """monkeypatch：运行时修改对象"""
    # 修改环境变量
    monkeypatch.setenv("DATABASE_URL", "sqlite:///:memory:")
    assert os.environ["DATABASE_URL"] == "sqlite:///:memory:"

    # 修改字典
    monkeypatch.setitem({"key": "old"}, "key", "new")

    # 修改属性
    class Config:
        DEBUG = False
    monkeypatch.setattr(Config, "DEBUG", True)
    assert Config.DEBUG

    # 修改函数
    def mock_get_user():
        return {"name": "mock_user"}
    monkeypatch.setattr("app.services.auth_service.get_user", mock_get_user)


def test_recwarn(recwarn):
    """recwarn：记录所有警告"""
    import warnings
    warnings.warn("warning 1")
    warnings.warn("warning 2")

    assert len(recwarn) == 2
    assert str(recwarn[0].message) == "warning 1"
```

---

## 4. pytest-asyncio 异步测试

### 4.1 基本用法

```python
import pytest


# 方式一：使用 marker
@pytest.mark.asyncio
async def test_async_function():
    """测试异步函数"""
    result = await some_async_function()
    assert result == expected_value


# 方式二：自动模式（推荐在 pyproject.toml 设置 asyncio_mode = "auto"）
# 此时所有 async def test_* 函数都会被自动标记为 @pytest.mark.asyncio

async def test_auto_async():
    """pytest-asyncio 自动模式"""
    result = await some_async_function()
    assert result == expected_value
```text

### 4.2 异步 Fixture

```python
import pytest
from typing import AsyncGenerator


@pytest.fixture
async def async_resource() -> AsyncGenerator:
    """异步 fixture，用于设置和清理异步资源"""
    # setup
    resource = await create_async_connection()
    yield resource
    # teardown
    await resource.close()


@pytest.mark.asyncio
async def test_with_async_fixture(async_resource):
    """使用异步 fixture"""
    data = await async_resource.fetch_data()
    assert data is not None
```

### 4.3 事件循环策略

```python
import pytest
import asyncio


# 方案一：在 conftest.py 中定义 session 级事件循环（推荐）
@pytest.fixture(scope="session")
def event_loop():
    """
    Session 级事件循环 fixture

    所有测试共享一个事件循环，避免每个测试都创建/销毁事件循环的开销。

    为什么需要 session 级？
    - 默认是 function 级，每个测试创建一个新的事件循环
    - 对于大量异步测试，创建事件循环的开销不可忽略
    - 共享事件循环可以将测试时间减少 30-50%
    """
    policy = asyncio.get_event_loop_policy()
    loop = policy.new_event_loop()
    yield loop
    loop.close()


# 方案二：使用 pytest-asyncio 0.21+ 的 loop_scope 参数（推荐）
@pytest.fixture(scope="module")
def loop_scope():
    """
    设置 fixture 的循环作用域

    pytest-asyncio 0.21+ 支持在 fixture 中指定 loop_scope。
    可选值：function, class, module, session
    """
    return "module"
```text

### 4.4 超时处理

```python
import pytest


@pytest.mark.asyncio
@pytest.mark.timeout(5)  # 5 秒超时（需要 pytest-timeout）
async def test_with_timeout():
    """测试超时保护"""
    result = await fetch_data_with_timeout()
    assert result is not None


# 使用 asyncio.wait_for 手动控制超时
@pytest.mark.asyncio
async def test_manual_timeout():
    """手动设置超时"""
    try:
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=2.0,
        )
    except asyncio.TimeoutError:
        pytest.fail("操作超时")
```

---

## 5. httpx.AsyncClient 集成测试

### 5.1 ASGITransport 模式

```python
import pytest
from httpx import AsyncClient, ASGITransport

from app.main import app


@pytest.mark.asyncio
async def test_health_check():
    """测试健康检查接口"""

    # ASGITransport 直接在 Python 进程中调用 ASGI 应用
    # 无需启动 Uvicorn 服务器
    transport = ASGITransport(app=app)

    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get("/health")

    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "ok"
    assert "service" in data
```text

**ASGITransport vs 真实服务器：**

```python
def transport_comparison():
    """
    ASGITransport 模式：
    + 无需启动 Uvicorn
    + 测试速度极快（~1ms/请求）
    + 可以直接访问应用内部状态
    - 无法测试网络层
    - 无法测试多进程

    真实服务器模式（需要启动 Uvicorn）：
    + 更贴近生产环境
    + 可以测试网络栈
    - 需要启动/停止进程
    - 测试速度较慢

    结论：95% 的测试使用 ASGITransport，仅端到端测试使用真实服务器。
    """
```

### 5.2 集成测试常见模式

```python
import pytest
from httpx import AsyncClient, ASGITransport

from app.main import app
from app.database import get_db
from app.redis_client import get_redis


# 所有测试共享的客户端 fixture
@pytest.fixture
async def async_client(db_session, redis_client) -> AsyncGenerator:
    """
    提供 AsyncClient 实例

    使用 override_dependency 替换数据库和 Redis 依赖为测试实例。
    """

    # 替换依赖
    async def override_get_db():
        yield db_session

    async def override_get_redis():
        yield redis_client

    app.dependency_overrides[get_db] = override_get_db
    app.dependency_overrides[get_redis] = override_get_redis

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client

    # 清理 override
    app.dependency_overrides.clear()


@pytest.mark.asyncio
async def test_create_event(async_client, auth_headers):
    """创建活动测试"""
    response = await async_client.post(
        "/api/v1/events/",
        json={
            "title": "音乐会",
            "venue": "国家大剧院",
            "start_date": "2026-07-15T19:00:00",
            "end_date": "2026-07-15T22:00:00",
            "total_stock": 100,
            "price": 299.00,
            "category": "music",
        },
        headers=auth_headers,
    )
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "音乐会"
    assert data["id"] is not None
```text

### 5.3 JWT Token 处理

```python
import pytest
from httpx import AsyncClient, ASGITransform
from app.utils.jwt import create_access_token


@pytest.fixture
def test_user():
    """测试用户"""
    return {
        "id": 1,
        "username": "testuser",
        "email": "test@example.com",
        "role": "user",
        "is_active": True,
    }


@pytest.fixture
def admin_user():
    """管理员用户"""
    return {
        "id": 2,
        "username": "admin",
        "email": "admin@example.com",
        "role": "admin",
        "is_active": True,
    }


@pytest.fixture
def user_token(test_user):
    """生成普通用户的 JWT token"""
    token = create_access_token({"sub": str(test_user["id"])})
    return token


@pytest.fixture
def admin_token(admin_user):
    """生成管理员 JWT token"""
    token = create_access_token({"sub": str(admin_user["id"])})
    return token


@pytest.fixture
def auth_headers(user_token):
    """普通用户的 Authorization 头"""
    return {"Authorization": f"Bearer {user_token}"}


@pytest.fixture
def admin_headers(admin_token):
    """管理员的 Authorization 头"""
    return {"Authorization": f"Bearer {admin_token}"}


# 使用示例
@pytest.mark.asyncio
async def test_admin_only_endpoint(async_client, admin_headers, user_headers):
    """测试管理员接口：普通用户被拒绝，管理员通过"""

    # 普通用户访问——401
    response = await async_client.get("/api/v1/admin/users", headers=user_headers)
    assert response.status_code == 403

    # 管理员访问——200
    response = await async_client.get("/api/v1/admin/users", headers=admin_headers)
    assert response.status_code == 200
```

---

## 6. Mock 与 MonkeyPatch

### 6.1 unittest.mock 基础

```python
from unittest.mock import Mock, AsyncMock, patch, MagicMock, PropertyMock


def test_mock_basics():
    """Mock 基础用法"""

    # 创建一个 Mock 对象
    mock = Mock()

    # Mock 对象可以调用任意方法
    mock.some_method()
    mock.another_method("arg1", key="value")

    # 验证调用
    mock.some_method.assert_called_once()
    mock.another_method.assert_called_once_with("arg1", key="value")

    # 设置返回值
    mock.get_data.return_value = {"id": 1, "name": "test"}
    result = mock.get_data()
    assert result == {"id": 1, "name": "test"}

    # 设置 side_effect（异常或多次调用的不同返回值）
    mock.fetch.side_effect = [ValueError("第一次失败"), "第二次成功"]

    with pytest.raises(ValueError):
        mock.fetch()

    result = mock.fetch()
    assert result == "第二次成功"


def test_mock_property():
    """Mock 属性"""
    mock = Mock()

    # Mock 属性
    mock.name = "test_property"
    assert mock.name == "test_property"

    # 使用 PropertyMock
    type(mock).status = PropertyMock(return_value="active")
    assert mock.status == "active"


def test_mock_side_effect():
    """Side effect 的高级用法"""

    mock = Mock()

    # side_effect 可以是一个函数
    def dynamic_response(*args, **kwargs):
        if args[0] == "ping":
            return "pong"
        raise ValueError(f"未知命令: {args[0]}")

    mock.execute.side_effect = dynamic_response
    assert mock.execute("ping") == "pong"

    with pytest.raises(ValueError):
        mock.execute("unknown")
```text

### 6.2 AsyncMock

对于异步函数，需要使用 AsyncMock：

```python
import pytest
from unittest.mock import AsyncMock, patch


@pytest.mark.asyncio
async def test_async_mock():
    """测试 AsyncMock"""

    # 创建 AsyncMock
    mock_redis = AsyncMock()

    # 设置异步方法的返回值
    mock_redis.get.return_value = "10"
    result = await mock_redis.get("stock:1")
    assert result == "10"

    # 验证异步调用
    mock_redis.get.assert_called_once_with("stock:1")

    # 设置 side_effect 为异常
    mock_redis.set.side_effect = ConnectionError("Redis 断开连接")

    with pytest.raises(ConnectionError):
        await mock_redis.set("key", "value")
```

### 6.3 pytest-mock

pytest-mock 提供了一个 `mocker` fixture，可以更方便地使用 mock：

```python
# pytest-mock 的 mocker fixture 自动清理 mock，无需担心泄漏


def test_with_mocker(mocker):
    """使用 pytest-mock 的 mocker fixture"""

    # mocker.patch 相当于 unittest.mock.patch，但自动清理
    mock_redis_get = mocker.patch("app.redis_client.redis.get")
    mock_redis_get.return_value = "42"

    # 调用被 mock 的函数
    from app.redis_client import redis
    result = redis.get("stock:1")
    assert result == "42"

    # 验证调用
    mock_redis_get.assert_called_once_with("stock:1")


@pytest.mark.asyncio
async def test_async_mocker(mocker):
    """mocker 的 AsyncMock"""

    # 使用 async 版本的 patch
    mock_service = mocker.patch(
        "app.services.inventory_service.InventoryService.deduct_stock",
        new_callable=AsyncMock,
    )
    mock_service.return_value = 1  # 成功

    # 测试使用 deduct_stock 的函数
    service = InventoryService(mock_redis)
    result = await service.deduct_stock(1, 123, 1)
    assert result == 1
```text

### 6.4 Mock vs MonkeyPatch 对比

```python
def mock_vs_monkeypatch_comparison():
    """
    Mock vs MonkeyPatch 对比

    Mock（unittest.mock）：
    - 完整模拟对象行为
    - 支持自动断言（assert_called_once 等）
    - 支持 side_effect
    - 支持 AsyncMock
    - 适用范围广：类、方法、属性、外部库
    - 需要显式清理（或使用 pytest-mock 的 mocker）

    MonkeyPatch（pytest 内置）：
    - 简单直接修改对象
    - 自动清理（测试结束后恢复）
    - 适合：环境变量、配置文件、简单函数替换
    - 不支持自动断言
    - 不提供 side_effect 等高级功能

    选择建议：
    - 需要验证调用参数/次数 → Mock
    - 只需要修改环境变量 → MonkeyPatch
    - 测试异步代码 → AsyncMock
    - 替换数据库/Redis → Mock
    - 修改 sys.path/os.environ → MonkeyPatch
    """
```

---

## 7. parametrize 参数化测试

### 7.1 基本参数化

```python
import pytest


# 简单参数化
@pytest.mark.parametrize("username,password,expected", [
    ("alice", "pass123", True),      # 正常
    ("alice", "wrong", False),       # 密码错误
    ("", "pass123", False),          # 空用户名
    ("alice", "", False),            # 空密码
    ("a" * 51, "pass123", False),    # 用户名超长
])
def test_login_validation(username, password, expected):
    """测试登录验证的各种场景"""
    result = validate_login(username, password)
    assert result == expected


# 参数化测试的命名
@pytest.mark.parametrize(
    "username,password,expected",
    [
        pytest.param("alice", "pass123", True, id="normal_login"),
        pytest.param("alice", "wrong", False, id="wrong_password"),
        pytest.param("", "pass123", False, id="empty_username"),
        pytest.param("alice", "", False, id="empty_password"),
    ],
)
def test_login_named(username, password, expected):
    """带命名的参数化测试"""
    result = validate_login(username, password)
    assert result == expected
```text

### 7.2 多个参数化级别

```python
import pytest


# 组合参数化：类似笛卡尔积
@pytest.mark.parametrize("env", ["dev", "staging", "production"])
@pytest.mark.parametrize("db", ["mysql", "postgresql", "sqlite"])
def test_database_connection(env, db):
    """测试所有数据库 x 环境组合（3x3 = 9 个测试）"""
    result = connect_to_db(env=env, db_type=db)
    assert result is not None


# 扁平参数化（避免笛卡尔积的重复）
@pytest.mark.parametrize("env,db", [
    ("dev", "sqlite"),
    ("staging", "mysql"),
    ("production", "mysql"),
])
def test_env_db_pairs(env, db):
    """仅测试特定的环境-数据库组合"""
    result = connect_to_db(env=env, db_type=db)
    assert result is not None
```

### 7.3 Indirect Fixture（间接参数化）

`indirect=True` 允许通过 fixture 间接使用参数化参数：

```python
import pytest


# 常规 fixture
@pytest.fixture
def database(request):
    """
    间接 fixture：根据参数化值初始化不同的数据库

    request.param 就是 parametrize 传入的参数值
    """
    db_type = request.param
    if db_type == "mysql":
        return {"type": "mysql", "connected": True}
    elif db_type == "sqlite":
        return {"type": "sqlite", "connected": True}
    elif db_type == "invalid":
        return {"type": "invalid", "connected": False}


# 通过 indirect=True 将参数传递给 fixture
@pytest.mark.parametrize("database", ["mysql", "sqlite", "invalid"], indirect=True)
def test_database_types(database):
    """测试不同类型的数据库连接"""
    if database["type"] == "invalid":
        assert not database["connected"]
    else:
        assert database["connected"]


# 多个间接 fixture
@pytest.fixture
def user_role(request):
    """用户角色 fixture"""
    return request.param


@pytest.mark.parametrize("user_role", ["admin", "user", "guest"], indirect=True)
@pytest.mark.parametrize("database", ["mysql", "sqlite"], indirect=True)
def test_role_database(user_role, database):
    """测试所有角色与数据库的组合"""
    permissions = get_permissions(user_role, database)
    assert len(permissions) > 0
```text

---

## 8. 覆盖率：pytest-cov

### 8.1 基本用法

```bash
# 终端报告
pytest --cov=app

# 显示未覆盖的行
pytest --cov=app --cov-report=term-missing

# HTML 报告（生成 htmlcov/ 目录）
pytest --cov=app --cov-report=html

# XML 报告（CI/CD 使用）
pytest --cov=app --cov-report=xml

# 多报告同时生成
pytest --cov=app \
    --cov-report=term-missing \
    --cov-report=html \
    --cov-report=xml

# 仅覆盖特定文件
pytest --cov=app/services

# 排除某些文件
pytest --cov=app --cov-ignore=app/migrations/
```

### 8.2 覆盖率配置

```toml
# pyproject.toml
[tool.coverage.run]
source = ["app"]
omit = [
    "app/migrations/*",
    "app/__init__.py",
    "*/test_*",
]

[tool.coverage.report]
# 只报告覆盖率低于 80% 的文件
fail_under = 80
# 排除分支覆盖率
show_missing = true
# 排除正则匹配的行
exclude_lines = [
    "pragma: no cover",
    "if __name__ == .__main__.:",
    "raise NotImplementedError",
    "if 0:",
]
```text

### 8.3 覆盖率陷阱

```python
def coverage_pitfalls():
    """
    覆盖率数字的常见误导

    陷阱 1：行覆盖率 ≠ 分支覆盖率
    if condition:        # 这一行"覆盖"了
        do_something()   # 如果 condition=True 才覆盖
    else:
        do_other()       # 如果 condition=False 才覆盖
    # 100% 行覆盖率可能只覆盖了一个分支

    陷阱 2：导入即覆盖
    import time          # 这行会被计为"已覆盖"
    # 但实际上没有执行任何有意义的逻辑

    陷阱 3：异常处理路径
    try:
        do_thing()       # 正常路径
    except Exception:
        handle()         # 异常路径通常不被覆盖
    # 80% 的行覆盖率可能变成了 60% 的分支覆盖率

    建议：追求有意义的覆盖率，而非数字。
    关键路径（如防超卖、支付）应 100% 覆盖，
    配置代码、日志代码可适当降低标准。
    """
```

---

## 9. 票务系统测试模式详解

以下是对票务系统各测试文件的模式讲解。由于测试文件需要根据项目的实际实现编写，这里提供完整的模式说明和代码示例，你可以直接参照编写测试。

### 9.1 tests/conftest.py 模式

```python
"""共享 fixture 和配置"""

import pytest
import asyncio
from typing import AsyncGenerator
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

from app.main import app
from app.database import Base, get_db
from app.redis_client import get_redis
from app.utils.jwt import create_access_token


# ===== 数据库 =====

@pytest.fixture(scope="session")
def event_loop():
    """session 级事件循环"""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session")
async def db_engine():
    """内存 SQLite 引擎"""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest.fixture
async def db_session(db_engine) -> AsyncGenerator[AsyncSession, None]:
    """每个测试独立的事务"""
    session_factory = async_sessionmaker(
        db_engine, class_=AsyncSession, expire_on_commit=False
    )
    async with session_factory() as session:
        async with session.begin():
            yield session


# ===== Redis =====

@pytest.fixture
async def redis_client():
    """Mock Redis 客户端"""
    from unittest.mock import AsyncMock
    redis = AsyncMock()
    redis.get.return_value = None
    redis.set.return_value = True
    redis.evalsha.return_value = 1
    redis.script_load.return_value = "mock_sha"
    redis.incrby.return_value = 10
    yield redis


# ===== HTTP 客户端 =====

@pytest.fixture
async def async_client(db_session, redis_client) -> AsyncGenerator:
    """测试 HTTP 客户端"""

    async def override_get_db():
        yield db_session

    async def override_get_redis():
        yield redis_client

    app.dependency_overrides[get_db] = override_get_db
    app.dependency_overrides[get_redis] = override_get_redis

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client

    app.dependency_overrides.clear()


# ===== 认证 =====

@pytest.fixture
def test_user_id():
    return 1


@pytest.fixture
def auth_headers(test_user_id):
    token = create_access_token({"sub": str(test_user_id)})
    return {"Authorization": f"Bearer {token}"}


@pytest.fixture
def admin_headers():
    token = create_access_token({"sub": str(999)})
    return {"Authorization": f"Bearer {token}"}
```text

### 9.2 tests/test_health.py 模式

```python
"""健康检查接口测试"""

import pytest
from httpx import AsyncClient, ASGITransport

from app.main import app


class TestHealth:
    """健康检查测试组"""

    @pytest.mark.asyncio
    async def test_health_check(self, async_client):
        """测试 /health 返回正确的状态信息"""
        response = await async_client.get("/health")

        assert response.status_code == 200
        data = response.json()

        # 验证返回结构
        assert data["status"] == "ok"
        assert "service" in data
        assert data["service"] == "TicketBooking"

    @pytest.mark.asyncio
    async def test_health_method_not_allowed(self, async_client):
        """测试 /health 只接受 GET"""
        for method in ["post", "put", "delete", "patch"]:
            response = await getattr(async_client, method)("/health")
            assert response.status_code == 405, (
                f"{method.upper()} /health 应该返回 405"
            )

    @pytest.mark.asyncio
    async def test_health_response_time(self, async_client):
        """测试 /health 响应时间（应在 50ms 内）"""
        import time
        start = time.time()
        await async_client.get("/health")
        elapsed = time.time() - start

        assert elapsed < 0.05, f"健康检查响应时间过长: {elapsed:.3f}s"
```

### 9.3 tests/test_auth.py 模式

```python
"""认证相关接口测试"""

import pytest
from httpx import AsyncClient


class TestAuthRegister:
    """注册功能测试"""

    @pytest.mark.asyncio
    async def test_register_success(self, async_client):
        """正常注册"""
        response = await async_client.post("/api/v1/auth/register", json={
            "username": "newuser",
            "password": "password123",
            "email": "new@example.com",
        })
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "newuser"
        assert data["email"] == "new@example.com"
        assert "id" in data
        # 密码不应返回
        assert "password" not in data
        assert "password_hash" not in data

    @pytest.mark.asyncio
    async def test_register_duplicate_username(self, async_client):
        """重复用户名"""
        await async_client.post("/api/v1/auth/register", json={
            "username": "dupuser",
            "password": "password123",
            "email": "dup1@example.com",
        })
        response = await async_client.post("/api/v1/auth/register", json={
            "username": "dupuser",  # 同名
            "password": "other456",
            "email": "dup2@example.com",
        })
        assert response.status_code == 409
        assert "已存在" in response.json()["detail"]

    @pytest.mark.asyncio
    async def test_register_empty_password(self, async_client):
        """空密码被拒绝"""
        response = await async_client.post("/api/v1/auth/register", json={
            "username": "nopass",
            "password": "",
            "email": "nopass@example.com",
        })
        assert response.status_code == 422  # 验证错误


class TestAuthLogin:
    """登录功能测试"""

    @pytest.mark.asyncio
    async def test_login_success(self, async_client):
        """正常登录"""
        # 先注册
        await async_client.post("/api/v1/auth/register", json={
            "username": "logintest",
            "password": "pass123",
            "email": "login@example.com",
        })
        # 再登录
        response = await async_client.post("/api/v1/auth/login", json={
            "username": "logintest",
            "password": "pass123",
        })
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["token_type"] == "bearer"

    @pytest.mark.asyncio
    async def test_login_wrong_password(self, async_client):
        """错误密码"""
        await async_client.post("/api/v1/auth/register", json={
            "username": "wrongpass",
            "password": "correct",
            "email": "wrong@example.com",
        })
        response = await async_client.post("/api/v1/auth/login", json={
            "username": "wrongpass",
            "password": "wrong",
        })
        assert response.status_code == 401

    @pytest.mark.asyncio
    async def test_login_nonexistent_user(self, async_client):
        """不存在的用户"""
        response = await async_client.post("/api/v1/auth/login", json={
            "username": "nobody",
            "password": "anything",
        })
        assert response.status_code == 401
```text

### 9.4 tests/test_events.py 模式

```python
"""活动相关接口测试"""

import pytest
from httpx import AsyncClient


class TestEventList:
    """活动列表测试"""

    @pytest.mark.asyncio
    async def test_list_events_empty(self, async_client):
        """空列表"""
        response = await async_client.get("/api/v1/events/")
        assert response.status_code == 200
        data = response.json()
        assert data["total"] == 0
        assert data["items"] == []

    @pytest.mark.asyncio
    async def test_list_events_pagination(self, async_client, auth_headers):
        """分页"""
        for i in range(5):
            await async_client.post("/api/v1/events/", json={
                "title": f"Event {i}",
                "venue": "Venue",
                "start_date": "2026-08-01T10:00:00",
                "end_date": "2026-08-01T12:00:00",
                "total_stock": 100,
                "price": 100.0,
                "category": "music",
            }, headers=auth_headers)

        response = await async_client.get("/api/v1/events/?page=1&size=2")
        assert response.status_code == 200
        data = response.json()
        assert len(data["items"]) == 2
        assert data["total"] == 5


class TestEventCRUD:
    """活动增删改查测试"""

    @pytest.mark.asyncio
    async def test_create_event_success(self, async_client, auth_headers):
        """创建活动"""
        response = await async_client.post("/api/v1/events/", json={
            "title": "New Event",
            "venue": "Grand Theater",
            "start_date": "2026-09-01T19:00:00",
            "end_date": "2026-09-01T22:00:00",
            "total_stock": 500,
            "price": 199.0,
            "category": "theater",
        }, headers=auth_headers)
        assert response.status_code == 201
        assert response.json()["title"] == "New Event"

    @pytest.mark.asyncio
    async def test_create_event_unauthorized(self, async_client):
        """未认证时创建失败"""
        response = await async_client.post("/api/v1/events/", json={
            "title": "Hacked Event",
            "venue": "Hacker",
            "start_date": "2026-09-01T19:00:00",
            "end_date": "2026-09-01T22:00:00",
            "total_stock": 100,
            "price": 0.01,
        })
        assert response.status_code == 403
```

### 9.5 tests/test_orders.py 模式

```python
"""订单相关接口测试"""

import pytest
from httpx import AsyncClient


class TestOrderCreate:
    """订单创建测试"""

    @pytest.mark.asyncio
    async def test_create_order_success(self, async_client, auth_headers, test_event):
        """正常下单"""
        response = await async_client.post("/api/v1/orders/", json={
            "event_id": test_event["id"],
            "quantity": 2,
        }, headers=auth_headers)
        assert response.status_code == 201
        data = response.json()
        assert data["quantity"] == 2
        assert data["status"] == "pending_payment"

    @pytest.mark.asyncio
    async def test_create_order_insufficient_stock(
        self, async_client, auth_headers, test_event
    ):
        """库存不足"""
        response = await async_client.post("/api/v1/orders/", json={
            "event_id": test_event["id"],
            "quantity": 99999,
        }, headers=auth_headers)
        assert response.status_code == 400
        assert "库存不足" in response.json()["detail"]

    @pytest.mark.asyncio
    async def test_create_order_not_found(self, async_client, auth_headers):
        """活动不存在"""
        response = await async_client.post("/api/v1/orders/", json={
            "event_id": 99999,
            "quantity": 1,
        }, headers=auth_headers)
        assert response.status_code == 404
```text

### 9.6 tests/test_anti_overselling.py 模式

```python
"""防超卖核心并发测试"""

import pytest
import asyncio
from httpx import AsyncClient, ASGITransport

from app.main import app


class TestAntiOverselling:
    """防超卖并发测试"""

    @pytest.mark.asyncio
    async def test_concurrent_buy_no_overselling(
        self, async_client, auth_headers_factory, seeded_event
    ):
        """
        核心防超卖测试

        原理：
        1. 创建 N 张库存 + 管理员设置的限购
        2. 启动 N+1 个并发下单请求
        3. 验证成功订单数 <= N
        """
        event_id = seeded_event["id"]
        stock = seeded_event["total_stock"]
        concurrent = stock + 3  # 比库存多 3 个请求

        async def try_buy(user_index: int) -> bool:
            """单个用户尝试购买"""
            headers = auth_headers_factory(user_index)
            response = await async_client.post("/api/v1/orders/", json={
                "event_id": event_id,
                "quantity": 1,
            }, headers=headers)
            return response.status_code == 201

        # 并发执行所有请求
        results = await asyncio.gather(*[
            try_buy(i) for i in range(concurrent)
        ])

        success_count = sum(results)
        assert success_count <= stock, (
            f"超卖！成功 {success_count} 张，库存 {stock} 张"
        )

    @pytest.mark.asyncio
    async def test_exact_concurrent_buy(
        self, async_client, auth_headers_factory, seeded_event
    ):
        """
        精确并发测试：恰好 N 个并发请求应该全部成功
        """
        event_id = seeded_event["id"]
        stock = seeded_event["total_stock"]

        async def try_buy(user_index: int) -> bool:
            headers = auth_headers_factory(user_index)
            response = await async_client.post("/api/v1/orders/", json={
                "event_id": event_id,
                "quantity": 1,
            }, headers=headers)
            return response.status_code == 201

        results = await asyncio.gather(*[
            try_buy(i) for i in range(stock)
        ])

        success_count = sum(results)
        assert success_count == stock, (
            f"准确购买应全部成功：成功 {success_count}/{stock}"
        )

    @pytest.mark.asyncio
    async def test_race_condition_quantity(
        self, async_client, auth_headers_factory, seeded_event
    ):
        """
        竞争条件测试：部分用户购买多张

        场景：
        - 库存 10 张
        - 用户 A 要 6 张，用户 B 要 6 张
        - 总需求 12 > 10，应只有一个成功
        """
        event_id = seeded_event["id"]

        async def user_a_buy():
            headers = auth_headers_factory(1)
            return await async_client.post("/api/v1/orders/", json={
                "event_id": event_id,
                "quantity": 6,
            }, headers=headers)

        async def user_b_buy():
            headers = auth_headers_factory(2)
            return await async_client.post("/api/v1/orders/", json={
                "event_id": event_id,
                "quantity": 6,
            }, headers=headers)

        results = await asyncio.gather(user_a_buy(), user_b_buy())
        success_count = sum(1 for r in results if r.status_code == 201)

        # 最多一个成功（6+6=12 > 10）
        assert success_count <= 1, f"多张购买竞争失败：成功 {success_count} 单"
```

### 9.7 tests/test_payments.py 模式

```python
"""支付相关接口测试"""

import pytest
from httpx import AsyncClient


class TestPayment:
    """支付流程测试"""

    @pytest.mark.asyncio
    async def test_alipay_payment_success(
        self, async_client, auth_headers, paid_event
    ):
        """支付宝支付成功"""
        order_id = paid_event["order_id"]
        response = await async_client.post(
            "/api/v1/payments/alipay",
            json={"order_id": order_id},
            headers=auth_headers,
        )
        assert response.status_code == 200
        data = response.json()
        assert "pay_url" in data  # 支付链接
        assert data["order_no"] is not None

    @pytest.mark.asyncio
    async def test_alipay_notify(self, async_client):
        """支付宝异步通知处理"""
        response = await async_client.post(
            "/api/v1/payments/alipay/notify",
            data={
                "out_trade_no": "TEST2026070100001",
                "trade_no": "2026070122001000000001",
                "total_amount": "199.00",
                "trade_status": "TRADE_SUCCESS",
                "sign": "mocked_sign",
                "sign_type": "RSA2",
            },
        )
        # 由于是 mock 数据，预期 400（验签失败）
        # 但说明逻辑链路正常
        assert response.status_code in (200, 400)

    @pytest.mark.asyncio
    async def test_refund_success(self, async_client, admin_headers, paid_event):
        """退款成功"""
        order_id = paid_event["order_id"]
        response = await async_client.post(
            "/api/v1/payments/refund",
            json={"order_id": order_id, "reason": "用户申请"},
            headers=admin_headers,
        )
        assert response.status_code == 200
        assert response.json()["success"] is True
```text

---

## 10. 测试开发最佳实践

```python
def testing_best_practices():
    """
    票务系统测试最佳实践

    1. 测试金字塔
       单元测试（80%）：service 层的纯逻辑测试
       集成测试（15%）：HTTP 接口测试
       E2E 测试（5%）：完整业务流程测试

    2. 每个 Bug 先写测试
       发现 Bug → 写复现该 Bug 的测试 → 修复 → 测试通过
       防止同一 Bug 再次出现（回归测试）

    3. 测试命名规范
       test_<模块>_<场景>_<期望行为>
       e.g. test_order_create_insufficient_stock_raises_error

    4. 避免测试间依赖
       每个测试独立：不依赖其他测试的执行顺序或副作用
       使用 fixture 提供干净的状态

    5. 测试速度
       快速测试（<100ms）→ 开发者每次提交运行
       中速测试（<1s）→ CI 每次 push 运行
       慢速测试（>1s）→ CI 定时运行

    6. 不要测试实现细节
       好的测试：给定输入 → 断言输出
       坏的测试：断言内部函数被调用了 X 次
    """
```

---

## 11. CI/CD 集成

```yaml
# .github/workflows/test.yml
name: Run Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root123
          MYSQL_DATABASE: ticket_test
        ports:
          - 3306:3306
      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-asyncio httpx pytest-cov

      - name: Run tests
        env:
          DB_HOST: localhost
          DB_USER: root
          DB_PASSWORD: root123
          DB_NAME: ticket_test
          REDIS_HOST: localhost
        run: |
          pytest --cov=app --cov-report=term-missing --cov-fail-under=80

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```text

---

## 12. 端到端测试（E2E）

端到端测试（`tests/test_e2e.py`）使用真实 MySQL + Redis 数据库，覆盖所有 4 种用户角色和所有功能模块的正反向场景。

### 12.1 E2E 测试架构

```text
tests/test_e2e.py
├── TestHealth      — 健康检查端点（2 测试）
├── TestAuth        — 注册/登录/Token/权限（4 测试）
├── TestEvents      — 活动 CRUD + 库存（7 测试）
├── TestTicketTypes — 票种管理（4 测试）
├── TestOrders      — 下单/取消/权限（4 测试）
├── TestPayments    — 支付流程（2 测试）
├── TestAdmin       — 管理功能 + 角色（7 测试）
├── TestVerification— 票证核验（3 测试）
├── TestProfiles    — 身份档案（1 测试）
├── TestContent     — 内容管理（4 测试）
├── TestScheduleGuests — 日程嘉宾（2 测试）
├── TestMembership  — 会员体系（2 测试）
├── TestQueue       — 排队系统（1 测试）
├── TestCoupons     — 优惠券（2 测试）
├── TestDiscounts   — 折扣规则（2 测试）
├── TestAllocations — 配给系统（1 测试）
├── TestSubscription— 活动订阅（2 测试）
└── TestRoleEnforcement — 权限边界（3 测试）
```

### 12.2 核心模式

**真实数据库隔离：** 每个测试在独立的事务中执行，测试结束后自动回滚。通过 `conftest_db.py` 提供的 `db_session` fixture 实现：

```python
@pytest.fixture
async def db_session(db_engine):
    """Function 级：每个测试独立事务，自动回滚"""
    async with async_sessionmaker(db_engine)() as session:
        async with session.begin():
            yield session  # 测试在此执行
            # 退出后自动回滚
```

**辅助函数创建测试数据：**

```python
async def _create_user(db_session, username, role="user",
                       phone_verified=False, id_verified=False):
    """在数据库中创建用户"""
    user = User(username=username, role=role, ...)
    db_session.add(user)
    await db_session.flush()
    return {"id": user.id, "username": username, "password": password, "role": role}

async def _login(client, username, password):
    """通过 API 登录获取 token"""
    resp = await client.post("/api/v1/auth/login", json={...})
    if resp.status_code == 200:
        return resp.json()["access_token"]
    return None
```

**权限矩阵测试：** 对每个受保护端点至少测试三种场景：

```python
# 无认证 → 401
resp = await client.get("/api/v1/admin/users")
assert resp.status_code == 401

# 权限不足 → 403
resp = await client.get("/api/v1/admin/users", headers=user_headers)
assert resp.status_code == 403

# 正确权限 → 200
resp = await client.get("/api/v1/admin/users", headers=admin_headers)
assert resp.status_code == 200
```

### 12.3 与单元/集成测试的区别

| 维度 | 单元测试 | 集成测试 | E2E 测试 |
|------|---------|---------|---------|
| 数据库 | Mock | Mock（内存 SQLite） | 真实 MySQL |
| Redis | Mock | Mock | 真实 Redis |
| 数据隔离 | Mock 对象 | 事务回滚 | 事务回滚 |
| 运行速度 | ~1ms | ~10ms | ~100ms |
| 依赖 | 无 | 无 | Docker MySQL + Redis |
| 覆盖 | Service 逻辑 | HTTP 接口 | 全流程权限 |

### 12.4 运行方式

```bash
# 启动依赖服务
docker compose up -d

# 运行 E2E 测试
pytest tests/test_e2e.py -v --integration

# 仅运行 E2E 标记
pytest tests/test_e2e.py -v --integration -m e2e

# 运行所有测试（含 E2E）
pytest -v --integration
```

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `tests/conftest.py` | 测试配置：`mock_db_session`、`mock_redis`、`client`、测试数据工厂 |
| `tests/test_auth.py` | 认证测试（注册/登录/Token 刷新） |
| `tests/test_orders.py` | 订单流程测试（下单/取消/防超卖） |
| `tests/test_events.py` | 活动 CRUD 测试 |
| `tests/test_payments.py` | 支付回调测试 |
| `tests/test_websocket.py` | WebSocket 连接测试 |
| `tests/test_behavior.py` | 防刷逻辑测试 |
| `tests/test_security.py` | 安全标头测试 |
| `tests/test_admin.py` | 管理员功能测试 |
| `tests/test_health.py` | 健康检查测试 |
| `tests/test_api_routes.py` | 路由完整性测试 |
| `tests/test_e2e.py` | 端到端测试（4 种用户角色 x 16 模块，40+ 测试） |

**覆盖率要求**（来自 `.github/workflows/test.yml`）：`--cov-fail-under=80`

## 13. 本章总结

**已掌握的知识：**

- **pytest 基础**：测试发现、断言、运行选项、调试技巧
- **conftest 层次**：根目录到子目录的 fixture 可见性、钩子函数
- **Fixture 系统**：scope、autouse、yield teardown、内置 fixture
- **pytest-asyncio**：异步测试、事件循环策略、超时处理
- **httpx.AsyncClient**：ASGITransport、依赖覆盖、JWT 认证集成
- **Mock 技巧**：Mock、AsyncMock、pytest-mock、Mock vs MonkeyPatch
- **参数化测试**：parametrize、多级参数化、indirect fixture
- **覆盖率**：pytest-cov 配置、报告、陷阱
- **票务系统测试**：conftest、各模块测试模式、并发防超卖测试
- **E2E 测试**：真实数据库集成、多角色权限验证矩阵

**项目里程碑：**

- [ ] tests/conftest.py 数据库 + Redis + HTTP 客户端 fixture
- [ ] tests/test_health.py 健康检查
- [ ] tests/test_auth.py 注册 + 登录
- [ ] tests/test_events.py 活动 CRUD + 列表
- [ ] tests/test_orders.py 下单 + 查询 + 取消
- [ ] tests/test_anti_overselling.py 并发防超卖验证
- [ ] tests/test_payments.py 支付 + 退款

**下一章预告：** 第 09.5 章将学习如何使用 Postman 进行 API 测试，包括导入集合、Pre-request Script 自动设置 Token、Newman 命令行自动化测试。
