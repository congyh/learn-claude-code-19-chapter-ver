# s08-hook-system 个人解读

## 分析: Hook System原理是什么?

### 核心设计思想

Hook 系统解决的是**扩展性问题**：如何在不修改主循环代码的前提下，向 Agent 注入新行为。

```
团队A: 安全扫描每条 bash 命令
团队B: 文件写入后自动跑 lint
团队C: 记录所有工具调用审计日志
```

传统方案需要在主循环里加 `if/else`，最终变成一团乱麻。Hook 系统将扩展点**固化在主循环外**。

---

### 三个生命周期事件

| 事件 | 触发时机 | 能否阻断 |
|------|---------|---------|
| `SessionStart` | Agent 启动时触发一次 | 否 |
| `PreToolUse` | 每次工具调用**前** | 是 (exit 1) |
| `PostToolUse` | 每次工具调用**后** | 否 |

---

### 退出码协议（最核心的设计）

Hook 本质上是一个**子进程**，通过退出码与主循环通信：

```
exit 0 → 静默继续
exit 1 → 阻断工具执行（stderr 作为错误原因返回）
exit 2 → 注入消息（stderr 注入对话上下文，工具仍执行）
```

这个协议极简但强大：任何语言（bash/Python/Ruby）只要能控制退出码，就能参与扩展。

---

### 数据流

```
LLM 返回 tool_use
        │
        ▼
  [PreToolUse hooks]
  通过环境变量传入上下文:
    HOOK_EVENT=PreToolUse
    HOOK_TOOL_NAME=bash
    HOOK_TOOL_INPUT={"command": "ls"}
        │
        ├─ exit 1 → blocked=True → 跳过执行，返回 block_reason 给 LLM
        ├─ exit 2 → messages.append(stderr) → 注入，继续执行
        └─ exit 0 → 继续
        │
        ▼
  [执行工具] handler(**tool_input)
        │
        ▼
  [PostToolUse hooks]
  额外传入 HOOK_TOOL_OUTPUT
        │
        └─ exit 2 → output += f"\n[Hook note]: {msg}"
        │
        ▼
  返回 tool_result 给 LLM
```

---

### 关键实现细节

**1. 匹配器过滤**（避免每个 hook 都触发）

```python
matcher = hook_def.get("matcher")
if matcher and context:
    tool_name = context.get("tool_name", "")
    if matcher != "*" and matcher != tool_name:
        continue  # 跳过不匹配的 hook
```

**2. 工作区信任机制**（安全门）

```python
TRUST_MARKER = WORKDIR / ".claude" / ".claude_trusted"

def _check_workspace_trust(self) -> bool:
    if self._sdk_mode:
        return True          # SDK 模式隐式信任
    return TRUST_MARKER.exists()  # 必须有信任标记才运行 hooks
```

防止在不受信任的目录中执行任意 shell 命令。

**3. 结构化 stdout 扩展点**（exit 0 时的隐式协议）

```python
# 如果 hook stdout 是合法 JSON，可以做更多事：
hook_output = json.loads(r.stdout)
if "updatedInput" in hook_output:
    context["tool_input"] = hook_output["updatedInput"]  # 修改工具输入
if "additionalContext" in hook_output:
    result["messages"].append(...)                        # 追加上下文
if "permissionDecision" in hook_output:
    result["permission_override"] = ...                  # 覆盖权限决策
```

---

### 主循环与 Hook 的职责边界

| 角色 | 职责 |
|------|------|
| 主循环 | 控制流的唯一所有者（决定"接下来做什么"） |
| 工具处理器 | 执行的唯一所有者（决定"怎么做"） |
| Hook | **只能**观察、阻断、或注释——不拥有控制流 |

这是整个设计的核心约束：Hook 是旁路（sidecar），不是主路。

---

### 配置驱动，零代码变更

```json
{
  "hooks": {
    "PreToolUse": [
      {"matcher": "bash", "command": "/team-security/scan.sh"},
      {"matcher": "write_file", "command": "/team-qa/lint.sh"}
    ],
    "PostToolUse": [
      {"command": "/team-ops/audit-log.sh"}
    ]
  }
}
```

不同团队通过修改 `.hooks.json` 注入自己的逻辑，主循环代码从不改动。

## 分析: Hook执行的上下文是什么? 谁负责传递, 有什么内容

**上下文由主循环构建，通过环境变量注入到 Hook 子进程。**

### 谁负责传递

`agent_loop` 在调用 `hooks.run_hooks()` 前，手动构建 `ctx` 字典：

```python
# agent_loop 中
tool_input = dict(block.input or {})
ctx = {"tool_name": block.name, "tool_input": tool_input}  # 主循环构建

pre_result = hooks.run_hooks("PreToolUse", ctx)
# ...执行工具...
ctx["tool_output"] = output  # PostToolUse 时追加输出
post_result = hooks.run_hooks("PostToolUse", ctx)
```

### 如何传递

`HookManager.run_hooks()` 将 `ctx` 写入**子进程的环境变量**：

```python
env = dict(os.environ)   # 继承当前进程所有环境变量
if context:
    env["HOOK_EVENT"]      = event                          # 事件名
    env["HOOK_TOOL_NAME"]  = context.get("tool_name", "")  # 工具名
    env["HOOK_TOOL_INPUT"] = json.dumps(context.get("tool_input", {}))[:10000]  # 工具入参 (JSON)
    if "tool_output" in context:
        env["HOOK_TOOL_OUTPUT"] = str(context["tool_output"])[:10000]  # 仅 PostToolUse 有

r = subprocess.run(command, shell=True, env=env, ...)
```

### 上下文内容汇总

| 环境变量 | 内容 | 哪个事件有 |
|---------|------|-----------|
| `HOOK_EVENT` | `"PreToolUse"` / `"PostToolUse"` / `"SessionStart"` | 所有 |
| `HOOK_TOOL_NAME` | 工具名，如 `"bash"` / `"write_file"` | 所有 |
| `HOOK_TOOL_INPUT` | 工具入参的 JSON 字符串，截断至 10000 字符 | 所有 |
| `HOOK_TOOL_OUTPUT` | 工具执行结果字符串，截断至 10000 字符 | **仅 PostToolUse** |

### Hook 脚本如何使用

```bash
#!/bin/bash
# 一个审计 bash 命令的 hook

echo "[AUDIT] tool=$HOOK_TOOL_NAME event=$HOOK_EVENT" >> /var/log/agent-audit.log
echo "[AUDIT] input=$HOOK_TOOL_INPUT" >> /var/log/agent-audit.log

# 如果是危险命令，阻断执行
if echo "$HOOK_TOOL_INPUT" | grep -q "rm -rf"; then
    echo "危险命令被拦截" >&2
    exit 1   # 阻断
fi

exit 0  # 正常通过
```

值得注意的是：上下文是**单向传递**的（主循环 → Hook），Hook 无法直接修改上下文——除非通过 exit 0 + 结构化 JSON stdout 的方式返回 `updatedInput`，那是一个隐式的反向通道。

## 分析: hook context是通过环境变量传递的, 这是并发安全的吗?

**在当前实现中是安全的**

### 为什么现在安全

**1. 每次 hook 调用都创建全新的 env 字典**

```python
env = dict(os.environ)   # 每次调用都是一份新拷贝
env["HOOK_TOOL_NAME"] = ...
env["HOOK_TOOL_INPUT"] = ...

subprocess.run(command, env=env, ...)  # 子进程再拷贝一份
```

从不修改 `os.environ` 本身，只读取它。读操作并发安全。

**2. `subprocess.run()` 是阻塞调用**

子进程启动时，OS 会 `fork()` + `execve()`，env 被**深拷贝**进新进程的地址空间。两个子进程之间的环境变量完全隔离，互不干扰。

**3. 当前主循环是单线程串行的**

```python
for block in response.content:       # 串行遍历每个工具调用
    pre_result = hooks.run_hooks(...)  # 阻塞等待所有 hook 完成
    output = handler(...)
    post_result = hooks.run_hooks(...)
```

同一时刻只有一个工具调用在执行，不存在并发竞争。

---

### 其他方案

不用环境变量，改用 **stdin 传 JSON**：

```python
# Hook 通过 stdin 接收上下文，完全线程隔离
r = subprocess.run(
    command, shell=True,
    input=json.dumps(context),   # 每个子进程独享自己的 stdin pipe
    capture_output=True, text=True,
)
```

每个子进程的 stdin 是独立的 pipe，不存在全局共享状态，天然并发安全。这也是生产级 hook 系统（如 Git hooks 的某些实现）的常见做法。