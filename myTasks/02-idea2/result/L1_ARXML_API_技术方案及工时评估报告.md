# AutosarFactory L1 基础 ARXML API — 技术方案及工时评估报告

> **版本**：v1.0  
> **日期**：2026-07-07  
> **依据**：《ARXML_API_开发设计思路_v0.1.md》§2 L1 底座层 + 《ARXML_API_开发设计思路.md》（详细版）§4 + AutosarFactory 现有代码库分析  
> **假设**：完全从零开发 AutosarFactory L1 层（不含 L2/L3），包括 XSD 代码生成器、核心运行时、脏跟踪器、UI 可视化器、多版本快照等  
> **团队**：2 人（何康、刘鹏），均 100% 负荷 → 等效 **2.0 FTE**

---

## 目录

1. [项目概述与范围](#1-项目概述与范围)
2. [L1 架构总览](#2-l1-架构总览)
3. [任务拆解全景（WBS）](#3-任务拆解全景wbs)
4. [子系统 A：XSD 代码生成器](#4-子系统-axsd-代码生成器)
5. [子系统 B：核心运行时](#5-子系统-b核心运行时)
6. [子系统 C：XmlElementDirtyTracker 脏跟踪器](#6-子系统-cxmlelementdirtytracker-脏跟踪器)
7. [子系统 D：数据类型工具](#7-子系统-d数据类型工具)
8. [子系统 E：UI 可视化器](#8-子系统-eui-可视化器)
9. [子系统 F：多版本 Release 快照](#9-子系统-f多版本-release-快照)
10. [子系统 G：测试体系](#10-子系统-g测试体系)
11. [子系统 H：文档与项目管理](#11-子系统-h文档与项目管理)
12. [工时汇总与人力资源排布](#12-工时汇总与人力资源排布)
13. [里程碑与甘特图](#13-里程碑与甘特图)
14. [风险管理](#14-风险管理)
15. [交付物清单](#15-交付物清单)

---

## 1. 项目概述与范围

### 1.1 项目目标

从零构建 AutosarFactory 的 **L1 基础 ARXML API 底座层**。L1 是整个三层 IaC 架构的地基（L2/L3 在其上构建），它必须提供：

| 能力域 | 描述 |
|--------|------|
| **ARXML 解析与生成** | 读取 AUTOSAR 标准 `.arxml` 文件，解析为内存对象模型；生成/修改/新增后写回符合 XSD 的 XML |
| **完整元模型覆盖** | 覆盖 AUTOSAR R4.0 Schema（`AUTOSAR_00053`）中所有元模型类（~250+ 类），每个类提供一致的 Python API |
| **元模型访问 API** | `get_<attr>` / `set_<attr>`（单值）、`get_<attr>s` / `add_<attr>` / `remove_<attr>`（多值）、`new_<Element>(name)` 工厂方法 |
| **引用解析** | 跨文件引用自动解析、双向引用维护（`referenced_by`）、`ReferenceBase` 捷径 |
| **Schema 顺序保证** | XML 子元素写入顺序始终符合 AUTOSAR XSD 序列定义，与 Python 调用顺序无关 |
| **增量脏保存** | 仅序列化已变更（dirty）的子树，而非每次全量写入 |
| **多文件合并** | 多个 `.arxml` 文件累积读入一棵内存树；支持跨文件引用；可分裂（splitable）元素自动合并 |
| **多 Release 兼容** | 支持 AUTOSAR 多版本（R4.0.3 ~ R24-11），通过冻结快照的方式独立建模 |

### 1.2 不在本报告范围

- L2 模块/协议栈封装层（BSW Configurator）
- L3 高阶 Feature 业务层（CAN Builder、Diag Builder 等）
- MCP 服务器（AI 代理工具链）
- ECUC 参数定义数据库构建
- AUTOSAR 规范知识库（KB）构建

---

## 2. L1 架构总览

### 2.1 系统构成

```
┌──────────────────────────────────────────────────────────────────┐
│                     L1 AutosarFactory 底座                         │
│                                                                   │
│  ┌───────────────────┐  ┌──────────────────────────────────────┐ │
│  │ A. XSD Code Gen   │  │ B. Core Runtime (autosarfactory.py)   │ │
│  │ (生成器,一次运行)   │  │                                      │ │
│  │                    │  │  ┌──────────────────────────────┐    │ │
│  │ AUTOSAR XSD ──►   │  │  │ Hand-written Base Classes     │    │ │
│  │   ~250+ Python    │  │  │ AutosarNode → ARObject →      │    │ │
│  │   类 (~440k 行)    │  │  │ Referrable → ... → AUTOSAR    │    │ │
│  │                    │  │  │ (~2000 行)                    │    │ │
│  └───────────────────┘  │  └──────────────────────────────┘    │ │
│                          │  ┌──────────────────────────────┐    │ │
│                          │  │ Generated Classes (~438k 行)  │    │ │
│                          │  │ ~250+ ARXML meta-classes      │    │ │
│                          │  │ 每个类: get/set/new/add/remove │    │ │
│                          │  └──────────────────────────────┘    │ │
│                          │  ┌──────────────────────────────┐    │ │
│                          │  │ Module-level Public API        │    │ │
│                          │  │ read/save/get_node/new_file/   │    │ │
│                          │  │ saveAs/reinit/export_to_file   │    │ │
│                          │  └──────────────────────────────┘    │ │
│                          └──────────────────────────────────────┘ │
│  ┌───────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │ C. Dirty Tracker  │  │ D. datatype_utils │  │ E. autosar_ui │  │
│  │ (XmlElementDirty  │  │ (数据类型解析)     │  │ (Tk 可视化器)  │  │
│  │  Tracker.py)      │  │                    │  │               │  │
│  └───────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ F. Multi-Release Snapshots (autosar_releases/)              │   │
│  │    autosar422/ (R4.2.2)  autosar453/ (R24-11)  ...        │   │
│  └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心设计决策

| 决策点 | 选型 | 理由 |
|--------|------|------|
| XML 操作库 | **lxml** (`etree`) | 高性能、完整 XPath 支持、C 级速度 |
| 代码生成语言 | **Python 3.10+** | 与项目一致；代码即输出，无需编译 |
| 代码生成输入 | **AUTOSAR XSD** 文件 | 标准 Schema 定义，所有元模型信息已包含 |
| 全局状态模式 | **模块级单例** | 简单直观；CI 并发通过多进程隔离 |
| 脏跟踪策略 | **祖先链 + 反向引用边** | 已验证的可行方案，确保增量保存不丢引用 |
| UI 框架 | **Tkinter + sv_ttk** | Python 内置，零额外依赖，跨平台 |

### 2.3 类继承体系

```
AutosarNode                          # 所有节点的根基类
  ├── ARObject                       # 带 XML 元素包装的对象
  │   ├── Describable               # 带描述的抽象基类
  │   └── ReferenceBase             # 引用捷径基类
  ├── Referrable                     # 有短名称的元素
  │   ├── HwDescriptionEntity
  │   ├── MultilanguageReferrable
  │   │   ├── Identifiable          # 有 UUID 的可标识元素
  │   │   │   ├── CollectableElement  # 可被收集的元素
  │   │   │   │   ├── PackageableElement
  │   │   │   │   │   └── ARElement  # 最上层业务元素基类
  │   │   │   │   └── ARPackage     # 容器节点（核心）
  │   │   │   └── ...（其他 Identifiable）
  │   │   └── Traceable / ...
  │   └── ...
  └── AUTOSAR                        # 根节点（多重继承）
```

---

## 3. 任务拆解全景（WBS）

```
1.0 L1 AutosarFactory 从零开发
│
├── 1.1 子系统 A：XSD 代码生成器
│   ├── 1.1.1 需求分析（AUTOSAR XSD 结构分析）
│   ├── 1.1.2 架构设计（生成管线、模板引擎、输出结构）
│   ├── 1.1.3 代码实现（XSD 解析器 + Schema 模型 + 代码生成引擎）
│   └── 1.1.4 单元测试（生成代码正确性验证）
│
├── 1.2 子系统 B：核心运行时
│   ├── 1.2.1 需求分析（模型 API 约定、全局状态设计）
│   ├── 1.2.2 架构设计（类体系、模块 API 签名、内部机制设计）
│   ├── 1.2.3 代码实现（基类 + 模块 API + 内部引擎）
│   └── 1.2.4 单元测试（每机制独立验证）
│
├── 1.3 子系统 C：XmlElementDirtyTracker
│   ├── 1.3.1 架构设计
│   ├── 1.3.2 代码实现
│   └── 1.3.3 单元测试
│
├── 1.4 子系统 D：datatype_utils
│   ├── 1.4.1 代码实现
│   └── 1.4.2 单元测试
│
├── 1.5 子系统 E：autosar_ui
│   ├── 1.5.1 架构设计
│   ├── 1.5.2 代码实现
│   └── 1.5.3 单元测试
│
├── 1.6 子系统 F：多版本 Release 快照
│   ├── 1.6.1 快照机制实现
│   └── 1.6.2 多版本验证
│
├── 1.7 子系统 G：测试体系
│   ├── 1.7.1 测试基建设计（fixtures、teardown/reinit 模式）
│   ├── 1.7.2 集成测试（端到端场景）
│   └── 1.7.3 性能基准测试
│
├── 1.8 子系统 H：文档与项目管理
│   ├── 1.8.1 API 参考文档
│   ├── 1.8.2 开发者指南（CLAUDE.md）
│   └── 1.8.3 项目评审与进度管理
│
└── 1.9 集成验证
    ├── 1.9.1 端到端场景验证
    └── 1.9.2 回归测试套件建立
```

---

## 4. 子系统 A：XSD 代码生成器

> **定位**：将 AUTOSAR XSD Schema 文件（`AUTOSAR_00053.xsd`）转换为 ~250+ 个 Python 类定义（~440k 行），每个类提供标准化的 get/set/new/add/remove 方法。本生成器是**开发期工具**，运行一次即产出成品代码；后续 Schema 升级时重新运行生成。

### 4.1 需求分析（5 PD）

| 子任务 | 描述 | 输出 |
|--------|------|------|
| AUTOSAR XSD 结构分析 | 深入理解 AUTOSAR XSD 的命名空间、类型系统、继承模式、compositor（`xs:sequence`/`xs:choice`/`xs:all`）、`xs:extension`/`xs:restriction`、`xs:simpleType`/`xs:complexType`、枚举定义、引用/包含机制 | 《AUTOSAR XSD 分析报告》 |
| 元模型映射规则 | 确定 XSD 类型 → Python 类的映射规则：类名生成、属性名生成（单复数规则）、方法签名生成、枚举命名、包结构映射 | 《XSD-Python 映射规则表》 |
| 生成代码契约 | 确定生成代码必须遵循的 API 约定（`get_<attr>` / `set_<attr>` / `new_<Type>` / `add_<ref>` / `remove_<ref>`），与手写基类的对接接口 | 《生成代码契约》 |

### 4.2 架构设计（5 PD）

| 子任务 | 描述 | 输出 |
|--------|------|------|
| 生成管线设计 | 输入（XSD）→ 解析 → 中间表示（Schema Model）→ 代码生成 → 输出（Python）；每阶段独立可测 | 《生成器架构设计文档》 |
| 中间表示（IR）设计 | 定义 Schema Model 数据结构：ClassDef、AttributeDef、ReferenceDef、EnumDef、InheritanceGraph | 《IR 数据结构设计》 |
| 模板引擎设计 | 选择代码生成策略：字符串模板 vs AST 生成 vs Jinja2；类模板、方法模板、枚举模板的设计；import 管理 | 《模板引擎设计》 |

### 4.3 代码实现（25 PD）

| 模块 | PD | 实现要点 |
|------|-----|---------|
| **XSD 解析器** | 8 | 使用 lxml 解析 XSD；处理 `xs:include`/`xs:import` 跨文件引用；解析所有 `simpleType`/`complexType`/`element`/`attribute`/`group`/`attributeGroup`；处理 `xs:extension`（继承链提取）、`xs:restriction`（约束提取）；compositor 结构（`sequence`/`choice`/`all`）的深度解析 |
| **Schema Model（IR）** | 5 | 构建完整的类型/元素/属性/枚举/继承关系图；引用 vs 包含 vs 属性 的语义区分（关键：引用是 `set_<x>(node)` 传节点对象，包含是 `new_<X>(name)` 工厂创建）；multiplicity 解析（minOccurs/maxOccurs → 单值/多值 API） |
| **代码生成引擎** | 10 | 类定义生成（含 docstring、基类声明、`__init__`）；单值属性：`get_<attr>()` / `set_<attr>(val)`；多值属性：`get_<attr>s()` / `add_<attr>(val)` / `remove_<attr>(val)` / `set_<attr>s(list)`；引用属性：额外的 `_as_string()` 方法（悬空引用保护）；工厂方法：`new_<Type>(name)` 含重名检查、schema 顺序插入；枚举类：`<Name>Enum` 含 `unique` 装饰器；`get_children()` / `get_property_values()` 自省方法 |
| **边界情况与验证** | 2 | 特殊字符处理（AUTOSAR 命名中的 `/`、`-` 转义）；复数拼写规则（XSD 不一致的复数形式）；保留字冲突检查；生成代码的 AST 语法校验、import 完整性校验 |

### 4.4 单元测试（7 PD）

- XSD 解析器：10+ AUTOSAR XSD 片段 → 中间表示的正确性断言
- 代码生成引擎：关键类（`ApplicationSwComponentType`、`ISignal`、`SenderReceiverInterface`）的生成代码与预期 Python 代码 diff 测试
- 回归测试：XSD 升级时生成代码的 diff 可控

**子系统 A 小计**：42 PD

---

## 5. 子系统 B：核心运行时

> **定位**：手工编写的 Python 代码，包括基类继承体系、模块级公开 API、XML 操作引擎（读/写/合并/引用解析/Schema 顺序插入）。这部分是 L1 的核心智力产出，决定整个底座的可靠性和性能特征。

### 5.1 需求分析（5 PD）

| 子任务 | 描述 |
|--------|------|
| 公开 API 契约精确定义 | `read`、`new_file`、`save`、`saveAs`、`get_root`、`get_node`、`get_all_instances`、`export_to_file`、`reinit` 的完整行为规约（含边界条件、错误语义） |
| 元模型访问约定 | `get_<attr>` / `set_<attr>` / `new_<Type>` / `add_<ref>` / `remove_<ref>` 的语义定义、命名规则（单复数处理）、类型检查契约 |
| 内部机制需求 | 多文件合并的 splitable 判定逻辑、引用解析时序、Schema 插入顺序保证、增量保存触发条件 |
| 非功能需求 | 大文件（100MB+ ARXML）解析性能、1000 节点 dirty track 性能、`save()` 增量写入延时、内存占用基线 |

### 5.2 架构设计（5 PD）

| 子任务 | 描述 |
|--------|------|
| **类继承体系设计** | 8 级基类链（`AutosarNode → ARObject → Referrable → MultilanguageReferrable → Identifiable → CollectableElement → PackageableElement → ARElement`）的职责分配、多重继承策略（`ARPackage` 同时是 `CollectableElement` + `AtpBlueprint` + `AtpBlueprintable`） |
| **模块 API 设计** | 全局状态变量布局（`__autosarNode__`、`__pathsToNodeDict__`、`__autosarPathsToNodeDict__`、`__refBaseToArPackageDict__`、`XDT`）；公开函数签名与内部辅助函数的边界 |
| **XML 引擎设计** | lxml wrapper 抽象层设计；`_build()` 方法（lxml → Python 对象树构建）流程；`_add_or_get_xml_node` / `_insert_element_after_given_tags` 算法；`_save_element` / `_save_children` 序列化流程 |
| **合并引擎设计** | `__merge_to_parent_node` 算法；splitable 判定接口；名称冲突处理策略（warning + 跳过）；`_mergedFromNodes` 记录机制 |
| **引用引擎设计** | `_resolve_references` 时序（ReferenceBase 优先 → 其他）；`referenced_by` 反向边双向维护；`get_<attr>_as_string()` 悬空引用容错 |

### 5.3 代码实现（38 PD）

#### 5.3.1 基类实现（8 PD）

| 类/模块 | PD | 实现要点 |
|---------|-----|---------|
| `AutosarNode` | 2 | `__init__`（lxml node + parent + file + dirty 标记）；`unique_hash` 属性；`_build()` 模板方法；`get_children()` 反射；`add_child()`；`get_parent()`；`referenced_by` 列表维护；`is_atp_splitable()` 判定；`mark_dirty()` 委托 XDT |
| `ARObject` | 1 | lxml 元素包装；`_node` 属性访问；`_add_or_get_xml_node`（Schema 顺序插入核心）；`_insert_element_after_given_tags`；XML 序列化辅助 |
| `Referrable` | 1 | `name`（SHORT-NAME）属性；`full_path` 计算；`__eq__`（基于路径比较） |
| `MultilanguageReferrable` | 0.5 | 多语言长名称支持 |
| `Identifiable` | 1 | UUID 管理；`_splitable` 判定；路径注册（`__pathsToNodeDict__` 缓存更新） |
| `CollectableElement` | 0.5 | `export_to_file` 导出 |
| `PackageableElement` + `ARElement` | 0.5 | ARElement 特定语义 |
| `ARPackage` | 1 | 容器节点的子元素管理；路径缓存更新 |
| `AUTOSAR` 根节点 | 0.5 | 根节点特殊逻辑；`save()` / `saveAs()` 入口；`_mergedFromNodes` 管理 |

#### 5.3.2 模块级公开 API（8 PD）

| 函数 | PD | 实现要点 |
|------|-----|---------|
| `read(files)` | 2 | 文件/文件夹遍历；`etree.parse`（`huge_tree=True`）；XML 根元素校验；首次读取 vs 累积合并分支；`_resolve_references` 调用 |
| `__read_file` + `__read_folder` | 1 | lxml 解析 + 错误处理；AUTOSAR 根元素校验 |
| `__merge_to_parent_node` | 1 | 按路径/短名称查重；splitable → 递归合并子节点；非 splitable Identifiable → warning |
| `new_file(path, ...)` | 1 | 创建 lxml Element 树；写入空模板 ARXML；注册到全局状态 |
| `save(files)` / `saveAs(file)` | 1 | Dirt check → 跳未变更文件；调用根节点序列化 |
| `get_node(path)` | 0.5 | 缓存查询 |
| `get_all_instances(node, clazz)` | 0.5 | 深度优先遍历 + `isinstance` 过滤 |
| `export_to_file(element, path)` | 0.5 | 类型校验（限 `CollectableElement`）+ 序列化 |
| `reinit()` | 0.5 | 清所有全局变量 + XDT.clear |

#### 5.3.3 XML 序列化引擎（7 PD）

| 模块 | PD | 实现要点 |
|------|-----|---------|
| `_save_element` | 2 | 递归序列化单个元素；仅处理 dirty 元素（增量保存核心）；属性 → XML 属性、子元素 → XML 子元素 的正确映射 |
| `_save_children` | 2 | 遍历子元素；保持 Schema 顺序；处理 splitable 元素的特殊保存逻辑 |
| Schema 顺序保证 | 3 | 从 XSD 提取每个类的子元素序列定义（由代码生成器嵌入）；`_add_or_get_xml_node` 在正确位置插入；`_insert_element_after_given_tags` 精确定位 |

#### 5.3.4 引用解析引擎（8 PD）

| 模块 | PD | 实现要点 |
|------|-----|---------|
| `_resolve_references`（通用） | 3 | 遍历所有属性检测引用路径；`get_node(path)` 解析 → `set_<attr>(node)` 自动关联；悬空引用处理（保留路径字符串 + logging warning） |
| `ReferenceBase` 处理 | 2 | 对应的 `shortLabel` → ARPackage 映射（`__refBaseToArPackageDict__`）；引用解析快捷路径 |
| `referenced_by` 双向维护 | 3 | `set_<attr>` / `add_<attr>` 时自动更新目标的 `referenced_by` 列表；`remove_<attr>` 时清理反向边；`del` / `set_<attr>(None)` 也要清理；跨 save/reload 保持一致性 |

#### 5.3.5 异常体系（2 PD）

| 异常类 | 触发条件 |
|--------|---------|
| `NoShortNameException` | `new_<Type>()` 创建 Referrable 时未提供 `name` |
| `InvalidRefOrChildNodeException` | `set_<attr>` / `add_<attr>` 传入类型不符的值 |
| `InvalidInputException` | 模块函数入参类型错误（如 `read("not_a_list")`） |

#### 5.3.6 杂项：命名空间、常量、Enum（3 PD）

- `__autosarNsMap__`：AUTOSAR XML 命名空间定义
- `__rootSchemaAttr__`：`xsi:schemaLocation` 属性
- `__str_to_class__`：字符串 → 类对象映射
- Enum 基类：所有 `*Enum` 使用 `@unique` 保证枚举值不重复

### 5.4 单元测试（16 PD）

| 测试对象 | PD | 测试重点 |
|---------|-----|---------|
| `AutosarNode` 基类 | 2 | 构造、路径计算、父子关系、`get_children()`、`add_child()` |
| `read()` / `__merge_to_parent_node` | 3 | 单文件读取、多文件合并、splitable 合并、重复 Identifiable 处理、文件不存在的错误处理 |
| `new_file()` | 1 | 文件创建、覆盖保护、ARPackage 创建正确性 |
| `save()` / `saveAs()` | 3 | 增量保存（仅 dirty 写入）、全量保存、文件不存在时创建、dirty 传播验证 |
| `get_node()` / `get_all_instances()` | 1 | 缓存命/未命中、深度搜索正确性 |
| 引用解析 | 3 | 同文件引用、跨文件引用、悬空引用、`referenced_by` 双向维护、`ReferenceBase` 映射 |
| Schema 顺序 | 2 | 乱序 set → 输出顺序符合 XSD、`_add_or_get_xml_node` 位置正确 |
| `export_to_file()` | 0.5 | 非 CollectableElement 拒绝、导出内容正确 |
| `reinit()` | 0.5 | 全局状态完全清空、后续操作不受污染 |

**子系统 B 小计**：64 PD

---

## 6. 子系统 C：XmlElementDirtyTracker 脏跟踪器

> **定位**：独立模块，管理 lxml 元素的"脏"状态。任何变更操作（`set_`/`add_`/`new_`/`remove_`）均触发脏标记，并沿祖先链和 `referenced_by` 反向边传播，确保 `save()` 不遗漏任何受影响元素。

### 6.1 架构设计（2 PD）

- 脏标记数据结构设计（dict：key = element hash + attribute name）
- 传播策略：祖先链单向向上 + referenced_by 反向扩展
- 与 `save()` 的集成点设计
- 属性级 vs 元素级脏标记

### 6.2 代码实现（10 PD）

| 模块 | PD | 实现要点 |
|------|-----|---------|
| 核心数据结构 | 2 | `__dirty` dict；`get_unique_hash()` 作为 key；元素 key vs 属性 key 区分 |
| `mark_dirty()` | 3 | 祖先链向上遍历直到 root 或已标记；`referenced_by` 反向边遍历；parent 边界检查 |
| `mark_attribute_dirty()` | 1 | 属性级脏标记；同时标记所属元素 |
| `is_dirty()` / `is_attribute_dirty()` | 1 | 查询接口 |
| `clear_dirty()` | 0.5 | 全清（`reinit` 调用） |
| 集成到基类 | 2.5 | `AutosarNode.mark_dirty()` 委托；`set_`/`add_`/`new_`/`remove_` 自动调用 `mark_dirty`；引用变更的双向脏传播 |

### 6.3 单元测试（3 PD）

| 测试场景 | PD |
|---------|-----|
| 单元素 mark → 祖先链全脏验证 | 1 |
| 被引用元素 mark → 引用方也脏验证（双向传播） | 1 |
| 属性级脏标记 → `is_attribute_dirty` 验证 | 0.5 |
| `clear_dirty` → 全部恢复 clean | 0.5 |

**子系统 C 小计**：15 PD

---

## 7. 子系统 D：数据类型工具

> **定位**：`datatype_utils.py`——解析 AUTOSAR 中不同进制/格式的文字量值（十进制、十六进制、八进制、二进制、浮点、布尔），统一返回 Python 原生类型。

### 7.1 代码实现（3 PD）

| 函数 | 描述 |
|------|------|
| `get_int_value(text)` | 自动识别 0x/0X（十六进制）、0b/0B（二进制）、0（八进制）、十进制 |
| `get_float_value(text)` | 浮点数字符串 → float |
| `get_bool_value(text)` | `"1"` / `"true"` → `True`；其他 → `False` |
| `get_string_value(text)` | `strip()` 处理 |

### 7.2 单元测试（1 PD）

- 各进制边界值测试（0x00、0xFF、0b11111111、0777、255）
- 异常输入容错（空字符串、非法字符）

**子系统 D 小计**：4 PD

---

## 8. 子系统 E：UI 可视化器

> **定位**：`autosar_ui.py`——基于 Tkinter + sv_ttk 的桌面端 ARXML 模型浏览器，提供树形导航、属性查看、反向引用查看、搜索功能。面向开发调试和集成工程师日常浏览。

### 8.1 架构设计（1 PD）

- MVC 分离：Application 主框架 → TreeExplorer（树面板） + PropertyView（属性面板） + ReferredByView（引用面板）
- 主题系统（sv_ttk 集成、多主题支持）
- 搜索架构（路径搜索/类型搜索）
- 菜单与右键上下文设计

### 8.2 代码实现（7 PD）

| 模块 | PD | 实现要点 |
|------|-----|---------|
| **Application 主框架** | 1.5 | Tk `Frame` 布局；PanedWindow 可调节分割；菜单栏；主题切换（scidgreen / dark 等） |
| **TreeExplorer 树导航** | 2 | Treeview 虚拟树；惰性加载子节点（大量节点时性能关键）；节点图标区分（ARPackage / SWC / Interface / Signal）；右键上下文菜单（展开/折叠/定位） |
| **PropertyView 属性面板** | 1.5 | 选中节点 → 属性表格（名/值/类型）；`get_property_values()` 驱动；只读展示 |
| **ReferredByView 引用面板** | 1 | 展示 `referenced_by` 列表；点击跳转到引用方节点 |
| **Search 搜索** | 1 | 支持按名称/路径/类型搜索；下拉建议；搜索结果列表；点击跳转 |

### 8.3 单元测试（2 PD）

- TreeExplorer 节点构建正确性
- PropertyView 各类属性展示验证
- 搜索功能回归测试

**子系统 E 小计**：10 PD

---

## 9. 子系统 F：多版本 Release 快照

> **定位**：`autosar_releases/` 目录——将 L1 代码（`autosarfactory.py` + `datatype_utils.py` + `XmlElementDirtyTracker.py` + `__init__.py`）冻结为特定 AUTOSAR Release 的快照，以支持旧版本建模。

### 9.1 架构设计与实现（1.5 PD）

- 快照目录结构（`autosar_releases/autosar422/` 等）
- 当前支持 10+ 个 Release（R4.0.3 ~ R24-11）
- 快照生成脚本（从顶层包复制 + 必要时剥离新版本特有类）
- 版本号 → 目录名的映射约定

### 9.2 验证（0.5 PD）

- 快照目录 import 测试
- 各快照不互相污染的隔离验证

**子系统 F 小计**：2 PD

---

## 10. 子系统 G：测试体系

> **定位**：覆盖所有 L1 组件的集成测试和端到端验证。各子系统的单元测试已在上面分别计入；本子系统聚焦于跨组件的集成场景、性能基准、测试基建设计。

### 10.1 测试基建（3 PD）

- pytest fixture 体系：准备标准 ARXML 样本文件（`components.arxml`、`datatypes.arxml`、`interfaces.arxml` 等）
- `teardown()` 函数：每个测试结束后调用 `AF.reinit()`（避免全局状态污染）
- 测试资源目录 `tests/resources/`：包含 split1/split2（合并测试）、invalid.arxml（错误处理测试）

### 10.2 集成测试（10 PD）

| 测试场景 | PD | 描述 |
|---------|-----|------|
| **端到端：新建 → 配置 → 保存 → 重读** | 3 | `new_file → new_SwBaseType → set_* → save → read → get_node 验证` |
| **端到端：跨文件引用** | 2 | 文件 A 定义 DataType → 文件 B 引用 → read 合并 → 引用自动解析 |
| **端到端：多文件合并** | 2 | 多文件 read → splitable 元素合并 → 非 splitable 重复 warning → 合并后树结构验证 |
| **端到端：增量保存** | 1.5 | 读入 2 文件 → 仅修改文件 A 的一个元素 → save → 验证文件 A 被重写、文件 B 未触及 |
| **端到端：脏传播完整性** | 1.5 | 创建子元素 → 修改 → 验证祖先也被标脏 → save 产物包含所有关联修改 |

### 10.3 性能基准测试（3 PD）

| 指标 | PD | 目标 |
|------|-----|------|
| 大文件解析 | 1 | 100MB+ ARXML 文件 `read()` < 30s |
| 增量保存 | 1 | 1000 节点模型中修改 1 个元素 → `save()` < 1s |
| 路径查找 | 1 | 10000 节点模型中 `get_node()` < 0.1ms（缓存命中） |

**子系统 G 小计**：16 PD

---

## 11. 子系统 H：文档与项目管理

### 11.1 文档（8 PD）

| 文档 | PD | 内容 |
|------|-----|------|
| **API 参考文档** | 5 | 模块级 API（`read` / `save` / `get_node` / ...）的完整签名与示例；元模型访问约定（`get_<attr>` / `set_<attr>` / `new_<Type>` / ...）的模式说明；异常类型与触发条件；枚举使用方式 |
| **CLAUDE.md / 开发者指南** | 3 | 项目架构概览；代码生成 vs 手写代码的边界说明；如何添加新的元模型支持；测试规范（`teardown/reinit` 模式）；常见陷阱（不直接操作 lxml、`reinit` 边界、脏传播规则） |

### 11.2 项目管理（8 PD）

| 活动 | PD | 说明 |
|------|-----|------|
| 设计评审 | 4 | 启动时 2 次（XSD 生成器架构 + 核心运行时架构） |
| 代码评审 | 2 | 关键模块合入前 peer review |
| 进度跟踪 | 2 | 双周里程碑检查、风险应对 |

**子系统 H 小计**：16 PD

---

## 12. 工时汇总与人力资源排布

### 12.1 总工时汇总

| 子系统 | 需求分析 | 架构设计 | 代码实现 | 单元测试 | 小计 (PD) |
|--------|---------|---------|---------|---------|-----------|
| A. XSD 代码生成器 | 5 | 5 | 25 | 7 | **42** |
| B. 核心运行时 | 5 | 5 | 38 | 16 | **64** |
| C. 脏跟踪器 | — | 2 | 10 | 3 | **15** |
| D. 数据类型工具 | — | — | 3 | 1 | **4** |
| E. UI 可视化器 | — | 1 | 7 | 2 | **10** |
| F. 多版本快照 | — | — | 1.5 | 0.5 | **2** |
| G. 测试体系 | — | — | — | 16 | **16** |
| H. 文档与项目管理 | — | — | 8 | 8 | **16** |
| **合计** | **10** | **13** | **92.5** | **53.5** | **169** |

> **等效人月**：169 ÷ 22 工作日/月 ≈ **7.7 人月**

### 12.2 人力资源排布

| 团队成员 | 负荷 | 主攻方向 | 说明 |
|---------|------|---------|------|
| **何康** | 100% | 子系统 B（核心运行时）+ 子系统 C（脏跟踪器）+ 子系统 G（集成测试） | L1 核心引擎的主力开发者，负责运行时正确性与性能 |
| **刘鹏** | 100% | 子系统 A（XSD 代码生成器）+ 子系统 D/E/F（工具链 + UI + 快照）+ 子系统 H（文档） | 生成器与周边工具，负责元模型覆盖的完整性与开发者体验 |

### 12.3 工期计算

| 计算步骤 | 数值 |
|---------|------|
| 总工时 | 169 PD |
| 等效人数 | 2.0 FTE |
| 理论最短工期 | 169 ÷ 2 = **84.5 个工作日** ≈ **17 周** ≈ **4.2 个日历月** |
| 并行度损失（~15%，接口协商/代码合并/联调） | 84.5 × 1.15 ≈ **97 个工作日** |
| 缓冲（~10%，应对需求变更/技术难点返工） | 97 × 1.10 ≈ **107 个工作日** |

> **建议项目周期：5 ~ 5.5 个日历月（~22 周）**

### 12.4 关键路径分析

```
XSD 分析 ──► 代码生成器实现 ──► 生成第一版类代码 ──► 核心运行时开发 ──► 集成测试
    │              │                    │                    │
    └── 5 PD ──────┴── 25 PD ──────────┴── 38 PD ───────────┴── 16 PD
    <────────────────── 关键路径：~84 PD（单人串行）────────────────────>
```

**说明**：核心运行时（64 PD）和 XSD 生成器（42 PD）有一定依赖关系——运行时基类必须先稳定，生成器才能产出与基类契约一致的代码。但同时，生成器的 XSD 解析和 IR 构建阶段（前 18 PD）可与运行时架构设计并行。真正的阻塞点是：运行时基类 API 冻结后，生成器才能最终确定生成的代码模板。

---

## 13. 里程碑与甘特图

### 13.1 里程碑清单

| 里程碑 | 时间点 | 验收标准 |
|--------|-------|---------|
| **M0** 项目启动 | Week 0 | 需求评审通过、XSD 分析完成 |
| **M1** 架构冻结 | Week 3 | 生成器架构 + 运行时架构设计文档评审通过 |
| **M2** XSD 生成器可用 | Week 8 | 生成器输出 ~250+ 类，Python 语法校验通过 |
| **M3** 核心运行时就绪 | Week 13 | `read` / `save` / `new_file` / `get_node` 端到端可用 |
| **M4** 脏跟踪 + UI + 快照就绪 | Week 16 | 增量保存正确、UI 可浏览模型、多版本可导入 |
| **M5** 集成验证完成 | Week 19 | 所有集成测试通过、性能达标、39+ 回归用例通过 |
| **M6** 正式交付 | Week 22 | 文档完备、CLAUDE.md 就绪、验收评审通过 |

### 13.2 甘特图（按周）

```
Week:  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22
       ├──┴──┴──┼──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┤

A. XSD 代码生成器
何康  ░░░░░░░░░░                                                    
刘鹏  ██████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░

B. 核心运行时
何康  ░░░░░░░░░░████████████████████████████████████████░░░░░░░░░░░░
刘鹏  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████████████████░░░░░░

C. 脏跟踪器
何康  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████░░░░░░░░░░░░░░

D+E+F. 工具链+UI+快照
刘鹏  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████░░░░░░░░

G. 测试体系
何康  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████
刘鹏  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████

H. 文档与PM
何康  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████
刘鹏  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████

M0  M1         M2                  M3                  M4       M5  M6
▼    ▼          ▼                   ▼                   ▼        ▼   ▼
```

### 13.3 分工详解

| 阶段 | 周次 | 何康 | 刘鹏 |
|------|------|------|------|
| 需求+架构 | W1-3 | B. 核心运行时需求/架构（10 PD）| A. XSD 生成器需求/架构 + 设计评审（10 PD） |
| 实现 Phase 1 | W3-8 | B. 基类实现（8 PD）+ 开始模块 API（8 PD）| A. XSD 生成器代码实现（25 PD）+ 单元测试（7 PD）|
| 实现 Phase 2 | W8-13 | B. XML 序列化引擎（7 PD）+ 引用引擎（8 PD）+ 异常/杂项（5 PD）| D+E+F. 数据工具+UI+快照（14 PD）+ B. 引用引擎协助（6 PD）|
| 实现 Phase 3 | W13-16 | C. 脏跟踪器（15 PD）+ B. Unit Test（16 PD）| E. UI 完善 + 文档初稿（8 PD）|
| 集成验证 | W16-19 | G. 集成测试 + 性能测试（10 PD）| G. 集成测试 + 测试基建（6 PD）|
| 交付 | W19-22 | H. 文档完成 + 代码评审（5 PD）| H. 文档完成 + 验收准备（5 PD）|

---

## 14. 风险管理

| # | 风险 | 概率 | 影响 | 缓解措施 |
|---|------|------|------|---------|
| R1 | **XSD Code Gen 产量过大，生成代码有隐蔽 bug** | 中 | 高 | 分阶段生成：先产 10 个核心类验证模板正确性 → 批量生成全量 → 逐类 diff 验证；AST 语法校验自动化 |
| R2 | **引用双向维护出现遗漏（类比 0.6.3 修复的 bug）** | 中 | 高 | 引用引擎实现时用 TDD：先写 `referenced_by` 测试用例，后写实现；集成测试增加"修改引用 → save → reload → 验证反向边"循环 |
| R3 | **Schema 顺序保证算法在复杂 XSD 上不完整** | 中 | 中 | 核心类（`ARPackage`、`ApplicationSwComponentType`）手写顺序测试；CI 增加 ARXML Schema 校验步骤 |
| R4 | **2 人团队单人 bus factor 高** | 高 | 高 | 关键模块（运行时基类、XSD 生成器核心逻辑）要求交叉 code review；CLAUDE.md 必须写作到另一人可接手程度 |
| R5 | **AUTOSAR XSD 版本升级导致重新生成代码 diff 巨大** | 低 | 中 | 生成器设计时考虑 XSD 兼容性；生成代码包含版本标记注释；diff 工具链预先准备 |
| R6 | **大文件性能不达标（100MB+ ARXML 解析超时）** | 低 | 中 | lxml 的 `huge_tree=True` 已验证可行；性能基准测试在 Week 12 提前介入，预留 profiling 优化时间 |
| R7 | **UI 可视化器在大模型（10000+ 节点）上卡顿** | 中 | 低 | 树视图采用惰性加载（展开时才加载子节点）；不做全量一次性渲染 |

---

## 15. 交付物清单

### 15.1 代码交付

| 交付物 | 路径 | 规模（估算） | 说明 |
|--------|------|------------|------|
| XSD 代码生成器 | `codegen/` 或独立脚本 | ~3000 行 | XSD 解析 + IR 构建 + 代码生成引擎 |
| 生成的元模型类 | `autosarfactory/autosarfactory.py` | ~440k 行 | ~250+ Python 类，代码生成器输出 |
| 手写运行时基类 | `autosarfactory/autosarfactory.py`（顶部） | ~2000 行 | 8 级基类体系 + 模块级函数 + 内部引擎 |
| 脏跟踪器 | `autosarfactory/XmlElementDirtyTracker.py` | ~100 行 | 增量为保存标记引擎 |
| 数据类型工具 | `autosarfactory/datatype_utils.py` | ~50 行 | 多进制/格式量值解析 |
| UI 可视化器 | `autosarfactory/autosar_ui.py` | ~2000 行 | Tk 树导航 + 属性/引用面板 + 搜索 |
| 多版本快照 | `autosar_releases/` | 10+ 目录 | 各 Release 冻结副本 |
| 包初始化 | `autosarfactory/__init__.py` | ~5 行 | 包导入 |
| 单元测试 | `tests/test_autosarmodel.py` | ~2000 行 | 39+ 用例，含 fixtures |
| 集成测试 | `tests/test_integration.py` | ~1000 行 | 端到端场景验证 |
| 测试资源 | `tests/resources/` | 10+ 样本文件 | 正常/异常/多文件 ARXML 样本 |

### 15.2 文档交付

| 交付物 | 说明 |
|--------|------|
| 《AUTOSAR XSD 分析报告》 | XSD 结构、继承关系、命名规律 |
| 《XSD-Python 映射规则表》 | 类型/属性/引用/枚举的映射规则 |
| 《L1 架构设计文档》 | 类体系、模块 API、XML 引擎、引用引擎、合并引擎设计 |
| 《生成器架构设计文档》 | 管线、IR、模板引擎设计 |
| 《API 参考文档》 | 所有公开 API 的签名、参数、返回值、异常 |
| `CLAUDE.md` | 项目架构、开发规范、测试规范、常见陷阱 |
| 《工时评估报告》（本文档） | v1.0 |

---

## 附录 A：PD 估算方法与假设

| 假设 | 说明 |
|------|------|
| **1 PD = 8 有效工时** | 扣除会议、沟通、文档等 overhead |
| **开发效率** | 2 人均为中级以上工程师，可独立承担子系统 |
| **代码生成器复杂度** | 类比业界 XSD/JAXB/XAML 代码生成器的实现经验，AUTOSAR XSD 规模较大（~250 类型）但结构规律性强 |
| **测试 = 实现 × 0.25~0.4** | 单元测试开发量约为功能代码的 25-40% |
| **并行度损失 15%** | 2 人团队因接口协商、代码合并、联调带来的效率折损 |
| **缓冲 10%** | 应对需求变更、技术难点攻关、返工等不可预见工作 |
| **L1 代码生成器产出的 440k 行代码不计入手工 PD** | 生成器本身 ~3000 行，其输出的 440k 行是机器产物 |

---

## 附录 B：与现有 AutosarFactory 代码的对应关系

| 现有文件 | 本报告对应子系统 | 现有规模 | 说明 |
|---------|----------------|---------|------|
| `autosarfactory.py`（顶部基础代码） | B. 核心运行时 | ~2000 行手写 | 基类 + 模块 API + 引擎 |
| `autosarfactory.py`（类定义） | A. XSD 代码生成器输出 | ~438k 行生成 | 生成器产生 |
| `XmlElementDirtyTracker.py` | C. 脏跟踪器 | ~46 行 | 独立模块 |
| `datatype_utils.py` | D. 数据类型工具 | ~38 行 | 小工具 |
| `autosar_ui.py` | E. UI 可视化器 | ~数千行 | Tk 应用 |
| `__init__.py` | B. 核心运行时 | ~3 行 | 包导入 |
| `autosar_releases/` | F. 多版本快照 | 10+ 目录 | 快照机制 |
| `tests/` | G. 测试体系 | ~2000 行 | 39 用例 |

---

> **结论**：2 人团队（何康、刘鹏）从零构建 AutosarFactory L1 底座（含 XSD 代码生成器、核心运行时、脏跟踪器、UI 可视化器、多版本快照、完整测试体系），预计需要 **169 人天**，建议项目周期 **5 ~ 5.5 个日历月（~22 周）**。关键路径由 XSD 生成器 → 核心运行时 → 集成测试构成，两线并行（何康主攻运行时，刘鹏主攻生成器与工具链）可有效缩短工期。
