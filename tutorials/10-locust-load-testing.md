# 第10章：Locust 压力测试 — 从入门到防超卖验证

## 本章学习目标

本章将系统学习 Locust 压力测试框架，设计三种压测场景（浏览、抢票、最后一票），验证票务系统在高并发下不超卖。

**学习路径：**

1. Locust 完全教程：HttpUser、task、wait_time、TaskSet
2. Web UI 与 Headless 模式
3. CSV 报告与分布式压测
4. 三场景压测设计
5. Little's Law 公式证明
6. 防超卖压力验证

---

## 1. Locust 完全教程

### 1.1 安装与入门

```bash
pip install locust
```text

**最简单的 Locust 脚本：**

```python
# locustfile.py
from locust import HttpUser, task, between


class WebsiteUser(HttpUser):
    """模拟用户行为"""

    # 每个用户的操作间隔（秒），between(a, b) 在 [a, b] 范围内随机
    wait_time = between(1, 5)

    @task
    def view_homepage(self):
        """访问首页"""
        self.client.get("/")
```

**启动：**

```bash
locust -f locustfile.py --host=http://localhost:8000
```text

打开浏览器访问 `http://localhost:8089` 进入 Web UI。

### 1.2 HttpUser 详解

```python
from locust import HttpUser, task, between


class TicketUser(HttpUser):
    """
    HttpUser 是 Locust 中最常用的用户类。

    关键属性：
    - client: httpx 的 HTTP 会话，自动管理 Cookie
    - wait_time: 任务之间的等待时间
    - host: 目标服务器地址（在命令行指定 --host）
    - weight: 用户类权重（多用户类时概率分配）
    """

    # ===== wait_time 策略 =====

    # 固定等待时间
    # wait_time = 1  # 每个任务后等待 1 秒

    # 随机范围等待
    wait_time = between(0.5, 3.0)

    # 常数等待（所有用户相同）
    # wait_time = constant(1.5)

    # 通过时间函数（如指数分布的思考时间）
    # wait_time = constant_throughput(10)  # 每秒 10 个请求

    # ===== 权重 =====

    # 如果有多个 HttpUser 子类，weight 决定了虚拟用户分配的比例
    # weight = 3  # 这个类的用户占总用户的 3/(3+5) = 37.5%

    # ===== on_start / on_stop =====

    def on_start(self):
        """
        每个虚拟用户启动时执行一次。
        适合做登录、获取 token 等初始化操作。

        注意：
        - on_start 在第一个 @task 之前执行
        - on_start 的执行时间不计入 wait_time
        - 如果 on_start 抛出异常，该用户会退出
        """
        # 登录并保存 token
        response = self.client.post("/api/v1/auth/login", json={
            "username": "loadtester",
            "password": "test123",
        })
        if response.status_code == 200:
            token = response.json()["access_token"]
            # 为后续请求设置认证头
            self.client.headers.update({
                "Authorization": f"Bearer {token}",
            })

    def on_stop(self):
        """
        每个虚拟用户停止时执行一次。
        适合做资源清理。
        """
        pass
```

### 1.3 @task 装饰器详解

```python
from locust import HttpUser, task, between


class TaskExample(HttpUser):
    wait_time = between(1, 3)

    # ===== 基本用法 =====

    @task
    def task_one(self):
        """简单任务，权重默认为 1"""
        self.client.get("/page1")

    # ===== 带权重的任务 =====

    @task(3)  # 这个任务被选中的概率是 weight=1 的任务的 3 倍
    def task_two(self):
        """权重为 3，表示比其他权重的任务更容易被选中"""
        self.client.get("/page2")

    @task
    def task_three(self):
        """权重为 1"""
        self.client.get("/page3")

    """
    以上三个任务的选中概率：
    task_two: 3 / (1 + 3 + 1) = 60%
    task_one: 1 / 5 = 20%
    task_three: 1 / 5 = 20%
    """
```text

### 1.4 TaskSet 详解

TaskSet 允许将相关任务组织在一起，实现更复杂的用户行为：

```python
from locust import HttpUser, task, between, TaskSet


class BrowseBehavior(TaskSet):
    """浏览行为：查看活动列表和详情"""

    def on_start(self):
        """进入 TaskSet 时执行"""
        print("开始浏览行为")

    @task(4)
    def list_events(self):
        """浏览活动列表（高频）"""
        self.client.get("/api/v1/events/")

    @task(1)
    def view_event_detail(self):
        """查看活动详情"""
        # 使用一个固定活动 ID（压测环境预置）
        self.client.get("/api/v1/events/1")

    @task(1)
    def search_events(self):
        """搜索活动"""
        self.client.get("/api/v1/events/?keyword=音乐会&page=1")

    def on_stop(self):
        """离开 TaskSet 时执行"""
        print("结束浏览行为")


class BuyBehavior(TaskSet):
    """购买行为：登录后抢票"""

    token = None

    def on_start(self):
        """登录获取 token"""
        response = self.client.post("/api/v1/auth/login", json={
            "username": "load_test_user",
            "password": "test123",
        })
        if response.status_code == 200:
            self.token = response.json()["access_token"]
            self.client.headers.update({
                "Authorization": f"Bearer {self.token}",
            })

    @task(1)
    def buy_ticket(self):
        """下单购买"""
        with self.client.post(
            "/api/v1/orders/",
            json={"event_id": 1, "quantity": 1},
            catch_response=True,
            name="/api/v1/orders/ [BUY]",
        ) as response:
            if response.status_code == 201:
                response.success()
            elif response.status_code == 400:
                # 库存不足是预期行为，标记为"失败"但不产生异常
                response.failure(f"购买失败: {response.text}")
            else:
                response.failure(f"意外状态码: {response.status_code}")


class WebsiteUser(HttpUser):
    """组合用户行为"""

    tasks = [BrowseBehavior, BuyBehavior]
    wait_time = between(1, 3)

    # TaskSet 的选中权重
    # BrowseBehavior 被选中的概率 / BuyBehavior 被选中的概率
```

**TaskSet 嵌套：**

```python
from locust import HttpUser, task, between, TaskSet


class SearchBehavior(TaskSet):
    """搜索子任务"""

    @task(3)
    def search_by_keyword(self):
        self.client.get("/api/v1/events/?keyword=test")

    @task(1)
    def search_by_category(self):
        self.client.get("/api/v1/events/?category=music")

    @task(1)
    def search_empty(self):
        self.client.get("/api/v1/events/?keyword=__nonexistent__")


class MainBehavior(TaskSet):
    """主任务集"""

    @task(2)
    class SearchTasks(SearchBehavior):
        """嵌套 TaskSet：搜索行为"""
        weight = 2

    @task(1)
    def index(self):
        self.client.get("/")

    @task(1)
    def about(self):
        self.client.get("/about")


class WebsiteUser(HttpUser):
    tasks = [MainBehavior]
    wait_time = between(1, 3)
```text

### 1.5 on_start 详细用法

```python
from locust import HttpUser, task, between
import random


class AuthenticatedUser(HttpUser):
    """带认证的用户"""
    wait_time = between(1, 3)

    def on_start(self):
        """
        虚拟用户启动时的初始化。

        重要的设计考虑：
        1. on_start 的执行时间不计入 wait_time
        2. 如果 on_start 失败（抛出异常），用户会直接退出
        3. 适合做：登录、创建资源、加载数据

        对于大规模压测（1000+ 用户），on_start 中的操作
        会成为瓶颈。建议：
        - 预先生成一批 token
        - on_start 中直接从队列取 token
        - 或使用固定测试账号
        """
        # 从预定义的测试用户池中选取
        test_users = [
            {"username": "loadtest_1", "password": "test123"},
            {"username": "loadtest_2", "password": "test123"},
            # ... 更多用户
        ]
        user = random.choice(test_users)

        # 登录
        with self.client.post(
            "/api/v1/auth/login",
            json=user,
            catch_response=True,
        ) as response:
            if response.status_code == 200:
                token = response.json()["access_token"]
                self.client.headers["Authorization"] = f"Bearer {token}"
                response.success()
            else:
                response.failure(f"登录失败: {response.text}")
                raise Exception("登录失败，用户退出")

    @task
    def my_profile(self):
        """获取个人信息"""
        self.client.get("/api/v1/auth/me")
```

### 1.6 catch_response 详解

`catch_response=True` 是 Locust 中最重要的参数之一。它允许手动控制请求的成功/失败状态：

```python
from locust import HttpUser, task, between


class ResponseHandling(HttpUser):
    wait_time = between(1, 3)

    @task
    def manual_response_handling(self):
        """
        使用 catch_response 的场景：

        1. 将非 2xx 但业务逻辑正确的响应标记为成功
        2. 将 2xx 但业务逻辑错误的响应标记为失败
        3. 自定义失败原因
        """

        # 场景一：将 4xx 标记为成功（预期行为）
        with self.client.get(
            "/api/v1/events/99999",
            catch_response=True,
        ) as response:
            if response.status_code == 404:
                # 404 是预期的（活动不存在），不记为失败
                response.success()
            elif response.status_code == 200:
                response.success()
            else:
                response.failure(f"意外响应: {response.status_code}")

        # 场景二：将 2xx 标记为失败（业务错误）
        with self.client.post(
            "/api/v1/orders/",
            json={"event_id": 1, "quantity": 1},
            catch_response=True,
        ) as response:
            if response.status_code == 201:
                response.success()
            elif response.status_code == 400:
                # 虽然是 400，但返回值可能表示"库存不足"
                # 这应该算业务失败
                response.failure(f"购买失败: {response.text}")
            else:
                response.failure(f"未知错误: {response.status_code}")

        # 场景三：检查响应内容
        with self.client.get(
            "/api/v1/events/",
            catch_response=True,
        ) as response:
            if response.status_code != 200:
                response.failure(f"获取活动列表失败: {response.status_code}")
            else:
                data = response.json()
                if "items" not in data:
                    response.failure("响应缺少 items 字段")
                elif data["total"] < 0:
                    response.failure("total 为负数")
                else:
                    response.success()
```text

### 1.7 wait_time 策略对比

```python
from locust import HttpUser, task, constant, between, constant_throughput, constant_pacing


class WaitTimeStrategies:
    """
    Locust 的 wait_time 策略

    between(min, max):
      均匀随机等待 [min, max] 秒
      最常用，模拟用户的思考时间
      wait_time = between(1, 3)

    constant(seconds):
      每次固定等待 N 秒
      适合模拟恒定速率的请求
      wait_time = constant(1)

    constant_throughput(requests_per_second):
      控制每秒请求数（所有用户总计）
      系统会自动调整等待时间以达到目标吞吐量
      wait_time = constant_throughput(100)  # 总计 100 req/s

    constant_pacing(seconds):
      每次请求后固定等待 N 秒（从请求开始到下次请求开始）
      适合精确控制请求间隔
      wait_time = constant_pacing(1.5)

    自定义函数：
      def my_wait_time():
          return random.expovariate(0.5)  # 指数分布
      wait_time = my_wait_time
    """


class ConstantUser(HttpUser):
    wait_time = constant(1)

    @task
    def t(self):
        self.client.get("/")


class ThroughputUser(HttpUser):
    wait_time = constant_throughput(50)  # 所有用户合计 50 req/s

    @task
    def t(self):
        self.client.get("/")
```

---

## 2. Web UI 详解

### 2.1 启动 Web UI

```bash
locust -f locustfile.py --host=http://localhost:8000
```text

然后打开浏览器访问 `http://localhost:8089`。

### 2.2 页面设置

在 Web UI 中可以设置：

- **Number of total users to simulate**：模拟用户总数（峰值并发数）
- **Spawn rate**：用户生成速率（每秒增加的用户数）
- **Host**：目标服务器地址

**建议的设置模式：**

```

阶段一：预热（1分钟）
  users: 10,  spawn rate: 1

阶段二：正常负载（3分钟）
  users: 50,  spawn rate: 5

阶段三：峰值负载（3分钟）
  users: 200, spawn rate: 10

阶段四：熔断观察（1分钟）
  users: 500, spawn rate: 20

```text

### 2.3 图表详解

#### 总请求数/每秒 (Total Requests Per Second / RPS)

**RPS 图表解读：**

```

RPS (Requests Per Second)
  ▲
  │    ██
  │   ████
  │  ██████
  │ ████████
  │██████████████████████    ← 稳定期
  └─────────────────────────→ 时间

- 上升阶段：用户逐渐增加
- 稳定阶段：系统处理能力
- 下降阶段：用户退出
- 平台期高度 = 系统最大吞吐量

```text

**如何判断是否达到系统瓶颈：**

- RPS 不再随用户数增加而增加 → 系统达到瓶颈
- RPS 突然下降 → 系统可能崩溃或触发限流
- RPS 稳定但有大量失败 → 部分请求失败但系统仍在处理

#### 响应时间 (Response Times)

**百分位响应时间：**

```

Response Time (ms)
  ▲
  │                  ×××  P99
  │                ××
  │              ××
  │            ××           P95
  │          ××
  │        ××
  │      ××                 P50 (中位数)
  │    ××
  │  ××
  │××
  └────────────────────────→ 时间

解读：

- P50（中位数）：一半的请求在这个时间内完成
- P95：95% 的请求在这个时间内完成
- P99：99% 的请求在这个时间内完成
- P50 和 P99 差距大 → 存在长尾延迟

```text

**平均响应时间 vs 百分位：**

```python
def response_time_analysis():
    """
    只用平均响应时间是很危险的！

    例子：10 个请求的响应时间
    请求: [10ms, 12ms, 11ms, 13ms, 10ms, 15ms, 14ms, 11ms, 12ms, 5000ms]
    平均: (10+12+11+13+10+15+14+11+12+5000) / 10 = 510.8ms

    但如果去掉那个异常值：
    平均: (10+12+11+13+10+15+14+11+12) / 9 = 12ms

    结论：看 P99 而不是平均值！
    平均值会被异常值严重扭曲。
    """
```

#### 每分钟百分位 (Percentiles per Minute)

这个图表显示每分钟的响应时间百分位变化趋势。如果 P99 持续上升，说明系统负载过高。

#### 失败率 (Failures/s)

```text
Failures/s
  ▲
  │
  │         ████████
  │         ████████
  │         ████████
  │══════════════════════════    ← 1% 阈值
  │
  └────────────────────────→ 时间

- 失败率突然飙升：系统可能崩溃
- 失败率缓慢上升：系统逐渐饱和
- 失败集中在新用户：可能是认证/初始化问题
```

#### 用户数 (Number of Users)

显示当前活跃用户数，通常与 RPS 图表对比查看。

### 2.4 图表关联分析方法

```python
def chart_correlation_analysis():
    """
    图表关联分析指南

    健康系统：
      RPS ↑, RT 稳定, Failures 低 → 系统正常扩容

    异常模式 1：队列堆积
      RPS 不变, RT ↑, Failures 不变
      → 请求排队，可能需要增加 worker

    异常模式 2：资源耗尽
      RPS ↓, RT ↓, Failures ↑
      → 系统过载，部分请求被拒绝
      例：MySQL max_connections 被耗尽
      → 新连接被拒绝（失败），已有连接在工作

    异常模式 3：雪崩
      RPS ↓↓↓, RT ↑↑↑, Failures ↑↑↑
      → 系统正在崩溃
      例：数据库连接池耗尽 → 请求等待连接 → 更多请求堆积
      → 最终所有请求超时

    异常模式 4：瞬时尖峰
      RPS 尖峰, RT 尖峰（瞬时有秒级响应）
      → 慢查询 / GC pause / 网络抖动
    """
```text

---

## 3. 命令行模式

### 3.1 Headless 模式

```bash
# 基本 Headless 模式
locust -f locustfile.py \
    --host=http://localhost:8000 \
    --headless \
    -u 100 \          # 并发用户数
    -r 10 \           # 每秒生成用户数
    --run-time 30s    # 运行时间

# 指定时间单位
# --run-time 1m30s  # 1 分 30 秒
# --run-time 1h     # 1 小时
```

### 3.2 CSV 报告生成

```bash
# 生成 CSV 报告
locust -f locustfile.py \
    --host=http://localhost:8000 \
    --headless \
    -u 100 -r 10 --run-time 30s \
    --csv=reports/stress_test

# 会生成以下文件（--csv 前缀）：
# reports/stress_test_stats.csv      # 总体统计
# reports/stress_test_stats_history.csv  # 时间序列统计
# reports/stress_test_failures.csv   # 失败详情
# reports/stress_test_exceptions.csv # 异常详情
```text

**CSV 文件格式：**

```csv
# stress_test_stats.csv
Type,Name,Request Count,Failure Count,Median Response Time,Average Response Time,Min Response Time,Max Response Time,Average Content Size,Requests/s,Failures/s,50%,66%,75%,80%,90%,95%,98%,99%,99.9%,99.99%,100%
GET,/api/v1/events/,1500,0,12,15,5,120,450,50.0,0.0,12,14,16,18,25,40,60,80,110,120,120
POST,/api/v1/orders/,500,25,45,52,10,300,200,16.7,0.83,45,50,55,60,80,120,180,220,280,300,300
```

**CSV 分析脚本：**

```python
import csv
import matplotlib.pyplot as plt
from datetime import datetime


def analyze_locust_csv(stats_file: str):
    """
    分析 Locust CSV 报告
    """
    with open(stats_file, "r") as f:
        reader = csv.DictReader(f)
        rows = list(reader)

    for row in rows:
        print(f"=== {row['Type']} {row['Name']} ===")
        print(f"  请求数: {row['Request Count']}")
        print(f"  失败数: {row['Failure Count']}")
        print(f"  失败率: {float(row['Failure Count'])/float(row['Request Count'])*100:.2f}%")
        print(f"  中位数响应: {row['Median Response Time']}ms")
        print(f"  平均响应: {row['Average Response Time']}ms")
        print(f"  95% 响应: {row['95%']}ms")
        print(f"  99% 响应: {row['99%']}ms")
        print(f"  最大响应: {row['Max Response Time']}ms")
        print(f"  吞吐量: {row['Requests/s']} req/s")
```text

### 3.3 分布式模式

当单台机器无法生成足够的负载时（例如需要 10000+ 并发），可以使用分布式模式。

```bash
# 步骤 1：启动 Master（控制节点）
locust -f locustfile.py \
    --master \
    --master-bind-host=0.0.0.0 \
    --master-bind-port=5557 \
    --host=http://localhost:8000 \
    --web-port=8089

# 步骤 2：启动 Worker（工作节点，可以在多台机器上运行）
locust -f locustfile.py \
    --worker \
    --master-host=192.168.1.100 \  # Master 的 IP
    --master-port=5557

# 可以同时启动多个 Worker
# 在另外的终端中：
locust -f locustfile.py --worker --master-host=192.168.1.100 --master-port=5557
```

**分布式架构：**

```text
Master (控制节点)
  ├── Web UI (port 8089): 配置参数、查看图表
  ├── 协调所有 Worker
  └── 汇总统计信息

Worker 1 (机器 A)
  ├── 生成虚拟用户
  ├── 发送 HTTP 请求
  └── 报告结果给 Master

Worker 2 (机器 B)
  ├── 生成虚拟用户
  ├── 发送 HTTP 请求
  └── 报告结果给 Master

Worker 3 (机器 C)
  ├── ...
```

**分布式模式关键点：**

- Master 不生成负载，只做协调和统计
- Worker 数量 * 每个 Worker 的用户数 = 总并发用户数
- 如果 Worker 都在同一台机器，考虑 CPU/内存限制
- 网络延迟：Worker 到 Master 的通信延迟会影响统计准确性

**Docker 分布式部署：**

```yaml
# docker-compose.yml
version: '3.8'

services:
  master:
    image: locustio/locust
    ports:
      - "8089:8089"
    volumes:
      - ./locustfile.py:/mnt/locust/locustfile.py
    command: -f /mnt/locust/locustfile.py --master --host=http://host.docker.internal:8000

  worker:
    image: locustio/locust
    volumes:
      - ./locustfile.py:/mnt/locust/locustfile.py
    command: -f /mnt/locust/locustfile.py --worker --master-host=master
    deploy:
      replicas: 3  # 3 个 Worker
```text

---

## 4. 三场景压测设计

以下是针对票务系统的三个压力测试场景设计。

### 4.1 场景一：浏览密集 (Browsing)

```python
"""
场景一：浏览密集 (Browsing)

特点：
- 用户行为：浏览活动列表和详情
- 思考时间：1-3 秒（模拟浏览行为）
- 权重：5（主要场景）
- 目标：测试活动列表和详情接口的读性能
"""

from locust import HttpUser, task, between
import random


class BrowsingUser(HttpUser):
    """
    浏览型用户：查看活动列表和详情

    模拟真实的浏览行为：
    1. 先看列表（列表页）
    2. 选择感兴趣的活动点进去看详情
    3. 可能搜索或过滤
    4. 偶尔翻页
    """
    wait_time = between(1, 3)
    weight = 5  # 占压测流量的 5/9 ≈ 56%

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.event_ids = []

    def on_start(self):
        """首次启动时获取活动列表（获取 ID 列表）"""
        try:
            response = self.client.get(
                "/api/v1/events/",
                catch_response=True,
            )
            if response.status_code == 200:
                data = response.json()
                self.event_ids = [e["id"] for e in data.get("items", [])]
                response.success()
            else:
                response.failure(f"获取活动列表失败: {response.status_code}")
                self.event_ids = []
        except Exception as e:
            print(f"Error fetching events: {e}")
            self.event_ids = []

    @task(6)
    def list_events(self):
        """浏览活动列表（最高频）"""
        with self.client.get(
            "/api/v1/events/",
            catch_response=True,
            name="GET /events/",
        ) as response:
            data = response.json()
            if response.status_code == 200 and "items" in data:
                response.success()
            else:
                response.failure(f"列表异常: {response.status_code}")

    @task(2)
    def view_event_detail(self):
        """查看活动详情"""
        if not self.event_ids:
            return
        event_id = random.choice(self.event_ids)
        with self.client.get(
            f"/api/v1/events/{event_id}",
            catch_response=True,
            name="GET /events/{id}",
        ) as response:
            if response.status_code == 200:
                response.success()
            else:
                response.failure(f"详情异常: {response.status_code}")

    @task(1)
    def search_events(self):
        """搜索活动"""
        keywords = ["音乐会", "演出", "话剧", "展览"]
        keyword = random.choice(keywords)
        with self.client.get(
            f"/api/v1/events/?keyword={keyword}",
            catch_response=True,
            name="GET /events/?search",
        ) as response:
            if response.status_code == 200:
                response.success()
            else:
                response.failure(f"搜索异常: {response.status_code}")
```

### 4.2 场景二：抢票高峰 (Flash Sale)

```python
"""
场景二：抢票高峰 (Flash Sale)

特点：
- 用户行为：疯狂下单抢票
- 思考时间：极少（0.1-0.3 秒），或者无等待
- 权重：3
- 目标：测试下单接口的写性能，验证 Redis 库存扣减速度
"""

from locust import HttpUser, task, between, constant
import random


class FlashSaleUser(HttpUser):
    """
    抢票型用户：登录后疯狂下单

    模拟抢票高峰行为：
    1. on_start 中完成登录
    2. 登录后连续下单，几乎没有思考时间
    3. 即使是库存不足，也会继续尝试
    """
    wait_time = between(0.1, 0.3)  # 思考时间极短
    weight = 3  # 占压测流量的 3/9 ≈ 33%

    def on_start(self):
        """登录并获取 token"""
        username = f"flashbuyer_{random.randint(1, 10000)}"
        # 先注册
        self.client.post("/api/v1/auth/register", json={
            "username": username,
            "password": "test123",
            "email": f"{username}@test.com",
        })
        # 再登录
        response = self.client.post("/api/v1/auth/login", json={
            "username": username,
            "password": "test123",
        })
        if response.status_code == 200:
            token = response.json()["access_token"]
            self.client.headers.update({
                "Authorization": f"Bearer {token}",
            })

    @task
    def rush_buy(self):
        """疯狂抢票"""
        # 尝试购买 1-2 张
        quantity = random.choice([1, 1, 1, 2])  # 大部分买 1 张

        with self.client.post(
            "/api/v1/orders/",
            json={"event_id": 1, "quantity": quantity},
            catch_response=True,
            name="POST /orders/ [rush]",
        ) as response:
            if response.status_code == 201:
                response.success()
            elif response.status_code == 400:
                # 库存不足：这是预期的失败
                # 但标记为"fail"供统计使用
                response.failure("库存不足")
            else:
                response.failure(f"意外错误: {response.status_code}")
```text

### 4.3 场景三：最后一票并发 (Last Ticket)

```python
"""
场景三：最后一票并发 (Last Ticket)

特点：
- 用户行为：100 人同时抢最后 1 张票
- 思考时间：无（0 等待）
- 权重：1
- 目标：验证防超卖系统的正确性
"""

from locust import HttpUser, task, constant, between
import random


class LastTicketUser(HttpUser):
    """
    最后一票型用户：100 人抢 1 张

    这是防超卖的关键测试场景：
    - 预设库存只有 1 张
    - 100 个用户同时下单
    - 验证：成功数 <= 1
    """
    wait_time = constant(0)  # 不做等待，立即发送下一个请求
    weight = 1  # 占压测流量的 1/9 ≈ 11%

    def on_start(self):
        """登录"""
        # 使用预生成的 token（避免注册成为瓶颈）
        username = f"lastticket_{random.randint(1, 100000)}"
        self.client.post("/api/v1/auth/register", json={
            "username": username,
            "password": "test123",
            "email": f"{username}@test.com",
        })
        response = self.client.post("/api/v1/auth/login", json={
            "username": username,
            "password": "test123",
        })
        if response.status_code == 200:
            self.client.headers["Authorization"] = (
                f"Bearer {response.json()['access_token']}"
            )

    @task
    def grab_last_ticket(self):
        """抢最后一票"""
        with self.client.post(
            "/api/v1/orders/",
            json={"event_id": 1, "quantity": 1},
            catch_response=True,
            name="POST /orders/ [last ticket]",
        ) as response:
            if response.status_code == 201:
                response.success()
            elif response.status_code == 400:
                # 库存不足：预期行为
                response.failure("库存不足")
            else:
                response.failure(f"意外错误: {response.status_code}")
```

### 4.4 完整 Locustfile 组合

```python
"""
完整的 locustfile.py — 三场景压测

启动命令：
  locust -f locustfile.py --host=http://localhost:8000
  locust -f locustfile.py --host=http://localhost:8000 --headless -u 200 -r 20 --run-time 60s --csv=report
"""

from locust import HttpUser, task, between, constant
import random


# ===== 场景一：浏览密集 =====

class BrowsingUser(HttpUser):
    """浏览型用户（流量占比最高）"""
    wait_time = between(1, 3)
    weight = 5

    event_ids = []

    def on_start(self):
        response = self.client.get("/api/v1/events/")
        if response.status_code == 200:
            data = response.json()
            self.event_ids = [e["id"] for e in data.get("items", [])]

    @task(6)
    def list_events(self):
        with self.client.get(
            "/api/v1/events/",
            catch_response=True,
            name="GET /events/",
        ) as resp:
            if resp.status_code == 200:
                resp.success()
            else:
                resp.failure(f"{resp.status_code}")

    @task(2)
    def view_detail(self):
        if not self.event_ids:
            return
        eid = random.choice(self.event_ids)
        with self.client.get(
            f"/api/v1/events/{eid}",
            catch_response=True,
            name="GET /events/{id}",
        ) as resp:
            if resp.status_code == 200:
                resp.success()
            else:
                resp.failure(f"{resp.status_code}")


# ===== 场景二：抢票高峰 =====

class FlashSaleUser(HttpUser):
    """抢票型用户（写密集型）"""
    wait_time = between(0.1, 0.3)
    weight = 3

    def on_start(self):
        uid = random.randint(100000, 999999)
        self.client.post("/api/v1/auth/register", json={
            "username": f"flash_{uid}",
            "password": "p123",
            "email": f"f{uid}@t.com",
        })
        resp = self.client.post("/api/v1/auth/login", json={
            "username": f"flash_{uid}",
            "password": "p123",
        })
        if resp.status_code == 200:
            self.client.headers["Authorization"] = (
                f"Bearer {resp.json()['access_token']}"
            )

    @task
    def buy(self):
        with self.client.post(
            "/api/v1/orders/",
            json={"event_id": 1, "quantity": 1},
            catch_response=True,
            name="POST /orders/",
        ) as resp:
            if resp.status_code == 201:
                resp.success()
            elif resp.status_code == 400:
                resp.failure("库存不足")
            else:
                resp.failure(f"{resp.status_code}")


# ===== 场景三：最后一票 =====

class LastTicketUser(HttpUser):
    """最后一票并发验证"""
    wait_time = constant(0)
    weight = 1

    def on_start(self):
        uid = random.randint(1, 999999)
        self.client.post("/api/v1/auth/register", json={
            "username": f"last_{uid}",
            "password": "p123",
            "email": f"l{uid}@t.com",
        })
        resp = self.client.post("/api/v1/auth/login", json={
            "username": f"last_{uid}",
            "password": "p123",
        })
        if resp.status_code == 200:
            self.client.headers["Authorization"] = (
                f"Bearer {resp.json()['access_token']}"
            )

    @task
    def grab(self):
        with self.client.post(
            "/api/v1/orders/",
            json={"event_id": 1, "quantity": 1},
            catch_response=True,
            name="POST /orders/ [last]",
        ) as resp:
            if resp.status_code == 201:
                resp.success()
            elif resp.status_code == 400:
                resp.failure("库存不足")
            else:
                resp.failure(f"{resp.status_code}")
```text

---

## 5. Little's Law 公式证明与应用

### 5.1 公式定义

Little's Law 是排队论中最基本的定律之一：

```

L = λ × W

其中：
L = 系统中平均请求数（并发数）
λ = 系统吞吐率（每秒完成的请求数）
W = 平均响应时间（秒）

```text

### 5.2 数学证明

```python
def littles_law_proof():
    """
    Little's Law 的直观证明

    考虑一个系统在时间区间 [0, T] 内的行为：

    设：
    - A(T): 在 [0, T] 内到达的请求数
    - C(T): 在 [0, T] 内完成的请求数
    - N(t): 在时刻 t，系统中的请求数（包括正在服务和排队中的）

    定义：
    到达率 λ = A(T) / T
    吞吐率 X = C(T) / T
    在稳态下，λ = X

    定义：
    平均并发数 L = (1/T) * ∫₀ᵀ N(t) dt
    即系统中请求数的时间平均值

    定义：
    平均响应时间 W = (1/C(T)) * Σᵢ Wᵢ
    其中 Wᵢ 是每个请求在系统中花费的时间

    关键观察：
    ∫₀ᵀ N(t) dt = Σᵢ Wᵢ
    解释：所有请求的停留时间之和 = 系统随时间积分的"请求-时间"面积

    证明：
    L = (1/T) * ∫₀ᵀ N(t) dt        [定义]
      = (1/T) * Σᵢ Wᵢ              [上述关键观察]
      = (C(T)/T) * (1/C(T)) * Σᵢ Wᵢ  [乘以 C(T)/C(T)]
      = λ * W                        [因为 λ = C(T)/T, W = (1/C(T)) * Σᵢ Wᵢ]
      = λ × W                        [得证！]

    注：这个证明假设系统是稳态的，即 λ = X。
    对于瞬态系统，Little's Law 仍然成立，但需取极限 T → ∞。
    """
```

### 5.3 在压测中的应用

```python
def littles_law_application():
    """
    Little's Law 在压测中的三个应用

    应用 1：验证压测结果的一致性
    假设压测报告显示：
    - 并发用户数 L = 100
    - 吞吐量 λ = 500 req/s
    - 平均响应时间 W = 200ms = 0.2s

    验证：L = λ × W → 100 = 500 × 0.2 = 100 ✓
    如果不等式成立，说明压测数据内部一致。

    应用 2：预测所需并发数
    目标：吞吐量 λ = 1000 req/s，目标响应时间 W = 100ms
    L = λ × W = 1000 × 0.1 = 100
    需要 100 并发用户。

    应用 3：诊断系统瓶颈
    实际测量：
    - L = 200（我们创建了 200 个用户）
    - W = 500ms = 0.5s
    计算预期吞吐量：λ = L / W = 200 / 0.5 = 400 req/s
    实际吞吐量：320 req/s

    差异 320 < 400 说明：
    - 有一部分并发用户没有产生有效请求
    - 可能是：等待 I/O、线程阻塞、数据库连接池耗尽
    """


def littles_law_validation_script():
    """
    在压测过程中实时验证 Little's Law

    使用场景：压测时监控数据一致性
    """
    # 伪代码
    locust_stats = {
        "user_count": 100,        # L
        "rps": 500,               # λ
        "avg_response_time": 0.2,  # W (秒)
    }

    L = locust_stats["user_count"]
    lam = locust_stats["rps"]
    W = locust_stats["avg_response_time"]

    expected_L = lam * W
    deviation = abs(L - expected_L) / L

    if deviation > 0.1:
        print(f"⚠️ Little's Law 偏离 {deviation*100:.1f}%")
        print(f"   预期并发数: {expected_L:.0f}")
        print(f"   实际并发数: {L}")
        print("   可能原因: 用户正在启动/退出，或系统未达稳态")
    else:
        print("✅ Little's Law 成立，数据自洽")
```text

---

## 6. 防超卖压力验证

### 6.1 验证流程

防超卖的核心验证是：压测后检查订单数是否超过库存数。

```bash
# 步骤 1：在数据库中预设库存
mysql -uroot -proot123 ticket_dev -e "
    UPDATE events SET total_stock = 100, sold_count = 0 WHERE id = 1;
"

# 步骤 2：初始化 Redis 缓存库存
redis-cli SET stock:1 100

# 步骤 3：启动 Locust 压测
locust -f locustfile.py --host=http://localhost:8000 \
    --headless -u 200 -r 50 --run-time 60s \
    --csv=reports/anti_overselling

# 步骤 4：压测后验证
echo "=== 总订单数 ==="
docker exec -it ticket-mysql mysql -uroot -proot123 ticket_dev -e "
    SELECT COUNT(*) AS order_count FROM orders WHERE event_id = 1;
"

echo "=== 已支付订单数 ==="
docker exec -it ticket-mysql mysql -uroot -proot123 ticket_dev -e "
    SELECT COUNT(*) AS paid_count FROM orders WHERE event_id = 1 AND status = 'paid';
"

echo "=== 售出数量 ==="
docker exec -it ticket-mysql mysql -uroot -proot123 ticket_dev -e "
    SELECT sold_count FROM events WHERE id = 1;
"

echo "=== 库存剩余（Redis）==="
redis-cli GET stock:1
```

### 6.2 验证脚本

```python
#!/usr/bin/env python3
"""
抗超卖压测验证脚本

用法：
  python verify_anti_overselling.py --event-id 1 --max-orders 100

功能：
  1. 预设库存
  2. 启动 Locust 压测
  3. 压测后检查订单总量 <= 库存量
  4. 生成验证报告
"""

import subprocess
import argparse
import time
import json
import sys
from datetime import datetime


class AntiOversellingVerifier:
    """防超卖验证器"""

    def __init__(self, event_id: int, stock: int, host: str):
        self.event_id = event_id
        self.stock = stock
        self.host = host
        self.start_time = None
        self.end_time = None

    def reset_stock(self):
        """重置库存"""
        print(f"[{datetime.now()}] 重置活动 {self.event_id} 库存为 {self.stock}...")
        # MySQL 更新
        subprocess.run([
            "docker", "exec", "ticket-mysql", "mysql",
            "-uroot", "-proot123", "ticket_dev",
            "-e", f"UPDATE events SET total_stock={self.stock}, sold_count=0 WHERE id={self.event_id}",
        ], capture_output=True)
        # Redis 更新
        subprocess.run([
            "redis-cli", "SET", f"stock:{self.event_id}", str(self.stock),
        ], capture_output=True)
        print(f"[{datetime.now()}] 库存重置完成")

    def run_locust(self, users: int, spawn_rate: int, run_time: str):
        """运行 Locust 压测"""
        print(f"[{datetime.now()}] 启动 Locust 压测: {users} users, {spawn_rate}/s, {run_time}")
        self.start_time = time.time()

        result = subprocess.run([
            "locust",
            "-f", "locustfile.py",
            f"--host={self.host}",
            "--headless",
            "-u", str(users),
            "-r", str(spawn_rate),
            "--run-time", run_time,
            "--csv", f"reports/verify_event_{self.event_id}",
        ], capture_output=True, text=True)

        self.end_time = time.time()
        print(f"[{datetime.now()}] Locust 压测完成")
        print(result.stdout[-500:] if len(result.stdout) > 500 else result.stdout)

    def verify_orders(self) -> dict:
        """验证订单数"""
        print(f"[{datetime.now()}] 验证订单...")

        # 查询订单总数
        result = subprocess.run([
            "docker", "exec", "ticket-mysql", "mysql",
            "-uroot", "-proot123", "ticket_dev", "-N",
            "-e", f"SELECT COUNT(*) FROM orders WHERE event_id={self.event_id}",
        ], capture_output=True, text=True)

        order_count = int(result.stdout.strip())

        # 查询已支付订单
        result = subprocess.run([
            "docker", "exec", "ticket-mysql", "mysql",
            "-uroot", "-proot123", "ticket_dev", "-N",
            "-e", f"SELECT COUNT(*) FROM orders WHERE event_id={self.event_id} AND status='paid'",
        ], capture_output=True, text=True)

        paid_count = int(result.stdout.strip()) if result.stdout.strip() else 0

        # 查询售出数量
        result = subprocess.run([
            "docker", "exec", "ticket-mysql", "mysql",
            "-uroot", "-proot123", "ticket_dev", "-N",
            "-e", f"SELECT sold_count FROM events WHERE id={self.event_id}",
        ], capture_output=True, text=True)

        sold_count = int(result.stdout.strip())

        return {
            "event_id": self.event_id,
            "stock": self.stock,
            "total_orders": order_count,
            "paid_orders": paid_count,
            "sold_count": sold_count,
            "oversold": order_count > self.stock,
        }

    def generate_report(self, result: dict):
        """生成验证报告"""
        print("\n" + "=" * 60)
        print("  防超卖压力测试验证报告")
        print("=" * 60)
        print(f"   活动 ID:          {result['event_id']}")
        print(f"   预设库存:          {result['stock']}")
        print(f"   总订单数:          {result['total_orders']}")
        print(f"   已支付订单数:      {result['paid_orders']}")
        print(f"   数据库已售数量:    {result['sold_count']}")
        print(f"   是否超卖:          {'❌ 超卖!' if result['oversold'] else '✅ 未超卖'}")
        print(f"   压测耗时:          {self.end_time - self.start_time:.1f}s")
        print("=" * 60)

        if result['oversold']:
            print("\n⚠️  严重警告：系统存在超卖漏洞！")
            sys.exit(1)
        else:
            print("\n✅  防超卖系统工作正常")

    def run(self, users: int, spawn_rate: int, run_time: str):
        """完整验证流程"""
        self.reset_stock()
        self.run_locust(users, spawn_rate, run_time)
        result = self.verify_orders()
        self.generate_report(result)


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="防超卖压测验证")
    parser.add_argument("--event-id", type=int, default=1, help="活动 ID")
    parser.add_argument("--stock", type=int, default=100, help="预设库存")
    parser.add_argument("--host", default="http://localhost:8000", help="目标服务器")
    parser.add_argument("--users", type=int, default=200, help="并发用户数")
    parser.add_argument("--spawn-rate", type=int, default=50, help="生成速率")
    parser.add_argument("--run-time", default="60s", help="运行时间")
    args = parser.parse_args()

    verifier = AntiOversellingVerifier(args.event_id, args.stock, args.host)
    verifier.run(args.users, args.spawn_rate, args.run_time)
```text

### 6.3 验证标准

```python
def validation_criteria():
    """
    防超卖验证标准

    必要条件（必须满足）：
    1. total_orders <= total_stock
       所有（有效）订单总数 <= 预设库存

    2. paid_orders <= total_stock
       已支付订单数 <= 预设库存

    3. sold_count <= total_stock
       数据库已售数量 <= 预设库存

    4. Redis 库存 >= 0
       压测后 Redis 库存不应为负数

    理想条件（最好满足）：
    5. total_orders ≈ sold_count
       订单数和已售数量基本一致
       （可能差少量 pending_payment 的订单）

    6. paid_orders <= sold_count
       支付数不应该超过已售数
       （支付前必须先扣减库存）

    性能指标：
    7. P99 下单响应时间 < 500ms
    8. 下单接口错误率 < 1%
    9. 库存扣减 100% 正确
    """
```

### 6.4 常见超卖问题排查

```python
def overselling_debug_guide():
    """
    超卖问题排查指南

    场景 1：MySQL 中 orders 数量 > total_stock
    → Redis 扣减和 MySQL 扣减不一致
    → 检查事务是否有遗漏的 rollback
    → 检查 InventoryService.deduct_stock 和 restore_stock 的配对

    场景 2：同一用户下了超过限购数量
    → Redis 限购脚本失效
    → 检查 Lua 脚本中的 INCRBY + EXPIRE
    → 检查是否忘记设置 max_per_user

    场景 3：Redis 库存为 0 但 MySQL sold_count < total_stock
    → 有重复归还
    → 检查 cancel_order 是否多次调用 restore_stock
    → 检查支付超时和手动取消是否冲突

    场景 4：发现负库存
    → 扣减操作不是原子的
    → 检查是否用了 GET + SET 而不是 DECRBY 或 Lua 脚本
    → 检查 Lua 脚本的原子性条件
    """
```text

---

## 7. 压力测试结果分析

### 7.1 性能基准

| 指标 | 目标值 | 警告值 | 严重值 |
| ------ | -------- | -------- | -------- |
| P50 响应时间 | < 100ms | > 200ms | > 500ms |
| P99 响应时间 | < 300ms | > 500ms | > 1s |
| RPS | > 500 | < 300 | < 100 |
| 错误率 | < 0.1% | > 0.5% | > 1% |
| CPU 使用率 | < 70% | > 80% | > 90% |
| 内存使用率 | < 70% | > 80% | > 90% |

### 7.2 压测报告模板

```markdown
# 压力测试报告

## 测试信息
- 日期：2026-05-05
- 测试工具：Locust
- 目标系统：票务系统 (localhost:8000)
- 测试场景：三场景混合（浏览:抢票:最后一票 = 5:3:1）

## 测试参数
- 并发用户数：200
- 用户生成速率：20/s
- 运行时间：5 分钟

## 结果摘要

| 指标 | 值 |
| ------ | ----- |
| 总请求数 | 45,000 |
| 平均 RPS | 150 |
| 平均响应时间 | 45ms |
| P50 响应时间 | 32ms |
| P95 响应时间 | 120ms |
| P99 响应时间 | 280ms |
| 错误率 | 0.5% |
| 总成功数 | 44,775 |
| 总失败数 | 225 |

## 每个接口的统计

| 接口 | 请求数 | 平均(ms) | P50 | P99 | RPS |
| ------ | -------- | --------- | ----- | ------ | ------ |
| GET /events/ | 25,000 | 15 | 12 | 80 | 83.3 |
| GET /events/{id} | 8,000 | 18 | 15 | 90 | 26.7 |
| POST /orders/ | 12,000 | 95 | 85 | 280 | 40.0 |

## 防超卖验证结果
- 预设库存：100
- 总订单数：98
- 已支付订单：95
- 超卖：否 ✅
```

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `locustfile.py` | Locust 压测脚本：三场景（浏览/抢票/最后一票）、概率权重配置 |

**运行压测命令：**

```bash
# 启动 Web UI 模式
locust -f locustfile.py --host=http://localhost:8000

# 启动 Headless 模式（100 用户，10 分钟）
locust -f locustfile.py --host=http://localhost:8000 \
  --headless -u 100 -r 10 --run-time 10m --csv=report
```text

## 8. 本章总结

**已掌握的知识：**

- **Locust 基础**：HttpUser、@task、wait_time、TaskSet、on_start
- **Web UI**：RPS 图表、响应时间图表、百分位、错误率、用户数
- **Headless 模式**：命令行参数、CSV 报告、分布式压测
- **三种压测场景**：浏览密集（weight=5）、抢票高峰（weight=3）、最后一票（weight=1）
- **Little's Law**：L = λW 的数学证明、压测验证、瓶颈诊断
- **防超卖验证**：完整验证脚本、压测后数据库检查、问题排查

**项目里程碑：**

- [ ] locustfile.py 三场景压测文件
- [ ] 浏览密集场景：500 RPS 以上无错误
- [ ] 抢票高峰场景：Redis 库存原子扣减正确
- [ ] 最后一票场景：100 人抢 1 张不超卖
- [ ] 防超卖验证脚本
- [ ] 压测报告生成
- [ ] 订单数 <= 库存数（核心验证）

**最终验证命令：**

```bash
# 一键压测验证
python verify_anti_overselling.py --event-id 1 --stock 100 --users 200 --run-time 60s
```
