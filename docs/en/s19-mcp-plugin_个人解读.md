# s19-mcp-plugin 个人解读

## 分析: mcp plugin的原理

基于附件中的源码和文档，MCP Plugin 的核心原理如下：

### 架构分层

```
plugin manifest (.claude-plugin/plugin.json)
    ↓ 声明哪些 MCP server 需要启动
PluginLoader  →  发现并解析 manifest
    ↓
MCPClient     →  用 stdio 启动 server 进程，握手，拉取工具列表
    ↓
MCPToolRouter →  注册 client，统一路由 mcp__ 前缀的调用
    ↓
build_tool_pool() →  native tools + MCP tools 合并成一个 flat list 给 LLM
```

### 三个核心概念的区别

| 层级 | 是什么 | 作用 |
|------|--------|------|
| plugin manifest | JSON 配置文件 | **发现**：告诉 harness 要启动哪些 server |
| MCP server | 外部子进程 | **连接**：通过 stdio 暴露工具集 |
| MCP tool | 单个可调用能力 | **调用**：model 实际 invoke 的东西 |

### 关键机制

**1. stdio 通信（JSON-RPC）**
MCPClient 用 `subprocess.Popen` 启动 server，通过 stdin/stdout 发送 JSON-RPC 消息。握手流程：`initialize` → 收到 result → 发 `notifications/initialized`，之后才能 `tools/list` / `tools/call`。

**2. 命名前缀防冲突**
外部工具统一加 `mcp__{server}__{tool}` 前缀：
```
bash            ← native
mcp__postgres__query   ← MCP
```
前缀同时承担两个职责：防命名碰撞 + 路由标识（router 通过 split `__` 找到目标 server）。

**3. 统一工具池**
`build_tool_pool()` 将 native tools 和所有 MCP tools 合并，native 优先（防外部覆盖核心工具），LLM 看到的是一个扁平列表，不感知来源。

**4. 同一权限门**
这是安全边界的关键——MCP 工具**不绕过** `CapabilityPermissionGate`：
```python
decision = permission_gate.check(block.name, block.input or {})
# block.name 可能是 "bash" 也可能是 "mcp__postgres__query"
# 走同一套 risk 判断逻辑
```

**5. 运行时路由**
```python
if tool_name.startswith("mcp__"):
    return mcp_router.call(tool_name, arguments)  # 拆前缀 → 找 client → call_tool
else:
    return NATIVE_HANDLERS[tool_name](**tool_input)
```

### 本质

> MCP 解决的是**能力增长问题**：不改动 agent 核心代码，通过 plugin manifest 声明 + server 进程启动，就能让 agent 获得新工具。代价是引入了一个标准协议层（JSON-RPC over stdio），收益是 agent loop 完全不需要感知外部工具的来源。

## 分析: 关闭session, 是否有资源泄漏?

查看 `disconnect()` 和主程序清理逻辑，存在以下几个资源泄漏/风险点：

### 1. stderr pipe 从不被读取（最严重）
```python
self.process = subprocess.Popen(
    ...,
    stderr=subprocess.PIPE,  # ← 打开了管道但从未读取
)
```
如果 server 进程向 stderr 写入足够多的内容（填满 OS 管道缓冲区，通常 64KB），子进程会**阻塞在 write** 上，导致死锁。

### 2. `kill()` 后没有 `wait()`，产生僵尸进程
```python
def disconnect(self):
    if self.process:
        try:
            self._send({"method": "shutdown"})
            self.process.terminate()
            self.process.wait(timeout=5)   # ← 若 terminate() 失败会跳到 except
        except Exception:
            self.process.kill()            # ← kill 之后没有 wait()
        self.process = None
```
`kill()` 分支缺少 `self.process.wait()`，Unix 下该进程变为**僵尸进程**，直到 Python GC 回收 `Popen` 对象时才清理。

### 3. stdin/stdout 未显式关闭
`disconnect()` 结束后 `self.process = None`，依赖 GC 关闭 pipe fd，而不是显式调用 `process.stdin.close()` / `process.stdout.close()`。

### 4. 无 `try/finally`，agent_loop 抛异常时清理不执行
```python
# __main__
while True:
    ...
    agent_loop(history)   # ← 若此处抛出未捕获异常

# Cleanup MCP connections   ← 永远不会到达
for c in mcp_router.clients.values():
    c.disconnect()
```

### 5. `_recv()` 阻塞无超时
```python
line = self.process.stdout.readline()  # ← server 无响应时永久阻塞
```

## 建议修复

```python
def disconnect(self):
    if self.process:
        try:
            self._send({"method": "shutdown"})
            self.process.terminate()
            self.process.wait(timeout=5)
        except Exception:
            self.process.kill()
            self.process.wait()          # ← 防僵尸进程
        finally:
            self.process.stdin.close()   # ← 显式关闭
            self.process.stdout.close()
            self.process = None
```

```python
# __main__ 中加 try/finally
try:
    while True:
        ...
        agent_loop(history)
finally:
    for c in mcp_router.clients.values():
        c.disconnect()
```

```python
# stderr 改为丢弃，避免管道死锁
self.process = subprocess.Popen(
    ...,
    stderr=subprocess.DEVNULL,   # ← 或用线程异步消费
)
```

## 分析: plugin概念是什么? 是一种独立的机制吗?

Plugin **不是独立机制**，它只是 MCP 的**发现层（discovery layer）**，本质是一个配置声明。

> 个人补充, 最好不要叫plugin , 容易混淆概念. 在claude code生态中, plugin是为了方便分发的一个打包单元. 和本章节的plugin没有一毛钱关系.

```json
// .claude-plugin/plugin.json
{
  "name": "my-db-tools",
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"]
    }
  }
}
```

这个 JSON 文件就是完整的 plugin。它**不包含任何逻辑**，只声明：启动哪个进程、用什么命令。

## 三层关系

```
Plugin manifest  →  "我知道哪里有一个 MCP server"
      ↓
MCP server       →  "我是一个外部进程，能提供工具"
      ↓
MCP tool         →  "这是我暴露的具体工具"
```

`PluginLoader.scan()` 读 manifest → `get_mcp_servers()` 提取配置 → 主程序用这个配置创建 `MCPClient` 并 `connect()`。Plugin 的生命周期在 `connect()` 之后就结束了，后续全是 MCPClient 的工作。

## 类比

| 概念 | 类比 |
|------|------|
| Plugin manifest | package.json 里的 dependencies 声明 |
| MCP server | 实际安装的 npm 包进程 |
| MCP tool | 包里 export 的某个函数 |

Plugin 解决的唯一问题是：**agent 启动时怎么知道要连接哪些 MCP server**。没有 plugin 机制，你也可以手写 `MCPClient("postgres", "npx", [...])` 达到同样效果，只是硬编码了。