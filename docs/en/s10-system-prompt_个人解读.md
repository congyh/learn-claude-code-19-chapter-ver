# s10-system-prompt 个人解读

## 分析: system prompt原理是什么?

### 核心思想：流水线，不是字符串

System prompt 不是一个硬编码的大字符串，而是**按固定顺序组装的流水线**：

```
core identity
+ tool catalog
+ skills
+ memory
+ CLAUDE.md chain
+ dynamic context
= final model input
```

源码中 `SystemPromptBuilder.build()` 体现了这点——每个 `_build_*` 方法只负责一个来源：

```python
def build(self) -> str:
    sections = []
    sections.append(self._build_core())        # 角色定义
    sections.append(self._build_tool_listing()) # 工具列表
    sections.append(self._build_skill_listing())# 技能元数据
    sections.append(self._build_memory_section())# 记忆注入
    sections.append(self._build_claude_md())    # 指令文件链
    sections.append(DYNAMIC_BOUNDARY)           # ← 关键分隔线
    sections.append(self._build_dynamic_context())# 动态上下文
```

---

### 关键边界：稳定内容 vs 动态内容

源码用 `DYNAMIC_BOUNDARY = "=== DYNAMIC_BOUNDARY ==="` 做分隔：

| 稳定（静态前缀，可跨 turn 缓存） | 动态（每 turn 重新生成） |
|---|---|
| 角色描述、工具契约、安全规则 | 当前日期、工作目录 |
| 技能列表、长期记忆 | 临时 mode、per-turn 警告 |
| CLAUDE.md 指令链 | 即时上下文 |

稳定内容不变，不需要模型每轮重读数千 token；动态内容每轮更新但体积小。

---

### CLAUDE.md 分层叠加（不覆盖）

`_build_claude_md()` 按优先级依次叠加三层，每层**追加**而非覆盖：

1. `~/.claude/CLAUDE.md` — 用户全局指令
2. `<project>/CLAUDE.md` — 项目级指令  
3. `<subdir>/CLAUDE.md` — 子目录指令

---

### 记忆的双向循环

记忆不是单向保存。**保存只完成了一半**：

```
s09 写入 .memory/*.md  →  s10 通过 _build_memory_section() 重新注入 prompt
```

没有注入这一步，记忆对模型毫无影响。

---

### Per-turn 提醒：独立于 prompt 之外

极短生命周期的内容（单轮有效）通过 `build_system_reminder()` 以独立 user 消息注入，而不是混入稳定的 system prompt：

```python
{"role": "user", "content": "<system-reminder>...</system-reminder>"}
```

---

### System Prompt 只是整体输入的一部分

根据 s10a，模型的完整输入是**三个并行 surface 的汇合**：

```
system prompt (稳定规则)
  + messages  (对话历史 + tool_result)
  + tools     (工具 schema)
      ↓ normalize ↓
    final API payload
```

System prompt 适合放：身份、规则、工具描述。  
**不**适合放：`tool_result`、单轮 hook、动态记忆附件——这些属于 message 流。

## 问题: s10a中的messages, tools是不是就是s10中system prompt中的`_build_dynamic_context`部分?

不是，这是两个不同层次的概念。

### `_build_dynamic_context` 是什么

它仍然是 **system prompt 内部**的一个 section，内容是运行时元数据：

```python
def _build_dynamic_context(self) -> str:
    lines = [
        f"Current date: {datetime.date.today().isoformat()}",
        f"Working directory: {self.workdir}",
        f"Model: {MODEL}",
        f"Platform: {os.uname().sysname}",
    ]
```

它的"动态"只是指"每轮重新生成"，本质上还是写进 system prompt 字符串里的文字。

---

### s10a 的 `messages` 和 `tools` 是什么

它们是与 system prompt **平行的 API 参数**，完全在 system prompt 之外：

```python
payload = {
    "system":   build_system_prompt(),  # ← system prompt，含 dynamic_context
    "messages": build_messages(...),    # ← 对话历史、tool_result、reminders
    "tools":    build_tools(...),       # ← 工具 schema，传给 API 做 function calling
}
```

---

### 对应关系

| s10a 概念 | s10 对应位置 | 说明 |
|---|---|---|
| `system prompt` | `prompt_builder.build()` 的全部输出 | 包含 dynamic_context |
| `tools` | `TOOLS` 列表直接传给 `client.messages.create(tools=TOOLS)` | 独立参数，不是 prompt 文字 |
| `messages` | `history` 列表，含用户消息、assistant 回复、`tool_result` | 独立参数 |
| `system reminder` | `build_system_reminder()` 注入为 user 消息 | 属于 messages，不是 system prompt |

**结论**：`_build_dynamic_context` 是 system prompt 字符串里的一小节；`messages` 和 `tools` 是完全独立的 API 输入面，三者在最终 `client.messages.create()` 调用时才汇合。