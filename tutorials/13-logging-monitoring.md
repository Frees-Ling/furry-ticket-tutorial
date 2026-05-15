# 第13章：日志/监控/健康检查 — structlog + Prometheus 零基础到实战

## 本章学习目标

本章将深入学习 Python 日志架构、structlog 结构化日志、Prometheus 指标监控和健康检查。我们不仅会学习理论知识，还会**逐行解释本项目中的实际代码**，让你理解每一行代码的含义和作用。

**学习路径：**

1. Python logging 架构（Logger/Handler/Filter/Formatter/Propagation）
2. structlog 结构化日志完全教程
3. 逐行解读本项目 `app/utils/logging_config.py`
4. 逐行解读本项目 `app/middleware/logging.py`（中间件）
5. Prometheus 四大指标类型
6. 为本项目的票务系统添加业务监控指标
7. 健康检查（liveness + readiness）
8. 逐行解读本项目 `app/main.py` 中的健康检查

---

## 1. Python Logging 架构

### 1.1 五大核心组件

Python 的 `logging` 模块基于一种**分层的事件分发系统**，由五个核心组件构成：

```text
Logger（日志记录器）
  ├── 应用程序调用入口
  ├── 可以设置级别（level）
  └── 可以设置 propagate 属性

Handler（处理器）
  ├── 将日志记录发送到目标（控制台/文件/网络）
  ├── 每个 Logger 可以有多个 Handler
  └── 可以设置级别（独立于 Logger）

Formatter（格式化器）
  ├── 定义日志输出的格式
  └── 字符串模板：%(name)s %(levelname)s %(message)s

Filter（过滤器）
  ├── 更精细地控制哪些日志被处理
  └── 可以根据任意条件过滤

LogRecord（日志记录）
  └── 一次日志调用的完整信息载体
```

### 1.2 Logger 层级结构

Python Logger 按**名称**形成层级结构，名称用点号分隔：

```text
root (根 Logger，名称为 "")
├── "app"
│   ├── "app.main"
│   ├── "app.routers"
│   │   ├── "app.routers.orders"
│   │   └── "app.routers.auth"
│   ├── "app.services"
│   │   ├── "app.services.order_service"
│   │   └── "app.services.inventory_service"
│   └── "app.middleware"
└── "uvicorn"
    ├── "uvicorn.access"
    └── "uvicorn.error"
```

**子 Logger 自动继承父 Logger 的设置：**

```python
# 设置父 Logger
logger_router = logging.getLogger("app.routers")
logger_router.setLevel(logging.INFO)

# 子 Logger 自动继承 INFO 级别
logger_order = logging.getLogger("app.routers.orders")
print(logger_order.level)  # → 0 (NOTSET，表示继承父级)
print(logger_order.getEffectiveLevel())  # → 20 (INFO)
```text

### 1.3 日志传播（Propagation）

这是 Python logging 最核心也最容易引起**重复日志**的机制。

**传播规则：**

```

当 Logger 产生一条日志时：

1. 该日志被发送给 Logger 本身的所有 Handler
2. 如果 propagate=True（默认），日志会继续传播给父 Logger
3. 父 Logger 的 Handler 也会处理这条日志
4. 一直传播到 root Logger

```text

**图解：**

```

getLogger("app.routers.orders").info("hello")
                │
                ▼
┌──────────────────────────────────┐
│ Logger("app.routers.orders")      │
│ 设置 level=INFO                    │
│ 没有添加 Handler                   │
│ propagate=True（默认）              │
│ 处理日志 → 传播给父 Logger          │
└────────────┬─────────────────────┘
             │
             ▼
┌──────────────────────────────────┐
│ Logger("app.routers")             │
│ 设置 level=INFO                    │
│ Handler: ConsoleHandler           │ → 打印到控制台
│ propagate=True                     │
│ 处理日志 → 传播给父 Logger          │
└────────────┬─────────────────────┘
             │
             ▼
┌──────────────────────────────────┐
│ Logger("app")                     │
│ 设置 level=DEBUG                   │
│ Handler: FileHandler              │ → 写入文件（重复了！）
│ propagate=True                     │
│ 处理日志 → 传播给父 Logger          │
└────────────┬─────────────────────┘
             │
             ▼
┌──────────────────────────────────┐
│ Root Logger                       │
│ 设置 level=WARNING                 │
│ Handler: StreamHandler            │ → 如果 level >= WARNING 才处理
└──────────────────────────────────┘

```text

**如果 `app.routers` 和 `app` 都添加了 Handler，那么 orders 的日志会被打印两次！**

**避免重复日志的两种方法：**

```python
# 方法 1：只在具体的 Logger 上添加 Handler，关闭传播
logger = logging.getLogger("app.routers.orders")
logger.addHandler(console_handler)
logger.propagate = False  # ← 关键：阻止传播到父 Logger

# 方法 2：只在根 Logger 上添加 Handler，子 Logger 只设置级别
logging.basicConfig(level=logging.INFO, format="%(message)s")
logging.getLogger("app.routers").setLevel(logging.DEBUG)
logging.getLogger("app.routers.orders").setLevel(logging.WARNING)
# 所有日志最终由根 Logger 的 Handler 统一输出
```

### 1.4 日志级别

| 级别 | 数值 | 用途 |
| ------ | ------ | ------ |
| `CRITICAL` | 50 | 严重错误，程序可能无法继续运行 |
| `ERROR` | 40 | 错误，某项操作失败但程序可以继续 |
| `WARNING` | 30 | 警告，潜在的问题或不推荐的做法 |
| `INFO` | 20 | 正常运行的信息（订单创建、用户登录等） |
| `DEBUG` | 10 | 调试信息，仅在开发时使用 |
| `NOTSET` | 0 | 继承父 Logger 的级别 |

**当 Logger 的级别为 WARNING 时：**

```text
logger.debug("调试信息")    → 不输出（10 < 30）
logger.info("普通信息")     → 不输出（20 < 30）
logger.warning("警告")     → 输出（30 >= 30）
logger.error("错误")       → 输出（40 >= 30）
logger.critical("严重")    → 输出（50 >= 30）
```

**在本项目中的应用建议：**

```python
# 各级别的使用场景
logger.debug("扣减库存参数", event_id=event_id, quantity=quantity, user_id=user_id)
# ↑ 开发时看数据流转，生产环境通常关闭

logger.info("订单创建成功", order_no="ORD001", amount=680.00)
# ↑ 业务关键节点，了解系统运行状态

logger.warning("支付回调重试", order_no="ORD002", retry_count=3)
# ↑ 非错误但值得关注（支付回调重试、限流触发等）

logger.error("数据库连接失败", db_host="localhost", error="timeout")
# ↑ 需要人介入处理的问题

logger.critical("库存数据不一致", event_id=1, db_stock=50, redis_stock=100)
# ↑ 需要立即处理的严重问题
```text

### 1.5 dictConfig — 声明式配置

推荐使用 `logging.config.dictConfig` 进行配置，而非在代码中逐个创建组件：

```python
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,  # 是否禁用已存在的 Logger（通常设为 False）

    # 格式化器
    "formatters": {
        "standard": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
            "datefmt": "%Y-%m-%d %H:%M:%S",
        },
        "json": {
            "format": '{"time":"%(asctime)s","level":"%(levelname)s","logger":"%(name)s","message":"%(message)s"}',
        },
    },

    # 处理器
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "level": "DEBUG",
            "formatter": "standard",
            "stream": "ext://sys.stdout",
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "INFO",
            "formatter": "json",
            "filename": "logs/app.log",
            "maxBytes": 10485760,  # 10MB
            "backupCount": 5,      # 保留 5 个备份文件
            "encoding": "utf-8",
        },
        "error_file": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "ERROR",
            "formatter": "json",
            "filename": "logs/error.log",
            "maxBytes": 10485760,
            "backupCount": 5,
            "encoding": "utf-8",
        },
    },

    # 日志记录器
    "loggers": {
        "app": {
            "handlers": ["console", "file"],
            "level": "DEBUG",
            "propagate": False,
        },
        "uvicorn": {
            "handlers": ["console"],
            "level": "INFO",
            "propagate": False,
        },
    },

    # 根 Logger
    "root": {
        "handlers": ["console", "file", "error_file"],
        "level": "WARNING",
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
```

---

## 2. structlog 结构化日志

### 2.1 为什么需要 structlog？

传统的 Python logging 输出是**纯文本字符串**：

```text
2026-05-05 10:30:00 [INFO] app.orders: Order created: ORD001, user=1, amount=680
```

这种格式难以被机器解析——你需要写正则表达式提取字段，不同开发者的格式不统一，日志分析工具难以处理。

**structlog 将日志输出为结构化数据（自动序列化为 JSON）：**

```python
logger.info("订单创建", order_no="ORD001", user_id=1, amount=680.00)
```text

输出：

```json
{"event": "订单创建", "order_no": "ORD001", "user_id": 1, "amount": 680.0, "timestamp": "2026-05-05T10:30:00", "logger": "app.orders", "level": "info"}
```

**优势：**

1. **机器可读**：JSON 格式可以被 ELK、Datadog、Splunk 等工具直接解析
2. **键值对而非格式化字符串**：不再需要 `"Order created: %s, user=%d" % (no, uid)` 这种脆弱格式
3. **上下文绑定**：可以在 Logger 上绑定全局上下文
4. **丰富的处理器链**：时间戳、异常格式化、上下文变量注入

### 2.2 get_logger / bind / unbind

**基本使用：**

```python
import structlog

# 获取 Logger
logger = structlog.get_logger()

# 普通日志
logger.info("用户登录成功", user_id=1, ip="192.168.1.1")
# → {"event": "用户登录成功", "user_id": 1, "ip": "192.168.1.1"}

logger.error("支付失败", order_no="ORD001", error="余额不足", amount=680.0)
# → {"event": "支付失败", "order_no": "ORD001", "error": "余额不足", "amount": 680.0}
```text

**bind / unbind（绑定上下文）：**

```python
# 绑定上下文后，后续所有日志自动携带这些字段
logger = structlog.get_logler().bind(
    request_id="a1b2c3d4",
    service="ticket-api",
    env="production",
)

# 每次都携带 request_id 和 service
logger.info("订单查询", order_no="ORD001")
# → {"event": "订单查询", "order_no": "ORD001", "request_id": "a1b2c3d4", "service": "ticket-api"}

logger.info("用户登录", user_id=100)
# → {"event": "用户登录", "user_id": 100, "request_id": "a1b2c3d4", "service": "ticket-api"}

# 临时添加字段
logger.info("库存扣减", event_id=1, quantity=2)
# → {"event": "库存扣减", "event_id": 1, "quantity": 2, "request_id": "a1b2c3d4", ...}

# 解绑特定上下文
logger = logger.unbind("request_id")

# 临时覆盖上下文（new 方法）
logger.info("特殊操作", request_id="temp_x")
# → 这次使用 temp_x，恢复后仍然用 a1b2c3d4
```

### 2.3 Processor 链

structlog 的核心概念是 **Processor Chain（处理器链）**。日志事件像流水线一样依次经过每个处理器：

```text
logger.info("创建订单", order_no="ORD001")
    │
    ▼
┌──────────────────────────────────┐
│ Processor 1: merge_contextvars     │ ← 注入 contextvars 中的上下文
│                                   │
│ Processor 2: filter_by_level       │ ← 按级别过滤
│                                   │
│ Processor 3: add_logger_name       │ ← 注入 logger 名称
│                                   │
│ Processor 4: add_log_level         │ ← 注入日志级别（info/error）
│                                   │
│ Processor 5: PositionalArguments   │ ← 格式化位置参数
│                                   │
│ Processor 6: TimeStamper           │ ← 注入 ISO 时间戳
│                                   │
│ Processor 7: StackInfoRenderer     │ ← 注入堆栈信息（按需）
│                                   │
│ Processor 8: format_exc_info       │ ← 格式化异常信息
│                                   │
│ Processor 9: UnicodeDecoder        │ ← Unicode 解码
│                                   │
│ Processor 10: ConsoleRenderer      │ ← 最终输出（开发环境：彩色控制台）
│               或是 JSONRenderer    │ ← 或 JSON 格式（生产环境）
└──────────────────────────────────┘
    │
    ▼
输出：{"event": "创建订单", "order_no": "ORD001", ...}
```

**各处理器详解：**

| 处理器 | 作用 |
| -------- | ------ |
| `merge_contextvars` | 从 Python 3.7+ 的 `contextvars` 中读取上下文变量并合并到日志事件中 |
| `filter_by_level` | 根据日志级别过滤（与 Python logging 的级别机制协同） |
| `add_logger_name` | 添加 `logger` 字段，值为 logger 的名称 |
| `add_log_level` | 添加 `level` 字段，值为 `info`、`error` 等 |
| `PositionalArgumentsFormatter` | 处理 `logger.info("msg %s", arg)` 这种位置参数格式 |
| `TimeStamper(fmt="iso")` | 添加 `timestamp` 字段，值为 ISO 8601 格式时间 |
| `StackInfoRenderer` | 如果事件包含 `stack_info`，将其格式化输出 |
| `format_exc_info` | 如果事件包含异常信息，格式化为易读的 traceback |
| `UnicodeDecoder` | 确保所有字符串是 Unicode 编码 |
| `ConsoleRenderer()` | 开发环境：带颜色的可读文本输出 |
| `JSONRenderer()` | 生产环境：JSON 格式输出 |

### 2.4 contextvars 与请求 ID 绑定

在一个异步 Web 应用中，**一个线程可以处理多个请求**（协程切换）。不能使用 `threading.local` 来存储请求级别的上下文。

**Python 3.7 引入的 `contextvars` 模块解决了这个问题**——它为每个协程维护独立的上下文：

```python
import contextvars
import uuid

# 定义一个请求 ID 上下文变量
request_id_var: contextvars.ContextVar[str] = contextvars.ContextVar(
    "request_id", default=""
)

# 在每个请求开始时设置
async def handle_request(request):
    request_id = uuid.uuid4().hex[:8]
    token = request_id_var.set(request_id)
    try:
        # 在这个协程及其子协程中，request_id_var.get() 返回正确的值
        await process_request(request)
    finally:
        request_id_var.reset(token)  # 清理

# 在项目的任何地方获取
def get_current_request_id() -> str:
    return request_id_var.get()
```text

**结合 structlog：**

```python
import structlog
import contextvars
import uuid

request_id_var = contextvars.ContextVar("request_id", default="")

# 在 structlog 配置中启用 merge_contextvars
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,  # ← 关键：自动合并 contextvars
        # ... 其他处理器
        structlog.processors.JSONRenderer(),
    ],
)

# 在中间件中设置 request_id
async def logging_middleware(request, call_next):
    request_id = uuid.uuid4().hex[:8]
    structlog.contextvars.bind_contextvars(request_id=request_id)
    # 或直接：request_id_var.set(request_id)
    # structlog.contextvars.bind_contextvars 内部会设置 contextvars

    response = await call_next(request)
    structlog.contextvars.clear_contextvars()
    return response

# 在业务代码中，自动携带 request_id
# process_order 内部调用 logger.info("扣减库存") 
# 输出自动包含 request_id 字段，无需手动传递
```

### 2.5 JSONRenderer vs ConsoleRenderer

**开发环境（ConsoleRenderer）：**

```python
structlog.dev.ConsoleRenderer()
```text

输出示例（带颜色和高亮）：

```

2026-05-05 10:30:00 [info     ] 订单创建成功           order_no=ORD001 amount=680.0 user_id=1
2026-05-05 10:30:01 [warning  ] 库存不足               event_id=1 stock=0 request_id=a1b2c3
2026-05-05 10:30:02 [error    ] 数据库连接失败          error=timeout db=localhost

```text

**生产环境（JSONRenderer）：**

```python
structlog.processors.JSONRenderer()
```

输出示例：

```json
{"event": "订单创建成功", "order_no": "ORD001", "amount": 680.0, "user_id": 1, "timestamp": "2026-05-05T10:30:00", "level": "info", "logger": "app.services.order_service"}
```text

**本项目中的条件渲染：**

```python
# app/utils/logging_config.py 第 22-24 行
structlog.dev.ConsoleRenderer() if __debug__
else structlog.processors.JSONRenderer()
```

`__debug__` 是 Python 内置常量：

- 运行 `python app.py`（无 -O 标志）时：`__debug__ = True` → ConsoleRenderer
- 运行 `python -O app.py`（优化模式）时：`__debug__ = False` → JSONRenderer
- 在生产容器中通常使用 JSONRenderer

---

## 3. 逐行解读：app/utils/logging_config.py

现在让我们逐行分析本项目中的实际日志配置代码。

```python
# 文件路径：app/utils/logging_config.py

import structlog           # 导入 structlog 库
import logging             # 导入 Python 标准 logging 库
import time                # 导入时间模块（虽然本文件中未直接使用 time）
import uuid                # 导入 UUID 模块（虽然本文件中未直接使用 uuid）
from contextvars import ContextVar  # 导入 ContextVar，用于请求级别的上下文

# ── 第 7 行 ──
request_id_var: ContextVar[str] = ContextVar("request_id", default="")
"""定义一个名为 request_id 的 ContextVar，默认值为空字符串。

ContextVar[str] 表示这是一个类型为 str 的上下文变量。
default="" 表示当没有显式设置时，get() 返回空字符串。
这个变量可以被同一个协程及其子协程共享读取。
在 structlog 中通过 merge_contextvars 处理器自动注入到日志中。
"""


# ── 第 9-30 行：setup_logging 函数 ──
def setup_logging():
    """配置结构化日志（structlog）

    这是整个应用日志系统的初始化入口，在应用启动时调用一次。
    配置一次后，所有模块通过 structlog.get_logger() 获取的 logger
    都会自动应用这些全局配置。
    """
    
    # ── 第 12-30 行：structlog.configure ──
    structlog.configure(
        # processors: 处理器链（按顺序依次处理每条日志）
        processors=[
            # Processor 1: 合并 contextvars
            # 将 contextvars 中绑定的变量（如 request_id）自动合并到日志事件中
            # 这样业务代码中不需要手动传递 request_id
            structlog.contextvars.merge_contextvars,

            # Processor 2: 按级别过滤
            # 配合 logging.basicConfig 中设置的 level 工作
            # 低于设定级别的日志被丢弃
            structlog.stdlib.filter_by_level,

            # Processor 3: 添加 logger 名称
            # 在日志事件中添加 "logger": "app.services.order_service" 字段
            structlog.stdlib.add_logger_name,

            # Processor 4: 添加日志级别
            # 在日志事件中添加 "level": "info" 或 "level": "error" 等
            structlog.stdlib.add_log_level,

            # Processor 5: 格式化位置参数
            # 处理 logger.info("Hello %s", name) 这种传统风格
            # 建议使用 logger.info("Hello", name=name) 键值对风格
            structlog.stdlib.PositionalArgumentsFormatter(),

            # Processor 6: 添加时间戳
            # fmt="iso" 表示 ISO 8601 格式：2026-05-05T10:30:00.123456
            structlog.processors.TimeStamper(fmt="iso"),

            # Processor 7: 堆栈信息渲染
            # 如果日志事件包含 stack_info，格式化为可读的堆栈
            structlog.processors.StackInfoRenderer(),

            # Processor 8: 格式化异常信息
            # 如果日志中包含异常（logger.exception(...)），
            # 将 exc_info 格式化为完整的 traceback
            structlog.processors.format_exc_info,

            # Processor 9: Unicode 解码
            # 确保所有值都是 Unicode 字符串，避免编码问题
            structlog.processors.UnicodeDecoder(),

            # Processor 10: 最终输出渲染器
            # __debug__ 为 True（开发模式）→ 带颜色的控制台输出
            # __debug__ 为 False（生产模式）→ JSON 格式输出
            structlog.dev.ConsoleRenderer() if __debug__
            else structlog.processors.JSONRenderer(),
        ],

        # wrapper_class: 包装类
        # BoundLogger 提供 .bind()/.unbind() 方法，
        # 可以在 logger 上绑定上下文变量
        wrapper_class=structlog.stdlib.BoundLogger,

        # context_class: 上下文存储类型
        # 使用 dict 存储键值对上下文
        context_class=dict,

        # logger_factory: Logger 工厂
        # 使用标准库的 logging.Logger 作为底层 Logger
        # 这样 structlog 与 Python logging 兼容
        logger_factory=structlog.stdlib.LoggerFactory(),

        # cache_logger_on_first_use: 首次使用后缓存 Logger
        # 提高日志性能，避免重复创建
        cache_logger_on_first_use=True,
    )

    # ── 第 31 行 ──
    # 配置 Python 标准 logging 的基础设置
    # 将日志格式设为 "%(message)s"，因为 structlog 已经处理了格式化
    # 级别设为 INFO，意味着 DEBUG 级别的日志被过滤掉
    logging.basicConfig(format="%(message)s", level=logging.INFO)


# ── 第 33-40 行：get_request_id 函数 ──
def get_request_id() -> str:
    """获取当前请求的 ID

    从 contextvars 中读取当前协程的 request_id。
    如果尚未设置（比如在请求处理之外调用），则生成一个新的 8 位随机 ID。
    
    返回:
        str: 请求 ID（8 位十六进制字符串）
    """
    # 读取当前协程上下文中的 request_id
    rid = request_id_var.get()
    
    if not rid:
        # 如果为空（请求上下文外调用），生成新的 ID
        rid = uuid.uuid4().hex[:8]
        # 设置到 contextvars 中
        request_id_var.set(rid)
    
    return rid
```text

**关键设计思想：**

1. **统一配置，一次调用**：`setup_logging()` 在应用启动时调用一次，所有模块共享同一配置
2. **开发/生产分离**：通过 `__debug__` 自动切换输出格式，开发者无需手动修改
3. **上下文自动注入**：`merge_contextvars` + `request_id_var` 实现请求 ID 自动传递
4. **与标准库兼容**：`LoggerFactory()` + `BoundLogger` 使 structlog 建立在标准 logging 之上

---

## 4. 逐行解读：app/middleware/logging.py

```python
# 文件路径：app/middleware/logging.py

import time                        # 用于计算请求处理耗时
import structlog                   # 结构化日志
from uuid import uuid4             # 生成唯一请求 ID
from starlette.middleware.base import BaseHTTPMiddleware  # FastAPI/Starlette 中间件基类
from starlette.requests import Request                    # Starlette 请求对象

# ── 第 7 行 ──
# 获取 structlog logger 实例
# 这个 logger 会自动应用 logging_config.py 中配置的所有处理器
logger = structlog.get_logger()


# ── 第 10-32 行：LoggingMiddleware 类 ──
class LoggingMiddleware(BaseHTTPMiddleware):
    """请求日志中间件：记录 method/path/status/耗时

    继承自 BaseHTTPMiddleware，是 FastAPI 推荐的中间件实现方式。
    对每个 HTTP 请求，记录开始和结束两条日志，包含请求 ID。
    
    日志输出示例（开发环境 ConsoleRenderer）：
    2026-05-05T10:30:00 [info     ] request_start     method=POST path=/api/v1/orders request_id=a1b2c3d4
    2026-05-05T10:30:01 [info     ] request_end       method=POST path=/api/v1/orders status=201 elapsed=0.523s request_id=a1b2c3d4
    """

    # ── 第 13 行：dispatch 方法 ──
    async def dispatch(self, request: Request, call_next):
        """中间件调度方法

        Args:
            request: FastAPI 请求对象，包含 method、url、headers 等
            call_next: 调用链中的下一个处理器（路由处理函数或下一个中间件）

        Returns:
            Response: 经过中间件处理后的响应对象
        """
        
        # ── 第 14 行：生成请求 ID ──
        request_id = uuid4().hex[:8]
        # 使用 uuid4().hex 生成 32 位十六进制字符串，取前 8 位作为请求 ID
        # 请求 ID 用于关联同一请求的所有日志（request_start → 业务日志 → request_end）
        # 8 位的唯一性对于开发环境足够，生产环境建议使用更长的 ID

        # ── 第 15 行：将请求 ID 注入到 request.state 中 ──
        request.state.request_id = request_id
        # request.state 是 Starlette 提供的请求级状态存储
        # 路由处理函数可以通过 request.state.request_id 访问
        # 例如在业务代码中：request.state.request_id

        # ── 第 16 行：记录开始时间 ──
        start = time.time()
        # 使用 time.time() 记录高精度时间戳（秒，浮点数）
        # 用于计算请求处理总耗时

        # ── 第 18 行：记录请求开始日志 ──
        logger.info(
            "request_start",                          # event 名称
            method=request.method,                     # GET/POST/PUT/DELETE
            path=request.url.path,                     # /api/v1/orders
            request_id=request_id,                     # 请求 ID
        )
        # 这条日志标记请求的开始。
        # 如果后续出现问题，可以通过 request_id 在日志中完整追踪该请求。

        # ── 第 20 行：调用下一个处理器 ──
        response = await call_next(request)
        # await call_next(request) 将请求传递给路由处理函数（或其他中间件）
        # 这是异步调用，在此等待期间，路由处理函数执行并返回响应

        # ── 第 22 行：计算耗时 ──
        elapsed = time.time() - start
        # 当前时间减去开始时间，得到请求处理总耗时（单位：秒）

        # ── 第 23-29 行：记录请求结束日志 ──
        logger.info(
            "request_end",                            # event 名称
            method=request.method,                     # 请求方法
            path=request.url.path,                     # 请求路径
            status=response.status_code,               # 响应状态码（200/400/500 等）
            elapsed=f"{elapsed:.3f}s",                 # 格式化耗时，保留 3 位小数
            request_id=request_id,                     # 请求 ID
        )
        # 这条日志标记请求的结束。
        # 通过比较 request_start 和 request_end 可以分析请求延迟

        # ── 第 31 行：在响应头中添加请求 ID ──
        response.headers["X-Request-ID"] = request_id
        # 将请求 ID 放入 HTTP 响应头，方便客户端或前端调试
        # 例如，如果客户端收到 500 错误，可以将 X-Request-ID 提供给后端排查
        # 这符合 HTTP 标准化实践（RFC 7230）

        # ── 第 32 行：返回响应 ──
        return response
```

**数据流图：**

```text
客户端请求 → LoggingMiddleware.dispatch()
                │
                ├─ 生成 request_id: a1b2c3d4
                ├─ logger.info("request_start", ...)
                │
                ▼
         call_next(request)
                │
                ├─ 路由匹配 → 路由处理函数
                │   ├─ 依赖注入（DB session、Redis）
                │   ├─ 业务逻辑（创建订单、扣减库存）
                │   └─ 返回响应
                │
                ▼
                ├─ logger.info("request_end", ...)
                ├─ response.headers["X-Request-ID"] = "a1b2c3d4"
                └─ return response

客户端收到响应 → 响应头包含 X-Request-ID: a1b2c3d4
```

**注册中间件：**

```python
# 在 app/main.py 中（或工厂函数中）
from app.middleware.logging import LoggingMiddleware

app = FastAPI(...)
app.add_middleware(LoggingMiddleware)
```text

---

## 5. Prometheus 监控

### 5.1 四大指标类型

Prometheus 有四种核心指标类型：

**Counter（计数器）：**

只增不减的计数器，用于累计值。

```python
from prometheus_client import Counter

# 定义：名称、说明、标签
request_total = Counter(
    "http_requests_total",           # 指标名称
    "Total HTTP requests",           # 指标说明
    ["method", "path", "status"],    # 标签维度
)

# 使用：每次请求 +1
request_total.labels(method="POST", path="/api/v1/orders", status="201").inc()

# 也可以增加任意值
request_total.labels(method="POST", path="/api/v1/orders", status="201").inc(5)

# Counter 的特点：
# - 只能增加（inc / inc_by）
# - 重启后会归零
# - 适用于：请求总数、错误总数、订单总数、用户注册总数
```

**Gauge（仪表盘）：**

可增可减的测量值，用于瞬时值。

```python
from prometheus_client import Gauge

# 定义
temperature = Gauge("cpu_temperature_celsius", "CPU temperature in Celsius")
memory_usage = Gauge("memory_usage_bytes", "Current memory usage", ["type"])

# 使用
temperature.set(45.2)          # 设置绝对值
temperature.inc(1.5)           # 增加
temperature.dec(2.0)           # 减少

memory_usage.labels(type="used").set(4_200_000_000)
memory_usage.labels(type="free").set(2_100_000_000)

# Gauge 的特点：
# - 可以增加、减少、设置为任意值
# - 适用于：当前内存使用、在线用户数、库存余量、队列长度
```text

**Histogram（直方图）：**

对观测值进行分桶计数，用于分析值的分布。

```python
from prometheus_client import Histogram

# 定义：指定桶边界
request_duration = Histogram(
    "http_request_duration_seconds",    # 指标名称
    "HTTP request duration in seconds", # 说明
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],  # 桶边界
)

# 使用：自动归入对应桶
request_duration.observe(0.324)   # 归入 0.25~0.5 桶

# 使用上下文管理器自动计时
with request_duration.time():
    process_request()

# Histogram 自动生成的指标：
# _bucket{le="0.01"}  → 耗时 ≤ 0.01s 的请求数
# _bucket{le="0.05"}  → 耗时 ≤ 0.05s 的请求数
# ... ...
# _bucket{le="+Inf"}  → 请求总数（所有桶的合计）
# _sum                → 所有请求耗时的总和
# _count              → 请求总次数（与 _bucket{le="+Inf"} 相同）

# 通过 _bucket 可以计算分位数（P50/P90/P99）：
# rate(http_request_duration_seconds_bucket{le="0.5"}[5m])
# 表示最近 5 分钟的请求中，耗时 ≤ 0.5s 的比例

# PromQL P99 查询：
# histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

**Summary（摘要）：**

类似 Histogram，但直接计算分位数（客户端侧）。

```python
from prometheus_client import Summary

# 定义
request_duration_summary = Summary(
    "http_request_duration_summary_seconds",
    "Summary of HTTP request duration",
)

# 使用
request_duration_summary.observe(0.324)

# Summary 自动生成的指标：
# _sum        → 总和
# _count      → 计数
# {quantile="0.5"}   → P50 分位
# {quantile="0.9"}   → P90 分位
# {quantile="0.99"}  → P99 分位

# Summary vs Histogram：
# Summary：客户端计算分位数，无法跨进程聚合
# Histogram：服务端计算分位数，可跨进程聚合（推荐）
```text

### 5.2 Histogram vs Summary 选择指南

| 特征 | Histogram | Summary |
| ------ | ----------- | --------- |
| 分位数计算位置 | 服务端（Prometheus Server） | 客户端（应用内） |
| 跨进程聚合 | 支持（PromQL 计算） | 不支持（只能算单进程） |
| 精确度控制 | 通过 `buckets` 参数 | 通过目标和误差 |
| 存储开销 | 更多（多个 bucket 指标） | 更少（直接分位数） |
| 推荐场景 | 生产环境、需要跨实例聚合 | 只需单实例观察 |

**结论：优先用 Histogram。**

### 5.3 prometheus-fastapi-instrumentator

`prometheus-fastapi-instrumentator` 是一个自动为 FastAPI 应用添加 Prometheus 指标的库：

```python
from prometheus_fastapi_instrumentator import Instrumentator

# 最简单的方式：一行代码自动采集
Instrumentator().instrument(app).expose(app)
```

**这行代码做了三件事：**

```text
1. .instrument(app)
   └── 注册中间件，自动采集每个请求的以下指标：
       ├── http_request_duration_seconds (Histogram) → 请求耗时
       ├── http_request_size_bytes (Summary)        → 请求体大小
       ├── http_response_size_bytes (Summary)       → 响应体大小
       ├── http_requests_total (Counter)            → 请求总数（按 method/handler/status 分）
       └── http_request_duration_highr_seconds (Histogram) → 高精度耗时

2. .expose(app)
   └── 添加 GET /metrics 端点，返回 Prometheus 格式的指标数据
```

**自定义 Instrumentator：**

```python
from prometheus_fastapi_instrumentator import Instrumentator, metrics

instrumentator = Instrumentator(
    should_group_status_codes=True,       # 是否将 2xx/3xx/4xx/5xx 分组
    should_ignore_untemplated=True,       # 是否忽略未模板化的路径
    should_instrument_requests_inprogress=True,  # 是否记录处理中的请求数
    excluded_handlers=["/health", "/metrics"],   # 排除健康检查和 metrics 端点
    inprogress_name="http_requests_in_progress",
    inprogress_labels=True,
)

# 注册默认指标
instrumentator.add(metrics.default())

# 注册自定义指标
@instrumentator.register()
def my_custom_metric(metric_name="my_custom_total"):
    """自定义指标示例"""
    # 返回一个指标注册函数
    def _registration(**kwargs):
        from prometheus_client import Counter
        return Counter(metric_name, "Custom metric description")

# 应用
instrumentator.instrument(app).expose(app)
```text

### 5.4 自定义业务指标 — 票务系统监控

为本项目的票务系统添加业务监控指标：

```python
# app/services/metrics.py — 票务系统自定义指标

from prometheus_client import Counter, Gauge, Histogram
import structlog

logger = structlog.get_logger()

# ── 1. ticket_bookings_total (Counter) ──
# 统计总购票数，按 event_id 和 status 标签细分
ticket_bookings_total = Counter(
    "ticket_bookings_total",
    "Total number of ticket bookings",
    ["event_id", "status"],     # status: success/failed/cancelled
)
# 使用示例：
# ticket_bookings_total.labels(event_id="1", status="success").inc()
# 查询：所有活动的总购票数
# PromQL: sum(ticket_bookings_total{status="success"})
# 查询：某活动的购票数
# PromQL: ticket_bookings_total{event_id="1"}


# ── 2. ticket_seats_available (Gauge) ──
# 当前可用座位数，按 event_id 细分
ticket_seats_available = Gauge(
    "ticket_seats_available",
    "Current number of available seats",
    ["event_id"],
)
# 使用示例：
# ticket_seats_available.labels(event_id="1").set(200)
# 查询：所有活动的总可用座位
# PromQL: sum(ticket_seats_available)


# ── 3. ticket_order_duration_seconds (Histogram) ──
# 订单处理耗时分布（从下单到支付成功）
ticket_order_duration_seconds = Histogram(
    "ticket_order_duration_seconds",
    "Order processing duration in seconds",
    ["event_id"],
    buckets=[1, 5, 10, 30, 60, 120, 300, 600, 1800],  # 从 1s 到 30min
)
# 使用示例：
# ticket_order_duration_seconds.labels(event_id="1").observe(duration)
# 查询 P50 订单处理时间：
# histogram_quantile(0.5, rate(ticket_order_duration_seconds_bucket[5m]))


# ── 4. ticket_revenue_total (Counter) ──
# 累计收入
ticket_revenue_total = Counter(
    "ticket_revenue_total",
    "Total revenue from ticket sales",
    ["event_id", "currency"],
)
# 使用示例：
# ticket_revenue_total.labels(event_id="1", currency="CNY").inc(680.00)
# 查询：总收入
# PromQL: sum(ticket_revenue_total)


# ── 5. ticket_inventory_operations (Counter) ──
# 库存操作统计
ticket_inventory_operations = Counter(
    "ticket_inventory_operations_total",
    "Total inventory operations",
    ["event_id", "operation"],   # operation: deduct/restore/init
)


# ── 辅助函数 ──

def observe_booking(event_id: int, status: str = "success"):
    """记录一次购票事件"""
    ticket_bookings_total.labels(event_id=str(event_id), status=status).inc()
    logger.debug("booking_observed", event_id=event_id, status=status)


def observe_order_duration(event_id: int, duration_seconds: float):
    """记录订单处理耗时"""
    ticket_order_duration_seconds.labels(event_id=str(event_id)).observe(duration_seconds)


def observe_revenue(event_id: int, amount: float, currency: str = "CNY"):
    """记录收入"""
    ticket_revenue_total.labels(event_id=str(event_id), currency=currency).inc(amount)
```

**在订单服务中使用这些指标：**

```python
# app/services/order_service.py 中集成监控

from app.services.metrics import (
    observe_booking,
    observe_order_duration,
    observe_revenue,
)
import time

async def create_order(self, user_id: int, data: OrderCreate) -> Order:
    order_start = time.time()
    
    try:
        # ... 业务逻辑（库存扣减、订单创建）...
        
        elapsed = time.time() - order_start
        
        # 记录成功
        observe_booking(event_id=data.event_id, status="success")
        observe_order_duration(event_id=data.event_id, duration_seconds=elapsed)
        observe_revenue(event_id=data.event_id, amount=total_amount)
        
        return order
        
    except ValueError as e:
        # 记录失败
        observe_booking(event_id=data.event_id, status="failed")
        raise
```text

### 5.5 访问 /metrics 端点

启动应用后，访问 `/metrics` 端点查看所有 Prometheus 指标：

```bash
curl http://localhost:8000/metrics
```

输出示例：

```text
# HELP python_info Python platform information
# TYPE python_info gauge
python_info{implementation="CPython",major="3",minor="12",patchlevel="0",version="3.12.0"} 1.0

# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{handler="POST /api/v1/orders",method="POST",status="201"} 42.0
http_requests_total{handler="GET /api/v1/events",method="GET",status="200"} 156.0
http_requests_total{handler="/health",method="GET",status="200"} 1000.0

# HELP http_request_duration_seconds HTTP request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{handler="POST /api/v1/orders",le="0.01"} 0
http_request_duration_seconds_bucket{handler="POST /api/v1/orders",le="0.05"} 5
http_request_duration_seconds_bucket{handler="POST /api/v1/orders",le="0.1"} 20
http_request_duration_seconds_bucket{handler="POST /api/v1/orders",le="0.25"} 35
http_request_duration_seconds_bucket{handler="POST /api/v1/orders",le="0.5"} 42
http_request_duration_seconds_bucket{handler="POST /api/v1/orders",le="+Inf"} 42
http_request_duration_seconds_sum{handler="POST /api/v1/orders"} 15.234
http_request_duration_seconds_count{handler="POST /api/v1/orders"} 42.0

# HELP ticket_bookings_total Total number of ticket bookings
# TYPE ticket_bookings_total counter
ticket_bookings_total{event_id="1",status="success"} 38.0
ticket_bookings_total{event_id="2",status="failed"} 3.0
ticket_bookings_total{event_id="1",status="cancelled"} 2.0

# HELP ticket_seats_available Current number of available seats
# TYPE ticket_seats_available gauge
ticket_seats_available{event_id="1"} 200.0
ticket_seats_available{event_id="2"} 150.0

# HELP ticket_order_duration_seconds Order processing duration in seconds
# TYPE ticket_order_duration_seconds histogram
ticket_order_duration_seconds_bucket{event_id="1",le="1.0"} 5
ticket_order_duration_seconds_bucket{event_id="1",le="5.0"} 28
ticket_order_duration_seconds_bucket{event_id="1",le="30.0"} 38
ticket_order_duration_seconds_bucket{event_id="1",le="+Inf"} 38

# HELP ticket_revenue_total Total revenue from ticket sales
# TYPE ticket_revenue_total counter
ticket_revenue_total{currency="CNY",event_id="1"} 25840.0
```

**Prometheus 配置拉取：**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'ticket-api'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'
```text

**Grafana 面板建议：**

| 面板 | 数据源 | 用途 |
| ------ | -------- | ------ |
| QPS | `rate(http_requests_total[1m])` | 每秒请求数 |
| 错误率 | `rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m])` | 错误占比 |
| P99 延迟 | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` | 99% 请求延迟 |
| 购票数 | `rate(ticket_bookings_total{status="success"}[5m])` | 购票速率 |
| 库存水位 | `ticket_seats_available` | 当前余票 |
| 收入 | `rate(ticket_revenue_total[1h])` | 每小时收入 |

---

## 6. 健康检查

### 6.1 liveness vs readiness

Kubernetes 定义了两类健康检查：

| 检查类型 | 端点 | 失败后的行为 | 用途 |
| ---------- | ------ | ------------- | ------ |
| **Liveness（存活）** | `/health` | 重启容器 | 检测死锁/崩溃，应用是否还活着 |
| **Readiness（就绪）** | `/health/ready` | 停止路由流量 | 检测依赖服务，应用能否处理请求 |

### 6.2 逐行解读：app/main.py 中的健康检查

```python
# 文件路径：app/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import settings
from app.routers import api_router


# ── 第 11-15 行：lifespan 上下文管理器 ──
@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期管理

    FastAPI 的 lifespan 机制替代了旧的 startup/shutdown 事件。
    在 yield 之前执行启动逻辑，在 yield 之后执行关闭逻辑。
    """
    # Startup: 初始化数据库连接、Redis 连接、加载 Lua 脚本等
    # ── 启动时执行 ──
    # 1. 数据库连接池初始化（在第一次获取数据库会话时自动初始化）
    # 2. Redis 连接池初始化（通过 get_redis_pool() 懒加载）
    # 3. 库存 Lua 脚本加载（见 inventory_service.py 的 load_script）
    # 4. 日志配置初始化（应该在更早的地方调用 setup_logging()）
    yield
    # ── 关闭时执行 ──
    # 1. 关闭数据库连接池
    # 2. 关闭 Redis 连接池


# ── 第 19-26 行：创建 FastAPI 实例 ──
app = FastAPI(
    title=settings.APP_NAME,              # "TicketBooking"
    description="本地部署+内网穿透的票务系统 API",  # 描述
    version="0.1.0",                       # 版本号
    lifespan=lifespan,                     # 生命周期管理
    docs_url="/docs",                      # Swagger UI 地址
    redoc_url="/redoc",                    # ReDoc 地址
)


# ── 第 29 行：注册路由 ──
app.include_router(api_router, prefix="/api/v1")
# api_router 来自 app/routers/__init__.py
# 包含了所有路由：auth、events、orders、payments、ws


# ── 第 32-38 行：CORS 中间件 ──
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],   # 开发阶段允许所有来源
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# ── 第 41-44 行：Liveness 健康检查 ──
@app.get("/health")
async def health_check():
    """健康检查接口（Liveness）

    Liveness 检查：只检查应用进程是否存活。
    不需要检查数据库/Redis 等外部依赖（那是 readiness 的责任）。
    
    如果这个端点返回非 200 状态码，Kubernetes 会重启容器。
    在实际生产环境中可能包含：
    - Python 解释器是否运行正常
    - asyncio 事件循环是否正常
    - 是否有死锁（goroutine leak 等）
    
    返回：
        dict: {"status": "ok", "service": APP_NAME, "env": ENV}
        HTTP 状态码：200 OK
    """
    return {
        "status": "ok",
        "service": settings.APP_NAME,     # "TicketBooking"
        "env": settings.ENV,               # "development"（从 .env 读取）
    }


# ── Readiness 健康检查（需要自行添加） ──
# 本项目当前的 /health 只实现了 liveness。
# 下面是一个完整的 readiness 实现示例：

@app.get("/health/ready")
async def readiness_check():
    """就绪检查接口（Readiness）

    Readiness 检查：验证应用的所有外部依赖是否可用。
    如果这个端点返回非 200，负载均衡器会停止向该实例发送流量，
    但不会重启容器（与 liveness 不同）。
    
    检查的项目：
    1. 数据库连接是否正常（ping）
    2. Redis 连接是否正常（ping）
    3. 其他依赖服务是否可用
    
    返回：
        dict: {"status": "ok"|"degraded"|"unavailable",
               "checks": {每个依赖的检查结果},
               "version": 版本号}
    """
    from app.database import get_db
    from app.redis_client import get_redis
    
    checks = {}
    all_healthy = True
    
    # 检查 MySQL 连接
    try:
        async with async_session_factory() as session:
            await session.execute(text("SELECT 1"))
        checks["database"] = {"status": "ok", "latency_ms": 0}
    except Exception as e:
        checks["database"] = {"status": "error", "message": str(e)}
        all_healthy = False
    
    # 检查 Redis 连接
    try:
        redis = await anext(get_redis())  # FastAPI 依赖方式
        await redis.ping()
        checks["redis"] = {"status": "ok", "latency_ms": 0}
    except Exception as e:
        checks["redis"] = {"status": "error", "message": str(e)}
        all_healthy = False
    
    # 汇总状态
    if all_healthy:
        status = "ok"
        http_status = 200
    else:
        status = "degraded"
        http_status = 503  # Service Unavailable
    
    return JSONResponse(
        status_code=http_status,
        content={
            "status": status,
            "service": settings.APP_NAME,
            "version": "0.1.0",
            "checks": checks,
            "uptime_seconds": int(time.time() - start_time),
        },
    )
```

**Docker 配置健康检查：**

```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```text

**Kubernetes 配置健康检查：**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ticket-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ticket-api
  template:
    metadata:
      labels:
        app: ticket-api
    spec:
      containers:
      - name: api
        image: ticket-api:latest
        ports:
        - containerPort: 8000
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 15
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 2
```

---

## 7. 统一配置与最佳实践

### 7.1 完整的日志初始化流程

```python
# app/main.py — 完整版

import structlog
import logging.config
from contextlib import asynccontextmanager
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator

from app.config import settings
from app.utils.logging_config import setup_logging
from app.middleware.logging import LoggingMiddleware
from app.routers import api_router


# 1. 在应用启动前初始化日志（必须在创建 app 之前）
setup_logging()
logger = structlog.get_logger()


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期"""
    logger.info("app_startup", env=settings.ENV, debug=settings.DEBUG)
    yield
    logger.info("app_shutdown")


app = FastAPI(
    title=settings.APP_NAME,
    description="票务系统 API",
    version="0.1.0",
    lifespan=lifespan,
    docs_url="/docs",
)

# 2. 注册中间件（顺序重要：先注册的外层后执行）
app.add_middleware(LoggingMiddleware)           # 日志中间件（最外层）
app.add_middleware(
    CORSMiddleware,                             # CORS 中间件
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 3. 注册路由
app.include_router(api_router, prefix="/api/v1")

# 4. 初始化 Prometheus 监控
Instrumentator(
    should_group_status_codes=True,
    should_ignore_untemplated=True,
    excluded_handlers=["/health", "/health/ready", "/metrics"],
).instrument(app).expose(app)

# 5. 健康检查端点
@app.get("/health")
async def health_check():
    return {"status": "ok", "service": settings.APP_NAME, "env": settings.ENV}

@app.get("/health/ready")
async def readiness_check():
    # ... 检查 MySQL + Redis ...
    pass
```text

### 7.2 日志级别动态调整

生产环境中有时需要临时启用 DEBUG 日志排查问题：

```python
# app/routers/admin.py — 动态日志级别调整端点
from fastapi import APIRouter, Depends, HTTPException
import logging

router = APIRouter()

@router.post("/admin/log-level")
async def set_log_level(
    logger_name: str,
    level: str,
    current_user: User = Depends(get_current_admin),
):
    """动态调整日志级别（管理用）"""
    valid_levels = {"DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"}
    if level.upper() not in valid_levels:
        raise HTTPException(status_code=400, detail=f"无效级别，可选: {valid_levels}")
    
    logger = logging.getLogger(logger_name)
    logger.setLevel(getattr(logging, level.upper()))
    
    return {
        "logger": logger_name,
        "previous_level": logging.getLevelName(logger.level),
        "new_level": level.upper(),
    }

# curl -X POST http://localhost:8000/api/v1/admin/log-level \
#   -H "Content-Type: application/json" \
#   -d '{"logger_name": "app.services.order_service", "level": "DEBUG"}'
```

### 7.3 安全注意事项

```text
1. 不要在日志中记录敏感信息
   × logger.info("用户密码", password=user.password)
   √ logger.info("用户登录成功", user_id=user.id)

2. 在生产环境使用 JSON 格式
   - 便于日志收集系统（ELK、Loki、Datadog）解析

3. 健康检查端点不要泄露敏感信息
   - /health 只需要返回简单的 status: ok
   - /health/ready 的内部检查错误不要暴露堆栈

4. Prometheus /metrics 端点应限制访问
   - 生产环境用 Nginx 限制 IP
   - 或使用 Basic Auth 保护

5. 设置合理的日志轮转策略
   - RotatingFileHandler 或 TimedRotatingFileHandler
   - 避免日志文件无限增长耗尽磁盘
```

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/middleware/logging.py` | 请求日志中间件：记录方法/路径/状态码/耗时/客户端 IP |
| `app/utils/logging_config.py` | 日志配置：JSON 格式、脱敏处理器、structlog 配置 |
| `app/utils/profiling.py` | Prometheus 指标：请求计数、耗时直方图、QPS 监控 |

**日志输出示例：**

```json
{"time": "2026-05-11T10:30:00Z", "level": "INFO",
 "method": "POST", "path": "/api/v1/orders",
 "status": 201, "duration_ms": 45,
 "client_ip": "192.168.1.100", "user_id": 42}
```text

## 本章总结

**核心知识点回顾：**

1. **Python Logging 架构**：Logger 形成层级结构，日志通过 propagate 机制从子 Logger 传播到父 Logger，避免重复日志的关键是只在需要的地方添加 Handler 并控制 propagate

2. **structlog**：通过 Processor Chain 将普通日志转换为结构化 JSON 输出，支持上下文绑定（bind/unbind）和 contextvars 自动注入请求上下文

3. **本项目 logging_config.py**：配置了 9 个处理器链，从 contextvars 合并到最终 JSON/Console 输出，通过 `__debug__` 自动切换开发/生产模式

4. **本项目 middleware/logging.py**：生成请求 ID、记录 request_start/request_end 日志、计算耗时、在响应头注入 X-Request-ID

5. **Prometheus 指标**：
   - Counter：只增不减（购票总数、收入）
   - Gauge：可增可减（可用座位数、在线用户）
   - Histogram：分桶计数（请求耗时、订单处理耗时）
   - Summary：客户端计算分位数（不推荐跨进程聚合）

6. **健康检查**：liveness（进程是否存活）和 readiness（依赖是否就绪），检查数据库和 Redis 连接

---

**项目检查清单：**

- [ ] `setup_logging()` 在应用启动时调用
- [ ] LoggingMiddleware 已注册
- [ ] 所有业务日志使用结构化格式（key=value）
- [ ] Prometheus Instrumentator 已配置
- [ ] 自定义业务指标已添加（购票数、库存、收入等）
- [ ] `/metrics` 端点可访问并返回指标数据
- [ ] `/health` 端点返回 200
- [ ] `/health/ready` 端点包含 MySQL + Redis 检查
- [ ] 日志在生产环境中输出 JSON 格式
- [ ] 日志文件配置了轮转策略
- [ ] /metrics 端点做了访问限制（Nginx + IP 白名单）

---

## 日志安全

### 敏感字段脱敏

日志中绝不能出现明文密码、令牌、密钥等敏感信息。本系统在 `logging_config.py` 中实现了自动脱敏处理器：

```python
SENSITIVE_FIELDS = re.compile(
    r"password|secret|token|key|authorization|credential|credit_card|cvv",
    re.IGNORECASE
)

def _mask_sensitive_events(logger, log_method, event_dict):
    for key in list(event_dict.keys()):
        if SENSITIVE_FIELDS.search(key):
            event_dict[key] = "[REDACTED]"
    return event_dict
```

### 生产环境日志格式

开发环境使用彩色可读的 ConsoleRenderer，生产环境必须使用 JSONRenderer：

```python
if settings.IS_PRODUCTION:
    processors.append(structlog.processors.JSONRenderer())
else:
    processors.append(structlog.dev.ConsoleRenderer())
```text

JSON 格式可被 ELK、Datadog、Splunk 等日志聚合工具解析。

### 审计日志事件

本系统记录以下安全关键事件（可用于入侵检测）：

| 事件 | 级别 | 说明 |
| ------ | ------ | ------ |
| LOGIN_FAILED | WARNING | 登录失败，含失败次数 |
| ACCOUNT_LOCKED | WARNING | 账户因多次失败被锁定 |
| TOKEN_REFRESH | INFO | 令牌轮换日志 |
| PAYMENT_VERIFY_FAILED | WARNING | 支付宝验签失败 |
| ORDER_CANCELLED | INFO | 订单取消操作 |
| WEB_SOCKET_CONNECT | INFO | WebSocket 连接记录 |

- [ ] 敏感信息不记录到日志
