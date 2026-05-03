# 子技能 SKILL.md 生成模板
# ASC for Lark Stage 1 Step 8 使用此模板生成子技能的 SKILL.md 文件

> **使用方式：** 将以下模板中的 `[占位符]` 替换为 Stage 1 收集的实际值后写入子技能目录。
>
> **版本对应：** 与 ASC for Lark SKILL.md v1.0+ 配套使用。运行时指令部分（下半部分）来自 Stage 4 规范。

---
name: [技能名称]
version: 1.0.0
description: "[技能一句话描述]。当用户需要[触发场景]时使用。"
---

# [技能名称]

## 飞书知识库配置（由 ASC for Lark Stage 1 自动生成，勿手动修改）

> ⚠️ 以下配置块的任何手动修改可能导致检索功能异常。

LARK_KNOWLEDGE_BASE:
  bitable_app_token: "[APP_TOKEN]"
  wiki_space_id: "[SPACE_ID]"
  tables:
    knowledge_map: "[KM_TABLE_ID]"
    experience_log: "[EL_TABLE_ID]"
  wiki_nodes:
    reference: "[REF_TOKEN]"
    archives: "[ARC_TOKEN]"

## 领域变量定义（由 ASC for Lark Stage 1 自动生成，可手动维护）

DOMAIN_VARIABLES:
# 从需求访谈中收集的领域变量列表：
  - name: "[变量名]"
    type: enum  # 或 text
    options: [选项1, 选项2]  # enum 类型专用，text 类型省略此行
    hint: "[从用户语言识别此变量的提示]"
    el_field: "[对应 EL 表中的精确字段名]"

## 飞书授权 Scope 清单（子技能启动时一次性检查）

LARK_REQUIRED_SCOPES: >
  wiki:wiki:readonly wiki:wiki:write
  bitable:app bitable:app:readonly
  docs:doc:readonly docs:doc:write
  lark:auth

---

# ═══════════════════════════════════════════
# 运行时指令（由 ASC for Lark Stage 4 注入）
# ═══════════════════════════════════════════

## 启动序列

每次子技能被触发时按顺序执行：

```
步骤0: 解析上方配置块（LARK_KNOWLEDGE_BASE / DOMAIN_VARIABLES / SCOPES）
步骤1: 授权检查（下方 §授权检查）
步骤2: 领域变量提取（§事件A — 背景捕获）
步骤3: 执行检索（§三级检索流程）
步骤4: 返回结果 + 触发学习事件（如适用）
```

## 授权检查

每次会话首次调用前执行一次：

```bash
lark-cli auth status
```

| tokenStatus | scopes | 操作 |
|-------------|--------|------|
| valid | 全部包含 | 正常执行 |
| valid | 有缺失 | 列出缺失项 → 生成 `lark-cli auth login --scopes "..."` |
| 其他 | — | 发起授权流程，暂停直到完成 |

## 三级检索流程

### 数据解析前置要求（所有优先级通用）

> ⚠️ **+record-list 返回的是二维数组，不是 key-value！**

```json
{"fields":["列名A","列名B",...], "data":[["值A1","值B1",...],[...]], "record_id_list":["rec_xxx"]}
```

**解析方法：**
1. 先读 `fields[]` 建立映射：字段名 → 列索引号
2. 再遍历 `data[i]` 用列索引取值：`data[i][col_index_of_字段名]`
3. 用 `record_id_list[i]` 关联记录 ID

### 优先级 1 — 经验库表（EL）精确匹配

**目标：** 按领域变量值找到最相关的情境化经验。

```bash
lark-cli base +record-list --base-token [APP_TOKEN] --table-id [EL_TABLE_ID]
```

**匹配算法：**

```
对每条 EL 记录：
  1. 提取动态领域变量字段的值（如"谈判地位"、"交易规模"等）
  2. 与当前查询中事件 A 提取的 CONTEXT_VARS 对比：

     匹配规则：
       枚举类型字段：选项完全匹配 = 命中
       文本类型字段：包含关系 = 命中

     红线特殊处理：
       若 红线标记=true → 无论其他匹配度如何，强制纳入结果

     匹配度评级：
       所有领域变量命中      → HIGH （直接采用）
       部分命中（>=50%）    → MEDIUM （参考使用）
       仅红线触发           → LOW （警告性提示）
       无命中               → 跳过，进入优先级 2

  3. 对 HIGH/MEDIUM 记录：
     a) 读取 link 字段（关联知识数组，格式 ["rec_xxx", ...]）
     b) 对每个 KM record_id 执行 +record-get 获取详情
     c) 组装完整的「情境化经验 + 关联知识规则」回复
```

### 优先级 2 — 知识地图表（KM）关键词模糊匹配

**触发条件：** 优先级 1 未产生 HIGH/MEDIUM 命中时执行。

```bash
lark-cli base +record-list --base-token [APP_TOKEN] --table-id [KM_TABLE_ID]
```

**n-gram 关键词匹配算法：**

```
步骤 A — 用户查询预处理：
  - 分词（按中文词汇边界拆分）
  - 去停用词（的、了、是、在、这、那...）
  - 保留有意义的词元

步骤 B — 关键词展开：
  - 对每条 KM 记录读取"关键词"字段值
  - 按 D6 分隔符 split("， ") 得到关键词数组

步骤 C — 打分：
  对每个查询词元 w 和每条记录的关键词数组 K：
    score(w, K) = max(
      1.0,   // w 完全等于 K 中某个词
      0.8,   // K 中某词包含 w 或 w 包含 K 中某词
      0.6,   // 编辑距离(w, k) <= 2
      0.0    // 无匹配
    )
  记录总分 = sum(score(w_i, K)) / num_query_terms

步骤 D — 阈值判定：
  total_score >= 0.5 → 命中，纳入结果集
  total_score < 0.5 → 未命中

步骤 E — 排序输出：
  按总分降序排列命中记录
  输出包含：节点名称、核心规则、参考来源、红线状态
```

### 优先级 3 — 参考文件全文降级

**触发条件：** 优先级 1 和 2 均未命中时。

```bash
# 获取参考文件节点的 obj_token
REF_OBJ=$(lark-cli wiki spaces get_node `
  --params '{"token":"[REF_TOKEN]"}' `
  --jq '.data.node.obj_token' 2>&1)

# 读取全文
lark-cli docs +fetch --doc $REF_OBJ
```

将全文作为 LLM 上下文回答用户问题。

**⚠️ 必须在回复中标明：**
> "以下回答基于参考文件的通用知识，未匹配到具体的经验条目或知识节点。"

## 时间衰减排序

**启用条件：** EL 表记录数 > 30 条时启用（<= 30 条按创建日期倒序即可）。

**公式：**

```
score(i) = base_score(i) × exp(-λ(i) × days_old(i))

其中 days_old(i) = 当前日期 - 记录创建日期（天）
```

**base_score 赋值表（可叠加）：**

| 属性 | 条件 | 倍数 |
|------|------|------|
| 红线标记 | true | ×1.5 |
| 置信度 | high | ×1.2 |
| 来源类型 | manual | ×1.1 |
| 默认 | — | ×1.0 |

**叠加示例：** 红线(true) + high置信度 + manual来源 = 1.5 × 1.2 × 1.1 = **1.98**

**衰减系数 λ：**

| 场景 | λ 值 | 半衰期 |
|------|------|--------|
| 标准衰减 | 0.02 | ~35 天 |
| 线慢衰减 | 0.002 | ~346 天（红线专用）|

计算完成后按 score 降序重排，再送入匹配算法。

## 持续学习三事件

### 事件 A — 背景捕获（零成本，每次查询自动执行）

**操作：**
1. 从上方 `DOMAIN_VARIABLES:` 块读取变量定义
2. 分析用户的自然语言指令
3. 提取每个变量的值 → 存入会话上下文 `CONTEXT_VARS`
4. 传入「优先级 1」的匹配算法中使用

**推断原则：**
- 明确提及的变量 → 直接取值
- 可从上下文推断的 → 标注置信度(高/中/低)
- 完全无法推断的 → 留空，不猜测

### 事件 B — 反馈意图提取

**触发：** 用户表达纠错/修改意图时。

| 反馈类型 | 识别特征 | 处理动作 | API 调用 |
|----------|----------|----------|----------|
| **知识级缺陷** | "这条规则错了""应该改成..." | 定位 KM 行 → 更新字段 | `+record-upsert` |
| **经验级调整** | "上次那种做法不太对" | 暂存到待写队列 | 无（内存暂存）|
| **流程级缺陷** | "还缺一步""应该先X再Y" | 提示用户更新本 SKILL.md | 无 |

**级联处理（仅知识级缺陷）：**
1. 更新 KM 表对应行
2. 检查是否有 EL 条目通过 link 关联到此 KM 记录
3. 若存在且调整逻辑与新规则矛盾 → 该 EL 条目 confidence 降为 medium
4. 向用户报告变更和级联影响

### 事件 C — 差分分析

**触发：** 用户上传新的工作定稿时。

**完整流程：**

```
Step 1 — 读取定稿
  wiki spaces get_node → 获取 obj_token → docs +fetch

Step 2 — 差分分析（内部 LLM 过程）
  输入：定稿内容 + 相关已有经验（通过领域变量匹配找到的）
  分析维度：
    a) 结果差异：最终结果与经验预期是否一致？
    b) 方法差异：实际做法 vs 已有调整逻辑
    c) 新增情境：是否覆盖了之前没有的新情况？

Step 3 — 生成候选条目
  基于 Step 2 差异生成 1-3 个候选经验条目
  每个候选设置：confidence=medium, source=diff_analysis

Step 4 — 用户确认
  展示候选条目 → 用户确认/调整
  确认后：
    a) 写入 EL 表（+record-upsert）
    b) 保持 confidence=medium（需后续人工升为 high）
    c) 建立 KM 关联（若适用）
```

**产出格式：**

```markdown
## 📋 事件 C — 差分分析结果

基于《[新定稿名]》与已有经验的对比：

**发现差异：**
1. [具体差异描述]

**建议新增的经验条目：**
| 标题 | 调整逻辑 | 建议关联知识 | 来源 |
|------|----------|-------------|------|
| [候选1] | [...] | [KM节点名] | diff_analysis |

请确认或调整以上内容。
```

## 规模超限降级

**触发条件：** KM 表 > 50 行 或 EL 表 > 100 行

| 步骤 | 操作 |
|------|------|
| 1 | 红线记录全部保留 |
| 2 | 非红线记录按 time_decay_score 降序取 Top-N |
| 3 | N = max(20, floor(总记录数 × 0.3)) |
| 4 | 向用户预警："共 X 条，显示前 Y 条（红线 Z 条全部保留）" |
| 5 | 可选提供「查看全部」扩展入口 |

---

*本文件由 ASC for Lark Stage 1 自动生成。运行时指令部分由 Stage 4 注入。
手动维护 DOMAIN_VARIABLES 块即可，其余配置块请勿修改。*
