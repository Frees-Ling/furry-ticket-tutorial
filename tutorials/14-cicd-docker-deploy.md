# 第14章：CI/CD + Docker 多阶段构建 + Windows 部署

## 本章学习目标

本章将从零开始构建一整套 CI/CD 流水线、Docker 多阶段构建和 Windows 生产部署方案。完成本章后，你将掌握：

1. GitHub Actions 工作流的完整语法和所有触发器
2. Docker 多阶段构建优化镜像体积的原理与实践
3. Docker Compose 编排多服务生产环境
4. Windows 环境下使用 nssm 将应用注册为系统服务
5. Shell/PowerShell 自动化部署脚本编写

**前置知识：** 基本的 Git 操作、Linux 命令行基础、HTTP 协议基础知识。

---

## 1. GitHub Actions 完全教程

### 1.1 什么是 GitHub Actions

GitHub Actions 是 GitHub 内置的 CI/CD（持续集成/持续部署）平台。它允许你在代码仓库中定义自动化工作流，在代码推送、PR 创建、发布等事件触发时自动执行测试、构建、部署等任务。

**核心概念：**

| 概念 | 说明 | 类比 |
| ------ | ------ | ------ |
| Workflow（工作流） | 一个完整的自动化流程，由一个或多个 Job 组成 | 工厂的流水线 |
| Job（作业） | 工作流中的一个任务单元，在同一个 Runner 中执行 | 流水线上的工位 |
| Step（步骤） | Job 中的最小执行单元，可以是一个命令或 Action | 工位上的一个操作 |
| Action（动作） | 可复用的步骤单元，可以从 Marketplace 下载 | 一个预制的工具 |
| Runner（运行器） | 执行工作流的虚拟机或容器 | 工人 |
| Event（事件） | 触发工作流的条件 | 启动流水线的按钮 |

### 1.2 Workflow 基础语法

#### name — 工作流名称

```yaml
name: CI Pipeline  # 显示在 GitHub 仓库的 Actions 标签页
```text

名称是可选的，如果省略，GitHub 会使用工作流文件的路径作为名称。

#### on — 触发器

`on` 字段定义了什么事件会触发这个工作流。GitHub Actions 支持数十种事件，以下是最常用的：

**push — 代码推送时触发：**

```yaml
on:
  push:
    branches: [main, develop]    # 只监听特定分支
    paths:                       # 只监听特定文件变更
      - 'app/**'
      - 'tests/**'
      - 'requirements.txt'
    paths-ignore:                # 忽略特定文件变更
      - 'docs/**'
      - '*.md'
      - 'tutorials/**'
```

**pull_request — PR 时触发：**

```yaml
on:
  pull_request:
    branches: [main]             # 只监听目标为 main 的 PR
    types: [opened, synchronize, reopened]  # 默认包含这三种类型
```text

`types` 参数可选的值包括：`opened`（新建 PR）、`synchronize`（向 PR 推送新 commit）、`reopened`（重新打开 PR）、`closed`（关闭/合并 PR）。

**schedule — 定时触发（cron 表达式）：**

```yaml
on:
  schedule:
    # ┌─── 分钟 (0-59)
    # │ ┌── 小时 (0-23)
    # │ │ ┌─ 日期 (1-31)
    # │ │ │ ┌─ 月份 (1-12)
    # │ │ │ │ ┌─ 星期 (0-6, 0=周日)
    - cron: '0 2 * * *'          # 每天 UTC 2:00 执行
    - cron: '*/15 * * * *'       # 每 15 分钟执行一次
```

**release — 发布时触发：**

```yaml
on:
  release:
    types: [published, created, edited]
```text

**workflow_dispatch — 手动触发：**

```yaml
on:
  workflow_dispatch:             # 在 GitHub Actions 页面可以手动点击运行
    inputs:                      # 定义手动触发时可以传入的参数
      environment:
        description: '部署环境'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```

**多触发器组合：**

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'         # 每周一 UTC 6:00
  workflow_dispatch:
    inputs:
      debug_enabled:
        description: '启用调试日志'
        required: false
        default: false
        type: boolean
```text

#### jobs — 作业定义

```yaml
jobs:
  test:                          # Job ID，在同一工作流中唯一
    name: 运行测试               # 显示名称（可选）
    runs-on: ubuntu-latest       # 指定 Runner 操作系统
    timeout-minutes: 30          # 超时时间（默认 360 分钟）
    if: github.event_name != 'pull_request' || !github.event.pull_request.draft  # 条件执行

    services:                    # 服务容器（见 1.5 节）
      mysql:
        image: mysql:8.0
        # ...

    steps:                       # 步骤列表
      - name: Checkout 代码
        uses: actions/checkout@v4
      - name: 设置 Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: 安装依赖
        run: pip install -r requirements.txt
```

### 1.3 Steps 详解

每个 Job 由多个 Step 组成，Step 有三种类型：

**类型一：`uses` — 调用预定义的 Action**

```yaml
steps:
  - uses: actions/checkout@v4     # 将仓库代码签出到 Runner
  - uses: actions/setup-python@v5  # 安装指定版本的 Python
    with:
      python-version: "3.12"
      cache: 'pip'                 # 自动缓存 pip 依赖
  - uses: docker/login-action@v3   # 登录 Docker Registry
    with:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```text

**类型二：`run` — 直接执行命令**

```yaml
steps:
  - name: 安装项目依赖
    run: |
      pip install --upgrade pip
      pip install -r requirements.txt
      pip install -r requirements-dev.txt

  - name: 运行测试
    run: |
      pytest tests/ \
        --cov=app \
        --cov-report=term-missing \
        --cov-fail-under=80 \
        -v
    env:                          # 步骤级别的环境变量
      DATABASE_URL: mysql+pymysql://root:root123@127.0.0.1:3306/ticket_test
```

**类型三：`uses` + `run` 混用**

一个 Step 不能同时包含 `uses` 和 `run`，但一个 Job 中可以交替使用。

### 1.4 上下文对象 (Contexts)

GitHub Actions 提供了一系列上下文对象，用于在工作流中获取运行时信息：

| 上下文 | 类型 | 说明 |
| -------- | ------ | ------ |
| `github` | 对象 | 包含触发工作流的事件信息 |
| `env` | 对象 | 环境变量 |
| `job` | 对象 | 当前运行的 Job 信息 |
| `steps` | 对象 | 各步骤的输出 |
| `runner` | 对象 | Runner 本身的信息 |
| `secrets` | 对象 | 加密的机密信息 |
| `vars` | 对象 | 非加密的配置变量（GitHub 新功能） |
| `matrix` | 对象 | Matrix 策略中的当前参数 |

**常用示例：**

```yaml
- name: 打印上下文信息
  run: |
    echo "事件名称: ${{ github.event_name }}"
    echo "仓库: ${{ github.repository }}"
    echo "分支: ${{ github.ref_name }}"
    echo "Commit SHA: ${{ github.sha }}"
    echo "触发者: ${{ github.actor }}"
    echo "Runner OS: ${{ runner.os }}"
    echo "Runner 架构: ${{ runner.arch }}"
    echo "Job 状态: ${{ job.status }}"
    echo "Job ID: ${{ job.container.id }}"
```text

**条件执行中如何使用上下文：**

```yaml
- name: 仅在 main 分支上部署
  if: github.ref_name == 'main' && github.event_name == 'push'
  run: echo "部署到生产环境"

- name: 是否 PR 来自 Fork
  if: github.event.pull_request.head.repo.fork
  run: echo "这是一个 Fork 的 PR，需要额外审核"

- name: 使用步骤输出
  id: build
  run: echo "version=v1.0.0" >> $GITHUB_OUTPUT

- name: 使用上一步的输出
  run: echo "版本号: ${{ steps.build.outputs.version }}"
```

### 1.5 服务容器 (Service Containers)

GitHub Actions 支持在 Job 中启动额外的 Docker 容器作为服务，非常适合集成测试场景。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      # --- MySQL 8.0 服务容器 ---
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root123
          MYSQL_DATABASE: ticket_test
          MYSQL_CHARACTER_SET_SERVER: utf8mb4
          MYSQL_COLLATION_SERVER: utf8mb4_unicode_ci
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h localhost"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        # 容器健康检查：GitHub Actions 会等待 health 通过后才执行 steps

      # --- Redis 7 服务容器 ---
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v4

      - name: 等待 MySQL 就绪
        run: |
          for i in {1..30}; do
            if mysqladmin ping -h 127.0.0.1 -u root -proot123 2>/dev/null; then
              echo "MySQL 已就绪"
              break
            fi
            sleep 2
          done

      - name: 等待 Redis 就绪
        run: |
          for i in {1..10}; do
            if redis-cli -h 127.0.0.1 ping 2>/dev/null; then
              echo "Redis 已就绪"
              break
            fi
            sleep 1
          done

      - name: 运行数据库迁移
        run: alembic upgrade head
        env:
          DB_HOST: 127.0.0.1
          DB_USER: root
          DB_PASSWORD: root123
          DB_NAME: ticket_test

      - name: 运行测试
        run: pytest tests/ -v --cov=app --cov-fail-under=80
        env:
          DB_HOST: 127.0.0.1
          DB_USER: root
          DB_PASSWORD: root123
          DB_NAME: ticket_test
          REDIS_HOST: 127.0.0.1
```text

**健康检查机制详解：**

GitHub Actions 的服务容器使用 `options` 中的 `--health-cmd` 和 `--health-interval` 参数。Runner 会自动轮询容器的健康状态，在容器变为 healthy 之前不会开始执行 `steps`。这确保了数据库等依赖在测试开始前已经就绪。

### 1.6 Matrix 策略

Matrix 策略允许你在多个操作系统、语言版本或环境配置中并行运行 Job：

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ["3.11", "3.12"]
        include:                  # 添加额外的组合
          - os: ubuntu-latest
            python-version: "3.10"
        exclude:                  # 排除特定组合
          - os: windows-latest
            python-version: "3.11"
      fail-fast: false            # 一个组合失败后是否停止其他组合
      max-parallel: 4             # 最大并行数

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: 运行测试
        run: pytest tests/ -v
```

上面的配置会生成以下并行 Job：

| # | OS | Python |
| --- | ---- | -------- |
| 1 | ubuntu-latest | 3.11 |
| 2 | ubuntu-latest | 3.12 |
| 3 | ubuntu-latest | 3.10（通过 include 添加） |
| 4 | windows-latest | 3.12 |
| 5 | macos-latest | 3.11 |
| 6 | macos-latest | 3.12 |

**注意：** `exclude` 移除了 `(windows-latest, 3.11)` 这个组合。

### 1.7 Cache 缓存加速

使用 `actions/cache` 缓存 Python 依赖、Docker 层等，显著加速 CI：

```yaml
- name: 缓存 pip 依赖
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip          # 要缓存的目录
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-

- name: 安装依赖
  run: pip install -r requirements.txt

- name: 缓存 Docker 镜像层
  uses: actions/cache@v4
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
```text

### 1.8 Secrets 和环境变量

**仓库级别 Secrets：**

在 GitHub 仓库的 Settings > Secrets and variables > Actions 中设置。在工作流中通过 `${{ secrets.XXX }}` 引用。

**环境变量设置方式：**

```yaml
# 工作流级别
env:
  GLOBAL_VAR: "全局变量"

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production       # 引用 GitHub Environment

    # Job 级别
    env:
      JOB_VAR: "作业变量"
      DB_HOST: ${{ vars.PROD_DB_HOST }}  # 使用非加密变量

    steps:
      - name: 设置步骤级别环境变量
        env:
          STEP_VAR: "步骤变量"
        run: echo $STEP_VAR

# 使用 GitHub Environments 管理不同环境
```

**推荐的 Secrets 清单：**

| Secret 名称 | 说明 |
| ------------ | ------ |
| `DOCKER_USERNAME` | Docker Hub 用户名 |
| `DOCKER_PASSWORD` | Docker Hub 密码/Token |
| `DEPLOY_HOST` | 部署服务器 IP |
| `DEPLOY_USER` | SSH 用户 |
| `DEPLOY_KEY` | SSH 私钥 |
| `ALIPAY_APP_ID` | 支付宝应用 ID |
| `ALIPAY_PRIVATE_KEY` | 支付宝应用私钥 |

### 1.9 完整示例：test.yml

```yaml
# .github/workflows/test.yml
# 说明：代码推送到主分支或创建 PR 时自动运行测试

name: Test Suite

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - 'tutorials/**'
      - '*.md'
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

jobs:
  lint:
    name: 代码检查
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: 'pip'
      - name: 安装依赖
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install flake8 black
      - name: 运行 flake8
        run: flake8 app/ tests/ --max-line-length=100
      - name: 检查代码格式
        run: black --check app/ tests/

  test:
    name: 运行测试 (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    needs: lint                    # 依赖 lint Job 成功
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
      fail-fast: true

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root123
          MYSQL_DATABASE: ticket_test
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 3306:3306

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - name: 安装依赖
        run: pip install -r requirements.txt

      - name: 运行数据库迁移
        run: alembic upgrade head
        env:
          DB_HOST: 127.0.0.1
          DB_USER: root
          DB_PASSWORD: root123
          DB_NAME: ticket_test

      - name: 运行测试
        run: |
          pytest tests/ \
            --cov=app \
            --cov-report=xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -v
        env:
          DB_HOST: 127.0.0.1
          DB_USER: root
          DB_PASSWORD: root123
          DB_NAME: ticket_test
          REDIS_HOST: 127.0.0.1

      - name: 上传覆盖率报告
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-${{ matrix.python-version }}
          path: coverage.xml
```text

### 1.10 完整示例：deploy.yml

```yaml
# .github/workflows/deploy.yml
# 说明：发布 Release 或手动触发时构建 Docker 镜像并部署

name: Deploy

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      environment:
        description: '部署环境'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:
  REGISTRY: docker.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    name: 构建并推送 Docker 镜像
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout 代码
        uses: actions/checkout@v4

      - name: 设置 Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: 登录 Docker Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: 提取 Docker 元数据
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,format=short
            type=raw,value=latest,enable={{is_default_branch}}

      - name: 构建并推送镜像
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: 部署到服务器
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.event_name == 'release' || github.event.inputs.environment == 'production'

    steps:
      - name: SSH 登录并部署
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_KEY }}
          script: |
            cd /opt/ticket-booking
            docker compose pull
            docker compose up -d --force-recreate
            docker image prune -f
            echo "部署完成！"
```

---

## 2. Docker 多阶段构建

### 2.1 什么是多阶段构建

Docker 多阶段构建（Multi-stage Build）允许你在一个 Dockerfile 中使用多个 `FROM` 语句，每个 `FROM` 开始一个新的构建阶段。你可以从之前的阶段复制文件，最终只保留需要的产物。

**核心优势：**

1. **镜像体积大幅减小**：builder 阶段使用完整工具链，最终镜像只包含运行时依赖
2. **构建缓存复用**：不变的依赖层可以被缓存
3. **安全性提升**：最终镜像不包含编译工具、源码等
4. **构建过程更清晰**：不同职责分离到不同阶段

### 2.2 基础 Dockerfile

```dockerfile
# Dockerfile
# 第一阶段：builder — 安装所有依赖和工具
FROM python:3.12-slim AS builder

# 设置工作目录
WORKDIR /app

# 设置环境变量
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# 安装系统编译依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gcc \
    libc6-dev \
    && rm -rf /var/lib/apt/lists/*

# 先复制 requirements.txt，利用 Docker 缓存层
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 第二阶段：runtime — 最小化运行时镜像
FROM python:3.12-slim

# 设置工作目录
WORKDIR /app

# 设置环境变量
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    TZ=Asia/Shanghai

# 仅从 builder 阶段复制已安装的 Python 包
COPY --from=builder /usr/local/lib/python3.12/site-packages \
                    /usr/local/lib/python3.12/site-packages

# 复制应用代码
COPY . .

# 创建非 root 用户运行
RUN groupadd -r ticket && useradd -r -g ticket ticket && \
    chown -R ticket:ticket /app

USER ticket

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# 启动命令
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```text

### 2.3 多阶段构建原理深度解析

**Docker 镜像层结构：**

每个 `RUN`、`COPY`、`ADD` 指令都会创建一个新的镜像层。每一层只存储与上一层的差异。多阶段构建的关键优化是：

```

传统单阶段构建：
  base image (120MB)

+ apt packages (200MB)
+ pip packages (200MB)
+ app code (5MB)
  = 最终镜像约 525MB

多阶段构建：
  阶段1 (builder)：
    base image (120MB)
    + apt packages (200MB)     ← 临时存在，最终丢弃
    + pip packages (200MB)     ← 复制到最终阶段
    = builder 临时镜像约 520MB
  
  阶段2 (runtime)：
    base image (120MB)
    + site-packages (200MB)     ← 从 builder 复制
    + app code (5MB)
    = 最终镜像约 325MB

  节省约 200MB（apt 编译工具 + 构建缓存）

```text

**`COPY --from=builder` 详解：**

`COPY --from=<阶段名> <源路径> <目标路径>` 可以从指定的构建阶段复制文件或目录。关键优化点是只复制 `site-packages` 目录，而不复制 pip 缓存和编译中间产物。

```dockerfile
# 不推荐：复制整个 usr/local（包含缓存和不需要的文件）
COPY --from=builder /usr/local /usr/local

# 推荐：只复制 site-packages
COPY --from=builder /usr/local/lib/python3.12/site-packages \
                    /usr/local/lib/python3.12/site-packages

# 如果还需要其他工具（如 Alembic 的 bin 脚本）
COPY --from=builder /usr/local/bin/alembic /usr/local/bin/alembic
```

### 2.4 .dockerignore 文件

`.dockerignore` 的作用类似于 `.gitignore`，告诉 Docker 构建上下文中的哪些文件和目录不发送给 Docker daemon，从而加速构建并减少镜像中的敏感信息。

```dockerignore
# .dockerignore

# 版本控制
.git/
.gitignore

# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/
env/

# 本地环境文件
.env
.env.*
!.env.example

# IDE
.idea/
.vscode/
*.swp

# 文档和教程
tutorials/
docs/
*.md

# 测试
tests/
htmlcov/
.coverage
coverage.xml
.pytest_cache/

# Docker
.dockerignore
Dockerfile

# 日志
*.log

# 系统文件
.DS_Store
Thumbs.db
```text

**`.dockerignore` 的重要性：**

在没有 `.dockerignore` 的情况下，`docker build .` 会将整个项目目录发送给 Docker daemon。如果你的项目根目录有 `node_modules/`、`__pycache__/` 等大目录，构建上下文可能达到几百 MB，导致：

1. 构建启动极慢（传输上下文）
2. 频繁触发缓存失效（上下文哈希变化）
3. 无意中包含 `.env` 等敏感文件

**验证构建上下文大小：**

```bash
# 查看构建上下文大小（在 .dockerignore 前后对比）
docker build -t test --no-cache . 2>&1 | grep "context"
```

### 2.5 构建和运行命令

```bash
# === 基础构建命令 ===

# 构建镜像
docker build -t ticket-booking:latest .

# 指定 Dockerfile 路径
docker build -f Dockerfile -t ticket-booking:v1.0.0 .

# 带标签构建
docker build -t ticket-booking:latest -t ticket-booking:v1.0.0 .

# === 运行容器 ===

# 基本运行（适用于测试）
docker run -d \
  --name ticket-app \
  -p 8000:8000 \
  -e DB_HOST=host.docker.internal \
  -e REDIS_HOST=host.docker.internal \
  ticket-booking:latest

# 带环境变量文件
docker run -d \
  --name ticket-app \
  -p 8000:8000 \
  --env-file .env.prod \
  --restart unless-stopped \
  ticket-booking:latest

# === 镜像管理 ===

# 列出镜像
docker images

# 查看镜像大小
docker image ls ticket-booking

# 查看镜像构建历史
docker history ticket-booking:latest

# 删除镜像
docker rmi ticket-booking:latest

# 导出/导入镜像（离线部署）
docker save ticket-booking:latest | gzip > ticket-booking.tar.gz
gunzip -c ticket-booking.tar.gz | docker load
```text

### 2.6 镜像体积优化技巧

```dockerfile
# === 技巧1：使用 --no-cache-dir 避免缓存 pip 下载的包 ===
RUN pip install --no-cache-dir -r requirements.txt

# === 技巧2：合并 RUN 指令减少层数 ===
# 不推荐（3 层）：
RUN apt-get update
RUN apt-get install -y gcc
RUN rm -rf /var/lib/apt/lists/*

# 推荐（1 层）：
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# === 技巧3：仅安装运行时必需的系统包 ===
# 完整版：
RUN apt-get install -y gcc libc6-dev libssl-dev
# 最小化（仅编译时需要的留在 builder 阶段）：
# runtime 阶段无需任何 apt 包

# === 技巧4：COPY 顺序优化缓存 ===
# 不经常变化的内容先 COPY（pip install 可以缓存）
COPY requirements.txt .
RUN pip install -r requirements.txt

# 经常变化的内容后 COPY
COPY . .

# === 技巧5：使用 Alpine 还是 Slim ===
# Alpine（极小，但有兼容性问题）：
FROM python:3.12-alpine  # ~50MB base

# Slim（Debian 基础，兼容性好）：
FROM python:3.12-slim    # ~120MB base

# 推荐使用 slim，避免 musl libc 兼容性问题
```

---

## 3. Docker Compose

### 3.1 docker-compose.yml 解释

项目根目录已有的 `docker-compose.yml`：

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    container_name: ticket-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: ticket_dev
      MYSQL_CHARACTER_SET_SERVER: utf8mb4
      MYSQL_COLLATION_SERVER: utf8mb4_unicode_ci
    volumes:
      - mysql_data:/var/lib/mysql        # 数据持久化
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: ticket-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data                  # 数据持久化
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
  redis_data:
```text

**Compose 字段详解：**

| 字段 | 说明 |
| ------ | ------ |
| `version` | Docker Compose 文件格式版本，3.8 是稳定的最新版 |
| `services` | 定义要运行的服务（容器） |
| `image` | 使用的 Docker 镜像 |
| `container_name` | 容器名称（默认是 `项目_服务_序号`） |
| `restart` | 重启策略：`no`, `always`, `on-failure`, `unless-stopped` |
| `ports` | 端口映射 `主机端口:容器端口` |
| `environment` | 环境变量 |
| `volumes` | 卷挂载，持久化数据 |
| `healthcheck` | 健康检查配置 |
| `depends_on` | 依赖关系，控制启动顺序 |

### 3.2 docker-compose.prod.yml（生产环境）

生产环境需要包含应用服务本身，并且使用非 root 用户、限制资源、配置日志轮转等最佳实践：

```yaml
# docker-compose.prod.yml
# 说明：生产环境 Docker Compose 配置

version: "3.8"

services:
  mysql:
    image: mysql:8.0
    container_name: ticket-mysql
    restart: unless-stopped
    ports:
      - "127.0.0.1:3306:3306"            # 仅监听本地回环
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_CHARACTER_SET_SERVER: utf8mb4
      MYSQL_COLLATION_SERVER: utf8mb4_unicode_ci
    volumes:
      - mysql_data:/var/lib/mysql
      - ./sql/init:/docker-entrypoint-initdb.d  # 初始化脚本
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:                                   # 资源限制
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
    networks:
      - backend

  redis:
    image: redis:7-alpine
    container_name: ticket-redis
    restart: unless-stopped
    ports:
      - "127.0.0.1:6379:6379"            # 仅监听本地回环
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro  # 自定义配置
    command: redis-server /usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
    networks:
      - backend

  app:
    image: ${REGISTRY:-docker.io}/ticket-booking:${TAG:-latest}
    container_name: ticket-app
    restart: unless-stopped
    ports:
      - "0.0.0.0:8000:8000"
    environment:
      ENV: production
      DEBUG: "false"
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: ${DB_NAME}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      SECRET_KEY: ${SECRET_KEY}
      ALIPAY_APP_ID: ${ALIPAY_APP_ID}
      ALIPAY_APP_PRIVATE_KEY_PATH: /app/keys/app_private_key.pem
      ALIPAY_PUBLIC_KEY_PATH: /app/keys/alipay_public_key.pem
      ALIPAY_NOTIFY_URL: ${ALIPAY_NOTIFY_URL}
      ALIPAY_RETURN_URL: ${ALIPAY_RETURN_URL}
    volumes:
      - ./keys:/app/keys:ro                # 挂载密钥文件
      - ./logs:/app/logs                   # 日志持久化
    depends_on:
      mysql:
        condition: service_healthy         # 等待 MySQL 健康检查通过
      redis:
        condition: service_healthy         # 等待 Redis 健康检查通过
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s                   # 启动宽限期
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - backend
      - frontend                          # 如果需要 Nginx 反向代理

  nginx:
    image: nginx:1.25-alpine
    container_name: ticket-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro           # SSL 证书
      - ./logs/nginx:/var/log/nginx
    depends_on:
      app:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.5'
    networks:
      - frontend

networks:
  backend:
    driver: bridge
    internal: true                             # 后端网络不允许外部访问
  frontend:
    driver: bridge

volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local
```

**生产环境配置的 10 个最佳实践：**

| 实践 | 说明 |
| ------ | ------ |
| 1. `depends_on` + `condition: service_healthy` | 确保依赖服务健康后才启动 |
| 2. `127.0.0.1:3306:3306` | 数据库端口仅监听回环地址 |
| 3. `deploy.resources.limits` | 限制每个容器的 CPU 和内存 |
| 4. `logging` 配置 | 日志轮转，防止磁盘写满 |
| 5. 独立 `networks` | backend 设为 internal 隔离数据库 |
| 6. `.env` 文件管理环境变量 | 敏感信息不硬编码 |
| 7. 只读卷挂载 (`:ro`) | 配置文件不可从容器内修改 |
| 8. `healthcheck` + `start_period` | 健康检查和启动宽限期 |
| 9. 密钥文件挂载而非环境变量 | 私钥通过文件系统挂载更安全 |
| 10. `restart: unless-stopped` | 崩溃自动重启 |

### 3.3 生产环境启动命令

```bash
# 确保 .env 文件存在
cat > .env << 'EOF'
DB_USER=root
DB_PASSWORD=your_secure_password_here
DB_NAME=ticket_prod
SECRET_KEY=your-256-bit-long-secret-key-here
ALIPAY_APP_ID=your_app_id
ALIPAY_NOTIFY_URL=https://your-domain.com/api/v1/payments/alipay/notify
ALIPAY_RETURN_URL=https://your-domain.com/orders
REGISTRY=docker.io
TAG=latest
EOF

# 启动所有服务
docker compose -f docker-compose.prod.yml --env-file .env up -d

# 查看日志
docker compose -f docker-compose.prod.yml logs -f

# 查看单个服务日志
docker compose -f docker-compose.prod.yml logs -f app

# 重新构建并启动
docker compose -f docker-compose.prod.yml up -d --build --force-recreate

# 停止服务
docker compose -f docker-compose.prod.yml down

# 停止并删除卷（谨慎！会丢失数据）
docker compose -f docker-compose.prod.yml down -v

# 滚动更新（生产推荐）
docker compose -f docker-compose.prod.yml pull app
docker compose -f docker-compose.prod.yml up -d --no-deps app
```text

**重要安全提醒：**

1. `.env` 文件包含敏感信息，**绝对不要提交到 Git 仓库**。确保 `.gitignore` 中包含 `.env`。
2. 生产环境密码使用 `openssl rand -base64 32` 生成强随机密码。
3. MySQL root 密码建议使用 Docker Secrets 而非环境变量。
4. 定期轮换 SECRET_KEY 和数据库密码。

---

## 4. NSSM Windows 服务部署

### 4.1 什么是 NSSM

NSSM（Non-Sucking Service Manager）是一个轻量级的 Windows 服务管理工具。它可以将任何普通的 Windows 程序注册为 Windows 系统服务，使其能够：

- 随系统自动启动
- 在后台运行（无需登录用户）
- 崩溃后自动重启
- 记录标准输出到 Windows 事件日志

**下载地址：** <https://nssm.cc/download>

### 4.2 安装 NSSM

```powershell
# 将 nssm.exe 放到 PATH 路径中（推荐 C:\Windows\System32\）
Copy-Item nssm-2.24\win64\nssm.exe C:\Windows\System32\

# 验证安装
nssm --version
# 输出: NSSM 2.24 (32-bit or 64-bit)
```

### 4.3 注册 FastAPI 为 Windows 服务

```powershell
# 安装 Python 应用为 Windows 服务
nssm install TicketAPI

# 此时会弹出 GUI 配置窗口，填写：
#   Application Tab:
#     Path: C:\Python312\python.exe
#     Arguments: -m uvicorn app.main:app --host 0.0.0.0 --port 8000
#     Startup directory: C:\ticket-booking
#
#   Details Tab:
#     Display name: 票务系统 API 服务
#     Description: 提供票务系统后端 API 服务
#     Startup type: Automatic (Delayed Start)
#
#   Log On Tab:
#     Log on as: Local System account
#
#   I/O Tab:
#     Input (stdin): C:\ticket-booking\logs\stdin.log
#     Output (stdout): C:\ticket-booking\logs\stdout.log
#     Error (stderr): C:\ticket-booking\logs\stderr.log

# 或者使用命令行（无 GUI）：
nssm install TicketAPI `
  "C:\Python312\python.exe" `
  "-m uvicorn app.main:app --host 0.0.0.0 --port 8000"

# 设置服务参数（通过注册表）
nssm set TicketAPI AppDirectory "C:\ticket-booking"
nssm set TicketAPI DisplayName "票务系统 API 服务"
nssm set TicketAPI Description "提供票务系统后端 API 服务"
nssm set TicketAPI Start SERVICE_AUTO_START
nssm set TicketAPI AppStdout "C:\ticket-booking\logs\stdout.log"
nssm set TicketAPI AppStderr "C:\ticket-booking\logs\stderr.log"
nssm set TicketAPI AppRotateFiles 1
nssm set TicketAPI AppRotateSeconds 86400
nssm set TicketAPI AppRotateBytes 10485760
```text

### 4.4 注册 Nginx 为 Windows 服务

```powershell
# 下载 Nginx Windows 版
# 解压到 C:\nginx\

# 安装为服务
nssm install TicketNginx "C:\nginx\nginx.exe"

# 设置参数
nssm set TicketNginx AppDirectory "C:\nginx"
nssm set TicketNginx DisplayName "票务系统 Nginx 反向代理"
nssm set TicketNginx Description "Nginx 反向代理和静态文件服务"
nssm set TicketNginx Start SERVICE_AUTO_START
nssm set TicketNginx AppStdout "C:\nginx\logs\stdout.log"
nssm set TicketNginx AppStderr "C:\nginx\logs\stderr.log"
```

### 4.5 服务管理命令

```powershell
# === 启动服务 ===
nssm start TicketAPI
nssm start TicketNginx

# === 停止服务 ===
nssm stop TicketAPI
nssm stop TicketNginx

# === 重启服务 ===
nssm restart TicketAPI
nssm restart TicketNginx

# === 查看服务状态 ===
nssm status TicketAPI
nssm status TicketNginx

# === 编辑服务配置（弹出 GUI） ===
nssm edit TicketAPI

# === 查看服务详细状态 ===
nssm get TicketAPI Status
nssm get TicketAPI AppDirectory
nssm get TicketAPI AppStdout

# === 暂停/继续 ===
nssm suspend TicketAPI  # 暂停（不停止进程）
nssm continue TicketAPI # 继续

# === 卸载服务 ===
nssm remove TicketAPI confirm
nssm remove TicketNginx confirm
```text

### 4.6 PowerShell 系统命令替代方案（不使用 nssm）

```powershell
# 使用 sc 命令创建服务（Windows 内置）
sc create "TicketAPI" binPath="C:\Python312\python.exe -m uvicorn app.main:app --host 0.0.0.0 --port 8000" start=auto

# 启动服务
sc start TicketAPI

# 停止服务
sc stop TicketAPI

# 删除服务
sc delete TicketAPI

# 使用 PowerShell cmdlet
New-Service -Name "TicketAPI" `
  -BinaryPathName "C:\Python312\python.exe -m uvicorn app.main:app --host 0.0.0.0 --port 8000" `
  -DisplayName "票务系统 API 服务" `
  -StartupType Automatic

Get-Service -Name TicketAPI
Start-Service -Name TicketAPI
Stop-Service -Name TicketAPI
Restart-Service -Name TicketAPI
Set-Service -Name TicketAPI -Status Running
```

---

## 5. 部署脚本

### 5.1 scripts/deploy.sh（Linux 部署脚本）

```bash
#!/bin/bash
# scripts/deploy.sh
# 说明：Linux 生产环境自动化部署脚本
# 用法：bash scripts/deploy.sh [environment]

set -euo pipefail  # 严格模式：任何错误即退出

# ============================================================
# 配置区
# ============================================================
PROJECT_DIR="/opt/ticket-booking"
GIT_REPO="git@github.com:your-org/ticket-booking.git"
BRANCH="${1:-main}"          # 默认部署 main 分支
ENVIRONMENT="${2:-production}"
LOG_FILE="/var/log/deploy.log"
DOCKER_COMPOSE_FILE="docker-compose.prod.yml"

# ============================================================
# 日志函数
# ============================================================
log() {
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[${timestamp}] $1" | tee -a "${LOG_FILE}"
}

log "===== 开始部署 ====="
log "项目目录: ${PROJECT_DIR}"
log "部署分支: ${BRANCH}"
log "部署环境: ${ENVIRONMENT}"

# ============================================================
# 步骤1：进入项目目录
# ============================================================
cd "${PROJECT_DIR}"
log "当前目录: $(pwd)"

# ============================================================
# 步骤2：拉取最新代码
# ============================================================
log "正在拉取最新代码..."
git fetch origin
git checkout "${BRANCH}"
git pull origin "${BRANCH}"
log "当前 Commit: $(git rev-parse --short HEAD)"

# ============================================================
# 步骤3：运行数据库迁移
# ============================================================
log "正在运行数据库迁移..."
source .venv/bin/activate || true
# 如果使用 Docker，直接在容器内执行迁移
if command -v docker &> /dev/null; then
    docker compose -f "${DOCKER_COMPOSE_FILE}" exec -T app alembic upgrade head
else
    alembic upgrade head
fi
log "数据库迁移完成"

# ============================================================
# 步骤4：重启服务
# ============================================================
log "正在重启服务..."

if command -v docker &> /dev/null && docker compose ls &> /dev/null; then
    # Docker 部署方式
    log "使用 Docker Compose 重启..."
    docker compose -f "${DOCKER_COMPOSE_FILE}" pull
    docker compose -f "${DOCKER_COMPOSE_FILE}" up -d --force-recreate
    docker image prune -f
elif command -v systemctl &> /dev/null; then
    # Systemd 部署方式
    log "使用 systemctl 重启..."
    systemctl restart ticket-api
    systemctl status ticket-api --no-pager
else
    # Supervisor 部署方式
    log "使用 supervisor 重启..."
    supervisorctl restart ticket-api
fi

log "服务重启完成"

# ============================================================
# 步骤5：健康检查
# ============================================================
log "正在进行健康检查..."
for i in {1..12}; do
    if curl -sf http://localhost:8000/health > /dev/null 2>&1; then
        log "健康检查通过！"
        break
    fi
    if [ "$i" -eq 12 ]; then
        log "ERROR: 健康检查失败！部署可能存在问题！"
        exit 1
    fi
    log "等待服务就绪 (${i}/12)..."
    sleep 5
done

# ============================================================
# 步骤6：检查 Nginx 配置并重载
# ============================================================
if command -v nginx &> /dev/null; then
    log "检查 Nginx 配置..."
    nginx -t && nginx -s reload
    log "Nginx 配置已重载"
fi

log "===== 部署完成 ====="
```text

### 5.2 scripts/deploy.ps1（Windows PowerShell 部署脚本）

```powershell
# scripts/deploy.ps1
# 说明：Windows 生产环境自动化部署脚本
# 用法：.\scripts\deploy.ps1 [-Branch main] [-Environment production]

param(
    [Parameter(Mandatory=$false)]
    [string]$Branch = "main",

    [Parameter(Mandatory=$false)]
    [string]$Environment = "production"
)

# ============================================================
# 配置区
# ============================================================
$ProjectDir = "C:\ticket-booking"
$LogFile = "C:\logs\deploy-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
$NssmPath = "C:\Windows\System32\nssm.exe"
$PythonPath = "C:\Python312\python.exe"

# ============================================================
# 日志函数
# ============================================================
function Write-Log {
    param([string]$Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logMessage = "[${timestamp}] ${Message}"
    Write-Host $logMessage
    Add-Content -Path $LogFile -Value $logMessage
}

Write-Log "===== 开始部署 ====="
Write-Log "项目目录: ${ProjectDir}"
Write-Log "部署分支: ${Branch}"
Write-Log "部署环境: ${Environment}"

# ============================================================
# 步骤1：进入项目目录
# ============================================================
Set-Location -Path $ProjectDir
Write-Log "当前目录: $(Get-Location)"

# ============================================================
# 步骤2：拉取最新代码
# ============================================================
Write-Log "正在拉取最新代码..."
git fetch origin
git checkout $Branch
git pull origin $Branch
$latestCommit = git rev-parse --short HEAD
Write-Log "当前 Commit: ${latestCommit}"

# ============================================================
# 步骤3：更新 Python 依赖
# ============================================================
Write-Log "正在更新 Python 依赖..."
& $PythonPath -m pip install --upgrade pip
& $PythonPath -m pip install -r requirements.txt
Write-Log "依赖更新完成"

# ============================================================
# 步骤4：运行数据库迁移
# ============================================================
Write-Log "正在运行数据库迁移..."
$env:DB_HOST = "localhost"
$env:DB_USER = "root"
$env:DB_PASSWORD = "root123"
$env:DB_NAME = "ticket_prod"
& $PythonPath -m alembic upgrade head
Write-Log "数据库迁移完成"

# ============================================================
# 步骤5：停止服务
# ============================================================
Write-Log "正在停止服务..."

# 检查 nssm 是否存在
if (Test-Path $NssmPath) {
    Write-Log "使用 nssm 管理服务..."

    # 停止 Nginx 服务（先停反向代理）
    Write-Log "停止 Nginx 服务..."
    & $NssmPath stop TicketNginx
    Start-Sleep -Seconds 2

    # 停止 API 服务
    Write-Log "停止 API 服务..."
    & $NssmPath stop TicketAPI
    Start-Sleep -Seconds 2

    # 验证服务已停止
    $apiStatus = & $NssmPath status TicketAPI
    Write-Log "API 服务状态: ${apiStatus}"
} else {
    Write-Log "nssm 未找到，使用 sc 命令..."
    sc stop TicketAPI
    sc stop TicketNginx
}

# ============================================================
# 步骤6：重启服务
# ============================================================
Write-Log "正在启动服务..."

if (Test-Path $NssmPath) {
    # 启动 API 服务（先启应用）
    Write-Log "启动 API 服务..."
    & $NssmPath start TicketAPI
    Start-Sleep -Seconds 5

    # 启动 Nginx 服务
    Write-Log "启动 Nginx 服务..."
    & $NssmPath start TicketNginx

    # 查看服务状态
    $apiStatus = & $NssmPath status TicketAPI
    $nginxStatus = & $NssmPath status TicketNginx
    Write-Log "API 服务状态: ${apiStatus}"
    Write-Log "Nginx 服务状态: ${nginxStatus}"
} else {
    sc start TicketAPI
    sc start TicketNginx
}

# ============================================================
# 步骤7：健康检查
# ============================================================
Write-Log "正在进行健康检查..."
$healthOk = $false
for ($i = 1; $i -le 12; $i++) {
    try {
        $response = Invoke-RestMethod -Uri "http://localhost:8000/health" -TimeoutSec 5
        if ($response.status -eq "ok") {
            Write-Log "健康检查通过！"
            $healthOk = $true
            break
        }
    } catch {
        # 服务未就绪
    }
    Write-Log "等待服务就绪 (${i}/12)..."
    Start-Sleep -Seconds 5
}

if (-not $healthOk) {
    Write-Log "ERROR: 健康检查失败！部署可能存在问题！"
    exit 1
}

# ============================================================
# 步骤8：清理旧的日志文件（保留最近 7 天）
# ============================================================
Write-Log "正在清理旧日志..."
$logDir = "C:\ticket-booking\logs"
$cutoffDate = (Get-Date).AddDays(-7)
Get-ChildItem -Path $logDir -Filter "*.log" | Where-Object {
    $_.LastWriteTime -lt $cutoffDate
} | Remove-Item -Force
Write-Log "日志清理完成"

Write-Log "===== 部署完成 ====="
```

### 5.3 设置脚本执行权限

```bash
# Linux：使脚本可执行
chmod +x scripts/deploy.sh

# 使用脚本
./scripts/deploy.sh main production
```text

```powershell
# Windows：启用 PowerShell 脚本执行（以管理员身份运行）
Set-ExecutionPolicy -Scope LocalMachine RemoteSigned

# 运行部署脚本
.\scripts\deploy.ps1 -Branch main -Environment production
```

---

## 6. 完整 CI/CD 流程总结

### 6.1 工作流流程图

```text
开发者推送代码
      │
      ▼
┌─────────────────────────────────────────┐
│              .github/workflows/test.yml │
│                                         │
│  Step 1: Checkout 代码                  │
│  Step 2: Setup Python                   │
│  Step 3: Install 依赖                   │
│  Step 4: 启动 MySQL 8.0 服务容器        │
│  Step 5: 启动 Redis 7 服务容器          │
│  Step 6: 运行 Alembic 迁移              │
│  Step 7: 运行 Pytest 测试               │
│  Step 8: 生成覆盖率报告                 │
└───────────────┬─────────────────────────┘
                │
                ▼ (测试通过)
┌─────────────────────────────────────────┐
│          .github/workflows/deploy.yml   │
│                                         │
│  发布 Release 时触发                    │
│                                         │
│  Step 1: Checkout 代码                  │
│  Step 2: Docker Buildx 设置             │
│  Step 3: 登录 Docker Registry           │
│  Step 4: 多阶段构建 Docker 镜像         │
│  Step 5: 推送镜像到 Registry            │
│  Step 6: SSH 登录服务器                 │
│  Step 7: docker compose pull & up       │
└─────────────────────────────────────────┘
```

### 6.2 DevOps 最佳实践清单

+ [ ] 所有 Secrets 存储在 GitHub Secrets 中，不硬编码
+ [ ] .env 文件在 .gitignore 中排除
+ [ ] Docker 镜像使用多阶段构建优化体积
+ [ ] .dockerignore 排除不必要的文件和目录
+ [ ] docker-compose.prod.yml 配置资源限制和健康检查
+ [ ] 生产环境使用 `depends_on: condition: service_healthy`
+ [ ] 所有容器配置 restart 策略
+ [ ] 生产环境数据库端口仅监听 127.0.0.1
+ [ ] 部署脚本包含健康检查验证
+ [ ] 部署失败时自动回滚机制
+ [ ] 日志文件配置轮转策略
+ [ ] Windows 服务配置自动重启
+ [ ] CI 流水线包含代码检查（flake8/black）
+ [ ] 测试覆盖率达到 80% 以上

---

✅ **本章项目里程碑：**

+ [ ] GitHub Actions test.yml 全部绿色通过
+ [ ] GitHub Actions deploy.yml 可执行
+ [ ] Docker 多阶段构建成功，镜像体积 < 400MB
+ [ ] docker-compose.prod.yml 配置完成且可启动
+ [ ] Windows 服务注册成功（nssm）
+ [ ] 部署脚本可正常执行

---

## 安全 CI/CD 实践

### 密钥管理

**基本原则：构建时绝不包含密钥！**

```text
# ❌ 错误：在 Dockerfile 中设置密钥
ENV SECRET_KEY=my-production-key

# ✅ 正确：运行时通过环境变量注入
docker run -e SECRET_KEY=$(cat /run/secrets/secret_key) myapp
```

### Docker 镜像安全

1. **使用最小基础镜像**：`python:3.12-slim`（约 120MB）而非 `python:3.12`（约 900MB）
2. **多阶段构建**：构建阶段包含 gcc 等工具，运行阶段只复制产物
3. **以非 root 用户运行**：

```dockerfile
RUN useradd -m ticket
USER ticket
```text

1. **使用 `.dockerignore` 排除敏感文件**：

```

.env          # 环境变量（可能含密钥）
.git/         # 完整 git 历史
**pycache**/  # 缓存文件
tests/        # 测试不进入生产镜像

```text

### GitHub Actions 安全

1. **使用 Actions Secrets** 存储密钥（非明文）
2. **限制 CI 触发器**：不要在 pull request 中暴露密钥
3. **依赖扫描**：`pip-audit` 检查已知漏洞

### 生产部署前安全检查清单

- [ ] `SECRET_KEY` 已改为随机长字符串（非默认值）
- [ ] `ENV=production`（禁用 debug/docs）
- [ ] `CORS_ORIGINS` 设为前端域名（非 `*`）
- [ ] HTTPS 已配置
- [ ] 数据库密码已修改
- [ ] Redis 密码已设置
- [ ] `.env` 不在版本控制中

- [ ] CI/CD 流水线完整：push → test → build → deploy

---

## 7. Docker 高阶用法

### 7.1 Docker 网络模式详解

Docker 网络决定了容器之间、容器和宿主机之间如何通信。

**bridge 网络（默认）：**

```bash
# 创建自定义 bridge 网络
docker network create ticket-network

# 启动容器时指定网络
docker run -d --name mysql --network ticket-network mysql:8.0
docker run -d --name redis --network ticket-network redis:7-alpine
docker run -d --name app --network ticket-network -p 8000:8000 ticket-api:latest
```

在同一个自定义 bridge 网络中，容器可以通过服务名（容器名）直接通信。例如 `app` 容器可以通过 `mysql:3306` 和 `redis:6379` 访问数据库，这就是 `docker-compose.yml` 中 `DB_HOST=mysql` 能工作的原因。

**host 网络：**

```bash
# 容器直接使用宿主机网络栈（性能好但隔离差）
docker run -d --name app --network host ticket-api:latest
```text

host 模式下不需要 `-p` 映射端口，容器直接使用宿主机的 8000 端口。适合对网络性能要求极高的场景，但端口冲突风险大。

**overlay 网络（跨主机通信）：**
用于多台服务器上的 Docker 容器互相通信，是 Docker Swarm 和 Kubernetes 的基础网络模式。本项目单机部署不需要用到。

**本项目为什么用 bridge + 自定义网络：**

- docker-compose 默认会为所有服务创建一个名为 `项目名_default` 的 bridge 网络
- 在生产配置中（`docker-compose.prod.yml`），我们将网络拆分为 `backend`（内部）和 `frontend`（可对外）
- MySQL 和 Redis 只连接到 `backend`（`internal: true`，外部无法访问）
- 只有 `app` 同时连接 `backend`（访问数据库）和 `frontend`（接收用户请求）
- 这样即使 Nginx 或 App 被攻破，攻击者也无法直接访问数据库

### 7.2 Docker 数据持久化详解

**Volume vs Bind Mount：**

```

Volume（由 Docker 管理）：
  docker run -v mysql_data:/var/lib/mysql mysql:8.0
  → 存在 Docker 管理目录：/var/lib/docker/volumes/mysql_data/
  → 优点：跨平台兼容、Docker 命令可管理、适合生产

Bind Mount（直接映射主机目录）：
  docker run -v /my/data:/var/lib/mysql mysql:8.0
  → 直接存在 /my/data 目录
  → 优点：开发方便、可以实时查看文件

```text

**命名卷 vs 匿名卷：**

```yaml
volumes:
  mysql_data:       # 命名卷 — 在 volumes: 下定义，Docker 管理生命周期
  redis_data:

services:
  mysql:
    volumes:
      - mysql_data:/var/lib/mysql    # 命名卷挂载
      - /tmp/backup:/backup          # Bind Mount 挂载
```

**数据备份与恢复：**

```bash
# 备份 MySQL 数据
docker exec ticket-mysql mysqldump -u root -p${DB_PASSWORD} ticket_prod > backup.sql

# 从文件恢复
cat backup.sql | docker exec -i ticket-mysql mysql -u root -p${DB_PASSWORD} ticket_prod

# 备份整个数据卷
docker run --rm -v mysql_data:/source -v $(pwd):/backup alpine \
  tar czf /backup/mysql_data_$(date +%Y%m%d).tar.gz -C /source .

# 恢复数据卷
docker run --rm -v mysql_data:/target -v $(pwd):/backup alpine \
  tar xzf /backup/mysql_data.tar.gz -C /target
```text

### 7.3 Docker 日志管理

**日志驱动（Logging Driver）：**

Docker 支持多种日志驱动，默认是 `json-file`：

```yaml
# docker-compose.yml 中配置日志轮转
services:
  app:
    logging:
      driver: "json-file"        # 默认驱动，将日志以 JSON 格式写入文件
      options:
        max-size: "10m"          # 每个日志文件最大 10MB
        max-file: "3"            # 保留最近 3 个文件（超出自动删除旧的）
```

其他常用日志驱动：

| 驱动 | 说明 | 适用场景 |
| ------ | ------ | ---------- |
| `json-file` | 默认，写本地文件 | 单机开发/生产 |
| `local` | 比 json-file 更高效（nat 格式） | 简单场景 |
| `syslog` | 发送到系统 syslog | Linux 集中日志 |
| `gelf` | 发送到 Graylog | 集中日志平台 |
| `fluentd` | 发送到 Fluentd | 大型架构 |
| `awslogs` | 发送到 CloudWatch | AWS 部署 |

**查看日志的命令：**

```bash
# 查看所有日志
docker logs ticket-api

# 跟踪实时日志
docker logs -f ticket-api

# 查看最近 N 行
docker logs --tail=100 ticket-api

# 带时间戳
docker logs -t ticket-api

# 筛选关键词
docker logs ticket-api 2>&1 | grep ERROR

# Docker Compose 方式（推荐）
docker compose logs -f
docker compose logs -f --tail=100 app
docker compose logs -f worker
```text

**集中式日志方案：**
单机可以用 `docker logs`，多服务器建议用 ELK（Elasticsearch + Logstash + Kibana）或 Loki + Grafana 收集所有日志。

不过对于本项目的规模，配置好日志轮转后，`docker logs` 完全够用。

### 7.4 Docker 资源限制

不限制资源时，一个容器可能占满宿主机的所有 CPU 和内存，导致其他服务崩溃甚至系统卡死。

```yaml
# 生产环境合理的资源配置
services:
  mysql:
    deploy:
      resources:
        limits:
          memory: 512M     # 最大 512MB 内存
          cpus: "1.0"      # 最多 1 个 CPU 核心
  redis:
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "0.5"
  app:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
  worker:
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "0.5"
```

还有几个相关的 Docker 运行参数：

```bash
# 限制内存 + 禁用 swap（防止内存不足时用磁盘交换拖慢系统）
--memory=512m --memory-swap=512m

# 当内存不足时，优先杀掉容器而不是宿主机的进程
--oom-score-adj=-500

# 限制 CPU（1.5 个核心）
--cpus=1.5

# 限制 CPU（预留 0.5 个核心保证可用）
--cpuset-cpus=0 --cpu-shares=512
```text

### 7.5 Docker 健康检查详解

健康检查是 Docker 判断容器"是否正常工作"的机制，不等同于"容器进程是否在运行"。

```dockerfile
# Dockerfile 中定义
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

```yaml
# docker-compose.yml 中定义
healthcheck:
  test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
  interval: 30s        # 每 30 秒检查一次
  timeout: 5s          # 每次检查 5 秒超时
  retries: 3           # 连续失败 3 次标记为 unhealthy
  start_period: 10s    # 启动后等待 10 秒才开始检查
```text

**为什么 healthcheck 和 depends_on 要配合使用：**

```yaml
depends_on:
  mysql:
    condition: service_healthy   # 等 MySQL 健康了才启动 App
  redis:
    condition: service_healthy   # 等 Redis 健康了才启动 App
```

不加 `condition: service_healthy` 的话，`depends_on` 只保证 MySQL 容器启动了，但 MySQL 可能需要几秒到几十秒才能接受连接。App 启动时 MySQL 还没准备好，就会报错退出。

不同服务合适的健康检查方式：

| 服务 | 检查方式 | 命令 |
| ------ | ---------- | ------ |
| MySQL | mysqladmin ping | `mysqladmin ping -h localhost` |
| Redis | redis-cli ping | `redis-cli ping` |
| FastAPI | HTTP GET /health | `curl -f http://localhost:8000/health` |
| Nginx | HTTP GET / | `curl -f http://localhost/` |

### 7.6 Docker 多阶段构建进阶

除了基本的两阶段构建，还有几种高级模式：

**多阶段 + 多架构构建（BuildX）：**

```bash
# 同时构建 amd64 和 arm64 镜像
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t username/ticket-api:latest \
  --push .
```text

这样在 AMD（Intel/AMD）和 ARM（Apple Silicon、树莓派）服务器上都能运行。

**缓存挂载加速 pip install：**

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

`--mount=type=cache` 是 BuildKit 的特性，可以把 pip 的下载缓存保留下来，下次构建更快。

**区分开发和生产依赖：**

```dockerfile
# 开发阶段：安装所有依赖（包括测试工具）
FROM python:3.12-slim AS development
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt

# 生产阶段：只安装生产依赖
FROM python:3.12-slim AS production
COPY requirements.txt .
RUN pip install -r requirements.txt
```text

### 7.7 Docker 安全最佳实践

1. **不要以 root 运行容器**：创建普通用户，用 `USER` 指令切换
2. **镜像漏洞扫描**：

   ```bash
   docker scout quickstart
   docker scout cves ticket-api:latest    # 查看已知漏洞
   ```

1. **只读根文件系统**：

   ```yaml
   services:
     app:
       read_only: true                    # 容器内不能写文件（除挂载卷）
       tmpfs: /tmp                        # 临时文件用内存
   ```text

2. **不要暴露 Docker Socket**：除非必要，不要把 `/var/run/docker.sock` 挂载到容器里

3. **使用 `.dockerignore`**：排除 `.env`、`.git`、`__pycache__` 等
4. **定期更新基础镜像**：`docker pull python:3.12-slim` 拉取最新的安全补丁

---

## 8. GitHub Actions 进阶用法

### 8.1 Matrix 构建策略详解

Matrix 能让你在不同配置下并行测试，快速发现兼容性问题。

```yaml
strategy:
  matrix:
    python-version: ["3.11", "3.12"]   # 同时测试两个 Python 版本
    os: [ubuntu-latest, windows-latest] # （可选）同时测试多个操作系统
  fail-fast: true                       # 任何一个 Matrix 任务失败，立即停止所有
  max-parallel: 4                       # 最多 4 个任务并行执行
```

**include / exclude：**

```yaml
strategy:
  matrix:
    python-version: ["3.11", "3.12"]
    include:                             # 添加额外的组合
      - python-version: "3.13-rc"        #  额外测试 RC 版本
        experimental: true
    exclude:                             # 排除特定组合
      - python-version: "3.11"           #  不测试 3.11 在 Windows
        os: windows-latest
```text

### 8.2 缓存优化

**Docker Layer Caching（gha 缓存）：**

Docker 构建时，每一层（RUN、COPY 等）都可以被缓存。使用了 GHA 缓存后：

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    push: true
    tags: username/ticket-api:latest
    cache-from: type=gha          # 从 GitHub Actions 缓存读取
    cache-to: type=gha,mode=max   # 写入缓存（mode=max 保存所有层）
```

第一次构建会慢一些（没有缓存），后续的构建会快很多（只需要重新构建改变的层）。

**缓存命中原理：**

```text
Dockerfile:
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .          ← 如果 requirements.txt 没变，这层缓存命中
RUN pip install -r requirements.txt  ← 上一层缓存命中，这层也命中
COPY . .                         ← 代码经常变，这层缓存失效
```

优化策略：把**不变的内容放在前面**（基础镜像、依赖文件）、**经常变的内容放后面**（代码）。

### 8.3 工作流触发方式详解

**workflow_dispatch（手动触发）：**

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "部署环境"
        required: true
        default: "staging"
        type: choice
        options:
          - staging
          - production
      debug_enabled:
        description: "启用调试"
        required: false
        default: false
        type: boolean
```text

配合 `github.event.inputs.environment` 使用：

```yaml
- name: 部署
  if: github.event.inputs.environment == 'production'
  run: echo "部署到生产环境"
```

**schedule（定时触发）：**

```yaml
on:
  schedule:
    # ┌─ 分钟 ┌─ 小时 ┌─ 日 ┌─ 月 ┌─ 星期
    - cron: "0 2 * * *"        # 每天 UTC 2:00（北京时间 10:00）
    - cron: "30 4 * * 1"       # 每周一 UTC 4:30
```text

定时任务适合：每天跑测试、定期检查 SSL 证书过期、定期备份。

### 8.4 Secrets 和环境变量管理

**多环境配置（GitHub Environments）：**

设置路径：仓库 Settings → Environments → 创建 `staging` 和 `production`

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production              # 引用 GitHub Environment
    env:
      DB_HOST: ${{ vars.PROD_DB_HOST }}  # vars 是普通变量（非加密）
    steps:
      - name: 使用 Secrets
        run: echo "登录到 ${{ secrets.DEPLOY_HOST }}"
```

不同的 Environment 可以有不同的：

+ 保护规则（需要审批才能部署）
+ 环境变量
+ 密钥

**OIDC（OpenID Connect）替代静态密钥：**

OIDC 可以让 GitHub Actions 直接向云服务商（AWS/Azure/GCP/阿里云）获取临时凭证，不需要在 Secrets 里存长期密钥。

```yaml
# 配置 AWS 的 OIDC（示例）
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456:role/GitHubActions
    aws-region: us-east-1
```text

### 8.5 步骤输出传递

一个 Step 的输出可以被后面的 Step 使用：

```yaml
steps:
  - name: 获取版本号
    id: version                     # 给这个步骤取个 ID
    run: echo "VERSION=v1.0.0" >> $GITHUB_OUTPUT

  - name: 使用版本号
    run: echo "当前版本是 ${{ steps.version.outputs.VERSION }}"
```

在部署中常用：先构建镜像，把镜像标签传给部署步骤。

### 8.6 Artifacts（构建产物）

Artifacts 是把工作流中生成的文件保存下来供下载。

```yaml
- name: 上传测试报告
  uses: actions/upload-artifact@v4
  if: always()                     # 即使测试失败也上传
  with:
    name: test-report-${{ github.sha }}
    path: |
      coverage.xml
      test-results/

- name: 下载之前的 Artifact
  uses: actions/download-artifact@v4
  with:
    name: test-report-${{ github.sha }}
```text

### 8.7 工作流调试

**tmate 调试会话：**

```yaml
- name: Setup tmate session
  if: ${{ failure() }}             # 仅在工作流失败时启用
  uses: mxschmitt/action-tmate@v3
  with:
    limit-access-to-actor: true    # 只有触发工作流的人能连接
```

当工作流失败时，会创建一个 SSH 会话让你登录到 Runner 查看现场。

**实时日志查看：**
在 GitHub Actions 页面点击运行中的工作流，可以看到每个 Step 的实时输出。点击右上角的 🔍 可以搜索日志。

**使用 `act` 在本地运行：**

```bash
# 安装 act（macOS）
brew install act

# 本地运行 GitHub Actions
act push                        # 模拟 push 事件
act pull_request                # 模拟 PR 事件
act -j test                     # 只运行 test job
act --secret-file .env          # 加载本地环境变量（用于测试）
```text

---

## 9. Windows 公网服务器部署完整指南

### 9.1 场景说明

如果你有一台配了**公网 IP** 的 Windows 机器（家里的台式机、办公室服务器、云服务器），你想让所有人都能访问你部署的票务系统。

本指南将详细讲解完整的部署流程。

### 9.2 环境准备

**Windows 版本要求：**

- Windows 10 Pro/Enterprise（专业版/企业版）— 支持 Hyper-V 和 Docker
- Windows 11 Pro — 同上
- Windows Server 2019/2022 — 服务器版，最适合生产环境
- Windows 10/11 家庭版 — 需要先安装 WSL2，再装 Docker Desktop

**安装步骤：**

1. **启用 WSL2（如果还没有）：**

   ```powershell
   # 以管理员身份运行 PowerShell
   wsl --install
   # 安装 Ubuntu 发行版
   wsl --install -d Ubuntu
   # 设置 WSL2 为默认版本
   wsl --set-default-version 2
   ```

1. **安装 Docker Desktop：**
   + 前往 <https://www.docker.com/products/docker-desktop/> 下载
   + 安装时选择**使用 WSL2 后端**
   + 安装完成后重启电脑
   + Docker Desktop 启动后，右下角托盘会显示 Docker 图标

2. **配置 Docker Desktop 资源：**
   + 右键托盘 Docker 图标 → Settings → Resources → WSL Integration
   + 确认 Ubuntu 集成已开启
   + Resources → Advanced：建议给 WSL2 分配 4-8GB 内存

3. **安装 Git for Windows：**

   ```powershell
   # 下载安装：https://git-scm.com/download/win
   # 或者用 winget：
   winget install --id Git.Git -e --source winget
   ```text

4. **验证安装：**

   ```powershell
   wsl -l -v                    # 确认 WSL2 运行中
   docker --version             # 确认 Docker 可用
   docker compose version       # 确认 Docker Compose 可用
   git --version                # 确认 Git 可用
   ```

### 9.3 公网访问配置（关键步骤）

这部分是把你的 Windows 机器从"只有局域网能访问"变成"全世界都能访问"。

**第一步：确认网络环境**

```powershell
# 查看你的内网 IP
ipconfig
# 找到"以太网适配器"或"WiFi"下的 IPv4 地址，类似 192.168.1.100

# 查看你的公网 IP（需要访问外网）
curl ifconfig.me
```text

**第二步：路由器端口转发**

在你的路由器管理页面（通常是 <http://192.168.1.1> 或 <http://192.168.0.1> ）设置端口转发：

```

需要转发的端口：
  WAN 80  → 内网服务器IP:80    （HTTP，供用户访问）
  WAN 443 → 内网服务器IP:443   （HTTPS，加密访问）
  WAN 8000 → 内网服务器IP:8000 （API，可选，调试用）

⚠️ 重要：不同路由器设置页面不同，常见名称有：
  端口转发（Port Forwarding）
  虚拟服务器（Virtual Server）
  NAT 映射

```text

**第三步：Windows 防火墙**

如果路由器转发了但还连不上，多半是 Windows 防火墙拦截了：

```powershell
# 以管理员身份运行 PowerShell

# 创建入站规则允许 80 端口（HTTP）
New-NetFirewallRule -DisplayName "Allow HTTP 80" `
  -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

# 创建入站规则允许 443 端口（HTTPS）
New-NetFirewallRule -DisplayName "Allow HTTPS 443" `
  -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# 创建入站规则允许 8000 端口（API 调试）
New-NetFirewallRule -DisplayName "Allow API 8000" `
  -Direction Inbound -Protocol TCP -LocalPort 8000 -Action Allow
```

**第四步：DDNS（动态域名解析）**

如果你没有固定公网 IP（大部分家庭宽带都是动态 IP），需要 DDNS：

```text
方案一：路由器自带 DDNS
  进路由器管理页面 → 高级设置 → DDNS
  支持：花生壳、每步、阿里云 DDNS、Dyndns

方案二：Windows 客户端
  推荐：DDNS-GO（免费开源，支持大部分 DNS 服务商）
  下载地址：https://github.com/jeessy2/ddns-go/releases
```

**常见问题：**

+ **运营商封锁 80/443 端口**：部分运营商（特别是家庭宽带）会封锁 80/443 端口。可以用其他端口（比如 8080/8443）或升级企业宽带
+ **没有公网 IP**：打电话给运营商要公网 IP（就说要装监控），或者用 ngrok/frp 内网穿透
+ **路由器重启后 IP 变了**：DDNS 会自动更新 DNS 记录，等 1-5 分钟生效

### 9.4 在 Windows 上部署项目

```powershell
# 1. 克隆代码（如果还没做）
cd C:\
git clone <你的仓库地址> ticket-booking
cd ticket-booking

# 2. 配置生产环境变量
copy .env.example .env
# ⚠️ 重要：编辑 .env，修改以下配置：
#   SECRET_KEY = openssl rand -hex 32 （生成随机密钥）
#   ENV = production
#   DB_PASSWORD = 强密码（至少 16 位）
#   REDIS_PASSWORD = 设置 Redis 密码
#   CORS_ORIGINS = [\"https://你的域名\"]

# 3. 启动所有服务
docker compose -f docker-compose.prod.yml up -d

# 4. 验证服务是否启动
docker ps
# 应该看到 ticket-mysql, ticket-redis, ticket-api, ticket-worker 都处于 Up 状态

# 5. 测试 API
curl http://localhost:8000/health
# 应该返回 {"status": "ok"}
```text

### 9.5 Nginx 反向代理（Windows）

如果直接暴露 8000 端口不太安全，建议用 Nginx 做反向代理，同时提供 HTTPS。

**1️⃣ 下载 Nginx for Windows：**

- 下载地址：<https://nginx.org/en/download.html>
- 下载 Stable 版本的 Windows 版（nginx-1.26.x.zip）
- 解压到 `C:\nginx`

**2️⃣ 配置 Nginx：**
编辑 `C:\nginx\conf\nginx.conf`：

```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout  65;

    # 后续所有请求都转发到本机的 API 服务
    server {
        listen       80;       # 监听 80 端口
        server_name  你的域名;  # 换成你的域名

        location / {
            proxy_pass http://127.0.0.1:8000;   # 转发到 FastAPI
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /ws {
            proxy_pass http://127.0.0.1:8000;   # WebSocket 转发
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

**3️⃣ 启动 Nginx：**

```powershell
# 测试配置是否正确
C:\nginx\nginx.exe -t

# 启动 Nginx
C:\nginx\nginx.exe

# 如果已经启动，重载配置
C:\nginx\nginx.exe -s reload
```text

**4️⃣ 注册为 Windows 服务（开机自启）：**

```powershell
# 下载 NSSM：https://nssm.cc/download
# 把 nssm.exe 放到 C:\Windows\System32\

# 注册 Nginx 服务
nssm install TicketNginx "C:\nginx\nginx.exe"
nssm set TicketNginx AppDirectory "C:\nginx"
nssm set TicketNginx DisplayName "票务系统 Nginx 反向代理"
nssm set TicketNginx Start SERVICE_AUTO_START

# 启动服务
nssm start TicketNginx
```

### 9.6 SSL/HTTPS 配置

**使用 Let's Encrypt 免费证书（Windows）：**

1️⃣ **安装 Certbot：**

```powershell
# 下载 Windows 版 certbot
# https://certbot.eff.org/instructions?ws=other&os=windows

# 或者用 chocolatey 安装
choco install certbot
```text

2️⃣ **申请证书：**

```powershell
# 申请前确保 80 端口在公网可以访问（certbot 需要验证域名所有权）
certbot certonly --standalone -d 你的域名
```

3️⃣ **配置 Nginx HTTPS：**

```nginx
server {
    listen 443 ssl;
    server_name 你的域名;

    ssl_certificate     C:\Certbot\live\你的域名\fullchain.pem;
    ssl_certificate_key C:\Certbot\live\你的域名\privkey.pem;

    # 安全配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP 自动跳转 HTTPS
server {
    listen 80;
    server_name 你的域名;
    return 301 https://$server_name$request_uri;
}
```text

4️⃣ **自动续期：**

Let's Encrypt 证书有效期 90 天，需要定期续期：

```powershell
# 创建一个 PowerShell 脚本 cert-renew.ps1：
certbot renew --quiet
C:\nginx\nginx.exe -s reload

# 用 Windows 计划任务每月执行一次：
# 开始 → 搜索"任务计划程序" → 创建基本任务
#   名称：Certbot Renew
#   触发器：每月 1 号
#   操作：启动程序 → C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
#   参数：-File C:\ticket-booking\scripts\cert-renew.ps1
```

### 9.7 域名和备案

**购买域名：**

+ 阿里云、腾讯云、GoDaddy、Namecheap
+ 推荐买 `.com` 或 `.cn` 后缀，价格几十到一百多一年

**DNS 解析（把域名指向你的公网 IP）：**

```text
在你的域名管理后台，添加 A 记录：
  主机记录  @   →  记录值  你的公网IP
  主机记录  www →  记录值  你的公网IP

等待 10 分钟到 2 小时生效
验证：ping 你的域名  # 应该返回你的公网IP
```

**ICP 备案（中国服务器必需）：**
如果你的服务器在中国大陆，域名必须备案，否则会被拦截：

1. 登录阿里云/腾讯云备案系统
2. 提交身份证、网站信息
3. 审核周期约 5-20 个工作日
4. 备案通过后才能用 80/443 端口

### 9.8 Windows 服务管理

**NSSM 完整使用指南：**

```powershell
# === 注册服务 ===

# 注册 API 服务（不使用 Docker 时）
nssm install TicketAPI "C:\Python312\python.exe" "-m uvicorn app.main:app --host 0.0.0.0 --port 8000"
nssm set TicketAPI AppDirectory "C:\ticket-booking"
nssm set TicketAPI DisplayName "票务系统 API 服务"
nssm set TicketAPI Start SERVICE_AUTO_START
nssm set TicketAPI AppStdout "C:\ticket-booking\logs\api-stdout.log"
nssm set TicketAPI AppStderr "C:\ticket-booking\logs\api-stderr.log"
nssm set TicketAPI AppRotateFiles 1
nssm set TicketAPI AppRotateSeconds 86400     # 每天轮转日志
nssm set TicketAPI AppRotateBytes 10485760    # 10MB 轮转
nssm set TicketAPI AppThrottle 3000           # 崩溃后 3 秒重启

# === 服务管理命令 ===
nssm start TicketAPI          # 启动
nssm stop TicketAPI           # 停止
nssm restart TicketAPI        # 重启
nssm status TicketAPI         # 查看状态
nssm edit TicketAPI           # 编辑配置（GUI 界面）

# === 注册 Docker Desktop 为开机自启 ===
# Docker Desktop 本身有开机自启选项：Settings → General → Start Docker Desktop when you log in

# === 注册 docker-compose 服务（Docker 方式部署时的最佳方案）===
nssm install TicketStack "C:\Program Files\Docker\Docker\resources\bin\docker.exe" "compose -f C:\ticket-booking\docker-compose.prod.yml up -d"
nssm set TicketStack AppDirectory "C:\ticket-booking"
nssm set TicketStack DisplayName "票务系统 Docker 服务栈"
nssm set TicketStack Start SERVICE_AUTO_START
```text

### 9.9 服务器安全加固

```powershell
# === 1. Windows 防火墙 ===
# 只开放必要的端口
New-NetFirewallRule -DisplayName "Allow HTTP 80" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
New-NetFirewallRule -DisplayName "Allow HTTPS 443" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
# 3389 远程桌面：如果不需要外网访问，只允许内网
New-NetFirewallRule -DisplayName "Allow RDP" -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow -RemoteAddress 192.168.0.0/16

# === 2. 修改远程桌面端口（可选，防扫描）===
# 注册表 HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp\PortNumber
# 改为 50000 之类的端口，然后重启

# === 3. 关闭不必要的服务 ===
# 控制面板 → 管理工具 → 服务
# 建议关闭：Print Spooler（如果没有打印机）、Windows Media Player Network Sharing

# === 4. 开启 Windows 更新 ===
# 设置 → 更新和安全 → Windows 更新 → 自动安装更新

# === 5. 防病毒软件 ===
# Windows Defender 足够，添加排除目录提高性能：
Add-MpPreference -ExclusionPath "C:\ticket-booking"
Add-MpPreference -ExclusionPath "C:\ProgramData\Docker"

# === 6. 账户安全 ===
# 重命名 Administrator 账户
Rename-LocalUser -Name "Administrator" -NewName "SecAdmin"
# 创建普通用户运行服务（不要用管理员跑应用）
New-LocalUser -Name "ticket-svc" -Password (ConvertTo-SecureString "强密码" -AsPlainText -Force)
# 给该用户最小权限

# === 7. 事件监控 ===
# Windows 事件查看器 → Windows 日志 → 安全
# 关注：登录失败（事件 ID 4625）、账户锁定（4740）
```

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `.github/workflows/test.yml` | CI 流程：pytest + 覆盖率检查，PR 触发 |
| `.github/workflows/deploy.yml` | CD 流程：构建 + 部署，合并到 main 触发 |
| `Dockerfile` | 多阶段构建：builder → runner 最小镜像 |
| `docker-compose.yml` | 开发环境编排（MySQL + Redis + App） |
| `docker-compose.prod.yml` | 生产环境编排（+ Nginx + Worker） |
| `scripts/` | 部署脚本 |

**关键 CI 配置：**

```yaml
# .github/workflows/test.yml 自动化测试
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql: ...  # 启动 MySQL 容器
      redis: ...  # 启动 Redis 容器
    steps:
      - run: pytest --cov=app --cov-fail-under=80
```text

## 10. 日志体系与错误处理

### 10.1 项目中的日志

本项目使用 `structlog` 进行结构化日志记录。相比 `print`，结构化日志包含更丰富的信息：

```json
# 实际日志输出（示例）
{"event": "HTTP Request", "method": "POST", "path": "/api/v1/orders/", "status_code": 201, "duration_ms": 245, "user_id": 1, "request_id": "abc123", "timestamp": "2026-05-06T10:30:00Z"}
```

**日志级别：**

| 级别 | 什么时候用 | 示例 |
| ------ | ----------- | ------ |
| DEBUG | 调试信息，开发时看 | SQL 语句、变量值 |
| INFO | 正常操作记录 | 用户注册、创建订单 |
| WARNING | 值得注意但不出错 | 请求速度异常、重试操作 |
| ERROR | 发生了错误但服务还在运行 | 数据库连接失败、支付验签失败 |
| CRITICAL | 严重错误，服务可能不可用 | 数据库完全不可用、磁盘满 |

### 10.2 日志查看命令大全

```bash
# === Docker 日志 ===
docker logs ticket-api -f                     # 实时追踪 API 日志
docker logs ticket-api --tail=100             # 查看最后 100 行
docker logs ticket-api -f -t                  # 实时追踪带时间戳
docker compose logs -f                        # 所有服务日志
docker compose logs -f --tail=50 app          # app 服务最后 50 行，实时追踪

# 筛选特定级别的日志
docker logs ticket-api 2>&1 | grep "ERROR"
docker logs ticket-api 2>&1 | grep "WARNING"

# 按时间筛选
docker logs --since 2026-05-06T10:00:00 ticket-api
docker logs --since 10m ticket-api            # 最近 10 分钟
docker logs --until 2026-05-06T12:00:00 ticket-api

# === Nginx 日志 ===
# Windows 默认在 C:\nginx\logs\
type C:\nginx\logs\access.log                  # 访问日志
type C:\nginx\logs\error.log                   # 错误日志

# Nginx 访问日志每行格式解读：
# 192.168.1.100 - - [06/May/2026:10:30:00 +0800] "GET /api/v1/events/ HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
# ├ 客户端IP     └ 请求时间            └ HTTP方法和路径       └ 状└ 大小 └ 用户代理
#   （谁访问的）       （什么时候）         （干了什么）          态码    （响应体）  （什么浏览器）

# === 应用日志 ===
# 如果配置了文件日志
type C:\ticket-booking\logs\app.log
```text

### 10.3 日志轮转配置

**Docker 日志轮转：**
已经在 `docker-compose.prod.yml` 中配置了，每个容器最多保留 3 个 10MB 的日志文件。

**如果 Docker 日志不轮转（系统默认配置）：**

```bash
# 查看 Docker 日志占了多少磁盘
docker system df

# 手动清理所有 Docker 日志（Linux）
sudo find /var/lib/docker/containers -name "*-json.log" -exec truncate -s 0 {} \;
```

**Windows 下 Nginx 日志轮转：**

```powershell
# 创建日志轮转脚本 C:\ticket-booking\scripts\rotate-logs.ps1
$logDir = "C:\nginx\logs"
$maxAge = 7  # 保留 7 天

Get-ChildItem $logDir -Filter "*.log*" | Where-Object {
    $_.LastWriteTime -lt (Get-Date).AddDays(-$maxAge)
} | Remove-Item

# 用 Windows 计划任务每天执行
```text

### 10.4 HTTP 错误码说明

FastAPI 返回的标准 HTTP 状态码：

| 状态码 | 含义 | 常见原因 | 排查方向 |
| -------- | ------ | ---------- | ---------- |
| **200** | 成功 | 请求正常处理 | ✅ 一切正常 |
| **201** | 创建成功 | 注册成功、订单创建成功 | ✅ 一切正常 |
| **400** | 请求参数错误 | JSON 格式不对、缺少字段 | 检查请求体是否符合 API 文档 |
| **401** | 未认证 | 没传 token 或 token 过期了 | 重新登录获取新 token |
| **403** | 无权限 | 普通用户点管理员接口 | 切换管理员账号 |
| **404** | 资源不存在 | 活动 ID 或订单 ID 不存在 | 检查传的 ID 是否正确 |
| **409** | 冲突 | 重复注册、票已售完 | 换一个活动或座位 |
| **422** | 参数校验失败 | 邮箱格式不对、密码太短 | 看响应体的 detail 字段 |
| **429** | 请求太多 | 触发 Nginx 或应用的限流 | 等一下再试 |
| **500** | 服务器错误 | 代码异常、数据库连不上 | 看 Docker 日志找具体错误 |

### 10.5 常见问题全解

#### 环境搭建

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| `pip install` 超慢 | 默认走 PyPI 国外源 | `pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple` |
| `python -m venv` 报错 | Python 没装或 PATH 不对 | 重装 Python，勾选"Add Python to PATH" |
| 虚拟环境激活不了 | PowerShell 执行策略 | 管理员运行 `Set-ExecutionPolicy RemoteSigned` |
| `alembic upgrade head` 报错 | MySQL 没启动或配置不对 | `docker ps` 检查 MySQL；检查 `.env` 中 DB_* |
| 端口 8000 被占用 | 其他程序在用 | `netstat -ano \| findstr :8000` → `taskkill /PID 进程ID /F` |

#### Docker

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| Docker Desktop 启动失败 | WSL2 或 Hyper-V 没开 | `wsl --install`；控制面板 → 开启 Hyper-V |
| 镜像拉不下来 | 被墙 | 配国内镜像加速器（阿里云/科大） |
| 容器反复重启 | 健康检查失败 | `docker logs 容器名` 看错误 |
| `docker compose up` 报语法错 | YAML 缩进问题 | 用空格代替 Tab，缩进对齐 |
| Volume 数据丢了 | 用了 `down -v` | `-v` 会删卷，备份后再删 |
| 宿主机和容器时间不一致 | 时区没设置 | 加 `TZ=Asia/Shanghai` 环境变量 |

#### 数据库

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| Lost connection to MySQL | 网络不通或密码错 | `docker logs ticket-mysql` 查看；检查 DB_HOST |
| `pymysql.err.OperationalError` | 连接配置不对 | 检查 `.env` 中数据库配置 |
| Migration 冲突 | 表已存在 | `alembic current` 看版本；`alembic downgrade -1` 回退 |
| MySQL 连接过多 | 应用没正确关闭连接 | 检查 FastAPI 是否用了连接池；重启应用 |

#### Redis

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| Redis 连接超时 | Redis 没启动 | `docker logs ticket-redis` 检查 |
| NOAUTH 错误 | 密码不对 | 检查 REDIS_PASSWORD 配置 |
| OOM 命令不允许 | Redis 内存满了 | `redis-cli info memory`；`CONFIG SET maxmemory 512mb` |
| Redis 数据丢了 | 没配置持久化 | 确认 `--appendonly yes` 参数 |

#### 网络和公网访问

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| 公网访问不了 | 端口没转发、防火墙拦截 | 检查路由器端口转发 + Windows 防火墙 |
| Nginx 502 Bad Gateway | 后端没启动 | `docker ps` 或 `nssm status` 检查服务 |
| Nginx 504 Gateway Timeout | 后端响应超时 | 检查后端处理时间；调大 `proxy_read_timeout` |
| SSL 证书无效 | 过期或域名不匹配 | `certbot renew`；确认域名正确 |
| WebSocket 连不上 | Nginx 没配置 upgrade | 检查 Nginx 中 WebSocket 的 proxy_set_header |
| CORS 跨域错误 | 域名不在白名单 | 检查 .env 中 CORS_ORIGINS |

#### 支付宝支付

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| 支付页面打不开 | APP_ID 没配、公钥没传 | 检查支付宝应用配置 |
| 异步通知收不到 | 通知地址需公网 HTTPS | 用 ngrok 本地测试；确认 Nginx 配置 |
| 验签失败 | 公钥私钥不匹配 | 重新生成密钥对重新上传 |
| 退款失败 | 订单状态不对 | 确认订单已支付 |

#### CI/CD 和部署

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| GitHub Actions 失败 | 测试没通过或 Secrets 缺失 | 点失败的工作流看具体错误 |
| Docker push 失败 | Docker Hub 登录失败 | 检查 DOCKER_USERNAME/DOCKER_PASSWORD |
| SSH 部署失败 | 服务器 SSH 配置不对 | 检查 DEPLOY_HOST/DEPLOY_KEY |
| 部署后访问不了 | 镜像没拉成功或配置不对 | SSH 到服务器手动调试 |

#### Windows 特有

| 问题 | 原因 | 解决方法 |
| ------ | ------ | ---------- |
| 路径用 `\` 还是 `/` | Windows 用 `\`，但 Python 中 `/` 也支持 | 用 `os.path.join()` 或 Pathlib |
| 换行符 CRLF | 和 Linux 不一样 | Git 配置 `core.autocrlf=true` |
| Docker Desktop 吃内存 | WSL2 默认占用太多 | 创建 `%UserProfile%\.wslconfig`，设 `memory=4GB` |
| Python 装在 Program Files | 路径有空格导致问题 | 装到 `C:\Python312\` |
| 环境变量改了不生效 | 需要重启终端 | 重启 PowerShell 或 `refreshenv` |

---

## 11. 运维 SOP（标准操作流程）

### 11.1 完整发布流程

```

开发者编写代码
      │
      ▼
git add → git commit → git push (main 分支)
      │
      ▼
GitHub Actions test.yml 自动执行
  ├── 启动临时 MySQL + Redis
  ├── pip install -r requirements.txt
  ├── alembic upgrade head
  ├── pytest --cov=app --cov-fail-under=80
  └── locust 压力测试
      │
      ├── ❌ 失败 → 修复代码 → 重新推送
      │
      └── ✅ 通过 →
            │
            ▼
      GitHub Actions deploy.yml 自动执行
        ├── docker login
        ├── docker build & push (latest + commit SHA)
        └── SSH 到服务器
              │
              ▼
        服务器执行：
        ├── docker compose pull app
        ├── docker compose up -d --no-deps app
        ├── docker image prune -f
        └── 健康检查
              │
              ├── ❌ 失败 → 自动回滚到上一个版本
              │
              └── ✅ 通过 → 部署完成！

```text

### 11.2 发布前检查清单

```markdown
- [ ] 本地 pytest 全部通过（全覆盖跑一遍）
- [ ] 所有新功能都写了测试
- [ ] SECRET_KEY 已改为随机字符串（`openssl rand -hex 32`）
- [ ] ENV=production（禁用 debug 和 API 文档）
- [ ] CORS_ORIGINS 设为前端实际域名
- [ ] HTTPS 证书未过期
- [ ] 数据库密码已改为强密码
- [ ] Redis 密码已设置
- [ ] .env 文件不在 Git 跟踪中（`git check-ignore .env`）
- [ ] GitHub Secrets 已全部配置
- [ ] Docker Hub 仓库已创建并有权限
- [ ] 生产服务器已做好部署准备
- [ ] docker-compose.prod.yml 配置完整
- [ ] Nginx 配置已更新
```

### 11.3 发布后验证清单

```markdown
- [ ] 访问 https://你的域名/health → 返回 200
- [ ] 注册一个新账号 → 成功
- [ ] 创建一个新活动 → 成功
- [ ] 下单购买 → 成功
- [ ] docker logs 没有 ERROR 级别日志
- [ ] Nginx 访问日志正常（没有大量 5xx 错误）
- [ ] 系统资源正常（CPU < 50%, 内存 < 60%, 磁盘 < 80%）
- [ ] SSL 证书检查（https://www.ssllabs.com/ssltest/）
```text

### 11.4 日常运维

**每日巡检（5 分钟）：**

```bash
# 1. 服务是否都在运行
docker ps

# 2. 健康检查
curl -f https://你的域名/health

# 3. 磁盘空间
# Windows: 打开"此电脑"看 C 盘剩余
# Linux: df -h

# 4. 检查错误日志
docker logs ticket-api --tail=50 | grep -i "ERROR\|CRITICAL"

# 5. 检查最近订单（确认支付功能正常）
curl https://你的域名/api/v1/orders/ | python -m json.tool
```

**每周操作：**

+ 检查 SSL 证书剩余天数（`certbot certificates`）
+ 检查数据库备份已成功生成
+ 检查日志文件大小
+ `docker system df` 查看 Docker 磁盘使用

**每月操作：**

+ Windows Update 更新（选非高峰时段重启）
+ Docker Desktop 升级到最新版
+ Python 依赖安全检查：`pip-audit`
+ 清理无用 Docker 镜像：`docker image prune -a`
+ 数据库表优化：`docker exec ticket-mysql mysqlcheck -o ticket_prod`

### 11.5 紧急响应流程

**服务宕机：**

```text
1. 发现问题：用户反馈无法访问，或监控告警
2. 检查现状：
   docker ps -a                    # 容器都活着吗？
   docker logs <容器名> --tail=100 # 最近 100 行有啥错误？
   nssm status TicketAPI          # 服务状态（Windows 非 Docker 部署）
3. 快速恢复：
   docker restart <容器名>         # 先重启试试
   不行就：docker compose down && docker compose up -d
4. 如果还不行：
   检查磁盘空间（日志写满了？）
   检查内存（OOM kill了？）
   检查网络（DNS / 端口转发正常吗？）
5. 回滚：
   docker compose pull app:旧版本号
   docker compose up -d --no-deps app
6. 事后复盘：什么原因导致的？怎么防止再发生？
```

**数据库故障：**

```text
1. docker logs ticket-mysql --tail=50  # 看错误
2. docker restart ticket-mysql         # 重启
3. 如果数据卷损坏：
   docker compose down
   docker volume ls                    # 找到数据卷
   # 从备份恢复
4. 恢复后验证数据完整性
```

**安全事件（被入侵）：**

```text
1. 立即下线服务（断开网络或关闭服务）
2. 保留现场（日志不要删、容器不要重启）
3. 重置所有密码：数据库、Redis、JWT Secret、Docker Hub、服务器登录
4. 检查入侵途径：SSH 爆破？应用漏洞？弱密码？
5. 修复漏洞
6. 从备份恢复干净数据
7. 加强安全措施后重新上线
```

### 11.6 数据库备份与恢复

```bash
# Windows 定时备份脚本 C:\ticket-booking\scripts\backup-db.ps1

$backupDir = "C:\ticket-booking\backups"
$date = Get-Date -Format "yyyyMMdd-HHmmss"
$dbName = "ticket_prod"
$dbUser = "root"
$dbPass = "你的数据库密码"

# 创建备份目录
New-Item -ItemType Directory -Force -Path $backupDir

# 用 Docker 执行 mysqldump（MySQL 在容器里）
docker exec ticket-mysql mysqldump -u $dbUser -p$dbPass $dbName |
  Out-File -FilePath "$backupDir\$dbName-$date.sql" -Encoding utf8

# 压缩备份（节省空间）
Compress-Archive -Path "$backupDir\$dbName-$date.sql" `
  -DestinationPath "$backupDir\$dbName-$date.zip"

# 删除原始 SQL 文件
Remove-Item "$backupDir\$dbName-$date.sql"

# 保留最近 30 天的备份，删除更早的
Get-ChildItem $backupDir -Filter "*.zip" | Where-Object {
    $_.LastWriteTime -lt (Get-Date).AddDays(-30)
} | Remove-Item

Write-Host "数据库备份完成：$backupDir\$dbName-$date.zip"
```text

在 Windows 计划任务中设置每天凌晨 3 点执行此脚本。

**从备份恢复：**

```bash
# 解压备份文件
Expand-Archive -Path "backup-20260506.zip" -DestinationPath .

# 恢复数据到 MySQL（会覆盖现有数据！）
Get-Content "ticket_prod-20260506-030000.sql" |
  docker exec -i ticket-mysql mysql -u root -p"密码" ticket_prod

# 验证恢复
curl https://你的域名/api/v1/events/
```
