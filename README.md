# Self-growing Skill Creator (Lark-Native) / 自我进化的技能创建工具（飞书版）

> 🏆 **飞书 CLI 创作者大赛参赛作品** | GitHub 技术赛道
>
> **让 AI 技能像人类一样从工作中持续学习和进化。**

```
传统 Skill        →  写死规则，越用越僵化
Self-growing Skill → 从文档中提取知识，从工作中积累经验，越用越聪明
```

---

## 🎯 一句话说明

**Self-growing Skill Creator** 是一个基于飞书 CLI 的 **AI 元技能（Meta-Skill）**——它不解决具体业务问题，而是**帮你在 15 分钟内创建一个能持续自我进化的专业领域 AI 技能**。所有知识存储在你自己的飞书 Wiki + Bitable 中，数据 100% 自主可控。

**适用场景：** 合同审核、代码审查、产品文案检查、客服话术优化、合规风险评估……任何需要"领域知识 + 情境经验"的专业工作。

---

## ✨ 五大核心创新

### 1. Meta-Skill 架构 —— "造 Skill 的 Skill"

传统思路：为每个场景写一个 Skill → 规则写死 → 无法进化  
**本作品**：写一个 Meta-Skill → 通过对话生成任意领域的子 Skill → 子 Skill 在工作中自动学习进化

```
Meta-Skill (本作品)
    │
    ├── 生成 → 合同审核助手
    ├── 生成 → 代码审查助手
    ├── 生成 → 产品文案检查助手
    └── 生成 → [任意领域] 专业助手
              │
              └── 在实际工作中持续积累经验，越用越准
```

### 2. 自我进化闭环 —— 知识越用越聪明

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Stage 2     │    │  Stage 3     │    │  Stage 4     │
│  知识提取     │ →  │  经验积累     │ →  │  运行时学习   │
│  (参考文件)   │    │  (工作定稿)   │    │  (用户交互)   │
└──────────────┘    └──────────────┘    └──────┬───────┘
      ↑                                         │
      └──────────── 持续更新知识库 ←─────────────┘
```

- **知识提取**：从 Playbook/SOP/规范中自动提取结构化知识节点
- **经验积累**：从历史工作定稿中挖掘情境化经验（"在什么情况下应该怎么做"）
- **运行时学习**：每次用户交互自动触发学习事件，反馈修正、差分分析、背景捕获
- **时间衰减**：旧经验自动降权，确保知识库始终反映最新实践

### 3. 飞书 CLI 原生深度整合 —— 零外部依赖

| 能力 | 实现方式 | 飞书 CLI 命令 |
|------|----------|---------------|
| 知识存储 | 飞书 Wiki 空间 | `lark-cli wiki nodes create` |
| 结构化数据 | Bitable 多维表格 | `lark-cli base +table-create` |
| 文档读取 | 飞书文档 API | `lark-cli docs +fetch` |
| 数据检索 | Bitable 记录查询 | `lark-cli base +record-list` |
| 权限管控 | 飞书原生权限 | 飞书 GUI 表级/字段级权限 |

**全部数据存储在用户自己的飞书空间中**，不经过任何第三方服务器，企业级安全合规。

### 4. 零代码构建专业 AI —— 对话即开发

无需写一行代码，通过自然对话完成全部配置：

```
用户："我想做一个合同审核助手"
AI："好的。这个技能主要帮你审核什么类型的合同？"
用户："主要是采购合同"
AI："有哪些关键变量会影响审核逻辑？比如甲方/乙方身份、金额大小？"
……（5分钟对话后）……
AI："已为你创建完成！知识库结构：领域变量=谈判地位/交易规模/合同类型，
      知识地图表已创建，经验库表已创建。现在可以上传参考文件开始提取知识了。"
```

### 5. 团队协作者模式 (v1.2) —— 多人共建知识库

```
团队成员A ──┐
团队成员B ──┼──→ 经验库(待审状态) ──→ 管理员审核 ──→ 晋升为正式知识
团队成员C ──┘         (飞书GUI操作)        (Bitable单选字段)
```

- 多人可并行写入经验，默认进入"待审"状态
- 管理员在飞书 Bitable GUI 中一键审核（单选字段 + 视图筛选）
- 审核后自动晋升为正式知识，全员实时共享
- 新成员通过 `space_id + app_token` 1 分钟接入已有库

---

## 🎬 3 分钟体验：做一个"开源 License 审阅助手"

### Step 1 — 启动（1 分钟）

```bash
# 安装飞书 CLI（如未安装）
npm install -g @larksuite/cli

# 授权
lark-cli auth login --scopes "wiki:wiki:readonly,wiki:wiki:write,bitable:app,bitable:app:readonly,docs:doc:readonly,docs:doc:write,lark:auth"

# 验证
lark-cli auth status
```

### Step 2 — 加载 Meta-Skill（1 分钟）

将 `SKILL.md` 加载到支持 Skill 的 AI Agent（如 Claude Code、Cursor、Trae 等），输入：

> "帮我创建一个开源 License 审阅助手"

### Step 3 — 对话配置（10 分钟）

AI 会引导你完成：
1. **需求访谈**：确认技能目标、触发场景、输出格式
2. **领域变量**：License 类型 / 使用方式 / 商业场景 / 分发方式 / 企业角色 / 合规级别
3. **一键创建**：Wiki 空间骨架 + Bitable 两张表 + 子技能 SKILL.md

### Step 4 — 注入知识（5 分钟/文件）

上传参考文件（如《GPL 义务速查表》《Apache 合规指南》），AI 自动：
- 提取结构化知识节点 → 写入知识地图表
- 识别红线规则 → 标记 Checkbox
- 去重自检 → 确保知识库干净

### Step 5 — 投入使用

子技能已可独立运行。输入一段代码引入声明：

> "这段代码引入了 MIT License 的库，我们商用有什么风险？"

子技能自动执行：
1. 提取领域变量（License 类型=MIT，使用方式=引入，商业场景=商用）
2. 三级检索：EL 精确匹配 → KM 关键词匹配 → 参考文件全文降级
3. 返回：风险评级 + 具体义务条款 + 应对建议

---

## 🏗️ 技术架构

### 数据模型：双层知识架构

```
知识地图表 (KM)                    经验库表 (EL)
┌─────────────┐                   ┌─────────────┐
│ 节点名称      │                   │ 标题         │
│ 核心规则      │◄────link字段────►│ 调整逻辑      │
│ 关键词        │                   │ 领域变量值    │
│ 规则类型      │                   │ 置信度        │
│ 红线标记 ★   │                   │ 来源类型      │
│ 创建日期      │                   │ 创建日期      │
└─────────────┘                   │ 状态(待审/通过)│  ← v1.2 新增
                                  └─────────────┘
```

### 运行时检索：三级递进

```
用户查询 + 领域变量
        │
        ▼
┌─────────────────┐
│ P1: EL 精确匹配  │ ── 领域变量完全匹配 → HIGH（直接使用）
│                 │ ── 部分匹配 → MEDIUM（参考使用）
│                 │ ── 仅红线触发 → LOW（警告提示）
└─────────────────┘
        │ 无命中
        ▼
┌─────────────────┐
│ P2: KM 模糊匹配  │ ── n-gram 关键词评分 → score >= 0.5 命中
└─────────────────┘
        │ 无命中
        ▼
┌─────────────────┐
│ P3: 参考文件降级 │ ── 全文作为 LLM 上下文，标注"基于通用知识"
└─────────────────┘
```

### 时间衰减排序

```
score = base_score × exp(-λ × days_old)

base_score 叠加：红线=×1.5 | 高置信度=×1.2 | 人工来源=×1.1

例：红线 + high + manual / 15天前 → score = 1.98
   无线 + medium + diff / 90天前 → score = 0.17
```

### 与飞书 CLI 的命令级整合

本作品的每一个操作都对应飞书 CLI 的原生命令，零封装、零抽象：

| 功能 | 命令 | 参数模式 |
|------|------|----------|
| 创建 Wiki 节点 | `lark-cli wiki nodes create` | `--params` + `--data` 双参数 |
| 创建 Bitable 表 | `lark-cli base +table-create` | `--fields @fields.json` |
| 添加表字段 | `lark-cli base +field-create` | `--json @payload.json` |
| 读取飞书文档 | `lark-cli docs +fetch` | `--doc $DOC_TOKEN` |
| 查询记录 | `lark-cli base +record-list` | `--base-token $APP_TOKEN` |
| 写入记录 | `lark-cli base +record-upsert` | `--record $RECORD_ID` |

**JSON 参数传递方案**（经 lark-cli v1.0.4 实测验证）：
- `--json/--fields` → `@file.json` 引用（支持中文）
- `--data/--markdown` → `"$(cat file.json)"` bash 替换（支持中文）
- `--params` → 内联 ASCII 简单对象

---

## 📊 Before / After 对比

| 维度 | 传统 AI Skill | Self-growing Skill |
|------|--------------|-------------------|
| 知识更新 | 手动改 Prompt | 上传新文件自动提取 |
| 经验积累 | 无 | 从每次工作定稿自动学习 |
| 领域适配 | 通用回答 | 基于你的业务规则精确回答 |
| 团队协作 | 单人使用 | 多人共建 + 审核机制 |
| 数据归属 | 依赖第三方 | 100% 存储在你的飞书空间 |
| 进化成本 | 需程序员维护 | 业务人员对话即可维护 |

---

## 🚀 快速开始

### 前置条件

```bash
# 1. 安装飞书 CLI
npm install -g @larksuite/cli

# 2. 授权（一次性）
lark-cli auth login --scopes "wiki:wiki:readonly,wiki:wiki:write,bitable:app,bitable:app:readonly,docs:doc:readonly,docs:doc:write,lark:auth"

# 3. 创建 Wiki 空间（飞书客户端 → 知识库 → 新建空间 → 复制 space_id）
```

### 使用方式

将 `SKILL.md` 加载到支持 Skill 的 AI Agent，按提示选择模式：

- **[A] 标准模式**（~15-20 分钟）：完整需求访谈，定制化字段结构
- **[B] 快速模板**（~3-5 分钟）：预设默认配置，一键部署
- **[C] 接入已有库**（~1-2 分钟）：团队成员加入现有知识库

---

## 📖 四阶段工作流

### Stage 1 — 初创（建立知识空间骨架）

通过对话引导完成需求访谈，然后在飞书中搭建：
- Wiki 三级节点（根节点 / 参考文件 / 工作归档）
- Bitable App「知识与经验库」
- 知识地图表 KM（7 字段）+ 经验库表 EL（11 字段，含 v1.2 审核字段）
- 子技能 SKILL.md（含完整资源标识符和运行时指令）

**幂等性保障**：每个 Step 前有 Checkpoint 检测，重复执行不产生脏数据，支持断点续传。

### Stage 2 — 知识提取（从参考文件到 KM 表）

上传 Playbook / SOP / 规范等参考文件，AI 自动：
- 五类文档适配策略（纯文字 / 中等 / 大型 / 表格密集 / 代码）
- LLM 结构化提取（7 字段，含关键词规范）
- 智能三路去重（完全匹配跳过 / 高度相似待确认 / 新增写入）
- Q1-Q7 质量自检清单

### Stage 3 — 经验积累（从工作定稿到 EL 表）

上传历史工作定稿，AI 执行三轮深化对话：
- **第一轮**：项目背景与领域变量填充
- **第二轮**：关键决策探究（P0/P1/P2 优先级）
- **第三轮**：红线底线与未来行动

写入前执行三类冲突检测：同标题不同逻辑 / 矛盾红线 / 关联悬空。

### Stage 4 — 运行时（子技能自主执行）

子技能被触发时自动执行：
```
用户查询 → 授权检查 → 领域变量提取(事件A) → 三级检索 → 结果返回 + 学习事件(B/C)
```

持续学习三事件：
- **事件 A**：背景捕获（零成本，每次交互自动执行）
- **事件 B**：反馈意图提取（"规则错了"→更新 KM / "做法不对"→更新 EL）
- **事件 C**：差分分析（上传新定稿 → 对比差异 → 生成候选经验）

---

## 🧪 质量保障

### E2E 测试覆盖

| 测试项 | 数量 | 状态 |
|--------|------|------|
| T2.2 Wiki 节点创建 | 3 项 | ✅ 通过 |
| T2.3 Bitable App 挂载 | 2 项 | ✅ 通过 |
| T2.4 表与字段创建 | 5 项 | ✅ 通过 |
| T3.1 文档读取 | 3 项 | ✅ 通过 |
| T3.2 知识提取 | 4 项 | ✅ 通过 |
| T3.3 批量写入 | 2 项 | ✅ 通过 |
| T4.1 领域变量提取 | 2 项 | ✅ 通过 |
| T4.2 三级检索 | 3 项 | ✅ 通过 |
| T4.3 关键词拆分 | 1 项 | ✅ 通过 |
| G1-G7 综合场景 | 7 项 | ✅ 通过 |
| **合计** | **79 项** | **✅ 0 Blocker** |

### 工程约束（C1-C11）

| 约束 | 说明 |
|------|------|
| C1 | 严格按 Stage 顺序执行，不得跳过 |
| C2 | 批量写入前必须展示摘要并获确认 |
| C3 | 不得假设文件内容，必须实际读取 |
| C4 | diff_analysis 来源经验不自动升为 high |
| C5 | 红线标记=true 的条目无论匹配度始终触发 |
| C6-C11 | 关联方向 / 飞书 URL / get_node 链路 / 动态字段 / 大文档分批 / 不捏造知识 |

---

## 📁 项目结构

```
self-growing-skill-creator-lark/
├── SKILL.md                      # 元技能主文件（AI 直接执行）
├── README.md                     # 本文件
├── DEV-PLAN.md                   # 开发计划与进度追踪
├── templates/
│   ├── quick-template.yaml       # 快速模板默认配置
│   ├── stage1-interview.md       # Stage 1 访谈引导 prompt
│   ├── stage2-extract.md         # Stage 2 知识提取 prompt
│   ├── stage3-dialogue.md        # Stage 3 三轮对话框架
│   └── subskill-skeleton.md      # 子技能 SKILL.md 生成模板
└── tests/
    ├── e2e-test-plan.md          # E2E 测试计划（69 检查项）
    ├── e2e-test-results.md       # E2E 测试结果报告
    ├── live-e2e-test-plan.md     # 线上环境测试计划
    └── live-e2e-test-results.md  # 线上环境测试结果
```

---

## 🏅 参赛信息

| 项目 | 内容 |
|------|------|
| **大赛** | 飞书 CLI 创作者大赛 |
| **赛道** | GitHub 技术赛道 |
| **作品名称** | Self-growing Skill Creator (Lark-Native) |
| **核心依赖** | [lark-cli](https://github.com/larksuite/cli)（飞书官方 CLI） |
| **GitHub 仓库** | https://github.com/JoshuaLI-AI/Self-growing-Skill-Creator-Lark-Native- |
| **数据存储** | 用户自有飞书空间（Wiki + Bitable） |
| **许可协议** | MIT |

### 评委快速验证指引

```bash
# 1. 安装 lark-cli
npm install -g @larksuite/cli

# 2. 授权
lark-cli auth login --scopes "wiki:wiki:readonly,wiki:wiki:write,bitable:app,bitable:app:readonly,docs:doc:readonly,docs:doc:write,lark:auth"

# 3. 将 SKILL.md 加载到 AI Agent，输入"创建一个开源 License 审阅助手"
# 4. 跟随对话完成配置（约 15 分钟）
# 5. 体验完整四阶段工作流
```

---

## 📝 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0 | 2026-04-22 | MVP 发布；E2E 测试通过（79 项 / 0 Blocker） |
| v1.1 | 2026-04-23 | 工程硬化：验证 lark-cli v1.0.4 `@file` 支持；Bitable 表名英文化；清除 `node -e` 嵌套调用 |
| **v1.2** | **2026-05-04** | **团队协作者模式**：多人写入 + 管理员审核晋升；接入已有库模式；环境自动安装检查 |

---

*本作品由 Self-growing Skill Creator 框架驱动开发，完全基于飞书 CLI 官方 OpenAPI 构建。*
