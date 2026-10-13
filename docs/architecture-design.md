# location-tracking 架构设计

> 本文件记录"为什么这么设计"的可复用结论,与 PRD(做什么)分开。
> 决策格式:状态 · 决定 · 理由 · 约束/影响 · 日期。

## 模块概述

实时定位追踪系统,三模块:
- `android-app`:GPS 采集 + 长连接上报(集成 LC AAR + lc-protocol)
- `server`:Spring Boot,收 lc-protocol 帧、存储、WebSocket 推送
- `web`:Vue 实时看板

协议库 `lc-protocol`(纯 JVM)由 Tracelet 抽出,App 与 server 共用。

---

## 决策记录

### 001 协议层抽成独立纯 JVM 库(lc-protocol)
- 状态:已实现
- 决定:把帧协议(FrameEncoder/FrameDecoder/FrameProtocol/LcFrameType + ParsedFrame/DecodeOutcome/ParseResult)从 tracelet-lc 抽成独立 `kotlin("jvm")` 库,App 与 server 共同依赖。
- 理由:协议是 App 与服务端唯一必须完全一致的部分。抽成共享库后"只写一遍、两端零偏差";若各写一份,帧格式/CRC/字节序极易产生难查的解析分歧。库为纯 JVM(无 android.*),故 Spring Boot(Java/Kotlin)可直接依赖。
- 约束/影响:协议类须 public(已从 internal 放开);JVM target 须与各端一致(当前 11);Android 侧经 LC(AAR)间接使用,新 server 必须**直接**依赖 lc-protocol(implementation 不传递导出)。
- 日期:2026-10-12

### 002 传输分两段:App↔Server 用 LC 长连接,Server↔Web 用 WebSocket
- 状态:已确认
- 决定:App 到服务端走 LC 自研 TCP 长连接(lc-protocol 二进制帧);服务端到网页走 WebSocket。协议层只覆盖 App↔Server 一段。
- 理由:App 侧要的是省电、实时、弱网可控,自研长连接 + 心跳 + 重连最合适,且复用既有能力;Web 端用二进制自研协议成本高、兼容差,WebSocket 原生支持、工作量小。两段解耦,各取所长。
- 约束/影响:服务端需实现 lc-protocol 的服务端解帧(复用同一库);Web 通道与协议层互不耦合,后续 Web 换 SSE/轮询不影响 App 链路。
- 日期:2026-10-12

### 003 后端选 Spring Boot(Java)
- 状态:已确认
- 决定:服务端用 Spring Boot,语言 Java。
- 理由:作者 Java 基础强(Android 出身),学的是框架而非新语言,落地最快;Java 可直接依赖 Kotlin/JVM 的 lc-protocol;Java 后端是国内接单/求职最被认可的组合。
- 约束/影响:切片1 只用 Spring Web + JPA + 一个数据库;微服务/Spring Cloud/MQ 等一律后置,避免生态兔子洞。
- 日期:2026-10-12

### 004 "承认一定会断、靠秒级重连兜底"的后台链路策略
- 状态:已确认
- 决定:不追求"后台永不断线",而是前台服务降低被杀概率 + 心跳假死检测 + 退避重连做到"断了用户无感"。
- 理由:Android(尤其国产 ROM)后台保活没有银弹,追求永不断是撞墙;工程上可控的是"快速恢复 + 数据不丢"。
- 约束/影响:切片1 验收须包含"飞行模式开关后自动恢复、数据不丢";重连需退避 + 抖动,避免狂连触发问题与耗电。
- 日期:2026-10-12

### 005 切片式交付,严格后置地图/多端/鉴权
- 状态:已确认
- 决定:切片1 只做"App→长连接→后端→WebSocket→Web 实时列表"闭环;地图、坐标纠偏(WGS84→GCJ-02)、iOS、鸿蒙、鉴权、集群全部后置。
- 理由:最小闭环能最快验证技术主线并产出可展示作品;范围膨胀是独立开发最大的空转风险。
- 约束/影响:切片1 Web 只展示坐标列表,不接地图;每切片完成即对外展示(作品/接单素材)。
- 日期:2026-10-12

### 006 三端统一 WebSocket + JSON,取代两段式/二进制(见 ADR 002)
- 状态:已确认
- 决定:App、Server、Web 三端统一用 WebSocket 传输,消息体统一 JSON;不再用自研 TCP + lc-protocol 二进制帧作为 App↔Server 线路协议。
- 理由:三端统一降复杂度、心智一致;Web 原生 JSON 友好;二进制在本场景带宽/省电收益不显著——省电瓶颈在 GPS 采集与射频唤醒,靠批量上报 + 自适应频率解决,而非压字节。
- 约束/影响:取代本文件 #决策002 与 ADR 001,并使 #决策001 的二进制共享库不再作线路协议;WebSocket 需应用层封装心跳/重连/超时/会话恢复。详见 `decisions/002-three-end-websocket-json.md`。
- 冲突:与本文件 #决策001、#决策002 及 decisions/001 冲突——按"只新增不改原文"原则,旧条目原文保留,有效性以 ADR 002 的取代关系为准。
- 日期:2026-10-12

### 007 数据访问层切片1 用 Spring Data JPA,后续可替换 MyBatis-Plus
- 状态:已确认
- 决定:切片1 数据访问层用 Spring Data JPA(底层 Hibernate);后续按需(复杂查询或国内求职对口)引入或替换为 MyBatis-Plus。
- 理由:定位点存取简单,`JpaRepository` 开箱即用、免写 SQL,最快跑通切片1 闭环(呼应 #决策005)。MyBatis-Plus 国内招聘更对口但非当前必需,留作后续;两者可在同一 Spring Boot 项目共存,替换成本可控。
- 约束/影响:实体用 `@Entity` + `JpaRepository`;切片1 配 H2 内存库,后续换 MySQL;若引入 MyBatis 需评估与 JPA 的事务/实体复用边界。
- 日期:2026-10-12

### 008 服务端技术栈锁定稳定版:Spring Boot 3.4.1 + Gradle 8.11.1 + Java 17
- 状态:已确认
- 决定:server 用 Spring Boot 3.4.1、Gradle 8.11.1、Java 17;打包用默认 jar(内嵌 Tomcat),不用 war。
- 理由:Initializr 默认生成的 Spring Boot 4.1.1 + Gradle 9.7.1 太前沿,实测连续构建失败——Gradle 9 移除了 `VersionNumber`(撞 IDE Android 工具插件)和 `Convention` API(撞 `io.spring.dependency-management`),且 4.x 教程稀少、求职对口度低。降到稳定组合可直接构建通过,文档海量,招聘对口(呼应 #决策005 避免生态兔子洞)。
- 约束/影响:Android app 与 server 用各自 IDE(IDEA 需禁用 Android 插件以免注入 Android 工具链);升级 Spring Boot/Gradle 前先确认插件兼容性;jar 打包用 `java -jar` 启动。
- 日期:2026-10-12

### 009 切片1 WebSocket 接口设计:双端点 + JSON + 单机内存广播
- 状态:已确认
- 决定:切片1 服务端用原生 WebSocket 开两个端点——`/ws/app`(App 上报)、`/ws/web`(Web 广播);消息 JSON(字段见设计文档);广播用单机内存会话集合扇出。
- 理由:端点按角色分离,鉴权/限流可分别施策;JSON 字段取最小集(deviceId/lat/lng/speed/bearing/accuracy/timestamp),userId/altitude/battery 后置以保持最小闭环(#决策005)。
- 约束/影响:单机内存广播不支持多实例(集群需 Redis Pub/Sub/MQ);`WebSocketSession` 非线程安全,并发发送需 `ConcurrentWebSocketSessionDecorator` 或加锁;两端点切片1 无鉴权(已知安全缺口,上线前必补)。详见 `websocket-interface-design.md`。
- 日期:2026-10-12

### 010 身份用语义中性的 sourceId 解耦,传输层不解释语义
- 状态:已确认
- 决定:定位点的身份标识用中性字段 `sourceId`(不透明字符串),服务端不解释其语义;填什么(设备/用户/车辆号)由上层接入方决定,Web 按相同 `sourceId` 归并轨迹。
- 理由:把"身份语义"从传输/存储层解耦——协议与表结构稳定,语义升级(设备→主体)无需改协议;关注点分离,服务端只负责按 sourceId 分流+存+广播。规避了"设备级标识不稳定(换手机/重装)"直接绑死协议的问题(用户提出)。
- 约束/影响:服务端不校验 sourceId 唯一性/稳定性,责任在上层;若需同时表达"设备 vs 主体"需另加字段(JSON 向后兼容);真正跨设备稳定的主体身份留待鉴权切片引入账号/设备绑定。详见知识卡片 `knowledge/002`。
- 日期:2026-10-12
