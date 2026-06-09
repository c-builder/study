# 实时通信（WebSocket / SSE / WebRTC）

## 学习目标

- 掌握三种实时通信方案的原理、选型与架构设计
- 能设计长连接体系：心跳、重连、鉴权、消息有序
- 理解 WebRTC 信令与点对点通信流程

## 为什么需要

IM、直播、协同编辑、AI 流式输出都依赖实时通信。大厂面试高频：**WebSocket 断了怎么办？SSE 和 WS 怎么选？**

## 一、三方案对比

| 方案 | 方向 | 协议 | 自动重连 | 适用 |
|------|------|------|---------|------|
| WebSocket | 双向 | ws:// | 需手动 | IM、游戏、协同 |
| SSE | 服务端→客户端 | HTTP | 浏览器自动 | 推送、AI 流式、股票行情 |
| WebRTC | 点对点 | UDP/P2P | 需 ICE 重连 | 音视频、屏幕共享 |

## 二、WebSocket 架构

```mermaid
flowchart LR
    Client[前端] <-->|ws://| Gateway[WS 网关]
    Gateway --> Broker[消息队列]
    Broker --> Services[业务服务]
```

### 连接管理（手写骨架）

```javascript
class ReconnectingWebSocket {
  constructor(url, { maxRetries = 10, heartbeat = 30000 } = {}) {
    this.url = url;
    this.retries = 0;
    this.heartbeat = heartbeat;
    this.connect();
  }
  connect() {
    this.ws = new WebSocket(this.url);
    this.ws.onopen = () => { this.retries = 0; this.startHeartbeat(); };
    this.ws.onclose = () => this.reconnect();
    this.ws.onmessage = (e) => this.onMessage?.(JSON.parse(e.data));
  }
  reconnect() {
    if (this.retries >= this.maxRetries) return;
    const delay = Math.min(1000 * 2 ** this.retries, 30000);
    setTimeout(() => { this.retries++; this.connect(); }, delay);
  }
  startHeartbeat() {
    this.pingTimer = setInterval(() => {
      if (this.ws.readyState === WebSocket.OPEN) this.ws.send(JSON.stringify({ type: 'ping' }));
    }, this.heartbeat);
  }
}
```

**鉴权**：连接时带 token（query `?token=` 或首条消息 auth）；HTTPS 页面必须用 `wss://`。

## 三、SSE 流式

```javascript
const es = new EventSource('/api/stream?topic=news');
es.onmessage = (e) => handleData(JSON.parse(e.data));
es.onerror = () => { /* 浏览器自动重连 */ };
// 关闭
es.close();
```

**限制**：仅 GET、单向；每个连接占一个 HTTP 连接（HTTP/2 多路复用可缓解）。

**AI 流式**：也可用 `fetch` + `ReadableStream` 替代 SSE，更灵活（POST body）。

## 四、WebRTC 原理

```mermaid
sequenceDiagram
    participant A as 用户A
    participant Signal as 信令服务器
    participant B as 用户B

    A->>Signal: offer SDP
    Signal->>B: 转发 offer
    B->>Signal: answer SDP
    Signal->>A: 转发 answer
    A->>B: ICE candidates 交换
    Note over A,B: P2P 连接建立，音视频直连
```

| 组件 | 作用 |
|------|------|
| SDP | 会话描述（编解码器、媒体类型） |
| ICE | 打洞穿透 NAT，收集候选地址 |
| STUN | 获取公网 IP |
| TURN | 中继（对称 NAT 无法直连时） |

前端职责：信令交换（通过 WebSocket）、`RTCPeerConnection` 管理、媒体流 `getUserMedia`。

## 五、消息可靠性

| 问题 | 方案 |
|------|------|
| 消息丢失 | ACK 确认 + 重发 |
| 消息乱序 | 服务端 seq + 客户端排序缓冲 |
| 重复消息 | 幂等 ID 去重 |
| 离线消息 | 上线后拉取增量（lastSeq） |

## 面试高频问答（追问链）

> 大厂对一个考点通常追问到能力边界：**概念 → 机制 → 边界 → ⭐ 原理（触底）→ 实战（落地）**。实战层是区分「背过」和「做过」的关键。

### 链一：WebSocket 连接管理

1. **概念**：WS 和长轮询区别？→ WS 全双工持久连接低开销；长轮询反复建连延迟高。
2. **机制**：心跳为什么必要？→ 检测死连接(NAT 超时/代理断开)，服务端清理僵尸连接。
3. **边界**：断线怎么重连？→ 指数退避重连 + 重连后状态补偿(拉取断连期间消息)。
4. ⭐ **原理（触底）**：百万长连接的 IM，前端和架构要注意什么？→ 前端连接复用/心跳节流/弱网降级；架构侧连接层水平扩展 + 网关路由 + 消息有序(seq)/不丢(ack)/不重(去重)，前端配合乱序缓冲。
5. **实战（落地）**：IM 长连接断线重连你怎么落地？→ 场景：IM 弱网频繁断连、消息丢失投诉；步骤：封装 ReconnectingWebSocket(指数退避+jitter) + 30s 心跳 + 重连后 lastSeq 增量拉取 + 服务端 seq 排序缓冲 + 幂等 ID 去重；验证：Chrome DevTools 模拟 offline/online 确认消息不丢不重；结果：断连恢复<3s、消息到达率 99.9%+，用户无感补全断连期间消息。

### 链二：SSE 与 WebRTC

1. **概念**：SSE 断线重连？→ 浏览器自动重连带 Last-Event-ID 续传。
2. **机制**：WebRTC 为什么需要信令服务器？→ P2P 前需交换 SDP 和 ICE candidate，信令中转(常用 WS)。
3. **边界**：WebRTC 连不通怎么办？→ NAT 穿透失败时经 TURN 中继兜底。
4. ⭐ **原理（触底）**：三种实时方案怎么按场景选？→ 服务端单向推送/AI 流式用 SSE；双向高频(IM/协同)用 WS；低延迟音视频/P2P 用 WebRTC，必要时组合(WS 做信令 + WebRTC 传流)。
5. **实战（落地）**：三种实时方案按场景你怎么组合落地？→ 场景：在线教育(聊天+直播+连麦)；步骤：IM 用 WS 双向(聊天/信令)、课程通知/AI 助教回复用 SSE 流式、音视频连麦用 WebRTC(WS 做 SDP/ICE 信令)+TURN 兜底；验证：弱网降级(WebRTC→仅音频→文字)、各通道独立重连互不影响；结果：聊天延迟<200ms、AI 流式首字<500ms、连麦 P2P 直连率 85% TURN 兜底 15%。

## 小结

- IM/协同用 WebSocket；推送/AI 流式用 SSE；音视频用 WebRTC
- 长连接必备：心跳、指数退避重连、鉴权、消息有序
- WebRTC 核心：SDP 协商 + ICE 打洞 + STUN/TURN

## 延伸阅读

- [WebSocket API - MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)
- [WebRTC 入门](https://webrtc.org/getting-started/overview)
