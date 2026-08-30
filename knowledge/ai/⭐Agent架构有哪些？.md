# ⭐ Agent架构有哪些？

### 重要程度
⭐

### 面试回答
Agent 架构没有唯一标准，核心是根据任务复杂度、是否需要动态工具调用、是否要先规划、是否需要多角色协作，以及业务流程是否固定来选型。常见可以分为单 Agent、ReAct、Plan-and-Execute、Multi-Agent、Router + Skill、Blackboard 和 Graph Workflow。

### 1. 单 Agent

最简单的一问一答：

```text
User
 ↓
LLM
 ↓
Answer
```

适合简单问答、总结、改写、轻量判断。

### 2. ReAct

边执行边根据结果决定下一步：

```text
User
 ↓
LLM 判断下一步
 ↓
Tool Call
 ↓
Tool Result
 ↓
LLM 再判断
 ↓
...
 ↓
Final Answer
```

特点：灵活，适合开放式工具调用任务；缺点是长链路中容易重复调用或失控。

### 3. Plan-and-Execute

先规划完整步骤，再逐步执行：

```text
User
 ↓
Planner
 ↓
Plan
1. 查数据
2. 清洗
3. 画图
4. 上传 OSS
 ↓
Executor
```

和 ReAct 的核心区别：

```text
ReAct = 走一步想一步
Plan-and-Execute = 先想整体，再一步步执行
```

更适合复杂、长链路、有依赖关系的任务。

### 4. Multi-Agent

多个 Agent 各自负责不同角色：

```text
            Router
             ↓
    ┌────────┼────────┐
    ↓        ↓        ↓
 数据Agent  分析Agent  报告Agent
```

适合角色差异大、上下文需要隔离、可以并行的场景。

不应该为了“看起来高级”而强上 Multi-Agent；很多任务单个 ReAct Agent + Tools 就够了。

### 5. Router + Skill

外层 Router 先判断任务类型，再选择不同执行方式或 Skill：

```text
User
 ↓
Router
 ↓
┌────────┬──────────────┬──────────┐
↓        ↓              ↓
ReAct   Plan&Execute   固定 Skill
```

例如：

```text
查一个库存
→ ReAct

分析半年数据并生成报告
→ Plan-and-Execute

生成标准采购报告
→ 固定 Skill
```

Skill 适合封装稳定的业务规则、Prompt、工具调用约束和流程经验。

### 6. Blackboard 黑板架构

多个 Agent 通过一个共享状态区协作：

```text
       Shared Blackboard
       /      |       \
      /       |        \
 Agent A   Agent B   Agent C
```

大家都可以把中间发现、待办、结论写到黑板上，再读取别人写入的状态继续工作。

适合异步、多 Agent 长时间协作和共享中间结论的场景。

### 7. Graph Workflow

把执行过程显式拆成节点和条件边：

```text
START
 ↓
Intent
 ↓
Query MCP
 ↓
Validate
 ↓
Generate Script
 ↓
Sandbox
 ↓
Upload OSS
 ↓
END
```

还可以加入条件分支：

```text
Sandbox success?
├─ Yes → OSS
└─ No  → Retry / Fix
```

优点：可控、可观测、容易做状态恢复；缺点：流程经常变化时 Graph 会越来越复杂。

### 怎么选？

可以先记这个简化版：

```text
简单任务
→ Single Agent

开放式工具任务
→ ReAct

复杂长任务
→ Plan-and-Execute

流程固定、强可控
→ Graph Workflow
```

`Router + Skill` 更像外层调度；`Multi-Agent / Blackboard` 只有在角色隔离、并行协作真的有价值时再引入。

### 最终心智模型

不是架构越复杂越好，而是：

> **任务越复杂、依赖越多、越需要可控和恢复，就越需要显式规划、路由或 Graph；能用一个 Agent 解决，就不要先上 Multi-Agent。**

### 一句话速记

`Single → ReAct → Plan-and-Execute → Graph` 是复杂度逐步提升的主线，Router/Skill 负责调度和复用，Multi-Agent/Blackboard 负责真正的多角色协作。
