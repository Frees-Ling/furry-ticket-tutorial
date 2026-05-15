# 第1章：Python/Docker/Git 基础与项目脚手架

## 本章学习目标

本章将掌握构建票务系统所需的全部基础工具，从零开始学习每个工具的完整语法和用法，最终搭建出项目的骨架结构。

**学习路径：**

1. ✅ Python 从零到高阶（全部语法/类型/控制流/OOP/异步/类型注解）
2. ✅ pip 和虚拟环境（包管理完整指南）
3. ✅ Docker 和 Docker Compose（容器化完全教程）
4. ✅ PowerShell 和 Git 基础
5. ✅ 搭建票务系统项目脚手架

---

## 1. Python 完全教程（从零到高阶）

Python 是整个票务系统的实现语言。我们将使用 Python 3.12+，充分利用 async/await 特性构建高性能 Web 服务。

### 1.1 Python 的运行方式

Python 代码可以通过三种方式执行：

**方式一：交互式 REPL**

```bash
python
>>> print("你好，票务系统")
你好，票务系统
>>> 1 + 2
3
```text

**方式二：脚本执行**

```bash
python app/main.py
```

**方式三：模块执行**（推荐，能正确处理相对导入）

```bash
python -m app.main
```text

### 1.2 词法结构

**缩进规则：** Python 使用缩进表示代码块，必须统一使用 4 个空格（禁止混用 Tab）。

```python
def say_hello(name):    # 冒号开始一个代码块
    print(f"你好, {name}")  # 4 个空格缩进
    if name == "admin":     # 内层代码块
        print("欢迎管理员")  # 8 个空格缩进
```

**注释：**

```python
# 这是单行注释

"""
这是多行字符串（通常用作文档字符串）
docstring 会作为模块/函数/类的 __doc__ 属性
"""
```text

**编码声明：** Python 3 默认 UTF-8 编码，无需额外声明。

### 1.3 全部内置类型详解

#### 数值类型（Numeric Types）

**int（整数）：** Python 的 int 是任意精度的，不会溢出。

```python
# 整数字面量
a = 42          # 十进制
b = 0b101010    # 二进制（以 0b 开头）
c = 0o52        # 八进制（以 0o 开头）
d = 0x2A        # 十六进制（以 0x 开头）
e = 1_000_000   # 下划线做千位分隔符（PEP 515）

# int 支持无限精度
huge = 10 ** 100  # 10 的 100 次方，googol
print(huge)       # 10000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

**float（浮点数）：** 基于 IEEE 754 双精度（64位）。

```python
f1 = 3.14          # 标准写法
f2 = 1.5e-3        # 科学记数法 = 0.0015
f3 = float('inf')  # 正无穷大
f4 = float('-inf') # 负无穷大
f5 = float('nan')  # Not a Number

# 浮点数精度问题（务必注意）
print(0.1 + 0.2)        # 0.30000000000000004 ← 不是 0.3！
print(0.1 + 0.2 == 0.3) # False ← 比较会失败！

# 解决方案：使用 Decimal 进行精确计算
from decimal import Decimal, ROUND_HALF_UP
price = Decimal('19.99')  # 注意是字符串，不是浮点数！
quantity = Decimal('3')
total = price * quantity  # Decimal('59.97')
print(total.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP))  # 四舍五入到分
```text

> **为什么 0.1 + 0.2 ≠ 0.3？** IEEE 754 双精度浮点数用 64 位表示一个数：1 位符号 + 11 位指数 + 52 位尾数。0.1 的二进制表示是无限循环小数 0.0001100110011...，只能近似存储。当两个近似值相加时，误差累积导致结果不等于期望值。票务系统涉及金额计算，务必使用 `Decimal` 而非 `float`。

**complex（复数）：**

```python
c1 = 1 + 2j       # j 表示虚部
c2 = complex(3, 4)
print(c1.real)    # 1.0（实部）
print(c1.imag)    # 2.0（虚部）
print(c1.conjugate())  # (1-2j)
```

#### 序列类型（Sequence Types）

**str（字符串）：** Unicode 字符串，不可变。

```python
# 字符串创建方式
s1 = '单引号字符串'
s2 = "双引号字符串（推荐）"
s3 = '''多行字符串
可以跨越多行'''
s4 = """三个双引号也可以"""
s5 = r"原始字符串\n不会转义"   # \n 就是两个字符，不是换行
s6 = b"bytes 类型"             # bytes 对象

# f-string（Python 3.6+，推荐！）
name = "小明"
age = 25
s7 = f"我叫{name}，今年{age}岁"     # "我叫小明，今年25岁"
s8 = f"年龄翻倍是 {age * 2}"         # 表达式也能计算
s9 = f"十六进制: {age:x}"            # 支持格式说明符
s10 = f"对齐: {name:>10}"            # 右对齐10字符
s11 = f"DEBUG: {name=} {age=}"       # Python 3.8+，自动变量名：name='小明' age=25

# 字符串方法
text = "  Hello, 世界!  "
print(text.strip())      # 去除两端空格 → "Hello, 世界!"
print(text.lower())      # 转小写 → "  hello, 世界!  "
print(text.replace("世界", "World"))  # 替换
print(text.split(","))   # 分割 → ['  Hello', ' 世界!  ']
print(",".join(["a", "b", "c"]))  # 连接 → "a,b,c"

# 字符串索引和切片
s = "Python"
print(s[0])     # P（正索引从 0 开始）
print(s[-1])    # n（负索引从 -1 开始）
print(s[0:3])   # Pyt（切片：起始:结束:步长）
print(s[::-1])  # nohtyP（反转字符串）
```text

**list（列表）：** 可变、有序、可存放任意类型。

```python
# 列表创建
lst1 = [1, 2, 3]                    # 字面量
lst2 = list("abc")                   # ['a', 'b', 'c']
lst3 = [x * 2 for x in range(5)]    # 列表推导式 [0, 2, 4, 6, 8]
lst4 = [[0] * 3 for _ in range(3)]  # 二维列表 [[0,0,0],[0,0,0],[0,0,0]]

# ⚠️ 列表复制陷阱！
wrong = [[0] * 3] * 3   # ❌ 三个引用同一个内部列表！
wrong[0][0] = 1
print(wrong)  # [[1, 0, 0], [1, 0, 0], [1, 0, 0]] ← 全变了！

right = [[0] * 3 for _ in range(3)]  # ✅ 每次重新创建
right[0][0] = 1
print(right)  # [[1, 0, 0], [0, 0, 0], [0, 0, 0]]

# 列表常用操作
lst = [3, 1, 4, 1, 5]
lst.append(9)        # [3, 1, 4, 1, 5, 9]
lst.insert(0, 0)     # [0, 3, 1, 4, 1, 5, 9]
lst.pop()            # 删除并返回最后元素 → 9
lst.sort()           # 原地排序 [0, 1, 1, 3, 4, 5]
lst.reverse()        # 原地反转
```

**tuple（元组）：** 不可变序列。

```python
t1 = (1, 2, 3)         # 字面量
t2 = 1, 2, 3           # 省略括号也可以
t3 = (1,)              # 单元素元组（逗号不能省略！）
wrong = (1)            # ❌ 这是整数 1，不是元组！
```text

#### 映射类型

**dict（字典）：** 键值对存储，Python 3.7+ 保持插入顺序。

```python
# 字典创建
d1 = {"name": "小明", "age": 25}
d2 = dict(name="小明", age=25)         # 关键字参数
d3 = dict(zip(["name", "age"], ["小明", 25]))  # 从键值对列表
d4 = {x: x**2 for x in range(5)}      # 字典推导式 {0:0, 1:1, 2:4, 3:9, 4:16}

# 访问和操作
user = {"name": "小明", "role": "user"}
print(user["name"])        # 小明（键不存在会 KeyError）
print(user.get("email", "未设置"))  # "未设置"（安全访问）
user["role"] = "admin"     # 修改或添加
user.update({"email": "x@test.com", "age": 25})  # 批量更新
del user["age"]            # 删除键

# 遍历
for key in user:           # 遍历键（等价于 user.keys()）
    print(key, user[key])
for value in user.values():    # 遍历值
    print(value)
for key, value in user.items():  # 遍历键值对
    print(key, value)
```

#### 集合类型

```python
s1 = {1, 2, 3}              # 集合字面量
s2 = set([1, 2, 2, 3])      # {1, 2, 3}（重复被去重）
empty_set = set()           # 空集合（{} 是空字典，不是空集合！）

# 集合运算
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
print(a | b)   # 并集 {1, 2, 3, 4, 5, 6}
print(a & b)   # 交集 {3, 4}
print(a - b)   # 差集 {1, 2}
print(a ^ b)   # 对称差 {1, 2, 5, 6}
```text

#### 布尔类型

```python
t = True
f = False

# 布尔值是整数的子类！
print(True + True)    # 2（True=1, False=0）
print(True * 5)       # 5

# 假值判断：以下值被视为 False
# None, False, 0, 0.0, ''（空字符串）, []（空列表）, {}（空字典）, set()（空集合）
if not []:  # [] 是假值
    print("空列表是假值")
```

### 1.4 运算符优先级表（从高到低）

| 优先级 | 运算符 | 说明 |
| -------- | -------- | ------ |
| 1 | `(...)` | 圆括号（最高优先级） |
| 2 | `**` | 幂运算（右结合） |
| 3 | `+x`, `-x`, `~x` | 一元正号/负号/按位取反 |
| 4 | `*`, `/`, `//`, `%` | 乘/除/整除/取模 |
| 5 | `+`, `-` | 加/减 |
| 6 | `<<`, `>>` | 左移/右移 |
| 7 | `&` | 按位与 |
| 8 | `^` | 按位异或 |
| 9 | `\|` | 按位或 |
| 10 | `==`, `!=`, `<`, `>`, `<=`, `>=`, `is`, `in` | 比较/身份/成员 |
| 11 | `not x` | 逻辑非 |
| 12 | `and` | 逻辑与（短路运算） |
| 13 | `or` | 逻辑或（短路运算） |
| 14 | `:=` | 赋值表达式（海象运算符） |

```python
# 海象运算符示例（Python 3.8+）
if (n := len("hello")) > 3:  # 赋值并用于条件判断
    print(f"长度为 {n}")

# 短路运算示例
user_input = "" or "默认值"  # → "默认值"（或运算返回第一个真值）
```text

### 1.5 控制流

#### if/elif/else

```python
score = 85
if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
else:
    grade = "不及格"
```

#### match/case（Python 3.10+）

```python
def handle_order_status(status: str) -> str:
    match status:
        case "pending":
            return "待支付"
        case "confirmed":
            return "已确认"
        case "cancelled":
            return "已取消"
        case _:  # 默认匹配
            return "未知状态"

# 更复杂的模式匹配
def process_payment(payment):
    match payment:
        case {"method": "alipay", "amount": amount} if amount > 0:
            print(f"支付宝支付 {amount} 元")
        case {"method": "wechat", "amount": amount}:
            print(f"微信支付 {amount} 元")
        case _:
            print("未知支付方式")
```text

#### 循环

```python
# for 循环
for i in range(5):            # 0,1,2,3,4
    print(i)

for i, item in enumerate(["a", "b", "c"]):  # 带索引
    print(f"{i}: {item}")

for a, b in zip(["x", "y"], [1, 2]):        # 并行迭代
    print(a, b)

# while 循环
count = 0
while count < 5:
    print(count)
    count += 1

# break/continue/else
for n in range(10):
    if n == 3:
        continue  # 跳过本次
    if n == 7:
        break     # 终止循环
    print(n)
else:  # 循环正常结束（没有 break）时执行
    print("循环正常结束")
```

### 1.6 函数

#### 函数定义和参数类型

```python
# 基本函数
def greet(name: str) -> str:  # 类型注解（不会强制检查，但工具会用）
    return f"你好, {name}"

# 全部参数类型
def complex_func(
    pos_only: int,              # 位置参数
    /,                          # 分隔符：之前的参数只能位置传入
    pos_or_kw: str,            # 位置或关键字
    *,                          # 分隔符：之后的参数只能关键字传入
    kw_only: float,             # 仅关键字参数
    default: int = 42,          # 默认参数（⚠️ 不要用可变默认值！）
    *args: int,                 # 可变位置参数
    **kwargs: str               # 可变关键字参数
) -> None:
    pass

# ⚠️ 可变默认值的陷阱（重要！）
def add_item(item, items=[]):  # ❌ 默认列表只创建一次！
    items.append(item)
    return items

print(add_item(1))  # [1]
print(add_item(2))  # [1, 2] ← 不是 [2]！

def add_item_correct(item, items=None):  # ✅ 用 None 作为默认值
    if items is None:
        items = []
    items.append(item)
    return items
```text

#### 装饰器

```python
import functools
import time

# 最简单的装饰器
def log_calls(func):
    @functools.wraps(func)  # 保留原函数的元信息
    def wrapper(*args, **kwargs):
        print(f"调用 {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_calls
def say_hello(name):
    return f"Hello {name}"

# 带参数的装饰器
def retry(max_attempts: int = 3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for i in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if i == max_attempts - 1:
                        raise
                    print(f"重试 {i+1}/{max_attempts}: {e}")
            return None
        return wrapper
    return decorator

@retry(max_attempts=3)
def fetch_data(url):
    print(f"请求 {url}...")
    raise ConnectionError("网络错误")
```

#### 生成器

```python
def countdown(n: int):
    """生成器函数：倒计时"""
    while n > 0:
        yield n  # 暂停函数，返回 n
        n -= 1

for num in countdown(3):
    print(num)  # 3, 2, 1

# 生成器表达式
even_squares = (x * x for x in range(10) if x % 2 == 0)
print(list(even_squares))  # [0, 4, 16, 36, 64]
```text

### 1.7 类（面向对象编程）

```python
class User:
    """用户模型类"""
    
    # 类变量（所有实例共享）
    total_users = 0
    
    def __init__(self, username: str, email: str):
        """构造方法：创建实例时自动调用"""
        self.username = username  # 实例变量
        self.email = email
        self._role = "user"       # 单下划线：内部使用（惯例）
        User.total_users += 1
    
    # 实例方法
    def greet(self) -> str:
        return f"你好, {self.username}"
    
    # 属性访问器
    @property
    def role(self) -> str:
        return self._role
    
    @role.setter
    def role(self, value: str):
        if value not in ("user", "admin", "manager"):
            raise ValueError("无效角色")
        self._role = value
    
    # 类方法
    @classmethod
    def create_admin(cls, username: str) -> "User":
        """创建管理员用户"""
        user = cls(username, f"{username}@admin.com")
        user._role = "admin"
        return user
    
    # 静态方法（不需要实例或类）
    @staticmethod
    def validate_email(email: str) -> bool:
        return "@" in email and "." in email
    
    # 特殊方法（"双下方法"）
    def __str__(self) -> str:
        return f"User({self.username})"
    
    def __repr__(self) -> str:
        return f"User(username='{self.username}', email='{self.email}')"

# 使用
user = User("小明", "x@test.com")
print(user.greet())          # 你好, 小明
user.role = "admin"          # 使用 setter
print(user)                  # User(小明)
print(User.total_users)      # 1
```

#### 继承和 MRO（C3 线性化算法）

```python
class AdminUser(User):
    def __init__(self, username: str, email: str, permissions: list[str]):
        super().__init__(username, email)  # 调用父类构造方法
        self.permissions = permissions
        self._role = "admin"

class Ticket:
    def __repr__(self):
        return f"<Ticket {self.id}>"

# MRO（方法解析顺序）由 C3 线性化算法决定
# 证明：C3 线性化满足两个特性：
# 1. 单调性：如果 A 在 B 的 MRO 中排在前面，则在所有子类中 A 都在 B 前面
# 2. 局部优先顺序：子类的 MRO 中父类的顺序保持
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
print(D.__mro__)  # D → B → C → A → object
```text

#### 数据类（dataclass，Python 3.7+）

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Order:
    order_id: str
    user_id: int
    amount: float
    status: str = "pending"           # 默认值
    items: list = field(default_factory=list)  # ⚠️ 必须用 field factory，不能用 []
    note: Optional[str] = None

# 自动得到 __init__, __repr__, __eq__, __hash__ 等
order = Order("ORD-001", 1, 99.9)
print(order)  # Order(order_id='ORD-001', user_id=1, amount=99.9, status='pending', items=[], note=None)
```

### 1.8 asyncio 异步编程

票务系统使用 FastAPI + SQLAlchemy async，理解 async/await 至关重要。

```python
import asyncio
import time

# 协程函数
async def fetch_data(url: str) -> str:
    """模拟网络请求（非阻塞等待）"""
    print(f"开始请求 {url}")
    await asyncio.sleep(1)  # 模拟 IO 操作（不阻塞事件循环！）
    print(f"完成请求 {url}")
    return f"数据来自 {url}"

# 事件循环入口
async def main():
    # 顺序执行（慢）
    start = time.time()
    r1 = await fetch_data("url1")
    r2 = await fetch_data("url2")
    print(f"顺序执行耗时: {time.time() - start:.2f}s")
    
    # 并发执行（快！）
    start = time.time()
    task1 = asyncio.create_task(fetch_data("url1"))  # 创建任务（立即开始）
    task2 = asyncio.create_task(fetch_data("url2"))
    results = await asyncio.gather(task1, task2)     # 等待所有完成
    print(f"并发执行耗时: {time.time() - start:.2f}s")
    
    return results

# 运行入口
if __name__ == "__main__":
    results = asyncio.run(main())
```text

**asyncio 核心概念：**

- **协程（coroutine）**：`async def` 定义的函数返回协程对象
- **可等待对象（awaitable）**：协程、Task、Future
- **事件循环（event loop）**：管理所有异步任务的调度器
- **Task**：`asyncio.create_task()` 将协程包装为 Task，立即开始调度
- **await**：暂停当前协程，等待另一个可等待对象完成

**常用 asyncio API：**

```python
# 控制并发度
sem = asyncio.Semaphore(10)  # 最多10个并发

async def limited_request(url: str):
    async with sem:
        return await fetch_data(url)

# 超时控制
try:
    result = await asyncio.wait_for(fetch_data("url"), timeout=5.0)
except asyncio.TimeoutError:
    print("请求超时")

# 生产者-消费者模式
async def producer(queue: asyncio.Queue):
    for i in range(10):
        await queue.put(f"item-{i}")
    await queue.put(None)  # 发送结束信号

async def consumer(queue: asyncio.Queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"处理: {item}")
```

### 1.9 异常处理

```python
# 基本异常处理
def safe_divide(a: float, b: float) -> float:
    try:
        result = a / b
    except ZeroDivisionError:
        print("除数不能为0！")
        return float('inf')
    except TypeError as e:
        print(f"类型错误: {e}")
        raise  # 重新抛出当前异常
    else:  # 没有异常时执行
        print("计算成功")
        return result
    finally:  # 总是执行（用于清理资源）
        print("除法操作完成")

# 自定义异常
class BookingError(Exception):
    """订单相关异常基类"""
    def __init__(self, message: str, code: str = "UNKNOWN"):
        super().__init__(message)
        self.code = code

class InsufficientInventory(BookingError):
    def __init__(self, available: int, requested: int):
        super().__init__(
            f"库存不足：剩余 {available}，需要 {requested}",
            code="INSUFFICIENT_INVENTORY"
        )
        self.available = available
        self.requested = requested

# 使用自定义异常
def book_ticket(available: int, quantity: int):
    if quantity > available:
        raise InsufficientInventory(available, quantity)
    print(f"成功预订 {quantity} 张票")
```text

### 1.10 上下文管理器

```python
# with 语句（自动管理资源）
with open("file.txt", "r") as f:
    content = f.read()
# 文件在此自动关闭

# contextlib 工具
from contextlib import contextmanager, suppress

@contextmanager
def timed_block(label: str):
    """计时上下文管理器"""
    start = time.time()
    try:
        yield  # 进入 with 块时执行到这里
    finally:
        elapsed = time.time() - start
        print(f"{label} 耗时: {elapsed:.3f}s")

with timed_block("数据库查询"):
    time.sleep(0.5)  # 模拟数据库查询

# suppress：忽略指定异常
with suppress(FileNotFoundError):
    os.remove("不存在的文件.txt")  # 不会抛出异常
```

### 1.11 typing 模块（类型注解）

```python
from typing import (
    Optional, Union, Literal, Generic, TypeVar, 
    Protocol, Annotated, Any
)
from collections.abc import Sequence, Mapping, Callable

# 基础类型
user_id: int = 1
price: float = 99.9
name: str = "小明"

# 容器类型
users: list[str] = ["小明", "小红"]
scores: dict[str, int] = {"小明": 95}
maybe: Optional[int] = None  # 等价于 Union[int, None]

# 字面量类型（精确值约束）
Status = Literal["pending", "confirmed", "cancelled"]
def update_status(status: Status) -> None: ...

# 泛型
T = TypeVar("T")
def first(items: list[T]) -> T | None:
    return items[0] if items else None

# Protocol（结构子类型）
class Hashable(Protocol):
    def __hash__(self) -> int: ...

def store(obj: Hashable) -> None: ...
store("string")  # ✅ str 实现了 __hash__
# store([1, 2])  # ❌ list 没有 __hash__

# Annotated（附加元数据）
from fastapi import Depends
DatabaseSession = Annotated[Any, Depends(get_db)]
```text

### 1.8 常用标准库补充

#### itertools — 迭代器工具

```python
from itertools import chain, cycle, product, groupby, count

# chain — 串联多个可迭代对象
list(chain([1, 2], [3, 4]))           # [1, 2, 3, 4]
list(chain("AB", "CD"))                # ['A', 'B', 'C', 'D']

# cycle — 无限循环
colors = cycle(["red", "green", "blue"])
next(colors)  # 'red'
next(colors)  # 'green'

# product — 笛卡尔积（嵌套循环替代）
list(product([1, 2], ['a', 'b']))      # [(1,'a'), (1,'b'), (2,'a'), (2,'b')]

# groupby — 分组（需要先排序）
data = [("fruit", "苹果"), ("fruit", "香蕉"), ("tool", "锤子")]
for key, group in groupby(data, lambda x: x[0]):
    print(key, list(group))  # fruit → [('fruit','苹果'), ('fruit','香蕉')]

# count — 无限计数器
for i in count(start=1, step=2):
    if i > 10: break  # 1, 3, 5, 7, 9
```

#### pathlib — 现代文件路径处理

```python
from pathlib import Path

# 路径创建
base = Path("C:/Users/me/project")     # Windows 路径
data_dir = base / "data"                # 使用 / 拼接路径
config = data_dir / "config.json"

# 常用方法
config.read_text()                      # 读取文件内容
config.write_text('{"key": "value"}')   # 写入文件
config.exists()                         # 文件是否存在
config.is_file()                        # 是否是文件
config.suffix                           # '.json'
config.stem                             # 'config'（不带后缀）
config.parent                           # data_dir

# 遍历目录
for py_file in Path(".").glob("**/*.py"):     # 递归查找所有 .py 文件
    print(py_file.resolve())                   # 转绝对路径

# 创建目录
Path("output/logs").mkdir(parents=True, exist_ok=True)
```text

#### datetime — 日期时间处理

```python
from datetime import datetime, timedelta, timezone

# 当前时间
now = datetime.now()                        # 本地时间（不带时区）
utc_now = datetime.now(timezone.utc)        # UTC 时间（带时区）

# 格式化和解析
now.strftime("%Y-%m-%d %H:%M:%S")           # '2026-05-11 14:30:00'
datetime.strptime("2026-05-11", "%Y-%m-%d") # 解析字符串为日期

# 时间计算
tomorrow = now + timedelta(days=1)
last_week = now - timedelta(weeks=1)
diff = tomorrow - now  # timedelta 对象

# 实际项目用法（参考 order_service.py）
now = datetime.now(timezone.utc).replace(tzinfo=None)
if event.start_date <= now:
    raise ValueError("活动已开始")
```

#### functools — 函数工具

```python
from functools import partial, lru_cache, wraps

# partial — 固定函数的部分参数
def power(base, exp):
    return base ** exp

square = partial(power, exp=2)     # 固定 exp=2
cube = partial(power, exp=3)       # 固定 exp=3
square(5)  # 25
cube(5)    # 125

# lru_cache — 缓存函数结果（自动记忆化）
@lru_cache(maxsize=128)
def fib(n):
    """斐波那契数列（带缓存，避免重复计算）"""
    if n < 2:
        return n
    return fib(n-1) + fib(n-2)

fib(100)  # 瞬间返回（未缓存时需要循环到超时）

# wraps — 保留被装饰函数的元信息（用于装饰器）
def my_decorator(func):
    @wraps(func)  # 保留 func.__name__, func.__doc__ 等
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```text

#### collections — 扩展容器类型

```python
from collections import deque, Counter, defaultdict, namedtuple

# deque — 双端队列（两端高效插入/删除）
queue = deque(maxlen=100)       # 固定最大长度
queue.append("任务1")           # 右端添加
queue.appendleft("紧急任务")    # 左端添加
queue.pop()                     # 右端移除
queue.popleft()                 # 左端移除

# Counter — 计数
counts = Counter("hello world")
counts.most_common(3)           # [('l', 3), ('o', 2), ('h', 1)]
Counter([1, 2, 2, 3, 3, 3])    # Counter({3: 3, 2: 2, 1: 1})

# defaultdict — 带默认值的字典
dd = defaultdict(list)          # 访问不存在的键时自动创建空列表
dd['users'].append('小明')      # 无需先初始化列表
dd['users'].append('小红')      # ✅

# namedtuple — 命名元组（轻量级数据类）
Point = namedtuple('Point', ['x', 'y'])
p = Point(10, 20)
p.x       # 10（属性访问）
p[0]      # 10（索引访问）
x, y = p  # 解包
```

#### enum — 枚举类型

```python
from enum import Enum, auto, IntEnum

# 基本枚举
class OrderStatus(Enum):
    PENDING = "pending_payment"  # 待支付
    PAID = "paid"               # 已支付
    CANCELLED = "cancelled"     # 已取消
    REFUNDED = "refunded"       # 已退款

# 使用
status = OrderStatus.PENDING
status.value       # 'pending_payment'
status.name        # 'PENDING'

# 比较
status is OrderStatus.PENDING     # True（推荐用 is）
status == OrderStatus.PENDING     # True

# 遍历
for s in OrderStatus:
    print(f"{s.name}: {s.value}")

# auto — 自动赋值
class Priority(Enum):
    LOW = auto()      # 1
    MEDIUM = auto()   # 2
    HIGH = auto()     # 3
```text

#### 异步上下文管理器（async with）

```python
import asyncio
from contextlib import asynccontextmanager

# 方式一：实现 __aenter__ 和 __aexit__
class AsyncResource:
    async def __aenter__(self):
        print("获取资源")
        await asyncio.sleep(0.1)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("释放资源")
        await asyncio.sleep(0.1)

async def use_resource():
    async with AsyncResource() as res:
        print("使用资源中")

# 方式二：@asynccontextmanager（更简洁）
@asynccontextmanager
async def get_db():
    db = await connect_db()
    try:
        yield db
    finally:
        await db.close()

# 实际项目用法（参考 database.py）
async with async_session_factory() as session:
    result = await session.execute(query)
    # 退出时自动提交/回滚
```

#### 异步生成器（async for）

```python
# 异步生成器 — 用于分页查询、流式数据处理
async def paginate(query, page_size=100):
    offset = 0
    while True:
        page = await fetch_page(query, offset, page_size)
        if not page:
            break
        for item in page:
            yield item
        offset += page_size

# 使用
async for user in paginate("SELECT * FROM users"):
    print(user.name)
```text

---

## 2. pip 和虚拟环境

### 2.1 虚拟环境

虚拟环境为每个项目创建独立的 Python 环境，防止包版本冲突。

```bash
# 创建虚拟环境
python -m venv venv

# 激活虚拟环境 Windows PowerShell
venv\Scripts\Activate.ps1

# 激活后命令行前面会出现 (venv)

# 验证
where python  # 应该指向 venv/Scripts/python.exe
python --version
```

**虚拟环境隔离原理：** `venv` 会创建一个目录，复制 Python 解释器，创建独立的 `site-packages` 目录。激活后修改 `PATH` 环境变量，使 `python` 和 `pip` 指向虚拟环境内的版本。

### 2.2 pip 命令

```bash
# 安装包
pip install fastapi uvicorn

# 安装指定版本
pip install sqlalchemy==2.0.35

# 从 requirements.txt 安装
pip install -r requirements.txt

# 安装开发依赖
pip install -r requirements-dev.txt

# 导出已安装包
pip freeze > requirements.txt

# 显示已安装包
pip list
pip show fastapi

# 更新包
pip install --upgrade fastapi

# 卸载包
pip uninstall fastapi
```text

### 2.3 requirements.txt 语法

```txt
# 精确版本
fastapi==0.115.0

# 最低版本（向上兼容）
sqlalchemy>=2.0.35

# 兼容版本范围
pydantic>=2.0.0,<3.0.0

# 任意版本
redis

# 额外依赖（extras）
uvicorn[standard]

# Git 仓库安装
git+https://github.com/user/repo.git@tag

# 本地路径
-e ./local-package
```

### 2.4 pip 镜像源（速度慢的解决方案）

```bash
# 临时使用
pip install fastapi -i https://pypi.tuna.tsinghua.edu.cn/simple

# 永久配置
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```text

---

## 3. Docker 完全教程

Docker 让我们无需在 Windows 上直接安装 MySQL 和 Redis，而是通过容器运行它们。

### 3.1 Docker 架构

```

┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│ Docker Client │────▶│ Docker Daemon │────▶│  Registry    │
│  (docker CLI) │     │  (dockerd)    │     │  (Docker Hub)│
└──────────────┘     └───────┬───────┘     └──────────────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
              ┌──────────┐     ┌──────────┐
              │ Container │     │  Image   │
              │  (运行中)  │     │  (模板)   │
              └──────────┘     └──────────┘

```text

**核心概念：**

- **Image（镜像）**：只读模板，包含运行应用所需的全部文件
- **Container（容器）**：镜像的运行实例，可读写
- **Layer（层）**：每个 Dockerfile 指令创建一个只读层，UnionFS 合并成单一文件系统
- **Volume（卷）**：持久化容器数据（容器删除后数据保留）

### 3.2 Dockerfile 全部指令

```dockerfile
# FROM - 基础镜像（必须第一条）
FROM python:3.12-slim

# RUN - 执行命令（构建时）
RUN apt-get update && apt-get install -y curl

# WORKDIR - 设置工作目录
WORKDIR /app

# COPY - 从主机复制文件到镜像
COPY requirements.txt .
COPY app/ ./app/

# ADD - 增强版 COPY（支持 URL 下载、自动解压 tar）
ADD https://example.com/file.tar.gz /tmp/

# ENV - 设置环境变量
ENV PYTHONUNBUFFERED=1

# ARG - 构建参数（docker build --build-arg）
ARG VERSION=latest

# EXPOSE - 声明端口（只是文档，不实际映射）
EXPOSE 8000

# CMD - 容器启动命令（可以被 docker run 覆盖）
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

# ENTRYPOINT - 容器入口点（比 CMD 更难覆盖）
ENTRYPOINT ["python", "-m"]

# 组合使用：ENTRYPOINT 固定命令，CMD 提供默认参数
ENTRYPOINT ["uvicorn"]
CMD ["app.main:app", "--host", "0.0.0.0"]

# VOLUME - 声明挂载点
VOLUME ["/data"]

# HEALTHCHECK - 健康检查
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# USER - 切换用户（安全最佳实践）
RUN groupadd -r app && useradd -r -g app app
USER app

# LABEL - 添加元数据
LABEL maintainer="dev@example.com" version="1.0"
```

### 3.3 docker CLI 全部常用命令

```bash
# 镜像管理
docker pull python:3.12-slim    # 拉取镜像
docker images                   # 列出镜像
docker rmi python:3.12-slim     # 删除镜像
docker build -t myapp:latest .  # 构建镜像

# 容器管理
docker run -d \                  # -d: 后台运行
  --name ticket-api \            # 容器名称
  -p 8000:8000 \                 # 端口映射: 主机:容器
  -v ./data:/app/data \          # 卷挂载
  --env-file .env \               # 环境变量文件
  --restart unless-stopped \      # 重启策略
  myapp:latest

docker ps                    # 列出运行中的容器
docker ps -a                 # 列出所有容器（含已停止）
docker stop ticket-api       # 停止容器
docker start ticket-api      # 启动已停止的容器
docker restart ticket-api    # 重启容器
docker rm ticket-api         # 删除容器
docker logs -f ticket-api    # 查看日志（-f 持续跟踪）
docker exec -it ticket-api bash  # 在容器中执行命令
docker cp file.txt container:/path/  # 复制文件

# 系统管理
docker system prune         # 清理未使用的资源
docker system df            # 查看磁盘使用
docker stats                # 实时资源监控
```text

### 3.4 Docker Compose

Docker Compose 用 YAML 文件定义多容器应用。我们用它同时启动 MySQL 和 Redis。

**YAML 语法要点：**

```yaml
# 缩进表示层级
# 列表用短横线
# 字典用 key: value
# 锚点 & 和引用 * 可以复用配置

config: &defaults
  restart: unless-stopped

service:
  <<: *defaults  # 引用锚点
  image: mysql:8.0
```

**常用命令：**

```bash
docker-compose up -d       # 后台启动所有服务
docker-compose down        # 停止并删除容器
docker-compose logs -f     # 查看日志
docker-compose ps          # 查看服务状态
docker-compose exec mysql bash  # 在服务中执行命令
docker-compose restart     # 重启服务
```text

---

## 4. PowerShell 和 Git 基础

### 4.1 PowerShell 常用命令

因为我们在 Windows 上开发，掌握 PowerShell 基础很有帮助。

```powershell
# 文件操作
Get-ChildItem          # ls / dir：列出文件
Set-Location ..        # cd：切换目录
New-Item -Path dir -ItemType Directory  # mkdir：创建目录
Remove-Item file.txt   # rm：删除文件
Copy-Item src dest     # cp：复制
Move-Item src dest     # mv：移动
Get-Content file.txt   # cat：查看文件内容

# 文本处理
Select-String "error" *.log   # grep：搜索文本

# 网络请求
Invoke-RestMethod -Uri http://localhost:8000/health -Method Get

# 环境变量
$env:DB_HOST = "localhost"
Get-ChildItem Env:     # 查看所有环境变量
```

### 4.2 Git 基础

```bash
# 初始化
git init
git clone https://github.com/user/repo.git

# 日常操作
git status                 # 查看状态
git add file.py            # 暂存文件
git add .                  # 暂存所有
git commit -m "feat: init" # 提交
git push                   # 推送
git pull                   # 拉取

# 分支管理
git branch feature-x       # 创建分支
git checkout feature-x     # 切换分支
git switch feature-x       # 切换分支（新语法）
git merge feature-x        # 合并分支

# 查看历史
git log --oneline --graph  # 简洁历史
git diff                   # 查看改动
```text

---

## 5. 票务系统项目脚手架搭建

### 5.1 创建项目目录

```

ticket-system/
├── .env                  # 环境变量
├── .gitignore            # Git 忽略规则
├── requirements.txt       # Python 依赖
├── docker-compose.yml     # MySQL + Redis 服务
├── app/                   # FastAPI 应用
│   ├── **init**.py
│   ├── main.py            # 应用入口
│   ├── config.py          # 配置管理
│   ├── database.py        # 数据库连接
│   ├── redis_client.py    # Redis 连接
│   ├── models/            # ORM 模型
│   ├── schemas/           # Pydantic 模型
│   ├── routers/           # API 路由
│   ├── services/          # 业务逻辑
│   ├── middleware/        # 中间件
│   └── scripts/           # Lua 脚本等
├── tests/                 # 测试
├── scripts/               # 运维脚本
├── nginx/                 # Nginx 配置
├── tutorials/             # 教程文件
└── postman/               # Postman 集合

```text

### 5.2 启动 MySQL 和 Redis

```bash
# 启动 Docker 容器
docker-compose up -d

# 验证 MySQL
docker exec -it ticket-mysql mysql -uroot -proot123 -e "SHOW DATABASES;"

# 验证 Redis
docker exec -it ticket-redis redis-cli ping
# 应该返回 PONG
```

### 5.3 启动 FastAPI 应用

```bash
# 创建并激活虚拟环境
python -m venv venv
venv\Scripts\Activate.ps1

# 安装依赖
pip install -r requirements.txt

# 启动开发服务器（热重载）
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# 或者使用模块方式
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```text

### 5.4 验证

打开浏览器访问以下地址：

- API 文档（Swagger UI）：<http://localhost:8000/docs>
- API 文档（Redoc）：<http://localhost:8000/redoc>
- 健康检查：<http://localhost:8000/health>

看到 `{"status":"ok","service":"TicketBooking","env":"development"}` 即表示成功！

## 6. 本章完整代码

本章创建的文件：

- `requirements.txt` — Python 依赖列表
- `.env` — 环境变量配置
- `.gitignore` — Git 忽略规则
- `docker-compose.yml` — MySQL + Redis 容器定义
- `app/__init__.py` — Python 包标记
- `app/main.py` — FastAPI 应用入口 + 健康检查
- `app/config.py` — 配置管理（加载 .env）
- `app/database.py` — SQLAlchemy 异步引擎
- `app/redis_client.py` — Redis 连接池
- `app/models/__init__.py` — 模型包
- `app/schemas/__init__.py` — 模式包
- `app/routers/__init__.py` — 路由包
- `app/services/__init__.py` — 服务包
- `app/middleware/__init__.py` — 中间件包
- `app/utils/__init__.py` — 工具包

## 7. 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `pip install` 速度极慢 | 默认 PyPI 源在国外 | `pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple` |
| Docker Desktop 无法启动 | WSL2 未正确安装 | 检查 `wsl --status`，确保内核更新 |
| `ModuleNotFoundError: No module named 'app'` | PYTHONPATH 或运行目录 | 在项目根目录运行，或用 `python -m app.main` |
| MySQL 连接被拒绝 | Docker 容器未就绪 | `docker-compose ps` 检查，等 healthcheck 通过 |
| Redis ping 返回错误 | Redis 端口映射不对 | 检查 `docker-compose.yml` 的 ports 配置 |
| 虚拟环境激活后 pip 仍指向全局 | 路径顺序问题 | 重新激活：`deactivate` 再 `.venv\Scripts\Activate.ps1` |

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `.env.example` | 项目环境变量模板（配置数据库/Redis/支付宝密钥） |
| `app/config.py` | 配置管理（BaseSettings，从环境变量加载） |
| `docker-compose.yml` | 开发环境 Docker 编排（MySQL + Redis + App） |
| `Dockerfile` | 应用镜像构建（多阶段构建） |
| `requirements.txt` | Python 依赖清单 |

**关键配置示例：** 详见 `app/config.py`，使用 Pydantic `BaseSettings` 统一管理所有配置项。

## 8. 本章总结

✅ **已掌握：**

- Python 完整语法：全部类型、运算符、控制流、函数（含装饰器/生成器）、类（含MRO）、异常、异步编程
- 虚拟环境和 pip 包管理
- Docker 和 Docker Compose 的完整用法
- PowerShell 和 Git 基础
- 票务系统项目骨架搭建并成功运行

✅ **项目里程碑：**

- [x] `uvicorn` 启动成功
- [x] `/health` 返回 200
- [x] MySQL 可连接（Docker）
- [x] Redis 可连接（Docker）
- [x] Swagger /docs 可访问

**下一章预告：** 第2章将深入学习 SQL 和 MySQL，设计票务系统的数据库 Schema（用户表、活动表、订单表、票务表），掌握从 DDL 到事务隔离级别的全部知识。
