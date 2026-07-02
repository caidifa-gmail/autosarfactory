# AUTOSAR IaC 三层 ARXML API 开发设计思路

> 版本：v0.1 ｜ 日期：2026-06-30
> 依据：`cai-input/AUTOSAR Integration-as-Code (IaC) 技术方案0.1.md` + AutosarFactory 现有代码能力
> 定位：将「Integration-as-Code (IaC)」理念，落到 AutosarFactory 真实代码之上的**可执行 API 开发设计**

---

## 0. 文档目的与范围

本设计文档回答一个问题：**如何在 AutosarFactory 之上，构建一套支撑「软件定义集成」的三层 Python API 体系？**

- **不重复** IaC 技术方案中的行业背景与价值论述（已由输入文档阐明）。
- **聚焦** 三层 API 的逻辑边界、接口契约、扩展机制、落地路径，并明确标注每一层在 AutosarFactory 中的**现状（已有 / 待建）**。
- **面向** 两类读者：①工具/API 维护者（厂商底座）；②集成工程师（企业业务资产开发者）。

### 关键结论速览

| 层 | 输入文档定位 | AutosarFactory 现状 | 本设计结论 |
|---|---|---|---|
| L1 基础 ARXML API | 元模型原子操作底座 | **已具备且成熟**（440k 行生成元模型 + 手写运行时） | 固化为「不可改底座」，对外稳定 API 契约 |
| L2 模块/协议栈 API | 封装单模块参数约束 | **数据已有**（`ecuc_param_def.json` + MCP），**Python 封装缺失** | 以「数据驱动 + Builder/Helper」补齐封装层 |
| L3 高阶 Feature API | 跨模块整车集成业务 | **未提供**（仅 `Examples/` 示例） | 由企业基于 L1/L2 自建，厂商只给规范与脚手架 |

---

## 1. 总体架构

### 1.1 分层全景

```
┌──────────────────────────────────────────────────────────────────────┐
│  L3  高阶 Feature API（业务层 / 企业资产）                               │
│      跨模块整车集成场景 · 企业规范沉淀 · 最佳实践模板                       │
│      例：build_can_gateway() / apply_pdu_packing_rules() / apply_diag_route() │
├────────────────────────────────────────────────────────────────────────┤
│  L2  模块 & 协议栈 API（封装层）                                          │
│      BSW 模块参数约束 · 标准化接口 · 屏蔽 ARXML 细节 · 参数依赖            │
│      例：OsBuilder / CanIfConfigurator / RteConfigurator                 │
│      数据来源：ecuc_param_def.json（ECUC 参数定义）                       │
├────────────────────────────────────────────────────────────────────────┤
│  L1  基础 ARXML API（底座 / AutosarFactory 内核，不可修改）                 │
│      原子增删改查 · 格式校验 · 自动生成 · 多文件合并 · 引用解析             │
│      read / new_file / get_node / get_all_instances / save / saveAs …    │
│      元模型约定：get_/set_/new_/add_/remove_/get_children                  │
├────────────────────────────────────────────────────────────────────────┤
│  横切：MCP 服务（AI 赋能）· CI/CD 适配 · 多 Release 快照（R4.0 … R24-11）    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 三层逻辑边界（必须严格隔离）

| 边界规则 | L1 | L2 | L3 |
|---|---|---|---|
| 业务语义 | **无** | 单模块内 | **跨模块整车级** |
| 可见抽象 | 元模型类/元素 | 模块参数对象 | 业务 Feature 函数 |
| 谁维护 | 厂商（固化） | 厂商 + 企业共建 | 企业（Git 私有） |
| 失败语义 | 元模型异常 | 参数约束校验 | 业务规则校验 |
| 允许直接调 L1 | — | ✅ 可调 | ✅ 可调 |

> **设计铁律**：依赖只能**向下**流动。L3 可调 L2/L1；L2 可调 L1；L1 对上层一无所知。这保证了「厂商底座」可独立演进而不破坏上层资产。

---

## 2. L1 基础 ARXML API（底座层）

> **现状：已具备且成熟。** 本节定义其对外契约，作为 L2/L3 的稳定地基。

### 2.1 设计原则

1. **无业务语义** — 只提供元模型语法级原子操作，绝不内置任何集成业务逻辑。
2. **schema 合规优先** — 任何插入都走 schema 顺序保证机制，确保输出可被 AUTOSAR 工具链校验通过。
3. **状态显式** — 全局单例模型，但提供 `reinit()` 做干净重置；调用方对生命周期负责。
4. **多 Release 兼容** — 顶层包固定为最新版（`AUTOSAR_00053` = R24-11），旧版以冻结快照形式独立导入。

### 2.2 核心 API 契约（模块级函数）

| 函数 | 签名 | 语义 |
|---|---|---|
| `read` | `read(files: list[str] = []) -> (AUTOSAR, bool)` | 解析文件/文件夹，**累积合并**进全局 root，解析引用；返回 `(root, ok)` |
| `new_file` | `new_file(path, defaultArPackage='RootPackage', overWrite=False) -> ARPackage` | 新建 arxml，返回默认 ARPackage |
| `get_root` | `get_root() -> AUTOSAR \| None` | 当前全局根 |
| `get_node` | `get_node(path) -> AutosarNode \| None` | 按 AUTOSAR 全路径查节点 |
| `get_all_instances` | `get_all_instances(node, clazz) -> list[AutosarNode]` | 深度搜索某类全部实例 |
| `export_to_file` | `export_to_file(element, path, overWrite=False)` | 导出 CollectableElement 子树为独立文件 |
| `save` | `save(files: list[str] = [])` | 回写**已变更**子树到原文件（或指定子集） |
| `saveAs` | `saveAs(file, overWrite=False)` | 整树合并序列化为单文件 |
| `reinit` | `reinit() -> None` | 清空 root + 全部缓存 + 脏集 |

### 2.3 元模型访问约定（全类一致，生成层固化）

对任意属性 `<attr>` / 类型 `<T>`：

| 模式 | 形态 | 说明 |
|---|---|---|
| `get_<attr>() / set_<attr>(v)` | 单值 | `set_x(None)` 即删除对应 XML 元素 |
| `get_<attr>s() / add_<attr>(v) / set_<attr>s(l) / remove_<attr>(v)` | 多值 | 单复数拼写遵循 XSD（如 `add_hwCategorie`） |
| `new_<Type>(name=...)` | 工厂 | 父节点上创建并挂接子节点；Referrable 必须给 name |
| `get_<attr>_as_string()` | 引用退化 | 悬空引用时仍可取路径字符串 |
| `get_children() / get_property_values()` | 结构/内省 | 供遍历、diff、调试 |

**基类链**：`AutosarNode → ARObject → Referrable → MultilanguageReferrable → Identifiable → CollectableElement → PackageableElement → ARElement`；容器 `ARPackage`；根 `AUTOSAR`。

### 2.4 四个必须对外讲清的关键机制

这些机制是 L2/L3 **不可绕过、必须顺应**的底层行为：

1. **脏标记驱动的增量保存（`XDT`）**
   - 任何 `set_/add_/new_/remove_` 都会 `XDT.mark_dirty`，并**沿祖先链 + `referenced_by` 反向边**传播。
   - `save()` 仅序列化脏子树；文件未变更则**整体跳过**（性能优化）。
   - **含义**：L2/L3 无需手写「哪些变了」，但**绝不能绕过 setter 直接操作 lxml**，否则脏传播缺失 → 引用目标丢失（0.6.3 修复的就是这类问题）。

2. **累积式多文件合并**
   - `read()` 可重复调用、多文件**合并为一棵内存树**；可分裂（splitable）的重复元素合并其子节点。
   - **含义**：跨文件引用（如接口在 A 文件、SWC 在 B 文件）天然可解析；但**无会话隔离** → 独立任务前必须 `reinit()`。

3. **schema 元素顺序保证**
   - 插入经 `_add_or_get_xml_node` / `_insert_element_after_given_tags`，**与 Python 设置顺序无关**，输出始终符合 AUTOSAR 序列。
   - **含义**：L2/L3 可任意顺序配置参数，L1 负责最终排序。

4. **双向引用维护**
   - `node.referenced_by` 在增删引用时双向维护，跨 save/reload 存活。
   - **含义**：L2/L3 可安全做「重定向引用 / 迁移元素」操作。

### 2.5 异常契约

| 异常 | 触发 |
|---|---|
| `NoShortNameException` | `new_*` 创建 Referrable 未给 name |
| `InvalidRefOrChildNodeException` | `set_/add_` 类型不符 / 子节点重名 |
| `InvalidInputException` | 模块函数入参非法（如 read 非 list） |
| `FileExistsError`（内建） | `new_file/export_to_file/saveAs` 目标已存在且 `overWrite=False` |

> **L2/L3 应捕获并**翻译**为业务可读错误**，而非把元模型异常直接抛给最终用户。

### 2.6 会话与并发模型（设计约束）

AutosarFactory 是**模块级单例**：root、路径缓存、引用反向边、脏集均为全局态。

- ✅ 推荐：**单进程单模型流水线**（一个 CI 任务 = 一个模型 = 一次 `reinit()` 前后边界）。
- ⚠️ 限制：**不支持同进程并发处理多模型**。需并发时，用**多进程**隔离（每进程独立 `reinit()`），而非多线程共享。
- 📌 文档/契约责任：L2/L3 的公共入口应在内部做好 `reinit()` 边界管理，避免上层误用导致跨任务污染。

---

## 3. L2 模块 & 协议栈 API（封装层）

> **现状：数据已具备（`ecuc_param_def.json` + MCP 的 4 个 ECUC 工具），但缺少面向 Python 脚本的标准化封装。** 本层是本设计的**主要补齐工作**。

### 3.1 设计目标

- 把「零散的 ECUC 参数 + 模块容器层级」封装为**单模块标准化接口**。
- 屏蔽 ARXML 元素细节，让集成工程师面对的是 `Os` / `CanIf` / `Rte` 等业务对象，而非 `EcucPartitionCollection` / `EcucNumericalParamDef`。
- 内建**参数依赖与标准约束**（multiplicity 下限、枚举合法值、必填项）。

### 3.2 数据驱动：以 `ecuc_param_def.json` 为单一事实源

复用 MCP 已建的 ECUC 层级库（模块 → 容器 → 参数/引用），避免在 Python 里硬编码 BSW 知识：

```
ecuc_param_def.json
  └─ modules[]
       ├─ name: "Os"            type / path
       └─ containers[]
            ├─ name / type / path / valueClass / parentValueClass
            ├─ parameters[]     ← multiplicity 下限 = 必填判据
            └─ references[]
```

### 3.3 推荐接口形态：Builder + Definition 绑定

采用 **Builder 模式** + **definition 绑定**（与 AUTOSAR ECUC 语义对齐：每个值对象通过 `set_definition` 绑到参数定义）。

```python
# —— 厂商提供的标准化封装（L2 示意）——
class OsConfigurator:
    """封装 Os 模块内部参数约束，屏蔽 ARXML 细节。"""

    def __init__(self, arpkg, os_module_def_path: str):
        self._pkg = arpkg
        self._def = AF.get_node(os_module_def_path)   # ECUC 定义必须先 read 进来

    def add_task(self, name: str, *, priority: int, activation: int = 1,
                 schedule: bool = True) -> "OsTask":
        """创建 OsTask 并施加 Os 模块约束（必填项、默认值、范围）。"""
        if not (1 <= priority <= 255):
            raise OsConfigError(f"priority {priority} 越界（1..255）")  # 业务可读异常
        container = self._pkg.new_EcucModuleConfigurationValues(...)      # L1 原子操作
        task = container.new_EcucPartitionCollection(name)
        task.set_definition(self._resolve_def("/Os/OsTask"))              # 绑定义
        task.set_parameter("OS_TASK_PRIORITY", priority)                  # 标准化 setter
        task.set_parameter("OS_TASK_ACTIVATION", activation)
        return task

    def _resolve_def(self, path):
        node = AF.get_node(path)
        if node is None:
            raise OsConfigError(f"缺少 ECUC 定义：{path}（请先 read 模块定义 arxml）")
        return node
```

**设计要点**：

1. **约束内建**：`priority` 范围、必填项（来自 `multiplicity` 下限）、枚举合法值（来自 `get_ecuc_param_literals`）都在 Builder 内校验，**把 L1 的元模型异常翻译成业务可读的 `OsConfigError`**。
2. **definition 绑定显式**：与 AUTOSAR ECUC 一致，值对象经 `set_definition` 指向参数定义节点；定义文件必须先 `read`。
3. **不跨模块**：`OsConfigurator` 只懂 Os；CAN 路由在 L3 协调。

### 3.4 L2 的能力清单（建议逐步补齐）

- [ ] **Definition Registry**：加载并索引 `ecuc_param_def.json`，提供 `path` → 参数定义的快速查询（复用 MCP 的 `_ecuc_modules/_ecuc_containers/_ecuc_enum_literals` 索引逻辑）。
- [ ] **标准 Builder 族**：每个常用 BSW 模块（Os / CanIf / Can / EthIf / Rte / Dcm / Dem …）一个 Configurator。
- [ ] **约束校验器**：基于 multiplicity + 枚举字面量，提供 `validate(module_config)`。
- [ ] **依赖解析器**：处理模块内参数依赖（如 `OsTask` 需绑定 `OsScheduleTable` / `OsAlarm`）。

---

## 4. L3 高阶 Feature API（业务层）

> **现状：未提供。** 仅 `Examples/create_autosar_basic_communication.py` 给出端到端范式。本层**由企业自建**，厂商只给规范、脚手架与示例。

### 4.1 设计目标

- 封装**跨模块整车集成场景**（CAN 网关、诊断路由、PDU 打包、变体复用）。
- 沉淀**企业规范与工程最佳实践**为可复用、可评审、可单测的代码。
- 直接对齐**整车功能需求**，是 IaC 核心价值载体。

### 4.2 Feature 模板：声明式 + 组合

推荐「**声明式数据 → Feature 函数 → ARXML 输出**」三段式，让业务规范脱离脚本细节：

```python
# —— 企业业务资产（L3 示意，纳入 Git）——

# 1) 业务规范（声明式，可评审、可 diff）
CAN_GATEWAY_SPEC = {
    "cluster": "CAN1",
    "rx_signals": ["VehicleSpeed", "EngineRpm"],
    "rx_ecu": "BCM",
    "tx_cluster": "CAN2",
    "pack_rule": "company_pdu_packing_v3",   # 企业 PDU 打包规范版本
}

# 2) Feature 函数：编排 L1 + L2，落地企业规范
def build_can_gateway(spec: dict, out_arxml: str):
    AF.reinit()                                       # 任务边界：干净起点
    AF.new_file(out_arxml, defaultArPackage="GW", overWrite=True)
    root_pkg = AF.get_root().get_arPackages()[0]

    # 编排多个模块（L2 封装 + L1 原子操作）
    canif = CanIfConfigurator(root_pkg, os_def_path="/CanIf")
    for sig in spec["rx_signals"]:
        canif.add_rx_signal(sig, cluster=spec["cluster"], ecu=spec["rx_ecu"])
    apply_pdu_packing(canif, rule=spec["pack_rule"])  # 企业打包规则
    route_to_cluster(canif, target=spec["tx_cluster"])

    AF.save()                                         # 增量回写（脏标记驱动）
    return out_arxml
```

### 4.3 企业规范沉淀机制

| 机制 | 做法 |
|---|---|
| 规范版本化 | 业务规则（如 `company_pdu_packing_v3`）以**代码 + 版本号**纳入 Git，可追溯、可评审 |
| 模板库 | 高频整车场景（网关、诊断路由、NM、E2E 保护）沉淀为 Feature 模板，新车型 fork 即用 |
| 变体复用 | 同一 Feature 函数按车型变体参数化，避免多车型重复手工配置 |
| 合规闸门 | Feature 末尾内置 `assert_compliant()`，输出即合规 |

### 4.4 L3 编写规范（厂商约束）

1. **入口必做 `reinit()` 边界管理**（避免污染）。
2. **同型元素分置不同 ARPackage**；同父下不得重名（沿用 L1 约束）。
3. **枚举值用裸表达式** `AF.EnumName.VALUE_*`，不写魔法字符串。
4. **不得绕过 L1 setter 直改 lxml**（破坏脏传播与 schema 顺序）。
5. **跨文件引用**用 `set_<x>(node)` 传节点对象，而非拼路径字符串。

---

## 5. AI 赋能：MCP 服务集成（IaC 的未来演进，已部分落地）

> 输入文档「未来方向 ①：AI 赋能集成」——AutosarFactory 已通过 MCP 服务**先行落地**了 Agent 工具链。

### 5.1 现有 MCP 工具（14 个，三组）

| 组 | 工具 | 作用 |
|---|---|---|
| 知识检索（可选） | `search_autosar_knowledge` | 语义检索 AUTOSAR 规范 PDF（需构建 KB） |
| API 参考（必装） | `get_class` / `find_creators_of` / `find_creation_chain` / `search_classes` / `get_enum` / `list_enums` | 让 Agent **永不猜测 API 名**，按链查找创建路径与签名 |
| 系统建模地图 | `get_system_element_map` | 按用例（S-R、CAN、SOME/IP、DEXT、数据类型…）列出所需元素 |
| ECUC 配置 | `list_ecuc_modules` / `get_ecuc_module` / `get_ecuc_container` / `get_ecuc_param_literals` | BSW 模块参数定义查询 |
| 脚手架 | `new_file_script_template` / `read_file_script_template` | 生成新建/改写脚本骨架 |

### 5.2 Agent 强制工作流（MCP `_INSTRUCTIONS` 已固化）

```
get_system_element_map(用例)            # ① 定向：该用例需要哪些元素
   ↓
find_creation_chain(元素类型)           # ② 发现创建路径（一次调用取全链）
   ↓
get_class(类, section=references|attributes)  # ③ 按需取引用/属性签名
   ↓
get_enum(枚举名)                        # ④ 取合法枚举值（不臆测）
   ↓
new_file_script_template(...)           # ⑤ 先拿脚手架，再填业务代码
   ↓
AF.save() + 可视化 walker               # ⑥ 输出 + ASCII/Mermaid 自检
```

### 5.3 「自然语言需求 → 可执行集成代码」的落点

- L1 元模型 + L2 封装的**命名规律性**（严格 get_/set_/new_ 约定）是 Agent 可靠生成代码的前提。
- MCP 把 API 参考做成**查询接口**而非让模型记忆，规避了 440k 行元模型的「记忆不可靠」问题。
- **演进**：当前 Agent 生成 L1/L2 级代码；未来可训练 Agent 直接识别并调用**企业 L3 Feature 库**，实现「需求 → Feature 编排」。

---

## 6. IaC + CP 协同工作流（对接输入文档第 6 章）

```
        ┌──────────────────────────┐
需求/规范│ ① IaC 自动生成 ARXML        │  L3 Feature 脚本（Git 管理）
────────▶│  AF.reinit → 编排 L1/L2 → save│  批量、标准统一
        └─────────────┬────────────┘
                      ▼ .arxml
        ┌──────────────────────────┐
        │ ② CP 工具人工走查 + 内置校验 │  可视化审查、问题前置
        └─────────────┬────────────┘
                      ▼ 确认
        ┌──────────────────────────┐
        │ ③ CP 生成 C 代码 + 工程文件  │  编译 / 整车集成 / 版本管控
        └──────────────────────────┘
```

**与 AutosarFactory 的契合点**：

- ① 完全由本设计的 L1/L2/L3 承担；`save()` 增量回写、schema 合规保证输出可被 CP 工具直接吃进。
- 多文件/可分裂合并能力，使「按模块拆分文件 → CP 按模块导入」天然对齐。
- MCP 的可视化 walker（ASCII/Mermaid）可在 ① 与 ② 之间做**机器侧自检**，减少 CP 走查返工。

---

## 7. CI/CD 适配

AutosarFactory 的「无业务语义 + 状态显式 + 单进程单模型」天然适配流水线：

```yaml
# 伪代码：CI 流水线一个车型变体的集成任务
- run: |
    python integrate_vehicle.py --variant SUV_H7 \
           --spec configs/suv_h7.yaml \
           --out   out/suv_h7.arxml
    python validate_arxml.py out/suv_h7.arxml      # L2 约束校验
```

**设计要求**：

1. 每个 CI 任务 = **独立进程** = 一次 `reinit()` 边界；禁止同进程跨车型复用全局态。
2. L3 Feature 输出**幂等**（同输入同输出），便于 diff 评审与缓存。
3. 失败快速失败：L2 约束校验、L3 合规闸门任一不通过即阻断流水线。

---

## 8. 权责划分与扩展机制

### 8.1 权责边界（对接输入文档第 5 章）

| 角色 | 提供 | 性质 |
|---|---|---|
| **厂商** | L1 底座（固化）、L2 标准 Configurator 族、通用 Demo、MCP 工具、API 参考 DB | 标准化「积木底座」，不可由用户修改 |
| **企业** | L3 Feature 库、企业规范（PDU 打包/诊断路由/变体复用规则）、私有脚本 | Git 版本管理，团队协作，自主迭代 |

### 8.2 扩展机制

- **新 BSW 模块支持**：重建 `ecuc_param_def.json`（`paramDefDbBuilder/ecuc_module_def_to_json.py`）+ 新增对应 L2 Configurator；L1/L3 无需改动。
- **新 AUTOSAR Release**：顶层包已固定 R24-11；旧版直接 `import autosar_releases.autosar422` 等冻结快照建模（**无运行时切换**，按需导入）。
- **新 Feature**：纯 L3 工作，不动底座；通过 Git 评审纳入资产库。

### 8.3 API 参考的可维护性

`af_api_reference.json` 是 Agent 的**单一事实源**，由 `kb_builder/` 重建。任何 L1 元模型升级 → 重建参考 DB → Agent 自动获得新 API，**无需改 Prompt**。

---

## 9. 端到端示例（真实 API 范式）

下列范式取自 `Examples/create_autosar_basic_communication.py`，体现三层如何协作（此处以 L1 原子操作直述，L2/L3 在其上封装）：

```python
import autosarfactory.autosarfactory as AF

AF.reinit()
AF.new_file("dataTypes.arxml", defaultArPackage="Types", overWrite=True)
types = AF.get_node("/Types")

# —— 数据类型（L1 原子操作）——
bt = types.new_SwBaseType("uint8").new_BaseTypeDirectDefinition()
bt.set_baseTypeEncoding("") ; bt.set_nativeDeclaration("uint8")
bt.set_memAlignment(8)      ; bt.set_baseTypeSize(8)

idt = types.new_ImplementationDataType("uint8").new_SwDataDefProps().new_SwDataDefPropsVariant()
idt.set_baseType(AF.get_node("/Types/uint8"))

# —— 接口 + SWC + 端口（L1，跨文件引用传节点对象）——
AF.new_file("components.arxml", defaultArPackage="Swcs", overWrite=True)
swcs = AF.get_node("/Swcs")
sri = swcs.new_SenderReceiverInterface("srif1")
sri.new_DataElement("de1").set_type(AF.get_node("/Types/uint8"))

asw = swcs.new_ApplicationSwComponentType("asw1")
asw.new_PPortPrototype("outPort").set_providedInterface(sri)

# —— 枚举用裸表达式 ——
sig = swcs.new_ISignal("sig1")
sig.set_dataTypePolicy(AF.DataTypePolicyEnum.VALUE_LEGACY)
sig.set_iSignalType(AF.ISignalTypeEnum.VALUE_PRIMITIVE)

AF.save()                       # 增量回写（仅变更子树）
AF.saveAs("merged.arxml", overWrite=True)   # 合并为单文件
```

> L2 封装后，上述将简化为 `TypesBuilder.uint8(...)` / `SwcBuilder.with_sender_port(...)`；L3 进一步封装为 `build_basic_communication(...)`。**底座不变，抽象递进。**

---

## 10. 测试与质量策略

复用并扩展现有 `tests/test_autosarmodel.py` 契约（38 个用例 + `teardown/reinit` 模式）：

| 层 | 测试重点 | 现状 |
|---|---|---|
| L1 | 增删改查、save 增量、多文件合并、schema 顺序、引用双向、悬空容错 | **已有 38 用例**，作为底座回归基线 |
| L2 | 各 Configurator 的参数约束、必填、枚举合法值、definition 绑定 | **待建**；每个 Builder 一组单测 |
| L3 | Feature 幂等性、企业规范合规闸门、跨模块一致性 | **待建**；Feature 输出做 round-trip 断言 |

**硬性约束**：任何层的新测试**必须**在 `teardown` 调 `AF.reinit()`，否则污染全局态（沿用现有规范）。

---

## 11. 演进路线

| 阶段 | 目标 | 关键交付 |
|---|---|---|
| **P0 底座固化**（已完成） | L1 稳定 + MCP 工具就绪 | 440k 元模型、脏标记保存、多文件合并、14 个 MCP 工具、API 参考 DB |
| **P1 封装层补齐** | L2 标准化 | Definition Registry + 主流 BSW Configurator（Os/CanIf/Can/Rte）+ 约束校验器 |
| **P2 业务层示范** | L3 范式 | 2~3 个标杆 Feature（CAN 网关、诊断路由、PDU 打包）模板 + 企业规范沉淀指南 |
| **P3 AI 赋能** | 需求 → 代码 | 构建 KB（`kb_builder`）激活语义检索；Agent 调用 L3 Feature 库做编排 |
| **P4 资产生态** | 行业共建 | 可复用 Feature 资产库、跨项目共享最佳实践 |

---

## 12. 设计要点小结

1. **三层严格隔离，依赖只向下**：L1 无业务、L2 单模块、L3 跨模块；厂商固化底座，企业自建业务。
2. **顺应而非绕过 L1 机制**：脏标记、累积合并、schema 顺序、双向引用 —— L2/L3 必须**只用 setter**，否则破坏保存正确性。
3. **数据驱动 L2**：以 `ecuc_param_def.json` 为单一事实源，约束随定义重建而更新。
4. **L3 沉淀企业资产**：声明式规范 + Feature 函数 + 合规闸门，纳入 Git 可评审。
5. **AI 已先行落地**：MCP 把 API 参考做成查询接口，规避元模型记忆不可靠；未来 Agent 直接编排 L3 Feature。
6. **单进程单模型**：CI 并发用多进程 + `reinit()` 边界，禁止同进程跨模型复用全局态。

---

> 本设计为 v0.1，后续随 P1 L2 Configurator 实际落地迭代细化接口签名与约束清单。
