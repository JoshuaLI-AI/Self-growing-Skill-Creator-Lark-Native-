# ASC for Lark — 飞书知识库元技能

> **版本：** v1.0 MVP  
> **规格依据：** `2026-04-08-asc-for-lark-spec.md` (v1.5)  
> **E2E 测试状态：** ✅ 通过（79 项检查，0 Blocker，G1-G7 全通过）

---

## 一、这是什么？

**ASC for Lark** 是一个 **AI 元技能（Meta-Skill）**——它不直接参与业务工作流，而是帮你**创建一个能持续学习和进化的专业领域飞书技能**。

### 核心价值

| 痛点 | 解决方案 |
|------|----------|
| AI 助手不懂你的业务规则 | 从参考文件中提取结构化知识库 |
| 经验随人员流失 | 从工作定稿中自动积累情境化经验 |
| 技能无法进化 | 三级检索 + 时间衰减 + 持续学习事件 |
| 数据散落各处 | 统一存储在飞书 Wiki + Bitable |

### 产物一览

运行完成后，你将获得：

```
飞书 Wiki 空间
├── Bitable App「知识与经验库」
│   ├── 知识地图表 (KM)    ← 结构化知识节点
│   └── 经验库表 (EL)      ← 情境化经验条目
├── 参考文件节点           ← 原始参考文档
├── 工作归档节点           ← 历史工作定稿
└── 技能介绍文档

本地文件
└── [技能名称]/SKILL.md    ← 生成的子技能（可独立部署）
```

---

## 二、快速上手（5 分钟体验）

### 前置条件

1. **安装 lark-cli：** 确保 `lark-cli` 已安装且版本 ≥ 1.0.4
2. **飞书授权：**
   ```bash
   lark-cli auth login --scopes "wiki:wiki:readonly,wiki:wiki:write,bitable:app,bitable:app:readonly,docs:doc:readonly,docs:doc:write,lark:auth"
   ```
3. **创建 Wiki 空间：** 飞书客户端 → 知识库 → 新建空间 → 复制 `space_id`

### 使用方式

在支持 Skill 的 AI Agent 中加载 `SKILL.md`，按提示选择：

- **[A] 标准模式**（~15-20 分钟）：完整需求访谈，定制化字段结构
- **[B] 快速模板**（~3-5 分钟）：预设默认配置，一键部署

### 标准模式流程概览

```
Stage 1 初创 (~15min)        Stage 2 提取 (~5min/文件)     Stage 3 积累 (~10min/文件)
┌─────────────┐             ┌─────────────────┐           ┌─────────────────┐
│ 需求访谈      │ ──→         │ 读取参考文件       │ ──→       │ 读取工作定稿       │
│ → 建Wiki骨架  │             │ → LLM提取知识节点  │           │ → 三轮对话深挖     │
│ → 创建Bitable │             → 批量写入KM表     │           → 写入EL表+关联KM   │
│ → 写子技能MD  │             → 质量自检去重      │           → 冲突检测+确认     │
└─────────────┘             └─────────────────┘           └─────────────────┘
                                                                      ↓
                                                           Stage 4 运行时（自动）
                                                        三级检索 + 学习事件
```

---

## 三、四阶段详解

### Stage 1 — 初建骨架

**输入：** 用户需求访谈（技能目标 / 触发场景 / 领域变量）  
**输出：** Wiki 空间 + Bitable 两张表 + 子技能 SKILL.md

| Step | 操作 | 产物 |
|------|------|------|
| 1 | 需求访谈（收集领域变量） | 领域变量定义 |
| 2 | 创建 Wiki 三级节点（根/参考/归档） | ROOT/REF/ARC token |
| 3 | 在 Wiki 内创建 Bitable App | APP_TOKEN |
| 4 | 创建 KM 表(7字段) + EL 表(7固定+ N动态字段) + link 关联 | KM_TABLE / EL_TABLE |
| 5 | 为每个领域变量创建 EL 动态字段 | 动态字段 |
| 6 | 创建并挂载技能介绍文档 | DOC_TOKEN |
| 8 | 写入子技能 SKILL.md | 子技能文件 |

**幂等性保障：** 每个 Step 前都有 Checkpoint 检测（CP-S1-2 ~ CP-S1-8），重复执行不产生脏数据。

---

### Stage 2 — 知识提取

**输入：** 参考文件（Playbook/SOP/规范等）  
**输出：** KM 表中的结构化知识节点

核心能力：
- **5 类文档适配策略**（纯文字/中等/大型/表格密集/代码）
- **LLM 结构化提取 prompt**（7 字段，含 D6 关键词规范）
- **智能三路去重**（完全匹配跳过 / 高度相似待确认 / 新增写入）
- **Q1-Q7 质量自检清单**

---

### Stage 3 — 经验积累

**输入：** 历史工作定稿  
**输出：** EL 表中的情境化经验条目 + 与 KM 的关联网络

核心能力：
- **预分析框架**（4 维度：匹配度/差异识别/变量快照/提问清单）
- **三轮深化对话**：
  - 第一轮：项目背景与领域变量填充
  - 第二轮：关键决策探究（P0/P1/P2 优先级）
  - 第三轮：红线底线与未来行动
- **三类冲突检测**（同标题不同逻辑 / 矛盾红线 / 关联悬空）

---

### Stage 4 — 运行时注入

**这是写入子技能的指令，非 ASC for Lark 自身执行。**  
子技能被触发时自动执行：

```
用户查询 → 授权检查 → 变量提取(A) → 三级检索 → 结果返回 + 学习事件(B/C)

三级检索：
  P1. EL 精确匹配（领域变量 → HIGH/MEDIUM/LOW）
  P2. KM n-gram 模糊匹配（关键词 → score >= 0.5）
  P3. 参考文件全文降级（C11 免责标注）

时间衰减：score = base_score × exp(-λ × days_old)
学习事件：A背景捕获 / B反馈意图 / C差分分析
```

---

## 四、行为约束（C1-C11）

| # | 约束 | 严重度 | 落地位置 |
|---|------|--------|----------|
| C1 | 严格按 Stage 顺序执行 | CRITICAL | §3.0 / §4.0 启动断言 |
| C2 | 批量写入前必须获确认 | CRITICAL | §3.2d / §4.6 |
| C3 | 必须用 docs +fetch 实际读取 | HIGH | §3.1b / §4.1a |
| C4 | diff_analysis 不自动升 high | CRITICAL | §5.4c |
| C5 | 红线标记始终触发 | CRITICAL | §5.2b |
| C6 | 仅 EL→KM 单向关联 | HIGH | 字段定义 |
| C7 | 仅使用飞书 URL | HIGH | §3.2b |
| C8 | get_node → fetch 链路 | HIGH | §3.1a / §4.1a |
| C9 | 动态字段 Stage 1 完成 | HIGH | §2.5 定义 + §4.0 检查 |
| C10 | 大文档 >5000字分批 | MEDIUM | §3.1c 分批策略 |
| C11 | 全文降级不捏造知识 | CRITICAL | §5.2d |

---

## 五、开发者注意事项（D1-D6）

> 这些约束已在 SKILL.md 正文中以 ⚠️D-N 标记落地为执行约束。

| 编号 | 约束 | 影响 |
|------|------|------|
| D1 [CRITICAL] | bash shell 下 JSON 参数分三种传递模式：`--json/--fields` 用 `@file.json`，`--data/--markdown` 用 `"$(cat file)"`，`--params` 内联 ASCII | §0.3 所定义的全部命令块 |
| D2 [CRITICAL] | link 字段的 `link_table` 必须传 table_id（格式 `tbl...`），不能传表名 | §2.4c EL→KM 关联字段创建 |
| D3 [CRITICAL] | `wiki nodes create` 必须使用 `--params` + `--data` 双参数格式 | 所有 Wiki 节点创建命令 |
| D6 [CRITICAL] | 关键词分隔符仅使用中文逗号+空格(`, `)，禁止英文逗号等其他符号 | §3.2b 提取 + §5.2c n-gram 匹配 |

---

## 六、项目结构

```
asc-for-lark/
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
    └── e2e-test-results.md       # E2E 测试结果报告
```

---

## 七、FAQ

**Q: 快速模式和标准模式的区别？**  
A: 快速模式跳过需求访谈，使用通用默认配置（~5 分钟）。标准模式完整收集领域变量和定制化字段（~15-20 分钟）。两者产出的结构完全兼容，可随时从快速升级到标准。

**Q: 数据存在哪里？安全吗？**  
A: 所有数据存放在**你自己的飞书空间**中（Bitable + Wiki）。ASC for Lark 不经过任何第三方服务器。lark-cli 通过飞书官方 OpenAPI 访问。

**Q: 可以中途停止吗？**  
A: 可以。每个关键步骤都有 Checkpoint 检测（CP-S1-x / CP-S2-0 / CP-S3-0 等），重新启动时会自动检测已完成的部分，支持断点续传。

**Q: 子技能可以脱离 ASC for Lark 独立运行吗？**  
A: 可以。Stage 1 生成的 `SKILL.md` 包含完整的飞书资源标识符、领域变量定义和运行时检索指令，可作为独立 Skill 加载到任何支持的 AI Agent 中。

**Q: 支持哪些文档格式？**  
A: 飞书原生文档（docx）最佳支持。其他格式建议先转为飞书文档以确保 `docs +fetch` 可正常读取。

---

## 八、版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.0 | 2026-04-22 | MVP 发布；E2E 测试通过（79 项 / 0 Blocker）；Phase 4 修复 13 个 WARN |
| v1.1 | 2026-04-23 | **工程硬化**：验证 lark-cli v1.0.4 的 `@file` 支持（dry-run 证实），弃用脆弱的 Node.js 中转方案，改用 bash `@file` + `$(cat)` + 内联三模式；Bitable 表名切换为英文 `KnowledgeMap`/`ExperienceLog`（修 REAL-m02）；所有 `node -e` 嵌套调用已清除；补充 `el_field` 前导空格 trim 保护 |

---

*本项目由 ASC (Advanced Skill Creator) 框架驱动开发。*
