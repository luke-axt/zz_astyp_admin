# astpy 项目架构全局分析

> 分析日期：2026-05-12  
> 分析范围：astpy 全项目（含旧架构代码与 asrc 新架构基座）  
> 核心结论：项目已进入"新旧架构并行"阶段，旧架构存在**基类碎片化、配置硬编码、数据库连接散落、目录结构不符合 Python 包规范**等结构性问题，建议按 asrc 新基座分域逐步迁移。

---

## 一、项目概述

astpy 是一个面向跨境电商企业的**内部数据整合与自动化平台**，核心定位可概括为：

- **实时数据采集**：通过 RPA（Selenium + 浏览器自动化）和 API 从外部系统（领星、通途、金蝶、万邑通、橙联等）采集业务数据。
- **离线数仓**：基于 MySQL 构建 odl → dml → adl → bal 四层数据仓库，以 SQL 跑批为主，做数据清洗和指标计算。
- **桌面客户端**：基于 PySide6 的 Windows 桌面应用，供业务人员下载报表、维护主数据。
- **定时任务调度**：基于 APScheduler 实现 200+ 个批处理作业的分布式调度和 T+1 事件依赖。
- **7×24 独立任务**：部分高可用任务部署在云服务器，使用 SQLite 与内网生产环境隔离（内网无 UPS）。

目前已对接 **8 个海外仓系统、2 个 ERP、1 个金蝶云星空**，并集成百度云、腾讯云、163 邮件、企业微信等基础服务能力。

---

## 二、技术栈

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| 语言 | Python 3.x | 全栈 Python，无其他服务端语言 |
| 桌面端 | PySide6 | 客户端界面与业务交互 |
| 浏览器自动化 | Selenium + ChromeDriver | RPA 核心能力 |
| 定时调度 | APScheduler | BackgroundScheduler，15 分钟心跳刷新作业清单 |
| 数据处理 | Pandas + SQL | DataFrame 转换与 SQL 跑批 |
| 数据库 | MySQL（主）/ SQLite（独立部署） | 内网 MySQL，云端 SQLite |
| 操作系统 | Windows Server / Windows 11 | 方便浏览器自动化和定时任务常驻 |
| Web 服务 | 若依（外部系统） | 固定 IP 绑定在若依应用服务上，提供 Web 交互和 API 网关 |

---

## 三、目录结构与核心业务逻辑

### 3.1 当前目录结构（旧架构）

```
astpy/
├── admin/service/          # 系统管理服务
│   ├── DBcore.py           # 数据库连接封装（所有模块共用）
│   ├── adminconfig.py      # 全局配置（含加密密码）
│   ├── AdminService.py     # 配置读取、系统参数查询
│   ├── EmailService.py     # 邮件告警
│   ├── MonitorService.py   # 监控服务
│   └── EventService.py     # 事件服务
├── client/                 # PySide6 桌面应用
│   ├── index.py            # 客户端入口
│   └── src/view/           # 各业务界面（供应链、财务、运营等）
├── common/                 # 通用实体类
│   ├── BaseObj.py          # AdminBase 基类（含邮件、数据库、节假日检查）
│   ├── ResultObj.py        # 统一返回结果对象
│   └── entity/             # 业务实体
├── CoreBussiness/          # 外部系统对接服务
│   ├── LingXingService.py  # 领星 ERP
│   ├── TongtoolService.py  # 通途
│   ├── OrangeConnexService.py # 橙联
│   ├── k3/                 # 金蝶 K3
│   ├── openapi/            # 若依/OpenAPI 网关
│   └── ...                 # 共 20+ 个外部服务类
├── dwd/                    # 离线数仓
│   ├── odl/                # 操作数据层（原始表 + SQL 文件）
│   ├── dml/                # 数据中间层（清洗、关联）
│   ├── adl/                # 应用数据层（面向主题）
│   └── bal/                # 平衡/校验层（指标计算、对账）
├── ETL/                    # 实时数据采集
│   ├── EtlAdmin.py         # ETL 作业基类
│   ├── EtlService.py       # ETL 业务服务
│   └── ods/                # 各系统 ODS 接入作业
├── rpa/                    # RPA 自动化
│   ├── amazon/             # 亚马逊 PDP 监控
│   ├── lingxing/           # 领星数据同步
│   ├── lastmile/           # 尾程物流测算
│   ├── kingdee/            # 金蝶单据同步
│   └── ...                 # 共 15+ 个业务域
├── rpt/                    # 报表模块
├── task/                   # 定时任务调度核心
│   ├── JobCore.py          # APScheduler 封装 + 作业分发
│   ├── start_task.py       # 调度器启动入口
│   └── modules/            # 各域任务分发器（TaskEtl、TaskRpa、TaskDwd...）
├── utils/                  # 工具类
│   ├── LogUtils.py         # 日志工具（按作业名分文件）
│   ├── BrowserUtils.py     # Selenium 浏览器封装
│   ├── ExcelUtil.py        # Excel 读写
│   └── dateutil.py         # 日期处理
├── zzlc/                   # 脚本、模板、部署工具、一次性脚本
├── 辅助工具/               # 各类手动运行脚本
└── asrc/                   # [新架构] 重构基座（当前仅含基类和空包）
    ├── core/
    │   ├── base/base_component.py   # 第一级统一基类
    │   ├── config/settings.py       # 新配置管理
    │   ├── database/
    │   ├── entity/
    │   ├── service/
    │   └── utils/
    ├── etl/
    ├── rpa/
    ├── client/
    └── tasks/
```

### 3.2 核心业务逻辑与数据流

```
┌─────────────────────────────────────────────────────────────┐
│                        外部业务系统                           │
│  领星  通途  金蝶  万邑通  橙联  亚马逊  eBay  九方  IML ...   │
└────────┬────────────────┬──────────────┬─────────────────────┘
         │ API / Selenium │              │
         ▼                ▼              ▼
┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
│ CoreBussiness/  │ │    rpa/     │ │  若依(OpenAPI)   │
│ 外部服务对接层   │ │ 浏览器自动化 │ │   API 网关层     │
└────────┬────────┘ └──────┬──────┘ └────────┬────────┘
         │                 │                 │
         └────────┬────────┘─────────────────┘
                  ▼
         ┌─────────────────┐
         │   ETL / ods/    │  ← 实时数据采集，写入 ODS
         │   dwd/odl/      │  ← 离线数仓贴源层
         └────────┬────────┘
                  ▼
         ┌─────────────────┐
         │   dwd/dml/      │  ← 数据清洗、关联、中间表
         │   dwd/adl/      │  ← 面向业务主题的数据集市
         │   dwd/bal/      │  ← 指标计算、对账、尾程测算
         └────────┬────────┘
                  ▼
         ┌─────────────────┐
         │   client/       │  ← PySide6 桌面应用（报表下载、主数据维护）
         │   rpt/          │  ← 通用报表导出
         └─────────────────┘
```

**定时任务调度逻辑**：
1. `start_task.py` 启动 `JobCore`。
2. `JobCore` 每 15 分钟从 `dc_job_info` 表读取本机作业清单。
3. 根据 `component` 字段（如 `TaskEtl:EtlService.j00002_getSuspOrder`）使用 `importlib` 动态反射调用对应类方法。
4. 执行前检查 `dc_job_rely` 依赖表，不满足则添加 `rely_redo` 作业（20 分钟后重试，最多 3 次）。
5. 执行成功向 `dc_job_event_log` 插入 `SUCC` 事件；失败则记录 `dc_job_error_log` 并邮件告警。

---

## 四、关键入口文件

| 入口文件 | 职责 | 调用链 |
|---------|------|--------|
| `task/start_task.py` | 定时任务调度器启动 | `JobCore().start_task()` |
| `task/JobCore.py` | APScheduler 封装、作业分发、依赖检查、重做机制、超时监控 | `dynamic_import_class_method_call()` → `task.modules.*` |
| `client/index.py` | 桌面应用启动 | `UpdateApp` → `QApplication` → `MainWindow` |
| `ETL/EtlAdmin.py` | ETL 作业基类 | `run()` → `action()` → 各子类实现 |
| `dwd/DwdAdmin.py` | 数仓作业基类 | `run()` → `action()`（带时间段限制） |
| `rpa/lastmile/task_lastmile.py` | 尾程独立任务入口 | 直接被 JobCore 或手动调用 |
| `admin/service/adminconfig.py` | 全局配置读取 | 被几乎所有模块静态引用 |
| `admin/service/DBcore.py` | 数据库连接 | 被所有需要读写数据库的类实例化 |

---

## 五、依赖关系全景

### 5.1 核心依赖链路

```
                        ┌─────────────────────┐
                        │  adminconfig.py     │
                        │  （全局配置中心）     │
                        └──────────┬──────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
   ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
   │   DBcore    │         │  AdminService│        │  LogUtils   │
   │ （数据库连接）│         │ （配置/密钥） │        │ （日志工具）  │
   └──────┬──────┘         └─────────────┘         └──────┬──────┘
          │                                               │
          │         ┌─────────────────────┐               │
          │         │    common/BaseObj   │               │
          │         │    (AdminBase)      │               │
          │         └──────────┬──────────┘               │
          │                    │                          │
          └────────────────────┼──────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ EtlAdmin │    │DwdAdminNew│   │ RpaXxxJob│
        │ EtlService│   │ BalXxx   │    │ ...      │
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
             └───────────────┼───────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  CoreBussiness/  │
                    │  外部系统 Service │
                    └─────────────────┘
```

### 5.2 关键依赖问题

- **配置中心单点**：`adminconfig.py` 被 90% 以上的模块直接 `from admin.service.adminconfig import AdminConfig`，任何配置结构调整都会引发全局改动。
- **数据库连接散落**：业务代码中随处可见 `DBcore(AdminConfig.get_db_info())` 或 `DBcore(AdminConfig.get_dwd_db_info())`，连接参数散落在数百个文件中。
- **外部服务强耦合**：`LingXingService`、`TongtoolService` 等直接引用 `AdminConfig` 和 `AdminService`，无法独立打包部署。
- **动态导入无约束**：`JobCore` 通过字符串拼接做 `importlib.import_module`，编译期无法检查类和方法是否存在。

---

## 六、潜在架构问题

### 6.1 设计层面问题

| 问题 | 现状 | 影响 |
|------|------|------|
| **缺乏统一继承体系** | `AdminBase`、`EtlAdmin`、`DwdAdminNew` 各自为政，各自初始化 `dbs`/`email`/`logger` | 代码重复严重，新作业需要复制粘贴模板；无法统一植入监控、熔断、重试逻辑 |
| **配置管理混乱** | 所有环境配置堆在一个 YAML/JSON 文件，密码以 Fernet 加密硬编码 | 无法做环境隔离（开发/测试/生产），CI/CD 和容器化困难；密钥轮换成本高 |
| **数据库方言切换困难** | `DBcore` 仅面向 MySQL，SQLite 独立部署时各模块需自己改写连接逻辑 | 7×24 云服务器任务与主系统代码无法复用，维护两套相似逻辑 |
| **目录结构非标准 Python 包** | 根目录下平铺 `admin`、`client`、`common` 等模块，没有统一源码根 | `sys.path` 需动态修改，易引发循环导入；PySide6 打包路径处理复杂 |
| **无统一异常处理框架** | 各任务类自己 `try...except` 包裹，错误码和告警逻辑分散 | 新增告警渠道（如企业微信、短信）需要改 N 个文件 |

### 6.2 代码层面问题

| 问题 | 具体表现 | 风险 |
|------|---------|------|
| **日志体系割裂** | `LogUtils` 基于配置文件路径生成 logger；新架构 `BaseComponent` 使用 `StreamHandler` + 环境变量控制级别 | 新旧模块日志格式、路径、级别不一致，排查问题困难 |
| **任务类无接口约束** | `TaskEtl`、`TaskRpa` 等分发器的方法仅靠文档约定返回 `ResultObj` | 运行时若返回非 `ResultObj`，`JobCore` 可能抛异常或静默失败 |
| **SQL 与 Python 混杂** | `dwd/odl/`、`dwd/dml/` 中存在大量 `.sql` 文件与 `.py` 文件混放 | SQL 变更无版本管理，数仓血缘关系靠人工记忆 |
| **一次性脚本污染仓库** | `zzlc/一次性脚本`、`辅助工具/` 中存在大量临时脚本和手工批处理 | 仓库体积膨胀，新人难以分辨哪些是生产代码 |
| **浏览器自动化与业务逻辑耦合** | `LingXingService.loginChrome` 中直接操作 `find_element(By.NAME)` | 页面结构一变，业务类也需跟着改；Selenium 异常处理逻辑散落在各 Service 中 |

### 6.3 运维与安全隐患

| 问题 | 现状 | 风险 |
|------|------|------|
| **内网无 UPS** | MySQL 跑在内网，断电会导致 T+1 依赖链断裂 | 即使 7×24 任务拆到云端，主调度的事件依赖表仍在内网 MySQL |
| **作业超时监控粒度粗** | `JobCore` 只有全局超时检查，无单作业熔断 | 某 RPA 作业因页面卡死阻塞，可能导致同节点其他作业 miss |
| **固定 IP 绑定** | 若依应用服务固定 IP，新架构若做微服务拆分需要重新规划网络 | 独立部署模块回写内网数据时存在网络联通性问题 |
| **测试覆盖率为零** | `tests/` 目录为空 | 200+ 作业的任何改动都无法通过自动化测试回归 |

---

## 七、重构建议

以下建议完全贴合用户提出的重构思路，按优先级和落地难度分层。

### 7.1 基类与继承体系（最高优先级）

**目标**：实现"单一入口，统一继承"。

```
BaseComponent（第一级：全局统一基类）
    ├── ViewBase（第二级：PySide6 视图基类）
    ├── TaskBase（第二级：RPA / ETL / DWD 任务基类）
    │       └── MultiDbTaskBase（第三级：多数据库用户支持）
    ├── ServiceBase（第二级：外部 API 服务基类）
    └── EntityBase（第二级：数据库/业务实体基类）
```

- **第一级 `BaseComponent`**（已在 `asrc/core/base/base_component.py` 实现）：
  - 识别部署模式（`independent` / `integrated`）。
  - 从 `.py` 配置文件和环境变量加载配置。
  - 提供统一 logger，支持 `ASTPY_LOG_LEVEL` 环境变量。
  - 提供数据根目录和 SQLite 目录的统一路径方法。

- **第二级按模块分基类**：
  - **任务基类**（`TaskBase`）：统一 `run()` → `action()` 模板，内置异常捕获、结果包装、邮件/企微告警钩子。
    - **`ResultObj` 保留与改造方案**（已决策）：
      - `ResultObj` 作为**调度层返回值契约**保留，不修改现有类定义（避免影响 200+ 作业）。
      - `TaskBase.run()` 的返回值强制为 `ResultObj`，作为 `JobCore` 调度引擎的统一接口。
      - `run()` 内部兼容逻辑：
        - 若 `action()` 已返回 `ResultObj`（老作业模式），直接透传。
        - 若 `action()` 返回裸数据（新作业模式），自动包成 `ResultObj.success(data=...)`。
        - 若 `action()` 抛出异常，自动捕获并包成 `ResultObj.error(ResultObj.FATAL_ERROR, ...)`，同时触发统一告警。
      - **渐进迁移策略**：老作业维持 `return ResultObj.error(...)` 不变；新作业和重构中的作业直接抛异常或返回裸数据，由 `run()` 统一包装。
  - **服务基类**（`ServiceBase`）：封装 HTTP Client、重试策略、限流逻辑，外部 Service 不再直接引用 `AdminConfig`。
  - **视图基类**（`ViewBase`）：封装 PySide6 通用布局、线程池（`AstWorker`）、进度条、消息弹窗。

- **第三级引入数据库方言和多用户**：
  - `MultiDbTaskBase` 在 `TaskBase` 基础上增加 `get_dbs(user='astdcusr')` 方法，支持按用户切换连接。

### 7.2 配置管理重构

**目标**：配置驱动、环境隔离、方便 PySide6 打包。

- **废弃 `adminconfig.yml`**，改用 `astpyconfig.py`：
  - 本地只放数据库连接信息（ host / port / user / database ），**不含密码**。
  - 密码通过环境变量 `ASTPY_DB_PASS` 注入（集成部署模式）。
  - 独立部署模式（`ASTPY_DEPLOYMENT_MODE=independent`）不读取环境变量，使用本地 SQLite。

- **其他配置入数据库**：
  - 沿用现有的 `astdc.dc_sys_config` 表做配置中心。
  - SQLite 部署时，配置默认保存在 `D:\adata\sqlitedb\ast_sys_config.db` 或 `~/adata/sqlitedb/ast_sys_config.db`。

- **数据根目录统一**：
  - 优先级：`D:\adata` > `~/adata`。
  - 子目录规划：`sqlitedb/`（数据库）、`log/`（日志）、`ctl/`（控制文件）。

### 7.3 数据库层重构

**目标**：多数据库方言支持，连接逻辑收敛，不引入连接池。

- 在 `asrc/core/database/` 中封装新的数据库管理器：
  - 保留"短平快"原则：每次 `action()` 内按需创建连接，执行完关闭。
  - 支持 `mysql` / `sqlite` 方言切换，由 `Settings.get_db_config()` 统一决定。
  - 业务代码不再直接 `new DBcore()`，而是通过基类方法 `self.get_dbs()` 获取。

- **多数据库用户支持**：
  - 当前已有 `astdcusr`（应用）、`astusr`（数仓）、`planusr`（计划）等多个用户。
  - 在第三级基类中实现 `get_dbs(user='astdcusr')`，自动读取对应用户的连接配置。

### 7.4 目录与包结构迁移

**目标**：符合标准 Python 包规范，消除 `sys.path` 动态修改。

```
astpy/
├── asrc/                        # [新] 统一源码根
│   ├── core/                     # 全局基础设施
│   │   ├── config/               # 配置管理
│   │   ├── base/                 # 三级基类体系
│   │   ├── database/             # 数据库封装（替代旧 DBcore.py）
│   │   ├── utils/                # 通用工具（迁移自 utils/）
│   │   ├── entity/               # 通用实体（迁移自 common/entity/）
│   │   └── service/              # 外部系统 API 集成（迁移自 CoreBussiness/）
│   ├── etl/                      # 数据采集 + 离线数仓（整合 ETL/ + dwd/）
│   ├── rpa/                      # RPA 脚本（迁移自 rpa/）
│   ├── client/                   # PySide6 桌面应用（迁移自 client/src/）
│   └── tasks/                    # 定时任务调度引擎 + 任务分发器
├── tests/                        # 单元测试 / 集成测试
├── scripts/                      # 运维脚本、数据库迁移脚本
├── pyproject.toml                # 项目依赖与配置（建议补充）
└── README.md
```

- 迁移完成后，逐步删除根目录下的旧 `admin/`、`common/`、`CoreBussiness/`、`dwd/`、`ETL/`、`rpa/`、`task/` 等顶层模块。
- `zzlc/` 和 `辅助工具/` 中的一次性脚本评估后：有价值的迁入 `scripts/`，废弃的删除。

### 7.5 外部服务解耦

**目标**：Service 层可独立打包部署，自带配置。

- 所有外部 Service 继承 `ServiceBase`，配置通过基类注入。
- 浏览器自动化（Selenium）抽取为 `BrowserDriver` 组件：
  - `BrowserDriver` 负责 `openChrome`、`login`、`wait_element`、`screenshot`。
  - `LingXingRpaService` 只负责业务逻辑（打开哪个菜单、填什么表单、取什么数据）。
- 不再直接引用 `AdminConfig` 单例，改为通过 `self.settings.get('lx_api_url')` 读取。

### 7.6 定时任务调度增强

**目标**：动态导入更安全，任务类有统一契约，支持完全热加载。

- **取消 `task.modules` 分发器模式**：
  - 旧架构通过 `TaskEtl`、`TaskRpa`、`TaskDwd` 等分发器中转调用，所有任务方法堆在一个类里，新增任务需要改分发器并重启调度器。
  - 新架构直接由 `JobCore` 通过 `component` 字段定位到具体任务类，一个任务一个文件一个类，彻底去掉中间分发器层。
- 任务类直接实例化运行：
  - 旧：`EtlService(kwargs['jobname']).j00002_getSuspOrder()`
  - 新：`J00002GetSuspOrderJob(**kwargs).run()`（一个类一个文件）
- `JobCore.dynamic_import_class_method_call()` 增加类型校验：
  - 检查 `cls` 是否为 `TaskBase` 子类。
  - 检查是否实现了 `action` 方法。
  - 启动期校验失败立即报错，而不是运行时才暴露。
- **完全热加载（不停机部署）**：
  - `JobCore` 在每次调度任务前，根据类文件修改时间判断是否需要 `importlib.reload`。
  - 新增类文件或更新现有类文件后，直接部署到目标目录即可，无需重启 `start_task.py` 或 `JobCore`。
  - 运行中的任务实例绑定到其启动时的类版本，不会被中途重载，避免状态污染。
  - 结合 `dc_job_info` 表的 `component` 字段（格式：`etl.jobs.j00002_get_susp_order:J00002GetSuspOrderJob`），实现"配置即路由"。
- 作业配置表的 `component` 字段规范：`模块路径:类名`，如 `etl.jobs.j00002_get_susp_order:J00002GetSuspOrderJob`。

### 7.7 日志与监控统一

**目标**：全项目一套日志规范，错误处理统一。

- 统一使用 `BaseComponent._init_logger()`：
  - 格式：`[时间] [组件名] [级别] [文件名:行号] 消息`
  - 文件输出：使用 `RotatingFileHandler`（5MB / 366 个备份），路径收敛到数据根目录的 `log/`。
  - 控制台输出：保留，但取消 `InfoFilter`，允许 WARNING 及以上级别打印（便于排查）。
- 错误处理下沉到 `TaskBase.run()`：
  - 子类 `action()` 只需抛异常或返回 `ResultObj`。
  - 基类自动捕获异常，写入 `dc_job_error_log`，并根据维护人员列表发邮件/企微通知。

### 7.8 测试与质量门禁

**目标**：从 0 到 1 建立测试体系。

- 为核心基类（`BaseComponent`、`Settings`、数据库方言切换）编写单元测试。
- 为 `ResultObj`、`LogUtils` 等纯工具类编写测试。
- 外部 Service 类使用 `responses` / `pytest-httpx` 做 API Mock 测试。
- 不建议对 200+ 个定时作业全部补测试，但**新开发的任务必须附带测试用例**。

---

## 八、新旧架构对比

| 维度 | 旧架构 | 新架构（asrc） |
|------|--------|---------------|
| **基类** | 各自为政（AdminBase、EtlAdmin、DwdAdminNew） | 三级统一继承体系（BaseComponent → TaskBase/ServiceBase → MultiDbTaskBase） |
| **配置** | 单一 YAML 文件，密码硬编码 | `.py` 配置 + 环境变量 + 数据库配置表，支持 independent/integrated 模式 |
| **数据库** | DBcore 散落各处，仅支持 MySQL | core/database 统一封装，MySQL/SQLite 方言切换，多用户支持 |
| **目录** | 根目录平铺模块 | 标准 Python 包结构，`asrc/` 为源码根 |
| **任务开发** | 一个分发器类里堆多个方法 | 一个任务一个文件一个类，强制继承 TaskBase |
| **外部服务** | 直接依赖 AdminConfig | 继承 ServiceBase，配置注入，可独立部署 |
| **日志** | LogUtils 基于配置文件 | BaseComponent 统一 logger，环境变量控制级别 |
| **错误处理** | 各任务自己 try/except | TaskBase.run() 统一捕获，自动记录并告警 |
| **打包部署** | PySide6 打包路径复杂 | 标准包结构，打包逻辑简化 |

---

## 九、迁移路径建议

### 阶段一：夯实基座（2~3 周）
1. 完善 `asrc/core/`：
   - `BaseComponent` 已就绪，补充 `TaskBase`、`ServiceBase`、`ViewBase`。
   - `database/` 实现 MySQL/SQLite 方言切换，支持多用户。
   - `config/settings.py` 增加配置热刷新能力（可选）。
2. 编写核心基类单元测试，确保基座稳定。

### 阶段二：试点迁移（2~3 周）
1. 挑选一个**低风险、业务简单**的域做试点，例如：
   - `task/modules/TaskSys.py`（系统级任务，如刷新权限、清理日志）。
   - 或 `rpt/` 报表类任务（纯读数据，无外部 API 依赖）。
2. 整域迁移到 `asrc/tasks/` 或 `asrc/etl/`，验证：
   - 基类初始化流程是否正常。
   - 数据库切换（MySQL ↔ SQLite）是否无缝。
   - `JobCore` 动态导入新架构类是否正常。
   - 日志、告警、邮件是否统一生效。

### 阶段三：分域迁移（持续 2~3 个月）
1. 按业务域逐个迁移：
   - 先 ETL/ODS（数据采集逻辑相对标准）。
   - 再 DWD/BAL（SQL 多，但主要是 `DwdAdminNew` 子类）。**已决策**：`dwd/` 中的 `.sql` 文件均为业务逻辑 SQL（非 DDL），不做版本号文件命名；迁移时逐步将 `.sql` 改造为 `.py` 脚本，由 Python 统一包裹执行，便于在 SQL 运行前后植入事前检查（如数据完整性、幂等性校验）和事后检查（如行数核对、指标波动告警）。
   - 最后 RPA（浏览器自动化逻辑复杂，改动面大）。
2. 每迁移一个域，就在 `task/modules/` 中替换对应的分发器，旧代码逐步下线。

### 阶段四：客户端与收尾（2~4 周）
1. 将 `client/src/` 迁移到 `asrc/client/`，基于 `ViewBase` 重构通用界面组件。
2. 删除根目录下旧模块（`admin/`、`common/`、`CoreBussiness/`、`dwd/`、`ETL/`、`rpa/`、`task/`）。
3. `zzlc/` 和 `辅助工具/` 处理（**已决策**：有价值的运维脚本和部署工具迁入 `scripts/`，一次性脚本全部删除或归档）。
4. 补充 `pyproject.toml`，规范依赖管理。

---

## 附录：核心代码位置索引

| 模块 | 关键文件 | 说明 |
|------|---------|------|
| 调度引擎 | `task/JobCore.py` | APScheduler 封装、作业分发、依赖检查 |
| 调度入口 | `task/start_task.py` | 调度器启动脚本 |
| 配置中心 | `admin/service/adminconfig.py` | 全局 YAML 配置读取 |
| 数据库核心 | `admin/service/DBcore.py` | MySQL 连接封装（pymysql + pandas） |
| 通用基类 | `common/BaseObj.py` | `AdminBase`（邮件、数据库、节假日检查） |
| 返回对象 | `common/ResultObj.py` | 统一结果对象（code + message + data） |
| ETL 基类 | `ETL/EtlAdmin.py` | ETL 作业模板（run → action） |
| 数仓基类 | `dwd/DwdAdmin.py` | 数仓作业模板（带夜间运行时间限制） |
| 日志工具 | `utils/LogUtils.py` | 按作业名分文件的 RotatingFileHandler |
| 新基类 | `asrc/core/base/base_component.py` | 第一级统一基类（已就绪） |
| 新配置 | `asrc/core/config/settings.py` | 新配置管理（.py + 环境变量） |
| 客户端入口 | `client/index.py` | PySide6 应用启动 |
| 领星服务 | `CoreBussiness/LingXingService.py` | 领星 API + 浏览器登录封装 |
| 尾程测算 | `dwd/bal/BalCalculate.py` | 美东美西尾程配仓测算（线程池版） |

---

> **总结**：astpy 项目功能成熟、业务覆盖广，但早期架构设计导致基类碎片化、配置硬编码、数据库连接散落、目录结构非标准。当前 `asrc/` 新基座（`BaseComponent` + `Settings`）方向正确，建议按"**夯实基座 → 试点验证 → 分域迁移 → 收尾清理**"四阶段推进重构，优先统一基类和配置管理，再逐步迁移 200+ 个作业。
