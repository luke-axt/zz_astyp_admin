# 公网服务架构选型建议

## 一、选型原则

基于当前团队规模、技术储备及业务需求，公网服务选型遵循以下原则：

1. **团队匹配优先**：3 名初级数据开发均以 Python 为主力语言，1 名中级 Java 开发需兼顾内网若依系统。公网 7×24 服务的运维主力应当是人数更多、更熟悉对应技术栈的 Python 团队。
2. **学习成本极低**：摒弃需要额外学习复杂配置、新概念模型或新运维体系的组件。
3. **轻量化**：单台或两台低配比云服务器即可承载全部公网服务，降低硬件与运维成本。
4. **平滑兼容**：与现有 FlaskAPI 服务渐进式融合，而非推倒重来。

---

## 二、推荐架构：LNRP + Docker

采用 **Nginx + FastAPI + Redis + RQ + PostgreSQL**，以 **Docker Compose** 统一编排部署。

### 2.1 总体拓扑

```
公网流量（企业微信 / 独立站 / eBay / OC Token）
           │
           ▼
    ┌──────────────┐
    │    Nginx     │  ← 反向代理、SSL 终结、域名代理
    └──────────────┘
           │
     ┌─────┴─────┬────────────────────┐
     ▼           ▼                    ▼
┌─────────┐ ┌──────────┐  ┌──────────────────────┐
│ FastAPI │ │ FastAPI  │  │   Vue3 + Element     │
│ API网关  │ │ 业务服务  │  │   Plus (客服后台)     │
│ (通用)   │ │(eBay等)  │  │   Nginx 托管 SPA     │
└────┬────┘ └────┬─────┘  └──────────┬───────────┘
     │           │                   │
     │           │ WebSocket / SSE   │
     └─────┬─────┘◄──────────────────┘
           ▼
    ┌──────────────┐
    │    Redis     │  ← 消息队列 + 缓存 + 状态存储
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  RQ Worker   │  ← 异步任务消费（eBay 客服消息处理等）
    └──────────────┘
           │
           ▼
    ┌──────────────┐
    │  PostgreSQL  │  ← 数据持久化（内置多租户扩展能力）
    └──────────────┘
```

### 2.2 核心组件选型说明

| 组件 | 选型 | 角色 | 推荐理由 |
|------|------|------|----------|
| **Web / 代理** | **Nginx** | 统一流量入口 | 成熟稳定，配置语法简单，直接解决「公网域名代理」和 SSL 证书管理问题。可将不同域名/路径路由到不同后端，天然适合多模块共存。 |
| **API 服务** | **FastAPI** (Python) | 业务 API 与网关 | 与现有 FlaskAPI 语法接近，迁移成本极低；原生异步（ASGI），高并发下比 Flask 更省资源；自动生成 Swagger 文档，前后端联调方便。 |
| **管理后台** | **Vue3 + Element Plus** | 客服工作台等 Web 应用 | 由独立站前端开发负责，组件丰富（表格、对话框、实时消息列表）、中文文档成熟，打包后为纯静态文件，由 Nginx 直接托管。 |
| **消息队列** | **Redis + RQ** | 异步任务队列 | RQ 是 Python 世界最轻量的队列方案，纯 Redis 实现，API 极度简洁（比 Celery 少 80% 的配置概念）。eBay 智能客服的异步消息收发、耗时任务均可基于 RQ 实现。 |
| **实时通信** | **WebSocket (FastAPI)** | 服务端推送 | FastAPI 原生支持 WebSocket，客服后台可实时接收新消息通知、会话状态变更，避免前端轮询浪费资源。若仅需服务端单向推送，SSE 是更简单的备选。 |
| **缓存 / 状态** | **Redis** | 缓存、Token 暂存 | 一物多用：既作 RQ 的 Broker，也作 WebSocket 连接状态、高频 Token、会话上下文的缓存，减少数据库读写压力。 |
| **数据库** | **PostgreSQL** | 持久化 | 日均 3500 笔订单 + 多客服并发，SQLite 写串行化已成为瓶颈。PostgreSQL 行级锁、JSONB 字段、全文搜索等特性与 Python 生态匹配度高，且为 SaaS 多租户预留了扩展空间。 |
| **进程守护** | **Docker Compose** | 7×24 进程管理 | 将 Nginx、FastAPI、RQ Worker、PostgreSQL、Redis 全部容器化编排，实现「配置即代码」。用 Docker 的 `restart: always` 做进程守护，环境一致性远优于裸机部署，且为横向扩展预留能力。 |

### 2.3 需求映射

| PRD 需求 | 实现方式 |
|----------|----------|
| **刷新 OC 系统 Token** | 现有 FlaskAPI 逻辑迁移至 FastAPI（或保留 Flask，由 Nginx 统一代理），Redis 缓存 Token 避免频繁刷新。 |
| **独立站后端接口转发** | Nginx 作为反向代理将独立站域名请求转发至 FastAPI 服务；FastAPI 再对内做鉴权、聚合、协议转换。 |
| **公网域名代理（企业微信对接）** | Nginx 监听企业微信回调域名，路由到 FastAPI 对应接口，解决内网无公网 IP 的回调问题。 |
| **eBay 智能客服** | **前端**：Vue3 + Element Plus 搭建客服工作台（消息列表、对话窗口、客户资料、人工转接、快捷回复等）。<br>**后端**：FastAPI 提供客服 Webhook 接收接口 + RESTful API（会话管理、人工回复、机器人策略配置）；新消息与状态变更通过 **WebSocket** 实时推送至客服后台。<br>**异步**：eBay API 调用、消息解析、自动回复策略等耗时逻辑放入 **RQ Worker** 执行。<br>**存储**：Redis 暂存会话上下文；PostgreSQL 记录聊天记录与客服操作日志。 |
| **定时任务调度** | 公网 7×24 的定时需求（如 Token 刷新兜底）继续使用 APScheduler，以独立容器运行在 Docker Compose 中，降低团队学习成本。 |

---

## 三、为什么不选其他方案

### 3.1 不选 Java / Spring Cloud 作为公网主栈

- **人力风险**：仅 1 名中级 Java 开发，且需维护内网若依系统。公网 7×24 服务若采用 Java，一旦该同事请假或离职，Python 团队难以快速接手。
- **重量问题**：Spring Cloud 体系组件多（Gateway、Nacos、Sentinel 等），对于当前仅 4 个公网模块的场景属于过度设计，且学习曲线陡峭。

### 3.2 不选 Celery 作为消息队列

- RQ 与 Celery 相比，省去了 Broker、Backend、Beat、Canvas 等复杂概念。对于「团队很小、只有初级开发」的现状，Celery 的调试和配置成本会成为负担。RQ 的 `queue.enqueue(func, args)` 足够直观。

### 3.3 不选 Kubernetes / Docker Swarm

- 当前阶段仅需要在一到两台云服务器上运行不到 10 个容器，引入 K8s 的运维复杂度远大于收益。Docker Compose 在单机上已实现环境一致性和进程守护，问题定位更直接。

### 3.4 不选若依（内网 Java 栈）做公网客服后台

- 内网环境没有 UPS，无法做到真正的 7×24 稳定运行；公网客服需要随时在线处理买家消息，必须独立部署。
- 若依系 Java 栈只有 1 名中级开发能深度维护，公网服务不能对内网人员形成强依赖。

### 3.5 不选 Jinja2 模板引擎或纯静态页面

- eBay 客服存在大量动态交互（实时消息、人工转接、表单提交、状态管理），传统服务端模板渲染开发效率低、用户体验差，无法满足「人工介入」类复杂业务场景。
- 前后端分离（Vue3 + FastAPI）让前端专注交互、后端专注 API，职责清晰，且 Element Plus 提供了现成后台组件，开发速度反而更快。

### 3.6 不选 RabbitMQ / Kafka

- RQ 基于 Redis 实现队列，团队无需维护额外中间件。Redis 本身已是缓存刚需，一物两用减少组件数量。

### 3.7 不选 SQLite 作为生产数据库

- 日均 3500 笔订单 + 多客服并发写入，SQLite 的串行写锁会导致频繁阻塞。继续用 SQLite 需要在应用层自行实现写队列、重试机制、定时归档，复杂度反而高于直接使用 PostgreSQL。
- 7×24 运行下，SQLite 单文件损坏风险高，且无热备恢复手段。

### 3.8 不选 Django 替代 FastAPI

- Django 的 Admin、ORM、认证体系对 SaaS 有吸引力，但其「大而全」的世界观与现有 Flask 经验差异大，3 名初级开发切换成本高。
- Django 的异步与 WebSocket 支持（Channels）配置复杂，不如 FastAPI 原生简洁。
- 已确定 Vue3 前后端分离，Django 最大的模板/Admin 优势被削弱。FastAPI + `fastapi-users` + `Alembic` 能以更低成本补齐同等能力。

---

## 四、部署与硬件建议

### 4.1 服务器规划

建议以 **1 台低配比 Linux 云服务器**（如 2核 4G，CentOS/Ubuntu）作为公网统一服务节点，以 Docker Compose 运行以下容器：

- Nginx（含 Vue3 前端静态资源托管）
- FastAPI (Uvicorn)（含 WebSocket 服务）
- PostgreSQL
- Redis
- RQ Worker（可启动 2-4 个容器实例）
- APScheduler 定时任务容器

若追求高可用，可再加 1 台同规格服务器做 Nginx + FastAPI 双节点，前置阿里云/腾讯云 CLB（负载均衡）即可，无需自建复杂集群。

### 4.2 关键配置提示

1. **PostgreSQL 连接池**：FastAPI 使用 `asyncpg` + SQLAlchemy 异步连接池，建议池大小设置为 `min: 5, max: 20`，避免高并发下连接耗尽。
2. **Uvicorn 进程数**：单台服务器建议 `--workers 2` 即可，配合 Nginx 的负载均衡足够支撑当前业务。
3. **前端部署**：Vue3 项目 `npm run build` 后生成纯静态文件，直接放在 Nginx 的 `html/` 目录下；Nginx 对前端路由配置 `try_files` 回退到 `index.html`，支持 Vue Router 的 History 模式。
4. **WebSocket 代理**：Nginx 需开启 `proxy_http_version 1.1` 与 `proxy_set_header Upgrade $http_upgrade`，确保客服后台的实时消息通道正常穿透。
5. **Docker Compose 分服务管理**：建议将 Nginx、FastAPI、RQ Worker、PostgreSQL、Redis、APScheduler 分为独立 Service，任一容器重启不影响其他服务。
6. **Nginx 配置拆分**：每个域名/模块使用独立的 `conf.d/` 配置文件，方便后续独立维护。
7. **数据持久化**：PostgreSQL 和 Redis 的数据目录必须挂载到宿主机 Volume，避免容器重启后数据丢失。

---

## 五、实施路线建议（渐进式）

避免一次性重写，建议分阶段落地：

| 阶段 | 目标 | 周期建议 |
|------|------|----------|
| **Phase 1：搭建基座** | 云服务器安装 Docker + Docker Compose；部署 Nginx + Redis + PostgreSQL；将现有 FlaskAPI（Token 刷新）接入 Nginx 反向代理，验证 7×24 稳定运行。 | 1-2 天 |
| **Phase 2：服务统一** | 新模块（独立站接口转发、企业微信代理）统一使用 FastAPI 开发；旧 FlaskAPI 按需迁移或共存。 | 1-2 周 |
| **Phase 3：客服模块开发** | 前端基于 Vue3 + Element Plus 搭建客服工作台；后端 FastAPI 提供 RESTful API + WebSocket 推送；引入 RQ + Redis 处理 eBay 消息收发与机器人策略。APScheduler 定时任务同步迁入 Docker Compose 管理。 | 3-5 周 |
| **Phase 4：可观测性补齐** | 引入 Prometheus + Grafana 监控 API 与队列；配置企业微信告警；数据定时备份到云存储。 | 1-2 周 |

---

## 六、SaaS 化演进预留

若未来计划将 eBay 智能客服等能力开放为 SaaS 对外收费，当前架构已预留扩展空间，核心只需在应用层补齐以下能力：

### 6.1 多租户隔离

- **方案**：在现有 PostgreSQL 单库中增加 `tenant_id` 字段，通过 FastAPI 中间件自动拦截并追加租户过滤条件。这是成本最低的多租户方案，客户量达百级后再考虑分表或 Schema 隔离。
- **敏感数据**：eBay API Token、店铺配置等必须 AES-256 加密存储，防止租户间数据泄露。

### 6.2 订阅与配额控制

- 限制每个商家的客服坐席数、消息条数、存储时长。初期可用 Redis 计数器实现，避免引入复杂计费系统。

### 6.3 横向扩展路径

| 阶段 | 客户规模 | 关键动作 |
|------|----------|----------|
| **内部使用期** | 1 家公司 | 当前架构完全够用。 |
| **MVP 验证期** | 1-5 家（免费内测） | 应用层加 `tenant_id`；引入 RQ Dashboard；API 增加限流（Rate Limiting）。 |
| **早期收费期** | 5-50 家 | 补齐监控告警；数据定时备份到云存储。 |
| **增长期** | 50-500 家 | PostgreSQL 读写分离；Redis 主从；FastAPI 多节点 + Nginx 负载均衡；按需引入 Celery 替代 RQ。 |

---

## 七、总结

对于「小团队、Python 为主、成本敏感、7×24 在线」的公网服务场景，**Nginx + FastAPI + Redis + RQ + PostgreSQL + Docker Compose** 是当前最优解。

- **数据库**：放弃 SQLite，直接以 PostgreSQL 为基线，支撑日均 3500 笔订单的并发写入，并为 SaaS 多租户预留空间。
- **部署**：以 Docker Compose 替代裸机 Supervisor，实现环境一致性与 7×24 容器化运维。
- **前后端**：Vue3 + Element Plus 搭建客服后台，FastAPI + WebSocket 实现实时消息推送，职责清晰。
- **扩展**：现有技术栈在 50 家租户以内完全够用，可边盈利边迭代，无需过早引入 K8s 或 Celery。
