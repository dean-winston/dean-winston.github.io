---
layout: article
title: "AI Agent 时代，软件项目需要一层新的基础设施：Agent Interface"
description: "从 AI Coding 的真实研发流程出发，讨论软件项目为什么需要面向 Agent 的观察和操作接口。"
category: Agent Engineering
weight: 1
---

# AI Agent 时代，软件项目需要一层新的基础设施：Agent Interface

AI Coding 正在迅速进入真实的软件研发流程。

今天的 Coding Agent 已经可以完成很多过去只能由程序员完成的工作：

- 阅读大型代码库
- 修改代码
- 重构模块
- 分析调用链
- 修复 Bug
- 编写测试
- 生成脚本与工具

但在真实项目中，经常会出现一个非常明显的断层：

> Agent 会写代码，却不会“使用你的项目”。

它可能能够找到一个 Bug 的代码位置，也能给出看起来合理的修改方案，但修改完成之后，它往往无法继续往下走：

- 服务怎么启动？
- 如何创建一个测试玩家？
- 如何让玩家进入特定状态？
- 如何构造 Bug 出现时的环境？
- 如何发送一个客户端协议？
- 如何查询运行中的对象状态？
- 如何搜索指定时间段的日志？
- 如何判断修改之后问题真的消失了？

于是很多 AI Coding 的实际流程仍然停留在：

```text
Agent
  ↓
修改代码
  ↓
无法进入真实运行环境
  ↓
“从代码上看应该已经修复”
```

这说明，真正限制 AI Agent 参与软件研发的因素，正在逐渐从“Agent 会不会写代码”，转变为：

> **项目本身有没有给 Agent 提供进入真实系统的接口。**

---

## 一、今天的大多数软件项目，本质上仍然是为人类研发者设计的

传统软件工程默认的研发执行者是人。

因此，一个项目往往会形成这样的研发环境：

```text
程序员
  ↓
IDE / SSH / Unity / GM 后台 / 数据库客户端 / 运维平台
  ↓
真实系统
```

很多关键研发能力并没有统一的机器接口，而是分散在不同地方。

例如一个游戏项目中，复现一个问题可能需要：

1. 执行某个脚本启动 GC。
2. 再启动三个 GS。
3. 打开 Unity 客户端。
4. 登录指定账号。
5. 使用 GM 指令调整等级和任务状态。
6. 进入指定玩法。
7. 在后台查询数据库。
8. SSH 到机器上搜索日志。
9. 根据经验判断行为是否正常。

一个熟悉项目五年的程序员可能觉得这些操作很自然。

因为大量信息实际上存在于：

```text
人的经验
+
Wiki
+
脚本
+
GUI 工具
+
历史习惯
+
口口相传
```

人能够把这些碎片拼起来。

但对于 AI Agent 来说，这种项目几乎是封闭的。

Agent 可能能够读懂几十万行代码，却不知道怎样真正进入这个系统。

从 Agent 的视角来看：

> **这是一个没有 API 的研发环境。**

---

## 二、软件项目正在出现一种新的接口需求：Agent Interface

过去，我们会为不同角色设计不同接口：

```text
玩家     → Client Interface
运营     → GM Interface
程序员   → Debug Interface
运维     → Ops Interface
```

现在，软件项目开始出现一种新的使用者：

```text
AI Agent → Agent Interface
```

这里的 Agent Interface，并不是给 Agent 一个聊天窗口。

它真正需要解决的是三个核心问题：

```text
Observe
Operate
Verify
```

也就是：

> **观察系统、操作系统、验证系统。**

只有这三种能力形成闭环，AI Agent 才可能真正从“代码生成器”变成“研发执行者”。

---

# 三、Observation：让 Agent 看见真实系统

Agent 读取代码，只是在观察系统的静态世界。

但绝大多数真实软件问题，最终都发生在运行时。

例如：

- 某个请求为什么只在线上超时？
- 某个玩家为什么进入了错误状态？
- 某个服务之间的消息为什么没有到达？
- 某个对象为什么长期没有释放？
- 某个配置为什么只在特定服务器上生效？
- Matchmaking 为什么在低峰期等待时间突然增加？

这些问题仅靠阅读代码通常无法确定答案。

Agent 需要能够读取真实运行环境中的信息。

一个成熟的 Agent Interface，至少应该逐渐暴露这些能力：

```text
logs
metrics
trace
database
runtime state
process state
network state
configuration
```

例如 Agent 应该能够直接查询：

```text
当前启动了多少个 GS？

玩家 10001 当前在哪个 GS？

最近 5 分钟 MatchService 出现了多少 ERROR？

request_id = xxx 的完整调用链是什么？

玩家当前拥有哪些 Buff？

Matchmaking 队列里当前有多少玩家？

数据库中的比赛状态是什么？

当前服务器实际加载的是哪个配置版本？
```

这意味着 Observability 的价值正在发生扩展。

过去我们建设 Observability，主要是为了：

```text
SRE
程序员
运维
```

未来还会多一个非常重要的消费者：

```text
AI Agent
```

因此可以重新理解 Observability：

> **Observability 不再只是给人看的 Dashboard，它正在成为 Agent 理解运行中系统的输入接口。**

---

# 四、Operation：让 Agent 能够改变系统

仅仅能够观察系统还不够。

研发过程本质上经常需要主动制造条件。

例如测试一个游戏服务器功能，Agent 可能需要：

```text
start_server()

stop_server()

restart_server()

create_player()

login_player()

set_level()

create_team()

enter_dungeon()

send_protocol()

execute_lua()

reload_config()
```

今天很多老项目其实已经拥有这些能力。

只是它们可能分别存在于：

- Unity Editor
- GM 工具
- Shell 脚本
- 后台页面
- 数据库工具
- 内部客户端
- Debug Console

这些工具有一个共同特点：

> **它们主要是给人操作的。**

未来这些关键能力应该逐渐被重新抽象成：

> **Machine Callable Capability**

也就是能够被程序、自动化系统和 AI Agent 调用的能力。

理想结构应该逐渐变成：

```text
Human
  ↓
GUI
            Domain Capability
    /
   /
Agent
  ↓
API / Tool
```

例如：

```text
Human Interface          Machine Interface

Unity 按钮       →       API / Command
GM 后台          →       HTTP / RPC
日志平台         →       Log Query API
人工造号         →       Fixture API
手动登录         →       Client Simulator
启动脚本         →       Lifecycle API
```

由此可以形成一条非常重要的软件工程原则：

> **任何重要的研发操作，都不应该只存在 Human Interface，还应该尽可能存在 Machine Interface。**

因为 Machine Interface 不只是服务 Agent。

它同时可以服务：

- 自动化测试
- CI
- Debug 工具
- QA 平台
- 运维系统
- 数据分析
- AI Agent

Agent 只是让这件事情的重要性突然被放大了。

---

# 五、Verification：让 Agent 知道自己做对了没有

如果 Agent 可以修改代码、启动服务、发送请求、查询日志，却无法判断结果是否正确，它仍然无法形成真正的研发闭环。

最弱的流程是：

```text
Agent
  ↓
修改代码
  ↓
运行系统
  ↓
读取一些日志
  ↓
根据文本猜测是否成功
```

真正需要建立的是：

```text
Agent
  ↓
修改
  ↓
制造场景
  ↓
执行操作
  ↓
读取真实状态
  ↓
Expected vs Actual
  ↓
Pass / Fail
```

例如测试一个 Matchmaking 场景：

```text
Scenario: 两个玩家进入匹配

Input:
- Player A rating = 1500
- Player B rating = 1520
- 同一区域
- 同一玩法

Expected:
- 10 秒内 MatchSuccess
- A/B 进入同一场比赛
- 两名玩家状态变为 InGame
- MatchService 不产生 ERROR
- DB 中比赛记录创建成功
```

Agent 可以：

1. 创建两个玩家。
2. 登录两个玩家。
3. 发起匹配。
4. 等待匹配事件。
5. 查询双方状态。
6. 查询数据库。
7. 搜索对应日志。
8. 对照 Expected 判断结果。

到了这里，Agent 才第一次真正拥有了：

> **执行研发任务之后自行验收结果的能力。**

所以 Agent Interface 最终的目的，并不是让 AI 可以调用几个 API。

它真正的目标是：

> **建立 AI Agent 的研发闭环。**

---

# 六、MCP 很重要，但 MCP 本身不是核心

MCP 是当前非常适合承载 Agent Interface 的协议之一。

它提供了一种标准方式，让 Agent 能够发现并调用项目中的工具。

例如：

```text
                 AI Agent
                    │
                    │
                   MCP
                    │
                    ▼
           Agent Interface Layer
                    │
      ┌─────────────┼─────────────┐
      │             │             │
   Server Tool    Log Tool    Client Simulator
      │             │             │
      └─────────────┼─────────────┘
                    │
                Real System
```

Agent 最终看到的可能是：

```text
start_server
stop_server
query_log
query_db
create_player
send_protocol
inspect_state
run_scenario
```

但是这里真正值得长期投资的，并不是 MCP 本身。

真正值得建设的是下面这些：

> **Project Capability**

也就是项目本身是否拥有稳定的、可编程调用的研发能力。

因为协议会变化。

今天可能是 MCP。

未来可能出现新的 Agent Protocol。

但这些能力：

```text
start_server()
create_player()
query_logs()
inspect_state()
run_scenario()
```

不会失去价值。

因此，一个更长期的工程判断应该是：

> **不要为了 MCP 而 MCP。先把项目的重要研发能力变成 Machine Callable Capability，再选择如何暴露给 Agent。**

---

# 七、老项目尤其需要增加这一层

老项目往往有一个非常有意思的特点：

> **系统能力很强，但机器可访问性很差。**

一个运行多年的大型游戏项目，可能已经拥有：

- 非常成熟的 GM 系统
- 大量运维脚本
- 丰富的 Debug 指令
- 日志查询平台
- 测试客户端
- 内部协议工具
- 配置管理平台
- 数据查询工具

换句话说：

> 能力其实已经存在。

问题只是它们基本都围绕：

```text
人 → 工具
```

设计。

而不是：

```text
Agent → Capability
```

因此，对于很多老项目来说，并不一定需要重新建设一整套系统。

更高 ROI 的方式可能是：

```text
原有 GM 命令
      ↓
Agent Tool

原有日志查询平台
      ↓
Log API

原有服务器脚本
      ↓
Server Lifecycle Tool

原有协议测试客户端
      ↓
Client Simulator

原有 Debug 指令
      ↓
Runtime Inspection Tool
```

也就是说：

> **把已经存在的人类研发能力，重新包装成机器可以调用的研发接口。**

这是老项目进入 Agent 时代非常现实的一条路径。

---

# 八、新项目应该从第一天考虑 Agent Ready

对于新项目，这件事情应该进一步前移。

以后设计一个系统时，我们可能不能只问：

> 人怎么使用这个系统？

还应该多问一句：

> **Agent 怎么使用这个系统？**

例如设计 Matchmaking。

不应该让核心逻辑长期隐藏在：

```text
点击按钮
  ↓
UI 状态
  ↓
客户端内部逻辑
  ↓
若干隐式状态
  ↓
最终才进入 Matchmaking
```

更好的方式是让核心业务能力天然存在一个明确入口：

```text
Match(Request)
```

然后：

```text
                  Matchmaking

                 Match(Request)

         ┌────────────┼────────────┐
         │            │            │
        UI          Agent      Automation
```

UI 调用它。

自动化测试调用它。

AI Agent 也调用它。

当你要求系统拥有 Agent Interface 时，会自然推动很多良好的工程属性：

- API 化
- 模块化
- 状态显式化
- 更清晰的边界
- 更好的 Observability
- 更好的 Reproducibility
- 更少隐藏在 GUI 中的业务逻辑

因此：

> **Agent Ready 并不只是 AI 工程，它很可能反过来提升软件架构本身。**

---

# 九、从 Tool Interface 进一步发展到 Scenario Interface

Agent Interface 的第一阶段通常是提供工具。

例如：

```text
start_server
create_player
login
send_protocol
query_log
query_db
```

但如果 Agent 每执行一次任务，都需要自己重新拼装几十个底层工具，那么复杂度依然很高。

更成熟的方向应该继续向上抽象。

例如：

```text
run_scenario("two_player_match")
```

或者：

```text
run_scenario(
    name = "matchmaking_rating_test",
    players = 10,
    rating_range = [1400, 1600]
)
```

Scenario Runner 内部负责：

```text
启动环境
↓
创建玩家
↓
登录
↓
配置状态
↓
发送协议
↓
等待事件
↓
收集日志
↓
读取状态
↓
验证结果
```

这样 Agent Interface 会逐渐形成清晰的层次：

```text
┌───────────────────────────────┐
│        Scenario Layer         │
│ 登录 / 匹配 / 战斗 / Bug 复现 │
├───────────────────────────────┤
│        Control Layer          │
│ create / send / execute       │
├───────────────────────────────┤
│      Observation Layer        │
│ logs / state / db / trace     │
├───────────────────────────────┤
│        Runtime Layer          │
│ start / stop / reset          │
├───────────────────────────────┤
│          Real System          │
│ Server / Client / DB          │
└───────────────────────────────┘

               ↑
             Agent
```

这一层演化很重要。

因为最终 Agent 不应该只是：

> 会调用工具。

而应该能够：

> **调用研发场景。**

---

# 十、软件项目可能需要一种新的工程属性：Agentability

过去评价一个软件项目，我们会问：

```text
好不好维护？
容不容易扩展？
稳定不稳定？
性能怎么样？
容易部署吗？
容易观察吗？
```

AI Agent 大规模进入研发之后，可能还需要增加一个新的问题：

> **这个项目对 AI Agent 友好吗？**

可以把这种能力称为：

# Agentability

也就是：

> **一个软件系统允许 AI Agent 理解、观察、操作、复现和验证它的能力。**

它大致包含：

```text
Understandable
    +
Observable
    +
Operable
    +
Reproducible
    +
Verifiable
```

Agentability 高的项目，Agent 可以：

```text
理解问题
↓
阅读代码
↓
修改实现
↓
启动系统
↓
制造场景
↓
复现问题
↓
观察运行状态
↓
验证修改结果
```

而 Agentability 很低的项目，即使接入最强的 Coding Agent，AI 依然只能站在系统外面：

> 阅读代码，然后猜测真实系统会发生什么。

两种项目在未来 AI 研发效率上的差距，很可能会越来越大。

---

# 十一、AI Coding 的下一阶段，不只是让 Agent 更会写代码

过去两年，大量注意力集中在模型能力：

- 哪个模型更会写代码？
- 哪个模型上下文更长？
- 哪个 Agent 能连续工作更久？
- 哪个 Agent 更擅长理解大型代码库？

这些当然重要。

但当 Coding Agent 的代码能力不断提高以后，一个新的瓶颈会越来越明显：

> **项目本身是否允许 Agent 真正参与研发。**

一个完全封闭的项目，即使接入非常强的 AI，也可能只能做到：

```text
阅读代码
+
修改代码
```

而一个高度 Agent Ready 的项目，可以让 AI 完成：

```text
理解
↓
修改
↓
运行
↓
观察
↓
复现
↓
验证
↓
再次修改
```

这两者已经不是同一种 AI Coding。

前者只是：

> **AI Assisted Coding**

后者开始接近：

> **Agent Driven Development**

---

# 结语

过去的软件项目主要为人提供接口。

玩家有客户端。

运营有 GM 后台。

程序员有 Debug 工具。

运维有监控平台。

而 AI Agent 成为新的研发执行者之后，现代软件项目可能还需要增加一种新的基础设施：

> **Agent Interface。**

它让 Agent 能够：

```text
Observe
Operate
Verify
```

最终建立：

```text
修改代码
↓
运行系统
↓
制造场景
↓
观察行为
↓
验证结果
↓
继续迭代
```

只有当这个闭环真正建立之后，AI 才算真正进入软件研发过程，而不仅仅进入代码编辑过程。

因此，未来软件工程的一个重要问题可能不再只是：

> 这个项目是否容易被人维护？

还会增加：

> **这个项目，是否允许 AI Agent 真正工作？**

而这也许会成为未来项目工程质量的一项基础指标：

# Agentability
