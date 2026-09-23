---
layout: article
title: "从 Spec + TDD 到 Agent Interface：我对 AI 研发着力点的一次变化"
description: "从规范和测试，到 Harness 与 Agent Interface，梳理 AI 研发实践中的思路变化。"
category: AI Coding
weight: 2
---

# 从 Spec + TDD 到 Agent Interface：我对 AI 研发着力点的一次变化

过去一段时间，我一直在思考一个问题：

> **如果希望 AI Agent 真正参与软件研发，我们最应该建设的是什么？**

一开始，我把注意力放在开发范式上，希望通过更明确的 Spec、更严格的测试约束 Agent 的行为；后来，我开始构建 Harness，希望 Agent 能够自己运行项目、模拟客户端、复现 Bug 并完成验证；再往后，我逐渐意识到，更上位的问题其实不是“怎么让 Agent 完成某一种开发任务”，而是：

> **项目本身是否给 Agent 提供了足够好的观察和操作能力。**

这几次变化并不是前一种方案被后一种方案推翻，而是我对问题本质的认识逐渐下沉。

最终，关注点从“如何约束 Agent 写代码”，转向了“如何把整个项目改造成 Agent 可以真正工作的环境”。

---

## 一、阶段一：通过 Spec + TDD 约束 Agent 开发

最开始的思路比较接近传统的软件工程方法。

既然 Agent 容易：

- 理解偏差；
- 改动范围失控；
- 局部实现合理但整体不符合需求；
- 写出“看起来正确”但实际上不可用的代码；

那么一个自然的想法就是：

> **给 Agent 更清晰的输入和更明确的验收标准。**

于是开发流程可以设计成：

```text
Spec
  ↓
明确需求与边界
  ↓
定义测试
  ↓
Agent Coding
  ↓
测试反馈
  ↓
继续修改
```

这一阶段关注的核心问题是：

> **怎么让 Agent 按正确的方法写代码？**

Spec 用来约束需求理解。

测试用来约束实现结果。

如果输入足够明确、测试覆盖足够完整，那么 Agent 理论上就能在一个相对受控的范围内工作。

这个方向当然有价值，但它背后隐含了一个假设：

> **Agent 已经拥有足够完整的研发环境，我们主要缺的是行为约束。**

在真实项目里，这个假设很快就会遇到问题。

Agent 可能已经正确理解需求，也写出了合理的代码，但修改之后，它往往无法回答：

- 服务到底能不能启动？
- Bug 是否真的可以复现？
- 真实客户端会不会触发这段逻辑？
- 玩家当前状态到底是什么？
- 数据库里是否产生了正确的数据？
- 运行时有没有出现新的错误日志？
- 这次修改究竟是真的修复，还是只是代码上看起来合理？

问题开始从“怎么写”转向“怎么获得真实反馈”。

---

## 二、阶段二：构建 Harness，让 Agent 自己运行、复现和验证

第二阶段，我开始把重点放在 Harness 上。

这里的思路发生了一个明显变化：

> **Agent 最大的问题不只是缺少约束，它还缺少反馈。**

对于一个真实软件项目来说，研发并不是：

```text
读代码
↓
改代码
↓
结束
```

而应该是：

```text
理解问题
↓
修改代码
↓
运行系统
↓
制造输入
↓
观察行为
↓
验证结果
↓
继续修改
```

于是，需要给 Agent 补齐这些能力。

以游戏项目为例，一个 Server Harness 可能需要提供：

```text
start_server
stop_server
restart_server
get_server_status
execute_lua
http_call
send_protocol
read_logs
```

同时，再增加一个 Client Simulator：

```text
login_player
keep_connection
send_protocol
receive_protocol
record_protocol_log
```

这样 Agent 才能真正执行类似这样的流程：

```text
收到一个 Bug
↓
定位相关代码
↓
修改实现
↓
启动 GC / GS
↓
模拟客户端登录
↓
构造玩家状态
↓
发送协议
↓
读取日志
↓
检查状态
↓
判断 Bug 是否修复
```

这里尤其重要的是“模拟客户端”。

很多游戏服务端行为并不是调用一个函数就能触发，而是依赖：

```text
客户端状态
↓
协议输入
↓
服务端状态变化
↓
进一步协议交互
```

如果 Agent 没有一个可控的客户端模拟能力，它仍然无法进入大量真实业务路径。

因此，Client Simulator 实际上是在给 Agent 增加一只可以操作真实系统的“手”。

这一阶段关注的问题变成：

> **怎么让 Agent 自己运行代码，并知道自己写得对不对？**

相比阶段一，这已经不只是约束代码生成，而是在构建完整的反馈闭环。

---

## 三、阶段三：从 Harness 转向 Agent Interface

继续往下做之后，我逐渐意识到一个更上位的问题。

如果我们始终围绕某个具体目标建设工具：

> 为了自动修 Bug，做一个启动工具。

> 为了测试匹配，做一个客户端模拟器。

> 为了判断结果，做一个日志工具。

那么整个工具体系很容易变成：

```text
具体任务
↓
临时发现缺什么
↓
补一个工具
```

这样虽然也能不断提高 Agent 能力，但容易逐渐碎片化。

于是第三阶段的关注点开始发生变化：

> **不要先问“为了自动修 Bug 还缺哪个工具”，而应该先问“Agent 对这个项目究竟能观察什么、操作什么”。**

也就是把重点从某个具体自动化任务，提升到整个项目的 Agent Interface。

---

## 四、Agent 最基础的两种能力：Observation 与 Operation

如果希望 Agent 真正进入一个项目，它首先需要两类最基础的接口。

### 1. Observation：观察系统

Agent 需要能够知道真实系统里发生了什么。

例如：

```text
query_logs
query_db
query_metrics
trace_request
inspect_player
inspect_service
get_runtime_state
get_config
```

它应该可以回答：

- 当前有哪些服务正在运行？
- 玩家当前位于哪个 GS？
- 最近五分钟出现了哪些错误？
- 某个 request_id 经过了哪些服务？
- 当前数据库记录是什么状态？
- 某个配置真实加载的值是什么？
- 某个对象当前内存状态是什么？

如果 Agent 只能读代码，它看到的只是系统的静态世界。

而绝大多数真正困难的软件问题，都存在于运行时世界。

所以 Observation Interface 本质上是在给 Agent 增加“眼睛”。

---

### 2. Operation：操作系统

仅仅能够观察还不够。

研发过程需要主动制造输入和状态变化。

例如：

```text
start_server
stop_server
create_player
login_player
set_level
set_state
create_team
send_protocol
execute_lua
reload_config
```

这些能力让 Agent 可以主动制造：

- 登录场景；
- 匹配场景；
- 战斗场景；
- 重连场景；
- 异常状态；
- Bug 发生条件。

Operation Interface 本质上是在给 Agent 增加“手”。

当 Agent 同时拥有：

```text
Observation
+
Operation
```

它才真正能够和项目发生交互。

---

## 五、Harness 的定位也因此发生了变化

到了第三阶段以后，我对 Harness 的理解也发生了变化。

Harness 依然非常重要，但它已经不再是最终目标。

例如：

```text
Server Harness
    ↓
Runtime Control

Client Simulator
    ↓
Operation Interface

Log Reader
    ↓
Observation Interface
```

也就是说：

> **Harness 是实现 Agent Capability 的基础设施，而不是 Agent 研发体系本身。**

这个区别非常重要。

如果目标只是“做一套 Harness”，很容易陷入不断增加工具。

但如果目标是提高项目的 Agentability，那么每增加一个工具时，都可以问：

> 它到底补齐了 Agent 的哪一种能力？

> 它是否让 Agent 的研发闭环向前推进了一步？

这样建设方向就会清晰很多。

---

## 六、下一层：Orchestration Layer

当项目逐渐拥有大量 Observation 和 Operation 工具之后，会出现新的问题：

> **Agent 应该怎样组合这些工具？**

例如调试一个线上 Bug，可能需要：

```text
搜索代码
↓
查询线上日志
↓
根据日志找到 player_id
↓
查询玩家状态
↓
建立测试环境
↓
创建对应测试玩家
↓
设置状态
↓
启动服务
↓
模拟客户端登录
↓
发送协议
↓
读取日志
↓
查询数据库
↓
判断结果
```

如果几十个步骤完全依赖 Agent 临场决定，会出现很多问题：

- 忘记关键步骤；
- 调用错误工具；
- 查询过量数据；
- 缺少明确停止条件；
- 不知道什么时候应该转向另一个假设；
- 不同 Agent 的执行路径差异很大；
- 验证标准不一致。

因此，在 Agent Capability 之上，还需要一层：

> **Orchestration Layer。**

它的职责不是替 Agent 写死所有行为，而是定义研发流程中的关键结构：

```text
先观察什么
↓
什么时候构造环境
↓
什么时候执行操作
↓
收集哪些证据
↓
什么时候判定成功
↓
什么时候停止或换方向
```

这一层开始把零散工具组合成稳定的研发流程。

---

## 七、再往上：Scenario Layer

仅有编排仍然偏底层。

真正成熟之后，Agent 不应该每一次都从：

```text
start_server
create_player
login_player
send_protocol
query_logs
inspect_state
```

开始自己拼流程。

更高层应该形成：

> **Scenario。**

例如：

```text
Scenario: 两名玩家跨服匹配
```

它内部可能包含：

```text
启动 1 个 GC + 2 个 GS
↓
创建两个玩家
↓
分别登录不同 GS
↓
设置匹配分
↓
进入匹配
↓
等待 MatchSuccess
↓
查询 match_id
↓
检查双方状态
↓
检查数据库
↓
检查错误日志
```

Agent 在高层只需要表达：

```text
run_scenario("cross_server_match")
```

或者在已有 Scenario 上做参数变化。

这时候整个体系已经从：

> **Tool Driven**

逐渐向：

> **Scenario Driven**

发展。

Agent 不再只是会调用工具，而是可以调用一个完整的研发场景。

---

## 八、最终形成的分层

经过这三个阶段后，我认为一个比较完整的 Agent 研发基础设施，大概可以形成这样的分层：

```text
┌─────────────────────────────┐
│       Development Task      │
│    Bug / Feature / Change   │
├─────────────────────────────┤
│       Scenario Layer        │
│ Match / Login / Battle ...  │
├─────────────────────────────┤
│    Orchestration Layer      │
│ 流程 / 状态机 / 验证策略     │
├─────────────────────────────┤
│ Agent Capability Interface  │
│ Observe / Operate / Verify  │
├─────────────────────────────┤
│          Harness            │
│ Server / Client / Log / DB  │
├─────────────────────────────┤
│        Real Project         │
└─────────────────────────────┘
```

这几层之间不是互相替代，而是逐渐向上抽象。

Harness 解决的是：

> 怎么接触真实系统。

Agent Interface 解决的是：

> Agent 能获得哪些稳定能力。

Orchestration 解决的是：

> 这些能力应该怎样组合。

Scenario 解决的是：

> 如何把组合后的流程提升成可复用的研发语义。

最终，最上层的 Bug、Feature、Debug 等真实研发任务，才有可能稳定运行在这套基础设施之上。

---

## 九、三个阶段真正变化的是什么

如果回头看，这三个阶段可以总结成三个完全不同的问题。

### 阶段一

> **怎么让 Agent 按正确的方法写代码？**

关注点是：

```text
Spec
Constraint
Test
Coding
```

Agent 仍然主要被看成代码生成者。

---

### 阶段二

> **怎么让 Agent 自己运行和验证代码？**

关注点变成：

```text
Harness
Server
Client Simulator
Feedback Loop
```

Agent 开始成为研发执行者。

---

### 阶段三

> **怎么让整个项目成为 Agent 可以自主工作的环境？**

关注点进一步变成：

```text
Observation
Operation
Verification
Orchestration
Scenario
```

这个阶段已经不再围绕某一种开发范式，而是在建设：

> **Agent 研发基础设施。**

---

## 十、真正的变化：从“AI Coding 方法”到“AI 研发基础设施”

我认为这几次思路变化里，最重要的不是某个具体工具发生了变化，而是问题的层级变了。

最开始的问题是：

> 怎样用好 Codex？

后来变成：

> 怎样让 Codex 能自动修 Bug？

再后来变成：

> **一个软件项目怎样才能允许 Agent 真正进入研发流程？**

这是完全不同的问题。

前者关注的是：

```text
Agent
```

后者关注的是：

```text
Project × Agent
```

Agent 再强，如果项目完全封闭，它也只能：

```text
读代码
↓
改代码
↓
猜测结果
```

而一个 Agentability 很高的项目，可以让 Agent：

```text
理解问题
↓
修改代码
↓
启动系统
↓
制造场景
↓
观察状态
↓
验证结果
↓
继续迭代
```

这才是 AI Agent 真正进入软件研发的基础。

---

## 十一、可能的下一阶段：Evaluation

当 Observation、Operation、Harness、Orchestration 和 Scenario 都逐渐建立之后，下一个问题很自然会出现：

> **我们怎么证明这些东西真的让 Agent 更强了？**

不能只靠主观感觉：

> “现在这个工具挺好用。”

更好的方式是准备一套固定的研发任务，例如：

```text
10 个真实 Bug
5 个小型 Feature
若干 Debug Task
若干 Regression Scenario
```

然后重复运行：

```text
Infrastructure A
vs
Infrastructure B

Orchestration A
vs
Orchestration B

Agent A
vs
Agent B
```

测量：

```text
任务成功率
完成时间
人工介入次数
工具调用次数
错误修改率
复现成功率
验证覆盖率
```

这样 Agent Infrastructure 才能形成自己的反馈闭环：

```text
建设工具
↓
运行评测
↓
发现瓶颈
↓
改进基础设施
↓
重新评测
```

最终，我们不仅可以说：

> “项目现在更适合 Agent 了。”

而是能够回答：

> **它具体在哪些研发任务上提高了多少。**

---

# 结语

回顾整个思考过程，我的关注点经历了这样一次变化：

```text
Spec + TDD
↓
约束 Agent 的开发行为

Harness
↓
给 Agent 建立运行与验证闭环

Agent Interface
↓
让 Agent 能观察和操作整个项目

Orchestration
↓
组织 Agent 的研发流程

Scenario
↓
把工具能力抽象成研发场景

Evaluation
↓
持续衡量这套体系是否真正有效
```

这不是几套互相竞争的方案。

更像是对同一个问题不断向下追问之后，逐渐形成的一套分层。

最终真正值得建设的，并不是某一个“自动修 Bug 工具”，也不是某一个 Coding Agent 的最佳使用方法。

而是一套能够让 Agent：

```text
Understand
↓
Observe
↓
Operate
↓
Reproduce
↓
Verify
↓
Iterate
```

的软件研发环境。

当这样的环境存在以后，自动修 Bug、自动开发、自动 Debug、自动验收，都只是运行在它之上的不同场景。

**AI Coding 的下一步，可能不只是继续增强 Agent，而是开始系统性地改造项目本身，让项目真正成为一个 Agent 可以工作的地方。**
