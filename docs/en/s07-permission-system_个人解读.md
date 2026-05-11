# s07-permission-system 个人解读

## 分析: permission system原理

### 核心思想：安全是管道，不是布尔值

> "Safety is a pipeline, not a boolean -- deny first, then consider mode, then check allow rules, then ask the user."

LLM 提出的工具调用意图（intent）和系统实际执行（execution）之间必须有一道门。这道门不是简单的 yes/no，而是一条有序的决策管道。

---

### 四阶段管道（First-Match-Wins）

```
LLM tool_call
     │
     ▼
[0] Bash 安全预校验   ← 正则匹配危险模式（sudo, rm -rf, 命令替换…）
     │
     ▼
[1] Deny Rules       ← 黑名单，bypass-immune，永远最先生效
     │
     ▼
[2] Mode Check       ← 三种模式决定"未命中规则时"的默认行为
     │
     ▼
[3] Allow Rules      ← 白名单，明确放行
     │
     ▼
[4] Ask User         ← 兜底：交互审批，支持"always"永久放行
     │
     ▼
execute / reject
```

**关键设计原则：Deny 规则不可绕过。** 无论当前是什么 mode，无论有没有 allow 规则，deny 永远在最前面跑。这保证了黑名单的绝对权威性。

---

### 三种权限模式（Mode）

| Mode | 未命中规则时的行为 | 典型用途 |
|------|----------------|---------|
| `default` | 全部 Ask 用户 | 日常交互，最安全 |
| `plan` | 写操作直接 Deny，读操作 Allow | 只想让 agent 探索、不动文件 |
| `auto` | 只读工具自动 Allow，写工具继续往下走 | 快速探索，减少打扰 |

Mode 解决的问题：同一套规则在不同场景下需要不同的"默认策略"，而不是每个工具都单独写规则。

---

### 规则匹配（Pattern Matching）

规则是有序列表，`_matches()` 支持三个维度的匹配：

```python
# tool 名称匹配
# path glob 匹配（用 fnmatch）
# content glob 匹配（bash 命令内容）
```

运行时用户回答 `always` 时，直接往规则列表末尾追加一条 allow rule，实现**运行时动态扩展白名单**，无需重启。

---

### 两层安全防御的分工

| 层级 | 实现 | 职责 |
|------|------|------|
| `BashSecurityValidator` | 正则 + 预定义 patterns | 静态分析命令字符串，拦截明确危险模式 |
| `PermissionManager` | 四阶段管道 | 运行时决策，支持 mode/rules/交互 |

两者分离的好处：validator 是无状态的纯函数，可独立测试；permission manager 持有状态（mode、rules、denial count）。

---

### 熔断器（Circuit Breaker）

```python
self.consecutive_denials = 0
self.max_consecutive_denials = 3
```

连续 3 次 Deny 后提示用户切换到 `plan` mode。这防止 agent 反复请求同一被拒绝的操作，造成无效循环消耗 token。

---

### 设计总结

| 关注点 | 设计决策 |
|-------|---------|
| **优先级** | Deny > Mode > Allow > Ask，顺序硬编码，不可配置 |
| **可扩展性** | 规则是普通列表，运行时可 append |
| **最小权限** | `safe_path()` 限制所有文件操作在 WORKDIR 内，路径穿越直接抛错 |
| **可观测性** | 每次决策都打印 `[DENIED]` / `[USER DENIED]` / `> tool_name: output` |
| **教学友好** | 刻意保持管道短小，每个 Step 单独注释，先教三种 mode 再讲高级 policy |

## 分析: 运行中动态更新rules是如何实现的?

关键在 `ask_user()` 方法的这段代码：

```python
def ask_user(self, tool_name: str, tool_input: dict) -> bool:
    ...
    answer = input("  Allow? (y/n/always): ").strip().lower()

    if answer == "always":
        # Add permanent allow rule for this tool
        self.rules.append({"tool": tool_name, "path": "*", "behavior": "allow"})
        self.consecutive_denials = 0
        return True
```

**机制很简单：`self.rules` 是一个普通 Python list，直接 `append` 即可。**

整个动态更新链路如下：

1. 管道走到第 4 阶段（Ask），调用 `ask_user()`
2. 用户输入 `always`
3. 往 `self.rules` 末尾追加一条 `{"tool": tool_name, "path": "*", "behavior": "allow"}`
4. 下次同名工具被调用时，`check()` 的 Step 3（Allow Rules 循环）会遍历到这条新规则并命中，直接返回 allow，不再到达 Step 4

**两个重要的设计细节：**

- **`path: "*"`** — 用 glob 通配符，对该工具的所有路径参数都放行，而不是只放行本次调用的具体路径
- **追加到末尾而非插入头部** — deny 规则在 Step 1 单独循环，allow 规则在 Step 3 单独循环，两者互不干扰。追加到末尾对 deny 规则没有影响，不会被覆盖

**限制：** 这条规则只存在于当前 `PermissionManager` 实例的内存中，进程退出即丢失。如需持久化，需要自行写入配置文件并在启动时加载。

## 分析: 当前权限系统设计存在哪些问题, 是否有更好的设计方案或开源解决方案?

### 当前权限系统的问题分析

#### 问题一：`always` 规则过于粗粒度

```python
# 用户只是允许了"写某个特定文件"，却变成了允许写任意文件
self.rules.append({"tool": tool_name, "path": "*", "behavior": "allow"})
```

用户回答 `always` 时，意图可能是"这次的 `write_file` 操作可以"，但实际效果是永久放行该工具的所有调用。正确做法应该是把当次调用的具体参数（path、command）固化进规则。

---

#### 问题二：Bash 校验规则可被绕过

```python
("rm_rf", r"\brm\s+(-[a-zA-Z]*)?r"),   # 无法匹配 rm --recursive
("sudo", r"\bsudo\b"),                   # 无法匹配 /usr/bin/sudo
```

且 Deny Rules 里的 content 用 `fnmatch` 做 glob 匹配：

```python
# "rm -rf /" 能匹配
# "rm -rf /home/user" 不能匹配 —— 漏掉了！
if not fnmatch(command, rule["content"]):
```

`fnmatch("rm -rf /home", "rm -rf /")` → `False`，deny rule 失效。

---

#### 问题三：`is_workspace_trusted()` 定义了但从未被调用

```python
def is_workspace_trusted(workspace: Path = None) -> bool:
    trust_marker = ws / ".claude" / ".claude_trusted"
    return trust_marker.exists()
```

这个函数完全游离在权限管道之外，是死代码。

---

#### 问题四：规则无持久化、无审计日志

- `always` 规则进程退出即丢失
- 没有任何 allow/deny 决策的日志记录，事后无法审计

---

#### 问题五：Bash 以完整用户权限运行

```python
subprocess.run(command, shell=True, ...)  # shell=True 本身就是高风险
```

没有任何沙箱隔离，通过校验的命令拥有进程的全部权限。

---

#### 问题六：模式降级无保护

`/mode plan` → `/mode auto` 是安全降级，但没有任何确认机制。

---

### 更好的设计方案

#### 方案一：修复 `always` 规则粒度

```python
if answer == "always":
    # 把当次调用的具体参数固化进规则，而不是通配
    rule = {"tool": tool_name, "behavior": "allow"}
    path = tool_input.get("path")
    if path:
        rule["path"] = path  # 只放行这个具体路径
    self.rules.append(rule)
```

#### 方案二：用正则替换 fnmatch 做 Bash 内容匹配

```python
import re
# deny rules 的 content 字段改为 regex
{"tool": "bash", "content": r"rm\s+(-\S+\s+)*(-r|-R|--recursive)", "behavior": "deny"}
```

#### 方案三：引入审计日志

```python
import logging
logger = logging.getLogger("permissions")

def check(self, tool_name, tool_input):
    decision = self._do_check(tool_name, tool_input)
    logger.info({"tool": tool_name, "input": tool_input, 
                 "decision": decision["behavior"], "reason": decision["reason"]})
    return decision
```

#### 方案四：规则持久化到 YAML

```yaml
# .permissions.yaml
mode: default
rules:
  - tool: bash
    content: "sudo *"
    behavior: deny
  - tool: read_file
    path: "*"
    behavior: allow
```

启动时加载，`always` 时同时写回文件。

---

### 开源解决方案对比

| 方案 | 适用场景 | 核心能力 |
|------|---------|---------|
| **[Casbin](https://casbin.org/)** | 需要 RBAC/ABAC 模型 | 支持 ACL、RBAC、ABAC，规则持久化，Python 有 `pycasbin` |
| **[OPA (Open Policy Agent)](https://www.openpolicyagent.org/)** | 策略复杂、需要独立策略服务 | Rego 语言表达策略，REST API，与语言无关 |
| **[Firejail](https://firejail.wordpress.com/)** | 需要真正沙箱隔离 bash | Linux namespaces 沙箱，限制文件系统/网络访问 |
| **[RestrictedPython](https://restrictedpython.readthedocs.io/)** | 沙箱化 Python 执行 | 编译期检查，禁止危险内置函数 |

**对于教学场景**，Casbin 最值得引入：`pycasbin` 可以直接替换当前的 `rules` list + `_matches()` 逻辑，支持规则持久化到文件，且 RBAC/ABAC 模型能表达"agent 角色只能读，admin 角色可以写"这类更细粒度的策略。

## 问题: fnmatch是什么? 和正则匹配区别是什么?

详见`Python_正则_regex_vs_fnmatch`一文

## 问题: 为什么前面提到shell=True是高风险, 能解释下吗? 推荐做法是什么?

详见`Python_subprocess_系统调用_shell调用`一文