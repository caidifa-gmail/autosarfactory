# ARXML API 开发设计思路

> 基于 AUTOSAR Integration-as-Code (IaC) 技术方案，结合 autosarfactory 项目现状的设计思路文档

**日期**：2026-06-30

**参考输入**：[AUTOSAR Integration-as-Code (IaC) 技术方案 v0.1](../cai-input/AUTOSAR%20Integration-as-Code%20(IaC)%20技术方案0.1.md)

---

## 目录

1. [背景与定位](#1-背景与定位)
2. [当前 autosarfactory 能力盘点](#2-当前-autosarfactory-能力盘点)
3. [三层 API 架构总体设计](#3-三层-api-架构总体设计)
4. [L1：底层基础 ARXML API——当前状态与增强](#4-l1底层基础-arxml-api当前状态与增强)
5. [L2：模块与协议栈 API——封装层设计](#5-l2模块与协议栈-api封装层设计)
6. [L3：高阶 Feature API——业务层设计](#6-l3高阶-feature-api业务层设计)
7. [与现有架构的融合方案](#7-与现有架构的融合方案)
8. [开发路线建议](#8-开发路线建议)
9. [风险与应对](#9-风险与应对)
10. [总结](#10-总结)

---

## 1. 背景与定位

### 1.1 行业背景

软件定义汽车时代，AUTOSAR 配置集成长期依赖 GUI 手动操作。传统模式存在四大痛点：

| 痛点 | 表现 |
|------|------|
| **效率瓶颈** | 手动配置繁复，无法应对多车型、多变体指数级增长 |
| **质量黑洞** | 集成逻辑不可追溯，人工操作易出错 |
| **知识流失** | 经验仅存于个人脑中，人员流动即流失 |
| **流程割裂** | 难以对接 CI/CD，开发-测试-部署脱节 |

IaC 技术方案提出：将集成从"界面手动点击"转向"代码编程描述"，实现可编程、可复用、自动化的整车集成。

### 1.2 autosarfactory 的战略定位

autosarfactory 已在 **L1 层（底层基础 ARXML API）** 形成了成熟的能力底座：

- 支持最新 AUTOSAR R24-11 及十余个历史版本
- 全元模型元素覆盖（~440k 行生成的类定义）
- 完备的读/写/改/保存/导出能力
- 多文件合并与路径导航
- 增量脏跟踪保存
- 图形化模型浏览器（autosar_ui）
- MCP 服务器（AI 代理辅助开发）

**核心命题**：如何在 L1 底座之上构建 L2（模块/协议栈封装）和 L3（高阶业务 Feature）层，形成完整的三层 IaC 能力。

---

## 2. 当前 autosarfactory 能力盘点

### 2.1 已具备的核心能力（L1 完备）

```
┌──────────────────────────────────────────────────┐
│              autosarfactory L1 API               │
├──────────────────────────────────────────────────┤
│ autosarfactory.read(files/folders)     读取/合并  │
│ autosarfactory.new_file(path, pkg)     创建文件   │
│ autosarfactory.save([files])           增量保存   │
│ autosarfactory.saveAs(path)            全量另存   │
│ autosarfactory.get_node(path)          路径查找   │
│ autosarfactory.get_all_instances(cls)  实例遍历   │
│ autosarfactory.export_to_file(...)     元素导出   │
│ autosarfactory.reinit()                状态重置   │
├──────────────────────────────────────────────────┤
│ node.get_<attr>()       属性读取                 │
│ node.set_<attr>(val)    属性设置                 │
│ node.add_<ref>(obj)     多值引用添加             │
│ node.remove_<ref>(obj)  多值引用移除             │
│ node.new_<Element>(...) 子元素创建               │
│ node.get_children()     子元素列表               │
│ node.get_property_values() 属性值列表            │
├──────────────────────────────────────────────────┤
│ _add_or_get_xml_node()     Schema 顺序插入       │
│ _insert_element_after_given_tags()  位置控制     │
│ XDT.mark_dirty()            脏节点追踪           │
└──────────────────────────────────────────────────┘
```

### 2.2 架构约束（来自 AGENTS.md 和当前实现）

| 约束 | 说明 | 对上层设计的影响 |
|------|------|-----------------|
| **全局状态模式** | `read()` 累积合并，`reinit()` 清空 | L2/L3 必须遵守 reinit 边界 |
| **代码生成主导** | ~440k 行类体为生成代码，不可手改 | L2/L3 在生成类之外构建 |
| **lxml 底层** | 每个模型节点封装 lxml element | L2/L3 通过公共 API 访问，不绕开 |
| **Schema 顺序约束** | 子元素顺序由 AUTOSAR XSD 决定 | L2/L3 使用现有插入方法 |
| **脏跟踪优化** | `XDT.mark_dirty` + `referenced_by` 回边 | L2/L3 的修改路径需确保脏标记传播 |
| **多版本兼容** | 主包 R24-11 + 历史快照 10 个版本 | L2/L3 可能需要版本适配层 |

### 2.3 能力缺口分析

| 层次 | 现有能力 | 缺失能力 |
|------|---------|---------|
| **L1 底层** | ✅ 完备 | 可增强：验证规则引擎、版本差异适配 |
| **L2 封装** | ❌ 无 | 需要从零构建：BSW 模块、协议栈约束封装 |
| **L3 业务** | ❌ 无 | 需要从零构建：跨模块整车集成场景 |

---

## 3. 三层 API 架构总体设计

### 3.1 架构全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                       用户脚本 / CI/CD Pipeline                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  L3: 高阶 Feature API (业务层)                            │  │
│  │  autosarfactory.features / autosarfactory.integration     │  │
│  │                                                            │  │
│  │  整车级场景封装：                                          │  │
│  │  - CanCommunicationBuilder    CAN 通信矩阵一键生成         │  │
│  │  - DiagnosticConfigurator     诊断路由配置                │  │
│  │  - VariantManager             软件变体管理                │  │
│  │  - SecurityConfigurator       SecOC 安全配置              │  │
│  │  - EcuExtractMerger           ECU Extract 合并            │  │
│  │  - SignalMappingBuilder       信号映射编排               │  │
│  │                                                            │  │
│  │  特点：跨模块联动、企业规范沉淀、业务语义明确             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │ 依赖                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  L2: 模块与协议栈 API (封装层)                            │  │
│  │  autosarfactory.modules / autosarfactory.protocols        │  │
│  │                                                            │  │
│  │  单模块封装：                                              │  │
│  │  - ComStack: Com / ComM / PduR / CanIf / CanTrcv ...     │  │
│  │  - DiagStack: Dcm / Dem / Fim / NvM ...                  │  │
│  │  - MemStack: NvM / Fee / Ea / Fls ...                    │  │
│  │  - SysServices: BswM / EcuM / WdgM / SchM ...            │  │
│  │  - CryptoStack: CryIf / Crypto / Csm ...                 │  │
│  │  - EthStack: EthIf / TcpIp / SoAd / DoIp ...             │  │
│  │                                                            │  │
│  │  特点：参数约束封装、内置校验、屏蔽 ARXML 细节            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │ 依赖                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  L1: 底层基础 ARXML API (元模型层) — autosarfactory 已有  │  │
│  │  autosarfactory (当前)                                     │  │
│  │                                                            │  │
│  │  原子操作：read / save / get_node / new_<Element> / ...   │  │
│  │  全 AUTOSAR 元模型覆盖 (~440k 行生成代码)                  │  │
│  │  多版本兼容 (R4.0.3 ~ R24-11)                              │  │
│  │                                                            │  │
│  │  特点：无业务语义、纯 ARXML 语法级操作                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 分层职责矩阵

| 维度 | L1 底层 | L2 封装层 | L3 业务层 |
|------|---------|----------|----------|
| **操作粒度** | 元模型原子操作 | 单模块参数配置 | 跨模块整车场景 |
| **业务语义** | 无 | 模块级约束 | 整车级业务流程 |
| **跨模块联动** | 不支持 | 不支持 | 核心能力 |
| **ARXML 暴露** | 完全暴露 | 大量屏蔽 | 完全屏蔽 |
| **参数校验** | Schema 级 | 模块级 + 业务规则 | 跨模块一致性 |
| **复用范围** | 通用 | 同模块跨项目 | 跨车型跨项目 |
| **版本兼容策略** | 基础兼容 | 模块标准版本适配 | 项目基线适配 |
| **目标用户** | 工具开发者 | 集成工程师 | 系统工程师 |

---

## 4. L1：底层基础 ARXML API——当前状态与增强

### 4.1 当前成熟度评估

**成熟度：★★★★☆（完备但可增强）**

autosarfactory 的 L1 层已经相当成熟。以下是保持现状和可增强的方面：

#### 保持现状的部分

- `read()` / `save()` / `saveAs()` 文件 I/O 管线
- `get_<attr>` / `set_<attr>` 属性访问
- `new_<Element>()` 工厂方法体系
- `get_node()` / `get_all_instances()` 查找遍历
- `XDT` 脏跟踪与增量保存
- lxml 底层 + Schema 顺序插入

#### 建议增强的部分

**a) 上下文管理器支持**

当前 `reinit()` 需要手动调用，容易遗漏导致状态泄漏。建议增加上下文管理器：

```python
# 方案：增加 with 语句支持
with autosarfactory.session():
    autosarfactory.read(['components.arxml'])
    swc = autosarfactory.get_node('/Swcs/asw1')
    swc.set_CATEGORY('APPLICATION')
    autosarfactory.save()
# 退出时自动 reinit()
```

**b) 验证规则引擎**

当前无内置验证，仅在 lxml 解析阶段有 Schema 校验。建议增加可插拔的验证框架：

```python
# 增强方向
autosarfactory.validate(
    files=['output.arxml'],
    rules=['schema', 'reference_integrity', 'naming_convention']
)
```

**c) 查询 DSL**

当前 `get_node()` 仅支持精确路径。对于复杂查询场景不够灵活。可考虑引入简单查询表达式：

```python
# 增强方向
results = autosarfactory.query(
    type='ApplicationSwComponentType',
    filter={'SHORT-NAME': r'Swc*'},
    package='/Swcs'
)
```

**d) 版本适配器**

当前多版本通过快照目录隔离。可增加版本适配层，使同一份上层代码在不同 AUTOSAR 版本间平滑切换：

```python
# 增强方向
autosarfactory.set_target_release('r24-11')
# 或
autosarfactory.set_target_release('r4.2.2')
```

### 4.2 L1 增强的实现优先级

| 增强项 | 优先级 | 理由 |
|--------|--------|------|
| 上下文管理器 `session()` | P0 | 直接影响 L2/L3 代码健壮性 |
| 版本适配器 | P1 | L2/L3 多版本兼容的基础设施 |
| 验证规则引擎 | P1 | 提升交付质量，L2/L3 可复用 |
| 查询 DSL | P2 | 锦上添花，不影响核心流程 |

---

## 5. L2：模块与协议栈 API——封装层设计

### 5.1 设计目标

L2 层是连接底层 ARXML 原子操作与上层业务场景的关键桥梁。核心目标：

- **屏蔽 ARXML 细节**：集成工程师无需关心 `EcucContainerValue`、`EcucModuleConfigurationValues` 等底层结构
- **封装参数约束**：将 AUTOSAR 规范中的参数依赖、取值范围、互斥规则以代码形式固化
- **提供声明式接口**：通过有意义的方法名和参数传递模块配置意图

### 5.2 包结构设计

```
autosarfactory/
├── autosarfactory.py          # L1: 现有，保持不变
├── datatype_utils.py          # L1: 现有
├── XmlElementDirtyTracker.py  # L1: 现有
├── autosar_ui.py              # L1: 现有
│
├── modules/                   # L2: 新增——模块封装
│   ├── __init__.py
│   ├── base.py                # L2 基类 ModuleConfigurator
│   ├── com/                   # 通信协议栈
│   │   ├── __init__.py
│   │   ├── com.py             # Com 模块
│   │   ├── comm.py            # ComM 模块
│   │   ├── pdur.py            # PduR 模块
│   │   ├── canif.py           # CanIf 模块
│   │   ├── cantrcv.py         # CanTrcv 模块
│   │   ├── cansm.py           # CanSM 模块
│   │   └── linif.py           # LinIf 模块
│   ├── diag/                  # 诊断协议栈
│   │   ├── __init__.py
│   │   ├── dcm.py             # Dcm 模块
│   │   ├── dem.py             # Dem 模块
│   │   └── fim.py             # FiM 模块
│   ├── memory/                # 存储协议栈
│   │   ├── __init__.py
│   │   ├── nvm.py             # NvM 模块
│   │   ├── fee.py             # Fee 模块
│   │   └── ea.py              # Ea 模块
│   ├── system/                # 系统服务
│   │   ├── __init__.py
│   │   ├── bswm.py            # BswM 模块
│   │   ├── ecum.py            # EcuM 模块
│   │   └── wdgm.py            # WdgM 模块
│   ├── crypto/                # 加密协议栈
│   │   ├── __init__.py
│   │   └── csm.py             # Csm 模块
│   └── ethernet/              # 以太网协议栈
│       ├── __init__.py
│       ├── tcpip.py           # TcpIp 模块
│       ├── soad.py            # SoAd 模块
│       └── doip.py            # DoIp 模块
│
├── protocols/                 # L2: 新增——协议栈级协同
│   ├── __init__.py
│   ├── com_stack.py           # 完整 ComStack 协同配置
│   ├── diag_stack.py          # 完整 DiagStack 协同配置
│   └── mem_stack.py           # 完整 MemStack 协同配置
│
└── features/                  # L3: 新增——业务 Feature
    ├── __init__.py
    ├── can_builder.py         # CAN 通信矩阵构建
    ├── diag_builder.py        # 诊断配置构建
    └── variant_manager.py     # 变体管理
```

### 5.3 L2 基类设计

```python
# modules/base.py —— 所有 L2 模块封装器的基类

class ModuleConfigurator:
    """
    L2 模块封装器基类。
    
    职责：
    - 封装单个 BSW 模块的 ECUC 参数配置
    - 提供声明式 get/set 接口
    - 内置参数约束校验
    - 管理容器层级导航
    
    子类需要实现：
    - module_name: 模块短名称 (e.g. "Com", "CanIf")
    - module_def_ref: 模块定义引用路径
    - _build_containers(): 构建内部容器树
    """
    
    module_name: str
    module_def_ref: str
    
    def __init__(self, ar_package):
        self._package = ar_package
        self._module_config = None
        self._containers = {}
        self._locate_or_create_module()
    
    def _locate_or_create_module(self):
        """定位已有模块配置，若不存在则创建"""
        pass
    
    # --- 容器导航 ---
    def container(self, path: str) -> 'ContainerProxy':
        """通过路径获取容器代理对象"""
        pass
    
    # --- 参数读写 ---
    def get_param(self, container_path: str, param_name: str):
        """读取指定容器的参数值"""
        pass
    
    def set_param(self, container_path: str, param_name: str, value):
        """设置指定容器的参数值，含校验"""
        pass
    
    # --- 子容器管理 ---
    def add_container(self, parent_path: str, container_name: str, 
                      def_ref: str):
        """添加子容器（如 ComIpdu, CanIfHoh）"""
        pass
    
    def remove_container(self, container_path: str):
        """移除子容器"""
        pass
    
    # --- 引用管理 ---
    def add_reference(self, container_path: str, ref_name: str, target):
        """添加配置引用（如 PduR 路由目标）"""
        pass
    
    # --- 校验 ---
    def validate(self) -> list:
        """运行模块级校验，返回问题列表"""
        pass
    
    # --- 导入导出 ---
    def export_summary(self) -> dict:
        """导出模块配置摘要（JSON 友好）"""
        pass
```

### 5.4 典型 L2 模块设计示例：CanIf 封装器

```python
# modules/com/canif.py

class CanIfConfigurator(ModuleConfigurator):
    """
    CanIf (CAN Interface) 模块配置器。
    
    封装 CanIf 模块的标准配置模式：
    - HOH (Hardware Object Handle) 管理
    - PDU 通道配置
    - Tx/Rx 处理模式
    """
    
    module_name = "CanIf"
    module_def_ref = "/AUTOSAR/EcucDefs/CanIf"
    
    def add_rx_pdu(self, can_id: int, can_id_mask: int,
                   lower_layer: str, upper_layer: str) -> str:
        """
        添加接收 PDU。
        自动创建配套的 HOH、缓冲配置。
        返回 PDU 容器路径。
        """
        pass
    
    def add_tx_pdu(self, can_id: int, can_id_mask: int,
                   lower_layer: str, upper_layer: str) -> str:
        """添加发送 PDU"""
        pass
    
    def configure_channel(self, channel_id: int, baudrate: int):
        """配置 CAN 通道参数"""
        pass
    
    def set_hoh_properties(self, hoh_id: int, 
                           can_object_type: str, ...):
        """设置 HOH 属性"""
        pass
```

### 5.5 L2 设计原则

| 原则 | 说明 |
|------|------|
| **单一模块职责** | 每个封装器仅对应一个 BSW 模块 |
| **声明式接口** | 方法名表达业务意图（如 `add_rx_pdu` 而非 `create_ecuc_container`） |
| **约束内聚** | 模块内部的参数依赖、取值范围在封装器内部校验 |
| **不跨模块** | L2 层不做跨模块联动，那是 L3 的职责 |
| **版本感知** | 封装器应声明支持的 AUTOSAR 版本范围 |
| **可测试** | 每个封装器应可独立单元测试 |

---

## 6. L3：高阶 Feature API——业务层设计

### 6.1 设计目标

L3 层是 IaC 的核心价值载体。它将复杂的跨模块整车集成逻辑封装为业务场景函数，直接对齐整车功能需求。

核心理念：

> 系统工程师用一句代码表达一个整车功能需求，背后自动完成数十个模块的协同配置。

### 6.2 典型 Feature 场景设计

#### 6.2.1 CAN 通信矩阵构建器

```python
# features/can_builder.py

class CanCommunicationBuilder:
    """
    CAN 通信矩阵构建器。
    
    输入：高层通信需求描述
    输出：完整的 Com/CanIf/CanTrcv/PduR/CanSM/EcuC 配置
    
    使用示例：
        builder = CanCommunicationBuilder(ar_package)
        builder.add_ecu('EngineECU', ['CAN1', 'CAN2'])
        builder.add_signal('EngineSpeed', src='EngineECU', 
                           dst=['BcmECU', 'IpcECU'], 
                           dlc=8, cycle_time_ms=10)
        builder.build()
    """
    
    def __init__(self, ar_package):
        self._package = ar_package
        self._ecus = {}
        self._isignals = []
        self._ipdus = []
        self._can_clusters = {}
    
    # --- 声明式输入接口 ---
    
    def define_can_cluster(self, name: str, baudrate: int,
                           protocol_type='CAN_FD'):
        """定义一个 CAN 集群"""
        pass
    
    def add_ecu(self, name: str, connected_clusters: list):
        """
        注册一个 ECU 节点，自动创建：
        - EcuInstance
        - ComGwMapping（若为网关）
        - CanIf 通道配置
        """
        pass
    
    def add_pdu(self, name: str, id: int, dlc: int, 
                src_ecu: str, dst_ecus: list,
                cycle_time_ms: int = 0,
                trigger_type: str = 'PERIODIC'):
        """
        添加一个 PDU（含 I-PDU + N-PDU），自动：
        - 创建 ISignal → ISignalIPdu 映射
        - 创建 ISignalIPdu → NPdu 映射
        - 配置 Com IPdu / Com IPduGroup
        - 配置 PduR 路由路径
        - 配置 CanIf HOH / CanIf TxPdu
        """
        pass
    
    def add_signal(self, name: str, start_bit: int, length: int,
                   data_type, pdu_name: str, 
                   init_value=None, 
                   endianness='MOTOROLA'):
        """添加信号，关联到已有 PDU"""
        pass
    
    # --- 构建接口 ---
    
    def build(self, target_files=None):
        """
        执行构建：
        1. 创建所有 ArPackage 层级
        2. 递归创建 EcuInstance / Cluster / Pdu / Signal
        3. 跨模块一致性校验
        4. 输出 ARXML
        """
        pass
    
    # --- 校验 ---
    
    def validate(self) -> 'ValidationReport':
        """
        跨模块一致性校验：
        - PduR 路由路径完整性
        - Com 信号长度与 PDU DLC 一致性
        - CanIf 配置与 Can 驱动通道匹配性
        - 信号不重叠校验
        """
        pass
```

#### 6.2.2 诊断配置构建器

```python
# features/diag_builder.py

class DiagnosticConfigurator:
    """
    诊断配置构建器。
    
    场景：
    - 诊断服务 (Dcm) 配置
    - 诊断事件 (Dem) 定义与 Fim 映射
    - NvM 故障存储块
    - DoIp 传输层映射
    
    使用示例：
        diag = DiagnosticConfigurator(ar_package)
        diag.configure_ecu('EngineECU', 
                           diag_address=0x0A,
                           transport='CAN')
        diag.add_dtc('P0101', 'MAF_Circuit_Range_Performance',
                     severity='CRITICAL',
                     debounce_ms=1000,
                     aging_counter=True)
        diag.add_did(0xF100, 'EngineSpeed', 
                     data_type='uint16',
                     read_scaling=0.25)
        diag.build()
    """
    
    def __init__(self, ar_package):
        pass
    
    def configure_ecu(self, ecu_name: str,
                      diag_address: int,
                      transport: str = 'CAN'):
        """
        配置 ECU 诊断基础参数：
        - Dcm: 诊断地址、会话控制、安全访问
        - Dem: 事件存储阈值
        - DoIp: 诊断传输层配置
        """
        pass
    
    def add_dtc(self, dtc_code: str, description: str,
                severity: str = 'WARNING',
                debounce_ms: int = 0,
                aging_counter: bool = False,
                enable_record: bool = False):
        """添加诊断故障码，自动创建 DemEvent + Fim 映射"""
        pass
    
    def add_did(self, did: int, name: str,
                data_type: str,
                read_scaling: float = 1.0,
                read_offset: float = 0.0):
        """添加诊断标识符 (DID)"""
        pass
    
    def add_routine(self, rid: int, name: str, ...):
        """添加诊断例程控制 (RID)"""
        pass
    
    def build(self):
        """执行构建"""
        pass
```

#### 6.2.3 软件变体管理器

```python
# features/variant_manager.py

class VariantManager:
    """
    软件变体管理器。
    
    场景：
    - 基于 Post-Build / Pre-Compile 变体管理
    - 变体条件定义与关联
    - 多车型共享部件的条件化配置
    """
    
    def define_variant(self, name: str, binding_time='POST_BUILD'):
        """定义一个变体条件"""
        pass
    
    def add_variant_condition(self, element, variant_name: str):
        """为元素附加变体条件"""
        pass
    
    def extract_variant(self, variant_name: str, output_file: str):
        """
        提取指定变体的完整配置子集。
        基于变体条件剪枝 ARXML 树。
        """
        pass
```

### 6.3 L3 设计原则

| 原则 | 说明 |
|------|------|
| **业务语义优先** | 接口名称 = 业务术语，如 `add_dtc`、`add_signal` |
| **一次调用，多处生效** | 一个方法自动完成跨模块联动配置 |
| **输入极简** | 最小化必填参数，其余使用合理默认值 |
| **构建模式** | 声明式积累输入 → 调用 `build()` 一次性生成 |
| **可组合** | Feature 之间可以相互引用和组合 |
| **企业规范可插拔** | 通过策略模式注入企业特定规则 |
| **输出可校验** | 每个 Feature 自带 `validate()` |

---

## 7. 与现有架构的融合方案

### 7.1 模块组织原则

```
autosarfactory/
│
├── autosarfactory.py          ★ L1: 保持现有不变
├── datatype_utils.py          ★ L1: 保持现有不变
├── XmlElementDirtyTracker.py  ★ L1: 保持现有不变
│
├── modules/        ← 新增    ★ L2: 模块封装
├── protocols/      ← 新增    ★ L2: 协议栈协同
├── features/       ← 新增    ★ L3: 业务 Feature
│
├── validation/     ← 新增    ★ 跨层验证框架
│   ├── __init__.py
│   ├── rules.py             # 验证规则基类
│   ├── arxml_rules.py       # ARXML Schema 级规则
│   ├── module_rules.py      # 模块级约束规则
│   └── integration_rules.py # 跨模块一致性规则
│
├── adapters/       ← 新增    ★ 版本适配层
│   ├── __init__.py
│   ├── base.py              # 适配器基类
│   ├── r403.py              # R4.0.3 适配
│   ├── r422.py              # R4.2.2 适配
│   └── r2411.py             # R24-11 适配 (默认)
│
└── session.py      ← 新增    ★ 会话管理（上下文管理器）
```

### 7.2 依赖关系约束

```
L3 (features/)    →  依赖  →  L2 (modules/ + protocols/)
                   →  可访问 →  L1 (autosarfactory)
                   →  不应   →  直接操纵 lxml

L2 (modules/)     →  依赖  →  L1 (autosarfactory)
                   →  可访问 →  validate/
                   →  不应   →  直接操纵 lxml

L1 (现有)          →  独立，保持不变
```

### 7.3 与全局状态模式的兼容

autosarfactory 采用全局状态模式（`read()` 累积合并，`reinit()` 清空）。L2/L3 的设计必须与之兼容：

```python
# 推荐的用户代码模式

from autosarfactory import autosarfactory
from autosarfactory.features import CanCommunicationBuilder

# 1. 初始化全局状态
autosarfactory.reinit()
autosarfactory.read(['input1.arxml', 'input2.arxml'])

# 2. 获取 AR-Package（L1 操作）
root_pkg = autosarfactory.get_node('/RootPackage')

# 3. 使用 L3 Feature（内部使用 L2 模块）
builder = CanCommunicationBuilder(root_pkg)
builder.add_ecu('EngineECU', ['CAN1'])
builder.add_signal('EngineSpeed', ...)
builder.build()

# 4. 保存
autosarfactory.save(['output.arxml'])

# 5. 清理
autosarfactory.reinit()
```

或者使用上下文管理器（增强后）：

```python
with autosarfactory.session():
    autosarfactory.read(['input.arxml'])
    builder = CanCommunicationBuilder(
        autosarfactory.get_node('/RootPackage')
    )
    builder.add_ecu(...).add_signal(...).build()
    autosarfactory.save()
    # 自动 reinit()
```

### 7.4 对现有公共 API 的影响

**零破坏性变更**：新增的 `modules/`、`features/` 等包作为独立命名空间，不影响现有 `autosarfactory.*` 公共 API。

用户可选择：
- 继续使用 L1 API（向后兼容）
- 渐进式采用 L2/L3 API
- 混合使用（Feature 脚本中夹杂 L1 微调）

---

## 8. 开发路线建议

### 8.1 三阶段路线图

```
Phase 1 (2-3 个月)              Phase 2 (3-6 个月)              Phase 3 (持续演进)
┌─────────────────────┐     ┌─────────────────────────┐     ┌──────────────────────┐
│ 基础设施 + MVP      │     │ L2 协议栈覆盖 + L3 试点  │     │ 生态完善 + AI 集成   │
│                     │     │                         │     │                      │
│ ▢ session 上下文管  │     │ ▢ ComStack 全栈模块     │     │ ▢ 企业规范模板库    │
│   理器              │  ─► │ ▢ DiagStack 全栈模块    │  ─► │ ▢ AI Agent 集成      │
│ ▢ 验证框架         │     │ ▢ MemStack 全栈模块     │     │ ▢ Asset Library     │
│ ▢ 版本适配层       │     │ ▢ L3: CAN Builder MVP   │     │ ▢ 跨版本迁移工具    │
│ ▢ L2: CanIf/Com    │     │ ▢ L3: Diag Builder MVP  │     │ ▢ CI/CD 模板        │
│   封装器 MVP       │     │ ▢ L3: Variant Manager   │     │ ▢ MCP 增强知识库    │
│ ▢ L2: PduR 封装器  │     │ ▢ 集成测试套件         │     │                      │
└─────────────────────┘     └─────────────────────────┘     └──────────────────────┘
```

### 8.2 Phase 1 详细任务

| 任务 | 说明 | 优先级 |
|------|------|--------|
| **session 上下文管理器** | `with autosarfactory.session()` 语法，自动 reinit | P0 |
| **验证框架基座** | `validation/` 包，规则基类和注册机制 | P0 |
| **版本适配层** | `adapters/` 包，多版本 API 差异封装 | P0 |
| **L2 基类** | `ModuleConfigurator` 基类，容器导航、参数读写 | P0 |
| **L2 CanIf 封装器** | CanIf 模块 HOH/PDU 通道配置 | P1 |
| **L2 Com 封装器** | Com 模块 IPdu/IPduGroup/信号配置 | P1 |
| **L2 PduR 封装器** | PduR 路由路径配置 | P1 |
| **L3 CAN Builder Alpha** | 简单 CAN 通信矩阵构建 | P2 |
| **单元测试框架** | L2/L3 层独立测试能力 | P1 |
| **示例工程更新** | 新增 L2/L3 使用示例 | P2 |

### 8.3 技术选型决策

| 决策点 | 选型 | 理由 |
|--------|------|------|
| **L2/L3 实现语言** | Python 3.10+（与项目一致） | 无需引入新运行时 |
| **验证规则 DSL** | Python 原生类 + 装饰器 | 简单直观，学习成本低 |
| **版本适配策略** | 适配器模式（非继承） | 避免版本差异污染核心逻辑 |
| **测试框架** | pytest（与项目一致） | 统一工具链 |
| **类型标注** | 完整的 type hints | 提升 IDE 支持和可维护性 |

---

## 9. 风险与应对

| 风险 | 影响 | 应对策略 |
|------|------|---------|
| **ARXML 版本碎片化** | L2/L3 在不同版本间行为不一致 | Phase 1 即构建版本适配层；每个封装器声明兼容版本范围 |
| **全局状态泄漏** | L2/L3 多步骤操作间状态污染 | Phase 1 优先实现 `session()` 上下文管理器 |
| **性能退化** | L3 批量构建场景耗时超标 | 利用现有 XDT 增量保存；大型构建考虑分片 |
| **L2 封装覆盖不足** | 用户仍需绕过 L2 直接操作 L1 | 明确的 L2 未覆盖降级路径；接受渐进覆盖策略 |
| **MCP 服务器与 L2/L3 脱节** | AI 代理无法感知新 API | 同步更新 `af_api_reference.json`，扩展 MCP 知识库 |
| **测试资源不足** | 缺少 BSW 模块标准 ARXML 样本 | Phase 1 创建足量测试 fixture；与示例工程保持一致 |

---

## 10. 总结

autosarfactory 已经在 L1（底层 ARXML API）层具备了成熟的能力底座，这为向上构建 L2（模块/协议栈封装）和 L3（高阶业务 Feature）层提供了坚实的基础。

### 设计核心要点

1. **分层清晰、职责明确**
   - L1 做原子操作，不掺业务逻辑
   - L2 做单模块封装，不跨模块联动
   - L3 做整车场景编排，屏蔽底层细节

2. **渐进增强、向后兼容**
   - 现有 L1 公共 API 零破坏
   - L2/L3 作为独立命名空间叠加
   - 用户可选择使用层次

3. **基础设施先行**
   - 上下文管理器解决状态管理痛点
   - 验证框架提升质量保障
   - 版本适配层护航多版本兼容

4. **以场景驱动 L2/L3 开发**
   - 从 CAN 通信和诊断两个最高频场景切入
   - 用场景需求反推 L2 封装器优先级
   - 每个 Feature 可独立验证价值

### 下一步行动建议

1. **立即启动**：Phase 1 基础设施（session、验证框架、版本适配器）
2. **30 天内产出**：CanIf + Com + PduR 封装器 MVP + 可运行的 CAN Builder Alpha
3. **60 天内产出**：DiagStack 封装器 + Diagnostic Builder Alpha
4. **90 天内评审**：评估 L2/L3 覆盖度，规划 Phase 2 优先级

---

> **设计理念**：将车载软件集成从"界面点击"变为"代码编程"，从"个人经验"变为"企业资产"，从"手动操作"变为"自动流水线"。autosarfactory 的三层 API 体系正是实现这一 IaC 愿景的工程基石。
