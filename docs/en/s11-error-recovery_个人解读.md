# s11-error-recovery 个人解读

## 分析: error recovery过程中, budget是如何控制的

从文档和脚本来看，budget 通过 **独立计数器 + 最大值常量** 来控制，每类错误有自己的预算：

---

### 1. `max_tokens` 续写预算

```python
MAX_RECOVERY_ATTEMPTS = 3         # 全局上限
max_output_recovery_count = 0     # 局部计数器，在 agent_loop() 内初始化
```

每次 `stop_reason == "max_tokens"` 时递增，超过上限则终止：

```python
if response.stop_reason == "max_tokens":
    max_output_recovery_count += 1
    if max_output_recovery_count <= MAX_RECOVERY_ATTEMPTS:
        # 注入 CONTINUATION_MESSAGE，继续
    else:
        # 耗尽，return
```

**关键细节**：只要有一次正常响应（非 `max_tokens`），计数器会被重置为 0，防止跨不同轮次的 max_tokens 错误被累计。

---

### 2. 连接/限流 backoff 预算

同样用 `MAX_RECOVERY_ATTEMPTS = 3`，通过 `for attempt in range(MAX_RECOVERY_ATTEMPTS + 1)` 控制：

```python
for attempt in range(MAX_RECOVERY_ATTEMPTS + 1):   # 0,1,2,3 共4次机会
    try:
        response = client.messages.create(...)
        break
    except APIError as e:
        if attempt < MAX_RECOVERY_ATTEMPTS:
            delay = backoff_delay(attempt)
            time.sleep(delay)
            continue
        # attempt == 3, 耗尽，return
```

延迟本身用指数 backoff + jitter 来避免 thundering herd：
$$delay = \min(1.0 \times 2^{attempt},\ 30.0) + \text{random}(0, 1)$$

---

### 3. `prompt_too_long` compact 预算

**没有独立计数器**——这是一个设计选择。compact 后 prompt 会真正缩小，理论上不会无限触发。它在同一个 `for attempt` 循环里用 `continue` 重试，受外层 `MAX_RECOVERY_ATTEMPTS` 隐式约束。

---

## 总结

| 错误类型 | 计数器 | 上限 | 重置时机 |
|---------|--------|------|---------|
| `max_tokens` | `max_output_recovery_count`（局部变量） | 3 | 每次正常响应后归零 |
| 连接/限流 | `attempt`（for 循环变量） | 3 | 每次 `agent_loop` 调用重新开始 |
| `prompt_too_long` | 无独立计数器 | 受 backoff 循环隐式约束 | — |

文档中提到的核心原则：**每类 failure 用独立计数器**，防止不同类型的错误相互"借用"预算，也让 recovery 的原因始终以结构化状态保留，而不是淹没在 catch 块里。

## 分析: recovery 的原因是如何保留和传递的?

这里有一个值得注意的**文档与实现之间的差距**。

---

### 文档描述的理想模式

文档提倡"classify first, act second"，用结构化对象显式保留原因：

```python
def choose_recovery(stop_reason, error_text) -> dict:
    if stop_reason == "max_tokens":
        return {"kind": "continue", "reason": "output truncated"}
    if ...:
        return {"kind": "compact", "reason": "context too large"}
    ...

decision = choose_recovery(...)
# decision["reason"] 始终可见，不会消失在 catch 块里
```

这样 `reason` 作为结构化状态存在，可以被日志、监控、上层逻辑读取。

---

### 实际脚本的实现

脚本**没有实现** `choose_recovery()`，recovery 原因**只活在 `print()` 输出里**：

```python
except APIError as e:
    if "overlong_prompt" in error_body or ...:
        print(f"[Recovery] Prompt too long. Compacting...")  # 原因在这里消失了
        messages[:] = auto_compact(messages)
        continue
    ...
    print(f"[Recovery] API error: {e}. Retrying in {delay:.1f}s ...")
```

```python
if response.stop_reason == "max_tokens":
    max_output_recovery_count += 1
    print(f"[Recovery] max_tokens hit ({max_output_recovery_count}/...)...")
```

**原因没有被结构化保存**——它被打印到 stdout 后就丢弃了，没有写入 `messages`、没有挂载到任何状态对象。

---

### 总结

| | 文档理想 | 实际脚本 |
|--|---------|---------|
| 原因载体 | `{"kind": "...", "reason": "..."}` dict | `print()` 字符串 |
| 可被程序读取？ | 是 | 否 |
| 传递方式 | 结构化 decision 对象向下传递 | 仅人类可读的控制台输出 |

这是一个**教学简化**：脚本的重点是演示三条恢复路径的控制流，而不是完整的可观测性设计。如果要在生产中使用，应该补上文档描述的 `choose_recovery()` 模式，把原因写入结构化日志或状态对象。