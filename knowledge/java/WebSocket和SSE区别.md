# WebSocket 和 SSE 区别

## 一句话结论

- **WebSocket：客户端和服务端都可以主动发送消息，是双向通信。**
- **SSE（Server-Sent Events）：服务端可以持续向客户端推送消息，但客户端不能沿着同一条 SSE 连接反向发送消息。**

---

## 1. SSE 和 WebSocket 的核心区别

| 对比项 | WebSocket | SSE |
|---|---|---|
| 通信方向 | 双向通信 | 服务端 → 客户端单向推送 |
| 协议 | 先 HTTP 握手，再升级成 WebSocket 协议 | 始终基于 HTTP |
| 连接 | 长连接 | 长 HTTP 连接 |
| 客户端发送消息 | 可以直接通过同一条 WebSocket 连接发送 | 需要额外发起 HTTP 请求 |
| 数据类型 | 文本、二进制都支持 | 主要是文本 |
| 断线重连 | 通常需要自己处理 | 浏览器 `EventSource` 原生支持自动重连 |
| 实现复杂度 | 较高 | 较低 |
| 典型场景 | 聊天、游戏、在线协作 | AI 流式输出、消息通知、日志推送、任务进度 |

---

## 2. SSE 的通信流程

客户端先发起一个 HTTP 请求：

```text
客户端
   |
   | GET /stream
   ↓
服务端
```

服务端返回 Response，但这个 Response 不马上结束，而是持续写数据：

```text
客户端
   |
   | GET /stream
   ↓
服务端
   |
   | data: 你好
   | data: ，我是
   | data: ChatGPT
   ↓
客户端
```

因此 SSE 本质上是：

```text
客户端 → 服务端：普通 HTTP Request
服务端 → 客户端：持续不结束的 HTTP Response
```

如果客户端还要向服务端发送新数据，需要另外再发一个 HTTP 请求：

```text
POST /chat
```

所以 LLM 常见的流式架构是：

```text
客户端 --POST 提交问题--> 服务端
客户端 <--SSE 持续返回 token-- 服务端
```

---

## 3. 为什么长 HTTP 连接不能天然双向？

关键点：

> **TCP 能双向，不代表 HTTP 这一层允许双方随时互相发送应用层消息。**

TCP 本身是全双工：

```text
客户端 <==========> 服务端
          TCP
```

两个方向都可以同时传输字节。

但是 HTTP 在 TCP 之上定义的是 Request-Response 模型：

```text
客户端
   |
   | Request
   ↓
服务端
   |
   | Response
   ↓
客户端
```

SSE 只是把 Response 拉得很长：

```text
Request
   ↓
Response
   ├── data: A
   ├── data: B
   ├── data: C
   └── ...一直不结束
```

因此服务端可以不断往 Response 中写数据。

但是客户端不能在这个 Response 中突然反向插入一条应用层消息：

```text
服务端
   ↓ data: A
客户端
   ↑ 我也往这个 Response 里发一句
```

HTTP 没有定义这种通信方式。

如果客户端需要继续发数据，它要重新发起新的 HTTP Request。

所以：

> **SSE 的单向不是 TCP 做不到双向，而是 HTTP 应用层仍然遵循 Request-Response 语义。**

---

## 4. WebSocket 为什么可以双向？

WebSocket 一开始也是通过 HTTP 建立连接：

```text
客户端
   |
   | HTTP Upgrade: websocket
   ↓
服务端
   |
   | 101 Switching Protocols
   ↓
```

升级完成后，双方不再按照普通 HTTP 的 Request-Response 消息模型通信，而是使用 WebSocket Frame：

```text
客户端
   ↓ ↑
   ↓ ↑
服务端
```

客户端和服务端都可以随时主动发送消息。

所以 WebSocket 的关键不是“TCP 才能双向”，而是：

> **WebSocket 在 HTTP 握手完成后，脱离了普通 HTTP Request-Response 模型。**

---

## 5. 协议层级关系

SSE：

```text
SSE
 ↓
HTTP
 ↓
TCP
```

WebSocket：

```text
HTTP 握手
   ↓
Upgrade
   ↓
WebSocket
   ↓
TCP
```

因此不要说：

> SSE 是 HTTP，WebSocket 是 TCP。

这个说法不准确。

它们最终都通常运行在 TCP 之上，区别主要在于**应用层协议和通信模型**。

---

## 6. 怎么选？

```text
需要服务端主动推送？
        ↓
       是
        ↓
客户端也需要在同一条连接上频繁主动发消息？
     /                 \
   是                   否
   ↓                    ↓
WebSocket              SSE
```

适合 SSE：

```text
AI 流式输出
日志推送
任务进度
消息通知
```

适合 WebSocket：

```text
聊天室
在线游戏
实时协同编辑
实时控制
```

---

## 面试回答

> SSE 和 WebSocket 都可以保持长连接，但通信模型不同。SSE 仍然基于 HTTP 的 Request-Response 模型，只是服务端让 Response 长时间不结束，因此服务端可以持续向客户端推送数据；客户端如果要发送新数据，仍然需要重新发起 HTTP Request。WebSocket 则先通过 HTTP 完成握手，再 Upgrade 到 WebSocket 协议，之后双方不再受普通 HTTP Request-Response 模型限制，可以在同一条连接上进行全双工通信。SSE 不能天然双向不是因为 TCP 不支持，而是因为 HTTP 应用层协议的通信语义限制。
