# 第12章：ngrok / frp 内网穿透 — 零基础到生产实践

## 本章学习目标

本章将深入学习两种主流的**内网穿透**工具：ngrok 和 frp。本地开发环境（如本项目的 FastAPI 票务系统运行在 `localhost:8000`）默认无法被外网访问，内网穿透工具可以在本地与公网服务器之间建立隧道，使外部用户能够访问本地服务。

**为什么需要内网穿透？**

```text
日常开发流程：
┌──────┐     localhost:8000     ┌──────────┐
│ 你   │ ────────────────────── │ FastAPI  │
│      │                       │ 票务系统  │
└──────┘                       └──────────┘
  ↑ 只有你能访问

需要对接外部服务时：
┌──────┐                       ┌──────────┐
│支付宝 │                       │ FastAPI  │
│ 回调  │ ─── 需要公网可达 ──── │ 票务系统  │
└──────┘                       └──────────┘

解决方案：内网穿透
┌──────┐   隧道     ┌──────────┐    反向代理    ┌──────────┐
│支付宝 │ ──────── │ ngrok/frp│ ──────────── │ FastAPI  │
│ 回调  │          │ 公网入口  │               │ 票务系统  │
└──────┘          └──────────┘               └──────────┘
```

**学习路径：**

1. ngrok 完整教程（安装、CLI 参数、配置文件、Dashboard、安全配置）
2. ngrok.yml 配置详解
3. frp 架构（frps + frpc）
4. frps.toml / frpc.toml 配置详解
5. 代理类型：TCP / HTTP / STCP / XTCP
6. ngrok vs frp 对比
7. 项目实战：为本项目的支付宝回调配置隧道

---

## 1. ngrok 完整教程

### 1.1 什么是 ngrok？

ngrok 是一个商业 + 开源的**反向代理隧道**工具。它在公网提供一个入口地址，将所有到达该地址的流量通过隧道转发到本地服务器。

核心原理：

```text
                    ngrok 云服务
                    ┌───────────┐
                    │ 隧道服务器  │
                    │ xxxxx.ngrok│ ← 公网可访问
                    │    .app    │
                    └─────┬─────┘
                          │ 长连接隧道（outbound）
                          │
                    ┌─────▼─────┐
                    │  ngrok 客户端 │
                    │  (本地运行)  │
                    └─────┬─────┘
                          │
                    ┌─────▼─────┐
                    │ localhost:8000 │
                    │  FastAPI 票务  │
                    └───────────┘
```

**关键特性：**

- 客户端和服务器之间建立的是**出站连接**（outbound），所以即使本地没有公网 IP 也能用
- 所有的 HTTP 请求在隧道中封装为 WebSocket 帧传输
- 支持 HTTP/TCP/TLS 隧道类型
- 提供 Dashboard 用于查看和重放请求（`localhost:4040`）

### 1.2 安装与认证

**安装方式：**

```bash
# macOS（Homebrew）
brew install ngrok

# Linux
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc > /dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok

# Windows（winget）
winget install ngrok

# 或从官网 https://ngrok.com/download 下载二进制文件
```text

**注册与认证：**

```bash
# 1. 注册账号：https://dashboard.ngrok.com/signup

# 2. 获取 Authtoken：Dashboard → Your Authtoken

# 3. 配置到本地
ngrok config add-authtoken YOUR_AUTHTOKEN_STRING

# 验证
ngrok config check
```

**Authtoken 的作用：**

- 关联到你的 ngrok 账号
- 解锁免费版功能（1 个在线隧道、4 个隧道/分钟）
- 使用保留域名和自定义域名
- 在 Dashboard 中查看历史请求

### 1.3 CLI 参数详解

**基础用法：**

```bash
# 暴露 HTTP 服务（最常用）
ngrok http 8000

# 指定域名（仅付费用户可用保留域名）
ngrok http 8000 --domain ticket-api.ngrok.app

# 暴露多个端口
ngrok http 8000 8001
```text

**输出解析：**

```

ngrok                                                                 (Ctrl+C to quit)

Session Status                online
Account                       YourName (Plan: Free)
Version                       3.x.x
Region                        United States (us)
Latency                       45ms
Web Interface                 <http://127.0.0.1:4040>
Forwarding                    <https://xxxx-xx-xxx-xxx-xx.ngrok-free.app> -> <http://localhost:8000>

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00

```text

**各区域解释：**

```

Session Status   → 隧道连接状态（online/offline）
Account          → 登录账号及套餐
Region           → 接入点所在区域
Latency          → 客户端到 ngrok 边缘的延迟
Web Interface    → 本地 Dashboard 访问地址
Forwarding       → 公网入口 → 本地服务的映射
Connections      → 连接统计（总连接数/当前打开数/最近1分钟/5分钟速率）

```text

**CLI 完整参数：**

```bash
# 隧道类型
ngrok http [port]              ← HTTP 隧道（自动支持 HTTPS）
ngrok tcp [port]               ← TCP 隧道（适用于非 HTTP 协议，如 SSH、MySQL）
ngrok tls [port]               ← TLS 隧道（自定义 TLS 终止）

# 常用选项
--domain=DOMAIN                ← 指定域名（免费用户只能乱序域名，付费用户可用保留域名）
--basic-auth="user:pass"       ← 基础认证保护隧道
--host-header=HOST             ← 重写 Host 头（例如 --host-header="ticket.example.com"）
--subdomain=SUBDOMAIN          ← 指定子域名（旧版语法，新版用 --domain）
--region=REGION                ← 选择区域（us/eu/ap/au/sa/jp/in）
--inspect=DISABLE              ← 禁用请求检查（默认为启用）
--log=LOGFILE                  ← 日志文件路径
--log-format=FORMAT            ← 日志格式（json/term）
--config=PATH                  ← 指定配置文件路径（默认 ~/.config/ngrok/ngrok.yml）
--scheme=SCHEME                ← 隧道协议（http/https 或同时）

# 实际示例
ngrok http 8000 --basic-auth="admin:secret123" --host-header="ticket.example.com"
ngrok tcp 3306 --remote-addr 1.tcp.ngrok.io:23306    ← 暴露本地 MySQL
ngrok http 8000 --domain=my-ticket.ngrok.app --region=ap
```

**Region 区域选择：**

| Region 代码 | 位置 | 延迟（对中国用户） |
| ------------- | ------ | -------------------- |
| `us` | 美国（默认） | 150-200ms |
| `eu` | 欧洲 | 250-300ms |
| `ap` | 亚太（新加坡/东京） | 50-100ms |
| `au` | 澳大利亚 | 150-200ms |
| `sa` | 南美 | 300-400ms |
| `jp` | 日本 | 40-80ms |
| `in` | 印度 | 100-150ms |

选择建议：本项目面向中文用户，选择 `ap`（亚太）或 `jp`（日本）区域可获得更低延迟。

### 1.4 ngrok.yml 配置详解

ngrok 支持 YAML 格式的配置文件，可以代替 CLI 参数，便于管理和复用。

**默认配置文件位置（按优先级）：**

```text
1. 启动目录下的 ngrok.yml
2. $HOME/.config/ngrok/ngrok.yml（Linux/macOS）
3. %HOMEPATH%\.config\ngrok\ngrok.yml（Windows）
4. --config 参数指定的路径
```

**完整的 ngrok.yml 配置：**

```yaml
# ngrok/ngrok.yml — 本项目的 ngrok 配置文件
version: "2"
authtoken: YOUR_AUTHTOKEN

# 日志配置
log: /var/log/ngrok.log
log_format: json

# 区域选择
region: ap

# 隧道定义（可以定义多个隧道）
tunnels:
  # 主 API 隧道：暴露 FastAPI
  ticket-api:
    proto: http
    addr: 8000
    domain: ticket-api.ngrok.app        # 需要付费套餐
    host_header: "ticket.example.com"
    basic_auth:
      - "admin:MySecurePass123"
    ip_restriction:
      allow_cidrs:
        - 0.0.0.0/0                     # 允许所有来源（开发环境）
        # - 203.0.113.0/24              # 或仅允许特定 IP 段
    inspect: true                        # 启用请求检查

  # 支付宝回调专用隧道
  alipay-callback:
    proto: http
    addr: 8000
    domain: alipay-callback.ngrok.app
    host_header: "ticket.example.com"
    # 支付宝回调的 IP 范围
    ip_restriction:
      allow_cidrs:
        - 110.75.0.0/16                  # 支付宝公网 IP 范围
        - 121.0.27.0/24
    inspect: true
    request_header:
      remove: ["X-Forwarded-For"]        # 安全考虑：移除某些头

  # TCP 隧道：用于 SSH
  ssh-tunnel:
    proto: tcp
    addr: 22
    remote_addr: 1.tcp.ngrok.io:23322    # 分配远程端口

  # TLS 隧道：用于自定义 TLS 终止
  tls-api:
    proto: tls
    addr: 8000
    domain: tls-tunnel.ngrok.app
    # 需要提供密钥和证书
    key: /path/to/key.pem
    cert: /path/to/cert.pem

# 启动指定隧道
# 命令行：ngrok start ticket-api alipay-callback
```text

**启动指定隧道：**

```bash
# 启动所有隧道
ngrok start --all

# 启动特定隧道
ngrok start ticket-api

# 启动多个隧道
ngrok start ticket-api alipay-callback
```

### 1.5 Dashboard — 请求检查与重放

ngrok 的本地 Dashboard 运行在 `http://localhost:4040`（默认启用），是一个强大的开发调试工具。

**主要功能：**

```text
主界面：
├── Overview    ← 隧道概览，请求速率图
├── Requests   ← 请求列表（最近 50 个请求的详细信息）
│   ├── Method     ← GET/POST/PUT/DELETE
│   ├── Path       ← 请求路径
│   ├── Status     ← 响应状态码
│   ├── Duration   ← 响应时间
│   └── Timestamp  ← 请求时间戳
│
├── Inspect    ← 请求详情
│   ├── Request
│   │   ├── Headers     ← 完整的请求头
│   │   ├── Query Params← 查询参数
│   │   ├── Body        ← 请求体（JSON 格式化显示）
│   │   └── Raw         ← 原始请求
│   └── Response
│       ├── Headers
│       ├── Body
│       └── Status Code
│
└── Replay     ← 重放请求（最强大的调试功能）
    └── 可以修改任何参数后重新发送
    └── 支持编辑 Headers、Body 等
```

**使用场景：支付回调调试**

```text
场景：支付宝 POST 回调到你的 notify_url

1. 支付宝发送 POST 请求到 https://alipay-callback.ngrok.app/api/v1/payments/notify
2. ngrok Dashboard 在 localhost:4040 捕获这个请求
3. 你可以在 Dashboard 中查看完整的请求内容（headers + body + 签名参数）
4. 使用 Replay 功能，修改参数后重新发送
5. 验证你的回调处理逻辑是否正确
```

**禁用 Dashboard（如果需要）：**

```bash
ngrok http 8000 --inspect=false

# 或在配置文件中：
inspect: false
```text

### 1.6 安全配置

**IP 白名单（allow_cidr）：**

```yaml
# 只允许特定 IP 段访问隧道
tunnels:
  restricted-api:
    proto: http
    addr: 8000
    ip_restriction:
      allow_cidrs:
        - 203.0.113.0/24    ← 只允许公司 VPN 的 IP 段
        - 198.51.100.0/24
```

**基本认证（basic_auth）：**

```yaml
# 访问隧道需要用户名和密码
tunnels:
  secure-api:
    proto: http
    addr: 8000
    basic_auth:
      - "username:password123"
    # 支持多个账号
    # - "user2:pass456"
```text

**OAuth 集成（付费功能）：**

```yaml
# 使用 Google/GitHub/Facebook 等登录保护
tunnels:
  oauth-api:
    proto: http
    addr: 8000
    oauth_provider: google
    oauth_allow_emails:
      - "you@example.com"
    oauth_scopes:
      - "profile"
      - "email"
```

**Webhook 验证：**

```yaml
# ngrok 会自动验证 webhook 请求的签名
tunnels:
  webhook-test:
    proto: http
    addr: 8000
    verify_webhook_provider: stripe    ← 支持 stripe/github/slack 等
    verify_webhook_secret: whsec_xxx
```text

**MTLS（双向 TLS，付费功能）：**

```yaml
# 要求客户端提供证书
tunnels:
  mtls-api:
    proto: tls
    addr: 8000
    mutual_tls: true
    ca_cert: /path/to/ca.crt
```

### 1.7 保留域名与 Edges

**保留域名（Reserved Domains）：**

免费版 ngrok 每次启动会分配不同的随机域名。使用付费套餐后，可以**保留**一个固定的子域名：

```bash
# 在 Dashboard 中创建保留域名
# Dashboard → Domains → Add Domain
# 例如：ticket-api.ngrok.app

# 使用保留域名启动
ngrok http 8000 --domain ticket-api.ngrok.app
```text

**Edges（进阶功能）：**

Edge 是 ngrok 的云端配置层，允许在 ngrok 云侧配置 TLS 终止、IP 限制、OAuth 等，而无需在每个客户端上重复配置：

```

ngrok Edge 的优势：
┌─────────────┐
│  ngrok Edge  │  ← 云侧配置 TLS、认证、路由规则
│  云配置层     │
├─────────────┤
│ ngrok 客户端  │  ← 只需关注本地转发
│ (本地运行)    │
└─────────────┘

```text

```yaml
# 使用 Edge 的 ngrok.yml
version: "2"
authtoken: YOUR_TOKEN
tunnels:
  edge-tunnel:
    proto: http
    addr: 8000
    # 不需要指定 domain，由 Edge 配置决定
```

---

## 2. frp 完全教程

### 2.1 frp 架构

frp（Fast Reverse Proxy）是一个开源的**内网穿透**工具，由服务端（frps）和客户端（frpc）两部分组成。

**与 ngrok 的核心区别：**

- ngrok：商业服务 + 开源的客户端，**服务器由 ngrok 公司提供**
- frp：**完全开源**，需要自己准备一台公网 VPS 运行服务端

```text
frp 架构（需要自备公网服务器）：

                   公网 VPS                          内网服务器
               ┌──────────────┐                 ┌──────────────┐
               │    frps      │                 │    frpc      │
               │   (服务端)    │ ◄──隧道连接───  │   (客户端)    │
               │              │                 │              │
               │  端口: 7000  │ ← 控制连接      │  连接到      │
               │              │                  │  server:7000 │
               │  端口: 8080  │ ← HTTP 代理      │              │
               │  端口: 7500  │ ← Dashboard      │ 代理:        │
               └──────────────┘                 │ localhost:8000│
                     ▲                          └──────────────┘
                     │ HTTPS
                     │
               ┌─────┴──────┐
               │   用户/客户端  │
               └────────────┘
```

**两种连接类型：**

| 连接类型 | 方向 | 用途 |
| ---------- | ------ | ------ |
| **控制连接（Control Connection）** | frpc → frps | 传递控制指令、心跳保活、代理列表 |
| **代理连接（Proxy Connection）** | frpc ↔ frps | 传输实际业务数据（用户请求和响应） |

控制连接建立后，当有外部用户访问 frps 的代理端口时，frps 通过控制连接通知 frpc 建立新的代理连接，完成数据传输。

### 2.2 服务端 frps 配置

**安装 frps：**

```bash
# 下载最新版本（公网 VPS）
wget https://github.com/fatedier/frp/releases/download/v0.58.0/frp_0.58.0_linux_amd64.tar.gz
tar -xzf frp_0.58.0_linux_amd64.tar.gz
cd frp_0.58.0_linux_amd64
sudo cp frps /usr/local/bin/
sudo mkdir -p /etc/frp

# 复制配置
sudo cp frps.toml /etc/frp/
```text

**完整的 frps.toml 配置：**

```toml
# frp/frps.toml — 服务端配置
# 此文件部署在公网 VPS 上

# ── 核心配置 ──
bindPort = 7000                    # 控制连接监听端口（frpc 用这个端口连接）
bindAddr = "0.0.0.0"               # 监听地址

# ── HTTP 代理端口 ──
vhostHTTPPort = 8080               # HTTP 代理监听端口（用户访问这个端口）
vhostHTTPSPort = 8443              # HTTPS 代理监听端口

# ── 认证 ──
auth.method = "token"              # 认证方式（token/oidc）
auth.token = "your-secure-token"   # frpc 连接时使用的密钥（修改为强密码！）

# ── Dashboard ──
dashboardPort = 7500               # Dashboard 端口
dashboardUser = "admin"            # Dashboard 登录用户名
dashboardPwd = "your-admin-password"  # Dashboard 登录密码
dashboardAddr = "0.0.0.0"          # Dashboard 监听地址

# ── 日志 ──
log.to = "/var/log/frps.log"       # 日志文件路径
log.level = "info"                 # 日志级别（trace/debug/info/warn/error）
log.maxDays = 7                    # 日志保留天数

# ── 连接限制 ──
allowPorts = [                     # 允许的代理端口范围
    { start = 8000, end = 8100 },
    { port = 2000 },
    { port = 3000 }
]
maxPortsPerClient = 0              # 每个客户端最大端口数（0=无限制）

# ── 传输 ──
tls.force = false                  # 是否强制使用 TLS 加密 frp 控制连接
transport.tcpMux = true            # 启用 TCP 多路复用（可在一条连接上复用多个代理）

# ── 速率限制（服务端全局） ──
transport.bandwidthLimit = "10MB"  # 服务端全局带宽限制
```

### 2.3 客户端 frpc 配置

**安装 frpc：**

```bash
# 本地开发机器（macOS）
wget https://github.com/fatedier/frp/releases/download/v0.58.0/frp_0.58.0_darwin_amd64.tar.gz
tar -xzf frp_0.58.0_darwin_amd64.tar.gz
cd frp_0.58.0_darwin_amd64
sudo cp frpc /usr/local/bin/
sudo mkdir -p /etc/frp
```text

**完整的 frpc.toml 配置：**

```toml
# frp/frpc.toml — 客户端配置
# 此文件在本地开发机器上运行

# ── 服务端连接 ──
serverAddr = "your-vps-public-ip"      # 公网 VPS 的 IP 地址
serverPort = 7000                      # 对应 frps 的 bindPort

# ── 认证 ──
auth.method = "token"
auth.token = "your-secure-token"       # 必须与 frps 中的 token 一致

# ── 日志 ──
log.to = "/var/log/frpc.log"
log.level = "info"
log.maxDays = 7

# ── TLS ──
transport.tls.enable = false           # 启用 TLS 加密控制连接

# ════════════════════════════════════════════════
# 代理配置
# ════════════════════════════════════════════════

# ── 代理 1：HTTP 隧道（本项目的 API） ──
[[proxies]]
name = "ticket-api"                     # 代理名称（唯一标识）
type = "http"                           # HTTP 代理
localIP = "127.0.0.1"                   # 本地服务 IP
localPort = 8000                        # 本地服务端口（FastAPI 端口）
customDomains = ["ticket.example.com"]  # 通过哪个域名访问此服务
# 用户访问 http://ticket.example.com:8080 → 转发到 localhost:8000
# （DNS 需要将 ticket.example.com 解析到 frps 服务器的公网 IP）

# HTTP 代理特有配置
httpUser = ""                           # 可选：HTTP 基本认证用户名
httpPassword = ""                       # 可选：HTTP 基本认证密码
subdomain = "ticket"                    # 子域名（如 ticket.frps-ip:8080）
locations = ["/api", "/docs"]           # 仅代理指定路径

# ── 代理 2：TCP 隧道（MySQL 远程管理） ──
[[proxies]]
name = "ticket-mysql"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3306                        # 本地 MySQL 端口
remotePort = 23306                      # 远程端口（用户连接 VPS:23306 → 本地 MySQL）

# ── 代理 3：STCP 隧道（安全的点对点通信） ──
[[proxies]]
name = "secret-api"
type = "stcp"                           # 安全的 TCP 隧道（需要 frpc 对 frpc 连接）
localIP = "127.0.0.1"
localPort = 8000
secretKey = "your-shared-secret"        # 共享密钥（连接双方必须一致）
allowUsers = ["user-a", "user-b"]       # 允许访问的用户

# ── 代理 4：XTCP 隧道（P2P 直连，不经过服务器中转） ──
[[proxies]]
name = "p2p-api"
type = "xtcp"                           # P2P 隧道（UDP 打洞）
localIP = "127.0.0.1"
localPort = 8000
secretKey = "another-shared-secret"

# ── 代理 5：WebSocket 专用隧道 ──
[[proxies]]
name = "ticket-ws"
type = "http"
localIP = "127.0.0.1"
localPort = 8000
customDomains = ["ws.ticket.example.com"]
locations = ["/ws/"]                    # 仅转发 WebSocket 路径

# ── 代理 6：静态文件服务器 ──
[[proxies]]
name = "ticket-static"
type = "tcp"
localIP = "127.0.0.1"
localPort = 9000                        # 假设静态文件服务器在 9000 端口
remotePort = 29000

# ── 代理 7：SSH 隧道 ──
[[proxies]]
name = "ticket-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 2222                       # 通过 VPS:2222 连接本地 SSH
```

### 2.4 代理类型详解

frp 支持四种代理类型，各有不同的应用场景：

**TCP 代理：**

最简单的代理类型，将 TCP 端口映射到公网。

```text
用户 → VPS:remotePort → frps → 控制连接通知 frpc → 创建代理连接 → localhost:localPort
```

```toml
[[proxies]]
name = "tcp-example"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8000
remotePort = 28000           # 公网通过 VPS:28000 访问
```text

适用场景：MySQL、SSH、Redis、任意 TCP 协议服务。

**HTTP 代理：**

专门为 HTTP 协议优化，支持域名虚拟主机（多个网站共享 80/8080 端口）。

```toml
[[proxies]]
name = "http-example"
type = "http"
localIP = "127.0.0.1"
localPort = 8000
customDomains = ["www.example.com"]     ← 通过域名区分不同网站

# 同一 VPS:8080 端口可以代理多个 HTTP 服务：
# www.example.com → localhost:8000
# blog.example.com → localhost:3000
```

适用场景：Web 应用、REST API、一切 HTTP 服务。

**STCP 代理（安全 TCP）：**

STCP 需要两个 frpc 客户端：一个提供服务（如运行本项目的本地机器），另一个是消费者（如开发伙伴的机器）。

```text
服务提供方（本地）           frps                 消费者（远程开发伙伴）
┌──────────────┐         ┌────────┐            ┌──────────────┐
│  frpc         │ ◄──── │ frps  │ ────────► │  frpc         │
│  localhost:8000│         └────────┘            │  client       │
└──────────────┘                                 └──────────────┘
  提供者                                         用户访问 localhost:18000
```

提供者配置（`frpc.toml`）：

```toml
[[proxies]]
name = "stcp-api"
type = "stcp"
localIP = "127.0.0.1"
localPort = 8000
secretKey = "shared-secret-123"
allowUsers = ["dev-partner"]   ← 仅允许指定用户访问
```text

消费者配置（`frpc-consumer.toml`）：

```toml
serverAddr = "your-vps-ip"
serverPort = 7000
auth.token = "your-secure-token"

[[visitors]]
name = "stcp-api-visitor"
type = "stcp"
serverName = "stcp-api"           ← 对应提供者的代理名称
secretKey = "shared-secret-123"   ← 必须与提供者一致
bindAddr = "127.0.0.1"
bindPort = 18000                  ← 消费者绑定本地端口，访问 127.0.0.1:18000 即可
```

适用场景：数据库远程管理、开发环境共享、敏感服务只暴露给特定用户。

**XTCP 代理（P2P 直连）：**

XTCP 试图通过 UDP 打洞建立点对点连接，数据**不经过 frps 中转**，延迟更低、带宽更高。

```text
服务提供方 ────────────────────────► 消费者
  (UDP 打洞建立 P2P 通道)
          frps（仅用于信令交换）
```

```toml
# 提供者
[[proxies]]
name = "xtcp-api"
type = "xtcp"
localIP = "127.0.0.1"
localPort = 8000
secretKey = "p2p-secret-key"

# 消费者
[[visitors]]
name = "xtcp-api-visitor"
type = "xtcp"
serverName = "xtcp-api"
secretKey = "p2p-secret-key"
bindAddr = "127.0.0.1"
bindPort = 28000

# 如果 UDP 打洞失败（NAT 类型限制），自动回退到 frps 中转
transport.useEncryption = false   ← P2P 连接可选加密（由应用自己加密）
```text

适用场景：大文件传输、视频流、对延迟敏感的应用。

### 2.5 高级设置

**加密与压缩：**

```toml
[[proxies]]
name = "secure-api"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8000
remotePort = 28000

transport.useEncryption = true    ← 启用 TLS 加密隧道数据
transport.useCompression = true   ← 启用压缩（对 JSON/文本响应效果明显）
```

注意：加密会增加 CPU 开销，压缩适合文本密集型的 API 响应。本项目中的 JSON API 响应可以从压缩中受益。

**健康检查：**

```toml
[[proxies]]
name = "api-with-healthcheck"
type = "http"
localIP = "127.0.0.1"
localPort = 8000
customDomains = ["ticket.example.com"]

# 健康检查配置
healthCheck.type = "http"          ← 健康检查类型（tcp/http）
healthCheck.timeoutSeconds = 3     ← 超时时间
healthCheck.maxFailed = 3          ← 最大失败次数
healthCheck.intervalSeconds = 10   ← 检查间隔
healthCheck.path = "/health"       ← 健康检查路径（HTTP 类型需要）
healthCheck.healthCheckUrl = "http://127.0.0.1:8000/health"
```text

健康检查失败后，frps 会将该代理标记为不可用，不再转发流量到该客户端。

**速率限制（每个代理级别）：**

```toml
[[proxies]]
name = "rate-limited-api"
type = "http"
localIP = "127.0.0.1"
localPort = 8000
customDomains = ["ticket.example.com"]

transport.bandwidthLimit = "1MB"            ← 每个代理的带宽上限
transport.bandwidthLimitMode = "client"     ← 限制方向：client/server/both
```

**启动管理：**

```bash
# 启动服务端
nohup ./frps -c ./frps.toml > /dev/null 2>&1 &

# 或使用 systemd 管理
sudo vim /etc/systemd/system/frps.service

[Unit]
Description=frps Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

# 启动
sudo systemctl enable frps
sudo systemctl start frps
sudo systemctl status frps
```text

```bash
# 启动客户端
nohup ./frpc -c ./frpc.toml > /dev/null 2>&1 &

# 或 systemd
sudo vim /etc/systemd/system/frpc.service

[Unit]
Description=frpc Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

sudo systemctl enable frpc
sudo systemctl start frpc
sudo systemctl status frpc
```

### 2.6 frp Dashboard

frps 内置了一个 Web Dashboard，可以通过浏览器查看 frp 的运行状态。

**访问 Dashboard：**

```text
浏览器打开 http://VPS_PUBLIC_IP:7500
输入 frps.toml 中配置的 dashboardUser/dashboardPwd
```

**Dashboard 功能：**

```text
概览页：
├── 服务端基本信息（版本、运行时间、进程 PID）
├── 代理统计（总数/在线/离线）
├── 流量统计（总入站/出站、当前速率）
└── 客户端列表

代理页：
├── 所有代理列表
│   ├── 名称
│   ├── 类型（TCP/HTTP/STCP）
│   ├── 状态（online/offline）
│   ├── 本地地址
│   ├── 远程端口/域名
│   └── 流量统计

客户端页：
├── 所有客户端连接
│   ├── 客户端的 IP 地址
│   ├── 版本号
│   ├── 代理数量
│   ├── Ping 延迟
│   └── 最后活动时间

错误日志：
├── 实时错误输出
├── 连接失败记录
└── frps 日志
```

---

## 3. 项目实战：为票务系统配置支付宝回调

### 3.1 需求分析

本项目的票务系统使用支付宝支付。支付宝支付的异步通知流程：

```text
1. 用户在浏览器中完成支付宝支付
2. 支付宝服务端向 notify_url 发送 POST 请求通知支付结果
3. 本地服务需要更新订单状态为"已支付"
4. 本地服务需要返回 "success" 给支付宝（不返回则重试）

问题：开发阶段，本地 localhost 无法收到支付宝的 HTTP 回调。

解决方案：
使用 ngrok 或 frp 将本地的 FastAPI 服务暴露到公网，
支付宝能够 POST 到 ngrok/frp 的公网地址。
```

### 3.2 使用 ngrok 方案

```bash
# 启动隧道（指定域名以便支付宝配置）
ngrok http 8000 \
  --domain=alipay-ticket.ngrok.app \
  --region=ap \
  --basic-auth="alipay:callback123" \
  --host-header="ticket.example.com"
```text

然后修改项目配置：

```python
# app/config.py 中
ALIPAY_NOTIFY_URL = "https://alipay-ticket.ngrok.app/api/v1/payments/notify"
```

支付宝主动通知流程：

```text
支付宝 → POST https://alipay-ticket.ngrok.app/api/v1/payments/notify
                ↓
         ngrok 隧道
                ↓
         localhost:8000/api/v1/payments/notify
                ↓
         解析参数 → 验证签名 → 更新订单状态 → 返回 "success"
```

### 3.3 使用 frp 方案

```toml
# frpc.toml — 支付宝回调代理
[[proxies]]
name = "ticket-alipay"
type = "http"
localIP = "127.0.0.1"
localPort = 8000
customDomains = ["ticket.your-domain.com"]  ← 需要 DNS 指向 frps 服务器 IP
locations = ["/api/v1/payments/"]           ← 仅支付宝相关路径

# 额外配置
transport.useEncryption = true
transport.useCompression = true
```text

配置 DNS：

```bash
# 将域名 A 记录指向 frps 服务器的公网 IP
# ticket.your-domain.com → A → YOUR_VPS_IP
```

然后在项目配置中：

```python
ALIPAY_NOTIFY_URL = "http://ticket.your-domain.com:8080/api/v1/payments/notify"
# 注：8080 是 frps 的 vhostHTTPPort
```text

### 3.4 同时使用 nginx + frp 的生产架构

```

                        公网 VPS
                    ┌──────────────┐
                    │    nginx     │
                    │  (端口 80/443)│ ← 用户 + 支付宝直接访问
                    │      ↓       │
                    │    frps      │
                    │  (端口 7000)  │ ← 控制连接
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │    frpc      │
                    │   (本地运行)  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  FastAPI:8000│
                    │ 票务系统      │
                    └──────────────┘

```text

---

## 4. ngrok vs frp 对比

### 4.1 十维对比表

| 维度 | ngrok | frp |
| ------ | ------- | ----- |
| **开源协议** | 部分开源（客户端开源，服务端闭源） | 完全开源（MIT 协议） |
| **是否需要公网服务器** | 不需要（使用 ngrok 云服务） | 需要自备公网 VPS |
| **设置复杂度** | 极低（brew install 即可使用） | 中等（需要部署 frps + DNS 配置） |
| **带宽** | 免费版约 1MB/s 限制，付费版按流量计费 | 取决于 VPS 带宽（完全自控） |
| **延迟** | 取决于 ngrok 边缘节点距离（50-300ms） | 取决于 VPS 距离 |
| **免费功能** | 1 个在线隧道，4 个/分钟，随机域名 | 完全免费，无任何限制 |
| **付费价格** | ~$8-$50/月（按套餐）| 仅需 VPS 费用（~$5-20/月） |
| **HTTP 隧道** | 支持（自动 HTTPS） | 支持（需要额外配置） |
| **TCP 隧道** | 支持 | 支持 |
| **P2P 隧道** | 不支持 | 支持（XTCP 模式） |
| **自定义域名** | 付费版支持 | 完全支持（DNS 指向 VPS） |
| **Dashboard** | localhost:4040（请求检查/重放） | frps:7500（服务器监控） |
| **认证支持** | Basic Auth / OAuth / OIDC | Token 认证 |
| **IP 白名单** | 支持 | 需要配合 iptables |
| **WebSocket** | 天然支持 | 支持（配置 proxy 即可） |
| **TLS 加密** | 自动（免费 HTTPS） | 需自行配置或使用 TLS 模式 |
| **请求重放** | 支持（Dashboard 内置） | 不支持 |
| **速率限制** | 不内置 | 支持（带宽限制） |
| **多路复用** | 自动 | TCP Mux |
| **数据压缩** | 不内置 | 支持 |
| **健康检查** | 不内置 | 支持 |
| **企业特性** | 保留域名、Edges、MTLS | 无内置企业特性 |
| **适用场景** | 开发调试、快速原型、演示 | 生产环境、团队使用、自定义部署 |

### 4.2 选择建议

**选择 ngrok 的场景：**

- 个人开发者，快速调试支付宝/微信回调
- 需要给客户做远程演示
- 不希望管理服务器
- 需要请求检查/重放功能（Dashboard 太方便了）
- 临时使用，用完即弃

**选择 frp 的场景：**

- 团队长期需要内网穿透
- 已经有公网 VPS
- 对带宽有要求（音视频、大文件）
- 需要完全控制数据链路
- 需要 P2P 直连（XTCP）
- 生产环境需要稳定的穿透方案

**最佳实践：开发用 ngrok，生产用 frp。**

```bash
# 开发阶段（快速启动）
alias dev-ngrok="ngrok http 8000 --domain=dev-ticket.ngrok.app --region=ap"

# 生产 stage（使用 frp systemd 服务）
alias prod-frps="ssh vps 'sudo systemctl start frps'"
alias prod-frpc="sudo systemctl start frpc"
```

---

## 5. 常见问题与排查

### 5.1 ngrok 连接失败

```bash
# 1. 检查认证
ngrok config check

# 2. 检查网络
ping xxxxx.ngrok.io

# 3. 查看日志
tail -f ~/.ngrok/ngrok.log

# 4. 常见错误
ERR_NGROK_201  → 认证失败，重新运行 ngrok config add-authtoken
ERR_NGROK_302  → 域名已被使用，换一个后端或等待几分钟
ERR_NGROK_105  → 超过免费版隧道数量上限（每分钟最多 4 个）
ERR_NGROK_501  → 保留域名不可用，检查是否在你的账号下
```text

### 5.2 frp 连接失败

```bash
# 1. 检查 frps 是否运行
ssh vps "sudo systemctl status frps"

# 2. 检查防火墙
ssh vps "sudo ufw status"
# 放行必要端口：7000, 8080, 7500, 2222, 28000 等
ssh vps "sudo ufw allow 7000/tcp"
ssh vps "sudo ufw allow 8080/tcp"
ssh vps "sudo ufw allow 7500/tcp"

# 3. 测试 frpc 到 frps 的连接
ssh vps "nc -l 7000"        # 在 VPS 上监听
nc -v YOUR_VPS_IP 7000      # 在本地尝试连接

# 4. 检查 token 是否一致
grep "token" /etc/frp/frps.toml
grep "token" /etc/frp/frpc.toml

# 5. 查看 frps 日志
ssh vps "tail -f /var/log/frps.log"

# 6. 常见 frpc 日志错误
"login to server failed" → 服务器地址或端口不对
"auth token incorrect"   → token 不匹配
"proxy name already exist" → 代理名称重复
"can't find suitable port" → 远程端口已被占用或不在 allowPorts 范围内
```

### 5.3 安全注意事项

```text
1. 不要将 ngrok/frp 的 URL 泄露到公开场合
2. ngrok 免费版所有人都知道你的子域名（可能被扫描）
3. frp 的 token 使用强密码（至少 16 位，含大小写 + 数字 + 符号）
4. frps 的 Dashboard 不要暴露到公网（改为 localhost:7500 并用 SSH 隧道访问）
5. 使用完毕后及时关闭隧道
6. 不要在隧道中传输敏感数据（如密码、信用卡号）—— 除非启用了 TLS 加密
7. 定期检查 ngrok Dashboard / frps 日志，发现异常 IP 及时加入黑名单
8. 生产环境应使用 VPN + frp 的组合方案
```

---

## 6. ngrok.yml 和 frp README

### 6.1 本项目的 ngrok.yml 完整配置

```yaml
# ngrok/ngrok.yml — 票务系统 ngrok 配置
version: "2"
authtoken: ${NGROK_AUTH_TOKEN}         # 建议使用环境变量
region: ap

tunnels:
  # API 主入口
  ticket-api:
    proto: http
    addr: 8000
    domain: ticket-api.ngrok.app
    host_header: "ticket.example.com"
    basic_auth:
      - "admin:${NGROK_PASSWORD}"
    inspect: true

  # 支付宝回调
  alipay-notify:
    proto: http
    addr: 8000
    domain: alipay-notify.ngrok.app
    host_header: "ticket.example.com"
    inspect: true

# 启动：ngrok start --all
```text

### 6.2 本项目的 frp 完整配置

**frps.toml：**

```toml
# Production frps.toml
bindPort = 7000
vhostHTTPPort = 8080
vhostHTTPSPort = 8443

auth.method = "token"
auth.token = "${FRP_TOKEN}"

dashboardPort = 7500
dashboardUser = "admin"
dashboardPwd = "${DASHBOARD_PWD}"

log.to = "/var/log/frps.log"
log.level = "info"
```

**frpc.toml：**

```toml
# Local frpc.toml
serverAddr = "${FRP_SERVER_IP}"
serverPort = 7000

auth.method = "token"
auth.token = "${FRP_TOKEN}"

[[proxies]]
name = "ticket-api"
type = "http"
localIP = "127.0.0.1"
localPort = 8000
customDomains = ["ticket.your-domain.com"]
transport.useCompression = true

[[proxies]]
name = "ticket-mysql"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3306
remotePort = 23306
```text

### 6.3 frp/README.md

```markdown
# frp 内网穿透配置

## 目录结构

```

frp/
├── frps.toml        # 服务端配置（部署到公网 VPS）
├── frpc.toml        # 客户端配置（本地开发机器）
└── README.md        # 本文件

```text

## 前置条件

1. 一台具有公网 IP 的 Linux VPS
2. 一个域名（可选，HTTP 代理需要）

## 部署步骤

### 1. 服务端部署（VPS）

```bash
# 上传 frps 二进制和配置
scp frps frps.toml root@your-vps:/root/

# 在 VPS 上
mv frps /usr/local/bin/
mkdir -p /etc/frp
mv frps.toml /etc/frp/

# 创建 systemd 服务
cat > /etc/systemd/system/frps.service << 'SERVICE'
[Unit]
Description=frps Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
SERVICE

# 启动
systemctl enable frps
systemctl start frps
systemctl status frps
```

### 2. 防火墙开放端口

```bash
ufw allow 7000/tcp    # 控制连接
ufw allow 8080/tcp    # HTTP 代理
ufw allow 7500/tcp    # Dashboard
ufw allow 23306/tcp   # MySQL 隧道
```text

### 3. 客户端启动（本地）

```bash
# 编辑 frpc.toml 填入 VPS IP 和 token
vim frpc.toml

# 启动
chmod +x ../scripts/start-frpc.sh
./frpc -c frpc.toml
```

### 4. DNS 配置（HTTP 代理需要）

将域名 A 记录指向 VPS 的公网 IP。

## 验证

```bash
# 查看 Dashboard
curl http://VPS_IP:7500

# 测试 API 代理
curl -H "Host: ticket.your-domain.com" http://VPS_IP:8080/api/v1/events
```text

```

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `ngrok/` | ngrok 配置文件（开发环境内网穿透） |
| `frp/` | frp 客户端/服务端配置文件（生产环境内网穿透） |
| `app/config.py` | `ALLOWED_HOSTS` 配置（ngrok/frp 域名加入白名单） |
| `app/main.py` | TrustedHostMiddleware 允许 `*.ngrok.io`、`*.frp.com` |

**TrustedHost 配置**（在 `app/main.py` 中）：

```python
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "127.0.0.1", "*.ngrok.io", "*.frp.com", "*.app"],
)
```text

## 本章总结

**核心知识点回顾：**

1. **ngrok**：商业内网穿透服务，不需要自建服务器，开箱即用，适合开发和调试
2. **frp**：完全开源的内网穿透工具，需要自建 VPS 服务端，适合生产环境
3. **隧道类型**：HTTP（Web 应用） > TCP（数据库、SSH） > STCP（安全共享） > XTCP（P2P 直连）
4. **安全**：配置认证（Basic Auth / Token）、IP 白名单、启用加密和压缩
5. **ngrok Dashboard 的请求重放**是调试支付回调的利器
6. **开发用 ngrok，生产用 frp** 是最佳实践

---

**项目检查清单：**

- [ ] ngrok 安装并配置 Authtoken
- [ ] ngrok http 8000 测试通过
- [ ] ngrok Dashboard（localhost:4040）可用
- [ ] ngrok.yml 配置完成（含 Basic Auth）
- [ ] frp 服务端（VPS）部署完成
- [ ] frp 客户端（本地）配置完成
- [ ] frps Dashboard 可访问
- [ ] 支付宝回调隧道配置完成
- [ ] HTTPS 和加密已启用
- [ ] 手机浏览器验证通过
