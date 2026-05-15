# 第8章：WebSocket + SSE + ARQ — 实时通信与后台任务完全教程

## 本章学习目标

本章将深入讲解三种实时通信技术：WebSocket 协议（RFC 6455）的二进制细节与 FastAPI 实现、SSE 服务器推送事件、ARQ 异步任务队列。最终整合实现票务系统的实时座位更新、通知推送和后台任务。

**学习路径：**

1. WebSocket 协议 RFC 6455 逐字节解析
2. FastAPI WebSocket API 详解
3. ConnectionManager 房间级广播源码逐行分析
4. WebSocket 路由消息协议设计
5. SSE 服务器推送事件与 EventSource
6. ARQ 异步任务队列与 Worker 配置
7. Redis PubSub 跨实例广播
8. 算法复杂度分析与正确性证明

---

## 1. WebSocket 协议 (RFC 6455) 完全教程

### 1.1 协议概述

WebSocket 是 HTML5 规范的一部分，在单个 TCP 连接上提供**全双工**通信通道。与 HTTP 的请求-响应模式不同，WebSocket 允许服务器主动向客户端推送数据，是实时应用的基石。

**协议栈对比：**

```text
传统 HTTP 轮询：
  Client  ----HTTP Request---->  Server
  Client  <---HTTP Response---  Server
  Client  ----HTTP Request---->  Server  (间隔几秒再问一次)
  Client  <---HTTP Response---  Server
  问题：大量重复的 HTTP 头部开销，延迟高

WebSocket 长连接：
  Client  ----HTTP Upgrade---->  Server  (仅一次握手)
  Client  <---101 Switching---  Server
  Client  <---WebSocket Frame--  Server  (随时推送)
  Client  ---WebSocket Frame-->  Server  (全双工)
  优势：无头部开销，低延迟，真正的双向通信
```

### 1.2 HTTP Upgrade 握手（逐行详解）

WebSocket 连接始于一个 HTTP Upgrade 请求，让服务器将协议从 HTTP 切换到 WebSocket。

**客户端握手请求：**

```text
GET /ws/chat HTTP/1.1
Host: server.example.com
Upgrade: websocket              ← 告诉服务器要升级协议到 WebSocket
Connection: Upgrade              ← HTTP 标准：Upgrade 头需要 Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==  ← 16 字节随机数 Base64 编码
Sec-WebSocket-Version: 13       ← 协议版本，RFC 6455 要求必须为 13
Sec-WebSocket-Protocol: chat    ← 可选：应用层子协议
Origin: http://example.com       ← 可选：用于鉴权
```

**关键字段详解：**

- **Sec-WebSocket-Key**: 客户端生成的 16 字节随机值，Base64 编码后发送。目的是让服务器证明它收到了请求——服务器需要从这个值计算出 `Sec-WebSocket-Accept` 返回给客户端。**这不是认证机制**，而是为了防止缓存代理重新发送旧的 WebSocket 握手。
- **Sec-WebSocket-Version**: 必须为 13。RFC 6455 将版本号定为 13（之前的草案版本从 0 到 12 都有，互不兼容）。

**服务端响应：**

```text
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Sec-WebSocket-Accept 计算过程：**

```python
import hashlib
import base64

# 客户端发来的 Sec-WebSocket-Key
key = "dGhlIHNhbXBsZSBub25jZQ=="

# RFC 6455 定义的固定 GUID
MAGIC_GUID = "258EAFA5-E914-47DA-95CA-5AB5DC11B735"

# 1. 拼接 key + GUID
concat = key + MAGIC_GUID
# 结果: "dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-5AB5DC11B735"

# 2. SHA-1 哈希
sha1_hash = hashlib.sha1(concat.encode()).digest()
# 结果: 20 字节的二进制哈希值

# 3. Base64 编码
accept = base64.b64encode(sha1_hash).decode()
# 结果: "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="

# 验证
assert accept == "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="  # 正确！
```text

**为什么需要这个计算？**

这个机制是为了防止无意的 WebSocket 连接。如果不做这个验证：

1. **缓存代理问题**：一个 HTTP 缓存代理可能看到 `Upgrade` 请求后直接返回旧的 HTTP 响应，而不是转发给 WebSocket 服务器。服务器通过 `Key/Accept` 握手确保双方都理解 WebSocket 协议。
2. **服务器验证**：客户端可以确认连接的另一端确实是理解 WebSocket 协议的服务器，而不是某个中间件返回的陈旧响应。

### 1.3 WebSocket 帧格式（逐字节详解）

WebSocket 通信中的数据以**帧 (Frame)** 为单位传输。帧格式是二进制的，结构如下：

```

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     |      Masking-key (continued)   |          Payload Data        |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data (continued)                  |
     +---------------------------------------------------------------+

```text

**逐字段详解：**

| 字段 | 长度 | 说明 |
| ------ | ------ | ------ |
| **FIN** | 1 bit | 标记是否为消息的最后一帧。1=结束帧，0=还有后续帧（分片） |
| **RSV1-3** | 3 bits | 保留位，通常为 0。如果扩展协商使用则可为非零值 |
| **opcode** | 4 bits | 帧类型（见下文） |
| **MASK** | 1 bit | 是否使用掩码。客户端→服务器必须为 1，服务器→客户端必须为 0 |
| **Payload Length** | 7 bits | 载荷长度。有三种编码方式（见下文） |
| **Extended Length** | 0/16/64 bits | 当 Payload Length=126 时为 16 位，=127 时为 64 位 |
| **Masking-Key** | 0/32 bits | 当 MASK=1 时为 4 字节的掩码密钥 |
| **Payload Data** | 变长 | 实际数据 |

**载荷长度编码规则：**

```

0-125:    7 位直接表示载荷长度
126:      后面跟 2 字节（16 位无符号整数）表示长度（最大 65535）
127:      后面跟 8 字节（64 位无符号整数）表示长度（最大 2^63-1）

```text

**Python 解析示例：**

```python
import struct

def parse_websocket_frame(data: bytes) -> dict:
    """解析 WebSocket 帧（教学用，非生产代码）"""
    first_byte = data[0]
    fin = (first_byte >> 7) & 0x01
    rsv = (first_byte >> 4) & 0x07
    opcode = first_byte & 0x0F

    second_byte = data[1]
    mask = (second_byte >> 7) & 0x01
    payload_len = second_byte & 0x7F

    offset = 2

    if payload_len == 126:
        # 2 字节扩展长度
        payload_len = struct.unpack(">H", data[2:4])[0]
        offset = 4
    elif payload_len == 127:
        # 8 字节扩展长度（取低 63 位）
        payload_len = struct.unpack(">Q", data[2:10])[0]
        offset = 10

    masking_key = None
    if mask:
        masking_key = data[offset:offset + 4]
        offset += 4

    payload = data[offset:offset + payload_len]

    # 如有掩码，解码
    if mask and masking_key:
        payload = bytes(
            b ^ masking_key[i % 4] for i, b in enumerate(payload)
        )

    return {
        "fin": fin,
        "rsv": rsv,
        "opcode": opcode,
        "mask": mask,
        "payload_length": payload_len,
        "payload": payload,
    }

# 测试：客户端发送的文本帧（掩码）
frame = bytes([
    0x81,  # FIN=1, opcode=1(text)
    0x85,  # MASK=1, payload_len=5
    0x01, 0x02, 0x03, 0x04,  # masking-key
    0x65, 0x6B, 0x69, 0x6F, 0x73,  # masked payload
])
result = parse_websocket_frame(frame)
print(result["payload"].decode())  # "hello"
```

### 1.4 所有 Opcode 详解

| Opcode | 值 | 含义 | 说明 |
| -------- | ----- | ------ | ------ |
| **Continuation** | 0x0 | continuation 帧 | 表示这是分片消息的后续帧 |
| **Text** | 0x1 | 文本帧 | 载荷为 UTF-8 文本 |
| **Binary** | 0x2 | 二进制帧 | 载荷为任意二进制数据 |
| **Close** | 0x8 | 关闭帧 | 发起关闭握手，可选带状态码 |
| **Ping** | 0x9 | Ping 帧 | 心跳探测 |
| **Pong** | 0xA | Pong 帧 | 对 Ping 的响应 |

**Opcode 的 FIN 位关系：**

- **非控制帧**（Text/Binary/Continuation，opcode 0x0-0x7）：可以分片发送。第一个帧的 opcode 为 Text 或 Binary，后续 Continuation 帧的 opcode 为 0x0，最后一个帧的 FIN=1。
- **控制帧**（Close/Ping/Pong，opcode 0x8-0xF）：**不得分片**，FIN 必须为 1。帧长不得超过 125 字节。控制帧可以在消息分片的间隙插入。

### 1.5 掩码算法（Masking）详解

**为什么客户端到服务器的帧必须掩码？**

RFC 6455 要求客户端发送的所有帧必须设置 MASK=1，并携带 4 字节的 Masking-Key。这是因为一个著名的安全攻击：

**缓存投毒攻击 (Cache Poisoning Attack) 示例：**

假设攻击者控制了一个恶意 WebSocket 客户端，服务器是一个 HTTP 代理缓存：

```text
攻击场景：
1. 攻击者的恶意页面在浏览器中打开一个到 example.com 的 WebSocket 连接
2. 同时，该页面通过 XMLHttpRequest 触发一个 HTTP 请求到 example.com
3. 浏览器中的 TCP 连接复用可能导致 HTTP 请求体与 WebSocket 帧混合
4. 攻击者可以构造 WebSocket 数据，使其看起来像一个 HTTP 响应
5. 缓存代理可能将这段数据缓存为对后续 HTTP 请求的响应

掩码如何防御：
由于 WebSocket 数据经过掩码处理，攻击者无法预测掩码后的字节序列。
恶意 JavaScript 无法构造出合法的 HTTP 响应头，因为掩码是不可预测的。
```

**掩码算法（RFC 6455 第 5.3 节）：**

```python
def apply_mask(payload: bytes, masking_key: bytes) -> bytes:
    """
    对载荷应用掩码（编码/解码使用相同操作，因为 XOR 是对称的）

    算法：
    j = i MOD 4  （i 是字节索引，j 是 masking_key 索引）
    transformed_octet_i = original_octet_i XOR masking_key[j]
    """
    return bytes(
        payload[i] ^ masking_key[i % 4]
        for i in range(len(payload))
    )

# 示例
masking_key = b"\x37\xfa\x21\x3d"
payload = b"Hello, WebSocket!"

masked = apply_mask(payload, masking_key)
print(f"掩码后: {masked.hex()}")

# 再次应用掩码可还原
unmasked = apply_mask(masked, masking_key)
print(f"还原后: {unmasked}")  # b"Hello, WebSocket!"
```text

### 1.6 分片 (Fragmentation)

分片机制允许将一个大消息拆分成多个帧发送。分片对应用层是透明的。

**分片规则：**

```

一个没有分片的消息：
  [FIN=1, opcode=Text]  ← 唯一一帧，FIN=1 表示结束

一个分片为 3 帧的消息：
  [FIN=0, opcode=Text]        ← 第一帧：告知消息类型
  [FIN=0, opcode=Continuation]  ← 中间帧
  [FIN=1, opcode=Continuation]  ← 最后帧：FIN=1 表示消息完整

```text

**控制帧可以在分片间隙插入：**

```

  [FIN=0, opcode=Text]
  [FIN=1, opcode=Ping]   ← 控制帧在分片间隙发送
  [FIN=1, opcode=Pong]   ← 响应
  [FIN=1, opcode=Continuation]

```text

**为什么需要分片？**

1. **大消息传输**：当消息大小超过一个缓冲区大小时，可以边生成边发送。
2. **流式处理**：服务器收到第一帧就知道消息类型，可以开始处理，无需等待整个消息完成。
3. **多路复用**（扩展）：未来的扩展可以实现一个连接上的消息交织。

### 1.7 Close 握手与状态码

WebSocket 关闭握手是对称的：任何一方都可以发起，另一方必须响应。

**Close 帧格式：**

```

关闭帧的载荷（可选，最多 125 字节）：
  2 字节：状态码（无符号 16 位整数）
  剩余字节：UTF-8 编码的原因描述

示例：
  0x88 0x04 0x03 0xE8 0x4E 0x6F

- FIN=1, opcode=0x8 (Close)
- 载荷长度 4
- 0x03E8 = 1000 (Normal Closure)
- 0x4E 0x6F = "No" (原因描述)

```text

**标准状态码：**

| 状态码 | 值 | 含义 |
| -------- | ----- | ------ |
| **Normal Closure** | 1000 | 正常关闭 |
| **Going Away** | 1001 | 端点关闭（如服务器关闭） |
| **Protocol Error** | 1002 | 协议错误 |
| **Unsupported Data** | 1003 | 收到不支持的数据类型 |
| **No Status Received** | 1005 | 保留值，表示未收到状态码 |
| **Abnormal Closure** | 1006 | 保留值，连接异常断开 |
| **Invalid Payload** | 1007 | 载荷数据不一致（如非 UTF-8） |
| **Policy Violation** | 1008 | 违反策略 |
| **Message Too Big** | 1009 | 消息太大 |
| **Mandatory Extension** | 1010 | 缺少服务器要求的扩展 |
| **Internal Error** | 1011 | 服务器内部错误 |
| **TLS Handshake** | 1015 | 保留值，TLS 握手失败 |

**关闭流程：**

```

双向关闭示例：

客户端 → 服务器：Close(1000, "Goodbye")
  服务器收到后关闭 TCP 连接（或先发自己的 Close）
服务器 → 客户端：Close(1000, "Bye!")
  客户端收到后关闭 TCP 连接

```text

**FastAPI 中的关闭：**

```python
from fastapi import WebSocket

# WebSocket 断开时会抛出 WebSocketDisconnect
# 可以通过 close() 方法主动关闭

@router.websocket("/ws")
async def ws_endpoint(websocket: WebSocket):
    await websocket.accept()
    await websocket.close(code=1000, reason="Server shutting down")
```

---

## 2. FastAPI WebSocket API 完全详解

### 2.1 基本 API 列表

FastAPI 的 WebSocket 基于 Starlette 的 WebSocket 实现，提供以下核心 API：

```python
from fastapi import WebSocket, WebSocketDisconnect

websocket: WebSocket

# ---- 生命周期 ----
await websocket.accept()                    # 接受连接（发送 101）
await websocket.close(code=1000)            # 主动关闭连接

# ---- 接收 ----
data = await websocket.receive_text()       # 接收文本（str）
data = await websocket.receive_bytes()      # 接收二进制（bytes）
data = await websocket.receive_json()       # 接收 JSON（自动解析）
raw = await websocket.receive()             # 接收原始帧（dict）

# ---- 发送 ----
await websocket.send_text("hello")          # 发送文本
await websocket.send_bytes(b"hello")        # 发送二进制
await websocket.send_json({"msg": "hello"}) # 发送 JSON（自动序列化）
await websocket.send({"type": "websocket.send", "text": "hello"})  # 发送原始帧

# ---- 迭代器模式 ----
async for data in websocket.iter_text():
    print(data)
async for data in websocket.iter_bytes():
    print(data)
async for data in websocket.iter_json():
    print(data)
```text

### 2.2 WebSocketDisconnect 异常处理

当客户端断开连接或者连接异常时，`receive_*` 方法会抛出 `WebSocketDisconnect`：

```python
from fastapi import WebSocket, WebSocketDisconnect

@router.websocket("/ws")
async def ws_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect as e:
        # e.code：关闭状态码（如 1000, 1001, 1006）
        # e.reason：关闭原因
        print(f"客户端断开: code={e.code}, reason={e.reason}")
    except Exception as e:
        # 其他异常（如序列化错误）
        print(f"未知错误: {e}")
    finally:
        # 清理资源
        print("清理连接资源")
```

**WebSocketDisconnect 属性：**

```python
class WebSocketDisconnect(Exception):
    def __init__(self, code: int = 1000, reason: str = ""):
        self.code = code      # 状态码
        self.reason = reason  # 原因
```text

### 2.3 WebSocket 路由（路径参数）

FastAPI 的 WebSocket 路由支持路径参数：

```python
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

router = APIRouter()

@router.websocket("/ws/shows/{show_id}/seats/{seat_id}")
async def seat_websocket(websocket: WebSocket, show_id: int, seat_id: str):
    """支持路径参数的 WebSocket 端点"""
    await websocket.accept()
    try:
        await websocket.send_json({
            "type": "connected",
            "show_id": show_id,
            "seat_id": seat_id,
            "message": f"已连接到场次 {show_id} 的座位 {seat_id}",
        })
        while True:
            data = await websocket.receive_json()
            # 处理消息...
    except WebSocketDisconnect:
        ...
```

**注意事项：**

1. WebSocket 路由**不支持** OpenAPI 文档（因为 WebSocket 不在 OpenAPI 规范中），但 FastAPI 会将其显示在自动生成的文档中。
2. WebSocket 端点默认没有 `Depends` 注入，但 FastAPI 支持 `Depends` 用于 WebSocket 路由（需要手动解析 token）。
3. 路径参数的类型提示会触发 FastAPI 的自动校验（如 `int` 会自动转换）。

---

## 3. ConnectionManager 源码逐行解析

我们项目中的 `app/services/connection_manager.py` 是一个**房间级广播管理器**，用于管理多个 WebSocket 连接的广播。

### 3.1 完整源码分析

```python
from typing import Set, Dict
from fastapi import WebSocket

class ConnectionManager:
    """WebSocket 连接管理器：支持房间级广播"""

    def __init__(self):
        # room_id -> set of WebSocket connections
        self._rooms: Dict[str, Set[WebSocket]] = {}
```text

**第 1-2 行**：从标准库和 FastAPI 导入必要类型。`Set` 用于存储 WebSocket 连接的无序集合，`Dict` 映射房间 ID 到连接集合。

**第 5 行**：定义 `ConnectionManager` 类。

**第 6 行**：文档字符串，说明这是一个支持房间级广播的 WebSocket 管理器。

**第 8-9 行**：构造函数初始化 `_rooms` 字典。键是房间 ID 字符串（如 `"seats:1"`），值是 `Set[WebSocket]`——同一房间内的所有 WebSocket 连接。

**为什么用 `Set` 而不是 `List`？** `Set` 保证同一个 WebSocket 连接不会重复添加（因为 `add` 去重，`discard` 删除不存在的不报错）。如果是 `List`，`remove` 会引发 `ValueError`。

```python
    async def connect(self, websocket: WebSocket, room: str):
        """将 WebSocket 加入指定房间"""
        await websocket.accept()
        if room not in self._rooms:
            self._rooms[room] = set()
        self._rooms[room].add(websocket)
```

**第 12 行**：`connect` 方法是异步的，接收一个 WebSocket 实例和房间名。

**第 13 行**：文档字符串。

**第 14 行**：这里调用 `websocket.accept()` 接受连接。注意：这与 FastAPI 的 `@app.websocket` 路由不同——在路由器中我们需要手动调用 `accept()`。

**第 15-16 行**：如果房间还不存在，创建一个新的空 `set()`。

**第 17 行**：将 WebSocket 添加到房间集合中。

```python
    def disconnect(self, websocket: WebSocket, room: str):
        """将 WebSocket 从房间移除"""
        if room in self._rooms:
            self._rooms[room].discard(websocket)
            if not self._rooms[room]:
                del self._rooms[room]
```text

**第 19 行**：`disconnect` 是同步方法（不需要网络 I/O）。

**第 20 行**：文档字符串。

**第 21 行**：检查房间是否存在（防止 `KeyError`）。

**第 22 行**：`discard()` 是 Set 的安全删除方法——如果元素不存在也不会报错。

**第 23-24 行**：如果房间空了，删除这个房间的键，防止内存泄漏。

```python
    async def broadcast(self, room: str, message: dict):
        """向房间内所有客户端广播消息"""
        if room not in self._rooms:
            return
        stale = set()
        for ws in self._rooms[room]:
            try:
                await ws.send_json(message)
            except Exception:
                stale.add(ws)
        # 清理断开连接
        for ws in stale:
            self._rooms[room].discard(ws)
        if not self._rooms.get(room):
            del self._rooms[room]
```

**第 26 行**：`broadcast` 是异步的，接收房间名和字典消息。

**第 27 行**：文档字符串。

**第 28-29 行**：房间不存在则直接返回（没有连接需要广播）。

**第 30 行**：`stale` 集合用于收集已断开的连接。为什么需要单独收集？因为在迭代 `Set` 的过程中删除元素会导致 `RuntimeError: Set changed size during iteration`。

**第 31-35 行**：遍历房间中所有 WebSocket 连接。`send_json(message)` 内部会调用 `json.dumps()` 序列化字典，然后发送。如果发送失败（连接已断开、网络异常等），将连接加入 `stale` 集合。这里捕获 `Exception` 而不是 `WebSocketDisconnect`，因为连接可能因各种原因失败（如 `BrokenPipeError`、`ConnectionResetError`）。

**第 37-38 行**：遍历 `stale` 集合，从房间中删除断开的连接。再次使用 `discard` 保证安全。

**第 39-40 行**：`self._rooms.get(room)` 使用 `dict.get()` 防止键不存在时抛出 `KeyError`。如果房间空了，删除这个键。

```python
    async def broadcast_seat_update(self, event_id: int, seat_info: dict):
        """广播座位更新"""
        await self.broadcast(f"seats:{event_id}", {
            "type": "seat_update",
            "data": seat_info,
        })

# 全局单例
manager = ConnectionManager()
```text

**第 42-47 行**：`broadcast_seat_update` 是一个便捷方法，将事件 ID 格式化为 `seats:{event_id}` 的房间名，并包装消息为 `{"type": "seat_update", "data": seat_info}` 格式。

**第 50-51 行**：模块级单例。在 FastAPI 应用启动时，这个 `manager` 对象会在内存中共享。**注意**：在多进程部署中，每个进程有独立的 `manager` 实例，需要通过 Redis PubSub 实现跨实例广播。

### 3.2 Stale 连接清理算法

```python
# 脏连接清理的时间复杂度分析
def broadcast_stale_clearance_analysis():
    """
    设房间中有 N 个连接，其中 K 个已断开（stale）。

    时间复杂度：
    - 遍历: O(N)
    - 清理: O(K)
    - 总计: O(N + K) = O(N)  (因为 K <= N)

    空间复杂度：
    - stale 集合: O(K)

    最佳情况：所有连接正常，K=0，O(N)
    最差情况：所有连接断开，K=N，O(2N)

    每次广播都做清理是合理的：
    - 无需额外的心跳扫描线程
    - 断开的连接最多存活一个广播周期
    - 保证 _rooms 中始终保存有效连接
    """
```

---

## 4. WebSocket 路由消息协议设计

我们项目中的 `app/routers/ws.py` 定义了 WebSocket 端点和消息协议。

### 4.1 源码逐行分析

```python
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

from app.services.connection_manager import manager
from app.middleware.auth import get_current_user
from app.models.user import User

router = APIRouter()
```text

**第 1-2 行**：导入 FastAPI 的 WebSocket 相关类。

**第 4-6 行**：导入项目中的 `ConnectionManager` 单例、认证中间件和用户模型。

**第 8 行**：创建 APIRouter 实例。

```python
@router.websocket("/shows/{event_id}/seats")
async def seat_websocket(websocket: WebSocket, event_id: int):
    """WebSocket 端点：实时座位更新"""
    room = f"seats:{event_id}"
    await manager.connect(websocket, room)
```

**第 10-11 行**：`@router.websocket()` 装饰器定义一个 WebSocket 端点。路径参数 `event_id` 从 URL 中提取。

**第 13 行**：构造房间名（命名规范：`"seats:{event_id}"`）。

**第 14 行**：调用 `manager.connect()` 接受 WebSocket 连接并加入房间。

```python
    try:
        while True:
            data = await websocket.receive_json()
            # 客户端发来的消息（如选座/取消选座）
            msg_type = data.get("type")
```text

**第 15-16 行**：进入消息循环，持续接收客户端消息。

**第 17 行**：`receive_json()` 自动解析 JSON 字符串为 Python 字典。

**第 18-19 行**：从消息字典中提取 `type` 字段，根据不同的消息类型执行不同的逻辑。

```python
            if msg_type == "select":
                await manager.broadcast(room, {
                    "type": "seat_selected",
                    "seat": data.get("seat"),
                    "user": data.get("user", "anonymous"),
                })
            elif msg_type == "release":
                await manager.broadcast(room, {
                    "type": "seat_released",
                    "seat": data.get("seat"),
                })
            elif msg_type == "ping":
                await websocket.send_json({"type": "pong"})
```

**第 20-25 行**：**select 协议**：当用户选座时，广播 `"type": "seat_selected"` 消息给房间内所有客户端。消息包含座位信息和用户标识，让其他客户端可以看到谁选了座。

**第 26-31 行**：**release 协议**：当用户取消选座时，广播 `"type": "seat_released"` 消息。

**第 32-33 行**：**ping/pong 协议**：客户端发送 `ping`，服务器立即回复 `pong`。这是应用层的心跳机制。

```python
    except WebSocketDisconnect:
        manager.disconnect(websocket, room)
    except Exception:
        manager.disconnect(websocket, room)
```text

**第 34-36 行**：无论是正常的 `WebSocketDisconnect` 还是其他异常，都从房间中移除连接。这种"宽泛"的异常捕获确保连接不会在 `_rooms` 中变成僵尸连接。

### 4.2 消息协议完整文档

**客户端 → 服务器：**

| type | 字段 | 说明 |
| ------ | ------ | ------ |
| `select` | `seat` (str)、`user` (str) | 选座通知 |
| `release` | `seat` (str) | 释放座位 |
| `ping` | 无 | 心跳探测 |

**服务器 → 客户端：**

| type | 字段 | 说明 |
| ------ | ------ | ------ |
| `seat_selected` | `seat` (str)、`user` (str) | 有人选座 |
| `seat_released` | `seat` (str) | 有人释放座位 |
| `seat_update` | `data` (dict) | 后台座位状态变更 |
| `pong` | 无 | 心跳响应 |

### 4.3 心跳超时检测算法

应用层心跳是检测死连接的关键。我们的协议要求客户端定期发送 `ping`。

**心跳检测的两种策略：**

**策略一：客户端主动 ping（本项目的选择）**

```

客户端每 30 秒发送一次 {"type": "ping"}
服务器立即回复 {"type": "pong"}

如果 WebSocket receive 超时（没有收到任何消息），
服务器可以选择断开连接。

优点：实现简单，服务器无需做定时任务
缺点：依赖客户端配合

```text

**策略二：服务器主动检测（更可靠）**

```python
import asyncio

async def heartbeat_check(websocket: WebSocket, timeout: float = 60.0):
    """
    服务器端心跳检测
    时间复杂度: O(1) per check, O(1) per connection
    空间复杂度: O(1)
    """
    last_pong = time.time()

    async def receive_pongs():
        nonlocal last_pong
        try:
            while True:
                data = await websocket.receive_json()
                if data.get("type") == "pong":
                    last_pong = time.time()
        except WebSocketDisconnect:
            pass

    # 后台任务：持续接收 pong
    receive_task = asyncio.create_task(receive_pongs())

    try:
        while True:
            await asyncio.sleep(15)  # 每 15 秒检查一次
            elapsed = time.time() - last_pong
            if elapsed > timeout:
                # 超时，断开连接
                await websocket.close(code=1001, reason="Heartbeat timeout")
                break
    finally:
        receive_task.cancel()

# 在路由中使用
@router.websocket("/ws/with-heartbeat")
async def ws_with_heartbeat(websocket: WebSocket):
    await websocket.accept()
    heartbeat_task = asyncio.create_task(heartbeat_check(websocket, timeout=60))
    try:
        while True:
            data = await websocket.receive_json()
            # 处理业务消息
            ...
    finally:
        heartbeat_task.cancel()
```

**算法分析与证明：**

```python
def heartbeat_timeout_analysis():
    """
    心跳超时检测算法分析

    前提：
    - 检测间隔 Δt_check = 15s
    - 超时阈值 t_timeout = 60s
    - 客户端 ping 间隔 Δt_ping = 30s

    正确性证明：
    假设在时刻 t_last_pong 收到最后一次 pong。
    如果在时刻 t = t_last_pong + t_timeout + ε 仍然存活，
    则服务器会在 t + Δt_check 关闭连接。

    最大误杀延迟：t_timeout + Δt_check = 60 + 15 = 75s
    即断开的连接最多存活 75 秒。

    时间复杂度：
    - 每次检查: O(1)
    - 每个连接: 服务器维护 1 个 asyncio.Task
    - N 个连接: O(N) 任务，每个 O(1)

    空间复杂度：
    - 每个连接: O(1) 用于存储 last_pong
    - N 个连接: O(N)
    """
```text

---

## 5. SSE 服务器推送事件

### 5.1 SSE 协议格式

SSE (Server-Sent Events) 是 HTML5 规范中定义的一种轻量级服务器推送技术。它使用纯文本流在单个 HTTP 连接上推送事件。

**格式：text/event-stream**

```

data: This is a message\n\n

data: {"username": "bob", "timestamp": 123456}\n\n

event: userlogon\ndata: {"username": "bob"}\n\n

id: 12345\ndata: 这条消息有 ID\n\n

retry: 5000\n\n

```text

**字段说明：**

| 字段 | 格式 | 含义 |
| ------ | ------ | ------ |
| `data` | `data: <text>\n` | 消息数据，可以有多行，拼成一个事件 |
| `event` | `event: <type>\n` | 事件类型（浏览器通过 `addEventListener` 监听） |
| `id` | `id: <string>\n` | 事件 ID，用于断线重连时的 `Last-Event-ID` |
| `retry` | `retry: <ms>\n` | 重连间隔（毫秒） |

**分隔符：**

- 每个事件以 `\n\n` 结束（两个换行符）
- 如果 `data:` 有多行，它们拼接为一个事件（每行用 `\n` 连接）

```python
# 手动构造 SSE 事件的工具函数

def format_sse_event(data: str, event: str = None, event_id: str = None) -> str:
    """构造 SSE 事件字符串"""
    lines = []
    if event_id:
        lines.append(f"id: {event_id}")
    if event:
        lines.append(f"event: {event}")
    # data 可能包含换行，每行都要加 "data: " 前缀
    for line in data.split("\n"):
        lines.append(f"data: {line}")
    lines.append("")  # 空行表示事件结束
    return "\n".join(lines)

# 示例
event_str = format_sse_event(
    data='{"msg": "hello"}',
    event="notification",
    event_id="evt-001",
)
print(repr(event_str))
# 输出: 'event: notification\nid: evt-001\ndata: {"msg": "hello"}\n\n'
```

### 5.2 浏览器 EventSource API

```javascript
// 浏览器端 SSE 客户端
const eventSource = new EventSource("/api/v1/notifications/stream");

// 监听命名事件
eventSource.addEventListener("notification", (event) => {
    const data = JSON.parse(event.data);
    console.log("新通知:", data);
});

// 监听未命名事件（不指定 event 字段的事件）
eventSource.onmessage = (event) => {
    console.log("默认事件:", event.data);
};

// 监听连接打开
eventSource.onopen = (event) => {
    console.log("SSE 连接已建立");
};

// 监听连接错误
eventSource.onerror = (event) => {
    console.error("SSE 连接出错", event);
    // 浏览器会自动尝试重连（默认 3 秒后）
};

// 手动关闭连接
// eventSource.close();
```text

**EventSource API 特性：**

- 自动重连：连接断开后，浏览器会根据 `retry` 字段自动重连
- 断点续传：开启时会发送 `Last-Event-ID` 头，服务器可以从此 ID 之后的事件开始发送
- 同源限制：默认只支持同源请求（不支持跨域）

### 5.3 FastAPI StreamingResponse 实现 SSE

```python
from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse
import asyncio
import json
from typing import AsyncGenerator

router = APIRouter()


async def event_stream_generator(user_id: int) -> AsyncGenerator[str, None]:
    """
    SSE 事件生成器（无限流）

    Args:
        user_id: 用户 ID，用于查询该用户的通知

    Yields:
        SSE 格式的字符串

    此函数在 FastAPI 的 StreamingResponse 中使用，
    每个 await 都会产生一个 HTTP chunk 发送给客户端。
    """
    last_event_id = 0
    while True:
        # 模拟查询新通知（实际项目会查询数据库或 Redis 队列）
        notification = await check_user_notifications(user_id, after=last_event_id)
        if notification:
            last_event_id = notification["id"]
            yield format_sse_event(
                data=json.dumps(notification, ensure_ascii=False),
                event="notification",
                event_id=str(last_event_id),
            )
        else:
            # 没有新通知时，发送注释保持连接存活性
            yield ": heartbeat\n\n"

        await asyncio.sleep(1)  # 每秒轮询一次


@router.get("/users/{user_id}/notifications/stream")
async def notification_stream(user_id: int, request: Request):
    """SSE 端点：用户通知流"""
    # 检查客户端是否传来 Last-Event-ID（断线重连用）
    last_event_id = request.headers.get("Last-Event-ID")

    return StreamingResponse(
        event_stream_generator(user_id),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",           # 禁止缓存
            "Connection": "keep-alive",             # 长连接
            "X-Accel-Buffering": "no",              # 禁用 Nginx 缓冲（关键！）
            "Access-Control-Allow-Origin": "*",      # CORS（如需要跨域）
        },
    )
```

**关键配置说明：**

1. **`Cache-Control: no-cache`**：禁止中间代理缓存 SSE 数据。
2. **`X-Accel-Buffering: no`**：当使用 Nginx 反向代理时，Nginx 默认会缓冲响应。对于 SSE，缓冲会导致客户端延迟收到数据。这个头告诉 Nginx 不要缓冲。
3. **`text/event-stream`**：SSE 标准的 MIME 类型。

**Nginx 代理 SSE 的配置：**

```nginx
location /api/v1/users/ {
    proxy_pass http://backend;
    proxy_buffering off;          # 禁用缓冲
    proxy_cache off;              # 禁用缓存
    proxy_read_timeout 86400s;    # 长连接超时
    proxy_set_header Connection ''; # HTTP/1.1 keep-alive
    chunked_transfer_encoding on;
}
```text

### 5.4 断线重连与 Last-Event-ID

SSE 的一个关键特性是自动断线重连。当连接断开后，浏览器会自动重新连接，并在 HTTP 头中发送 `Last-Event-ID`：

```python
@router.get("/events/stream")
async def event_stream(request: Request):
    """SSE 端点：支持断线重连"""
    # 客户端自动发送的头部
    last_id = request.headers.get("Last-Event-ID")

    async def generate():
        # 从 last_id 之后的事件开始发送
        stream_id = int(last_id) if last_id else 0

        while True:
            stream_id += 1
            yield f"id: {stream_id}\n"
            yield f"data: 事件 #{stream_id}\n\n"
            await asyncio.sleep(2)

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache"},
    )
```

### 5.5 SSE vs WebSocket 对比表

| 特性 | SSE | WebSocket |
| ------ | ----- | ----------- |
| **通信方向** | 服务器→客户端（单向） | 全双工（双向） |
| **协议** | HTTP（标准 REST 兼容） | WebSocket（HTTP Upgrade） |
| **数据格式** | 纯文本（UTF-8） | 文本或二进制帧 |
| **浏览器支持** | 所有现代浏览器 | 所有现代浏览器 |
| **自动重连** | 内置（EventSource 自动处理） | 需要手动实现 |
| **断点续传** | 内置（Last-Event-ID） | 需要手动实现 |
| **二进制传输** | 需 Base64 编码 | 原生支持 |
| **连接数限制** | 每个域名 6 个（HTTP/1.1） | 无限制（单 TCP） |
| **实现复杂度** | 极低 | 中等 |
| **适用场景** | 通知推送、股价、日志流 | 在线游戏、聊天、协同编辑 |
| **跨域支持** | CORS 标准 | CORS + WebSocket 特有机制 |
| **反向代理** | 需关闭缓冲 | 需配置 WebSocket 代理 |
| **自定义头** | 不支持（EventSource 限制） | 支持 |

**选择建议：**

- **只用推送消息给客户端** → 选择 SSE（实现简单，自动重连）
- **需要双向实时通信** → 选择 WebSocket
- **需要二进制传输（如游戏）** → 选择 WebSocket
- **在 HTTP/2 环境下**：SSE 的语义在 HTTP/2 中效率更高（多路复用）

---

## 6. Redis PubSub 跨实例广播

### 6.1 为什么需要 Redis PubSub？

在多进程部署（如 Gunicorn + Uvicorn workers）或分布式部署中，每个进程有自己的 `ConnectionManager` 实例。一个进程中的 WebSocket 连接无法直接广播给另一个进程的连接。

```text
进程 A (port 8001)             进程 B (port 8002)
  ├── ConnectionManager A        ├── ConnectionManager B
  ├── WS-client-1                ├── WS-client-2
  ├── WS-client-2                └── WS-client-4
  └── WS-client-3
         │                              │
         └────── Redis PubSub ──────────┘
                    channel: "seat_update:1"
```

### 6.2 实现代码

```python
import json
import asyncio
from redis.asyncio import Redis as AsyncRedis
from app.services.connection_manager import manager


class RedisPubSubBroadcast:
    """Redis PubSub 实现跨实例 WebSocket 广播"""

    def __init__(self, redis: AsyncRedis, show_id: int):
        self.redis = redis
        self.show_id = show_id
        self.pubsub = None
        self.channel = f"seat_update:{show_id}"
        self._running = False

    async def subscribe(self):
        """订阅频道"""
        self.pubsub = self.redis.pubsub()
        await self.pubsub.subscribe(self.channel)

    async def publish(self, message: dict):
        """发布消息到频道"""
        await self.redis.publish(self.channel, json.dumps(message))

    async def get_message(self):
        """获取一条消息（非阻塞）"""
        if self.pubsub:
            return await self.pubsub.get_message(
                ignore_subscribe_messages=True
            )
        return None

    async def listen_loop(self):
        """
        持续监听 Redis PubSub 消息并在本地广播

        这是事件循环的核心：
        1. 接收 Redis PubSub 消息
        2. 在本地 ConnectionManager 广播给本机进程内的 WebSocket 客户端

        设计权衡：
        - 消息最多传送一次（at-most-once delivery）
        - 如果 Redis 崩溃，消息丢失
        - 但 WebSocket 是持久连接，可以接受偶然断连
        """
        self._running = True
        while self._running:
            message = await self.get_message()
            if message:
                data = json.loads(message["data"])
                # 在本进程广播
                await manager.broadcast(
                    f"seats:{self.show_id}",
                    data,
                )
            await asyncio.sleep(0.01)  # 避免 busy-wait
```text

### 6.3 Redis PubSub At-Most-Once 投递证明

```python
def pubsub_delivery_proof():
    """
    Redis PubSub 最多一次投递 (At-Most-Once Delivery) 证明

    定义：
    - PubSub 消息从 publisher 发出，subscriber 接收
    - 没有 ACK 机制
    - 如果 subscriber 在处理消息前崩溃，消息丢失

    证明：
    设 publisher 在时间 t_p 发送消息 m。
    Redis 服务器在时间 t_r 收到 m（t_r ≈ t_p）。
    Redis 将 m 发送给所有当前在线的 subscriber。

    情况 1：subscriber 在线
      t_p → Redis → t_r → subscriber → 处理成功
      最多一次 ✓

    情况 2：subscriber 在 t_r 后立即崩溃
      t_p → Redis → t_r → subscriber 崩溃（未处理）
      消息 m 丢失
      最多一次 ✓（消息至多投递一次，但可能丢失）

    情况 3：subscriber 掉线后重连
      t_p → Redis → t_r → subscriber 离线
      Redis 不会缓存消息（PubSub 没有消息持久化）
      重连后不会收到 m
      最多一次 ✓（消息至多投递一次）

    结论：Redis PubSub 保证 at-most-once 投递。
    如果需要精确一次投递，需要使用 Redis Streams 或消息队列（如 ARQ）。
    """
```

### 6.4 集成到 FastAPI lifespan

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期管理"""
    # startup
    redis = await get_redis()
    # 为每个活动创建 Broadcast 并启动监听
    broadcasters = {}
    # ...

    yield

    # shutdown
    for b in broadcasters.values():
        b._running = False
```text

---

## 7. ARQ 异步任务队列

### 7.1 ARQ vs Celery 对比

| 特性 | ARQ | Celery |
| ------ | ----- | -------- |
| **依赖** | Redis only | Redis/RabbitMQ/SQS |
| **异步** | 原生 asyncio | 需线程/进程池 |
| **序列化** | msgpack（默认） | JSON/Pickle/msgpack |
| **重试** | 内置 `@retry` 装饰器 | 内置，更丰富 |
| **定时任务** | `cron` 表达式 | Celery Beat |
| **任务去重** | 需手动实现 | 部分支持 |
| **监控** | 基础（ARQ Web UI） | Flower（完善） |
| **性能** | 约 10K tasks/s | 约 20K tasks/s |
| **包大小** | 极小 | 较大 |
| **学习曲线** | 低 | 中高 |
| **适用场景** | 异步微服务、高吞吐 | 大规模任务分发 |

**为什么选 ARQ？**

1. **原生异步**：ARQ 完全基于 `asyncio`，与 FastAPI 的异步模型天然契合。不需要像 Celery 那样启动额外的 worker 进程池。
2. **简单**：只需要 Redis，无需 RabbitMQ 等消息中间件。配置极少。
3. **性能足够**：10K tasks/s 对票务系统完全够用。

### 7.2 安装与基本使用

```bash
pip install arq
```

### 7.3 核心概念

**函数定义：**

```python
# functions.py — 任务函数
async def my_task(ctx, arg1: int, arg2: str):
    """
    任务函数签名：
    - 第一个参数必须是 ctx（任务上下文）
    - 后续参数是任务参数
    - 返回值会被记录（用于调试）
    """
    redis = ctx["redis"]  # 从上下文中获取资源
    await redis.set(f"task:{arg1}", arg2)
    return {"result": "ok", "arg1": arg1}
```text

**WorkerSettings：**

```python
# worker.py — Worker 配置
from arq.connections import RedisSettings

class WorkerSettings:
    functions = [my_task]                     # 注册的任务函数列表
    redis_settings = RedisSettings(
        host="localhost",
        port=6379,
        database=0,
    )
    on_startup = startup                      # Worker 启动时调用
    on_shutdown = shutdown                    # Worker 关闭时调用
    poll_delay = 0.5                          # 轮询间隔（秒）
    max_tries = 3                             # 默认最大重试次数
    max_burst_jobs = 10                       # burst 模式下最大任务数
    job_timeout = 300                         # 任务超时（秒）默认 300
    keep_result = 3600                        # 结果保留时间（秒）
    keep_result_forever = False               # 是否永久保留结果
```

**提交任务：**

```python
# 在 FastAPI 路由中提交任务
from arq import create_pool
from arq.connections import RedisSettings

async def submit_example():
    pool = await create_pool(RedisSettings())
    # 提交任务，enqueue_job 返回 Job 对象
    job = await pool.enqueue_job("my_task", 42, "hello")
    # 获取任务结果（阻塞等待）
    result = await job.result()     # 等待任务完成
    print(f"任务结果: {result}")
```text

### 7.4 重试机制

```python
from arq import cron
from arq.connections import RedisSettings

async def might_fail(ctx, data: str):
    """可能会失败的任务，带重试"""
    # 如果任务抛出异常，ARQ 会自动重试
    # retry_count 是当前重试次数（从 0 开始）
    if ctx["job_try"] > 0:
        print(f"第 {ctx['job_try']} 次重试")

    if data == "fail":
        raise ValueError("失败了！")
    return "success"

class WorkerSettings:
    functions = [might_fail]
    # 全局重试设置
    max_tries = 5        # 最多重试 5 次
    redis_settings = RedisSettings()

# 或者使用装饰器控制每次调用的重试
@arq.retry(max_tries=5, max_delay=60.0)
async def send_email(ctx, email: str):
    """发送邮件，最多重试 5 次"""
    # ...
```

### 7.5 Cron 定时任务

```python
from arq import cron

async def clean_expired_sessions(ctx):
    """每天凌晨 2 点清理过期会话"""
    redis = ctx["redis"]
    await redis.delete("expired_sessions")

async def generate_daily_report(ctx):
    """每天上午 9 点生成日报"""
    print("生成日报...")

class WorkerSettings:
    functions = [clean_expired_sessions, generate_daily_report]
    # Cron 任务定义
    cron_jobs = [
        cron(
            clean_expired_sessions,  # 任务函数
            hour=2,                   # 凌晨 2 点
            minute=0,
            description="清理过期会话",
        ),
        cron(
            generate_daily_report,
            hour=9,
            minute=0,
            description="生成日销售报表",
        ),
    ]
    redis_settings = RedisSettings()
```text

### 7.6 创建任务队列

```python
from arq import create_pool
from arq.connections import RedisSettings

# 创建连接池（单例）
_pool = None

async def get_pool():
    global _pool
    if _pool is None:
        _pool = await create_pool(RedisSettings())
    return _pool

async def schedule_order_timeout(order_no: str, timeout_minutes: int = 30):
    """调度订单超时任务：30 分钟后检查支付状态"""
    pool = await get_pool()
    # enqueue_job(fn_name, *args, _defer_by=timedelta)
    # _defer_by 表示延迟执行
    from datetime import timedelta
    await pool.enqueue_job(
        "release_expired_seats",
        order_no,
        _defer_by=timedelta(minutes=timeout_minutes),
    )
```

### 7.7 项目 Worker 源码逐行分析

```python
"""ARQ 后台任务 Worker"""

from redis.asyncio import Redis
from arq import create_pool
from arq.connections import RedisSettings
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.database import async_session_factory
```text

**第 1 行**：模块文档字符串。

**第 3-6 行**：导入依赖：`Redis` 异步客户端、`arq` 的连接池和设置类、SQLAlchemy 异步会话类型。

**第 8-9 行**：导入项目配置和数据库会话工厂。

```python
async def send_order_confirmation(ctx, order_no: str, user_id: int):
    """发送订单确认通知"""
    async with async_session_factory() as session:
        # 实际项目会调用通知服务
        print(f"Order confirmed: {order_no} for user {user_id}")
        return {"status": "ok", "order_no": order_no}
```

**第 12 行**：定义 `send_order_confirmation` 函数，接收 `ctx`（任务上下文）和 `order_no`、`user_id` 参数。

**第 13 行**：文档字符串。

**第 14 行**：使用 `async_session_factory()` 创建数据库会话（上下文管理器自动关闭）。

**第 15-16 行**：实际项目中会调用通知服务发送邮件/短信/推送。这里用 `print` 模拟。ARQ 的返回值会被记录到 JobResult 中。

**第 17 行**：返回执行结果。

```python
async def release_expired_seats(ctx, event_id: int, quantity: int):
    """释放过期未支付的座位"""
    redis: Redis = ctx["redis"]
    stock_key = f"stock:{event_id}"
    await redis.incrby(stock_key, quantity)
    print(f"Released {quantity} seats for event {event_id}")
    return {"status": "ok", "event_id": event_id, "released": quantity}
```text

**第 20 行**：定义 `release_expired_seats` 函数。这个任务在订单超时未支付时调用。

**第 21 行**：文档字符串。

**第 22 行**：从 `ctx["redis"]` 获取 Redis 连接。这是在 `startup` 函数中注入的。

**第 23 行**：构造库存 Redis key：`stock:{event_id}`。

**第 24 行**：`INCRBY` 原子增加库存（释放 N 张票）。

**第 25 行**：打印日志。

**第 26 行**：返回结果。

```python
async def startup(ctx):
    """Worker 启动时创建 Redis 连接"""
    ctx["redis"] = await create_pool(
        RedisSettings(
            host=settings.REDIS_HOST,
            port=settings.REDIS_PORT,
            database=settings.REDIS_DB,
        )
    )
```

**第 29 行**：定义 `startup` 函数。Worker 进程启动时自动调用。

**第 30 行**：文档字符串。

**第 31-36 行**：创建 ARQ 的 Redis 连接池，存入 `ctx["redis"]`。这样任务函数可以通过 `ctx["redis"]` 访问 Redis。

```python
async def shutdown(ctx):
    """Worker 关闭时清理连接"""
    if "redis" in ctx:
        await ctx["redis"].close()
```text

**第 40 行**：定义 `shutdown` 函数。Worker 关闭时调用。

**第 41 行**：文档字符串。

**第 42-43 行**：如果 `ctx` 中有 Redis 连接，关闭它。

```python
class WorkerSettings:
    """ARQ Worker 配置"""
    functions = [send_order_confirmation, release_expired_seats]
    on_startup = startup
    on_shutdown = shutdown
    redis_settings = RedisSettings(
        host=settings.REDIS_HOST,
        port=settings.REDIS_PORT,
        database=settings.REDIS_DB,
    )
```

**第 47 行**：定义 `WorkerSettings` 类。ARQ 使用这个类来配置 Worker。

**第 48 行**：文档字符串。

**第 49 行**：`functions` 列表注册所有可执行的任务函数。ARQ 会将这些函数暴露给客户端调用。

**第 50 行**：注册 `startup` 回调。

**第 51 行**：注册 `shutdown` 回调。

**第 52-55 行**：配置 Redis 连接参数。

### 7.8 启动 Worker

```bash
# 启动 ARQ Worker
arq app.worker.WorkerSettings

# 输出：
# Starting worker for app.worker.WorkerSettings
# Connected to redis at localhost:6379
# Listening for jobs...
```text

### 7.9 在路由中调度 ARQ 任务

```python
# 在订单创建成功后调度超时释放任务
from arq import create_pool
from arq.connections import RedisSettings
from datetime import timedelta

async def schedule_order_timeout(order_no: str, event_id: int, quantity: int):
    """订单创建后，调度 30 分钟超时释放任务"""
    pool = await create_pool(RedisSettings())
    await pool.enqueue_job(
        "release_expired_seats",        # 任务函数名（必须是 WorkerSettings.functions 中的函数）
        event_id,                       # 参数
        quantity,                       # 参数
        _defer_by=timedelta(minutes=30), # 延迟 30 分钟执行
        _job_id=f"order:timeout:{order_no}",  # 任务 ID（用于去重）
    )
```

---

## 8. 实时座位更新 — 端到端延迟计算

```python
def realtime_latency_analysis():
    """
    实时座位更新端到端延迟计算

    场景：用户 A 选中座位 S，用户 B 实时看到座位被选中

    时序图：
    User A                   FastAPI                  Redis            User B
      │                         │                       │                │
      │── {"type":"select"} ───→│                       │                │
      │                         │── broadcast(room) ───│                │
      │                         │── publish(channel) ──→│                │
      │                         │                       │── sub → ─────→│
      │                         │                       │                │── send_json
      │                         │                       │                │
      │←─── {"seat_selected"} ──│                       │                │

    延迟分解：

    1. 网络传输（客户 A → 服务器）
       假设在同一内网（或 localhost）：< 0.5ms
       公网（假设 30ms ping）：~15ms（单向）

    2. WebSocket 帧解码（服务器）:
       receive_json() → json.loads()
       数据量极小（< 500 bytes），~0.01ms

    3. 广播到同房间客户端（本地）：
       manager.broadcast():
         遍历房间连接: O(N)，N 为连接数
         假设 N=1000 连接，每个 send_json 约 0.1ms
         串行发送: 1000 × 0.1ms = 100ms
         并行发送（asyncio.gather）: ~0.1ms（最慢连接）

    4. Redis PubSub（跨进程）：
       publish → Redis 服务器 → subscriber
       内网：~0.5ms

    总延迟计算：

    最佳情况（内网，同进程，少量连接）：
    T = 0.01(网络) + 0.01(解码) + 0.1(广播) + 0(无 Redis) = 0.12ms

    典型情况（内网，多进程，1000 连接）：
    T = 0.01 + 0.01 + 0.1 + 0.5 = 0.62ms

    最差情况（公网，跨进程，大量连接）：
    T = 15 + 0.01 + 100(串行) + 0.5 = ~115ms
    T = 15 + 0.01 + 0.1(并行) + 0.5 = ~15.6ms

    结论：使用 asyncio.gather 并行广播可以将延迟
    从 O(N) 降低到 O(1)（受最慢连接限制）。

    优化建议：对 N > 1000 的房间，使用批量广播：
    async def broadcast_parallel(room, message, batch_size=100):
        connections = list(manager._rooms[room])
        for i in range(0, len(connections), batch_size):
            batch = connections[i:i+batch_size]
            await asyncio.gather(*[
                ws.send_json(message) for ws in batch
            ])
    """
```text

---

## 9. WebSocket 握手验证代码

```python
import hashlib
import base64
import struct


def compute_accept_key(client_key: str) -> str:
    """
    计算 Sec-WebSocket-Accept

    算法（RFC 6455 Section 4.2.2）：
    accept = BASE64(SHA1(key + "258EAFA5-E914-47DA-95CA-5AB5DC11B735"))

    Args:
        client_key: 客户端发送的 Sec-WebSocket-Key

    Returns:
        计算得到的 Sec-WebSocket-Accept 值
    """
    MAGIC_UUID = "258EAFA5-E914-47DA-95CA-5AB5DC11B735"
    sha1 = hashlib.sha1((client_key + MAGIC_UUID).encode()).digest()
    return base64.b64encode(sha1).decode()


def test_websocket_handshake():
    """
    验证握手算法的正确性

    RFC 6455 第 4.2.2 节给出了测试向量：
    Key: "dGhlIHNhbXBsZSBub25jZQ=="
    Accept: "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
    """
    client_key = "dGhlIHNhbXBsZSBub25jZQ=="
    expected_accept = "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="

    computed_accept = compute_accept_key(client_key)

    assert computed_accept == expected_accept, (
        f"握手计算错误：期望 {expected_accept}，得到 {computed_accept}"
    )
    print("握手算法验证通过！")


def encode_websocket_frame(
    payload: bytes,
    opcode: int = 0x1,
    mask: bool = False,
    masking_key: bytes = None,
    fin: bool = True,
) -> bytes:
    """
    手动编码 WebSocket 帧

    Args:
        payload: 载荷数据
        opcode: 帧类型（0x1=text, 0x2=binary, 0x8=close, 0x9=ping, 0xA=pong）
        mask: 是否掩码
        masking_key: 4 字节掩码密钥（如果 mask=True）
        fin: 是否为结束帧

    Returns:
        编码后的 WebSocket 帧字节序列
    """
    frame = bytearray()

    # 第一字节：FIN + RSV + opcode
    first_byte = (0x80 if fin else 0x00) | (opcode & 0x0F)
    frame.append(first_byte)

    # 第二字节：MASK + 载荷长度
    mask_bit = 0x80 if mask else 0x00
    length = len(payload)

    if length < 126:
        frame.append(mask_bit | length)
    elif length < 65536:
        frame.append(mask_bit | 126)
        frame.extend(struct.pack(">H", length))
    else:
        frame.append(mask_bit | 127)
        frame.extend(struct.pack(">Q", length))

    if mask:
        if masking_key is None:
            masking_key = secrets.token_bytes(4)
        frame.extend(masking_key)
        payload = bytes(
            payload[i] ^ masking_key[i % 4]
            for i in range(length)
        )

    frame.extend(payload)
    return bytes(frame)


# 测试编码/解码
def test_frame_encoding():
    """验证编码后的帧可以被正确解码"""
    test_cases = [
        (b"Hello", 0x1, True),      # 文本，掩码
        (b"World", 0x1, False),     # 文本，无掩码
        (b"\x00\x01\x02", 0x2, True),  # 二进制，掩码
        (b"Big" * 100, 0x1, False), # 长文本（超 126 字节）
    ]

    for payload, opcode, mask in test_cases:
        frame = encode_websocket_frame(payload, opcode, mask)
        parsed = parse_websocket_frame(frame)
        assert parsed["payload"] == payload, (
            f"编解码失败: 期望 {payload}, 得到 {parsed['payload']}"
        )
    print("帧编解码测试通过！")
```

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/routers/ws.py` | WebSocket 路由：CSWSH Origin 校验、JWT 查询参数验证、5 层安全 |
| `app/services/connection_manager.py` | ConnectionManager：房间级广播、stale 清理、seat/order 广播 |
| `app/worker.py` | ARQ Worker 定义：`auto_cancel_expired_orders()` 超时取消 |
| `app/services/order_service.py` | `cancel_order()` 和库存归还逻辑（被 Worker 调用） |

**ConnectionManager 5 层 WebSocket 安全**（详见 `app/routers/ws.py`）：

1. Origin 头校验（防止 CSWSH）
2. JWT 令牌验证（查询参数传递）
3. 消息大小限制（防止内存耗尽）
4. 消息类型白名单（仅允许 subscribe/unsubscribe/ping）
5. Seat ID 格式校验（仅允许纯数字）

## 10. 本章总结

**已掌握的知识：**

- **WebSocket 协议**：HTTP Upgrade 握手、帧格式（FIN/opcode/MASK/长度编码）、掩码算法及安全原理、分片机制、关闭握手和状态码
- **FastAPI WebSocket**：accept/receive/send API、WebSocketDisconnect 处理、路径参数路由
- **ConnectionManager**：房间级广播、stale 连接清理、单例模式
- **消息协议设计**：select/release/ping-pong、应用层心跳、时间复杂度分析
- **SSE**：text/event-stream 格式、EventSource 浏览器 API、StreamingResponse 实现、断线重连
- **ARQ 任务队列**：WorkerSettings、create_pool、enqueue_job、重试机制、Cron 定时任务
- **Redis PubSub**：跨实例广播、At-Most-Once 投递证明
- **端到端延迟计算**：延迟分解、串行 vs 并行广播的时间复杂度

**项目里程碑：**

- [x] WebSocket 实时座位更新（`app/routers/ws.py`）
- [x] ConnectionManager 房间级广播（`app/services/connection_manager.py`）
- [x] ARQ 后台任务 Worker（`app/worker.py`）
- [ ] SSE 用户通知流
- [ ] Redis PubSub 跨实例广播集成

---

## 安全防护

### WebSocket 安全措施

本系统 WebSocket 端点 (`/ws/shows/{event_id}/seats`) 实现了以下安全防护：

#### 1. CSWSH 防护（Cross-Site WebSocket Hijacking）

WebSocket 不受同源策略限制，恶意网站可以打开 WebSocket 连接到任意后端。我们在 `ws.py` 中实现了 Origin 头校验：

```python
async def verify_ws_origin(websocket: WebSocket) -> bool:
    origin = websocket.headers.get("origin", "")
    if not origin:
        return True  # 非浏览器客户端放行
    allowed = {"http://localhost:5173", "http://localhost:8000"}
    return any(origin.startswith(a) for a in allowed)
```text

#### 2. JWT 令牌认证

通过查询参数 `?token=xxx` 传递 JWT，连接时校验：

```python
if token:
    payload = decode_token(token)
    if payload and payload.get("type") == "access":
        user_id = int(payload["sub"])
        current_user = User(id=user_id)
```

#### 3. 消息安全措施

| 措施 | 实现位置 |
| ------ | --------- |
| 消息大小限制 | 最大 64KB（防内存耗尽 DoS） |
| 消息类型白名单 | 仅允许 select/release/ping |
| 座位 ID 格式校验 | 大写字母+数字，长度 2-5 |
| 用户身份防伪造 | user 从 JWT 提取，非客户端提供 |
| 异常日志记录 | 捕获异常时记录详细日志（不静默吞没） |

#### 4. Seats API 安全

`GET /api/v1/events/{id}/seats` 只读端点返回座位的公开信息（状态、价格），不会暴露任何敏感数据。
**下一章预告：** 第 9 章将学习 pytest 测试框架，实现单元测试、集成测试和并发防超卖测试。
