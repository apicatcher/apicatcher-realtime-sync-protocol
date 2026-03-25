# ApiCatcher 实时同步协议规范 (Real-time Sync Protocol)

[English](./README.md)

**版本:** 1.0.0-Draft
**状态:** 草案 (Draft)
**协议:** MIT License

## 1. 摘要 (Abstract)
本规范定义了 ApiCatcher 与接收端（如 PC 端的桌面应用或自定义 Server）之间进行 HTTP/HTTPS 实时抓包数据无缝同步的标准通信协议。协议基于全双工的 WebSocket 连接，采用 **分片流式 (Streaming) JSON 帧** 来推送抓包数据，旨在为开发者提供一个低延迟、极低内存占用、不丢包的开放同步生态。

## 2. 动机 (Motivation)
在实际的研发与测试场景中，仅依靠移动端内嵌的抓包界面往往无法满足复杂的调试需求。本协议致力于将 ApiCatcher 捕获的底层网络流量开放出来，以支持以下典型的扩展场景：

1. **深度可视化分析**：将移动端捕获的流量实时导入到 PC 端专业的网络调试工具中，借助桌面端的强大算力与丰富界面进行更深度的报文解析和场景重放。
2. **DevOps 与质量监控集成**：作为一个无侵入式的移动端网络探针，ApiCatcher 产生的实时数据流可以标准化地接入企业内部的 DevOps 平台或自动化测试管道，为 API 兼容性监控、安全审计等环节提供数据。

### 为什么不使用 HAR 规范？
在设计之初，我们曾考虑采用业界通用的 [HTTP Archive (HAR) 1.2 规范](http://www.softwareishard.com/blog/har-12-spec/)。但在移动端上，一旦抓包遇到带有极其庞大 Response Body（几 MB 或几十 MB）的请求，如果严格按照 HAR 的做法，必须把完整的 Body 读取、解析并组装在一个巨大无比的 JSON 对象中再一次性发出。这种机制在 iOS 系统下会直接撑爆极为严苛的网络扩展内存上限，导致 **VPN 抓包进程发生严重的 OOM 崩溃**。
因此，本协议自下至上拥抱了**流式分段 (Streaming Chunk)** 方案，牺牲了一定的免转换直读便利性，换取了 VPN 进程发送时极致的安全内存占用。

---

## 3. 通信规范 (Specification)

### 3.1 连接配置
- **角色约束**：ApiCatcher 客户端 (App) 充当 **WebSocket Client**；数据接收端充当 **WebSocket Server**。
- **传输层**：仅支持通过局域网内的 **`ws://`** 协议进行明文传输通讯。
- **连接重连**：当网络波动导致连接中断时，Client 将主动尝试静默重连。断网期间拦截的数据流（以及传输到一半的无效分片）作废，不再提供补偿重发。

### 3.2 帧格式与流式事件 (Streaming Events)
为了解决内存瓶颈，同一条完整的抓包记录将被横向拆解成数个独立触发的 JSON Text Frame 投递。
所有的通信报文外层必须包含 `type`、`requestId` 以及 `event` 这三个核心参数。由于 WebSocket 单极连接上会有并发的多个请求交替穿插，接收端必须严格依赖 `requestId` 将属于同一次请求的分片在本地进行状态组装。

基础的 JSON 消息壳格式如下：
```json
{
  "type": "http",
  "requestId": "<唯一标识符，如UUID>",
  "event": "<流式事件生命周期节点>",
  "timestamp": 1711268370123,
  "payload": { ... }
}
```

### 3.3 具体的流式事件列表

#### 3.3.1 `req_start`（请求发起）
当 App 拦截到出站 API 请求时第一时间触发，只包含最轻量的元数据。
```json
{
  "type": "http",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "event": "req_start",
  "timestamp": 1711268370123,
  "payload": {
    "url": "https://api.example.com/data",
    "method": "POST",
    "httpVersion": "HTTP/1.1",
    "headers": [{"name":"User-Agent","value":"ApiCatcher/1.0"}]
  }
}
```

#### 3.3.2 `req_body` 与 `res_body`（分片数据传输）
无论是 Request Body 还是 Response Body，抓包探针在底层流式消费时，每次读取一定容量（比如 16KB 或 32KB）就会立即打包成独立的一个 Frame 投递给接收端。
**(注：对于报文体较大的请求，接收端会按序收到多条具有相同 requestId 的此类型包。ApiCatcher 保证分片会按顺序发送，由于 WebSocket 基于 TCP 的有序性，接收端收到的顺序即为发送顺序，只需要按接收顺序直接进行 Base64 解码并合并即可。)**
```json
{
  "type": "http",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "event": "res_body",
  "timestamp": 1711268370123,
  "payload": {
    "data": "eyBzdWNjZXNz... (二进制被转为Base64字符串的碎片)"
  }
}
```

#### 3.3.3 `res_start`（响应头到达）
当服务端响应该请求并回传了 Header 时触发该起始事件。
```json
{
  "type": "http",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "event": "res_start",
  "timestamp": 1711268370123,
  "payload": {
    "status": 200,
    "httpVersion": "HTTP/1.1",
    "headers": [{"name":"Content-Type","value":"application/json"}]
  }
}
```

#### 3.3.4 `req_end`（请求完整闭环）
标识该请求的抓包流程终于彻底完结。（可能因成功响应完成，也可能因超时错误等中途终止而完成）。
```json
{
  "type": "http",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "event": "req_end",
  "timestamp": 1711268370123,
  "payload": {
    "error": null, // 若无错误则为 null，若有错误譬如 "Timeout" 或 "Connection Aborted"
    "timings": {
        "send": 0,
        "wait": 150,
        "receive": 5
    }
  }
}
```
*接收端在收到此事件或因故监测到 WebSocket 断开连接时，应当释放并完结对当前 `requestId` 的本地句柄或内存缓存。*

---

## 4. 异常与边界处理
**断网主动清理**：如果底层 WebSocket 连接发生非预期的抛错或彻底断开，PC 接收端程序必须主动废除并清理目前内存中还在组装阶段、且未收到 `req_end` 事件的失效碎片记录。

## 5. 许可与开源声明 (License)
这份实时同步协议规范受 **MIT License** 保护。任何开发者和机构均可免费、商业地修改或基于本规范实现兼容 ApiCatcher 的各类接收端与桌面分析软件，而不必承担额外的责任。
