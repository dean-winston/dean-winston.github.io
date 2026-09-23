---
layout: article
title: "MCP 为什么走向 Stateless：从一次协议升级看状态、连接与抽象边界"
description: "从 MCP 协议调整出发，讨论连接状态、任务状态与业务状态之间的边界。"
category: Agent Engineering
weight: 5
---

# MCP 为什么走向 Stateless：从一次协议升级看状态、连接与抽象边界

MCP 在 **2026-07-28** 版本里做了一次很重要的架构调整：取消强制的 `initialize / initialized` 握手和协议级 `Mcp-Session-Id`，把核心协议从一个偏 Stateful 的连接模型，转向 **Stateless Request/Response** 模型。

官方对这次变化的描述很直接：每个 Request 应尽可能自包含，可以被任何 Server 实例独立处理，从而获得更好的扩展性、可靠性和实现简洁性。

但为什么 MCP 会走到这一步？

一个很直观的解释是：**MCP 正从一对一走向一对多、多对多。**

这个解释是对的，但还不是问题的最底层。

---

## 从一对一到分布式，Session 开始成为负担

早期 MCP 很容易想象成：

```text
Agent
  │
  │ initialize
  ▼
MCP Server
  │
  └── Session A
```

后续调用：

```text
Session A

tools/call
tools/call
tools/call
```

Server 可以把协议版本、Client Capability，甚至部分应用状态都与这个 Session 联系起来。

在一个 Agent 对一个 Server 进程的环境中，这套模型非常自然。

但 MCP 进入真正的服务化环境后，结构逐渐变成：

```text
          Agent / Host
               │
               ▼
         Load Balancer
        /      |       \
       ▼       ▼        ▼
    MCP #1   MCP #2   MCP #3
```

第一次请求到了 MCP #1：

```text
Session ABC
```

第二次请求却可能到 MCP #2。

此时 MCP #2 并不知道 `ABC` 是什么。

工程上当然可以解决：

```text
方案一：Sticky Session

Client ──────────> MCP #1
```

或者：

```text
方案二：

            Redis
              ▲
        ┌─────┼─────┐
        │     │     │
       MCP1  MCP2  MCP3
```

但问题变成了：

> 为什么一个简单的 `query_logs()`，仅仅因为使用 MCP，就必须承担分布式 Session 管理的复杂度？

SEP-2575 因此把旧设计的问题总结为三类：**扩展困难、故障恢复困难，以及 Client/Server 实现复杂度增加**。

从这个角度看，“一对一走向一对多”确实是这次变化非常重要的现实推动力。

---

## 但一对多不是根因

如果进一步追问，会发现一个反例：

即使永远只有：

```text
一个 Agent
    │
    ▼
一个 MCP Server
```

Session 仍然有问题。

因为 Server 作者并不知道：

```text
Session
```

究竟意味着什么。

是：

```text
一次 Tool Call？
一次 Conversation？
一次页面打开？
一次 Agent 启动？
一次应用生命周期？
```

SEP-2567 总结了真实 MCP Client 的情况：有的 Client 每次 Tool Call 创建 Session，有的按应用生命周期创建，有的按页面生命周期创建，而且几乎没有形成一致的 Session 恢复语义。

于是一个 Server 如果把浏览器实例、数据库事务或者购物车绑定到 Session：

```text
Session
   │
   └── Application State
```

它实际上是在依赖一个**自己无法定义生命周期的抽象**。

这才是更深的问题。

---

# 真正改变的是：状态到底属于谁

旧模型实际上隐含了：

```text
Connection
    │
    ▼
Session
    │
    ▼
Context / State
```

也就是说：

> **通信通道拥有状态。**

新版 MCP 则更倾向：

```text
Request
   │
   ├── protocol version
   ├── client info
   ├── capabilities
   └── arguments

Application State
   │
   ▼
Explicit State Handle
```

例如旧模型可能是：

```text
Session ABC

里面隐含：
basket = 某个购物车
```

然后：

```text
add_item("apple")
```

新版更鼓励：

```text
create_basket()

→ basket_id = B123
```

之后：

```text
add_item(
    basket_id = "B123",
    item = "apple"
)
```

SEP-2567 把这个模式称为 **Explicit State Handles**：应用状态仍然可以存在，只是不再隐藏在 MCP Session 里，而是由业务对象自己显式表达。

所以 MCP 变成 Stateless，并不意味着：

> 系统不能有状态。

而是：

> **协议本身不要替业务偷偷持有状态。**

---

## 前后的结构差异

旧模型：

```text
             Client
               │
          initialize
               │
               ▼
        ┌──────────────┐
        │ MCP Server 1 │
        │              │
        │ Session ABC  │
        │ ├─ Version   │
        │ ├─ Capability│
        │ └─ State     │
        └──────────────┘
               ▲
               │
      后续调用依赖 ABC
```

状态和某一次连接、某一个实例存在较强绑定。

新版：

```text
                   Client
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
       Request    Request    Request
          │          │          │
          ▼          ▼          ▼
       MCP #1     MCP #3     MCP #2

每个 Request：
├─ Protocol Version
├─ Client Info
├─ Capability
├─ Tool Arguments
└─ Explicit State ID
```

于是请求落在哪个实例已经没那么重要。

官方甚至明确把这一点作为新版的重要收益：普通 round-robin load balancer 就可以把任意 Request 发送给任意 MCP Server 实例。

---

# 所以，一对多到底是不是原因？

可以说：

> **是重要诱因，但不是最根本原因。**

一对多、负载均衡和水平扩展让 Session 的问题迅速暴露：

```text
Topology 变化
      ↓
Session 变得昂贵
```

但真正值得学习的是下一层：

```text
Session 为什么会变得昂贵？
      ↓
因为状态被绑定到了错误的对象上
```

所以更完整的因果关系应该是：

```text
MCP使用规模扩大
        ↓
Host / Agent / Server形态复杂化
        ↓
Connection生命周期不再等于业务生命周期
        ↓
Session语义开始模糊
        ↓
隐式状态产生扩展、恢复、缓存等问题
        ↓
状态从Connection中拆出
        ↓
Stateless Request
+
Explicit State Handle
```

这比单纯理解为：

```text
1 : 1
 ↓
1 : N
 ↓
Stateless
```

更接近这次协议变化的本质。

---

# 从这次升级里可以学到的几个抽象思维

最值得学习的第一点是：

## 不要把“相关”误认为“属于”

一次 Incident 可能由某个 Agent 创建。

但：

```text
Incident
```

并不因此：

```text
属于 Agent Session。
```

同样：

```text
购物车
数据库事务
线上事故
测试任务
```

虽然通过某条 Connection 被操作，但它们不属于 Connection。

判断状态放在哪里时，更好的问题是：

> **如果通信连接消失，这个对象是否应该一起消失？**

如果答案是否定的，它大概率就不应该属于 Connection。

---

第二个抽象是：

## 隐式 Context 在规模扩大后往往会成为负债

隐式状态很方便：

```text
不用每次传 incident_id。
```

但它同时意味着：

```text
调用本身无法解释自己。
```

显式：

```text
execute_lua(
    incident_id="INC-102",
    server_id="GS17"
)
```

虽然参数更多，却天然获得：

```text
可重放
可审计
可测试
可迁移
可并发
```

很多大型系统最终都会经历：

```text
Implicit Context
       ↓
Explicit Context
```

的过程。

---

第三个抽象是：

## 优先降低默认复杂度，而不是满足所有复杂场景

SEP-2575 使用了一个很好的原则：

> **Pay as you go。**

简单调用：

```text
query_logs()
```

就应该是一个简单 Request。

真正需要长任务、状态或者多轮交互时，再引入：

```text
Task
State Handle
MRTR
```

而不是因为少数复杂场景，让所有 Tool 都承担 Session、恢复和状态同步成本。

这其实是一条很好的架构原则：

```text
Simple Case
    ↓
Simple Abstraction

Complex Case
    ↓
显式支付复杂度
```

---

# 对 Agent 系统设计的启发

这次 MCP 改动真正值得带走的一句话，我认为不是：

> MCP 以后更容易负载均衡了。

而是：

> **状态应该属于业务对象，而不是属于通信通道。**

例如线上问题分析系统：

不要设计成：

```text
Codex Session
    │
    └── 当前事故全部状态
```

而应该：

```text
Incident INC-102
├── Evidence
├── Hypothesis
├── Actions
├── Approval
└── Result
```

Agent 只是：

```text
INC-102 的一个执行者
```

于是今天可以：

```text
Codex
  ↓
INC-102
```

明天 Codex 挂掉：

```text
mini-swe-agent
       ↓
    INC-102
```

仍然能够继续。

这就是把：

```text
Agent Session
```

和：

```text
Task / Business State
```

真正分开。

而这可能才是 MCP 这次 Stateless 改造背后，最值得学习的抽象。
