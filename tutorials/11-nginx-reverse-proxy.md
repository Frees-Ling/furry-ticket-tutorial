# 第11章：Nginx 反向代理/SSL/限流 — 零基础到精通

## 本章学习目标

本章将从零开始全面学习 Nginx，涵盖架构原理、配置语法、反向代理、SSL/TLS 安全配置、速率限制、WebSocket 代理、负载均衡等核心主题。最终将为本项目的票务系统 FastAPI 应用配置完整的 Nginx 反向代理。

**学习路径：**

1. Nginx 架构与进程模型
2. Nginx 配置语法与指令上下文
3. Location 匹配规则与优先级（含证明）
4. 反向代理 proxy_pass 系列指令精讲
5. WebSocket 代理配置
6. SSL/TLS 完整配置（含 HSTS、OCSP Stapling）
7. 负载均衡 upstream 配置
8. 漏桶（Leaky Bucket）速率限制（含数学证明）
9. 安全响应头配置
10. 项目完整配置示例

---

## 1. Nginx 架构与进程模型

### 1.1 单进程 vs 多进程

Nginx 采用 **master-worker** 多进程架构，与传统的 Apache prefork 模型有本质区别。

```text
┌───────────────┐
│   Master      │  ← 读取配置、绑定端口、管理 worker
│   (root)      │
├───────┬───────┤
│       │       │
▼       ▼       ▼
Worker1 Worker2 WorkerN
(nobody) (nobody) (nobody)
```

**Master 进程（root 权限）：**

- 读取并验证配置文件
- 绑定监听端口（需要 root 权限绑定 80/443）
- 启动/停止/重载 worker 进程
- 执行平滑升级（`nginx -s reload` 无缝切换）
- 不处理任何客户端请求

**Worker 进程（nobody 权限）：**

- 处理所有客户端请求
- 数量由 `worker_processes` 指令控制，通常设置为 CPU 核心数（`auto`）
- 每个 worker 是单线程的，但使用异步非阻塞 I/O 模型处理数千并发连接

### 1.2 事件驱动模型（Event-Driven）

Nginx 的 worker 不使用多线程，而是使用操作系统提供的高效 I/O 多路复用机制：

| 平台 | 使用的机制 | 说明 |
| ------ | ----------- | ------ |
| Linux | **epoll** | 最高效的 I/O 多路复用，O(1) 复杂度 |
| macOS | **kqueue** | BSD 系高效事件通知接口 |
| Windows | **IOCP** | Windows 下的 I/O 完成端口 |
| 通用 | poll / select | 兼容模式，性能较低 |

```text
传统 Apache 模型（每个连接一个线程）：
┌─────┐┌─────┐┌─────┐┌─────┐
│Thread││Thread││Thread││Thread│  ← 5000 连接 = 5000 线程
└─────┘└─────┘└─────┘└─────┘
内存消耗 ≈ 5000 × 2MB = 10GB

Nginx epoll 模型：
┌──────────────────────┐
│     Worker(单线程)     │  ← 5000 连接 = 1 线程
│  epoll_wait() 循环     │
│  ↓ 有事件才处理         │
└──────────────────────┘
内存消耗 ≈ 5000 × 几 KB = 几十 MB
```

**epoll 的工作流程：**

```text
1. epoll_create()    → 创建 epoll 实例
2. epoll_ctl(ADD)    → 注册 socket 和关心的事件（读/写）
3. epoll_wait()      → 阻塞等待，有事件发生时返回
4. 处理有事件的 socket
5. 回到第 3 步
```

伪代码表示：

```c
// Nginx worker 主循环（简化）
while (1) {
    events = epoll_wait(epfd, event_list, max_events, timeout);
    for (i = 0; i < events; i++) {
        handle_event(event_list[i]);  // 读/写/accept
    }
}
```text

### 1.3 为什么 Nginx 可以处理高并发？

**关键不在于快，而在于少——不等待任何东西。**

一个传统 web 服务器在处理请求时可能是这样的：

```

1. accept() 客户端连接
2. recv() 读取 HTTP 请求头 → **阻塞等待客户端发送**
3. 解析请求，计算
4. sendfile() 发送静态文件 → **阻塞等待磁盘 I/O**
5. 代理到后端 → **阻塞等待后端响应**
6. close() 连接

```text

在等待期间，线程被白白占用。而 Nginx 的做法是：

```

1. accept() → 注册 EPOLLIN，回到 epoll_wait
2. 数据到达 → epoll 通知 → recv() → 解析完成
3. 如果是代理 → connect() 后端 → 注册 EPOLLOUT
4. 后端连接建立 → epoll 通知 → send() 请求 → 注册 EPOLLIN
5. 后端数据到达 → epoll 通知 → recv() → 发给客户端 → 注册 EPOLLOUT
6. 客户端可写 → epoll 通知 → send()

```text

**没有一行代码在"等待"——所有操作都是事件驱动的。** 这就是 Nginx 能在一个单线程 worker 中处理数万并发连接的秘密。

---

## 2. Nginx 配置语法与指令上下文

### 2.1 配置层次结构

Nginx 配置基于**指令**（directive），每个指令属于特定的**上下文**（context）。上下文形成严格的层次结构：

```

main (全局上下文)
├── events { ... }          ← 事件模块配置
├── http { ... }            ← HTTP 协议配置
│   ├── upstream { ... }    ← 后端服务器组
│   ├── server { ... }      ← 虚拟主机
│   │   ├── location { ... } ← URL 匹配（可嵌套）
│   │   │   └── if { ... }  ← 条件判断
│   │   └── if { ... }
│   └── server { ... }
└── mail { ... }            ← 邮件代理（较少使用）

```text

**指令继承规则：** 内层上下文可以重写（override）外层同名的指令，但并非所有指令都支持继承。

```

http {
    proxy_read_timeout 60s;     ← 默认值，所有 server 继承

    server {
        listen 80;
        # 继承 proxy_read_timeout = 60s

        location /api/ {
            proxy_read_timeout 120s;  ← 只对这个 location 生效
            # ...
        }
    }

    server {
        listen 443 ssl;
        # 继承 proxy_read_timeout = 60s（不受上面影响）
    }
}

```text

### 2.2 核心指令速查

| 指令 | 上下文 | 作用 |
| ------ | -------- | ------ |
| `worker_processes` | main | Worker 进程数，通常设为 auto |
| `worker_connections` | events | 每个 worker 最大连接数 |
| `include` | 任意 | 包含其他配置文件 |
| `listen` | server | 监听端口和协议 |
| `server_name` | server | 虚拟主机域名 |
| `location` | server | URL 路径匹配块 |
| `root` | http/server/location | 静态文件根目录 |
| `proxy_pass` | location | 反向代理目标 |
| `upstream` | http | 后端服务器组定义 |
| `error_page` | http/server/location | 自定义错误页面 |

### 2.3 变量系统

Nginx 提供了丰富的内置变量，可以在配置中引用：

**HTTP 请求相关变量：**

```

$host                  ← 请求的 Host 头（不含端口）
$http_name            ← 任意请求头，如 $http_user_agent
$remote_addr          ← 客户端 IP 地址
$remote_port          ← 客户端端口
$request_method       ← 请求方法（GET/POST/PUT/DELETE）
$request_uri          ← 完整原始 URI（含查询参数）
$uri                  ← 标准化后的 URI（不含参数）
$args / $query_string ← 查询参数部分
$arg_name             ← 特定查询参数的值

```text

**代理相关变量：**

```

$proxy_add_x_forwarded_for ← $remote_addr 追加到已有 X-Forwarded-For
$scheme                    ← HTTP 或 HTTPS
$server_name              ← 当前 server 块的 server_name
$upstream_addr            ← 后端服务器地址
$upstream_status          ← 后端响应状态码
$upstream_response_time   ← 后端响应时间

```text

**SSL 相关变量：**

```

$ssl_protocol    ← TLS 协议版本（TLSv1.2/TLSv1.3）
$ssl_cipher      ← 协商的加密套件
$ssl_server_name ← SNI 中的服务器名

```text

---

## 3. Location 匹配规则（核心难点）

### 3.1 六种匹配方式

location 是 Nginx 配置中最重要也最复杂的指令。理解 location 的匹配顺序是 Nginx 配置的**必修课**。

```nginx
# 语法形式
location [=|^~|~*|~] /pattern { ... }
```

| 修饰符 | 类型 | 说明 |
| -------- | ------ | ------ |
| `=` | **精确匹配** | URI 必须完全等于 pattern |
| `^~` | **前缀匹配 + 拒绝正则** | 匹配后不再检查正则 location |
| `~*` | **正则匹配（不区分大小写）** | 按配置文件中的出现顺序 |
| `~` | **正则匹配（区分大小写）** | 按配置文件中的出现顺序 |
| 无修饰符 | **普通前缀匹配** | 最长的匹配胜出 |
| `@` | **命名 location** | 不能嵌套，用于内部重定向 |

### 3.2 匹配顺序（重要，务必理解）

Nginx 按照以下流程匹配 location：

```text
Step 1: 对所有 = 精确匹配进行匹配
        → 如果命中，立即返回，不再继续
        ↓ 未命中

Step 2: 对所有无修饰符的普通前缀匹配、^~ 前缀匹配进行匹配
        → 记录最长匹配的 location（记为 longest_match）
        ↓

Step 3: 如果 longest_match 是 ^~ 修饰符
        → 立即使用 longest_match，不再继续
        ↓ 不是 ^~

Step 4: 按出现顺序检查所有正则 location (~ 和 ~*)
        → 第一个匹配成功的立即使用，不再继续
        ↓ 没有正则命中

Step 5: 使用 longest_match（最长前缀匹配）
        → 如果没有任何匹配，返回 404

总结口诀：
"精确优先，前缀次之，正则再次之，前缀兜底"
更精确地说：
"= 最高，^~ 阻断正则，正则顺序优先，前缀最长兜底"
```

### 3.3 匹配示例与证明

假设以下配置：

```nginx
# 精确匹配
location = / {                        # LocA: 精确 /
    return 200 "LocA: exact /";
}

# 前缀匹配 + 拒绝正则
location ^~ /static/ {                # LocB: ^~ /static/
    return 200 "LocB: ^~ /static/";
}

# 正则匹配（不区分大小写）
location ~* \.(jpg|png|gif)$ {        # LocC: ~* .(jpg|png|gif)$
    return 200 "LocC: ~* image";
}

# 正则匹配（区分大小写）
location ~ /api/admin {               # LocD: ~ /api/admin
    return 200 "LocD: ~ /api/admin";
}

# 普通前缀匹配
location / {                          # LocE: 前缀 /
    return 200 "LocE: prefix /";
}

# 普通前缀匹配
location /api/ {                      # LocF: 前缀 /api/
    return 200 "LocF: prefix /api/";
}

# 普通前缀匹配
location /api/v1/ {                   # LocG: 前缀 /api/v1/
    return 200 "LocG: prefix /api/v1/";
}
```text

**不同请求的匹配结果：**

| 请求 URI | 匹配结果 | 匹配过程 |
| ---------- | --------- | --------- |
| `/` | **LocA** | Step 1 精确匹配 `=`，立即返回 |
| `/index.html` | **LocE** | 无 = 匹配；前缀最长是 `/`(LocE)；无正则匹配 `/index.html` → LocE |
| `/static/js/app.js` | **LocB** | Step 2 前缀匹配到 `/static/`(LocB) 是 `^~`；Step 3 直接使用 LocB，跳过正则 |
| `/image/photo.jpg` | **LocC** | Step 2 最长前缀是 `/`(LocE) 或 `/image/`(无)；Step 4 正则 `~* \.(jpg)$` 匹配；结果 LocC |
| `/api/admin/users` | **LocD** | Step 2 最长前缀匹配 `LocG /api/v1/` 不对（v1）；实际最长是 LocF `/api/`。但等等！Step 4 正则 `~ /api/admin` 匹配 → LocD |
| `/api/v1/orders` | **LocG** | Step 2 最长前缀是 `/api/v1/`(LocG)；Step 4 无正则匹配这个 path；Step 5 使用 LocG |
| `/api/v1/users` | **LocG** | 同上，最长前缀 LocG |
| `/api/events/1` | **LocF** | Step 2 最长前缀是 `/api/`(LocF)；Step 4 检查正则，`/api/events/1` 不以图片结尾也不包含 admin → 无命中；Step 5 使用 LocF |

### 3.4 常见误区

**误区 1：认为 location 顺序决定前缀匹配优先级**

```nginx
# ❌ 错误观念：写在前面就更优先
location /api/v1/ { ... }  ← 并不比下面的优先级高
location /api/ { ... }     ← 匹配时比较的是长度，不按顺序
```

实际：Nginx 会扫描所有前缀 location，选择**最长匹配**，与配置顺序无关。只有正则 location 才按照配置顺序匹配。

**误区 2：location / 能匹配所有请求**

```text
location / { ... }  ← 它确实匹配所有请求，但优先级最低
                     只要有更长的前缀或任何正则命中，就不会用它
```

**误区 3：多个正则匹配时取第一个**

```text
# 正确。正则 location 按照配置顺序匹配，返回第一个命中的
location ~ ^/api/orders { ... }
location ~ ^/api/     { ... }  ← 这个永远不会被用于 /api/orders
```

---

## 4. 反向代理 — proxy_pass 系列指令

### 4.1 proxy_pass（核心！）

`proxy_pass` 是反向代理中最关键也最容易出错的指令。它的行为取决于**是否带 URI**。

```nginx
# 形式 1：不带 URI（只带 host:port）
location /api/ {
    proxy_pass http://backend;       ← 正确做法：不带 URI
}
# 客户端请求 /api/v1/orders
# → 后端收到 /api/v1/orders（保留原路径）

# 形式 2：带 URI（host:port + 路径）
location /api/ {
    proxy_pass http://backend/;      ← 带 URI（注意末尾的 /）
}
# 客户端请求 /api/v1/orders
# → 后端收到 /v1/orders（location 匹配部分被替换为 URI）
```text

**核心规则（面试高频考点）：**

```

请求 URI = /api/v1/orders
location = /api/

不带 URI 的 proxy_pass：→ <http://backend/api/v1/orders>
                        （原样传递）

带 URI 的 proxy_pass  /：→ <http://backend/v1/orders>
                        （/api/ 被替换为 /）

带 URI 的 proxy_pass /prefix/：→ <http://backend/prefix/v1/orders>

```text

**具体证明：**

```nginx
# 场景 A：不带 URI
location /api/ {
    proxy_pass http://127.0.0.1:8000;
    # GET /api/v1/orders → http://127.0.0.1:8000/api/v1/orders
}

# 场景 B：带上 URI（/ 替换 /api/）
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
    # GET /api/v1/orders → http://127.0.0.1:8000/v1/orders
}

# 场景 C：替换为不同路径
location /api/ {
    proxy_pass http://127.0.0.1:8000/newapi/;
    # GET /api/v1/orders → http://127.0.0.1:8000/newapi/v1/orders
}

# 场景 D：正则 location + proxy_pass 不带 URI
location ~ ^/api/(v[12])/ {
    proxy_pass http://127.0.0.1:8000;
    # GET /api/v1/orders → http://127.0.0.1:8000/api/v1/orders
}

# 场景 E：正则 location + proxy_pass 带 URI（必须带捕获组）
location ~ ^/api/(v[12])/ {
    proxy_pass http://127.0.0.1:8000/$1/;
    # GET /api/v1/orders → http://127.0.0.1:8000/v1/orders
}
```

### 4.2 proxy_set_header — 传递客户端信息

反向代理的经典问题是：后端应用不知道客户端的真实 IP 和协议。

```nginx
location / {
    proxy_pass http://backend;

    # 必须：传递原始 Host
    proxy_set_header Host $host;

    # 必须：传递真实客户端 IP
    proxy_set_header X-Real-IP $remote_addr;

    # 必须：如果有多级代理，追加 IP 链
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # 建议：传递原始协议（HTTP/HTTPS）
    proxy_set_header X-Forwarded-Proto $scheme;

    # 可选：传递请求方法（某些框架需要用）
    proxy_set_header X-Forwarded-Method $request_method;
}
```text

**$proxy_add_x_forwarded_for 和 $http_x_forwarded_for 的区别：**

```

第一次代理：
$remote_addr = 203.0.113.1（真实客户端 IP）
$http_x_forwarded_for = ""（请求头中没有）
$proxy_add_x_forwarded_for = "203.0.113.1"

第二次代理（如果有多层）：
$remote_addr = 10.0.0.1（上一层代理 IP）
$http_x_forwarded_for = "203.0.113.1"（由上一层设置）
$proxy_add_x_forwarded_for = "203.0.113.1, 10.0.0.1"

```text

### 4.3 proxy_buffering — 缓冲机制

Nginx 从后端收到响应后，可以先将响应数据缓冲到内存（或临时文件），然后再发送给客户端。

```nginx
location / {
    proxy_pass http://backend;

    # 启用/禁用缓冲（默认 on）
    proxy_buffering on;

    # 缓冲区数量，每个缓冲区大小
    proxy_buffers 8 8k;          ← 8 个缓冲区，每个 8KB

    # 响应头部分缓冲区大小
    proxy_buffer_size 4k;        ← 默认等于一页内存大小

    # 忙缓冲区上限（正在发送给客户端时允许占用的缓冲总量）
    proxy_busy_buffers_size 16k; ← 通常设为 proxy_buffers 单个大小的 2 倍

    # 临时文件最大大小
    proxy_max_temp_file_size 1024m;

    # 临时文件写入时的数据积累量
    proxy_temp_file_write_size 32k;
}
```

**缓冲机制的意义：**

```text
无缓冲时：
客户端 ←[慢]→ Nginx ←[快]→ 后端
如果客户端网速慢，Nginx 需要保持到后端的连接一直开着，
等待客户端慢慢读取，后端连接资源被大量浪费。

有缓冲时：
客户端 ←[慢]→ Nginx(buf) ←[快]→ 后端
Nginx 快速从后端读完整响应到缓冲区，释放后端连接。
然后以客户端的速度慢慢发送缓冲区的数据。
```

**何时关闭缓冲（proxy_buffering off）：**

- 需要实时响应的长连接（如 SSE、Server-Sent Events）
- 需要立即看到输出的流式响应
- 后端的响应数据量很小（关闭可以减少延迟）

### 4.4 proxy_cache — 缓存加速

```nginx
# 1. 定义缓存路径和参数（http 上下文）
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m
                 max_size=1g inactive=60m use_temp_path=off;

# 2. 在 location 中启用
location / {
    proxy_cache mycache;             ← 使用 mycache 缓存区
    proxy_cache_key "$scheme$request_method$host$request_uri";  ← 缓存键

    # 缓存有效期
    proxy_cache_valid 200 302 10m;   ← 200/302 响应缓存 10 分钟
    proxy_cache_valid 404 1m;        ← 404 响应缓存 1 分钟
    proxy_cache_valid any 1m;        ← 其他状态码缓存 1 分钟

    # 跳过缓存的条件
    proxy_cache_bypass $http_cache_control;
    proxy_no_cache $http_pragma $http_authorization;

    # 缓存状态变量（可用于日志统计）
    add_header X-Cache-Status $upstream_cache_status;

    proxy_pass http://backend;
}
```text

**缓存参数详解：**

```

keys_zone=mycache:10m → 缓存元数据使用的共享内存 10MB（约可存 8 万个 key）
levels=1:2           → 缓存文件目录结构 /var/cache/nginx/c/29/xxx
                       （防止单个目录文件过多）
max_size=1g          → 缓存数据最大 1GB
inactive=60m         → 60 分钟内未被访问的缓存将被清理
use_temp_path=off    → 直接写入缓存目录，不经过临时目录

```text

### 4.5 proxy_timeout — 超时控制

```nginx
location / {
    proxy_pass http://backend;

    # 与后端建立连接的超时时间
    proxy_connect_timeout 10s;    ← 默认 60s，设为较短值避免等待死连接

    # 从后端读取响应的超时时间（两次成功读取之间的间隔）
    proxy_read_timeout 60s;       ← 默认 60s，长连接场景可适当调大

    # 向后端发送请求的超时时间（两次成功写入之间的间隔）
    proxy_send_timeout 60s;       ← 默认 60s
}
```

**超时参数的合理设置建议：**

```text
API 场景：
  proxy_connect_timeout 5s;    → 连接超时短，快速失败切换
  proxy_read_timeout 30s;      → 常规 API 响应足够
  proxy_send_timeout 30s;

文件上传场景：
  proxy_connect_timeout 10s;
  proxy_read_timeout 300s;     → 大文件上传后处理需要更长时间
  proxy_send_timeout 300s;     → 上传大文件需要更长时间

WebSocket 场景（见下一节）：
  proxy_read_timeout 60s;      → 需要更长超时
```

---

## 5. WebSocket 代理

### 5.1 HTTP 升级机制

WebSocket 协议通过 HTTP Upgrade 机制从标准 HTTP 连接升级而来：

```text
客户端 → 服务端：

GET /ws/chat HTTP/1.1
Host: example.com
Upgrade: websocket                    ← 请求升级协议
Connection: Upgrade                   ← 告诉代理不要转发这个连接
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

服务端 → 客户端：

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### 5.2 Nginx WebSocket 反向代理配置

要让 Nginx 正确地代理 WebSocket 连接，关键的两行配置是：

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```text

**这两行为什么是必须的？**

`$http_upgrade` 是 Nginx 内置变量，其值取自请求头 `Upgrade`。如果客户端请求中带有 `Upgrade: websocket`，则 `$http_upgrade` 的值为 `websocket`；如果没有（普通 HTTP 请求），则为空。

而 `proxy_set_header Connection "upgrade"` 将 Connection 头固定为 `upgrade`。

**完整的 WebSocket 代理配置：**

```nginx
server {
    listen 80;
    server_name ticket.example.com;

    # HTTP API 代理
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket 代理：实时座位更新
    location /ws/ {
        proxy_pass http://127.0.0.1:8000;

        # HTTP/1.1 是必须的（WebSocket 使用长连接）
        proxy_http_version 1.1;

        # ★ 核心：Upgrade 和 Connection 头
        proxy_set_header Upgrade $http_upgrade;       ← 传递客户端的 Upgrade 请求
        proxy_set_header Connection "upgrade";         ← 固定为 upgrade

        # ★ WebSocket 需要更长的超时时间（默认 60s 可能不够）
        proxy_read_timeout 3600s;                      ← 一小时，避免连接意外断开

        # 其他标准头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 关闭缓冲（WebSocket 是实时全双工通信）
        proxy_buffering off;
    }
}
```

**针对本项目的 WebSocket 端点：**

本项目使用 WebSocket 端点 `/ws/shows/{event_id}/seats` 提供实时座位更新。当用户选择座位时，Nginx 必须正确代理这个连接。

```nginx
# 更精确的 WebSocket location（匹配本项目端点）
location ~ ^/ws/shows/\d+/seats$ {
    proxy_pass http://127.0.0.1:8000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;
    proxy_buffering off;
}
```text

### 5.3 什么是 proxy_http_version 1.1？

```

HTTP/1.0 默认使用短连接，每个请求完成后关闭 TCP 连接。
HTTP/1.1 支持长连接（持久连接），可以在一个 TCP 连接上发送多个请求。

WebSocket 依赖 HTTP Upgrade 机制，
而 HTTP Upgrade 要求使用 HTTP/1.1 或更高版本，
因为 HTTP/1.0 不支持 Upgrade 头。

所以必须有：proxy_http_version 1.1;

```text

---

## 6. SSL/TLS 完整配置

### 6.1 基本 SSL 配置

```nginx
server {
    listen 443 ssl http2;                    ← 官网：在 443 端口启用 SSL 和 HTTP/2
    server_name ticket.example.com;

    # 证书和私钥
    ssl_certificate     /etc/nginx/ssl/ticket.example.com.pem;
    ssl_certificate_key /etc/nginx/ssl/ticket.example.com.key;

    # 协议版本（只允许安全的版本）
    ssl_protocols TLSv1.2 TLSv1.3;           ← 禁止 SSLv3、TLSv1.0、TLSv1.1

    # 加密套件（现代配置，仅允许强加密）
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # 优先使用服务端定义的加密套件顺序
    ssl_prefer_server_ciphers on;
}
```

### 6.2 HTTP 自动跳转 HTTPS

```nginx
server {
    listen 80;
    server_name ticket.example.com;
    # 301 永久重定向到 HTTPS 版本
    return 301 https://$host$request_uri;
}
```text

### 6.3 SSL 会话缓存

SSL/TLS 握手需要 CPU 密集型非对称加密运算。使用会话缓存可以复用之前协商的会话参数，避免重复握手。

```nginx
# 会话缓存：共享内存，大小 10MB，约可存 40000 个会话
ssl_session_cache shared:SSL:10m;

# 会话超时时间（会话在缓存中的有效期）
ssl_session_timeout 10m;
```

**会话复用的效果：**

```text
第一次 HTTPS 请求：
TCP 三次握手（1 RTT） + TLS 完整握手（2 RTT） = 3 RTT 延迟

会话复用（简化的 TLS 握手）：
TCP 三次握手（1 RTT） + TLS 简化握手（1 RTT） = 2 RTT 延迟

TLS 1.3 的 0-RTT（需要额外配置）：
TCP 三次握手（1 RTT） + 第一次请求即可携带数据 = 1 RTT
```

### 6.4 HSTS（HTTP Strict-Transport-Security）

HSTS 告诉浏览器：**以后只能通过 HTTPS 访问这个网站**，杜绝中间人攻击将 HTTPS 降级到 HTTP。

```nginx
# 启用 HSTS，有效期 1 年，包含子域名
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```text

**参数说明：**

```

max-age=31536000     → 浏览器记住这个策略的时间（1 年）
includeSubDomains    → 对所有子域名也生效
preload              → 申请加入浏览器预加载列表（需要手动提交到 hstspreload.org）

```text

### 6.5 OCSP Stapling

OCSP（在线证书状态协议）用于检查证书是否被吊销。传统方式由浏览器直接访问 CA 的 OCSP 服务器，有隐私问题且增加延迟。OCSP Stapling 让 Nginx 代为查询并将结果"订书钉"在 TLS 握手时一起发给客户端。

```nginx
# 启用 OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;

# 配置 CA 证书链（用于验证 OCSP 响应）
ssl_trusted_certificate /etc/nginx/ssl/ca-certificates.crt;

# DNS 解析器（Nginx 需要解析 OCSP 服务器的域名）
resolver 8.8.8.8 1.1.1.1 valid=300s;
resolver_timeout 5s;
```

### 6.6 完整的 SSL 配置模板

```nginx
server {
    listen 443 ssl http2;
    server_name ticket.example.com;

    # ── 证书 ──
    ssl_certificate     /etc/nginx/ssl/ticket.example.com.pem;
    ssl_certificate_key /etc/nginx/ssl/ticket.example.com.key;

    # ── 协议和加密套件 ──
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # ── 会话缓存 ──
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;    ← 禁用 session ticket（某些场景可提高安全性）

    # ── OCSP Stapling ──
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/ssl/ca-certificates.crt;
    resolver 8.8.8.8 1.1.1.1 valid=300s;

    # ── 安全头 ──
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # ── 后端代理 ──
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```text

### 6.7 Let's Encrypt 免费证书获取

```bash
# 1. 安装 certbot
brew install certbot               # macOS
apt install certbot                 # Ubuntu/Debian
yum install certbot                 # CentOS/RHEL

# 2. 获取证书（standalone 模式，需要临时停止 Nginx）
certbot certonly --standalone -d ticket.example.com

# 3. 获取证书（webroot 模式，不需要停止 Nginx）
certbot certonly --webroot -w /var/www/html -d ticket.example.com

# 4. 查看证书
certbot certificates

# 5. 续期（自动）
certbot renew --dry-run

# 6. 证书文件位置
# /etc/letsencrypt/live/ticket.example.com/fullchain.pem
# /etc/letsencrypt/live/ticket.example.com/privkey.pem

# 7. 在 Nginx 中引用
# ssl_certificate     /etc/letsencrypt/live/ticket.example.com/fullchain.pem;
# ssl_certificate_key /etc/letsencrypt/live/ticket.example.com/privkey.pem;
```

---

## 7. 负载均衡 — upstream

### 7.1 定义后端服务器组

```nginx
upstream backend {
    # 定义多个后端服务器
    server 127.0.0.1:8000 weight=3;     ← weight 权重，值越大分配越多
    server 127.0.0.1:8001 weight=1;
    server 127.0.0.1:8002 backup;       ← 备份服务器，只在主服务器全部不可用时启用
}
```text

### 7.2 负载均衡算法

| 算法 | 指令 | 说明 |
| ------ | ------ | ------ |
| 轮询 | （默认） | 依次分配请求，权重越大次数越多 |
| 最少连接 | `least_conn` | 分配给当前连接数最少的服务器 |
| IP 哈希 | `ip_hash` | 同一 IP 始终分配到同一服务器（会话保持） |
| 随机 | `random` | 随机选择，可用于大集群的负载测试 |

```nginx
# 轮询（默认）—— 按顺序轮流分配
upstream backend {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

# 加权轮询
upstream backend {
    server 127.0.0.1:8000 weight=3;    ← 每 5 个请求分配 3 个到 8000
    server 127.0.0.1:8001 weight=1;    ← 每 5 个请求分配 1 个到 8001
    server 127.0.0.1:8002 weight=1;    ← 每 5 个请求分配 1 个到 8002
}

# 最少连接
upstream backend {
    least_conn;
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
}

# IP 哈希（可用于粘性会话——同一用户始终访问同一台服务器）
upstream backend {
    ip_hash;
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
}

# 随机
upstream backend {
    random;
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
}
```

### 7.3 服务器参数详解

```nginx
upstream backend {
    # weight: 权重，默认为 1
    server 127.0.0.1:8000 weight=5;

    # max_fails: 最大失败次数（超时、连接失败等算一次失败）
    # fail_timeout: 在 fail_timeout 时间内达到 max_fails 次失败，标记为不可用
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;

    # backup: 备份服务器
    server 127.0.0.1:8002 backup;

    # down: 手动下线（不参与负载均衡，可用于平滑摘除节点）
    server 127.0.0.1:8003 down;

    # 其他全局参数
    keepalive 32;           ← 保持到后端的空闲长连接数
}
```text

### 7.4 健康检查（开源版 Nginx）

开源版 Nginx 没有主动健康检查功能（需要通过 `nginx-plus` 商业版或第三方模块）。但可以通过 `max_fails` 和 `fail_timeout` 实现**被动健康检查**：

```nginx
upstream backend {
    server 127.0.0.1:8000 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
}
```

行为说明：

```text
1. 每个 30 秒的窗口期内（fail_timeout），如果某台服务器失败 3 次（max_fails）
2. 则这台服务器被标记为"不可用"，持续 30 秒（fail_timeout）
3. 30 秒后恢复尝试，如果仍然失败则再次标记
```

---

## 8. 速率限制 — 漏桶（Leaky Bucket）算法

### 8.1 基本配置

Nginx 使用漏桶算法实现速率限制，通过 `limit_req_zone` 和 `limit_req` 指令配置。

```nginx
# 1. 在 http 块定义限流区域
#    语法：limit_req_zone key zone=名称:内存大小 rate=速率
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    location /api/ {
        # 2. 应用限流规则
        limit_req zone=api_limit;

        # 或带突发和即时处理：
        limit_req zone=api_limit burst=20 nodelay;

        proxy_pass http://backend;
    }
}
```text

**参数说明：**

| 参数 | 说明 |
| ------ | ------ |
| `$binary_remote_addr` | 限流键，使用客户端 IP 的二进制形式（节省内存） |
| `zone=api_limit:10m` | 共享内存区域，名称 api_limit，大小 10MB（约可存 16 万个 IP 状态） |
| `rate=10r/s` | 平均速率，每秒 10 个请求（也可以是 r/m，每分钟多少请求） |
| `burst=20` | 突发容量，允许瞬时超过 rate 的请求数 |
| `nodelay` | 不延迟处理突发请求（默认：延迟处理，请求会排队） |

### 8.2 漏桶算法原理

漏桶算法（Leaky Bucket）的直观比喻：

```

          ▲ 请求到达率（可能不均匀）
          │
    ┌─────┼─────┐
    │  ↙  │  ↘  │  ← 桶（缓存区 burst）
    │   ↙ │ ↘   │
    │  ↙↙↙│↘↘↘  │
    └─────┼─────┘
          │
          ▼ 恒定速率流出 rate=10r/s

```text

**算法行为（burst=20, nodelay）：**

```

时间轴（秒）：
假设在 t=0 时，来了 25 个并发请求

请求 1-10： 立即处理（桶不满，直接通过）
请求 11-20：立即处理（桶被填满，burst=20 全部消耗）
请求 21-25：被拒绝（503 Service Unavailable）

接着在下一秒内：
如果有新请求到达，继续被拒绝，直到漏桶以 10r/s 的速度排空

分析：

- 桶容量（burst）= 20
- 桶流出速度 = 10r/s
- 所以空桶需要 20/10 = 2 秒排空
- 在这 2 秒内到达的新请求全部被拒绝

```text

### 8.3 不同模式的对比

**模式 1：limit_req zone=api_limit（无 burst，默认延迟模式）**

```

rate=10r/s 意味着每 100ms 处理一个请求
如果请求在 t=50ms 到达（上一次处理在 t=0）：
  → 等待到 t=100ms 才处理（延迟 50ms）
如果连续超过速率：
  → 请求被排队延迟处理
  → 队列满了直接返回 503

```text

**模式 2：limit_req zone=api_limit burst=20**

```

rate=10r/s, burst=20
同一秒内来了 25 个请求：
  前 1 个：立即处理（占用当前时隙）
  后 19 个：放入队列等待（burst = 排队 + 当前处理中）
  超出的 5 个：503
  队列以 100ms 一个的速度处理

```text

**模式 3：limit_req zone=api_limit burst=20 nodelay（推荐）**

```

rate=10r/s, burst=20, nodelay
同一秒内来了 25 个请求：
  前 20 个：立即处理（<= burst，不等待）
  后 5 个：503
  下一秒内：最多处理 10 个（但桶已空，需要等排空）

```text

### 8.4 数学证明：漏桶限流保证速率不超过设定值

**正式定义：**

```

rate = μ (处理速率，即桶的流出速率)
burst = B (桶容量)
nodelay：到达请求立即处理（直到桶满）

```text

**证明目标：在任意长时间窗口 T 内，处理的请求数 ≤ μT + B**

```

证明：

设时间从 t=0 开始，桶初始为空。
设函数 f(t) 为 [0, t] 内到达的请求数。
设函数 g(t) 为 [0, t] 内被处理的请求数。

漏桶算法的约束条件：

1. 桶中积压的请求数（队列长度）Q(t) = f(t) - g(t)
2. Q(t) ≤ B（桶满则丢弃）
3. 当 Q(t) > 0 时，处理速率 = μ（有积压时全速处理）
4. 当 Q(t) = 0 时，处理速率 ≤ μ（无积压时以到达速率处理）

推论：
在任何时刻，g(t) ≥ f(t) - B（因为被丢弃的请求不计入 f 也不计入 g）
（f - g 是已到达但未处理的，它不能超过 B）

所以：g(t) ≥ f(t) - B

对于任意时间窗口 [t₁, t₂]：
g(t₂) - g(t₁) ≤ μ(t₂ - t₁) + B

证明：
g(t₂) - g(t₁) = [g(t₂) - g(t₁)]

从 t₁ 到 t₂ 期间：

- 如果桶一直非空：处理了 μ(t₂ - t₁) 个请求
- 如果桶有变空：最多额外处理 B 个积压请求

因此：g(t₂) - g(t₁) ≤ μ(t₂ - t₁) + B

Q.E.D.
结论：任意窗口的处理速率被严格限制在 μ + B/T，当 T→∞ 时趋近于 μ。

```text

### 8.5 限流和并发限制的区别

```nginx
# 速率限制：限制每秒/每分钟的请求数
limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;

# 并发连接数限制：限制同时处理的连接数
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

server {
    location /api/ {
        limit_req zone=req_limit burst=20 nodelay;
        limit_conn conn_limit 10;    ← 每个 IP 最多 10 个并发连接
        proxy_pass http://backend;
    }
}
```

---

## 9. 安全响应头

### 9.1 安全头配置

```nginx
server {
    listen 443 ssl http2;
    server_name ticket.example.com;

    # ── 安全响应头 ──

    # 内容安全策略：限制资源加载来源
    add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' 'unsafe-inline';
        style-src 'self' 'unsafe-inline';
        img-src 'self' data:;
        font-src 'self';
        connect-src 'self' https://api.example.com;
        frame-ancestors 'none';
        form-action 'self';
    " always;

    # 禁止被嵌入 iframe（防止点击劫持）
    add_header X-Frame-Options DENY always;

    # 启用 XSS 过滤器（旧浏览器）
    add_header X-XSS-Protection "1; mode=block" always;

    # 禁止 MIME 类型嗅探（防止脚本类型混淆攻击）
    add_header X-Content-Type-Options nosniff always;

    # 控制 Referer 信息（隐私保护）
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # 权限控制（限制浏览器 API 访问）
    add_header Permissions-Policy "geolocation=(), camera=(), microphone=()" always;
}
```text

### 9.2 各安全头详解

**Content-Security-Policy（CSP）：**

最重要也是最复杂的安全头。它告诉浏览器哪些来源是可信的，可以有效防御 XSS 和数据注入攻击。

```nginx
# 严格模式（推荐生产环境使用）
add_header Content-Security-Policy "
    default-src 'self';
    script-src 'self';
    style-src 'self';
    img-src 'self';
    font-src 'self';
    connect-src 'self';
    frame-ancestors 'none';
    form-action 'self';
    base-uri 'self';
" always;
```

| 指令 | 作用 |
| ------ | ------ |
| `default-src` | 所有资源类型的默认策略 |
| `script-src` | 允许加载 JavaScript 的来源 |
| `style-src` | 允许加载 CSS 的来源 |
| `img-src` | 允许加载图片的来源 |
| `connect-src` | 允许 AJAX/Fetch/WebSocket 连接的目标 |
| `frame-ancestors` | 允许哪些来源通过 iframe 嵌入本页面 |
| `form-action` | 允许表单提交的目标 URL |
| `base-uri` | 允许在 `<base>` 标签中使用的 URL |

**X-Frame-Options DENY：**

告诉浏览器**绝对不允许**将本页面嵌入到 iframe 中，这是防御**点击劫持（Clickjacking）**的最简单手段。

```text
DENY        → 任何情况都不允许嵌入
SAMEORIGIN  → 同源页面可以嵌入
ALLOW-FROM uri → 指定来源可嵌入（已废弃，用 CSP frame-ancestors 替代）
```

**X-Content-Type-Options nosniff：**

禁止浏览器的 MIME 类型嗅探行为。如果没有这个头，浏览器可能会将 `text/plain` 的内容当作 `text/html` 渲染，导致 XSS 攻击。

```text
例如：攻击者上传一个看似 txt 文件，内容包含 JavaScript
没有 nosniff：浏览器检测内容后当作脚本执行
有 nosniff：浏览器严格遵守 Content-Type，不会执行
```

### 9.3 在 Nginx 中统一添加安全头

即使后端应用没有设置这些头，Nginx 也可以在代理响应中添加：

```nginx
location / {
    proxy_pass http://backend;

    # 覆盖或添加安全头
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;

    # 注意：add_header 是 Nginx 添加响应头
    # 如果后端本身已经设置了这些头，add_header 不会覆盖（需要用 proxy_hide_header 先删除）
}
```text

**关于 add_header 的生效范围：** `add_header` 在 location 中配置时，**只有当前 location 处理的请求才会收到**。如果多个 location 都需要，要分别在每个 location 中添加。从 Nginx 1.21.0 开始，`http` 和 `server` 上下文的 `add_header` 会向下继承。

---

## 10. 项目完整配置示例

### 10.1 nginx/nginx.conf — 主配置文件

本项目的主配置文件，定义了全局参数和 http 块：

```nginx
# /etc/nginx/nginx.conf
# Nginx 主配置文件

# ── 用户 ──
user nginx;

# ── 进程 ──
worker_processes auto;          ← 自动检测 CPU 核心数
worker_rlimit_nofile 65535;     ← 每个 worker 进程的最大文件描述符数

# ── 错误日志 ──
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

# ── 事件模块 ──
events {
    worker_connections 10240;    ← 每个 worker 最大连接数
    multi_accept on;             ← 一次 accept 多个新连接
    use epoll;                   ← 使用 epoll（Linux 上）
}

# ── HTTP 模块 ──
http {
    # 包含 MIME 类型映射
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # ── 日志格式 ──
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time ut=$upstream_response_time '
                    'cache=$upstream_cache_status';

    access_log /var/log/nginx/access.log main buffer=32k;

    # ── 性能优化 ──
    sendfile on;                 ← 使用 sendfile 系统调用（零拷贝）
    tcp_nopush on;               ← 优化 TCP 数据包发送
    tcp_nodelay on;              ← 禁用 Nagle 算法
    keepalive_timeout 65;
    keepalive_requests 1000;     ← 一个 keepalive 连接最多处理 1000 个请求
    client_max_body_size 10m;    ← 客户端请求体最大大小

    # ── Gzip 压缩 ──
    gzip on;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml text/javascript image/svg+xml;
    gzip_vary on;
    gzip_proxied any;

    # ── 限流定义 ──
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=static_limit:10m rate=30r/s;

    # ── 并发限制定义 ──
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # ── 上游服务器组 ──
    upstream ticket_backend {
        least_conn;
        server 127.0.0.1:8000 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;

        # keepalive 连接
        keepalive 32;
    }

    # ── HTTP 服务器（重定向到 HTTPS） ──
    server {
        listen 80;
        server_name ticket.example.com;
        return 301 https://$host$request_uri;
    }

    # ── HTTPS 服务器 ──
    server {
        listen 443 ssl http2;
        server_name ticket.example.com;

        # SSL 配置
        ssl_certificate /etc/nginx/ssl/ticket.example.com.pem;
        ssl_certificate_key /etc/nginx/ssl/ticket.example.com.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_session_tickets off;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /etc/nginx/ssl/ca-certificates.crt;
        resolver 8.8.8.8 1.1.1.1 valid=300s;

        # HSTS
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # ── favicon ──
        location = /favicon.ico {
            log_not_found off;
            access_log off;
        }

        # ── 静态文件 ──
        location /static/ {
            root /var/www/ticket;
            expires 7d;
            add_header Cache-Control "public, immutable";
            access_log off;
        }

        # ── API 反向代理 ──
        location /api/ {
            # 限流
            limit_req zone=api_limit burst=20 nodelay;
            limit_conn conn_limit 10;

            # 代理到后端
            proxy_pass http://ticket_backend;
            proxy_http_version 1.1;

            # 头部传递
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 超时
            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
            proxy_send_timeout 30s;

            # 缓冲
            proxy_buffering on;
            proxy_buffers 8 8k;
            proxy_buffer_size 4k;
            proxy_busy_buffers_size 16k;

            # 安全头
            add_header X-Frame-Options DENY always;
            add_header X-Content-Type-Options nosniff always;
            add_header X-XSS-Protection "1; mode=block" always;
        }

        # ── WebSocket 代理（实时座位更新） ──
        location ~ ^/ws/shows/\d+/seats$ {
            proxy_pass http://ticket_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_read_timeout 3600s;
            proxy_buffering off;
        }

        # ── Docs 反向代理 ──
        location /docs {
            proxy_pass http://ticket_backend;
            proxy_set_header Host $host;
        }

        location /openapi.json {
            proxy_pass http://ticket_backend;
            proxy_set_header Host $host;
        }

        # ── 错误页面 ──
        error_page 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

### 10.2 nginx/conf.d/default.conf — 分块配置文件

为了让配置更清晰，可以将不同功能拆分到 `conf.d/` 目录中：

```nginx
# /etc/nginx/conf.d/upstream.conf — 上游服务器定义
upstream ticket_backend {
    least_conn;
    server 127.0.0.1:8000 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
    keepalive 32;
}
```text

```nginx
# /etc/nginx/conf.d/rate-limit.conf — 限流定义
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
```

```nginx
# /etc/nginx/conf.d/ticket-ssl.conf — SSL 虚拟主机
server {
    listen 443 ssl http2;
    server_name ticket.example.com;

    ssl_certificate /etc/nginx/ssl/ticket.example.com.pem;
    ssl_certificate_key /etc/nginx/ssl/ticket.example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/ssl/ca-certificates.crt;
    resolver 8.8.8.8 1.1.1.1 valid=300s;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://ticket_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```text

然后在主配置文件中引入：

```nginx
# /etc/nginx/nginx.conf
http {
    include /etc/nginx/conf.d/*.conf;    ← 引入所有分块配置
}
```

### 10.3 nginx/ssl/README.md — SSL 证书说明

本项目 nginx/ssl/ 目录用于存放 SSL 证书文件：

```markdown
# SSL 证书目录

## 文件说明

| 文件 | 用途 | 来源 |
| ------ | ------ | ------ |
| ticket.example.com.pem | 服务器证书（含完整链） | Let's Encrypt / 商业 CA |
| ticket.example.com.key | 服务器私钥 | 生成证书时创建 |
| ca-certificates.crt | CA 证书链（用于 OCSP Stapling） | CA 提供 |

## 获取证书

### 方式一：Let's Encrypt（免费，推荐）

```bash
apt install certbot
certbot certonly --webroot -w /var/www/html -d ticket.example.com

# 证书位于：
# /etc/letsencrypt/live/ticket.example.com/fullchain.pem
# /etc/letsencrypt/live/ticket.example.com/privkey.pem

# 复制到 Nginx：
cp /etc/letsencrypt/live/ticket.example.com/fullchain.pem /etc/nginx/ssl/ticket.example.com.pem
cp /etc/letsencrypt/live/ticket.example.com/privkey.key /etc/nginx/ssl/ticket.example.com.key

# 自动续期（crontab）：
# 0 3 * * * certbot renew --quiet && systemctl reload nginx
```text

### 方式二：自签名证书（开发环境）

```bash
# 生成自签名证书（仅用于开发测试，浏览器会有安全警告）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/ticket.example.com.key \
    -out /etc/nginx/ssl/ticket.example.com.pem \
    -subj "/CN=ticket.example.com"
```

## 权限要求

私钥文件必须严格限制权限：

```bash
chmod 600 /etc/nginx/ssl/*.key
chmod 644 /etc/nginx/ssl/*.pem
chown -R root:root /etc/nginx/ssl/
```text

## 安全注意事项

1. 私钥文件绝对不能提交到 Git 仓库
2. 建议使用 acme.sh 或 certbot 自动续期
3. 证书过期前 30 天应发送告警
4. 使用 online 工具（如 SSL Labs）定期检查评分

```

### 10.4 Docker Compose 中的 Nginx

如果在 Docker 环境中运行本项目，可以使用 Docker Compose 配置 Nginx：

```yaml
version: "3.8"

services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./static:/var/www/ticket/static:ro
    depends_on:
      - api
    networks:
      - ticket_net
    restart: unless-stopped

  api:
    build: .
    expose:
      - "8000"
    environment:
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - ticket_net
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: ticket_dev
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - ticket_net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - ticket_net

networks:
  ticket_net:
    driver: bridge

volumes:
  mysql_data:
  redis_data:
```text

### 10.5 常用运维命令

```bash
# 测试配置是否正确
nginx -t

# 重新加载配置（不中断服务）
nginx -s reload

# 优雅停止
nginx -s quit

# 立即停止
nginx -s stop

# 重新打开日志文件（日志轮转时使用）
nginx -s reopen

# 查看编译参数
nginx -V

# 查看版本
nginx -v

# 测试配置时使用自定义路径
nginx -t -c /etc/nginx/nginx.conf

# 访问日志实时查看
tail -f /var/log/nginx/access.log | lnav

# 错误日志查看
tail -f /var/log/nginx/error.log

# 检查 SSL 证书到期时间
echo | openssl s_client -connect localhost:443 -servername ticket.example.com 2>/dev/null | openssl x509 -noout -dates
```

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `nginx/nginx.conf` | Nginx 主配置：反向代理、静态资源、WebSocket 代理、SSL |
| `nginx/ssl/` | SSL 证书目录 |
| `docker-compose.prod.yml` | 生产环境含 Nginx 容器编排 |

**WebSocket 代理配置**（关键部分）：

```nginx
location /ws/ {
    proxy_pass http://fastapi;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;  # WebSocket 长连接超时
}
```text

## 本章总结

**核心知识点回顾：**

1. **master-worker 架构**：master 管理配置和 worker 生命周期，worker 使用 epoll 事件驱动处理请求，不阻塞在任何操作上

2. **配置层次**：`main → http → server → location → if`，内层可以重写外层指令

3. **location 匹配顺序**：
   - `=` 精确匹配 → 立即返回
   - `^~` 前缀匹配 → 匹配后跳过正则
   - `~*` / `~` 正则匹配 → 按顺序第一个命中
   - 无修饰符前缀匹配 → 最长者胜出
   - 无一匹配 → 404

4. **proxy_pass 带 URI  vs 不带 URI**：带 URI 会将 location 匹配部分替换为 proxy_pass 中的 URI，是配置反向代理最常见的坑

5. **WebSocket 代理三要素**：`proxy_http_version 1.1`、`Upgrade` 头、`Connection: upgrade` 头

6. **SSL 关键配置**：`ssl_protocols TLSv1.2 TLSv1.3`、`ssl_ciphers` 强加密套件、HSTS、OCSP Stapling

7. **漏桶限流**：`rate` 控制平均速率，`burst` 控制突发容量，`nodelay` 控制是否延迟处理，数学证明保证任意时间窗口内速率不超过 `μ + B/T`

8. **upstream 负载均衡**：`round-robin`、`least_conn`、`ip_hash`、`random` 四种算法，`weight`、`backup`、`max_fails` 等参数

---

**项目检查清单：**

- [ ] Nginx 主配置 nginx.conf 已编写并测试
- [ ] 反向代理代理至 FastAPI 应用（`/api/v1/` 路由）
- [ ] SSL 证书配置完毕（Let's Encrypt 或自签名）
- [ ] HTTP 自动跳转 HTTPS（301 重定向）
- [ ] HSTS、安全响应头已配置
- [ ] WS WebSocket 代理已配置（座位实时更新）
- [ ] 漏桶速率限制已生效（`api_limit zone`）
- [ ] 负载均衡已配置（多副本部署时使用）
- [ ] `nginx -t` 测试通过

---

## Nginx 安全配置详解

### 速率限制（Rate Limiting）

Nginx 的 `limit_req_zone` 指令基于漏桶算法（Leaky Bucket）：

```

limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

```text

- `$binary_remote_addr`：以客户端 IP 为 key（相比 $remote_addr 节省内存）
- `zone=login:10m`：共享内存区域名称和大小（10MB 可存储约 16 万个 IP）
- `rate=5r/m`：平均速率限制为每分钟 5 个请求

应用时配合 `burst` 和 `nodelay`：

```

location /api/v1/auth/ {
    limit_req zone=login burst=10 nodelay;
    proxy_pass <http://fastapi>;
}

```text

- `burst=10`：允许突发最多额外 10 个请求排队
- `nodelay`：排队中的请求不延迟处理（但超出 burst 的请求立即返回 503）

### 安全标头配置

```

add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; frame-src 'none';" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

```text

### 常见安全误区

1. **不要暴露版本号**：`server_tokens off;` 隐藏 Nginx 版本
2. **限制请求方法**：仅允许 GET/POST/PUT/DELETE 等必要方法
3. **限制 body 大小**：`client_max_body_size 1m;` 防止大文件上传 DoS
4. **禁用不安全的 TLS 版本**：`ssl_protocols TLSv1.2 TLSv1.3;`
5. **启用 OCSP Stapling**：提高 TLS 握手性能和安全

- [ ] 错误日志无异常
