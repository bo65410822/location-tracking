# 知识库(docs/knowledge)

面向"主力开发学全栈"的知识沉淀区。与决策文档分工:
- `architecture-design.md` / `decisions/`:记录**我们定了什么**(精炼结论)。
- 本目录:记录**概念是什么、为什么、各方案优缺点对比**(学习向,展开讲)。

## 知识卡片模板

每张卡片含:1 要解决什么问题 · 2 核心概念 · 3 可选方案对比(bullet 列优缺点) · 4 本项目的选择与理由 · 5 已知约束/风险/延伸 · 关联决策。

## 已成文卡片

- [001 WebSocket 通信选型(原生 vs STOMP、JSON vs 二进制)](001-websocket-communication-choices.md)
- [002 身份建模(设备 vs 主体,以及用中性标识解耦)](002-identity-modeling.md)

## 待沉淀清单(backlog)

讨论中出现、尚未成文的知识点,按需展开:

- [ ] 自研 TCP 长连接 vs WebSocket:后台保活/重连/弱网控制到底谁做得了、区别在哪
- [ ] JDBC 规范 vs 数据库驱动 vs H2:三者关系与分层
- [ ] Spring Data JPA vs MyBatis:ORM 与 SQL 映射之争、国内生态与适用场景
- [ ] Spring Boot / Gradle 版本兼容坑:Convention API、VersionNumber 为何导致构建失败,如何选稳定版
- [ ] H2 内存库(mem)特性与 h2-console 用法:为什么重启数据就没了
- [ ] WebSocket 单机内存广播的集群约束:多实例为何广播不到,如何用 Redis Pub/Sub / MQ 解决
- [ ] WebSocketSession 非线程安全:并发向同一会话发送为何出错,ConcurrentWebSocketSessionDecorator 怎么解决

> 规则:讨论中冒出用户不熟的概念 → 当场一句话点出 + 登记到此 backlog → 适时成文。
