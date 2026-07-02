# ARXML 命令行（CLI）工具开发设计方案

> 基于 autosarfactory 项目，设计一套声明式、管道化的命令行工具，实现一行命令操作 ARXML 元素与属性

**日期**：2026-06-30

**参考输入**：[AUTOSAR Integration-as-Code (IaC) 技术方案 v0.1](../cai-input/AUTOSAR%20Integration-as-Code%20(IaC)%20技术方案0.1.md)

---

## 目录

1. [设计动机与场景](#1-设计动机与场景)
2. [CLI 架构总览](#2-cli-架构总览)
3. [核心命令设计](#3-核心命令设计)
4. [声明式输入与输出](#4-声明式输入与输出)
5. [进阶场景与管道组合](#5-进阶场景与管道组合)
6. [实现方案](#6-实现方案)
7. [开发路线](#7-开发路线)
8. [风险与应对](#8-风险与应对)
9. [总结](#9-总结)

---

## 1. 设计动机与场景

### 1.1 为什么需要 CLI

autosarfactory 当前提供的是 Python API，用户必须编写脚本才能操作 ARXML。在以下场景中存在明确痛点：

| 场景 | 痛点 | CLI 方案 |
|------|------|---------|
| **CI/CD 流水线** | 每次操作都需编写维护 Python 脚本 | 一行命令嵌入 Shell/YAML pipeline |
| **快速巡检** | 想查一个信号配置，要打开 Python REPL | `af get /Can/signals/sig1` 秒级返回 |
| **批量修改** | 改 50 个信号的属性要写循环脚本 | 管道 + `af set` 批量应用 |
| **跨团队协作** | 非 Python 开发者（系统工程师、测试）使用门槛高 | CLI 零编程，YAML 配置文件即可 |
| **Git diff/review** | ARXML 是 XML 格式，diff 难读 | `af diff` 输出结构化差异 |
| **自动化测试** | 需要快速做"添加-校验-删除" | Shell 脚本串联 CLI 命令 |

### 1.2 设计目标

1. **一行命令完成增删改查**：无 Python 编程基础也能操作 ARXML
2. **声明式而非命令式**：用户表达"想要什么"，而非"怎么做"
3. **管道化可组合**：`af list | af get | af set` 链式处理
4. **与 autosarfactory Python API 等效**：CLI 是 Python API 的薄封装，能力对齐
5. **CI/CD 友好**：退出码明确，支持 `--json`/`--yaml` 输出，可嵌入任意流水线

---

## 2. CLI 架构总览

### 2.1 命令体系

```
af
├── read         读取 ARXML 文件到内存
├── save         保存修改到文件
├── reinit       重置状态
│
├── get          读取元素/属性（核心读操作）
├── set          设置元素/属性（核心写操作）
├── create       创建新元素
├── delete       删除元素
│
├── list         列出子元素
├── query        条件查询（类型/名称/属性过滤）
├── tree         展示模型树（文本版 autosar_ui）
│
├── diff         比较两个 ARXML 的差异
├── validate     校验 ARXML 合规性
├── export       导出指定元素到文件
├── merge        合并多个 ARXML 文件
│
├── info         显示元素元信息（类型、路径、文件来源）
└── config       CLI 全局配置（默认路径、输出格式等）
```

### 2.2 全局选项

```
af [全局选项] <命令> [命令选项] [参数...]

全局选项：
  -f, --file <PATH>        预加载的 ARXML 文件（可多次指定）
  -d, --dir <PATH>         预加载的 ARXML 目录（递归扫描）
  -o, --output <FORMAT>    输出格式：table | json | yaml | csv | path
  --no-color               禁用颜色输出
  --dry-run                模拟执行，不实际写入文件
  -q, --quiet              仅输出结果，抑制日志
  -v, --verbose            详细日志
```

### 2.3 设计原则

| 原则 | 说明 |
|------|------|
| **路径即标识** | ARXML 路径（`/Swcs/asw1/outPort`）作为元素的主要寻址方式 |
| **命令语义自明** | `af get` 读、`af set` 写，直觉化 |
| **管道友好** | 每条命令都可从 stdin 读取输入、向 stdout 输出 |
| **幂等性** | `af set`、`af create` 重复执行结果一致 |
| **最小惊奇** | 不自动保存；显示 `--dry-run` 预览变更 |

---

## 3. 核心命令设计

### 3.1 `af read` — 加载 ARXML

```bash
# 加载单个文件
af read input.arxml

# 加载多个文件
af read ecu1.arxml ecu2.arxml datatypes.arxml

# 加载整个目录
af read ./project_config/

# 等效于
af -f ecu1.arxml -f ecu2.arxml get /Swcs/asw1
```

输出：
```
✓ 已加载 3 个文件，共 142 个 AR-PACKAGE，8721 个元素
  - ecu1.arxml (3 packages)
  - ecu2.arxml (5 packages)
  - datatypes.arxml (1 package)
```

### 3.2 `af get` — 读取元素/属性

```bash
# 获取元素的所有属性（表格输出）
af get /Swcs/asw1

# 输出：
# ┌─────────────────────┬───────────────────┐
# │ 属性                 │ 值                │
# ├─────────────────────┼───────────────────┤
# │ path                │ /Swcs/asw1        │
# │ type                │ ApplicationSwComp│
# │ SHORT-NAME          │ asw1              │
# │ CATEGORY            │ APPLICATION       │
# │ UUID                │ abc123...         │
# │ file                │ components.arxml  │
# ├─────────────────────┼───────────────────┤
# │ 子元素 (5)          │                   │
# │  - outPort          │ PPortPrototype    │
# │  - InternalBehavior │ be1               │
# │  ...                                    │
# └─────────────────────┴───────────────────┘

# 获取单个属性值（纯文本输出，管道友好）
af get /Swcs/asw1 --prop CATEGORY
# → APPLICATION

# 获取单个属性值（JSON 输出）
af get /Swcs/asw1 --prop CATEGORY -o json
# → {"path": "/Swcs/asw1", "attribute": "CATEGORY", "value": "APPLICATION"}

# 获取嵌套属性
af get /Swcs/asw1/InternalBehavior/beh1 --prop CAN_ENTER_EXCLUSIVE_AREA
# → true

# 获取所有子元素列表
af get /Swcs/asw1 --children
# → PPortPrototype outPort
# → RPortPrototype (none)
# → InternalBehavior beh1
# → ...

# 获取引用目标详情
af get /Swcs/asw1/outPort --ref providedInterface
# → /Interfaces/srif1  (SenderReceiverInterface)
```

### 3.3 `af set` — 设置属性/引用

```bash
# 设置属性值
af set /Swcs/asw1 --prop CATEGORY=SENSOR_ACTUATOR

# 设置多个属性
af set /Swcs/asw1 \
    --prop CATEGORY=APPLICATION \
    --prop UUID=auto

# 设置引用
af set /Swcs/asw1/outPort --ref providedInterface=/Interfaces/srif1

# 从 YAML 文件批量设置
af set /Swcs/asw1 --from config.yaml

# config.yaml:
# CATEGORY: APPLICATION
# ADMIN-DATA:
#   SDGS:
#     - GID: "project-x"
#       SD:
#         - GID: "version"
#           VALUE: "1.0"
#         - GID: "author"
#           VALUE: "team-a"

# 管道输入模式
af list /Can/signals --type ISignal | af set --prop LENGTH=16
# ↑ 将 /Can/signals 下所有 ISignal 的 LENGTH 设置为 16

# 预览变更
af set /Swcs/asw1 --prop CATEGORY=SENSOR_ACTUATOR --dry-run
# → [DRY-RUN] 将设置 /Swcs/asw1.CATEGORY = SENSOR_ACTUATOR
# → [DRY-RUN] 将影响 1 个元素，1 个属性
# → [DRY-RUN] 关联引用：0 个
```

### 3.4 `af create` — 创建元素

```bash
# 创建 AR-PACKAGE
af create / --type ARPackage --name MyNewPackage

# 创建 ApplicationSwComponentType
af create /MyNewPackage --type ApplicationSwComponentType --name asw3

# 创建 Port（含自动设置引用）
af create /MyNewPackage/asw3 \
    --type PPortPrototype \
    --name outPort2 \
    --ref providedInterface=/Interfaces/srif1

# 从 YAML 创建复杂层级
af create / --from create_spec.yaml

# create_spec.yaml:
# - path: /
#   type: ARPackage
#   name: CanConfig
#   children:
#     - type: ARPackage
#       name: signals
#       children:
#         - type: ISignal
#           name: sig_engine_speed
#           props:
#             LENGTH: 16
#             DATA_TYPE_POLICY: LEGACY
#             I_SIGNAL_TYPE: PRIMITIVE
#           refs:
#             systemSignal: /Can/systemsignals/EngineSpeed_syssig
#         - type: ISignal
#           name: sig_engine_temp
#           props:
#             LENGTH: 8
#     - type: ARPackage
#       name: ecus
#       children:
#         - type: EcuInstance
#           name: EngineECU
#           props:
#             WAKE_UP_OVER_BUS_SUPPORTED: true
#             SLEEP_MODE_SUPPORTED: false

# 全量预览
af create / --from create_spec.yaml --dry-run
# → [DRY-RUN] 将创建 1 个 ARPackage, 2 个子 ARPackage
# → [DRY-RUN] 将创建 1 个 EcuInstance
# → [DRY-RUN] 将创建 2 个 ISignal
# → [DRY-RUN] 将设置 3 个引用
# → [DRY-RUN] 总计：7 个元素，18 个属性变更
```

### 3.5 `af delete` — 删除元素

```bash
# 删除指定元素
af delete /Swcs/asw1/outPort

# 删除前确认
af delete /Swcs/asw1/outPort --confirm
# → ⚠ 将删除 /Swcs/asw1/outPort (PPortPrototype)
# → ⚠ 以下引用将失效：
# →    - /Can/system/Mappings/outportToSig1Mapping 引用了此元素
# → 确认删除？[y/N] y
# → ✓ 已删除 /Swcs/asw1/outPort

# 强制删除（跳过确认）
af delete /Swcs/asw1/outPort --force

# 批量删除（管道输入）
af list /Can/signals --type ISignal --prop LENGTH=0 | af delete --force

# 删除整个包（递归）
af delete /CanConfig --recursive --confirm
# → ⚠ 将递归删除 /CanConfig 及其下 47 个子元素
# → ⚠ 影响文件：can_config.arxml
# → 确认删除？[y/N]
```

### 3.6 `af list` — 列出子元素

```bash
# 列出所有子元素
af list /Swcs/asw1

# 输出：
# ┌────────────────────────┬──────────────────────────┬───────┐
# │ 名称                    │ 类型                      │ 路径  │
# ├────────────────────────┼──────────────────────────┼───────┤
# │ outPort                │ PPortPrototype           │ /Swcs/│
# │ InternalBehavior       │ be1                      │ /Swcs/│
# │ Runnable_1             │ RunnableEntity           │ /Swcs/│
# │ ...                                                │
# └────────────────────────┴──────────────────────────┴───────┘

# 按类型过滤
af list /Swcs --type ApplicationSwComponentType

# 按名称模式过滤
af list /Can/signals --name "sig_*"

# 输出纯路径（管道友好）
af list /Swcs/asw1 -o path
# → /Swcs/asw1/outPort
# → /Swcs/asw1/InternalBehavior.be1
# → /Swcs/asw1/InternalBehavior.be1/TimingEvent.te_5ms
# → ...

# 列出所有引用关系
af list /Swcs/asw1/outPort --refs
# → providedInterface → /Interfaces/srif1
```

### 3.7 `af query` — 高级查询

```bash
# 全局搜索指定类型
af query --type ISignal

# 条件过滤
af query --type ISignal --where "LENGTH >= 16"

# 带属性投影
af query --type ISignal --select NAME,LENGTH,SYSTEM_SIGNAL_REF -o table

# 搜索包含特定引用的元素
af query --type PPortPrototype --has-ref "providedInterface=/Interfaces/srif1"

# 正则名称匹配
af query --name-regex "sig_engine_.*" -o json

# 输出示例 (JSON)：
# [
#   {"path":"/Can/signals/sig_engine_speed","type":"ISignal","LENGTH":16},
#   {"path":"/Can/signals/sig_engine_temp","type":"ISignal","LENGTH":8}
# ]
```

### 3.8 `af diff` — 差异比较

```bash
# 比较两个 ARXML 文件
af diff ecu_v1.arxml ecu_v2.arxml

# 输出：
# ── 文件: ecu_v1.arxml vs ecu_v2.arxml ──
# + /Can/signals/sig_new           (新增 ISignal)
# ~ /Can/signals/sig_engine_speed  (修改)
#   LENGTH: 8 → 16
# ~ /Swcs/asw1                      (修改)
#   CATEGORY: APPLICATION → SENSOR_ACTUATOR
# - /Can/signals/sig_deprecated     (删除 ISignal)
#
# 汇总: +1 新增, ~2 修改, -1 删除

# 比较两个已加载的模型状态
af diff --from-loaded ecu1.arxml ecu2.arxml

# 结构化输出（适合 CI/CD）
af diff v1.arxml v2.arxml -o json
```

### 3.9 `af validate` — 校验

```bash
# 基础校验
af validate output.arxml

# 指定规则集
af validate output.arxml --rules schema,references,naming

# 输出：
# ✓ Schema 校验通过 (AUTOSAR_00053.xsd)
# ✓ 引用完整性: 所有 2341 个引用可解析
# ⚠ 命名规范: 发现 3 个命名问题
#   - /Can/signals/Signal_1: 建议使用 lowerCamelCase
#   - /Swcs/ASW_MAIN: 建议使用 lowerCamelCase
# ✗ 一致性: /Can/signals/sig1.LENGTH(8) 与关联 SystemSignal.dynamicLength(TRUE) 不一致
```

### 3.10 `af tree` — 树状展示

```bash
# 完整树
af tree /Swcs

# 输出：
# /Swcs
# ├── ApplicationSwComponentType: asw1
# │   ├── PPortPrototype: outPort
# │   │   └── ref providedInterface → /Interfaces/srif1
# │   └── InternalBehavior: beh1
# │       ├── TimingEvent: te_5ms
# │       └── RunnableEntity: Runnable_1
# │           └── DataSendPoint: dsp
# ├── ApplicationSwComponentType: asw2
# │   ├── RPortPrototype: inPort
# │   └── InternalBehavior: beh1
# └── CompositionSwComponentType: Comp
#     ├── Component: asw1_proto
#     ├── Component: asw2_proto
#     └── AssemblySwConnector: conn1

# 限制深度
af tree /Swcs/asw1 --depth 2

# 仅显示类型名
af tree / --types-only
```

---

## 4. 声明式输入与输出

### 4.1 YAML 配置规范

CLI 支持通过 YAML 文件描述复杂的创建/修改意图，实现声明式配置：

```yaml
# can_network_config.yaml —— 声明式描述一个 CAN 网络配置
version: "1.0"
description: "CAN 网络配置 - 动力域"

operations:
  - action: create
    path: /
    type: ARPackage
    name: CanNetwork
    
  - action: create
    path: /CanNetwork
    type: ARPackage
    name: signals
    
  - action: create
    path: /CanNetwork/signals
    type: ISignal
    name: sig_engine_rpm
    properties:
      LENGTH: 16
      DATA_TYPE_POLICY: LEGACY
      I_SIGNAL_TYPE: PRIMITIVE
    references:
      systemSignal: /CanNetwork/systemsignals/EngineRPM_syssig

  - action: set
    path: /Swcs/asw1
    properties:
      CATEGORY: APPLICATION

  - action: delete
    path: /CanNetwork/signals/sig_deprecated
    force: true
```

```bash
# 批量执行声明式配置
af apply can_network_config.yaml

# 预览
af apply can_network_config.yaml --dry-run

# 仅执行特定操作组
af apply can_network_config.yaml --filter "action=create"
```

### 4.2 JSON 输出模式

所有命令支持 `-o json` 输出，适用于 CI/CD 管道和程序消费：

```bash
# 获取所有信号信息
af query --type ISignal -o json

# 输出：
# {
#   "count": 47,
#   "items": [
#     {
#       "path": "/Can/signals/sig_engine_speed",
#       "type": "ISignal",
#       "file": "can_config.arxml",
#       "properties": {
#         "LENGTH": 16,
#         "DATA_TYPE_POLICY": "LEGACY",
#         "I_SIGNAL_TYPE": "PRIMITIVE"
#       },
#       "references": {
#         "systemSignal": "/Can/systemsignals/EngineSpeed_syssig"
#       }
#     }
#   ]
# }
```

### 4.3 管道协议

CLI 支持基于 JSONL（JSON Lines）的管道协议，实现命令链式调用：

```bash
# 管道示例：查找所有长度 > 8 的信号并改为 16
af query --type ISignal --where "LENGTH > 8" -o path \
  | af set --prop LENGTH=16 --dry-run

# 管道示例：批量导出特定类型元素
af query --type EcuInstance -o path \
  | xargs -I {} af export {} --output-dir ./ecu_extracts/
```

管道协议格式（JSONL）：

```jsonl
{"path": "/Can/signals/sig1", "type": "ISignal"}
{"path": "/Can/signals/sig2", "type": "ISignal"}
```

CLI 命令可以通过 `--from-stdin` 接收管道输入：

```bash
# 从 stdin 读取目标路径列表
cat targets.txt | af set --from-stdin --prop CATEGORY=APPLICATION
```

---

## 5. 进阶场景与管道组合

### 5.1 CI/CD 集成示例

**GitHub Actions 示例**：

```yaml
# .github/workflows/validate_arxml.yml
name: ARXML Validation

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install autosarfactory
        run: pip install autosarfactory[cli]
      
      - name: Validate ARXML
        run: |
          af read ./config/*.arxml
          af validate --rules schema,references,consistency -o json > validation_report.json
      
      - name: Diff against baseline
        run: |
          af diff baseline.arxml current.arxml -o json > diff_report.json
      
      - name: Upload reports
        uses: actions/upload-artifact@v4
        with:
          name: arxml-reports
          path: |
            validation_report.json
            diff_report.json
```

**Jenkins Pipeline 示例**：

```groovy
pipeline {
    agent any
    stages {
        stage('Generate ARXML') {
            steps {
                sh '''
                    af -f base_config.arxml apply variant_a_features.yaml
                    af save output/variant_a.arxml
                '''
            }
        }
        stage('Validate') {
            steps {
                sh 'af validate output/variant_a.arxml --strict'
            }
        }
    }
}
```

### 5.2 日常操作场景

**场景 1：快速巡检信号配置**

```bash
af -f can_config.arxml query --type ISignal \
    --select NAME,LENGTH,SYSTEM_SIGNAL_REF -o table
```

**场景 2：批量修改信号属性**

```bash
af -f can.arxml list /Can/signals --type ISignal -o path \
  | af set --prop DATA_TYPE_POLICY=LEGACY --from-stdin
af save
```

**场景 3：从模板创建 ECU 配置**

```bash
# 使用变量替换 + YAML 模板
ECU_NAME=EngineECU af apply --template ecu_template.yaml
```

模板 YAML：
```yaml
# ecu_template.yaml
operations:
  - action: create
    path: /Ecus
    type: EcuInstance
    name: ${ECU_NAME}
    properties:
      WAKE_UP_OVER_BUS_SUPPORTED: true
```

**场景 4：清理无效引用**

```bash
af -f project.arxml query --broken-refs -o path \
  | af delete --force --from-stdin
af save project.arxml
```

**场景 5：提取子集配置**

```bash
# 导出动力域所有 ECU 配置
af -f full_config.arxml \
    list /Ecus --name "*Engine*" -o path \
  | xargs -I {} af export {} --preserve-hierarchy --output-dir ./powertrain/
```

### 5.3 Shell 脚本编排

```bash
#!/bin/bash
# gen_variant.sh —— 生成软件变体配置

set -e

VARIANT=$1
OUTPUT="output/${VARIANT}.arxml"

echo "=== 加载基础配置 ==="
af read base/datatypes.arxml base/interfaces.arxml

echo "=== 应用变体 ${VARIANT} ==="
af read "variants/${VARIANT}.arxml"

echo "=== 应用业务规则 ==="
af apply "rules/${VARIANT}_rules.yaml"

echo "=== 校验 ==="
if ! af validate --strict; then
    echo "ERROR: 校验失败"
    exit 1
fi

echo "=== 导出 ==="
af saveAs "${OUTPUT}"

echo "=== 生成差异报告 ==="
af diff "baseline/${VARIANT}.arxml" "${OUTPUT}" -o json > "reports/${VARIANT}_diff.json"

echo "✓ 变体 ${VARIANT} 生成完成 → ${OUTPUT}"
```

---

## 6. 实现方案

### 6.1 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                     用户 / CI/CD Pipeline                     │
│         af get / af set / af create / af delete / ...        │
├──────────────────────────────────────────────────────────────┤
│                    CLI 命令层 (cli/)                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │ commands/ │ │ parsers/ │ │ formatter│ │ shell_complete│   │
│  │ get.py    │ │ path.py  │ │ json.py  │ │ af_complete.sh│   │
│  │ set.py    │ │ yaml.py  │ │ table.py │ │               │   │
│  │ create.py │ │ query.py │ │ diff.py  │ │               │   │
│  │ delete.py │ │ stdin.py │ │ tree.py  │ │               │   │
│  │ list.py   │ │          │ │          │ │               │   │
│  │ query.py  │ │          │ │          │ │               │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
├──────────────────────────────────────────────────────────────┤
│                    服务层 (cli/services/)                     │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │ SessionManager   │  │ DiffEngine       │                  │
│  │ - load/unload    │  │ - structural diff│                  │
│  │ - state tracking │  │ - semantic diff  │                  │
│  │ - auto reinit    │  │ - report gen     │                  │
│  └──────────────────┘  └──────────────────┘                  │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │ ValidationEngine │  │ QueryEngine      │                  │
│  │ - schema rules   │  │ - path/type/name │                  │
│  │ - ref integrity  │  │ - condition eval │                  │
│  │ - naming rules   │  │ - projection     │                  │
│  └──────────────────┘  └──────────────────┘                  │
├──────────────────────────────────────────────────────────────┤
│              autosarfactory Python API (不变)                 │
│  autosarfactory.read/save/get_node/new_<Element>/set_<attr>  │
│  XmlElementDirtyTracker / datatype_utils                     │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 技术选型

| 组件 | 选型 | 理由 |
|------|------|------|
| **CLI 框架** | `click` 或 `argparse` | Python 标准库首选；`click` 更好的嵌套子命令支持 |
| **YAML 解析** | `PyYAML` | Python YAML 事实标准 |
| **表格渲染** | `tabulate` 或自研 | 轻量终端表格输出 |
| **差异比较** | 自研（基于 lxml 结构对比） | 需要 AUTOSAR 语义感知的 diff |
| **输出格式** | `json` 标准库 / `PyYAML` | 无需额外依赖 |
| **Shell 补全** | `click` 内置 + bash/zsh 脚本 | 一键生成补全脚本 |

### 6.3 核心模块设计

#### 6.3.1 `SessionManager` — 会话管理

```python
# cli/services/session.py

class SessionManager:
    """
    管理 autosarfactory 全局状态的 CLI 会话。
    
    职责：
    - 处理 -f/-d 参数的文件加载
    - 跟踪已加载文件列表
    - 在操作完成后管理是否自动保存
    - 确保状态隔离
    """
    
    def __init__(self):
        self._loaded_files = []
        self._auto_save = False
        self._dry_run = False
    
    def load(self, files: list[str]) -> None:
        """加载 ARXML 文件，自动 reinit + read"""
        autosarfactory.reinit()
        if files:
            autosarfactory.read(files)
            self._loaded_files = files
    
    def reload(self) -> None:
        """重新加载所有文件（用于撤销）"""
        self.load(self._loaded_files)
    
    def close(self):
        """清理：如果 auto_save 开启则保存"""
        if self._auto_save and not self._dry_run:
            autosarfactory.save(self._loaded_files)
        autosarfactory.reinit()
    
    @property
    def root(self):
        return autosarfactory.get_root()
```

#### 6.3.2 `QueryEngine` — 查询引擎

```python
# cli/services/query.py

class QueryEngine:
    """
    查询执行器。
    将 CLI 的查询参数翻译为 autosarfactory API 调用。
    """
    
    def query(self, 
              root_path: str = '/',
              element_type: str = None,
              name_pattern: str = None,
              where_clause: str = None,
              projection: list[str] = None) -> list[dict]:
        """
        执行查询并返回结果列表。
        
        where_clause 语法（简化版）：
        - "LENGTH >= 16"
        - "CATEGORY = APPLICATION"
        - "name LIKE sig_*"
        - "has-ref providedInterface"
        """
        results = []
        root = autosarfactory.get_node(root_path)
        if root is None:
            return results
        
        # 类型过滤
        if element_type:
            klass = getattr(autosarfactory, element_type, None)
            if klass:
                candidates = autosarfactory.get_all_instances(root, klass)
            else:
                return results
        else:
            candidates = self._traverse(root)
        
        for node in candidates:
            item = self._node_to_dict(node)
            if self._match(item, name_pattern, where_clause):
                if projection:
                    item = {k: v for k, v in item.items() if k in projection}
                results.append(item)
        return results
    
    def _node_to_dict(self, node) -> dict:
        """将 AutosarNode 转为字典"""
        return {
            'path': node.autosar_path or node.path,
            'type': node.__class__.__name__,
            'name': node.name,
            'file': node.file,
        }
```

#### 6.3.3 CLI 命令入口 — `af get` 示例

```python
# cli/commands/get.py

import click
from cli.services.session import get_session

@click.command('get')
@click.argument('path', required=True)
@click.option('--prop', '-p', help='属性名，不指定时输出全部属性')
@click.option('--ref', '-r', help='引用名，不指定时输出全部引用')
@click.option('--children', '-c', is_flag=True, help='列出子元素')
@click.option('--output', '-o', type=click.Choice(['table','json','yaml','csv','text']), default='table')
@click.pass_context
def get_command(ctx, path, prop, ref, children, output):
    """读取 ARXML 元素或属性"""
    session = get_session()
    node = autosarfactory.get_node(path)
    
    if node is None:
        click.echo(f"错误：路径 '{path}' 不存在", err=True)
        raise SystemExit(1)
    
    result = {}
    
    if prop:
        # 获取单个属性
        getter_name = f'get_{prop}'
        getter = getattr(node, getter_name, None)
        if getter:
            result = {'path': path, 'attribute': prop, 'value': getter()}
        else:
            click.echo(f"错误：属性 '{prop}' 在 {path} 上不存在", err=True)
            raise SystemExit(1)
    elif ref:
        # 获取引用
        getter_name = f'get_{ref}'
        getter = getattr(node, getter_name, None)
        if getter:
            val = getter()
            result = {'path': path, 'reference': ref, 'target': str(val) if val else None}
        else:
            click.echo(f"错误：引用 '{ref}' 在 {path} 上不存在", err=True)
            raise SystemExit(1)
    elif children:
        # 列出子元素
        children_list = []
        for child in node.get_children():
            children_list.append({
                'name': child.name,
                'type': child.__class__.__name__,
                'path': child.autosar_path or child.path
            })
        result = {'path': path, 'children': children_list}
    else:
        # 获取全部属性
        props = {}
        for pv in node.get_property_values():
            props[pv[0]] = pv[1] if len(pv) > 1 else None
        result = {
            'path': path,
            'type': node.__class__.__name__,
            'file': node.file,
            'properties': props,
            'children': [{'name': c.name, 'type': c.__class__.__name__} for c in node.get_children()]
        }
    
    _format_output(result, output)
```

### 6.4 包结构

```
autosarfactory/
├── autosarfactory.py          # 现有，不变
├── datatype_utils.py          # 现有，不变
├── XmlElementDirtyTracker.py  # 现有，不变
├── autosar_ui.py              # 现有，不变
├── __init__.py                # 现有，不变
│
└── cli/                       # 新增——CLI 工具包
    ├── __init__.py
    ├── main.py                # CLI 入口，click 命令组
    │
    ├── commands/              # 各子命令实现
    │   ├── __init__.py
    │   ├── get.py
    │   ├── set.py
    │   ├── create.py
    │   ├── delete.py
    │   ├── list_cmd.py
    │   ├── query.py
    │   ├── diff.py
    │   ├── validate.py
    │   ├── export_cmd.py
    │   ├── tree.py
    │   └── apply.py
    │
    ├── services/              # 服务层
    │   ├── __init__.py
    │   ├── session.py         # SessionManager
    │   ├── query_engine.py    # QueryEngine
    │   ├── diff_engine.py     # DiffEngine
    │   ├── validation.py      # ValidationEngine
    │   └── yaml_ops.py        # YAML 操作执行器
    │
    ├── formatters/            # 输出格式化
    │   ├── __init__.py
    │   ├── table.py
    │   ├── json_fmt.py
    │   ├── yaml_fmt.py
    │   ├── tree_fmt.py
    │   └── diff_fmt.py
    │
    ├── parsers/               # 输入解析
    │   ├── __init__.py
    │   ├── path_parser.py     # ARXML 路径解析
    │   ├── query_parser.py    # WHERE 子句解析
    │   ├── yaml_parser.py     # YAML 配置解析
    │   └── stdin_parser.py    # 管道输入解析
    │
    └── completions/           # Shell 补全脚本
        ├── af-complete.bash
        ├── af-complete.zsh
        └── af-complete.ps1
```

### 6.5 安装与入口

```python
# setup.py 增加 CLI 入口点
setup(
    name='autosarfactory',
    # ... existing ...
    entry_points={
        'console_scripts': [
            'af = autosarfactory.cli.main:cli',
            'autosarfactory-cli = autosarfactory.cli.main:cli',
        ],
    },
)
```

```bash
# 安装后使用
pip install autosarfactory[cli]
af --help
```

---

## 7. 开发路线

### 7.1 三阶段路线图

```
Phase 1 (4-6 周)                Phase 2 (6-10 周)               Phase 3 (持续迭代)
┌─────────────────────┐     ┌─────────────────────────┐     ┌──────────────────────┐
│ CLI 基础框架 + CRUD │     │ 高级功能 + CI/CD 集成    │     │ 生态与体验优化        │
│                     │     │                         │     │                      │
│ ▢ CLI 框架搭建      │     │ ▢ af diff              │     │ ▢ Shell 自动补全     │
│ ▢ SessionManager    │  ─► │ ▢ af validate          │  ─► │ ▢ 进度条/彩色输出    │
│ ▢ af read / save    │     │ ▢ af query    条件查询 │     │ ▢ 批处理性能优化     │
│ ▢ af get            │     │ ▢ af apply   YAML 执行 │     │ ▢ 错误消息增强       │
│ ▢ af set            │     │ ▢ af tree              │     │ ▢ man page / --help   │
│ ▢ af create         │     │ ▢ af merge             │     │ ▢ Docker 镜像        │
│ ▢ af delete         │     │ ▢ JSON/YAML 输出      │     │ ▢ VS Code 扩展       │
│ ▢ af list           │     │ ▢ 管道协议实现        │     │ ▢ CI/CD 模板库       │
│ ▢ af export         │     │ ▢ CI/CD 集成示例      │     │ ▢ 与 MCP Server 协同 │
│ ▢ 基础 table 输出   │     │ ▢ 单元测试覆盖        │     │                      │
└─────────────────────┘     └─────────────────────────┘     └──────────────────────┘
```

### 7.2 Phase 1 详细任务

| 任务 | 说明 | 优先级 |
|------|------|--------|
| **CLI 框架** | `click` 命令组 + `main.py` 入口 | P0 |
| **SessionManager** | `-f`/`-d` 文件加载 + 状态管理 | P0 |
| **`af get`** | 路径查找 + 属性/引用/子元素读取 | P0 |
| **`af set`** | 属性设置 + 引用设置 + `--dry-run` | P0 |
| **`af create`** | 元素创建 + `--type`/`--name` 参数 | P0 |
| **`af delete`** | 元素删除 + `--force`/`--confirm` | P0 |
| **`af list`** | 子元素列表 + 类型过滤 | P1 |
| **`af read/save/export`** | 文件 I/O 基本命令 | P1 |
| **表格输出器** | table/json 格式化 | P1 |
| **路径解析器** | 支持多级路径 + 索引语法 | P1 |
| **单元测试** | 每个命令的基础用例 | P1 |

### 7.3 与 MCP Server 的协同

CLI 与现有 MCP Server 互补而非替代：

| 维度 | CLI (`af`) | MCP Server |
|------|-----------|------------|
| **使用者** | 人类（终端） + CI/CD 脚本 | AI 代理（Claude/Copilot） |
| **输入** | 命令行参数 + YAML 文件 | MCP 工具调用 |
| **输出** | 表格 / JSON / YAML / 纯文本 | 结构化 JSON（供模型解析） |
| **状态** | 跨命令共享（通过文件系统） | 无状态（每次调用独立） |
| **操作** | 可执行读写 | 参考级读操作 |

协同模式：AI 代理通过 MCP Server 理解 API，生成 CLI 命令，人类在终端执行。

---

## 8. 风险与应对

| 风险 | 影响 | 应对策略 |
|------|------|---------|
| **路径不唯一** | 多文件合并后同名元素冲突 | 优先使用 `autosar_path`（AR-PACKAGE 层级路径） |
| **大文件解析性能** | 数百 MB ARXML 加载缓慢 | 惰性加载 + 按需索引；Phase 3 优化 |
| **YAML 配置复杂度** | 用户难以编写正确 YAML | 提供 `af template` 命令生成模板；严格校验 + 友好错误 |
| **全局状态并发问题** | 多个 `af` 进程共享文件冲突 | 文件锁机制；建议同一目录不并发操作 |
| **管道错误传播** | 管线中某环节失败但后续仍执行 | `set -o pipefail` 等效；每命令标准退出码 |
| **Windows 兼容** | 路径分隔符、Shell 补全差异 | 统一使用 `/` 作为 ARXML 路径分隔符；提供 PowerShell 补全 |

---

## 9. 总结

### 设计核心要点

1. **命令行是 Python API 的声明式入口**
   - 同一个人，同一套语义，不同的交互方式
   - CLI 不引入新的模型层，直接封装 autosarfactory API

2. **CRUD 操作直觉化**
   - `af get` / `af set` / `af create` / `af delete` 语义自明
   - 路径作为元素唯一标识，`/Swcs/asw1/outPort` 即寻址

3. **管道化实现可组合性**
   - `af list | af set` 批量修改
   - `af query | xargs af delete` 条件删除
   - JSONL 作为管道协议

4. **声明式 YAML 支撑复杂场景**
   - 批量创建、嵌套层级、引用设置
   - 模板 + 变量替换实现参数化

5. **CI/CD 原生适配**
   - 结构化 JSON 输出
   - 明确退出码
   - `--dry-run` 预览
   - 零 Python 编程门槛

### 一句话总结

> **`af` 是 autosarfactory 的命令行面孔**——它将 L1 Python API 的完整能力转化为声明式、管道化、CI/CD 友好的命令行工具，让 AUTOSAR 配置操作从"写 Python 脚本"变为"敲一行命令"。

### 快速参考卡片

```
┌──────────────────────────────────────────────────────────┐
│                    af 命令速查表                          │
├──────────────────────┬───────────────────────────────────┤
│ af -f <file> read     │ 加载 ARXML 文件到内存             │
│ af get <path>         │ 读取元素/属性                     │
│ af get <path> -p NAME │ 读取指定属性值（纯文本输出）       │
│ af set <path> -p K=V  │ 设置属性                          │
│ af create <path> -t T │ 创建元素                          │
│ af delete <path>      │ 删除元素（需 --force 或 --confirm）│
│ af list <path>        │ 列出子元素                        │
│ af query -t <Type>    │ 全局类型查询                       │
│ af diff <a> <b>       │ 比较两个 ARXML                     │
│ af validate           │ 校验合规性                        │
│ af apply <yaml>       │ 执行 YAML 声明式配置              │
│ af save               │ 保存到文件                        │
│ af tree <path>        │ 树状展示                          │
├──────────────────────┴───────────────────────────────────┤
│ 全局选项：-f <文件> -o json|yaml|table --dry-run -v -q    │
│ 退出码：0=成功 1=通用错误 2=参数错误 3=校验失败            │
└──────────────────────────────────────────────────────────┘
```
