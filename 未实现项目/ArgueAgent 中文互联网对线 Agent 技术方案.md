# ArgueAgent 中文互联网对线 Agent 技术方案

## 1. 项目定位

ArgueAgent 是一个面向中文互联网争论场景的 Argumentation Agent。

它的核心能力不是“自动骂人”，而是：

> 读取帖子、评论和对话上下文，识别对方的核心主张、论据、逻辑漏洞和交流意图，自主选择反驳、追问、事实核查、讽刺或结束交流策略，最终生成自然、简短、有针对性的中文互联网回复。

第一阶段采用 Human-in-the-loop：

```text
用户复制评论 / 截图转文本
        ↓
Agent 分析
        ↓
生成 3 个候选回复
        ↓
用户选择 / 修改 / 放弃
        ↓
记录反馈
```

MVP 不自动登陆微博、小红书等平台发言。

这样一方面避免平台接口和风控问题，另一方面用户的“选择哪个回复”正好成为后续训练所需要的 preference data。

---

## 2. 整体技术架构

```text
                    ┌──────────────────────┐
                    │      Web / App       │
                    │ 评论 + 上下文 + 模式 │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Spring Boot API    │
                    │ Session / User / DB  │
                    └──────────┬───────────┘
                               │
                               ▼
                 ┌────────────────────────────┐
                 │   Debate Orchestrator      │
                 │    AgentScope Java 2.0     │
                 └─────────────┬──────────────┘
                               │
                ┌──────────────┼───────────────┐
                ▼              ▼               ▼
       ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
       │Argument      │ │Context / Fact│ │Target Guard  │
       │Analyzer      │ │Analyzer      │ │              │
       └──────┬───────┘ └──────────────┘ └──────┬───────┘
              │                                  │
              └────────────────┬─────────────────┘
                               ▼
                       ┌───────────────┐
                       │Strategy Router│
                       └───────┬───────┘
                               │
                               ▼
                     ┌──────────────────┐
                     │ Style Retriever  │
                     │ PostgreSQL       │
                     │ + pgvector       │
                     └────────┬─────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │ Reply Generator  │
                     │   生成 3~5 条    │
                     └────────┬─────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │ Critic / Ranker  │
                     └────────┬─────────┘
                              │
                              ▼
                    最终候选回复 + 分析
                              │
                              ▼
                     ┌──────────────────┐
                     │ Feedback System  │
                     │chosen/rejected   │
                     └────────┬─────────┘
                              │
                              ▼
                     Training Dataset
                              │
                     SFT / DPO / LoRA
```

Agent 主工程使用 Java。

模型训练、数据分析和离线 Evaluation 使用 Python。

不要为了“全 Java”硬用 Java 做模型微调。最终系统是：

```text
Java = Agent / Backend / Product

Python = Training / Dataset / Evaluation
```

这也是更合理的工程边界。

---

## 3. 技术栈

| 模块 | 技术 |
|---|---|
| Backend | Java 21 + Spring Boot |
| Agent Runtime | AgentScope Java 2.0 |
| LLM | Qwen / GPT / Claude 可插拔 |
| DB | PostgreSQL |
| Vector DB | pgvector |
| Cache | Redis，可后加 |
| ORM | MyBatis / MyBatis-Plus |
| 数据迁移 | Flyway |
| 前端 | React / Next.js，或 MVP Thymeleaf |
| Observability | OpenTelemetry + Langfuse |
| Training | Python + PyTorch |
| Fine-tuning | LLaMA-Factory / PEFT |
| Model | Qwen 系列 |
| Deployment | Docker Compose → Kubernetes |

MVP 的向量库不建议再引入 Pinecone、Milvus 或 Elasticsearch。

直接：

```text
PostgreSQL
+
pgvector
+
PostgreSQL Full Text Search
```

就够。

---

## 4. Agent 状态模型

不要简单写成：

```text
User → LLM → Reply
```

核心应该是一个显式状态流。

```text
RECEIVED
    ↓
CONTEXT_PARSED
    ↓
ARGUMENT_ANALYZED
    ↓
TARGET_CHECKED
    ↓
STRATEGY_SELECTED
    ↓
EXAMPLES_RETRIEVED
    ↓
REPLIES_GENERATED
    ↓
REPLIES_CRITIQUED
    ↓
RANKED
    ↓
WAITING_USER
    ↓
FEEDBACK_RECORDED
```

每个节点都应该产生结构化状态。

例如：

```json
{
  "claim": "现在年轻人找不到工作主要因为眼高手低",
  "evidence": [
    "我身边很多人都是这样"
  ],
  "implicitPremises": [
    "身边人的情况能够代表整个年轻群体"
  ],
  "fallacies": [
    "ANECDOTAL_EVIDENCE",
    "HASTY_GENERALIZATION"
  ],
  "responseQuality": 0.32,
  "engagementValue": 0.58
}
```

这样后续 Strategy Router 不需要重新让模型理解整个问题。

---

## 5. Argument Analyzer

这是系统的“脑”。

输入：

```text
原帖
评论
上下文
```

输出固定 Schema：

```json
{
  "mainClaim": "",
  "subClaims": [],
  "evidence": [],
  "assumptions": [],
  "fallacies": [],
  "emotion": "",
  "intent": "",
  "responsiveness": 0.0,
  "evidenceQuality": 0.0,
  "logicQuality": 0.0,
  "discussionValue": 0.0
}
```

Fallacy 初期控制在大约 15 类，而不是搞一百多个学术分类。

例如：

```text
HASTY_GENERALIZATION
ANECDOTAL_EVIDENCE
STRAW_MAN
AD_HOMINEM
FALSE_DILEMMA
MOVING_GOALPOST
BURDEN_SHIFTING
APPEAL_TO_EMOTION
CIRCULAR_REASONING
WHATABOUTISM
CHERRY_PICKING
FALSE_CAUSALITY
AUTHORITY_ABUSE
TOPIC_EVASION
SEMANTIC_SHIFT
```

真正重要的是：

> 不只判断“是什么谬误”，还必须给出“攻击点”。

例如：

```json
{
  "fallacy": "HASTY_GENERALIZATION",
  "attackPoint": "样本代表性不足",
  "weakPremise": "身边几个案例可以代表整个群体"
}
```

这是后续生成优秀回复的关键。

---

## 6. Strategy Router

Strategy Router 是这个项目区别于普通聊天机器人的核心。

不要直接 Prompt：

```text
帮我怼他。
```

应该变成：

```text
Argument Analysis
        ↓
Strategy Selection
        ↓
Reply
```

第一版可以支持这些策略：

| Strategy | 使用场景 | 示例 |
|---|---|---|
| DIRECT_LOGIC | 明显逻辑错误 | 直接指出漏洞 |
| SOCRATIC | 对方前提薄弱 | 连续追问 |
| MIRROR | 对方逻辑可反向套用 | “照你这个逻辑……” |
| REDUCTIO | 前提推到荒谬结果 | 归谬 |
| CONCEPT_DEFLATION | 滥用宏大概念 | 把概念祛魅 |
| FACT_CHECK | 事实型争论 | 数据/出处 |
| COLD_SARCASM | 轻度讽刺适合 | 一句话点破 |
| EXPOSE_EVASION | 回避问题 | 指出没有回答 |
| ONE_LINER | 对方内容价值很低 | 一句话结束 |
| EXIT | 无讨论价值 | 不回复 |

例如：

```text
对方：
“我身边的人都是这样，所以现在年轻人就是这样。”

Analyzer：
HASTY_GENERALIZATION

Attack Point：
样本代表性

Router：
MIRROR + COLD_SARCASM
```

此时 Generator 才开始生成语言。

---

## 7. Style Engine

这个模块解决最大的产品问题：

> “逻辑是对的，但一眼 AI。”

核心思想：

```text
Reasoning 和 Style 分离
```

Reasoning Agent 决定：

```json
{
  "target": "sample_size",
  "strategy": "mirror",
  "tone": "sarcastic",
  "intensity": 2,
  "length": "short"
}
```

Style Engine 再去语料库检索。

---

## 8. 中文互联网语料库

不是简单保存：

```text
典
急了
那咋了
绝绝子
```

而应该保存：

```json
{
  "id": 1827,
  "text": "你朋友圈什么时候升级成人口普查数据库了？",
  "strategy": "MIRROR",
  "attackPoint": "sample_size",
  "fallacy": "HASTY_GENERALIZATION",
  "tone": "sarcastic",
  "intensity": 2,
  "platformStyle": "xiaohongshu",
  "length": "short",
  "qualityScore": 0.91,
  "embedding": []
}
```

向量检索 Query 不只是原评论。

应该构造：

```text
fallacy = HASTY_GENERALIZATION
attack_point = sample_size
strategy = MIRROR
tone = sarcastic
length = short
```

然后进行混合召回。

建议：

```text
FinalScore
=
0.50 × VectorSimilarity
+
0.20 × StrategyMatch
+
0.15 × FallacyMatch
+
0.10 × ToneMatch
+
0.05 × QualityScore
```

取 Top 10。

再由模型挑 Top 3～5 作为 few-shot examples。

非常重要：

> 语料库用于“学习表达方式”，不是把原句复制出来。

Generator Prompt 明确禁止大段复刻检索文本。

---

## 9. Reply Generator

Generator 一次不要生成一个答案。

直接生成：

```text
3～5 candidate replies
```

例如：

```json
[
  {
    "reply": "...",
    "strategy": "MIRROR",
    "tone": "sarcastic"
  },
  {
    "reply": "...",
    "strategy": "SOCRATIC",
    "tone": "cold"
  },
  {
    "reply": "...",
    "strategy": "ONE_LINER",
    "tone": "deadpan"
  }
]
```

这样有两个优势。

第一，用户体验好。

第二：

> 用户选哪个答案，本身就是 preference label。

---

## 10. Critic / Ranker

Generator 不负责评价自己。

单独一个 Critic。

评价维度：

| 指标 | 权重 |
|---|---:|
| 是否准确攻击逻辑漏洞 | 25% |
| 是否回应上下文 | 20% |
| 是否自然 | 20% |
| 是否简洁 | 10% |
| 是否有新意 | 10% |
| 是否避免无意义人格攻击 | 10% |
| 是否存在事实风险 | 5% |

输出：

```json
{
  "logicHit": 0.94,
  "naturalness": 0.86,
  "conciseness": 0.93,
  "contextFit": 0.89,
  "personalAttackRisk": 0.08,
  "overall": 0.90
}
```

最后 Ranker：

```text
candidate A  0.90
candidate B  0.84
candidate C  0.81
```

默认展示第一名，但让用户看到三个候选。

---

## 11. Target Guard

这不是单纯为了“安全”。

它同时提高回复质量。

系统需要区分：

```text
攻击论点
vs
攻击身份
```

允许：

```text
“你这个结论没有论据。”
“你朋友圈不能代表总体。”
“你这个概念用得太宽了。”
```

不让系统主动把：

```text
学历
性别
外貌
地域
疾病
家庭
种族
```

当作攻击素材。

因为这类回复实际上也降低 Agent 的“高级感”。

设计成 Tool / Middleware：

```text
TargetGuard
```

如果 Generator 跑偏：

```text
REGENERATE
```

而不是直接输出。

---

## 12. Discussion Value

这是我认为特别适合做成项目亮点的模块。

Agent 并不一定回复。

计算：

```text
Discussion Value
```

维度：

```text
是否提出观点
是否提供论据
是否回应问题
是否新增信息
是否重复
是否纯情绪
是否持续转移话题
```

例如：

```json
{
  "discussionValue": 0.12,
  "reason": [
    "连续回避核心问题",
    "没有新增论据",
    "出现重复性情绪表达"
  ],
  "decision": "EXIT"
}
```

此时输出：

```text
建议：别回了。

预计继续交流的信息增量非常低。
```

这会让 Agent 很不像普通“怼人生成器”。

---

## 13. 数据库设计

核心表建议：

```text
conversation
message
argument_analysis
reply_candidate
feedback
corpus_example
training_sample
experiment
```

其中最重要的是 `feedback`。

```sql
feedback
--------
id
conversation_id
message_id
candidate_id

action
-- CHOSEN
-- REJECTED
-- EDITED
-- COPIED

natural_score
logic_score
funny_score
aggression_score

reject_reason
created_at
```

如果用户修改生成结果：

```text
generated_reply
↓
user_edited_reply
```

这个数据价值甚至比简单点赞更大。

因为它告诉系统：

> AI 写成这样，而用户希望写成这样。

这就是天然的 SFT 数据。

---

## 14. 数据飞轮

最终产品壁垒应该是：

```text
用户输入
↓
Agent 生成 A/B/C
↓
用户选择 B
↓
记录：

chosen = B

A rejected
C rejected
↓
Preference Dataset
↓
重新训练
↓
新 Model Version
↓
A/B Test
↓
继续产生反馈
```

也就是：

```text
Traffic
 → Feedback
 → Dataset
 → Fine-tuning
 → Better Replies
 → More Traffic
```

Agent 本身不是 moat。

**数据才是。**

---

## 15. Evaluation

不能只凭：

> “我感觉这句挺爽。”

建立固定 Benchmark，例如 500～1000 条评论。

每条人工标：

```text
claim
fallacy
attack_point
recommended_strategy
good_reply
bad_reply
```

然后测试每个模型版本。

核心指标：

| Metric | 含义 |
|---|---|
| Fallacy Accuracy | 谬误识别 |
| Attack Point Accuracy | 有没有打到真正漏洞 |
| Strategy Accuracy | 策略是否合理 |
| Naturalness | AI 味 |
| Pairwise Win Rate | 两回复人类更喜欢哪个 |
| Edit Distance | 用户修改程度 |
| Pick Rate | 用户选择率 |
| Regeneration Rate | 用户要求重写比例 |
| Exit Accuracy | 是否知道不该继续吵 |

真正最重要的线上指标不是 BLEU 或 ROUGE。

而是：

```text
Candidate Pick Rate

User Edit Rate

Pairwise Win Rate
```

---

## 16. Observability

每次 Agent 请求建立完整 Trace：

```text
Debate Trace

├── argument_analyzer
│   ├── model
│   ├── tokens
│   └── latency
│
├── strategy_router
│
├── corpus_retriever
│   ├── retrieved_examples
│   └── similarity
│
├── reply_generator
│
├── critic
│
└── final_output
```

同时记录：

```text
model
prompt_version
corpus_version
strategy
latency
input_tokens
output_tokens
cost
feedback
```

Java 侧可以直接使用 OpenTelemetry，再把 OTLP traces 发给 Langfuse。

---

## 17. AgentScope 设计

第一版甚至不用复杂 Multi-Agent。

推荐：

```text
DebateAgent
    │
    ├── ArgumentAnalyzerTool
    ├── StrategyRouterTool
    ├── CorpusSearchTool
    ├── ReplyGeneratorTool
    ├── CriticTool
    └── TargetGuardTool
```

也就是：

```text
Single Agent
+
Deterministic Pipeline
```

不要为了“Agent 味”强行搞六个 Agent 对话。

后续再把真正独立的模块拆成 SubAgent。

---

## 18. Prompt Versioning

所有 Prompt 必须版本化。

例如：

```text
argument-analyzer-v1.3
strategy-router-v1.1
reply-generator-v2.4
critic-v1.7
```

数据库记录：

```text
model_version
prompt_version
corpus_version
```

否则后面：

> “为什么这个月效果变差？”

根本无法回答。

---

## 19. 模型策略

MVP 不训练。

可以：

```text
Analyzer
→ 强模型

Router
→ 便宜模型 / 规则

Generator
→ 强模型

Critic
→ 强模型
```

等跑起来再优化：

```text
Analyzer → 小模型
Router → Rule / Small LLM
Generator → Fine-tuned Model
Critic → Judge Model
```

最终可能形成：

```text
ArgueModel
```

只负责回复生成。

而 Argument Analyzer 仍然使用更强的通用模型。

不要试图训练一个模型承担全部职责。

---

## 20. Fine-tuning 路线

### Stage 0

不微调。

```text
Prompt
+
Few-shot
+
RAG
```

目标：

证明这个产品有人想用。

### Stage 1

积累约：

```text
1,000～5,000
```

条人工确认的优质样本。

数据：

```json
{
  "context": "...",
  "analysis": "...",
  "strategy": "MIRROR",
  "response": "..."
}
```

做 SFT。

### Stage 2

积累：

```text
chosen
vs
rejected
```

Preference Data。

例如：

```json
{
  "prompt": "...",
  "chosen": "你朋友圈什么时候升级成人口普查数据库了？",
  "rejected": "你的观点存在以偏概全的问题。"
}
```

进行 DPO。

### Stage 3

形成：

```text
ArgueModel-v1
```

Benchmark：

```text
General LLM
vs

Prompted LLM
vs

Prompt + RAG
vs

SFT
vs

SFT + DPO
vs

SFT + DPO + RAG
```

看真实用户 Pairwise Win Rate。

---

## 21. 微调技术方案

模型建议从 Qwen 系列的小中型模型开始。

训练：

```text
LLaMA-Factory
+
LoRA / QLoRA
```

初期完全没必要 Full Fine-tuning。

直接：

```text
QLoRA
```

把成本降下来。

---

## 22. Repo 结构

```text
argue-agent/
│
├── argue-server/
│   ├── controller/
│   ├── service/
│   ├── agent/
│   │   ├── DebateAgent.java
│   │   ├── state/
│   │   └── prompt/
│   │
│   ├── analyzer/
│   ├── strategy/
│   ├── generator/
│   ├── critic/
│   ├── guard/
│   ├── retrieval/
│   ├── feedback/
│   └── repository/
│
├── argue-web/
│
├── argue-eval/
│   ├── benchmark/
│   ├── evaluator/
│   └── reports/
│
├── argue-training/
│   ├── dataset/
│   ├── preprocess/
│   ├── sft/
│   ├── dpo/
│   └── evaluate/
│
├── corpus/
│   ├── seeds/
│   └── schema/
│
├── docker/
│
└── docs/
```

Java 和 Python 完全解耦。

---

## 23. MVP 范围

第一版只做：

```text
输入：
评论 + 上下文

↓

Argument Analysis

↓

Strategy Router

↓

100～300 条人工优质 Style Examples

↓

生成 3 个回复

↓

Critic Ranking

↓

用户选择

↓

Feedback DB
```

不做：

```text
模型训练
自动微博回复
自动小红书回复
复杂 Multi-Agent
Kafka
Kubernetes
独立向量数据库
大规模爬虫
```

这些 MVP 都不需要。

---

## 24. 第二阶段

当 MVP 可以用了以后加入：

```text
1,000～10,000 条语料

pgvector Style RAG

Conversation Memory

Langfuse

Prompt Experiment

Offline Benchmark

Feedback Dashboard
```

此时项目已经足够成为一个完整 GitHub Agent 项目。

---

## 25. 第三阶段

做数据飞轮：

```text
A/B/C Candidates
↓
chosen / rejected
↓
user edited version
↓
自动 Dataset Builder
↓
Quality Filter
↓
Training Dataset Version
```

例如：

```text
dataset-v1
dataset-v2
dataset-v3
```

每个版本：

```text
来源
数量
质量
类别比例
策略比例
```

全部可追溯。

---

## 26. 第四阶段

真正开始 Post-training：

```text
Qwen
↓
QLoRA SFT
↓
ArgueModel-SFT
↓
DPO
↓
ArgueModel-v1
↓
Offline Eval
↓
Shadow Traffic
↓
A/B Test
↓
Production
```

这时候项目核心卖点就从：

> “我做了一个 Agent。”

升级为：

> “我设计并实现了一套由用户反馈驱动的数据飞轮，对中文互联网 Argument Agent 进行持续 Evaluation、SFT 和 Preference Optimization。”

这个项目层级就完全不同了。

---

## 27. 最终系统形态

成熟状态：

```text
                ArgueAgent
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
      Reasoning   Memory     Fact Tool
         │
         ▼
     Strategy Engine
         │
         ▼
      Style RAG
         │
         ▼
     ArgueModel
         │
         ▼
        Judge
         │
         ▼
    Human Feedback
         │
         ▼
      Dataset
         │
         ▼
 SFT / DPO / Evaluation
         │
         └──────────────→ ArgueModel vNext
```

---

## 28. 这个项目真正值得讲的三个技术点

如果以后面试问：

> “你这个项目有什么技术含量？”

最值得讲的不是“我调用了 Qwen”。

而是三个东西。

### 第一，Reasoning / Style 解耦

不是让模型直接生成，而是先建立 Argument Representation，再做 Strategy Planning，最后做 Style Realization。

### 第二，Evaluation-driven Agent

每个模块可独立 Benchmark：

```text
Analyzer
Router
Retriever
Generator
Critic
```

可以知道问题到底发生在哪里，而不是：

> “模型今天效果不好。”

### 第三，Feedback Data Flywheel

用户每次选择、修改、拒绝回复，都转换为：

```text
SFT data
Preference data
Evaluation data
```

模型不断迭代。

这才是整个项目真正的长期壁垒。

---

## 29. 最适合的实现顺序

```text
Phase 1
ArgumentAnalyzer
+
StrategyRouter
+
ReplyGenerator

↓

Phase 2
Critic
+
3 Candidate Ranking

↓

Phase 3
PostgreSQL + Feedback

↓

Phase 4
Corpus + pgvector

↓

Phase 5
Langfuse + Benchmark

↓

Phase 6
Dataset Builder

↓

Phase 7
QLoRA SFT

↓

Phase 8
DPO

↓

Phase 9
A/B Test
```

前五阶段主要依赖 Java / Agent / RAG / 可观测性能力。

真正需要重点补的是：

```text
Phase 6～8
Dataset Engineering
SFT
LoRA / QLoRA
DPO
```

项目推进原则：先验证“逻辑攻击点识别 + Strategy Router + Style RAG”是否真的比一句 Prompt 更好，再用真实用户选择数据推进后训练，避免为了微调而微调。
