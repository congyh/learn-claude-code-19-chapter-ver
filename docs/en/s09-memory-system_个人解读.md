# s09-memory-system 个人解读

## 分析: memory system的原理是什么?

### 核心问题
每次新会话，agent 从零开始 —— 不记得用户偏好、被纠正过的错误、项目约束。Memory system 解决的就是跨会话的持久化问题。

### 存储结构
每条记忆是一个带 YAML frontmatter 的 Markdown 文件，放在 `.memory/` 目录：

```
.memory/
  MEMORY.md          ← 索引文件（所有记忆的摘要）
  prefer_pnpm.md     ← 单条记忆
  ask_before_codegen.md
```

每个文件格式：
```markdown
---
name: prefer_pnpm
description: User prefers pnpm over npm
type: user
---
The user explicitly prefers pnpm for package management commands.
```

### 四类记忆

| 类型 | 用途 | 例子 |
|------|------|------|
| `user` | 稳定的用户偏好 | "喜欢 pnpm"、"要简洁回答" |
| `feedback` | 用户的纠正/要求 | "不要动 test snapshots" |
| `project` | 代码里看不出来的项目事实 | "legacy 目录不能删，部署依赖它" |
| `reference` | 外部资源指针 | incident board URL、监控面板 |

### 运作流程

```
新会话启动
    ↓
load_all() 扫描 .memory/*.md，解析 frontmatter
    ↓
load_memory_prompt() 把记忆格式化成文本块
    ↓
build_system_prompt() 将记忆注入 system prompt
    ↓
模型"看到"上一个会话保存的事实
    ↓
对话过程中触发 save_memory() → 写文件 → 重建 MEMORY.md 索引
```

关键点：**system prompt 在每次 LLM 调用前都重新构建**（`build_system_prompt()` 在 `agent_loop` 每轮都调用），所以本次会话新保存的记忆立刻在下一轮对话中可见。

### 什么不应该存

| 不存 | 原因 |
|------|------|
| 文件结构、函数签名 | 代码本身就是 source of truth，存了会过时 |
| 当前任务状态 | 属于 task/plan，任务完成后无意义 |
| 临时 branch 名、PR 号 | 很快过时 |
| 密钥/凭证 | 安全风险 |

### 一句话总结
> Memory 不是把 agent 见过的东西全存下来，而是一个**小型、精心筛选的持久化事实库** —— 只存那些跨会话仍然有价值、且无法从当前代码库廉价推导出来的信息。

## 分析: 本次会话更新记忆的时机是什么? 是否在本次会话中立即可见?

### 更新发生在何时

当模型调用 `save_memory` 工具时，`run_save_memory()` 立刻做两件事：

1. **写磁盘** —— 在 `.memory/` 下创建/覆盖对应的 `.md` 文件，并重建 `MEMORY.md` 索引
2. **更新内存中的 `self.memories` 字典** —— 不需要重新从磁盘读

### 是否在本次会话立即可见

**是**，但要等到下一轮 LLM 调用。

关键在 `agent_loop` 的循环结构：

```python
while True:
    system = build_system_prompt()   # ← 每轮都重新构建
    response = client.messages.create(...)
    ...
    # tool 执行（包括 save_memory）
    # → self.memories 已更新
    # 回到 while True 顶部，下一次调用就能看到新记忆
```

`build_system_prompt()` 调用 `memory_mgr.load_memory_prompt()`，后者直接读 `self.memories`（内存字典，已被 `save_memory` 更新），所以：

```
第 N 轮：模型调用 save_memory("prefer_pnpm", ...)
         └→ self.memories["prefer_pnpm"] 立刻存在

第 N+1 轮：build_system_prompt() 重新构建
           └→ system prompt 中包含 prefer_pnpm
           └→ 模型"看到"这条记忆
```

### 总结

| 场景 | 可见？ |
|------|--------|
| 同一轮（save_memory 调用后、同轮回复中） | 否 |
| 下一轮 LLM 调用（同会话内） | **是** |
| 下一次新会话启动 | **是**（从磁盘 `load_all()` 读取） |

## 分析: 模型如何决定是否更新memory, 更新哪种memory, 以及要如何更新?

### 决策依据：System Prompt 中的指导文本

模型没有"内置"判断逻辑 —— 完全靠 `MEMORY_GUIDANCE` 字符串注入到 system prompt 来引导：

```python
MEMORY_GUIDANCE = """
When to save memories:
- User states a preference ("I like tabs", "always use pytest") -> type: user
- User corrects you ("don't do X", "that was wrong because...") -> type: feedback
- You learn a project fact that is not easy to infer from current code alone -> type: project
- You learn where an external resource lives -> type: reference

When NOT to save:
- Anything easily derivable from code (function signatures, file structure...)
- Temporary task state (current branch, open PR numbers, current TODOs)
- Secrets or credentials
"""
```

模型读到这段文字，结合对话内容，自行判断"这句话值不值得存"。

### 决定存哪种类型：工具 schema 约束

`save_memory` 工具的 schema 把类型限制为枚举值，并为每种类型附上说明：

```python
"type": {
    "type": "string",
    "enum": ["user", "feedback", "project", "reference"],
    "description": "user=preferences, feedback=corrections, "
                   "project=non-obvious project conventions or decision reasons, "
                   "reference=external resource pointers"
}
```

模型根据 `description` 字段的文字描述匹配场景，选择对应的 `type`。

### 决定写什么内容：完全由模型自由生成

工具 schema 只规定了字段结构：

| 字段 | 约束 | 实际内容 |
|------|------|---------|
| `name` | 短标识符 | 模型自己命名，如 `prefer_pnpm` |
| `description` | 一行摘要 | 模型总结 |
| `type` | 枚举 4 选 1 | 按上面规则选 |
| `content` | 无限制 | 模型自由撰写，可多行 |

### 整个决策链

```
用户说："我以后都用 pnpm 不用 npm"
        ↓
模型对照 MEMORY_GUIDANCE：
  "User states a preference" → 触发保存
        ↓
对照 type description：
  "user=preferences" → type: user
        ↓
模型自主生成调用：
  save_memory(
    name="prefer_pnpm",
    description="User prefers pnpm over npm",
    type="user",
    content="Always use pnpm instead of npm for package management."
  )
```

### 本质

这套系统**没有规则引擎**，所有判断都是模型的"理解"行为。`MEMORY_GUIDANCE` 和工具 schema 的描述文字就是全部控制手段 —— 写得越清晰，模型的判断越准确。

## 分析: 上述问题, 有没有更好的解决方案?

当前方案有几个明显的设计缺陷，对应不同的改进思路：

---

### 缺陷 1：全量加载，不做检索

当前做法是把所有记忆一股脑塞进 system prompt。记忆多了会占用大量 context，且大部分与当前对话无关。

**改进：语义检索（RAG）**
```
用户输入 → 向量化 → 在记忆库中做相似度搜索 → 只加载 Top-K 条相关记忆
```
用 `chromadb` / `faiss` 存储记忆的 embedding，每轮只注入相关的几条。

---

### 缺陷 2：模型自己决定存什么，容易漏或存错

完全依赖模型判断，不稳定 —— 模型可能忽略重要偏好，也可能存了不该存的东西。

**改进 A：会话结束后的反思（Reflection Pass）**

对话结束时，单独跑一轮专门的"反思"调用：
```python
# 把整段对话历史传给模型，专门问：
"Review this conversation. What facts are worth remembering 
 for future sessions? Save them now."
```
比实时判断更全面，因为能看到完整上下文。

**改进 B：用户显式确认**
```
模型："我注意到你偏好 pnpm，是否要记住这个偏好？[y/n]"
```
把控制权还给用户，避免静默保存错误信息。

---

### 缺陷 3：没有过期/失效机制

记忆会无限积累，且没有机制标记"这条已经过时了"。

**改进：带时间戳 + 访问频率的衰减**
```markdown
---
name: prefer_pnpm
type: user
created: 2026-01-01
last_accessed: 2026-05-01
access_count: 12
---
```
长期未被引用的记忆降低优先级或自动归档。

---

### 缺陷 4：Dream Consolidator 几乎不会触发

7 个 gate 加上 24 小时冷却 + 最少 5 次会话，在实际使用中很难触发。

**改进：每次会话结束时轻量去重**
```python
# 不需要完整的 consolidate，只做：
# 1. 找 description 相似度 > 0.9 的记忆对
# 2. 合并它们
```

---

### 缺陷 5：记忆与 CLAUDE.md 职责模糊

当前方案没有清晰区分"这次存的是用户偏好"还是"这应该是项目级的规则"。

**改进：分层存储**

| 层 | 存储位置 | 内容 | 加载方式 |
|----|---------|------|---------|
| 项目规则 | `CLAUDE.md` | 编码规范、架构约束 | 始终全量加载 |
| 用户偏好 | `.memory/user/` | 个人习惯 | 始终加载 |
| 项目事实 | `.memory/project/` | 非显式约束 | 按相关性加载 |
| 外部引用 | `.memory/reference/` | URL、文档位置 | 按需检索 |

---

### 最务实的单一改进

如果只能改一处：**加 Reflection Pass**。成本低（一次额外 LLM 调用），收益最大 —— 把"实时判断"改为"事后复盘"，记忆质量显著提升，且不改变任何现有架构。