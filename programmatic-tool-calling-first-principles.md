---
layout: article
title: "为什么 Programmatic Tool Calling 会成为 Agent 的重要方向"
description: "分析 Agent 调用工具的需求，以及程序化工具调用带来的能力变化。"
category: Agent Engineering
weight: 3
---

# 为什么 Programmatic Tool Calling 会成为 Agent 的重要方向

当 AI Agent 开始从“会回答问题”走向“真正执行任务”之后，Tool Calling 很快成为了核心能力。

模型不再只是生成文本，而是可以：

- 查询数据库
- 搜索日志
- 调用 API
- 操作文件
- 控制服务
- 调用 MCP
- 与真实软件系统交互

这一步解决了一个非常重要的问题：

> **LLM 能不能使用外部能力？**

但当 Agent 开始拥有越来越多工具之后，另一个问题很快出现：

> **这些工具应该如何被组合？**

传统的 Agent Loop 通常类似：

```text
LLM 判断
   ↓
Tool A
   ↓
LLM 判断
   ↓
Tool B
   ↓
LLM 判断
   ↓
Tool C
   ↓
LLM 判断
```

这种模式可以工作。

但如果从第一性原理重新分析 Agent 的计算过程，会发现其中存在一个非常明显的问题：

> **大量根本不需要“智能”的工作，也被交给了 LLM。**

这也是为什么我认为 Programmatic Tool Calling（PTC）很可能会成为未来 Agent Harness 的重要基础能力。

它真正解决的，不只是“减少几次 Tool Call”。

它改变的是：

> **LLM、程序与工具之间应该如何分工。**

---

# 一、先从第一性原理看：Agent 到底在做哪些计算

一个 Agent 执行复杂任务时，大致会面对三种完全不同性质的问题。

## 1. 语义问题

例如：

```text
这个 Bug 最可能是什么原因？

我下一步应该检查哪里？

这些证据之间有什么关系？

现在的信息是否足够支持这个结论？
```

这些问题的特点是：

- 信息不完整
- 存在不确定性
- 需要理解上下文
- 需要根据语义进行推理

这是 LLM 擅长的领域。

可以简单理解为：

```text
LLM = Meaning
```

也就是负责：

> **理解、判断、假设。**

## 2. 过程问题

例如：

```text
查询 100 个玩家

过滤等级 > 50

分别查询 Match

并发查询日志

筛选 ERROR

按照 error_code 分组

取出现次数最多的前三个
```

这里真正需要“智能”的地方其实很少。

大量操作只是：

```text
if
for
while
filter
map
reduce
sort
join
retry
timeout
parallel
```

这些是标准的程序计算问题。

因此：

```text
Program = Composition
```

程序最擅长的是：

> **组合、控制流程、处理中间数据。**

## 3. 环境能力问题

最后还有一类问题：

```text
能不能查询玩家？

能不能启动服务器？

能不能发送协议？

能不能搜索日志？

能不能读取 DB？

能不能修改测试状态？
```

这些不是推理问题，也不是流程问题。

而是：

> **真实系统到底暴露了哪些能力。**

所以：

```text
Tool = Capability
```

Tool 决定：

> **Agent 能够对真实世界做什么。**

---

# 二、一个更自然的 Agent 架构

如果把三种计算拆开，就会得到一个非常自然的结构：

```text
               Task
                ↓
        Semantic Reasoning
               LLM
                ↓
          Generate Program
                ↓
      Deterministic Runtime
              Program
                ↓
       Atomic Capabilities
              Tools
                ↓
            Real System
                ↓
             Evidence
                ↓
               LLM
                ↓
        Semantic Judgment
```

可以进一步压缩成一句：

```text
LLM      = Meaning
Program  = Composition
Tool     = Capability
```

我认为这是理解 Programmatic Tool Calling 最重要的三个概念。

---

# 三、传统 Tool Calling 的根本问题：计算资源错配

传统 Agent Loop 最大的问题，并不是它不能完成任务。

而是它把很多不同性质的计算，都交给了同一个组件：

> LLM。

例如：

```text
查询玩家
↓
LLM

查询 Match
↓
LLM

读取日志
↓
LLM

过滤日志
↓
LLM

查询数据库
↓
LLM

比较结果
↓
LLM
```

其中真正需要 LLM 的可能只有：

```text
为什么我要查这些信息？
```

以及：

```text
这些信息说明了什么？
```

但：

```text
循环
过滤
聚合
排序
并发
重试
超时
数据转换
```

全部继续经过 LLM。

这是一种明显的计算资源错配。

可以把它总结成：

```text
语义推理 → LLM      合理
流程执行 → LLM      不合理
数据处理 → LLM      不合理
循环控制 → LLM      不合理
并发调度 → LLM      不合理
```

Programmatic Tool Calling 实际上是在重新划分这条边界。

---

# 四、PTC 的核心：让 LLM 写程序，而不是亲自执行每一步

传统模式：

```text
Reason
 ↓
Tool
 ↓
Reason
 ↓
Tool
 ↓
Reason
 ↓
Tool
```

Programmatic Tool Calling 之后：

```text
Reason
  ↓
生成临时程序
  ↓
┌──────────────────────┐
│ Tool A               │
│   ↓                  │
│ Tool B               │
│   ↓                  │
│ if (...)             │
│   Tool C             │
│ else                 │
│   Tool D             │
│                      │
│ for (...)            │
│   Tool E             │
└──────────────────────┘
  ↓
结构化结果
  ↓
Reason
```

这里出现了一个非常重要的变化：

> **LLM 不再直接承担执行器，而是开始承担程序生成器。**

真正执行大量确定性操作的是程序 runtime。

这意味着 Agent 的基本执行单位发生了变化：

过去：

```text
Tool Call
```

未来可能越来越多地变成：

```text
Generated Program
```

---

# 五、第一个第一性原因：组合爆炸无法靠增加 Tool 解决

假设一个项目拥有下面这些能力：

```text
get_player
get_match
query_logs
query_db
inspect_server
send_protocol
```

刚开始看起来已经够用了。

但随着需求增加，很容易继续出现：

```text
diagnose_player
diagnose_match
diagnose_match_timeout
diagnose_cross_server_match
diagnose_match_disconnect
verify_match_fix
batch_diagnose_match_failure
```

为什么 Tool 会越来越多？

因为现实任务是基础能力的组合。

而：

> **任务组合数量远远大于基础能力数量。**

如果每出现一种新的组合需求，就把它封装成新的 Tool：

```text
任务组合增加
        ↓
高级 Tool 增加
        ↓
Tool 数量膨胀
        ↓
Agent 选择难度增加
        ↓
维护成本增加
```

最终就会产生 Tool Explosion。

---

# 六、真正可扩展的方法：提供组合能力本身

软件工程早就遇到过类似的问题。

计算机不会为每一种计算任务提供一个 CPU 指令。

我们提供的是：

```text
有限 Instruction Set
        +
Programming Language
```

程序员通过组合有限的基础指令，实现近乎无限的任务。

Agent Tool 也一样。

与其：

```text
不断增加高级 Tool
```

不如：

```text
少量 Atomic Tool
        +
Programming Language
```

例如：

```text
get_player
get_match
query_logs
query_db
```

然后 Agent 根据当前任务生成：

```typescript
const player = await tools.get_player({ id });

const match = await tools.get_match({
    match_id: player.match_id
});

const logs = await tools.query_logs({
    player_id: id
});

const errors = logs.filter(
    x => x.level === "ERROR"
);

return {
    player,
    match,
    errors
};
```

明天面对完全不同的任务，可以重新生成完全不同的组合程序。

因此：

> **PTC 本质上是在 Agent Tool 世界里补上了一层“编程语言”。**

---

# 七、它会直接缓解 Tool / Workflow 膨胀

传统模式很容易变成：

```text
Atomic Tool
↓
高级 Tool
↓
更高级 Tool

Workflow A
Workflow B
Workflow C
Workflow D
...
```

因为所有组合都必须提前由人定义。

而 Programmatic Tool Calling 允许：

```text
Atomic Tools
      +
Generated Program
```

于是很多原本需要永久存在的 Workflow，可以变成：

> **一次性临时程序。**

例如：

> 调查昨天晚上这 37 个玩家为什么匹配失败。

这是一个明显的长尾任务。

没有必要永久增加：

```text
investigate_37_players_last_night_workflow
```

Agent 可以根据任务现场生成代码。

任务结束，程序可以直接丢弃。

所以更准确地说：

> **PTC 并不是消灭 Workflow。**

而是：

> **PTC 消灭那些原本不值得成为 Workflow 的 Workflow。**

---

# 八、第二个第一性原因：Context 是稀缺资源

Agent 还有另外一个非常重要的资源：

> **模型上下文和注意力。**

传统 Tool Calling 经常发生：

```text
Tool
↓
100KB Result
↓
LLM Context

Tool
↓
200KB Result
↓
LLM Context

Tool
↓
50KB Result
↓
LLM Context
```

大量机器数据不断被转换成 Token。

然后模型再负责：

```text
过滤
统计
排序
寻找某些字段
```

这是非常低效的。

例如：

> 从十万条日志里找到 ERROR，按照 player_id 分组，然后返回错误最多的 10 个玩家。

LLM 根本没有必要看到十万条日志。

程序可以先执行：

```typescript
const logs = await tools.query_logs(...);

const errors = logs.filter(
    x => x.level === "ERROR"
);

const grouped = groupBy(
    errors,
    x => x.player_id
);

return top10(grouped);
```

LLM 最后只需要看到：

```text
player 1001: 73 errors
player 2031: 51 errors
player 8271: 43 errors
```

于是数据流从：

```text
Raw Data
   ↓
LLM
```

变成：

```text
Raw Data
   ↓
Program
   ↓
Filter / Aggregate / Compress
   ↓
Relevant Evidence
   ↓
LLM
```

这实际上是在做一件非常基础的事情：

> **不要让智能系统的注意力消耗在不需要智能的信息上。**

---

# 九、第三个第一性原因：确定性的东西应该下沉

LLM 本质上是概率系统。

如果一个 Agent 要执行：

```text
检查状态

如果没有完成，等待 500ms

再次检查

最多执行 20 次

出现 ERROR 立即退出

最终无论成功失败都 cleanup
```

传统 Agent Loop 可能不断重新让模型决定：

```text
现在应该继续吗？

这是第几次？

要不要退出？

是不是应该 cleanup？
```

这些事情不应该反复交给概率模型。

程序表达：

```typescript
try {
    for (let i = 0; i < 20; i++) {
        const state = await getState();

        if (state.done) {
            return state;
        }

        await sleep(500);
    }

    throw new Error("timeout");
}
finally {
    await cleanup();
}
```

显然更加稳定。

因此可以形成一个非常重要的架构原则：

> **已经能够确定性表达的东西，就应该从概率模型下沉到确定性 runtime。**

也就是：

```text
不知道该做什么
        ↓
       LLM

已经知道怎么做
        ↓
     Program
```

---

# 十、第四个原因：Control Plane 与 Data Plane 应该分离

从系统架构角度，未来 Agent 很可能越来越像经典的 Control Plane / Data Plane 架构。

```text
             LLM
        Control Plane
             ↓
      判断 / 计划 / 生成程序
             ↓
──────────────────────────
             ↓
      Program Runtime
         Data Plane
             ↓
           Tools
             ↓
        Real System
```

LLM 不需要参与：

- 每一次循环
- 每一个数据项
- 每一次轮询
- 每一个 API 调用
- 每一个中间转换

它只需要在：

> **出现新的语义不确定性时重新介入。**

因此，未来更合理的 Agent Loop 可能不是：

```text
LLM
↓
Tool
↓
LLM
↓
Tool
↓
LLM
```

也不是：

```text
LLM
↓
生成一个巨大程序
↓
完全跑到底
```

而更可能是：

```text
Reason
  ↓
Program
  ↓
Checkpoint
  ↓
Reason
  ↓
Program
  ↓
Checkpoint
```

也就是：

> **语义决策低频发生，确定性执行高频发生。**

---

# 十一、第五个原因：Agent 的角色从 Tool Selector 变成 Program Synthesizer

传统 Tool Calling Agent 的核心问题经常是：

```text
下一步调用哪个 Tool？
```

它更像一个：

> **Tool Selector。**

而 Programmatic Tool Calling 之后，问题提升成：

```text
为了完成这个目标，

应该构造一个怎样的执行程序？
```

Agent 开始成为：

> **Program Synthesizer。**

这两个层级差别非常大。

可以类比人类程序员。

一个人如果只能：

```text
手工打开工具
点击按钮
执行命令
查看结果
```

他的工作效率是有限的。

程序员真正获得巨大杠杆，是因为他可以：

> **把操作编码成程序。**

Agent 也一样。

所以可以把两种模式类比成：

```text
Tool Calling Agent
≈ 人工操作工具

Programmatic Agent
≈ 写脚本控制工具
```

因此我认为：

> **PTC 是 Agent 从“工具使用者”向“计算机使用者”演化的重要一步。**

---

# 十二、PTC 对项目 Agent Interface 的意义

如果未来软件项目开始主动给 Agent 暴露接口：

```text
start_server
stop_server
create_player
login_player
send_protocol
query_logs
query_db
inspect_state
get_config
trace_request
```

那么很快一定会遇到：

> 这么多接口应该怎么组合？

一种思路是建立非常厚重的：

```text
Workflow Engine
Orchestrator
State Machine
DSL
```

但 Programmatic Tool Calling 提供了另一条路径：

```text
Project Agent Interface
        ↓
保持 Atomic
        ↓
暴露为 SDK
        ↓
Agent 根据任务生成 Program
        ↓
动态组合
```

因此，项目 Agent Interface 的 Tool 设计反而可以更加：

```text
Small
Stable
Orthogonal
Composable
```

也就是：

> **小、稳定、正交、可组合。**

而不需要为了减少 Tool Call 次数，不断构建越来越大的高级 Tool。

---

# 十三、一个更合理的三层沉淀模型

Programmatic Tool Calling 出现以后，我认为 Agent Workflow 可以采用一个非常自然的三层模型。

## 第一层：Atomic Capability

长期稳定存在。

例如：

```text
start_server
create_player
send_protocol
query_logs
inspect_state
query_db
```

这是项目的基础能力。

## 第二层：Generated Program

根据当前任务即时生成。

例如：

```text
调查某个玩家为什么匹配失败
```

Agent 临时生成程序：

```text
查询玩家
↓
查询 Match
↓
查询日志
↓
对比 DB
↓
生成结构化证据
```

任务结束后，这段程序不一定需要长期保留。

## 第三层：Scenario / Workflow

如果某一个 Generated Program：

```text
反复出现
业务语义稳定
执行要求稳定
需要进入 Regression
需要版本管理
```

再把它正式沉淀。

例如：

```text
cross_server_match
player_reconnect
dungeon_complete
```

因此形成：

```text
Atomic Capability
       ↓
Generated Program
       ↓
反复验证有价值
       ↓
Promote
       ↓
Scenario / Workflow
```

可以把这个原则概括为：

> **先动态组合，后稳定沉淀。**

这与软件开发中的抽象原则非常类似：

> 不要因为出现一次重复就立即设计框架，而是等稳定模式真正出现后再抽象。

---

# 十四、PTC 不会替代 Workflow 和 Scenario

Programmatic Tool Calling 并不是所有场景的答案。

一些流程必须保持：

```text
固定
可审核
可版本控制
可预测
可追责
```

例如：

```text
生产发布
数据迁移
支付操作
线上危险写操作
固定 Regression
关键验收流程
```

这些场景通常不应该让 Agent 每次现场重新生成执行逻辑。

它们更适合：

```text
Stable Workflow
+
Permission
+
Audit
+
Version Control
```

因此一个合理的边界是：

> **稳定、危险、重复的过程固化。**

而：

> **探索性、长尾、组合性的过程动态程序化。**

---

# 十五、最终的 Agent 架构可能是什么样

如果把前面的逻辑合在一起，一个比较自然的未来 Agent 架构可能是：

```text
                  Task
                   ↓
                  LLM
         理解 / 判断 / 假设
                   ↓
            Generate Program
                   ↓
      Programmatic Orchestration
                   ↓
           Atomic Tools
                   ↓
       Project Agent Interface
                   ↓
             Real System
                   ↓
              Evidence
                   ↓
         必要时重新进入 LLM
```

旁边再存在：

```text
Stable Scenario Library
```

专门保存那些：

- 已经稳定
- 经常重复
- 有明确业务语义
- 需要进入长期回归

的流程。

---

# 十六、从第一性原理重新定义三者关系

最终，可以把未来 Agent 的三个核心组成压缩成：

```text
LLM
=
处理语义不确定性

Program
=
处理过程确定性

Tool
=
提供环境能力
```

或者：

```text
LLM      = Meaning
Program  = Composition
Tool     = Capability
```

这三个部分分别解决三个完全不同的问题。

而 Programmatic Tool Calling 的意义，就是终于把中间缺失的：

> **Composition Layer**

补了出来。

---

# 结语：从 Tool Calling 到 Programmable Agent

Function Calling 解决的是：

> **LLM 能不能调用外部世界。**

Programmatic Tool Calling 进一步解决的是：

> **LLM 能不能把外部世界变成一个可编程环境。**

这两者看起来只差了一层“代码”。

但实际上，它改变了 Agent 的整个计算模型。

从：

```text
LLM
↓
Tool
↓
LLM
↓
Tool
↓
LLM
```

逐渐变成：

```text
LLM
↓
生成程序
↓
程序组合多个 Tool
↓
处理大量中间数据
↓
返回关键证据
↓
LLM
```

这意味着：

- LLM 更专注于语义推理
- 程序负责确定性执行
- Tool 保持原子和稳定
- Workflow 不再因为长尾任务无限膨胀
- Context 不需要承载大量无意义中间数据
- Agent 可以从 Tool Selector 向 Program Synthesizer 演化

因此，我认为 Programmatic Tool Calling 不是一个单纯的 Tool Calling 优化。

它更像是 Agent 架构里迟早应该出现的一层基础抽象：

> **Tool 提供能力，Program 提供组合，LLM 提供判断。**

当这一层真正成熟以后，今天这种：

```text
LLM → Tool → LLM → Tool → LLM → Tool
```

的 Agent Loop，可能会越来越像今天的人类手工重复执行几十条 Shell 命令：

不是不能工作。

而是：

> **明显缺了一层本该存在的程序化抽象。**
