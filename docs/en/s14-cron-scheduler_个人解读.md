# s14-cron-scheduler 个人解读

## 分析: cron scheduler的原理是什么

核心思想：**将未来意图存为数据记录，用后台线程对照时钟触发，再把触发结果注入同一个 agent 主循环**——不引入第二套执行体系。

---

### 架构总览

```
用户/工具调用
  cron_create(cron, prompt)
        │
        ▼
   [tasks 列表]  ←── 持久化到 .claude/scheduled_tasks.json (durable 模式)
        │
        │  后台线程每秒轮询
        ▼
  _check_loop()
    └─ 每分钟执行一次 _check_tasks(now)
         └─ cron_matches(expr, now) == True ?
                │
                ▼
         notification_queue.put(prompt)
                │
                ▼
  agent_loop() 顶部 drain_notifications()
        │
        ▼
  注入为 {"role": "user", "content": "[Scheduled task xxx]: ..."} 
        │
        ▼
  LLM 照常处理，无需感知来源
```

---

### 关键机制逐层拆解

#### 1. Cron 表达式匹配（`cron_matches`）

纯手工解析，无外部依赖。5 个字段按顺序匹配 `[minute, hour, day, month, weekday]`。

每个字段支持：
| 语法 | 含义 |
|------|------|
| `*` | 任意值 |
| `*/N` | 每 N 步长 |
| `N-M` | 范围 |
| `N,M` | 枚举 |
| `N-M/S` | 范围内步长 |

注意 Python weekday（0=周一）到 cron dow（0=周日）的转换：`cron_dow = (dt.weekday() + 1) % 7`

#### 2. 防重复触发

```python
self._last_check_minute = -1  # 记录上次检查的分钟数

if current_minute != self._last_check_minute:
    self._last_check_minute = current_minute
    self._check_tasks(now)
```

后台线程每秒 `wait(1)`，但只在**分钟变化时**真正检查，避免同一分钟内多次触发。

#### 3. Jitter（抖动）防止整点扎堆

对恰好落在 `:00` 或 `:30` 的 recurring 任务，自动计算一个 1-4 分钟的偏移：

```python
if minute_val in JITTER_MINUTES:  # [0, 30]
    return (hash(cron_expr) % JITTER_OFFSET_MAX) + 1  # 1-4
```

匹配时把 `now` 前移 `jitter_offset` 分钟，相当于让该任务"提前感知"到触发时刻，实际执行时间仍在原点附近但不会精确卡整点。

#### 4. 持久化与会话生命周期

| 模式 | 存储 | 生命周期 |
|------|------|---------|
| `durable=False` | 内存 `tasks` 列表 | 进程退出即消失 |
| `durable=True` | `.claude/scheduled_tasks.json` | 跨会话保留 |

启动时 `_load_durable()` 只恢复 `durable=True` 的任务；`detect_missed_tasks()` 可检测进程关闭期间本应触发但错过的任务（逐分钟回溯最多 24h）。

#### 5. CronLock 防多进程竞争

```python
# 写 PID 到 .claude/cron.lock
self._lock_path.write_text(str(os.getpid()))

# 检测：若 PID 存活则锁有效，否则为 stale lock 可强占
os.kill(stored_pid, 0)  # signal 0 = 存活探针
```

防止同一项目开两个终端时两个调度器都触发同一任务。

#### 6. 触发通知回流主循环

```python
def agent_loop(messages):
    while True:
        # ① 先排空通知队列
        for note in scheduler.drain_notifications():
            messages.append({"role": "user", "content": note})
        
        # ② 再调用 LLM（注入的 note 与用户消息完全一样）
        response = client.messages.create(...)
```

LLM 不区分消息来源——调度触发的 prompt 和用户手打的消息走完全相同的处理路径。

---

### 任务生命周期状态机

```
创建 ──► 等待匹配
              │
       cron_matches == True
              │
              ▼
         enqueue 通知
              │
        ┌─────┴─────┐
        │           │
   recurring    one-shot
        │           │
    继续等待     auto-delete
        │           │
   >7天自动      fired_oneshots
    auto-expire     移除
```

---

### 一句话总结

> **Cron Scheduler = 任务记录（数据）+ 后台时钟检查（触发）+ 通知队列（回流）**，三者解耦，调度逻辑不侵入 agent 主循环。

## 分析: fire了没完成session异常退出怎么办? 还能再次执行吗?

结合源码分析这个问题：

### 关键：`last_fired` 何时落盘？

```python
# _check_tasks 中
self.queue.put(notification)        # ① 通知入队（内存）
task["last_fired"] = time.time()    # ② 标记已触发（内存）

# _save_durable() 只在有 expired 或 oneshot 清理时才调用
if expired or fired_oneshots:
    ...
    self._save_durable()            # ③ 持久化（只在清理时触发）
```

### 两种场景

| 任务类型 | 触发后 session 崩溃 | 重启后状态 |
|---------|-------------------|-----------|
| **one-shot durable** | 被加入 `fired_oneshots` → `_save_durable()` 删除该任务 | 任务从磁盘消失，**永久丢失** |
| **recurring durable** | `last_fired` 仅在内存，未落盘 | 重启后磁盘里 `last_fired` 是旧值或 `null` |

### `detect_missed_tasks()` 能救回来吗？

```python
def detect_missed_tasks(self) -> list[dict]:
    # 从磁盘的 last_fired 往后逐分钟扫描，找 cron 匹配点
    check = last_dt + timedelta(minutes=1)
    while check <= cap:
        if cron_matches(task["cron"], check):
            missed.append(...)   # 检测到遗漏
            break
```

- **recurring 任务**：因为 `last_fired` 没落盘，重启时磁盘值是"上上次"触发时间，`detect_missed_tasks()` 会检测到这个窗口内有遗漏 → **可以补救**
- **one-shot 任务**：任务已从磁盘删除，无法检测 → **无法补救**

### 但还有一个坑

`detect_missed_tasks()` 在当前代码中**从未被自动调用**：

```python
def start(self):
    self._load_durable()   # 只加载任务
    self._thread = ...     # 启动后台线程
    # detect_missed_tasks() 没有在这里调用！
```

它是一个工具方法，需要用户自己在启动时主动调用并处理结果。

### 总结

| 问题 | 答案 |
|------|------|
| recurring durable 任务触发但未完成就崩溃 | 下次启动手动调用 `detect_missed_tasks()` 可找回，但需自己补充"重新执行"逻辑 |
| one-shot durable 任务触发后崩溃 | **彻底丢失**，任务已从磁盘删除 |
| session-only 任务任何情况 | 进程死即消失，不可恢复 |

> **这是教学代码的有意取舍**：通知队列是纯内存的，没有 WAL（Write-Ahead Log）或 ACK 机制。生产级系统需要在"任务完成"时才更新 `last_fired` 并落盘，而非在"通知入队"时。

## 分析: scheduled task怎么和当前session的user task区分开? 不会造成LLM理解困难吗?

### 当前代码的区分方式

只靠**消息内容的前缀文本**，没有任何结构性区分：

```python
# 后台线程：构造通知
notification = f"[Scheduled task {task['id']}]: {task['prompt']}"

# agent_loop：直接塞进 messages，role 仍是 "user"
messages.append({"role": "user", "content": note})
```

LLM 收到的消息历史是这样的：

```
user:      "Schedule 'run tests' every 5 minutes"
assistant: "已创建任务 abc12345，cron=*/5 * * * *"
user:      "帮我看一下昨天的日志"          ← 真实用户输入
assistant: "..."
user:      "[Scheduled task abc12345]: run tests"  ← 调度注入，role 完全一样
assistant: "..."
```

---

### 会造成 LLM 理解困难吗？

**轻度困难，但通常可以应对**，原因：

1. `[Scheduled task xxx]:` 前缀是语义明确的自然语言标记，LLM 能读懂"这是调度任务"
2. SYSTEM prompt 里已说明：`"Tasks fire automatically and their prompts are injected into the conversation"` ，给 LLM 提供了上下文
3. 调度 prompt 一般是独立指令（"run tests"），LLM 不太会把它和上一条用户消息混淆

**但有两个真实风险：**

| 风险 | 场景 |
|------|------|
| **上下文污染** | 用户正在进行多轮对话（"上面那个文件…"），调度通知突然插入，LLM 的 "上面那个" 指代可能错乱 |
| **角色混淆** | 如果调度 prompt 本身包含 "你之前说…" 这类回指，LLM 可能把调度任务当成用户在追问 |

---

### 生产级方案应该怎么做

更健壮的做法是用 **独立 role 或 metadata** 来结构化区分，而非依赖文本前缀：

```python
# 方案1：专用 system 消息插入（在调度触发前注入）
messages.append({
    "role": "user", 
    "content": [
        {
            "type": "text",
            "text": "<scheduled_task id='abc12345'>run tests</scheduled_task>"
        }
    ]
})

# 方案2：在每次调度注入前，先插一条 system 上下文重置
messages.append({"role": "user", "content": "---"})  # 视觉分隔
messages.append({"role": "user", "content": f"[SYSTEM TRIGGER] {note}"})
```

或者更彻底地：**调度触发时开一个新的独立对话历史**，不复用当前 session 的 messages，避免上下文污染。

---

### 结论

> 教学代码用文本前缀区分是够用的简化设计，LLM 一般能理解。真正的问题不是"LLM 能否区分"，而是**调度消息和用户对话交织时的上下文污染**——这才是生产系统需要隔离处理的根本原因。