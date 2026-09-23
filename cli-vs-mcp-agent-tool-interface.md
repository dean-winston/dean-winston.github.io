---
layout: article
title: "CLI 与 MCP：Agent 工具接口应该如何选择"
description: "从组合能力、数据规模、业务语义和操作风险出发，讨论 Agent 工具接口如何在 CLI 与 MCP 之间选择。"
category: Agent Engineering
weight: 4
---
# CLI 与 MCP：Agent 工具接口应该如何选择

## 1. 一个容易问错的问题

在构建 Agent 工具系统时，经常会遇到一个问题：

> 一个能力应该做成 CLI，还是做成 MCP？

例如，我们有一个游戏服务端运维系统，希望 Agent 能完成：

* 查询线上日志；
* 查询服务器运行状态；
* 查询玩家状态；
* 搜索代码；
* 查询数据库；
* 执行 Lua；
* 重启服务器；
* 修改线上配置。

最直观的设计有两种。

一种是 CLI：

```bash
gameops logs --server GS17 --last 10m
gameops player 82931
gameops status GS17
gameops lua --server GS17 --script fix.lua
```

另一种是 MCP：

```text
query_logs(...)
query_player(...)
get_server_status(...)
execute_lua(...)
```

于是问题似乎变成了：

> CLI 和 MCP 谁更适合 Agent？

但严格来说，这是一个**层次并不完全对等的问题**。

CLI 是一种**程序交互界面**。

MCP 是一种**Agent 与外部能力之间的标准通信协议**。

甚至 MCP Server 本身就可以通过 `stdio` 启动：Host 创建一个子进程，然后通过它的 stdin/stdout 交换 JSON-RPC 消息。因此一个程序完全可能：

```text
既是一个命令行程序
又是一个 MCP Server
```

MCP 官方当前支持本地 `stdio` 和远程 Streamable HTTP；旧的 HTTP+SSE 已进入弃用阶段。

所以更准确的问题应该是：

> **这个能力应该暴露成 Shell-native 的命令接口，还是暴露成 Agent-native 的结构化 Tool 接口？**

这才是 CLI 与 MCP 选型的本质。

---

# 2. CLI 的本质：把能力交给“计算机环境”

CLI 的基本模型非常简单：

```text
Caller
  │
  ▼
Process
  │
  ├── argv
  ├── stdin
  │
  ├── stdout
  ├── stderr
  └── exit code
```

例如：

```bash
gameops logs --server GS17 --last 10m
```

它实际上就是：

```text
启动 gameops 进程

argv:
[
  "logs",
  "--server",
  "GS17",
  "--last",
  "10m"
]
```

执行结束后：

```text
stdout → 数据
stderr → 错误 / 日志
exit code → 是否成功
```

CLI 最大的特点不是“简单”，而是：

> **它天然进入了 Unix/Shell 的计算环境。**

因此它可以立即和几十年来形成的工具生态组合。

例如：

```bash
gameops logs --server GS17 --last 30m |
rg "player=82931" |
grep ERROR |
tail -100
```

或者：

```bash
gameops logs --server GS17 --last 30m --json |
jq '.events[] | select(.player_id == 82931)'
```

甚至：

```bash
gameops logs --server GS17 --last 1h > /tmp/log.json

python analyze.py /tmp/log.json
```

Agent 不需要你提前设计：

```text
filter_log
sort_log
count_log
group_log
find_error_log
filter_player_log
```

这些能力已经存在于：

```text
grep
rg
awk
sed
jq
sort
uniq
python
```

之中。

这也是 CLI 对 Coding Agent 特别有吸引力的原因。

---

# 3. MCP 的本质：把能力描述给 Agent

MCP 的设计目标不同。

它不是：

> “这里有一台计算机，你自己想办法。”

而更接近：

> “这里有一组明确描述过的能力，你可以调用。”

MCP 当前使用 JSON-RPC 2.0 作为基础消息协议，其架构里主要存在三个角色：

```text
Host
 │
 │ 例如：
 │ Claude Code
 │ IDE
 │ ChatGPT 类应用
 │ 自己开发的 Agent
 │
 ▼
MCP Client
 │
 ▼
MCP Server
 │
 ├── Tools
 ├── Resources
 └── Prompts
```

官方规范将 Host、Client、Server 明确区分，并把 Resources、Prompts、Tools 作为服务器侧核心能力。

其中最重要的是：

```text
Tool
```

一个 MCP Tool 不只是一个函数名。

它通常包含：

```text
name
title
description
inputSchema
outputSchema
annotations
```

例如：

```json
{
  "name": "query_logs",
  "title": "Query game server logs",
  "description": "Query logs for a specific game server.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "server_id": {
        "type": "string"
      },
      "start_time": {
        "type": "string"
      },
      "end_time": {
        "type": "string"
      },
      "player_id": {
        "type": "integer"
      }
    },
    "required": [
      "server_id",
      "start_time",
      "end_time"
    ]
  }
}
```

这意味着 Agent 不需要先运行：

```bash
gameops --help
```

它可以直接知道：

> 我有一个 `query_logs` 能力。

并且知道：

> server_id 是字符串。

> player_id 是整数。

> start_time 和 end_time 是必填参数。

MCP Tool 的 `inputSchema` 使用 JSON Schema；官方 SDK会根据 schema 在实际 handler 执行前验证参数。

这就是 CLI 与 MCP 一个非常根本的差别：

```text
CLI
更强调“如何执行”

MCP
更强调“我拥有哪些能力，以及能力的契约是什么”
```

---

# 4. MCP 2026 已经不是很多旧教程里的 MCP

理解 MCP 时需要特别注意版本。

截至本文，正式规范为：

```text
2026-07-28
```

这是 MCP 一次非常大的架构变化。

2025 年及以前的 MCP 通常是：

```text
initialize
     ↓
notifications/initialized
     ↓
建立 session
     ↓
Mcp-Session-Id
     ↓
后续请求
```

而 MCP `2026-07-28` 将协议核心改成了：

> **Stateless、self-contained requests。**

`initialize / initialized` 被移除，协议级 `Mcp-Session-Id` 也被移除。每个请求携带自己的协议版本和客户端能力；客户端如果希望提前知道 Server 有哪些能力，可以调用新的 `server/discover`。

现在更像：

```text
Request
 ├── Protocol Version
 ├── Client Info
 ├── Client Capabilities
 ├── Method
 └── Parameters
```

单个请求就足以被任意 MCP Server 实例处理：

```text
                Load Balancer
               /      |       \
              /       |        \
          MCP #1    MCP #2    MCP #3
```

不再因为 Session 强制粘在某一个实例上。

这是 MCP 从“桌面 Agent 插件协议”逐渐走向真正服务基础设施的重要一步。

---

# 5. MCP 请求实际上长什么样

假设 Agent 调用：

```text
query_logs(
    server_id="GS17",
    minutes=10
)
```

使用 Streamable HTTP 时，一个简化后的实际请求可以理解成：

```http
POST /mcp HTTP/1.1
Content-Type: application/json
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: query_logs
```

body：

```json
{
  "jsonrpc": "2.0",
  "id": 17,
  "method": "tools/call",
  "params": {
    "name": "query_logs",
    "arguments": {
      "server_id": "GS17",
      "minutes": 10
    },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": {
        "name": "online-debug-agent",
        "version": "1.0"
      }
    }
  }
}
```

2026-07-28 的 Streamable HTTP 增加了 `Mcp-Method` 和 `Mcp-Name` 等 header，使网关、WAF、限流系统可以不解析 JSON body 就知道：

```text
这是 tools/call

调用的是 query_logs
```

这对企业级 MCP 很重要，因为你可以直接做：

```text
Mcp-Name=query_logs
       ↓
允许 1000 QPS

Mcp-Name=execute_lua
       ↓
允许 1 QPS
       ↓
要求额外授权
```

官方 2026 规范专门增加了这种 header-based routing。

---

# 6. `server/discover`：这个 MCP Server 会什么？

MCP 2026 新增：

```text
server/discover
```

Server 必须能够响应这个能力发现接口；Client 可以用它知道：

```text
Server 支持哪些协议版本

有没有 Tools

有没有 Resources

有没有 Prompts

支持哪些额外能力
```

官方 SDK 当前也普遍提供“先 discover，新协议失败后再 fallback 到旧 `initialize`”的兼容模式。

概念上：

```text
Client
  │
  │ server/discover
  ▼
Server
  │
  ├── protocol versions
  ├── tools capability
  ├── resources capability
  ├── prompts capability
  └── extensions
```

于是 Host 才知道后面可以调用哪些接口。

---

# 7. MCP 最重要的接口：`tools/list`

Agent 首先需要知道：

> 有哪些工具？

对应：

```text
tools/list
```

例如响应：

```json
{
  "tools": [
    {
      "name": "query_logs",
      "description": "Query logs from game servers.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "server_id": {
            "type": "string"
          },
          "minutes": {
            "type": "integer",
            "minimum": 1,
            "maximum": 1440
          }
        },
        "required": [
          "server_id"
        ]
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "events": {
            "type": "array"
          },
          "truncated": {
            "type": "boolean"
          }
        }
      }
    }
  ]
}
```

Host 可以把这段定义几乎直接转成 LLM 所理解的 Tool Definition。

也就是说：

```text
MCP tools/list

        ↓

LLM tool definitions
```

MCP 官方客户端文档也明确将 `listTools()` 返回的 `name + description + inputSchema` 视为可以交给 tool-calling 模型的完整定义。

---

# 8. `tools/call`

发现工具以后，真正执行：

```text
tools/call
```

请求：

```json
{
  "jsonrpc": "2.0",
  "id": 18,
  "method": "tools/call",
  "params": {
    "name": "query_logs",
    "arguments": {
      "server_id": "GS17",
      "minutes": 10
    }
  }
}
```

Server 调用：

```text
query_logs(...)
```

得到结果。

这里 MCP 有一个很值得注意的设计：

```text
content
```

和：

```text
structuredContent
```

可以同时存在。

例如：

```json
{
  "content": [
    {
      "type": "text",
      "text": "Found 27 relevant log events."
    }
  ],
  "structuredContent": {
    "count": 27,
    "truncated": false
  },
  "isError": false
}
```

两者面向的对象不同：

```text
content
    ↓
给模型读

structuredContent
    ↓
给程序读
```

`structuredContent` 可以通过工具声明的 `outputSchema` 验证。当前 SDK 已经将 Tool 返回类型与结构化输出紧密结合。

这个设计对你的 GameOps 特别有价值。

不要返回：

```text
"GS17状态正常，玩家82931在线，延迟23ms"
```

然后让下一个程序重新解析字符串。

而可以返回：

```json
{
  "server": "GS17",
  "healthy": true,
  "player": 82931,
  "online": true,
  "latency_ms": 23
}
```

LLM 可以看自然语言描述，程序则直接消费结构化字段。

---

# 9. `isError` 与真正的协议 Error

MCP 里还有一个很容易混淆的地方。

工具执行失败：

```text
query_player(不存在的玩家)
```

通常仍然是一次成功的：

```text
tools/call
```

只是结果：

```json
{
  "content": [
    {
      "type": "text",
      "text": "Player does not exist."
    }
  ],
  "isError": true
}
```

这意味着：

> Tool 执行失败是 Agent 工作过程的一部分。

模型可以：

```text
看到错误
↓
修正参数
↓
再次调用
```

而 JSON-RPC 协议层错误，例如：

```text
method 不存在
参数整体非法
协议版本错误
```

才属于另一类 Protocol Error。官方 SDK 也明确区分 Tool Error result 与 JSON-RPC Protocol Error。

对于 Agent 来说，这是一个非常合理的设计。

---

# 10. Tool Annotation：告诉 Host 这个操作危险不危险

MCP Tool 可以附加 annotation。

典型包括：

```text
readOnlyHint

destructiveHint

idempotentHint

openWorldHint
```

例如：

```text
query_logs
readOnlyHint = true
```

而：

```text
delete_player
readOnlyHint = false
destructiveHint = true
idempotentHint = false
```

Host 可以据此决定：

```text
直接执行

还是

弹出用户确认
```

但一定要强调：

> **Annotation 是 Hint，不是权限系统。**

官方同样明确要求 Tool annotations 应被视为不可信描述，真正的授权、用户同意和访问控制不能依赖它。

所以：

```text
destructiveHint=false
```

绝不意味着：

```text
安全
```

---

# 11. Resource：MCP 不只有 Tool

MCP 里另一个非常重要、却经常被忽略的概念是：

```text
Resource
```

Tool 是：

> 模型决定执行一个动作。

Resource 是：

> 应用程序读取一份数据，并把它作为 Context 提供给模型。

例如：

```text
config://game/production

server://GS17/status

incident://INC-1032

player://82931/profile
```

相关接口包括：

```text
resources/list

resources/templates/list

resources/read
```

比如：

```text
resources/list
```

返回：

```json
{
  "resources": [
    {
      "uri": "config://production",
      "name": "production-config",
      "description": "Current production configuration",
      "mimeType": "application/json"
    }
  ]
}
```

然后：

```text
resources/read
```

读取：

```json
{
  "uri": "config://production"
}
```

返回：

```json
{
  "contents": [
    {
      "uri": "config://production",
      "mimeType": "application/json",
      "text": "{...}"
    }
  ]
}
```

Resource 还能定义 URI Template：

```text
player://{player_id}/profile
```

然后：

```text
player://82931/profile
```

即可读取。

官方规范和 SDK 将固定 URI 的 Resource、Resource Template 与 `resources/read` 明确分开。

---

# 12. Prompt

MCP 还有：

```text
prompts/list
prompts/get
```

Prompt 更接近：

> MCP Server 提供的一套可复用工作流模板。

例如：

```text
analyze_incident
```

需要：

```text
incident_id
```

客户端：

```text
prompts/get(
    "analyze_incident",
    {
        "incident_id": "INC-1032"
    }
)
```

Server 返回若干消息：

```text
system / user / assistant
```

Host 把它们加入会话。

与 Tool 最大的区别可以简单记为：

```text
Tool
→ 模型选择调用

Resource
→ 应用读取数据

Prompt
→ 用户/应用选择一套提示工作流
```

官方 SDK 对 Prompt 的定义也是“用户从客户端选择的消息模板”。

---

# 13. MCP 2026 的 MRTR：工具执行到一半需要人确认怎么办？

这是新版 MCP 一个非常值得关注的能力：

```text
Multi Round-Trip Requests
```

缩写：

```text
MRTR
```

考虑你的场景：

Agent 调用：

```text
execute_lua
```

但 Server 发现：

```text
environment = production
```

于是不能直接执行。

过去可能要求 Server 主动向 Client 发请求。

而 MCP 2026 改成：

```text
Agent
 │
 │ tools/call
 ▼
MCP Server
 │
 │ resultType=input_required
 ▼
Client
 │
 │ 向人询问
 ▼
User
 │
 │ Approve
 ▼
Client
 │
 │ 重新发送 tools/call + inputResponses
 ▼
MCP Server
```

也就是说 Server 可以返回：

```text
input_required
```

要求：

```text
用户确认
补充参数
模型采样
其他输入
```

Client 获得输入后，重新执行原请求。

这是 2026-07-28 为 stateless MCP 重新设计的重要机制。

对于你的线上系统，这比简单的：

```text
confirm=true
```

更有价值。

因为：

```text
Production destructive action
        ↓
    input_required
        ↓
     Human approval
        ↓
     execute
```

可以成为协议流程本身的一部分。

---

# 14. MCP 的 Transport

MCP 本质上与传输方式分离。

当前最值得关注的是两种。

## 14.1 stdio

适合：

```text
本地工具
IDE Plugin
Coding Agent
桌面 Agent
```

结构：

```text
Host
 │
 │ spawn
 ▼
MCP Server Process

stdin  ◀──── JSON-RPC ────▶ stdout
```

例如：

```text
Codex
   ↓
启动
   ↓
gameops-mcp
   ↓
stdin/stdout
```

优点：

```text
不需要端口
不需要部署HTTP
生命周期由Host管理
本机集成简单
```

官方 SDK 将 stdio 定位为 Client 启动 Server 子进程并通过 stdin/stdout 交换 JSON-RPC 的本地集成方式。

---

## 14.2 Streamable HTTP

适合：

```text
共享服务
企业平台
远程 Agent
云端 MCP
多人共用 GameOps
```

结构：

```text
Agent A ─┐
Agent B ─┼── HTTP ──> MCP Gateway ──> GameOps
Agent C ─┘
```

对于新服务，这是远程 MCP 的主要选择。

旧的：

```text
HTTP + SSE
```

已经被明确标记为 legacy / deprecated，新实现应优先 Streamable HTTP。

---

# 15. List 接口还有两个容易忽略的能力

MCP 的 `list` 系列支持分页：

```text
cursor
nextCursor
```

例如：

```text
tools/list
resources/list
resources/templates/list
prompts/list
```

都可以分页。

典型过程：

```text
cursor=null
   ↓
第一页
   ↓
nextCursor="100"
   ↓
第二页
   ↓
...
```

直到：

```text
nextCursor=null
```

官方 Client SDK 当前也是以这种形式遍历所有 list 接口。

MCP 2026 同时加入：

```text
ttlMs
cacheScope
```

例如：

```json
{
  "tools": [...],
  "ttlMs": 60000,
  "cacheScope": "private"
}
```

意味着：

> 这份 Tool 列表一分钟内不用重新拉。

这对 Tool 数量很大的 Agent 平台非常重要。

---

# 16. 到这里再看 CLI 和 MCP，区别已经比较清楚

可以把两种模式画成：

```text
CLI：

LLM
 │
 │ 写程序 / shell
 ▼
Computer
 │
 ├── gameops
 ├── rg
 ├── git
 ├── jq
 ├── python
 └── curl
```

而 MCP：

```text
LLM
 │
 ▼
Host
 │
 ▼
MCP Client
 │
 │ tools/list
 │ tools/call
 ▼
MCP Server
 │
 ├── query_logs
 ├── query_player
 ├── server_status
 └── execute_lua
```

两者实际上代表两种不同的 Agent 哲学。

---

# 17. 第一条决策原则：是否需要大量“组合”

假设要找：

> 最近半小时内所有发生重连，并且随后出现背包异常的玩家。

CLI：

```bash
gameops logs --last 30m --json |
jq '...' |
rg "Reconnect|BagError" |
python correlate.py
```

这里一个 LLM 决策，可以产生：

```text
几十次
几百次
甚至几万次

确定性计算
```

最后只有结果进入 LLM。

这是极其重要的 Agent 模式：

```text
LLM

一次语义决策
      ↓
大量确定性计算
      ↓
压缩结果
      ↓
再次语义决策
```

如果完全把这些操作拆成 MCP Tool：

```text
LLM
 ↓
query_log
 ↓
LLM
 ↓
filter_log
 ↓
LLM
 ↓
query_player
 ↓
LLM
 ↓
compare
```

就会产生：

```text
更多模型调用
更多Token
更多Latency
更长Trajectory
```

因此：

> **需要大量 pipeline / loop / map / reduce / filter 的操作，优先 CLI / Code Execution。**

---

# 18. 第二条原则：数据量越大，越偏向 CLI

假设：

```text
10分钟日志
=
500MB
```

错误做法：

```text
query_logs MCP
      ↓
500MB
      ↓
LLM Context
```

Agent 根本不应该看到这么多东西。

应该：

```text
500MB日志
   ↓
CLI / SQL / Python
   ↓
filter
   ↓
aggregate
   ↓
5KB Evidence
   ↓
LLM
```

所以：

```text
大数据处理
批处理
日志分析
代码搜索
数据转换
```

CLI 更有优势。

---

# 19. 第三条原则：业务语义越强，越偏向 MCP

例如：

```bash
rg
grep
cat
git
python
```

这些是：

> Computer Primitive。

Agent 本来就非常熟悉。

没必要把：

```text
grep()
read_file()
list_directory()
```

全部重新设计成你自己的业务 MCP。

但：

```text
get_player_state

get_match_result

execute_online_lua

restart_game_server

compensate_player_item
```

这些已经属于：

> Domain Capability。

这里 MCP 的价值就非常明显。

因此可以形成一条边界：

```text
通用计算能力
────────────────
rg
grep
git
python
jq
curl
find

        ↓

       CLI


业务能力
────────────────
PlayerState
MatchState
Incident
ExecuteLua
RestartServer
Compensation

        ↓

       MCP
```

---

# 20. 第四条原则：副作用越大，越偏向 MCP

读取：

```text
logs
code
status
```

失败了通常只是：

```text
没得到答案。
```

而：

```text
restart_server
execute_lua
change_config
delete_data
```

失败可能导致真实生产事故。

这种能力更应该是：

```text
Agent
  ↓
Structured Tool
  ↓
Schema Validation
  ↓
Policy
  ↓
RBAC
  ↓
Approval
  ↓
Audit
  ↓
Executor
```

而不是：

```text
LLM
 ↓
bash
 ↓
ssh root@prod
```

因此：

> **风险越高，越应该减少 Agent 的操作空间。**

CLI 强在：

```text
Freedom
```

MCP 强在：

```text
Constraint
```

---

# 21. 第五条原则：是否需要 Tool Discovery

CLI 通常依赖：

```bash
gameops --help
```

或者：

```text
AGENTS.md
README
Skill
Prompt
```

告诉 Agent：

> 你可以使用哪些命令。

MCP 则拥有正式的：

```text
server/discover
tools/list
```

所以：

> 如果工具数量多、动态变化，或者希望任意 Host 自动接入，MCP 明显更适合。

例如未来公司内部可能有：

```text
GameOps
GEP
SVN
CI
Monitoring
DB
PlayerService
MatchService
```

几十、几百个 Tool。

这时候：

```text
自动 Discovery
+
Schema
+
Tool Metadata
```

的重要性会迅速提高。

---

# 22. 第六条原则：谁需要直接使用？

CLI 最大的优势之一是：

> 人也非常容易使用。

事故发生时工程师直接：

```bash
gameops status GS17
```

即可。

它同时天然服务：

```text
Human
Agent
Shell Script
CI
Cron
Python
```

而 MCP 更偏：

```text
Agent / AI Application
```

所以：

> **如果一个能力同时是工程师日常工具，优先确保有优秀的 CLI。**

---

# 23. 第七条原则：是否需要跨 Agent Host

CLI 的前提通常是：

```text
Agent 有 shell。
```

Coding Agent 几乎都有：

```text
Codex
Claude Code
mini-swe-agent
```

因此 CLI 非常有效。

但如果以后是：

```text
IDE Agent
Web Agent
手机 Agent
企业聊天 Agent
云端 Agent
```

它们不一定有：

```text
你的shell
你的机器
你的PATH
```

这时候：

```text
MCP Server
```

成为一个标准能力入口。

所以：

> Coding Agent 内部生态，CLI 通用性已经很强。

> 跨 Host、跨设备、跨服务时，MCP 的互操作性更有优势。

---

# 24. 第八条原则：审计

CLI：

```text
command:
gameops lua GS17 xxx

stdout:
...

exit_code:
0
```

当然也能审计。

但是 MCP 天然产生：

```json
{
  "tool": "execute_lua",
  "arguments": {
    "server_id": "GS17",
    "script_id": "fix_INC1032"
  },
  "result": {
    "success": true
  }
}
```

因此容易统计：

```text
query_logs        18372
query_player       2219
execute_test_lua     93
execute_prod_lua      4
restart_server        1
```

这对 Agent 平台长期的：

```text
Observability
Eval
Security
Cost Analysis
```

非常重要。

---

# 25. 第九条原则：探索还是控制？

这是我认为 CLI 与 MCP 最深层的区别。

CLI：

```text
你有一台电脑。

自己想办法。
```

Agent 可以：

```text
写Python
grep
git bisect
生成临时脚本
组合命令
建立缓存
创建中间文件
```

它可能发明出系统设计者完全没有想到的方法。

这是：

```text
Exploration
```

MCP：

```text
这里有20个合法动作。

请选择。
```

这是：

```text
Controlled Action Space
```

于是大体存在这样的关系：

```text
             自由度

CLI     ███████████████████

MCP     ███████████


             可控性

CLI     █████████

MCP     ███████████████████
```

Agent 系统设计本质上一直在两者之间寻找平衡。

---

# 26. 一张实用决策表

| 问题                    | 更偏 CLI | 更偏 MCP |
| --------------------- | ------ | ------ |
| 是否需要 Shell Pipeline   | ✓✓✓    |        |
| 是否需要 Python 批处理       | ✓✓✓    |        |
| 是否处理大量原始数据            | ✓✓✓    |        |
| 是否需要自由探索              | ✓✓✓    |        |
| 人是否频繁直接使用             | ✓✓✓    |        |
| 是否是通用计算机能力            | ✓✓✓    |        |
| 是否需要动态 Tool Discovery |        | ✓✓✓    |
| 参数是否需要强 Schema        |        | ✓✓✓    |
| 是否需要结构化输出契约           |        | ✓✓✓    |
| 是否跨多个 Agent Host      |        | ✓✓✓    |
| 是否业务语义很强              |        | ✓✓✓    |
| 是否有生产副作用              |        | ✓✓✓    |
| 是否需要统一治理              |        | ✓✓✓    |
| 是否需要结构化审计             |        | ✓✓✓    |

但这张表并不是让你二选一。

更合理的设计通常是下一种。

---

# 27. 不要让 CLI 和 MCP 各写一份业务逻辑

错误架构：

```text
gameops CLI
 ├── 自己查日志
 ├── 自己连DB
 └── 自己执行Lua


GameOps MCP
 ├── 又写一次查日志
 ├── 又写一次DB
 └── 又写一次Lua
```

这最终一定会产生：

```text
行为不一致
Bug修两次
权限逻辑不一致
测试重复
```

真正应该形成：

```text
                 GameOps Core

        ┌────────────┼────────────┐

     LogService   PlayerService   ServerService
         │              │              │
         │              │              │
         └──────────────┼──────────────┘
                        │
              ┌─────────┴─────────┐
              │                   │
             CLI                 MCP
              │                   │
             Human             Agent Host
             Shell             Codex
             Script            Claude
             Agent             IDE
```

也就是：

> **业务能力只实现一次。**

CLI 与 MCP 只是 Adapter。

---

# 28. 对 Online Debug Agent，我会采用双平面设计

你的线上问题分析系统非常适合分成：

```text
Investigation Plane

和

Control Plane
```

## Investigation Plane

负责：

```text
查询
搜索
分析
关联
统计
验证假设
```

例如：

```text
grep logs
search code
git history
query trace
download logs
python analysis
```

这里应该给 Agent 比较大的自由。

因此：

```text
CLI + Shell + Python
```

非常合适。

架构：

```text
                Agent
                  │
                 Shell
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
      rg        Python     gameops
       ↓          ↓          ↓
     Code       Analyze   Logs/Trace
```

---

# 29. Control Plane

当 Agent 从：

```text
“研究问题”
```

进入：

```text
“改变现实”
```

立刻收紧。

例如：

```text
execute_lua

restart_server

modify_config

kick_player

compensate_item

deploy_hotfix
```

架构：

```text
                Agent
                  │
               MCP Tool
                  │
             Validation
                  │
                Policy
                  │
                 RBAC
                  │
               Approval
                  │
                Audit
                  │
              Executor
                  │
             Production
```

这就是我认为最适合游戏线上 Agent 的边界。

---

# 30. 一个完整的事故分析例子

假设：

```text
GS17 19:21 崩溃
```

Scenario Runner 创建任务：

```text
Analyze Incident INC-1032
```

Agent 第一阶段：

```bash
gameops incident INC-1032

gameops logs GS17 \
    --from 19:20 \
    --to 19:22 \
    --json > /tmp/incident.json

jq ... /tmp/incident.json

rg "RemoveItem" ./server

git log -p -- server/bag
```

Agent 得出假设：

```text
Player重连后，
Bag Slot Cache未刷新，
UseItem进入RemoveItem，
访问了已经失效的Slot。
```

然后验证：

```bash
gameops trace-player 82931 --before 19:21
```

找到：

```text
19:20:51 reconnect
19:21:03 use_item
19:21:03 crash
```

此时：

```text
CLI
```

已经完成了绝大多数分析。

接下来 Agent 提议：

```text
执行一段Lua临时保护，
阻止该异常路径。
```

这里切换：

```text
MCP
```

调用：

```text
execute_lua
```

参数：

```json
{
  "server_id": "GS17",
  "environment": "production",
  "script_id": "INC-1032-hotfix",
  "reason": "Prevent invalid BagSlot access",
  "dry_run": false
}
```

MCP Server 返回：

```text
input_required
```

要求：

```text
Production modification requires approval.
```

工程师：

```text
Approve
```

然后执行。

这时：

```text
Agent自由分析能力

和

生产系统控制能力
```

就被非常清晰地分开了。

---

# 31. 长任务怎么办？

还有一种情况：

```text
全服扫描
大规模日志分析
构建测试环境
回放1000场比赛
```

可能持续较久。

MCP 2026 已经将长任务抽象移动到官方：

```text
Tasks Extension
```

它可以让 `tools/call` 返回一个 Task Handle，再由 Client 使用类似：

```text
tasks/get
tasks/update
tasks/cancel
```

的机制处理持久化长任务。

Tasks 已经从 2025 年的实验核心能力迁移成独立扩展，以适应新版 Stateless 架构。

所以以后：

```text
analyze_entire_cluster
```

这种操作并不一定需要阻塞一次 `tools/call`。

---

# 32. 一个简单的选型算法

以后每新增一个能力，可以问自己四个问题。

第一问：

```text
Agent 是否需要把它和
其他命令自由组合？
```

是：

```text
CLI
```

第二问：

```text
这是一个明确的业务动作吗？
```

是：

```text
MCP 候选
```

第三问：

```text
这个操作会改变线上状态吗？
```

是：

```text
MCP
+
Policy
+
Approval
```

第四问：

```text
是否涉及大量循环、
过滤、批处理和中间数据？
```

是：

```text
CLI / Code Execution
```

这四个问题基本能够解决大部分选型。

---

# 33. GameOps 能力可以这样划

```text
search_code
→ CLI

git_history
→ CLI

read_file
→ CLI

grep_logs
→ CLI

download_log
→ CLI

local_python
→ CLI
```

中间区域：

```text
query_player
query_server_status
query_match
query_incident
```

可以：

```text
CLI + MCP
```

危险动作：

```text
execute_lua
restart_server
change_config
kick_player
compensate_item
deploy_hotfix
```

主要：

```text
MCP
```

同时底层全部复用：

```text
GameOps Core
```

---

# 34. 一个更重要的结论

如果只把 MCP 理解成：

> “比 CLI 更适合 AI 的函数调用协议。”

其实低估了 MCP。

MCP 真正正在解决的是：

```text
Agent生态里的能力互操作问题。
```

它试图标准化：

```text
如何发现能力

如何描述输入

如何描述输出

如何调用

如何读取上下文

如何暴露Prompt

如何要求用户输入

如何订阅变化

如何认证

如何做扩展
```

而 CLI 解决的是一个更加古老，也更加底层的问题：

```text
一个程序如何成为计算机环境中的可组合能力。
```

因此两者并不存在真正意义上的替代关系。

---

# 35. 最终可以形成这样一个技术栈

```text
                    Scenario
                       │
                       ▼
                     Agent
                       │
        ┌──────────────┴──────────────┐
        │                             │
   Exploration                    Action
        │                             │
     Shell / CLI                     MCP
        │                             │
 ┌──────┼──────┐                      │
 │      │      │                      │
git    rg    python                Policy
 │      │      │                      │
 └──────┴──────┘                      │
        │                          Approval
        │                             │
     GameOps CLI                  GameOps MCP
        │                             │
        └──────────────┬──────────────┘
                       │
                  GameOps Core
                       │
           ┌───────────┼───────────┐
           │           │           │
         Logs        Server       Player
           │           │           │
           └───────────┼───────────┘
                       │
                  Production
```

在这套结构里：

```text
CLI
不是临时方案。

MCP
也不是CLI的升级版。
```

两者承担的是完全不同的责任。

CLI 提供：

> **自由组合的计算能力。**

MCP 提供：

> **结构化、可发现、可治理的 Agent 能力。**

而真正需要长期投资的部分，是二者下面的：

```text
Domain Core

Scenario

Policy

Verify

Eval

Observability
```

因为今天上层可能是：

```text
Codex
```

明天可能是：

```text
Claude Code
mini-swe-agent
Qoder
内部Agent
未来新的Agent Runtime
```

但：

```text
QueryLogs
PlayerTrace
ServerControl
Incident Model
Permission
Verification
Eval Dataset
```

这些才是企业真正长期积累下来的 Agent 基础设施。

因此，如果要用一句话总结 CLI 与 MCP 的选择：

> **让 Agent 探索计算世界时，用 CLI；让 Agent 调用业务世界时，用 MCP；让 Agent 改变生产世界时，用 MCP + Policy + Human Approval。**

而最理想的工程架构，不是：

```text
CLI OR MCP
```

而是：

```text
Core Capability
      │
 ┌────┴────┐
CLI       MCP
```

**能力实现一次，交互接口根据场景选择。**
