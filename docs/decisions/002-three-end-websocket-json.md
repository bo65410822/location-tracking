# ADR 002:三端统一 WebSocket + JSON 协议(取代 ADR 001)

- 状态:已确认
- 日期:2026-10-12
- 取代:ADR 001(decisions/001-protocol-sharing-and-transport.md);architecture-design.md#决策002,并使 #决策001 在传输层面失效

## 背景

原设计(ADR 001)采用两段式传输:App↔Server 用 LC 自研 TCP 长连接(二进制帧 + lc-protocol 共享库),Server↔Web 用 WebSocket。讨论后决定重新评估 App↔Server 这段的传输与协议格式,核心诉求是三端链路统一、降低整体复杂度。

## 决策

三端(App、Server、Web)统一使用 **WebSocket** 作为传输层,消息体统一用 **JSON 文本协议**。不再使用自研 TCP 长连接与 lc-protocol 二进制帧作为 App↔Server 的线路协议。

## 理由

- **统一传输栈**:一套 WebSocket,服务端不必为 App 单独实现二进制解帧;鉴权、监控、调试只需一套心智,没有两段式割裂。
- **Web 一等公民**:浏览器原生 `JSON.parse`;二进制帧在 JS 侧需手写 DataView 解析,痛苦且易错。三端统一就该选对三端都友好的格式。
- **可调试、可演进**:JSON 报文肉眼可读,排障快;加字段向后兼容,无需管字节序/帧版本。
- **二进制收益在本场景不显著**:单个定位点 JSON ≈ 150–250B、二进制 ≈ 30–50B;秒级上报下差值在现代移动网络基本无感;WebSocket `permessage-deflate` 压缩会进一步拉近差距。
- **耗电瓶颈不在协议字节数**:GPS/GNSS 持续采集与射频唤醒(RRC 状态迁移)才是耗电大头,远超传输字节差异。省电应靠"批量上报 + 自适应采集频率",而非二进制压字节。

## 影响与约束

- lc-protocol 二进制帧库不再作为线路协议(ADR 001 的决策基础在传输层面失效);若需复用其业务数据模型可另议,但不作为传输格式。
- WebSocket 需在应用层封装:心跳(ping/pong 假死检测)、退避重连(含抖动)、请求超时、断线会话恢复(带 lastMessageId 补发断线期间消息)。
- 省电策略:批量上报(攒几条/几秒一发,减少射频唤醒次数)、静止降频 / 移动升频、心跳间隔够维持连接即可。
- 后台保活策略(architecture-design.md#决策004:承认会断、靠秒级重连兜底)不变,仍适用。

## 冲突与解决

- 冲突:与 `decisions/001-protocol-sharing-and-transport.md` 冲突——该 ADR 选 App↔Server 自研 TCP + 二进制帧 + 两段式传输。
- 冲突:与 `architecture-design.md#决策002`(两段式传输)冲突;并使 `#决策001`(lc-protocol 共享库作为线路协议)在传输层面失效。
- 解决方案:采纳本 ADR(三端统一 WebSocket + JSON)。理由:三端统一降低整体复杂度与维护成本;Web 需 JSON 友好;二进制在本场景的带宽/省电收益经评估不显著(省电瓶颈在采集而非传输)。按"只新增不改原文"的文档卫生规则,**不改写 ADR 001 与 architecture-design.md 既有条目原文**,其有效性以本 ADR 的取代关系为准。

## 备选与放弃原因

- **WS + 自研二进制帧(binary frame)**:传输统一且保留紧凑二进制、复用 lc-protocol,但 Web 侧解析成本高、维护负担重,且本场景收益不显著——放弃。
- **维持 ADR 001 两段式**:复用既有自研实现,但三端心智割裂、服务端需维护解帧、Web 不友好——放弃。
