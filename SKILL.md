---
name: Self-growing Skill Creator (Lark-Native)
version: 1.2.0
description: >
  飞书原生 AI 元技能（Meta-Skill）。用于创建能持续学习和进化的专业领域飞书技能。
  支持多人协作写入经验 + 管理员审核晋升流程（v1.2 新增）。
  当用户需要"创建一个基于飞书知识库的领域专业技能"、或"用飞书 Wiki+Bitable 构建 AI 知识库"、
  或"让 AI 技能从工作中自动积累经验"、或"团队共享知识库并协作维护"时使用。
metadata:
  requires:
    bins: ["lark-cli"]
    # 注：lark-cli 未安装时，SKILL.md §0.0 包含自动安装流程（npm install -g @larksuite/cli）
    # 若 Node.js 也未安装，会提示用户先安装 Node.js
  references:
    - templates/quick-template.yaml
    - templates/subskill-skeleton.md
    - templates/stage1-interview.md
    - templates/stage2-extract.md
    - templates/stage3-dialogue.md
---

# Self-growing Skill Creator (Lark-Native) / 自我进化的技能创建工具（飞书版）

> **前置条件：** 先执行 §0.0 环境配置检查（含 lark-cli 自动安装），再执行 `lark-cli auth status` 确认已授权。若未授权或 scope 不完整，参见下方 §0 授权指南。

> **规格依据：** `2026-04-08-asc-for-lark-spec.md` (v1.5)。本 SKILL.md 是该规格的可执行实现。

> **开发者注意事项已内嵌：** T4 端对端测试验证的 D1-D6 全部转化为下文中的执行约束标记（⚠️D-N）。

---

## 零、前置条件与授权指南

### 0.0 环境配置检查与自动安装

> **目标：** 确保 `lark-cli` 已安装且可用。若缺失，自动尝试安装；若自动安装失败，给出明确的用户指引。
>
> **执行时机：** 每次运行 Stage 1 前 **MUST** 先执行本节检查（见 §2.0）。

#### Step 0a — 检查 lark-cli

```bash
lark-cli --version 2>/dev/null || echo "NOT_INSTALLED"
```

| 输出 | 操作 |
|------|------|
| 版本号（如 `1.x.x`） | 通过，跳至 Step 0d |
| `NOT_INSTALLED` 或报错 | 进入 Step 0b |

#### Step 0b — 检查 Node.js/npm

`lark-cli` 通过 npm 分发，安装前需确认 Node.js 环境。

```bash
node --version 2>/dev/null && npm --version 2>/dev/null
```

| 输出 | 操作 |
|------|------|
| 均有版本号 | 进入 Step 0c（自动安装 lark-cli） |
| 任一缺失 | 停止，提示用户安装 Node.js（见下方 **Node.js 安装指引**） |

**Node.js 安装指引：**

| 平台 | 方式 |
|------|------|
| Windows | 访问 https://nodejs.org 下载 LTS 安装包，双击安装 |
| macOS | `brew install node` 或官网下载 pkg 安装 |
| Linux | `sudo apt install nodejs npm` (Debian/Ubuntu) 或对应发行版包管理器 |

安装完成后，**重新打开终端**再返回执行本 SKILL.md。

#### Step 0c — 自动安装 lark-cli

```bash
npm install -g @larksuite/cli
```

**安装后验证：**

```bash
lark-cli --version
```

| 结果 | 操作 |
|------|------|
| 有版本号 | 继续 Step 0d |
| 仍报错 | 检查 npm global bin 是否在 PATH 中；若不在，执行 `npm prefix -g` 找到路径并加入 PATH，或重启终端 |

#### Step 0d — 飞书客户端检查（可选）

> 飞书桌面客户端不是 `lark-cli` 运行的必要前提。但以下功能需要客户端配合：
> - 人工审核 GUI 操作（v1.2「多写一审」模式）
> - 实时消息通知
> - 文档可视化编辑

**检查命令：**

```bash
# Windows (Git Bash / WSL)
test -d "$LOCALAPPDATA/Programs/Lark" 2>/dev/null || test -d "/c/Program Files/Lark" 2>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"

# macOS
test -d "/Applications/Lark.app" 2>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"

# Linux
which lark 2>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```

| 结果 | 操作 |
|------|------|
| `INSTALLED` | 继续 §0.1 |
| `NOT_INSTALLED` | 提示用户下载：https://www.feishu.cn/download |

---

### 0.1 最小 Scope 集合

Self-growing Skill Creator (Lark-Native) 运行需要以下飞书 OpenAPI 权限。首次使用前一次性授权即可。

| 类别 | Scope | 用途 |
|------|-------|------|
| Wiki | `wiki:wiki:readonly`, `wiki:wiki:write` | 创建/读取知识空间节点 |
| Bitable | `bitable:app`, `bitable:app:readonly` | 创建多维表格、读写记录 |
| 文档 | `docs:doc:readonly`, `docs:doc:write` | 创建/读取飞书文档 |
| 基础 | `lark:auth` | 认证鉴权 |

**一键授权命令：**

```bash
lark-cli auth login --scopes "wiki:wiki:readonly,wiki:wiki:write,bitable:app,bitable:app:readonly,docs:doc:readonly,docs:doc:write,lark:auth"
```

**验证方式：** `lark-cli auth status`

### 0.2 Wiki 知识空间创建指引

飞书 API 当前不支持程序化创建 Wiki 空间。需用户在飞书客户端手动创建：

1. 打开飞书 → 左侧「知识库」→「新建空间」→ 输入名称
2. 设置可见范围（建议先设为「仅自己可见」）
3. 创建后进入空间设置页，从 URL 复制 **space_id**
   - URL 格式：`https://xxx.feishu.cn/wiki/Spacexxxxxxxxx`
   - `Space` 后的字符串即为 space_id

### 0.3 JSON 参数传递约定（v1.1 简化版）

> ⚠️ **D1 [CRITICAL]：本章节定义所有 lark-cli 命令中 JSON 参数的标准传递方式。所有 Stage 中的命令块必须遵守。**

**环境要求：** 本 SKILL.md 使用 **bash shell** 执行（Git Bash / WSL / macOS / Linux）。PowerShell 下的参数解析有缺陷，**MUST NOT** 直接使用。

#### 三种标准模式

| 模式 | 适用参数 | 写法 | 验证状态 |
|------|----------|------|----------|
| **A. `@file` 引用** | `--json`, `--fields` | `--json @payload.json` | ✅ v1.0.4 原生支持，含中文可用 |
| **B. `$(cat)` 命令替换** | `--data`, `--markdown` | `--data "$(cat payload.json)"` | ✅ bash 原生功能，含中文可用 |
| **C. 内联 ASCII JSON** | `--params` | `--params '{"space_id":"xxx"}'` | ✅ 仅限纯 ASCII 简单对象 |

#### 标准写 JSON 的方式

```bash
# 使用 heredoc（推荐 — 支持中文、多行、无需转义）
cat > payload.json <<'EOF'
{
  "field_name": "关键词",
  "type": "text"
}
EOF
```

#### 三种模式的使用示例

```bash
# 模式 A：Bitable 操作（--json / --fields 接受 @file）
lark-cli base +field-create --base-token "$APP_TOKEN" --table-id "$TABLE_ID" --json @payload.json

# 模式 B：Wiki 节点创建（--data 不支持 @file，改用 $(cat)）
lark-cli wiki nodes create \
  --params "{\"space_id\":\"$SPACE_ID\"}" \
  --data "$(cat node.json)" \
  --jq '.data.node.node_token'

# 模式 C：--params 只含 ASCII 键值，直接内联
lark-cli base +record-list --base-token "$APP_TOKEN" --table-id "$TABLE_ID"
```

#### 为什么这样设计

- **v1.0.4 验证：** `--json @file` 和 `--fields @file` 经 dry-run 验证可用（含中文键/值）
- **`--data` / `--params`：** v1.0.4 不支持 `@file`，但 bash 的 `"$(cat file)"` 命令替换在 shell 层面解决，无需 CLI 升级
- **对比老方案：** 弃用了 "Node.js 中转脚本"——嵌套 execSync + 多层引号转义脆弱且易出错

**MUST NOT：**
- 在 PowerShell 下执行本 SKILL.md 中的任何命令（改用 Git Bash / WSL）
- 使用 `--data @file` 或 `--params @file`（v1.0.4 不支持，会报 `invalid JSON format`）
- 把含中文的 JSON 用 `-e` 字符串拼接进 shell 命令行（应存文件再用模式 A/B 引用）

---

## 一、产品概述

### 1.1 定位

本技能是一个**元技能（Meta-Skill）**——它不直接参与用户的业务工作流，而是帮助用户创建一个**子技能**。该子技能以飞书 Wiki + Bitable 为知识存储后端，能在实际工作中持续积累经验并自主进化。

### 1.2 核心产出

运行完四阶段流程后，产生：

| 产物 | 存储位置 | 说明 |
|------|----------|------|
| Wiki 知识空间骨架 | 飞书云端 | 含 Bitable App + 参考文件节点 + 工作归档节点 + 技能介绍文档 |
| 知识地图表（KM） | Bitable 内 | 结构化知识节点索引 |
| 经验库表（EL） | Bitable 内 | 情境化经验条目，通过 link 字段关联 KM 表 |
| 子技能 SKILL.md | 本地文件 | 包含飞书资源标识符 + 领域变量定义 + 运行时检索指令 |

### 1.3 架构图

```
SKILL.md 本地文件（领域变量定义）
       │
       ↓ Stage 1 生成
经验库表(EL) ──link字段(单向)──→ 知识地图表(KM)
  │                                    │
  ↓                                    ↓
工作归档Wiki节点                    参考文件Wiki节点
```

### 1.4 设计原则

- **零本地依赖：** 所有数据存飞书云端，无 Python 脚本、无本地数据库
- **幂等优先：** 重复执行任何 Step 不产生脏数据（详见各步骤 checkpoint 检测）
- **先检后写：** 每个写入操作前检查目标是否已存在
- **人机协同：** 批量写入前展示摘要获确认；diff_analysis 来源的经验须人工确认

---

## 二、Stage 1 — 初创（建立知识空间骨架）

> **目标：** 通过对话引导用户完成需求访谈，然后在飞书中搭建完整的知识空间骨架（Wiki 节点 + Bitable 两张表 + 全部字段 + 技能介绍文档），最后将资源标识符写入子技能 SKILL.md。

> **预估时间：** 标准模式 ~15-20 分钟 / 快速模板模式 ~3-5 分钟

### 2.0 启动检测

**MUST** 在开始 Stage 1 前依次执行以下检查：

#### ① 环境配置检查

执行 §0.0 环境配置检查与自动安装，确认 `lark-cli` 已安装且可用。若未安装，按 §0.0 Step 0b→0c 自动安装或提示用户。

#### ② 授权状态检查

```bash
lark-cli auth status
```

| 返回值 | 操作 |
|--------|------|
| `tokenStatus = "valid"` | 继续 |
| 其他值 | 停止，提示用户重新授权（见 §0.1 一键命令），不继续后续操作 |

然后向用户确认：
1. 已创建 Wiki 空间并获取到 **space_id**
2. 准备好参考文件的飞书文档链接（快速模式可跳过）

获取用户输入后进入模式选择（§2.0a 或 §2.0b）。

### 2.0a 模式选择

向用户提供三种路径：

```
请选择初始化模式：

[A] 标准模式 — 完整引导式流程（~15-20 分钟）
    · 需求访谈 → 定制化字段结构 → 领域变量提取
    · 适用：生产级技能、复杂专业领域、首次建库

[B] 快速模板模式 — 预设配置一键部署（~3-5 分钟）
    · 默认字段结构 + 通用领域变量占位符
    · 适用：概念验证、快速迭代、简单场景

[C] 接入已有库 — 团队成员加入现有知识库（~1-2 分钟）⬆️v1.2
    · 输入 space_id + app_token，自动探测表结构
    · 适用：团队第 2+ 位成员接入已有知识库
```

- 用户选择 **[A]** → 进入 **§2.1 标准模式**
- 用户选择 **[B]** → 进入 **§2.9 快速模板模式**
- 用户选择 **[C]** → 进入 **§2.10 接入已有库模式**

### 2.1 标准模式 — Step 1：需求访谈

通过对话逐步收集以下信息。**MUST** 以自然对话方式进行，避免一次性抛出问卷。

**必须收集的信息：**

| 信息 | 用途 | 引导示例 |
|------|------|----------|
| 技能目标 | 定义子技能的核心功能 | "这个技能主要帮你做什么？比如审核合同、管理项目、还是代码审查？" |
| 触发场景 | 写入子技能 description | "通常什么情况下你会用到它？你会怎么描述你的需求？" |
| 输出格式 | 决定子技能的行为模式 | "你希望它输出什么？一份报告？一个检查清单？还是直接修改文档？" |
| 专业领域 | 影响 Stage 2 的知识提取策略 | "这个领域有什么专业术语或特殊规则我需要注意？" |
| 领域变量列表 | 决定 EL 表的动态字段 | "有哪些关键变量会影响处理逻辑？比如'甲方/乙方'、'交易规模'等？" |

**对每个领域变量，进一步确认：**

```yaml
# 对话收集结果示例（内部暂存，不写入文件）
domain_variables:
  - name: "谈判地位"
    type: enum
    options: ["甲方", "乙方", "中立方"]
    hint: "从用户语言中识别我方角色；无法识别时留空"
  - name: "交易规模"
    type: text
    hint: "自由填写金额或规模描述"
```

**访谈完成后：** 向用户展示收集到的信息摘要，获得确认后再进入 Step 2。

### 2.2 Step 2：创建 Wiki 节点结构

> ⚠️ **D3 [CRITICAL]：`wiki nodes create` 必须使用 `--params` + `--data` 双参数格式。不支持独立 flag。**

> ⚠️ **D1 [CRITICAL]：`--data` 不支持 `@file` 引用。用 `$(cat file.json)` bash 替换（见 §0.3 模式 B）。`--params` 只含 ASCII 键值，直接内联（模式 C）。**

#### Step 2a — 创建根 Wiki 节点

```bash
cat > node-root.json <<'EOF'
{
  "node_type": "origin",
  "obj_type": "docx",
  "title": "[技能名称]知识库"
}
EOF

ROOT_TOKEN=$(lark-cli wiki nodes create \
  --params "{\"space_id\":\"$SPACE_ID\"}" \
  --data "$(cat node-root.json)" \
  --jq '.data.node.node_token')
```

**Checkpoint CP-S1-2 检测：** 若 ROOT_TOKEN 提取成功且长度 > 10，标记此步已完成。

#### Step 2b — 创建参考文件子节点

```bash
cat > node-ref.json <<EOF
{
  "node_type": "origin",
  "obj_type": "docx",
  "parent_node_token": "$ROOT_TOKEN",
  "title": "参考文件"
}
EOF

REF_TOKEN=$(lark-cli wiki nodes create \
  --params "{\"space_id\":\"$SPACE_ID\"}" \
  --data "$(cat node-ref.json)" \
  --jq '.data.node.node_token')
```

#### Step 2c — 创建工作归档子节点

```bash
cat > node-arc.json <<EOF
{
  "node_type": "origin",
  "obj_type": "docx",
  "parent_node_token": "$ROOT_TOKEN",
  "title": "工作归档"
}
EOF

ARC_TOKEN=$(lark-cli wiki nodes create \
  --params "{\"space_id\":\"$SPACE_ID\"}" \
  --data "$(cat node-arc.json)" \
  --jq '.data.node.node_token')
```

**验证：** 三级节点（ROOT / REF / ARC）均成功提取 token。任一失败则停止并报错。

### 2.3 Step 3：在 Wiki 内创建 Bitable App 并挂载

> **已验证（T2.3）：** `obj_type: bitable` 会在 Wiki 内新建 Bitable 并自动挂载为子节点。返回的 `obj_token` 即为 APP_TOKEN。

```bash
cat > node-bitable.json <<EOF
{
  "node_type": "origin",
  "obj_type": "bitable",
  "parent_node_token": "$ROOT_TOKEN",
  "title": "知识与经验库"
}
EOF

APP_TOKEN=$(lark-cli wiki nodes create \
  --params "{\"space_id\":\"$SPACE_ID\"}" \
  --data "$(cat node-bitable.json)" \
  --jq '.data.node.obj_token')
```

**Checkpoint CP-S1-3 检测：** APP_TOKEN 成功提取（格式 `BzQ...`，长度约 20）。

### 2.4 Step 4：创建两张数据表及全部固定字段

> ⚠️ **D1 [CRITICAL]：`--json` / `--fields` 支持 `@file` 引用（见 §0.3 模式 A），含中文值也可用。**
>
> ⚠️ **命名约束（REAL-m02 修复）：Bitable 表名必须使用纯 ASCII（英文+数字+下划线）。Chinese 表名会导致 `Invalid character` 错误。固定使用 `KnowledgeMap` / `ExperienceLog` 作为内部标识，中文名称仅在 UI 层显示。**

#### Step 4a — 知识地图表（KnowledgeMap）

```bash
cat > fields-km.json <<'EOF'
[
  {"field_name": "节点名称", "type": "text"},
  {"field_name": "核心规则", "type": "text"},
  {"field_name": "关键词", "type": "text"},
  {"field_name": "规则类型", "type": "select", "options": [{"name":"mandatory"},{"name":"recommended"},{"name":"conditional"}]},
  {"field_name": "参考来源", "type": "text"},
  {"field_name": "红线标记", "type": "checkbox"},
  {"field_name": "创建日期", "type": "datetime"}
]
EOF

KM_TABLE=$(lark-cli base +table-create \
  --base-token "$APP_TOKEN" \
  --name "KnowledgeMap" \
  --fields "$(cat fields-km.json)" \
  --jq '.data.table.id')
```

将输出赋值给变量 `KM_TABLE`。

**KM 表字段说明：**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 节点名称 | text | 知识点标题（唯一性标识） |
| 核心规则 | text | ≤80字骨架概括 |
| 关键词 | text | 3-7个检索关键词，中文逗号+空格分隔（⚠️D6 规范） |
| 规则类型 | select | mandatory / recommended / conditional |
| 参考来源 | text | 飞书 Wiki URL |
| 红线标记 | checkbox | 底线条款标记 |
| 创建日期 | datetime | 记录创建时间（用于时间衰减排序） |

#### Step 4b — 经验库表（ExperienceLog）

> ⚠️ **v1.2 新增：** EL 表包含审核工作流所需的 3 个额外字段（状态/审核备注/关联KM条目），用于支持"多写一审"团队协作模式。

```bash
cat > fields-el.json <<'EOF'
[
  {"field_name": "标题", "type": "text"},
  {"field_name": "调整逻辑", "type": "text"},
  {"field_name": "工作文件链接", "type": "text"},
  {"field_name": "红线标记", "type": "checkbox"},
  {"field_name": "置信度", "type": "select", "options": [{"name":"high"},{"name":"medium"}]},
  {"field_name": "来源类型", "type": "select", "options": [{"name":"manual"},{"name":"diff_analysis"}]},
  {"field_name": "创建日期", "type": "datetime"},
  {"field_name": "状态", "type": "select", "options": [{"name":"待审"},{"name":"已通过"},{"name":"已驳回"}]},
  {"field_name": "审核备注", "type": "text"},
  {"field_name": "关联KM条目", "type": "link", "link_table": "$KM_TABLE"}
]
EOF

EL_TABLE=$(lark-cli base +table-create \
  --base-token "$APP_TOKEN" \
  --name "ExperienceLog" \
  --fields "$(cat fields-el.json)" \
  --jq '.data.table.id')
```

**注意：** `关联KM条目` 字段的 `link_table` 值 **MUST 在运行时替换为 KM 表的实际 table_id**（与 Step 4c 中 link 字段同理，但方向相反——此字段在 EL 表中指向 KM 表记录）。若 KM 表尚未创建则暂留占位符 `"$KM_TABLE"`，在 Step 4c 之后用实际值补建该 link 字段。

> ⚠️ **v1.2 变更说明：** 原 §2.4 Step 4c 的「关联知识」(EL→KM) 字段仍保留不变。新增的 `关联KM条目` 用途不同——它用于 Stage 5 晋升时回填"本条经验已晋升为哪条 KM 记录"，实现双向追溯。两者共存不冲突。

**EL 表字段说明：**

| 字段名 | 类型 | 说明 | 审核工作流角色 |
|--------|------|------|---------------|
| 标题 | text | 经验条目标题 | — |
| 调整逻辑 | text | ≤80字情境化经验描述 | — |
| 工作文件链接 | text | 飞书文档 URL | — |
| 红线标记 | checkbox | 底线条款标记 | — |
| 置信度 | select: high/medium | AI 对经验的置信度评估 | — |
| 来源类型 | select: manual/diff_analysis | 来源渠道（手动 / 差分分析） | — |
| 创建日期 | datetime | 记录创建时间 | — |
| **状态** ⬆️v1.2 | **select: 待审/已通过/已驳回** | **审核工作流状态，默认"待审"** | **核心字段** |
| **审核备注** ⬆️v1.2 | **text** | **管理员驳回/通过时填写原因** | **管理员填写** |
| **关联KM条目** ⬆️v1.2 | **link → KM 表** | **Stage 5 晋升成功后回填目标 KM 记录 ID** | **Stage 5 自动填充** |
| 关联知识 | link → KM 表 | Stage 3 写入时手动关联的 KM 节点 | Stage 3 填充（保留不变） |

#### Step 4c — 关联字段（link：EL → KM）

> ⚠️ **D2 [CRITICAL]：link 字段的 `link_table` 必须传目标表的 **table_id**（如 `tbl2j6...`），不能传表名。否则返回 `not_found` 错误。**

> **执行顺序约束：** 必须在 KM 表和 EL 表都创建成功后才执行此步（因为需要两张表的 table_id）。

```bash
cat > field-link.json <<EOF
{
  "field_name": "关联知识",
  "type": "link",
  "link_table": "$KM_TABLE"
}
EOF

lark-cli base +field-create \
  --base-token "$APP_TOKEN" \
  --table-id "$EL_TABLE" \
  --json @field-link.json
```

**Checkpoint CP-S1-4 检测：** KM_TABLE 和 EL_TABLE 均成功提取；link 字段创建无报错。

#### Step 4d — 审核工作流关联字段（v1.2 新增）

> ⚠️ **执行顺序约束：** 此步骤 MUST 在 Step 4a（KM 表）和 Step 4b（EL 表）都完成后执行，因为需要两张表的 table_id。

> **说明：** Step 4b 中创建的 `关联KM条目` 字段使用了 `$KM_TABLE` 占位符。如果当时 KM_TABLE 已可用则无需此步；否则在此步补建该 link 字段。

```bash
# 仅当「关联KM条目」字段尚未存在时执行
cat > field-km-link.json <<EOF
{
  "field_name": "关联KM条目",
  "type": "link",
  "link_table": "$KM_TABLE"
}
EOF

lark-cli base +field-create \
  --base-token "$APP_TOKEN" \
  --table-id "$EL_TABLE" \
  --json @field-km-link.json
```

**幂等性：** 若字段已存在（Step 4b 创建时 KM_TABLE 已就绪），`+field-create` 会报重复错误，可安全忽略（视为已通过）。

### 2.5 Step 5：创建动态领域变量字段

> **⚠️ C9 [HIGH]：以下字段统称为「**动态领域变量字段**」，由 Stage 1 Step 1 需求访谈收集确定，**必须在本 Step（Step 5）全部创建完毕**，不得延迟到后续 Stage。Stage 3 写入 EL 时将依赖这些字段。**

> ⚠️ **标识符规范：** `field_name` 必须去除首尾空白字符并避免特殊符号。AI 从访谈结果生成字段名时 MUST 执行 `trim()` 处理——历史 bug `" NegotiationRole"`（前导空格）由此类疏漏导致。

根据 Step 1 收集的领域变量列表，为每个变量在 EL 表中创建对应字段：

```bash
# 枚举类型 → select 字段
cat > field-var1.json <<'EOF'
{
  "field_name": "谈判地位",
  "type": "select",
  "options": [{"name":"甲方"},{"name":"乙方"},{"name":"中立方"}]
}
EOF

lark-cli base +field-create \
  --base-token "$APP_TOKEN" \
  --table-id "$EL_TABLE" \
  --json @field-var1.json

# 文本类型 → text 字段
cat > field-var2.json <<'EOF'
{"field_name": "交易规模", "type": "text"}
EOF

lark-cli base +field-create \
  --base-token "$APP_TOKEN" \
  --table-id "$EL_TABLE" \
  --json @field-var2.json

# ... 对每个领域变量重复上述模式
```

**Checkpoint CP-S1-5 检测：** 所有预期字段均已存在。实现方式：调用 `base +field-list` 对比当前字段列表与规格定义的字段名集合，仅报告缺失字段（不在此处补建，仅做验证）。若发现缺失字段 → **停止并报错，提示用户重新执行 Step 5**。

### 2.6 Step 6：创建技能介绍文档并挂载

```bash
# 创建飞书文档（Markdown 内容由 AI 根据需求访谈结果生成）
# 内容较长时写入文件再用 $(cat) 引用，避免 shell 参数过长
cat > skill-intro.md <<'EOF'
[生成的 Markdown 内容]
EOF

DOC_TOKEN=$(lark-cli docs +create \
  --title "[技能名称]技能介绍" \
  --markdown "$(cat skill-intro.md)" \
  --jq '.data.doc_id')

# 挂载为 Wiki 子节点
cat > node-doc.json <<EOF
{
  "node_type": "origin",
  "obj_type": "docx",
  "obj_token": "$DOC_TOKEN",
  "parent_node_token": "$ROOT_TOKEN",
  "title": "技能介绍"
}
EOF

lark-cli wiki nodes create \
  --params "{\"space_id\":\"$SPACE_ID\"}" \
  --data "$(cat node-doc.json)" \
  --jq '.data.node.node_token'
```

**Checkpoint CP-S1-6 检测：** DOC_TOKEN 成功提取；wiki nodes create 无错误。

### 2.7 Step 8：将运行时参数写入子技能 SKILL.md

> **这是 Stage 1 的最终产出。** 将以下模板中的 **`[占位符]` 全部替换为 Stage 1 收集的实际值后**，写入 `[项目路径]/[技能名称]/SKILL.md`。

> ⚠️ **标识符规范（防 `el_field` 前导空格 bug）：** 生成以下 YAML 前，对所有标识符字段（`name`、`el_field`、字段名）MUST 执行 `value.strip()` 去除首尾空白字符。历史 bug 案例：访谈提取出 " NegotiationRole"（前导空格）直接写入 `el_field` 导致后续字段匹配失败。

```yaml
---
name: [技能名称]
version: 1.0.0
description: "[技能一句话描述]。当用户需要[触发场景]时使用。"
---

# [技能名称]

## 飞书知识库配置（由 Self-growing Skill Creator (Lark-Native) Stage 1 自动生成，勿手动修改）

> ⚠️ 以下配置块的任何手动修改可能导致检索功能异常。

LARK_KNOWLEDGE_BASE:
  bitable_app_token: "[APP_TOKEN]"              # Bitable App Token（格式 BzQ...）
  wiki_space_id: "[SPACE_ID]"                    # Wiki 空间 ID（格式 Spacexxx）
  tables:
    knowledge_map: "[KM_TABLE_ID]"               # KM 表 table_id（格式 tblxxx）
    experience_log: "[EL_TABLE_ID]"              # EL 表 table_id（格式 tblxxx）
  wiki_nodes:
    reference: "[REF_TOKEN]"                     # 参考文件节点 node_token
    archives: "[ARC_TOKEN]"                      # 工作归档节点 node_token

## 领域变量定义（由 Self-growing Skill Creator (Lark-Native) Stage 1 自动生成，可手动维护）

DOMAIN_VARIABLES:
# 以下内容来自 Stage 1 Step 1 需求访谈
  - name: "[变量名]"
    type: enum  # enum | text
    options: [选项1, 选项2]
    hint: "[从用户语言识别此变量的提示]"
    el_field: "[对应 EL 表中的精确字段名]"

## 飞书授权 Scope 清单（子技能启动时一次性检查）

LARK_REQUIRED_SCOPES: >
  wiki:wiki:readonly wiki:wiki:write bitable:app bitable:app:readonly
  docs:doc:readonly docs:doc:write lark:auth
```

> **⚠️ 注意：** 此模板中**不得包含** `domain_variables: "[DV_TABLE]"` 行。该字段已在 v1.1 从架构移除（领域变量固化到上方 `DOMAIN_VARIABLES` 块）。

**Checkpoint CP-S1-8 检测：** 文件写入成功（路径可访问、非空）→ **标记 Stage 1 全部完成**；YAML 块完整（frontmatter + LARK_KNOWLEDGE_BASE + DOMAIN_VARIABLES + LARK_REQUIRED_SCOPES 四块均存在）；无 `domain_variables` 残留。任一条件不满足 → 停止并报错，不允许进入后续 Stage。

### 2.8 Step 9：向用户展示成果

展示以下信息并获得用户确认：

1. **Wiki 空间结构总览**（含各节点的飞书 URL）
2. **Bitable 表结构摘要**（KM 表 N 个字段 + EL 表 M 个字段 + link 关联）
3. **子技能 SKILL.md 路径**和**领域变量列表**
4. **下一步提示**：建议用户手动设置知识空间访问权限，然后进入 Stage 2 补充参考文件

---

### 2.9 快速模板模式（§2.0b 选择 [B] 时进入）

> **跳过 Step 1 需求访谈，使用预设默认配置，3-5 分钟完成建库。**

详细默认配置见 `templates/quick-template.yaml`。

#### 快速模板执行步骤

| 步骤 | 操作 | 与标准模式的差异 |
|------|------|------------------|
| Q1 | 确认选择快速模板模式 | 替代 Step 1 访谈 |
| Q2 | 用户输入技能名称 | 仅需这一个输入项 |
| Q3 | 确认/输入 Wiki space_id | 同标准模式 |
| Q4 | 执行 §2.2 - §2.7 的 Step 2-8 | 但字段使用默认配置（见下表） |
| Q5 | 展示结果 + 升级提示 | 额外提示可在任意时间升级为标准模式 |

#### 快速模板默认字段配置

**KM 表字段（同标准模式）：** 节点名称(text), 核心规则(text), 关键词(text), 规则类型(select:mandatory/recommended/conditional), 参考来源(text), 红线标记(checkbox), 创建日期(datetime)

**EL 表固定字段（同标准模式）：** 标题(text), 调整逻辑(text), 工作文件链接(text), 红线标记(checkbox), 置信度(select:high/medium), 来源类型(select:manual/diff_analysis), 关联知识(link→KM), 创建日期(datetime)

**EL 表动态字段（通用占位符，不创建额外字段）：** 领域变量留空，由 AI 在子技能运行时从上下文动态推断。

**技能介绍文档：** 使用通用模板生成（而非定制内容）。

**DOMAIN_VARIABLES 写入：** 使用通用占位符模板：

```yaml
DOMAIN_VARIABLES:
  - name: "业务场景"
    type: text
    hint: "从用户指令中推断当前业务场景"
    el_field: null  # 快速模式下无对应 EL 字段
```

> **升级兼容性：** 快速模板创建的结构与标准模式**完全兼容**。用户可随时重新执行 Stage 1 标准模式来扩展字段和细化领域变量。幂等性设计确保已有数据不受影响。

---

### 2.10 接入已有库模式（§2.0b 选择 [C] 时进入）⬆️v1.2

> **目标：** 团队成员快速接入已有的飞书知识库，跳过所有建库步骤，直接生成本地子技能 SKILL.md。
>
> **适用场景：** 知识库已由团队管理员通过 [A]/[B] 模式创建完成，新成员需要加入协作。
>
> **预估时间：** ~1-2 分钟

#### 前提条件

用户必须从管理员处获取以下信息（通常在群聊/文档中共享）：

| 必需信息 | 来源 | 格式示例 |
|----------|------|----------|
| Wiki space_id | 管理员提供 | `Spacexxxxxxxxxx` |
| Bitable App Token | 管理员提供 | `BzQxxxxxxx` |
| 技能名称 | 约定或自选 | 如"合同审核助手" |

#### Step C1：验证连通性

```bash
# 验证 space_id 可访问
lark-cli wiki spaces get --params "{\"space_id\":\"$SPACE_ID\"}" 2>&1

# 验证 Bitable App 可访问
lark-cli base +table-list --base-token "$APP_TOKEN" 2>&1
```

| 返回值 | 操作 |
|--------|------|
| 两项均有返回数据 | 继续 Step C2 |
| 任一报权限错误 | 提示用户确认已被添加到知识空间/Bitable 协作成员列表 |
| `space_id` 不存在 | 提示管理员确认 space_id 是否正确 |

#### Step C2：探测表结构并提取 ID

```bash
# 列出所有表，识别 KM 表和 EL 表
TABLES=$(lark-cli base +table-list --base-token "$APP_TOKEN")
```

**自动识别规则：**

```
遍历返回的表列表：
  - 表名包含 "KnowledgeMap" 或 "知识地图" 或表名 = "KnowledgeMap"
    → 赋值 KM_TABLE = table_id

  - 表名包含 "ExperienceLog" 或 "经验库" 或表名 = "ExperienceLog"
    → 赋值 EL_TABLE = table_id

若无法自动识别（表名被修改过）：
  → 向用户展示可用表列表，让用户手动指定哪个是 KM、哪个是 EL
```

#### Step C3：探测字段结构（用于 DOMAIN_VARIABLES）

```bash
# 读取 EL 表全部字段定义
EL_FIELDS=$(lark-cli base +field-list \
  --base-token "$APP_TOKEN" \
  --table-id "$EL_TABLE")

# 读取 KM 表全部字段定义（可选，用于验证）
KM_FIELDS=$(lark-cli base +field-list \
  --base-token "$APP_TOKEN" \
  --table-id "$KM_TABLE")
```

**自动生成 DOMAIN_VARIABLES：**

从 EL 表字段列表中排除以下固定字段后，剩余的即为动态领域变量字段：

> **EL 表固定字段白名单（不纳入 DOMAIN_VARIABLES）：**
> 标题, 调整逻辑, 工作文件链接, 红线标记, 置信度, 来源类型, 创建日期, 状态, 审核备注, 关联KM条目, 关联知识

对每个非固定字段：
- `select` 类型 → 提取 options 列表作为枚举选项
- 其他类型 → 记录为 text

#### Step C4：探测 Wiki 节点（可选但推荐）

```bash
# 列出空间下所有节点
NODES=$(lark-cli wiki spaces list_nodes \
  --params "{\"space_id\":\"$SPACE_ID\"}")
```

尝试自动识别：
- 含「参考文件」关键词 → REF_TOKEN
- 含「工作归档」或「归档」关键词 → ARC_TOKEN
- 根节点（parent 为空）→ ROOT_TOKEN

若无法识别则留空，提示用户后续可手动补充。

#### Step C5：生成子技能 SKILL.md

使用与 §2.7 相同的模板，将探测到的实际值填入占位符：

```yaml
---
name: [技能名称]
version: 1.0.0
description: "[技能一句话描述]。当用户需要[触发场景]时使用。"
---

# [技能名称]

## 飞书知识库配置（由 Self-growing Skill Creator (Lark-Native) Stage 1-C 接入已有库生成）

LARK_KNOWLEDGE_BASE:
  bitable_app_token: "[APP_TOKEN]"
  wiki_space_id: "[SPACE_ID]"
  tables:
    knowledge_map: "[KM_TABLE_ID]"
    experience_log: "[EL_TABLE_ID]"
  wiki_nodes:
    reference: "[REF_TOKEN]"        # 可能为空
    archives: "[ARC_TOKEN]"         # 可能为空

## 领域变量定义（由 Stage 1-C 自动探测生成）

DOMAIN_VARIABLES:
# 以下字段来自 EL 表动态字段的自动探测
  - name: "[变量名]"
    type: enum / text
    options: [选项列表]   # select 类型有此行
    hint: "[从上下文推断此变量的提示]"
    el_field: "[对应 EL 表中的精确字段名]"

LARK_REQUIRED_SCOPES: >
  wiki:wiki:readonly wiki:wiki:write bitable:app bitable:app:readonly
  docs:doc:readonly docs:doc:write lark:auth
```

#### Step C6：向用户展示结果

```markdown
## ✅ 已成功接入知识库

### 连接信息
| 项目 | 值 |
|------|-----|
| Wiki 空间 | [空间名称](URL) |
| Bitable 应用 | APP_TOKEN: BzQ... |
| KM 表 | __ 条记录 |
| EL 表 | __ 条记录 |

### 探测到的领域变量
| 变量名 | 类型 | 对应字段 |
|--------|------|---------|

### 下一步
- 你现在可以直接运行 Stage 2（补充知识）或 Stage 3（写入经验）
- 写入的经验默认标记为「待审」，等待管理员审核
```

**Checkpoint CP-S1-C6：** SKILL.md 文件成功写入；所有探测到的 ID 非空且格式正确；DOMAIN_VARIABLES 与 EL 表实际字段匹配。

---

## 三、Stage 2 — 初学者（知识提取）

> **目标：** 从用户提供的参考文件（Playbook、模板、规范等）中提取结构化知识节点，批量写入知识地图表（KM）。

> **前置要求：** 用户已将参考文件上传至"参考文件"Wiki 节点，并提供飞书文档链接。建议转为飞书文档格式以确保 `docs +fetch` 可读取。
>
> **预估时间：** 单份文件 ~3-8 分钟（取决于篇幅和复杂度）

### 3.0 Stage 2 启动检测与恢复

> **⚠️ C1 [CRITICAL]：Stage 2 MUST 在 Stage 1（知识空间骨架搭建）全部完成后方可执行。若 Wiki 空间/Bitable/KM 表/EL 表任一不存在，停止并提示用户重新执行 Stage 1。**

**MUST** 在开始 Stage 2 前执行以下检查：

```bash
# 1. 验证 Bitable 可访问性
lark-cli base +table-list --base-token $APP_TOKEN

# 2. 检查 KM 表是否已有数据（断点续传判断）
lark-cli base +record-list --base-token $APP_TOKEN --table-id $KM_TABLE
```

| 检查项 | 通过条件 | 不通过时操作 |
|--------|----------|-------------|
| `+table-list` 返回包含"知识地图表" | 继续Stage 2 | 停止：KM表不存在，需重新执行Stage 1 |
| `+record-list` 返回 data 为空或已有记录 | 继续 | — |
| APP_TOKEN / KM_TABLE 变量存在 | 继续 | 停止：缺少必要参数，提示重新执行Stage 1 |

**Checkpoint CP-S2-0（断点续传）：** 若 KM 表已有 N 条记录：
- 向用户展示现有记录列表
- 询问："是否继续补充新知识？还是清除重建？"
- 选择「继续」→ 进入 Step 1，后续写入时自动跳过已存在的节点名称
- 选择「重建」→ 先清空再写入（需二次确认）
- **若 `+record-list` 返回空数组（首次运行）→ 自动进入首次模式，无需询问，直接开始 Step 1 读取参考文件**

### 3.1 Step 1：读取参考文件

#### 3.1a 获取文档 token

```bash
# 从 Wiki 节点 token 解析 obj_token
OBJ_TOKEN=$(lark-cli wiki spaces get_node \
  --params '{"token":"[REF_NODE_TOKEN]"}' \
  --jq '.data.node.obj_token' 2>&1)
```

若 OBJ_TOKEN 为空或报错：
- 检查 REF_NODE_TOKEN 是否正确（应为 Wiki 节点的 node_token，非 space_id）
- 提示用户确认节点是否存在、是否有权限访问

#### 3.1b 读取文档内容

```bash
DOC_CONTENT=$(lark-cli docs +fetch --doc $OBJ_TOKEN 2>&1)
```

#### 3.1c 文档类型适配策略

| 文件特征 | 处理方式 | 示例 |
|----------|----------|------|
| 纯文字文档（< 3000 字） | 全文一次性读取+提取 | SOP 文档、规则清单 |
| 中等文档（3000-10000 字） | 全文读取后按 H2/H3 标题拆分为知识块 | 完整 Playbook |
| 大型文档（> 10000 字） | 先提取目录结构后按章节分批精读，每批独立提取 | 行业标准、法规汇编 |
| 表格密集型文档 | 重点提取表格行/列标题作为知识节点字段 | 价格表、对照矩阵 |
| 含代码/配置示例 | 将代码块视为特殊知识节点，核心规则提取关键逻辑而非全文 | 技术规范 |

> **⚠️ C10 [MEDIUM]：当文档字数 > 5000 字时，MUST 按以下分批策略处理：**
> - 使用 `docs +fetch --doc $OBJ_TOKEN` 读取全文后，按 **H2/H3 标题边界** 将内容拆分为多个批次
> - 每批次 ≤ 4000 字，独立执行 §3.2 提取流程
> - 跨批次的同一知识点（如跨章节的完整规则）在 §3.5 全局汇总阶段合并去重

#### 3.1d 多文件批量处理

当用户提供多份参考文件时：

```
处理流程：
  对每个文件依次执行 Step 1-4（单文件完整提取周期）
    ↓
  所有文件处理完毕后执行 Step 5 全局质量自检
    ↓
  跨文件去重：对比所有已提取的"节点名称"，标记重复项供用户裁决
```

**MUST** 在开始前向用户展示文件清单和处理顺序：

```markdown
## 待处理的参考文件（共 N 份）

| # | 文件名 | 类型 | 预估规模 | 处理顺序 |
|---|--------|------|----------|----------|
| 1 | 合同审核SOP.docx | 规范 | ~5000字 | 第1个 |
| 2 | 风险控制手册.docx | 手册 | ~12000字 | 第2个（可能分批） |

确认后开始处理？
```

### 3.2 Step 2：LLM 结构化知识提取

#### 3.2a 提取角色设定

AI 扮演**领域知识架构师**角色：

```
身份：你是一位经验丰富的 [领域名称] 知识架构师。
任务：从给定的参考文档中识别并提炼出可被 AI 检索和复用的结构化知识节点。

核心原则：
1. 粒度适当：每个知识节点应是一个"原子级"的可执行规则或决策点，
   不是整章概述，也不是过于琐碎的操作步骤
2. 保留上下文：核心规则必须包含足够的上下文使读者能独立理解
3. 标签准确：关键词必须是该节点的独特检索入口，避免泛化
4. 忠于原文：不添加原文中没有的信息，不臆造边界条件
```

#### 3.2b 结构化输出格式

对每个知识节点提取以下字段：

| 提取项 | 格式要求 | 质量标准 | 反例（不合格） |
|--------|----------|----------|----------------|
| **节点名称** | 简短标题（4-20 字符） | 唯一标识该知识点，同文件内不可重复 | "第一章内容"（太泛）、超长句式 |
| **核心规则** | 小于等于80 字骨架概括 | 包含触发条件 + 动作 + 关键约束，不含废话 | 无实质内容的套话、照搬原文超长段落 |
| **关键词** | 3-7 个，中文逗号+空格分隔 | 覆盖所有检索维度，含专业术语 | 太泛词汇、英文逗号、重复词 |
| **规则类型** | mandatory / recommended / conditional | M=硬规则/R=最佳实践/C=有前提条件 | — |
| **参考来源** | 飞书 URL | 格式 `https://xxx.feishu.cn/docx/xxx` | 本地路径、相对路径 |
| **红线标记** | true / false | 仅法律/安全/合规底线标 true | 滥用 true 导致红线贬值 |
| **创建日期** | ISO 格式 `YYYY-MM-DD HH:MM:SS` | 当前日期时间 | — |

> ⚠️ **D6 关键词规范（强制）：**
> - 分隔符：**仅使用中文逗号+空格(`, `)**，禁止英文逗号 `,`、分号 `;`、竖线 `\|`
> - 每词长度：2-10 字符
> - 数量：每条记录 3-7 个关键词
> - 内容优先级：**专业术语** > **业务动作** > **对象名词**
> - 去重：单条内禁止重复词汇
> - 合格示例：`合同金额, 法务审核, 风险评估, 二审流程, 500万阈值`
> - 不合格示例：`合同, 审核, 合同审核, contract review`（含英文、重复）

#### 3.2c 提取密度指南

| 参考文件规模 | 建议节点数 | 粒度建议 |
|--------------|-----------|----------|
| < 10 页 / < 3000 字 | 5-10 个 | 每个 H2 标题提炼 1-2 个节点 |
| 10-30 页 / 3000-10000 字 | 15-25 个 | 每个 H2 提炼 2-3 个，H3 独立成节点 |
| 30-50 页 / 10000-20000 字 | 25-40 个 | 关注 H2/H3 的决策点和分支逻辑 |
| > 50 页 | 40-50 个（上限） | 聚焦 if-then 类规则性内容，跳过纯描述性段落 |

> **MUST NOT** 贪多。质量大于数量。一个精准的节点胜过十个模糊的概述。

#### 3.2d 提取结果审查交互

对每份文件展示提取结果供审查：

```markdown
## 从《[文件名]》中提取的知识节点（共 N 个）

| # | 节点名称 | 核心规则（预览） | 关键词 | 类型 | 红线 |
|---|----------|------------------|--------|------|------|
| 1 | 大额合同审核规则 | 合同额大于等于500万须法务二审... | 合同金额, 法务审核... | M | 是 |
| 2 | 甲方谈判优先权条款 | 甲方享有修改权行使窗口... | 谈判地位, 修改权... | R | 否 |

请检查：节点粒度？遗漏？红线准确性？
回复"确认"开始写入，或指出需调整的条目编号。
```

### 3.3 Step 3：批量写入 KM 表

**前置门禁： MUST** 已获得用户对提取结果的明确确认（Step 2 末尾的确认交互）。

#### 3.3a 写入命令模板

> ⚠️ **D1 [CRITICAL]：`+record-upsert` 的 `--json` 支持 `@file` 引用（见 §0.3 模式 A），含中文键名与值可用。**

```bash
# 对每条知识节点，先写 JSON payload 文件再用 @file 引用写入
cat > "record-km-${N}.json" <<'EOF'
{
  "节点名称": "大额合同审核规则",
  "核心规则": "合同金额超过500万须经法务二审，提前3天预约审批窗口",
  "关键词": "合同金额, 法务审核, 风险评估, 二审流程, 500万阈值",
  "规则类型": "mandatory",
  "参考来源": "https://xxx.feishu.cn/docx/abc123",
  "红线标记": true,
  "创建日期": "2026-04-16 00:00:00"
}
EOF

lark-cli base +record-upsert \
  --base-token "$APP_TOKEN" \
  --table-id "$KM_TABLE" \
  --json @"record-km-${N}.json"
```

**批量循环提示：** AI 提取出 N 条知识节点后，MUST 在 shell 中用 `for` 循环逐条写入并在每次循环末尾回读校验，不得一次性拼批（避免 REAL-m04 中观察到的"只写第 1 条"问题）。

#### 3.3b 幂等性：智能去重

**写入前 MUST 执行全表扫描去重：**

```bash
lark-cli base +record-list --base-token $APP_TOKEN --table-id $KM_TABLE
```

去重算法（以"节点名称"为唯一性标识）：

```
对新提取的每条节点：
  1. 在现有 records 中搜索"节点名称"字段
  2. 判断匹配度：
     - 完全匹配（字符串相等）      → 标记【跳过】，记录到跳过清单
     - 高度相似（编辑距离小等于2或包含关系）→ 标记【疑似重复】，加入待确认队列
     - 无匹配                     → 标记【新增】，正常进入写入队列

写入完成后汇报：新增 N 条 | 跳过 M 条（已存在）| K 条疑似重复待确认
```

#### 3.3c 批量写入顺序

按优先级排序写入：**mandatory 类（红线优先）** → **conditional 类** → **recommended 类**
同类内按原文出现顺序排列。

#### 3.3d 写入结果验证

每批写入后立即回读验证记录数变化。
- 若数量一致 → 标记本批通过
- 若数量不符 → **最多重试 3 次，每次重新执行写入命令**；仍失败则输出失败行 JSON 数据供人工排查，**停止当前批次并报错**（不跳过继续下一批）

### 3.4 Step 4：单文件质量自检

每份文件写入完成后执行：

```bash
lark-cli base +record-list --base-token $APP_TOKEN --table-id $KM_TABLE
```

**自检清单（逐条勾选）：**

| # | 检查项 | 通过标准 | 不通过时处理 |
|---|--------|----------|-------------|
| Q1 | URL 有效性 | 所有 `参考来源` 为有效 feishu.cn URL | 重写对应行 |
| Q2 | 规则长度 | 所有 `核心规则` 小于等于 80 字 | 拆分或精简 |
| Q3 | 关键词数量 | 每条大于等于 3 个且小于等于 7 个 | 补充或裁剪 |
| Q4 | 关键词格式 | 符合 D6 规范 | 修正分隔符和非法字符 |
| Q5 | 名称唯一性 | 无完全重复的"节点名称" | 合并或重命名 |
| Q6 | 关联字段 | `关联经验` 为空（Stage 3 才填充） | 警告但不阻断 |
| Q7 | 红线合理性 | 红线=true 的比例小于等于 30% | 建议复核 |

将自检结果汇总报告用户。Q1-Q6 中任何一项 FAIL 都需修复后才可进入下一文件。

**Checkpoint CP-S2-4（单文件完成）：** 自检全部通过后标记该文件完成，进入下一文件或全局汇总。

### 3.5 Step 5：全局汇总与跨文件去重

所有参考文件处理完毕后：

```markdown
## Stage 2 完成汇总

### 处理统计
| 文件名 | 新增 | 跳过(重复) | 疑似重复 | 自检状态 |
|--------|------|-----------|---------|----------|
| 文件A | 12 | 0 | 0 | PASS |
| 文件B | 8 | 2 | 1 | PASS |

### 总计
- KM 表当前总记录数：__ 条
- 本次新增：__ 条
- 跳过（已存在）：__ 条
- 疑似重复待确认：__ 条

### 跨文件疑似重复项（如有）
| # | 文件A 节点 | 文件B 节点 | 相似度 | 建议 |
|---|-----------|-----------|--------|------|
| 1 | "合同金额审核" | "大额合同审核" | 高 | 合并为一条 |

请确认处理方案：保持现状 / 按建议合并 / 逐一指定
```

**Stage 2 最终产出：** 知识地图表（KM）填充完毕，所有记录通过质量自检，用户已确认。

---

## 四、Stage 3 — 实习（经验提取）

> **目标：** 从历史工作定稿中提炼情境化经验条目，写入经验库表（EL），并与 KM 表的知识节点建立关联。
>
> **前置要求：** 用户已将 3-5 份历史工作定稿上传至"工作归档"Wiki 节点。
>
> **预估时间：** 每份文件 ~8-15 分钟（含三轮对话交互）

### 4.0 Stage 3 启动检测

> **⚠️ C1 [CRITICAL]：Stage 3 MUST 在 Stage 1 + Stage 2 全部完成后方可执行。Stage 2 至少应已产出 KM 表数据（≥1 条），否则关联功能不可用。**

**MUST** 在开始 Stage 3 前执行：

```bash
# 1. 验证 EL 表可访问
lark-cli base +table-list --base-token $APP_TOKEN

# 2. 检查 KM 表是否有数据（Stage 3 依赖 KM 表作为背景知识）
lark-cli base +record-list --base-token $APP_TOKEN --table-id $KM_TABLE
```

| 检查项 | 通过条件 | 不通过时操作 |
|--------|----------|-------------|
| EL 表存在 | 继续 | 停止：EL表不存在，需重新执行Stage 1 |
| KM 表有数据（>= 1 条记录） | 继续 | 警告：KM表为空，关联功能不可用。建议先执行Stage 2 |
| DOMAIN_VARIABLES 已定义 | 继续 | 警告：无领域变量，第三轮对话将简化 |
| 动态领域变量字段已创建（C9） | 继续 | **停止：EL表缺少动态字段（如"谈判地位"、"交易规模"等），需重新执行 Stage 1 Step 5 创建。Stage 3 写入时依赖这些字段。** |

**Checkpoint CP-S3-0：** 记录启动状态。KM 为空则全程跳过关联步骤。

### 4.1 Step 1：读取工作文件与元信息收集

#### 4.1a 获取文档 token 并读取内容

```bash
# 使用工作归档节点 token
ARC_OBJ=$(lark-cli wiki spaces get_node \
  --params '{"token":"[ARC_NODE_TOKEN]"}' \
  --jq '.data.node.obj_token' 2>&1)

ARC_CONTENT=$(lark-cli docs +fetch --doc $ARC_OBJ 2>&1)
```

#### 4.1b 工作文件元信息收集

读取内容后向用户确认基本上下文：

```markdown
## 正在处理：《[文件名]》

请提供背景信息（帮助精准提取经验）——**3 个问题均需回答**：
1. 这份工作什么时候完成的？
2. 最终结果如何？（成功/部分成功/失败）
3. 有没有特别想记录的经验教训？
```

### 4.2 Step 2：预分析

#### 4.2a 加载知识背景

**MUST** 先读取 KM 表全量数据：

```bash
lark-cli base +record-list --base-token $APP_TOKEN --table-id $KM_TABLE
```

#### 4.2b LLM 对照分析（内部过程）

AI 执行以下分析任务（内部暂存，不直接输出给用户）：

```
输入：工作文件内容 + KM 表全部知识节点 + 用户元信息回答
输出：预分析报告（内部暂存）

分析维度：
1. 知识匹配度：
   - 列出与此工作相关的 KM 节点（按相关度排序）
   - 标注匹配方式：直接引用 / 间接相关 / 未涉及

2. 差异识别：
   - 工作做法 vs KM 标准规则的差异点
   - 每个差异标注：合理偏离 / 有风险但可控 / 明显违规

3. 领域变量快照：
   - 从文件内容和用户回答推断各 DOMAIN_VARIABLES 值
   - 标注置信度（高/中/低）

4. 提问清单生成：
   - 基于差异点和领域变量，为三轮对话准备具体问题
   - 每轮 3-5 个问题，按优先级排序
```

### 4.3 Step 3：三轮结构化对话

针对每份工作文件依次进行。**MUST** 以自然对话方式进行。

#### 第一轮：项目背景与领域变量填充

**目标：** 确定本次工作的情境坐标（领域变量值）。

```
角色：经验丰富的 [领域] 专家，正在做项目复盘
语气：好奇、开放、不评判。一次一问题，等回答后再问下一个。

引导框架：
  从 DOMAIN_VARIABLES 逐个提取：
  - 枚举类型 → 提供选项让用户选择
    例："谈判地位？[A]甲方 [B]乙方 [C]中立方"
  - 文本类型 → 开放提问
    例："交易规模大概是多少？"

  补充通用背景（若变量未覆盖）：
  - 项目时间线？主要干系人？最具挑战的环节？
```

**输出：** 完整的领域变量赋值表，存入上下文。

#### 第二轮：关键决策与偏差探究

**目标：** 挖掘"为什么这样做而非按标准流程"，提取隐性经验。

```
角色：善于追问的复盘教练
语气：尊重、深入、聚焦学习价值

提问策略（按优先级）：
  P0 — 高价值差异（合理偏离且有正面结果）：
    "这里你没走标准二审流程，当时怎么考虑的？效果怎样？"

  P1 — 有风险但可控的偏离：
    "这和标准做法不太一样，当时有什么特殊情况？"

  P2 — 用户主动提到的关键决策（来自 Step 1）

追问模板：
  - "如果再做一次，还会这样选择吗？"
  - "什么情况下你会推荐别人也这么做？"
  - "什么情况下绝对不能这么做？"
```

**输出：** 一组「情境 → 决策 → 结果」三元组。

#### 第三轮：红线底线与未来行动

**目标：** 提取不可逾越的边界条件和可复用的经验模式。

```
角色：注重风险管控的资深顾问
语气：严肃但建设性

核心问题：
  A. 红线/底线：
     - "有没有什么是你绝对不会再做的？"
     - "有什么条件是你以后绝对不会妥协的？"

  B. 可复用模式：
     - "从这次工作中，什么经验下次可以直接套用？"
     - "用一句话告诉'三个月前的自己'什么？"

  C. 触发条件细化：
     - "这条经验在什么情况下适用/不适用？"
     - "有没有什么前置条件？"
```

**输出：** 红线标记决定、经验适用范围、触发条件约束。

#### 对话结束展示

```markdown
## 对话完成，拟生成以下经验条目：

| # | 标题 | 关联知识节点 | 置信度 |
|---|------|-------------|--------|
| 1 | 大额合同审核窗口期优化 | 大额合同审核规则 | high |

确认写入 EL 表？或需要调整？
```

### 4.4 Step 4：写入经验条目并建立关联

**前置门禁： MUST** 已获得用户明确确认。

#### 4.4a 写入经验条目

> ⚠️ **D1 [CRITICAL]：`--json` 支持 `@file` 引用（见 §0.3 模式 A），含中文键名可用。**

> ⚠️ 关联知识字段格式：link 类型写入值为字符串数组 `["record_id"]`。

```bash
cat > record-el-1.json <<'EOF'
{
  "标题": "大额合同审核窗口期优化",
  "调整逻辑": "对于超过500万的合同，提前3天预约法务二审窗口期",
  "谈判地位": "甲方",
  "交易规模": "800万",
  "红线标记": false,
  "置信度": "high",
  "来源类型": "manual",
  "创建日期": "2026-04-16 00:00:00"
}
EOF

RECORD_ID=$(lark-cli base +record-upsert \
  --base-token "$APP_TOKEN" \
  --table-id "$EL_TABLE" \
  --json @record-el-1.json \
  --jq '.data.record.record_id_list[0]')
```

> **注意：** JSON 中必须包含 Stage 1 定义的所有动态领域变量字段。

#### 4.4b 建立 EL → KM 关联

仅当预分析识别到明确的 KM 节点匹配时才创建 link：

```bash
cat > record-link.json <<EOF
{
  "关联知识": ["$KM_REC_1", "$KM_REC_2"]
}
EOF

lark-cli base +record-upsert \
  --base-token "$APP_TOKEN" \
  --table-id "$EL_TABLE" \
  --record-id "$RECORD_ID" \
  --json @record-link.json
```

无匹配则留空，后续可通过事件 C 补充关联。

#### 4.4c 幂等性去重

```
去重键："标题 + 第一个枚举类型领域变量"的组合
  例："大额合同审核窗口期优化 + 甲方"

判断流程：
  全表扫描 → 组合键完全匹配 → 跳过并告知
  → 部分匹配（同标题不同变量）→ 标记疑似重复
  → 无匹配 → 写入
```

### 4.5 Step 5：全库冲突检查

每份文件写入完成后对 EL 表全局冲突扫描：

```bash
lark-cli base +record-list --base-token $APP_TOKEN --table-id $EL_TABLE
```

三类冲突检测：

| 冲突类型 | 检测条件 | 处理方案 | 自动化程度 |
|----------|----------|----------|-----------|
| 同标题不同逻辑 | 标题相似(编辑距离<=2)且调整逻辑矛盾 | 展示对比表，用户裁决 | 需人工 |
| 矛盾红线 | 同一触发条件下一条红线=true另一条违反 | 强制人工确认，二选一修改 | 强制人工 |
| 关联悬空 | link 指向的 KM record_id 不存在 | 置空+标注，建议后续补充 | 半自动 |

冲突报告格式：

```markdown
## 冲突检查结果

无冲突 → "通过，共 N 条经验条目，关联正常。"

有冲突 → 逐条展示：
  冲突 #1 [同标题不同逻辑]
  记录A："..." vs 记录B："..."
  建议：？
```

### 4.6 Step 6：用户审阅与最终确认

```markdown
## Stage 3 完成 — 《[文件名]》汇总

| 指标 | 数值 |
|------|------|
| 新增经验条目 | __ 条 |
| 跳过（已存在） | __ 条 |
| EL→KM 关联 | __ 个 |
| 红线条目 | __ 条 |
| 发现冲突 | __ 处（__ 已解决 / __ 待处理） |

请确认后回复"确认"，继续下一文件或结束 Stage 3。
```

**Checkpoint CP-S3-6：** 用户确认后标记该文件完成。

### 4.7 多文件全局汇总

多份工作文件时，逐文件重复 Step 1-6。全部完成后：

```markdown
## Stage 3 全部完成

| 文件 | 经验条目 | 关联数 | 红线数 | 状态 |
|------|----------|--------|--------|------|
| 文件A | 3 | 5 | 1 | 已确认 |
| 文件B | 2 | 3 | 0 | 已确认 |
| 合计 | 5 | 8 | 1 | — |

EL 表当前状态：__ 条记录 / __ 个关联关系
Stage 3 产出：经验库表填充完毕，知识与经验的关联网络已建立。
```

---

## 五、Stage 4 — 实际工作（运行时指令注入）

> **Stage 4 不是 Self-growing Skill Creator (Lark-Native) 自身执行的阶段。** 它是在 Stage 1 中写入子技能 SKILL.md 的**运行时指令**，供子技能部署后自行驱动。

> 以下内容在 Stage 1 Step 8（§2.7）中被注入到子技能 SKILL.md 中。

### 5.0 子技能启动序列

每次子技能被触发时按此顺序初始化：

```
步骤0: 解析本地配置(读LARK_KNOWLEDGE_BASE/DOMAIN_VARIABLES/SCOPES)
步骤1: 授权检查(5.1)
步骤2: 领域变量提取(事件A, 5.4a)
步骤3: 执行检索(5.2)
步骤4: 返回结果+触发学习事件(如适用)
```

### 5.1 授权检查与恢复

```bash
lark-cli auth status
```

| tokenStatus | scopes | 操作 |
|-------------|--------|------|
| valid | 全部包含 | 正常执行 |
| valid | 有缺失 | 列出缺失scope，生成授权命令 |
| 其他 | — | 发起完整授权流程，暂停直到完成 |

命令模板：`lark-cli auth login --scopes "[缺失scope列表]"`

### 5.2 三级检索流程

#### 5.2a 数据解析前置要求

> ⚠️ **+record-list返回的不是key-value！是二维数组！**

```json
{"fields":["列名A","列名B",...], "data":[["值A1","值B1",...],[...]], "record_id_list":["rec_xxx"]}
```

解析：fields建映射(字段名→列号) → data[i][col_idx]取值 → record_id_list[i]关联ID

#### 5.2b 优先级1 — EL表精确匹配

```
输入：用户查询 + 领域变量赋值
操作：
  1. +record-list EL_TABLE → 解析为结构化记录
  2. 领域变量精确匹配：

     枚举字段：选项完全匹配=命中
     文本字段：包含关系=命中
     红线=true → 强制纳入结果集

     匹配度评级：
       全部命中     → HIGH（直接使用）
       部分>=50%   → MEDIUM（参考使用）
       仅红线触发  → LOW（警告提示）
       无命中      → 跳过→优先级2

  3. HIGH/MEDIUM记录：读link→+record获取KM详情→组装完整回复
```

#### 5.2c 优先级2 — KM表关键词模糊匹配(优先级1无命中时)

```
n-gram匹配算法（5 步）：

步骤A-用户查询预处理：分词→去停用词→保留有意义的词元
步骤B-关键词展开：每条KM的关键词按D6 split(", ")
步骤C-打分：score(w,K)=max(1.0完全匹配 / 0.8包含 / 0.6编辑距离<=2 / 0.0无)
        记录总分=sum(score)/num_terms
步骤D-阈值：total_score>=0.5 → 命中
步骤E-排序输出：按总分降序排列命中记录
```

> D6验证通过(T4.3)：中文逗号空格分隔关键词可被LLM准确拆分(100%)

#### 5.2d 优先级3 — 参考文件全文降级(前两级均未命中)

全文作为LLM上下文。MUST标注"基于参考文件通用知识，未匹配到具体经验"(C11)

#### 5.2e 两阶段检索优化(V-7替代方案)

阶段一(可选EL>30条): view_id过滤 | 阶段二(始终): 完整匹配+LLM精排

（以上流程图已整合到 5.2b-5.2d 各小节中）

### 5.3 时间衰减排序

#### 公式
score(i) = base_score(i) * exp(-lambda(i) * days_old(i))

#### base_score赋值(可叠加)
| 属性 | 倍数 |
|------|------|
| 红线=true | x1.5 |
| 置信度=high | x1.2 |
| 来源=manual | x1.1 |
| 默认 | x1.0 |
例: 线+high+manual = **1.98**

#### lambda衰减系数
| 场景 | 值 | 半衰期 |
|------|-----|--------|
| 标准衰减 | 0.02 | ~35天 |
| 红线慢衰 | 0.002 | ~346天 |

启用条件: EL>30条 (<=30按创建日期倒序)

计算示例(今天2026-04-16):
- 记录A: 线/high/manual/15天前 → score=**1.92**
- 记录B: 无线/medium/diff/90天前 → score=**0.17**

### 5.4 持续学习三事件

#### 5.4a 事件A — 背景捕获(零成本)
触发: 用户指令时自动执行
操作: DOMAIN_VARIABLES提取变量 → 暂存CONTEXT_VARS → 传入优先级1

#### 5.4b 事件B — 反馈意图提取

| 类型 | 特征 | 处理 |
|------|------|------|
| 知识缺陷 | "规则错了" | 更新KM表 +record-upsert |
| 经验调整 | "做法不对" | 暂存待事件C |
| 流程缺陷 | "还缺一步" | 提示更新SKILL.md |

级联处理: 更新KM→检查关联EL矛盾→降矛盾EL为medium

#### 5.4c 事件C — 差分分析
触发: 用户上传新定稿
Step1: 读取定稿(wiki get_node→docs+fetch)
Step2: 差分(内部): 定稿vs已有经验 → 结果/方法/新增差异
Step3: 生成候选(1-3个): confidence=medium/source=diff_analysis
Step4: 确认后写入EL, 保持medium, 建立KM关联

---

## 六、行为约束汇总

以下约束贯穿所有 Stage，**MUST** 始终遵守：

| # | 约束 | 严重度 |
|---|------|--------|
| C1 | 严格按 Stage 顺序执行，不得跳过或在用户确认前进入下一阶段 | CRITICAL |
| C2 | 批量写入 Bitable 前必须展示摘要并获得用户确认 | CRITICAL |
| C3 | 不得假设文件内容，必须通过 `docs +fetch --doc` 实际读取 | HIGH |
| C4 | diff_analysis 来源经验必须进入待确认队列，不可自动升为 high | CRITICAL |
| C5 | 红线标记=true 的条目无论匹配度如何始终触发 | CRITICAL |
| C6 | 关联字段仅在 EL 表维护（单向 EL→KM），不在 KM 维护反向 | HIGH |
| C7 | 所有飞书 URL 作为唯一文件引用方式，不用本地路径 | HIGH |
| C8 | wiki token 必须先 `get_node` 解析 obj_token 再 `docs +fetch` | HIGH |
| C9 | 动态字段必须在 Stage 1 完成、Stage 3 首次写入前创建完毕 | HIGH |
| C10 | 大文档（>5000 字）分批处理 | MEDIUM |
| C11 | 全表无命中时降级到参考文件全文，不捏造知识 | CRITICAL |

---

## 七、异常处理速查

| 异常 | 处理 |
|------|------|
| `tokenStatus` ≠ valid | 发起授权链接，等完成后重试；不继续 API 调用 |
| 网络 timeout | 静默重试一次；仍失败则提示"飞书连接超时" |
| `docs +fetch` 返回空 | 检查文件格式和权限 |
| Bitable 写入失败 | 展示失败行数据，请求确认后重试 |
| 全表无命中 | 降级到参考文件全文（C11） |
| EL 表 > 100 行 | 切换有限检索策略 + 预警（§5.5） |
