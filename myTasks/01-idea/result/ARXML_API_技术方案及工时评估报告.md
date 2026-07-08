# AUTOSAR IaC 三层 ARXML API 开发 — 技术方案及工时评估报告

> **版本**：v1.0  
> **日期**：2026-07-07  
> **依据**：《ARXML_API_开发设计思路_v0.1.md》+《ARXML_API_开发设计思路.md》（详细版）+《AUTOSAR Integration-as-Code (IaC) 技术方案 v0.1》  
> **假设**：从零启动开发（L1 底座已有 440k 行生成代码及手写运行时，本报告聚焦 L2/L3 补齐及 L1 增强）  
> **团队**：5 人（何康、刘学海、庄希琦、刘鹏、余小林），其中余小林 50% 负荷 → 等效 **4.5 FTE**

---

## 目录

1. [项目概述与范围](#1-项目概述与范围)
2. [总体架构回顾](#2-总体架构回顾)
3. [任务拆解全景（WBS）](#3-任务拆解全景wbs)
4. [Phase 1：基础设施与 L1 增强](#4-phase-1基础设施与-l1-增强)
5. [Phase 2：L2 模块与协议栈封装层](#5-phase-2l2-模块与协议栈封装层)
6. [Phase 3：L3 高阶 Feature API 业务层](#6-phase-3l3-高阶-feature-api-业务层)
7. [Phase 4：测试体系与质量保障](#7-phase-4测试体系与质量保障)
8. [Phase 5：文档、CI/CD 与交付](#8-phase-5文档cicd-与交付)
9. [工时汇总与人力资源排布](#9-工时汇总与人力资源排布)
10. [里程碑与甘特图](#10-里程碑与甘特图)
11. [风险管理](#11-风险管理)
12. [交付物清单](#12-交付物清单)

---

## 1. 项目概述与范围

### 1.1 项目目标

在 AutosarFactory 现有 L1 底座之上，构建完整的 **三层 Python API 体系**，支撑「软件定义集成」的 IaC（Integration-as-Code）理念落地：

| 层 | 定位 | 当前状态 | 本项目目标 |
|---|------|---------|-----------|
| **L1** 基础 ARXML API | 元模型原子操作底座 | ✅ 已成熟（440k 行代码） | 增强：会话管理、验证框架、版本适配 |
| **L2** 模块/协议栈 API | 封装单模块参数约束 | ❌ 缺失（仅 MCP 有数据） | **从零构建**：14+ BSW 模块 Configurator |
| **L3** 高阶 Feature API | 跨模块整车集成业务 | ❌ 缺失（仅 Examples/） | **从零构建**：4 个标杆 Feature + 脚手架 |

### 1.2 核心设计铁律

1. **依赖只向下流动**：L3 → L2 → L1，L1 对上层一无所知
2. **厂商固化底座，企业自建业务**：L1/L2 由厂商维护，L3 由企业基于 L1/L2 自建
3. **顺应 L1 机制，不可绕过**：脏标记、累积合并、schema 顺序、双向引用 —— L2/L3 必须只用 setter
4. **单进程单模型**：CI 并发用多进程 + `reinit()` 边界

### 1.3 不在本报告范围

- L1 元模型类代码生成器开发（生成器本身不在本仓库）
- MCP 服务器的全新工具开发（14 个已有工具仅需适配扩展）
- 企业私有 L3 Feature 资产库建设（本报告提供模板与脚手架）
- AUTOSAR 规范 PDF 知识库构建（`kb_builder` 独立项目）

---

## 2. 总体架构回顾

### 2.1 分层全景

```
┌──────────────────────────────────────────────────────────────────────┐
│  L3  高阶 Feature API（业务层 / 企业资产）                               │
│      跨模块整车集成场景 · 企业规范沉淀 · 最佳实践模板                       │
│      例：build_can_gateway() / apply_diag_route() / apply_pdu_packing()  │
├────────────────────────────────────────────────────────────────────────┤
│  L2  模块 & 协议栈 API（封装层）                                          │
│      BSW 模块参数约束 · 标准化接口 · 屏蔽 ARXML 细节 · 参数依赖            │
│      例：OsConfigurator / CanIfConfigurator / RteConfigurator            │
│      数据来源：ecuc_param_def.json（ECUC 参数定义）                       │
├────────────────────────────────────────────────────────────────────────┤
│  L1  基础 ARXML API（底座 / AutosarFactory 内核，不可修改）                 │
│      原子增删改查 · 格式校验 · 自动生成 · 多文件合并 · 引用解析             │
│      元模型约定：get_/set_/new_/add_/remove_/get_children                  │
├────────────────────────────────────────────────────────────────────────┤
│  横切：MCP 服务（AI 赋能）· CI/CD 适配 · 多 Release 快照（R4.0 … R24-11）    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 新增包结构

```
autosarfactory/
├── autosarfactory.py              # L1: 现有，保持（仅增加 session 上下文管理器）
├── datatype_utils.py              # L1: 现有
├── XmlElementDirtyTracker.py      # L1: 现有
│
├── modules/                       # ★ L2: 新增——单模块封装
│   ├── __init__.py
│   ├── base.py                    #    ModuleConfigurator 基类
│   ├── registry.py                #    Definition Registry（ecuc_param_def.json 索引）
│   ├── com/                       #    通信协议栈
│   │   ├── __init__.py
│   │   ├── com.py                 #    Com
│   │   ├── comm.py                #    ComM
│   │   ├── pdur.py                #    PduR
│   │   ├── canif.py               #    CanIf
│   │   ├── can.py                 #    Can
│   │   ├── cantrcv.py             #    CanTrcv
│   │   ├── cansm.py               #    CanSM
│   │   └── linif.py               #    LinIf
│   ├── diag/                      #    诊断协议栈
│   │   ├── __init__.py
│   │   ├── dcm.py                 #    Dcm
│   │   ├── dem.py                 #    Dem
│   │   └── fim.py                 #    FiM
│   ├── memory/                    #    存储协议栈
│   │   ├── __init__.py
│   │   ├── nvm.py                 #    NvM
│   │   └── fee.py                 #    Fee
│   ├── system/                    #    系统服务
│   │   ├── __init__.py
│   │   ├── bswm.py                #    BswM
│   │   ├── ecum.py                #    EcuM
│   │   └── wdgm.py                #    WdgM
│   ├── crypto/                    #    加密协议栈
│   │   ├── __init__.py
│   │   └── csm.py                 #    Csm
│   └── ethernet/                  #    以太网协议栈
│       ├── __init__.py
│       ├── tcpip.py               #    TcpIp
│       ├── soad.py                #    SoAd
│       └── doip.py                #    DoIp
│
├── protocols/                     # ★ L2: 新增——协议栈级协同
│   ├── __init__.py
│   ├── base.py                    #    ProtocolStackBase 基类
│   ├── com_stack.py               #    完整 ComStack 协同配置
│   ├── diag_stack.py              #    完整 DiagStack 协同配置
│   └── mem_stack.py               #    完整 MemStack 协同配置
│
├── features/                      # ★ L3: 新增——业务 Feature
│   ├── __init__.py
│   ├── base.py                    #    FeatureBase 基类 + 合规闸门
│   ├── can_builder.py             #    CAN 通信矩阵构建器
│   ├── diag_builder.py            #    诊断配置构建器
│   ├── pdu_packing.py             #    PDU 打包规则引擎
│   └── variant_manager.py         #    变体管理器
│
├── validation/                    # ★ 新增——跨层验证框架
│   ├── __init__.py
│   ├── base.py                    #    验证规则基类 + 注册机制
│   ├── rules.py                   #    内置规则集
│   └── report.py                  #    验证报告生成
│
├── adapters/                      # ★ 新增——版本适配层
│   ├── __init__.py
│   ├── base.py                    #    适配器基类
│   └── releases/                  #    各版本适配器
│       ├── r403.py
│       ├── r422.py
│       └── r2411.py               #    默认（R24-11）
│
└── session.py                     # ★ 新增——会话管理（上下文管理器）
```

---

## 3. 任务拆解全景（WBS）

```
1.0 AUTOSAR IaC 三层 ARXML API 开发
│
├── 1.1 需求分析与规格定义
│   ├── 1.1.1 L2 模块需求规格（14 个 BSW 模块参数约束清单）
│   ├── 1.1.2 L3 Feature 需求规格（4 个整车场景用例）
│   └── 1.1.3 非功能需求（性能、兼容性、CI/CD）
│
├── 1.2 架构设计
│   ├── 1.2.1 L2 基类框架详细设计
│   ├── 1.2.2 L3 基类框架详细设计
│   ├── 1.2.3 验证框架详细设计
│   ├── 1.2.4 版本适配层详细设计
│   └── 1.2.5 接口契约与异常规范定义
│
├── 1.3 基础设施实现（L1 增强）
│   ├── 1.3.1 会话上下文管理器（session）
│   ├── 1.3.2 验证框架基座
│   ├── 1.3.3 版本适配层基座
│   └── 1.3.4 Definition Registry（ecuc_param_def.json 加载与索引）
│
├── 1.4 L2 模块实现（分 4 个协议栈族并行）
│   ├── 1.4.1 ModuleConfigurator 基类 + ContainerProxy
│   ├── 1.4.2 通信协议栈（Com / ComM / PduR / CanIf / Can / CanTrcv / CanSM / LinIf）
│   ├── 1.4.3 诊断协议栈（Dcm / Dem / FiM）
│   ├── 1.4.4 存储协议栈（NvM / Fee）
│   ├── 1.4.5 系统服务（BswM / EcuM / WdgM）
│   ├── 1.4.6 以太网协议栈（EthIf / TcpIp / SoAd / DoIp）
│   ├── 1.4.7 加密协议栈（Csm / CryIf）
│   └── 1.4.8 协议栈协同层（ComStack / DiagStack / MemStack）
│
├── 1.5 L3 Feature 实现
│   ├── 1.5.1 FeatureBase 基类 + 合规闸门
│   ├── 1.5.2 CAN 通信矩阵构建器（CanCommunicationBuilder）
│   ├── 1.5.3 诊断配置构建器（DiagnosticConfigurator）
│   ├── 1.5.4 PDU 打包规则引擎
│   └── 1.5.5 变体管理器（VariantManager）
│
├── 1.6 单元测试
│   ├── 1.6.1 L1 增强功能测试
│   ├── 1.6.2 L2 各模块 Configurator 单元测试
│   ├── 1.6.3 L3 Feature 单元测试
│   └── 1.6.4 验证框架单元测试
│
├── 1.7 集成验证
│   ├── 1.7.1 端到端场景测试（读入→配置→保存→CP 校验）
│   ├── 1.7.2 多版本兼容性测试
│   ├── 1.7.3 性能基准测试
│   └── 1.7.4 回归测试套件
│
├── 1.8 文档与示例
│   ├── 1.8.1 API 参考文档
│   ├── 1.8.2 开发者指南
│   ├── 1.8.3 使用示例（Examples/ 扩充）
│   └── 1.8.4 CI/CD 模板与集成指南
│
└── 1.9 项目管理
    ├── 1.9.1 技术评审（设计评审、代码评审）
    ├── 1.9.2 进度跟踪与风险管理
    └── 1.9.3 交付验收
```

---

## 4. Phase 1：基础设施与 L1 增强

### 4.1 任务详情

#### 1.1.1 ~ 1.1.3 需求分析与规格定义（8 PD）

| 子任务 | 描述 | 输出 |
|--------|------|------|
| L2 模块需求规格 | 逐模块梳理 ECUC 参数定义（从 `ecuc_param_def.json` 提取），明确每个模块的容器层级、必填参数、枚举约束、参数依赖关系 | 《L2 模块需求规格说明书》 |
| L3 Feature 需求规格 | 定义 4 个整车场景的输入/输出契约、跨模块联动需求、企业规范规则 | 《L3 Feature 需求规格说明书》 |
| 非功能需求 | 性能基线（大模型 < 30s save）、兼容矩阵（R4.0.3 ~ R24-11）、CI/CD 集成要求 | 《非功能需求规格》 |

#### 1.2.1 ~ 1.2.5 架构设计（13 PD）

| 子任务 | 描述 | 输出 |
|--------|------|------|
| L2 基类框架设计 | `ModuleConfigurator` 完整接口定义，容器代理（ContainerProxy）设计，参数读写、引用管理、校验入口 | 《L2 架构设计文档》 |
| L3 基类框架设计 | `FeatureBase` 声明式输入 → `build()` → `validate()` 三段式设计，合规闸门机制 | 《L3 架构设计文档》 |
| 验证框架设计 | 可插拔规则引擎，规则注册/发现机制，验证报告格式设计 | 《验证框架设计文档》 |
| 版本适配层设计 | 适配器模式设计，API 差异屏蔽策略，版本能力声明机制 | 《版本适配层设计文档》 |
| 接口契约定义 | 跨层异常翻译规范（L1 元模型异常 → L2 业务异常 → L3 用户异常），API 稳定性契约 | 《API 契约与异常规范》 |

#### 1.3.1 ~ 1.3.4 基础设施实现（18 PD）

| 子任务 | 描述 | 关键实现点 |
|--------|------|-----------|
| **会话上下文管理器** | `autosarfactory.session()` 上下文管理器，支持 `with` 语法，退出自动 `reinit()` | 修改 `autosarfactory.py`；维护会话栈支持嵌套；异常安全保证 |
| **验证框架基座** | `validation/` 包，`ValidationRule` 基类、`RuleRegistry` 注册表、`ValidationReport` 结构化报告 | 装饰器注册规则；支持规则优先级与前置条件；报告含 WARNING/ERROR 分级 |
| **版本适配层基座** | `adapters/` 包，`ReleaseAdapter` 基类，版本能力查询 API | 适配器发现机制；默认 R24-11 适配器；按需加载而非全量导入 |
| **Definition Registry** | `registry.py`，加载 `ecuc_param_def.json` 构建内存索引，提供 `path → 参数定义` O(1) 查询 | 复用 MCP 索引逻辑；延迟加载；支持按模块/容器/参数三级查询 |

---

## 5. Phase 2：L2 模块与协议栈封装层

### 5.1 ModuleConfigurator 基类（8 PD）

**核心抽象**：

```
ModuleConfigurator
├── 模块定位          _locate_or_create_module()
├── 容器导航          container(path) → ContainerProxy
├── 参数读写          get_param() / set_param() / set_param_if_absent()
├── 子容器管理        add_container() / remove_container() / ensure_container()
├── 引用管理          add_reference() / resolve_reference()
├── 定义绑定          bind_definition() / get_definition()
├── 约束校验          validate() → list[ValidationError]
└── 序列化            export_summary() → dict
```

**ContainerProxy**：惰性代理对象，按需解析容器路径，缓存子容器索引，提供链式访问 `config.container("CanIf/CanIfInitHoh").set_param("CANIF_ARC_HANDLE_ID", 1)`。

### 5.2 BSW 模块封装器矩阵（共 77 PD）

#### 通信协议栈（ComStack）— 35 PD

| 模块 | PD | 封装要点 |
|------|-----|---------|
| **Com** | 8 | IPdu / IPduGroup / ISignal / ISignalIPdu 映射；周期/事件触发配置；传输模式（DIRECT / PERIODIC / MIXED）；信号网关路由 |
| **ComM** | 3 | 通信模式管理；通道状态机配置；用户/通道映射；网络管理协调 |
| **PduR** | 8 | 路由路径表配置（CanIf↔Com / CanIf↔Dcm / CanIf↔CanTp）；路由策略（DIRECT / FLEXIBLE）；多播/网关路由 |
| **CanIf** | 8 | HOH（Hardware Object Handle）管理；Tx/Rx PDU 通道配置；CAN ID 过滤；上层/下层接口绑定；CanTrcv 关联 |
| **Can** | 3 | CAN 控制器配置；波特率/位时序；CAN FD 支持；硬件对象（HOH）底层配置 |
| **CanTrcv** | 2 | CAN 收发器配置；唤醒模式；Pn 支持 |
| **CanSM** | 2 | CAN 状态机；Bus-Off 恢复策略；网络管理协调 |
| **LinIf** | 1 | LIN 接口基本配置；调度表绑定 |

#### 诊断协议栈（DiagStack）— 19 PD

| 模块 | PD | 封装要点 |
|------|-----|---------|
| **Dcm** | 8 | 诊断会话控制（Default / Programming / Extended）；安全访问等级；DID（Data Identifier）读写配置；RID（Routine Identifier）例程控制；传输层绑定（CanTp / DoIp） |
| **Dem** | 8 | 诊断事件（DemEvent）创建与 Debounce 算法配置；DTC 状态位管理；老化计数器/扩展数据记录；事件存储阈值；使能条件（Enable Condition）绑定 |
| **FiM** | 3 | 功能抑制管理；事件 → FiM 函数映射；抑制优先级；恢复策略 |

#### 存储协议栈（MemStack）— 8 PD

| 模块 | PD | 封装要点 |
|------|-----|---------|
| **NvM** | 5 | NvM 块（Block）管理；块类型（NATIVE / REDUNDANT / DATASET）；CRC/写入验证；作业优先级；Fee/Ea 绑定 |
| **Fee** | 3 | Flash EEPROM 模拟；扇区配置；写入队列；地址映射 |

#### 系统服务 — 11 PD

| 模块 | PD | 封装要点 |
|------|-----|---------|
| **BswM** | 5 | BSW 模式管理器；模式仲裁规则；模式切换动作列表；模式请求接口 |
| **EcuM** | 3 | ECU 状态管理器；启动/关机阶段配置；唤醒源配置；睡眠模式 |
| **WdgM** | 3 | 看门狗管理器；监控实体配置；存活指示；超时监督 |

#### 以太网协议栈 — 13 PD

| 模块 | PD | 封装要点 |
|------|-----|---------|
| **EthIf** | 5 | 以太网接口；VLAN 配置；控制器绑定；收发模式 |
| **TcpIp** | 5 | TCP/IP 协议栈；Socket 配置；地址分配；端口映射 |
| **SoAd** | 2 | Socket Adaptor；传输层绑定；组播配置 |
| **DoIp** | 1 | Diagnostic over IP；诊断地址；传输层绑定 |
| **Csm** | 2 | 加密服务管理；密钥配置；算法选择；Job 处理 |

### 5.3 协议栈协同层（13 PD）

| 模块 | PD | 封装要点 |
|------|-----|---------|
| **ComStack** | 5 | 跨 Com/ComM/PduR/CanIf/Can/CanTrcv/CanSM 的一致性校验；PDU 路由完整性检查；通信通道端到端验证 |
| **DiagStack** | 5 | 跨 Dcm/Dem/FiM/NvM 的一致性校验；DTC 事件链完整性；诊断地址冲突检查 |
| **MemStack** | 3 | 跨 NvM/Fee 的一致性校验；块大小/地址对齐检查 |

### 5.4 L2 约束校验器（内嵌于各模块，不单独计时）

每个 Configurator 内建 `validate()` 方法，校验维度：

| 校验类型 | 规则来源 | 示例 |
|---------|---------|------|
| 必填项检查 | `ecuc_param_def.json` multiplicity 下限 | `OsTask` 必须绑定 `OsScheduleTable` |
| 取值范围检查 | 参数 min/max | `priority` ∈ [1, 255] |
| 枚举合法性 | `get_ecuc_param_literals` | `trigger_type` ∈ {PERIODIC, MIXED, NONE} |
| 依赖完整性 | 模块内参数依赖关系 | `OsAlarm` 引用的 `OsCounter` 必须存在 |
| 引用有效性 | 目标节点存在性 | `set_definition(ref)` 的 ref 非空且类型正确 |

---

## 6. Phase 3：L3 高阶 Feature API 业务层

### 6.1 FeatureBase 基类（5 PD）

```
FeatureBase
├── 声明式输入        add_xxx() 系列（积累业务描述）
├── 构建              build() — 编排 L1 + L2，一次性生成 ARXML
├── 合规闸门          assert_compliant() — 企业规范合规校验
├── 幂等性            hash_input() — 支持 diff 与缓存
├── 输出摘要          export_summary() — 人类可读的配置摘要
└── 回滚              dry_run() — 预执行不写文件
```

### 6.2 四个标杆 Feature（38 PD）

#### CAN 通信矩阵构建器（10 PD）

```python
builder = CanCommunicationBuilder(ar_package)
builder.define_can_cluster("CAN1", baudrate=500000, protocol="CAN_FD")
builder.add_ecu("EngineECU", connected_clusters=["CAN1", "CAN2"])
builder.add_pdu("EngineData", id=0x100, dlc=8,
                src_ecu="EngineECU", dst_ecus=["BCM", "IPC"],
                cycle_time_ms=10, trigger="PERIODIC")
builder.add_signal("EngineSpeed", start_bit=0, length=16,
                   data_type=uint16, pdu="EngineData",
                   init_value=0, endianness="MOTOROLA")
builder.build()     # 自动创建：Com IPdu/IPduGroup/ISignal + PduR 路由
                    #          + CanIf HOH/TxPdu + Can 控制器 + EcuC
builder.validate()  # 跨模块一致性、信号不重叠、PDU DLC 校验
```

**编排的模块**：Com + ComM + PduR + CanIf + Can + CanTrcv + CanSM + EcuC（跨 8 个模块联动）

#### 诊断配置构建器（10 PD）

**场景**：配置 ECU 诊断基础参数、DTC 定义、DID 读写、安全访问

**编排的模块**：Dcm + Dem + FiM + NvM + CanTp/DoIp（跨 5 个模块联动）

#### PDU 打包规则引擎（5 PD）

**场景**：企业 PDU 打包规范版本化（如 `company_pdu_packing_v3`），信号排布规则、字节对齐、跨 PDU 空间优化

#### 变体管理器（8 PD）

**场景**：Post-Build / Pre-Compile 变体条件定义、绑定；多车型共享部件条件化配置；按变体提取子集输出

**剩余 5 PD** 用于 Feature 模板库建设（2~3 个额外 Feature 模板 + 企业规范沉淀指南文档）

---

## 7. Phase 4：测试体系与质量保障

### 7.1 单元测试（40 PD）

| 测试对象 | PD | 测试重点 |
|---------|-----|---------|
| L1 增强功能 | 3 | `session()` 嵌套/异常安全；验证框架规则注册与执行；版本适配器 API 查询；Definition Registry 索引正确性 |
| L2 基类 | 3 | ModuleConfigurator 生命周期；ContainerProxy 惰性解析；参数读写与校验联动 |
| L2 ComStack（7 个模块） | 10 | 各模块参数约束、枚举合法性、必填项、引用有效性、边界值 |
| L2 DiagStack（3 个模块） | 6 | DTC 状态位、Debounce 算法、会话控制、事件依赖 |
| L2 其他栈（Mem/System/Eth/Crypto） | 8 | 各模块核心参数校验、容器层级正确性 |
| L2 协议栈协同 | 3 | 跨模块一致性校验逻辑 |
| L3 Feature 基类 | 2 | 构建幂等性、合规闸门、dry_run |
| L3 CAN Builder | 3 | 输入合法性与输出正确性、信号不重叠 |
| L3 Diag Builder | 2 | DTC 链完整性、地址冲突 |

### 7.2 集成验证（25 PD）

| 测试类型 | PD | 描述 |
|---------|-----|------|
| 端到端场景测试 | 10 | 真实 ARXML 输入 → L2/L3 配置 → `save()` → CP 工具校验通过；覆盖 CAN 通信、诊断路由、PDU 打包、变体提取 4 个用例 |
| 多版本兼容性测试 | 5 | R4.0.3 / R4.2.2 / R24-11 三个版本的 L2 配置一致性验证 |
| 性能基准测试 | 5 | 100 信号 × 50 PDU × 10 ECU 的批量构建性能；`save()` 增量写入正确性；大模型内存占用 |
| 回归测试套件 | 5 | 扩展现有 38 个 L1 用例，确保 L2/L3 代码不破坏 L1 行为；CI 自动回归 |

**硬性约束**：所有新测试**必须**在 teardown 调用 `AF.reinit()`，沿用现有测试规范。

---

## 8. Phase 5：文档、CI/CD 与交付

### 8.1 文档与示例（13 PD）

| 文档 | PD | 内容 |
|------|-----|------|
| API 参考文档 | 5 | L2 所有 Configurator 的公开方法签名、参数说明、异常契约；L3 Feature 接口文档；验证规则清单 |
| 开发者指南 | 5 | 如何开发新的 L2 Configurator；如何开发 L3 Feature；如何贡献验证规则；版本适配器开发指南 |
| 使用示例 | 3 | 扩充 `Examples/` 目录，覆盖 CAN 通信、诊断配置、变体管理、PDU 打包 4 个端到端示例 |

### 8.2 CI/CD 集成（8 PD）

| 任务 | PD | 描述 |
|------|-----|------|
| CI Pipeline 模板 | 5 | `.github/workflows/` 或 Jenkinsfile 模板；lint → unit test → integration test → build check 流水线 |
| IaC 集成脚本模板 | 3 | `integrate_vehicle.py` 模板（`reinit → read → Feature.build → save → validate`）；变体批量构建脚本 |

### 8.3 项目管理（15 PD）

| 活动 | PD | 频次 |
|------|-----|------|
| 设计评审 | 5 | 每 Phase 启动时 1 次 × 5 phases × 1 PD |
| 代码评审 | 5 | 每个模块合入前 peer review |
| 进度跟踪与风险管理 | 5 | 每周例会、里程碑检查 |

---

## 9. 工时汇总与人力资源排布

### 9.1 总工时汇总

| Phase | 内容 | 人天 (PD) |
|-------|------|----------|
| **P1** | 需求分析 + 架构设计 | 21 |
| **P2** | 基础设施实现（L1 增强） | 18 |
| **P3** | L2 基类 + ComStack（7 模块） | 43 |
| **P4** | L2 DiagStack + MemStack + 系统服务（8 模块） | 38 |
| **P5** | L2 EthStack + Crypto + 协同层（8 模块） | 29 |
| **P6** | L3 Feature 业务层（4 Feature + 基类） | 43 |
| **P7** | 单元测试 + 集成验证 | 65 |
| **P8** | 文档 + CI/CD + 项目管理 | 36 |
| **合计** | | **293 PD** |

> **等效人月**：293 ÷ 22 工作日/月 ≈ **13.3 人月**

### 9.2 人力资源排布

| 团队成员 | 负荷 | 主攻方向 | 备注 |
|---------|------|---------|------|
| **何康** | 100% | L2 ComStack 模块封装（Com / CanIf / PduR）+ ComStack 协同层 | 核心通信协议栈 |
| **刘学海** | 100% | L2 DiagStack + MemStack + 系统服务模块封装 | 诊断与存储协议栈 |
| **庄希琦** | 100% | L3 Feature 业务层 + 验证框架 + 集成测试 | 业务层与质量保障 |
| **刘鹏** | 100% | L1 增强（session/验证框架/版本适配/Registry）+ L2 EthStack + Crypto | 基础设施与以太网 |
| **余小林** | 50% | L2 基类设计 + CI/CD + 文档 | 架构与 DevOps（跨组协调） |

### 9.3 工期计算

- **总工时**：293 PD
- **等效人数**：4.5 FTE
- **理论最短工期**：293 ÷ 4.5 ≈ **65 个工作日 ≈ 13 周 ≈ 3.3 个日历月**
- **考虑并行度损失（~15%）**：约 **75 个工作日 ≈ 15 周 ≈ 3.8 个日历月**
- **考虑评审/返工缓冲（~10%）**：约 **82 个工作日 ≈ 16.5 周 ≈ 4.1 个日历月**

**建议项目周期**：**4 ~ 4.5 个日历月**

---

## 10. 里程碑与甘特图

### 10.1 里程碑清单

| 里程碑 | 时间点 | 验收标准 |
|--------|-------|---------|
| **M0** 项目启动 | Week 0 | 团队就位、环境就绪、需求评审通过 |
| **M1** 设计冻结 | Week 3 | 架构设计文档评审通过；接口契约冻结 |
| **M2** 基础设施就绪 | Week 5 | `session()` / 验证框架 / Registry / ModuleConfigurator 基类可用 |
| **M3** ComStack 封装可用 | Week 8 | Com + CanIf + PduR + Can 封装器 + 单元测试可用 |
| **M4** DiagStack + MemStack 可用 | Week 11 | Dcm + Dem + NvM + 系统服务封装器 + 单元测试可用 |
| **M5** L3 Feature Alpha | Week 13 | CAN Builder + Diag Builder 可运行端到端场景 |
| **M6** 集成验证完成 | Week 15 | 4 个端到端场景测试通过；CI Pipeline 就绪 |
| **M7** 正式交付 | Week 17 | 文档完备、示例可运行、验收评审通过 |

### 10.2 甘特图（按周）

```
Week:  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17
       ├──┴──┴──┼──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┤
P1 需求+设计
何康  ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
刘学海 ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
庄希琦 ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
刘鹏  ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
余小林 ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
       ├──────────┼──────────────────────────────────────┤
P2 基础设施
刘鹏  ░░░░░░░░░░░░████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
余小林 ░░░░░░░░░░░░████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
庄希琦 ░░░░░░░░░░░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
       ├──────────┼─────┴────────────────────────────────┤
P3 L2 ComStack
何康  ░░░░░░░░░░░░░░░░░░████████████████████░░░░░░░░░░░░░░
       ├──────────┼─────────────────┴────────────────────┤
P4 L2 DiagStack+Mem+System
刘学海 ░░░░░░░░░░░░░░░░░░██████████████████████░░░░░░░░░░░░
       ├──────────┼─────────────────────────────┴────────┤
P5 L2 EthStack+Crypto+协同
刘鹏  ░░░░░░░░░░░░░░░░░░░░░░░░████████████████░░░░░░░░░░░░
余小林 ░░░░░░░░░░░░░░░░░░░░░░░░████████░░░░░░░░░░░░░░░░░░░░
何康  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████░░░░░░░░░░
       ├──────────┼─────────────────────────────────┴────┤
P6 L3 Feature
庄希琦 ░░░░░░░░░░░░░░░░░░░░░░░░████████████████████░░░░░░░░
       ├──────────┼──────────────────────────────────────┤
P7 测试
全员  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████
       ├──────────┼──────────────────────────────────────┤
P8 文档+CI/CD+交付
余小林 ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████
全员  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████
```

### 10.3 并行度分析

| 阶段 | 并行线程 | 关键路径 |
|------|---------|---------|
| Week 1-3 | 5 人并行需求+设计 | 需求评审完成 → 设计启动 |
| Week 3-5 | 刘鹏(Infra) ∥ 庄希琦(验证框架) ∥ 余小林(Registry) | 基础设施就绪 → L2 开发解锁 |
| Week 5-8 | 何康(ComStack) ∥ 刘学海(DiagStack) ∥ 刘鹏(EthStack) | ComStack = 关键路径（35 PD） |
| Week 8-13 | 何康(协同层) ∥ 刘学海(继续) ∥ 庄希琦(L3) ∥ 刘鹏(继续) | L3 CAN Builder 依赖 ComStack |
| Week 13-17 | 全员测试 + 庄希琦(集成测试) ∥ 余小林(CI/CD+文档) | 集成测试 → 验收 |

---

## 11. 风险管理

| # | 风险 | 概率 | 影响 | 缓解措施 |
|---|------|------|------|---------|
| R1 | **ECUC 参数定义数据不完整** | 中 | 高 | Phase 1 提前验证 `ecuc_param_def.json` 覆盖率；缺失模块优先补齐定义数据再开发封装器 |
| R2 | **L2 封装器跨模块依赖导致接口频繁变更** | 中 | 中 | 先冻结 L2 基类接口契约；各模块封装器独立开发，通过基类抽象隔离 |
| R3 | **全局状态泄漏导致测试不稳定** | 高 | 中 | 强制执行 `reinit()` teardown 规范；`session()` 上下文管理器作为推荐用法；CI 中隔离进程 |
| R4 | **AUTOSAR 版本差异导致 L2 行为不一致** | 中 | 中 | Phase 1 建立版本适配层；每个封装器声明兼容版本范围；多版本 CI 矩阵 |
| R5 | **L3 Feature 编排的模块过多导致集成复杂度爆炸** | 中 | 高 | CAN Builder 先做 MVP（最小模块集），验证通后才扩展；每个 Feature 独立可测 |
| R6 | **余小林 50% 负荷导致架构/DevOps 瓶颈** | 高 | 中 | 关键架构设计在 Phase 1 集中完成；CI/CD 模板可复用社区方案；文档工作分摊给全员 |
| R7 | **性能不达标（大模型 save 超时）** | 低 | 中 | 利用现有 XDT 增量保存；性能基准测试在 P7 阶段早期介入；必要时增加 profiling 任务 |

---

## 12. 交付物清单

### 12.1 代码交付

| 交付物 | 路径 | 说明 |
|--------|------|------|
| 会话管理器 | `autosarfactory/session.py` + 修改 `autosarfactory.py` | `with AF.session()` 上下文管理器 |
| 验证框架 | `autosarfactory/validation/` | 可插拔规则引擎 + 内置规则集 |
| 版本适配层 | `autosarfactory/adapters/` | 多 Release 适配器 |
| Definition Registry | `autosarfactory/modules/registry.py` | ECUC 参数定义索引 |
| L2 基类 | `autosarfactory/modules/base.py` | ModuleConfigurator + ContainerProxy |
| L2 ComStack | `autosarfactory/modules/com/` | 7 个通信协议栈模块封装器 |
| L2 DiagStack | `autosarfactory/modules/diag/` | 3 个诊断协议栈模块封装器 |
| L2 MemStack | `autosarfactory/modules/memory/` | 2 个存储协议栈模块封装器 |
| L2 System | `autosarfactory/modules/system/` | 3 个系统服务模块封装器 |
| L2 EthStack | `autosarfactory/modules/ethernet/` | 4 个以太网协议栈模块封装器 |
| L2 Crypto | `autosarfactory/modules/crypto/` | Csm 封装器 |
| L2 协同层 | `autosarfactory/protocols/` | ComStack / DiagStack / MemStack 协同校验 |
| L3 基类 | `autosarfactory/features/base.py` | FeatureBase + 合规闸门 |
| L3 Features | `autosarfactory/features/` | CAN Builder / Diag Builder / PDU Packing / Variant Manager |
| 单元测试 | `tests/test_l2_*.py` / `tests/test_l3_*.py` | 逐模块 + 逐 Feature 单测 |
| 集成测试 | `tests/test_integration_*.py` | 端到端场景测试 |
| CI/CD 模板 | `.github/workflows/` 或 `ci/` | 流水线配置 |
| 使用示例 | `Examples/` 扩充 | L2/L3 端到端示例脚本 |

### 12.2 文档交付

| 交付物 | 说明 |
|--------|------|
| 《L2 模块需求规格说明书》 | 14 个 BSW 模块参数约束清单 |
| 《L3 Feature 需求规格说明书》 | 4 个整车场景用例规格 |
| 《架构设计文档》（L2/L3/验证框架/版本适配） | 详细设计 |
| 《API 参考文档》 | L2/L3 公开接口文档 |
| 《开发者指南》 | 如何扩展新模块/Feature |
| 《API 契约与异常规范》 | 跨层交互契约 |
| 《企业规范沉淀指南》 | L3 Feature 开发最佳实践 |
| 《工时评估报告》（本文档） | v1.0 |

---

## 附录 A：与现有设计文档的对应关系

| 设计文档章节 | 本报告对应 Phase |
|-------------|-----------------|
| 设计思路 v0.1 §2 L1 底座 | Phase 2（基础设施：session / 验证框架 / 版本适配） |
| 设计思路 v0.1 §3 L2 封装层 | Phase 3-5（14 个 BSW 模块 + 基类 + 协同层） |
| 设计思路 v0.1 §4 L3 业务层 | Phase 6（4 个 Feature + 模板库） |
| 设计思路 v0.1 §5 MCP 服务 | 不在本报告范围（MCP 已有 14 工具，按需扩展） |
| 设计思路 v0.1 §6 IaC + CP 协同 | Phase 7（集成测试）+ Phase 8（CI/CD 模板） |
| 设计思路 v0.1 §7 CI/CD 适配 | Phase 8 |
| 设计思路 v0.1 §10 测试策略 | Phase 7 |
| 详细设计 §7 与现有架构融合 | Phase 2（包结构落地） |
| 详细设计 §8 开发路线建议 | Phase 1-8（对应其 Phase 1~3 的细化展开） |

---

## 附录 B：PD 估算方法与假设

| 假设 | 说明 |
|------|------|
| **1 PD = 8 有效工时** | 扣除会议、沟通、文档等 overhead |
| **开发效率** | 中级工程师基准，高级工程师 ×1.3，初级 ×0.7 |
| **模块复杂度** | Com/CanIf/PduR/Dcm/Dem = 复杂（含大量参数依赖）；CanTrcv/CanSM/LinIf/FiM/SoAd = 简单（参数少，逻辑直） |
| **测试 = 实现 × 0.3~0.5** | 单元测试开发量约为功能代码的 30-50% |
| **并行度损失 15%** | 5 人团队因接口协商、代码合并、技术讨论带来的效率折损 |
| **缓冲 10%** | 应对需求变更、技术预研、返工等不可预见工作 |

---

> **结论**：在现有 L1 底座之上，5 人团队（4.5 FTE）完成 L2 封装层 + L3 业务层 + L1 增强的完整开发，预计需要 **293 人天**，建议项目周期 **4 ~ 4.5 个日历月**。核心路径为 ComStack 封装 → 协同层 → L3 CAN Builder → 集成验证。
