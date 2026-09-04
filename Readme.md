# 动态智能研判系统架构设计

## 1. 项目定位

本系统面向需要持续跟踪和动态研判的问题，采用：

> **初始主动搜索论证 + 新信息增量更新**

的工作模式。

系统并非简单地接收问题后直接生成答案，而是首先围绕问题建立：

> **问题 → 假设 → 证据 → 推理 → 结论**

的初始研判状态。

在此基础上，当后续出现新的新闻、数据、舆情、公告或其他信息时，将新信息作为增量输入，对已有研判状态进行更新，而不是重新从零开始分析。

---

## 2. 核心思想

系统采用**假设驱动 + 证据驱动 + 状态持续更新**的模式。

### 初始阶段

```text
用户问题
   ↓
问题分析
   ↓
生成候选假设
   ↓
制定证据需求
   ↓
主动搜索
   ↓
证据评估
   ↓
假设修订
   ↓
判断是否需要继续搜索
   ↓
最终结论
```

### 持续更新阶段

```text
已有研判状态
      +
新信息
      ↓
事件相关性判断
      ↓
去重
      ↓
新增事实提取
      ↓
证据评估
      ↓
更新假设
      ↓
判断结论是否变化
      ↓
更新研判状态
```

因此，系统不是：

```text
Question → Answer → END
```

而是：

```text
Question
   ↓
Research State
   ↑
   │
New Information
   │
   └──── Continuous Update
```

---

# 3. 总体架构

```text
                         ┌──────────────────┐
                         │    Research API  │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
              初始问题输入                  新信息输入
                    │                           │
                    ↓                           ↓
          ┌──────────────────┐       ┌──────────────────┐
          │ Initial Research │       │ Incremental      │
          │ Engine           │       │ Update Engine    │
          └────────┬─────────┘       └────────┬─────────┘
                   │                          │
                   └────────────┬─────────────┘
                                ↓
                 ┌─────────────────────────────┐
                 │       Research State        │
                 │                             │
                 │ Question                    │
                 │ Hypotheses                  │
                 │ Facts                       │
                 │ Evidence                    │
                 │ Timeline                    │
                 │ Conclusion                  │
                 │ Confidence                  │
                 └──────────────┬──────────────┘
                                │
                                ↓
                       ┌─────────────────┐
                       │   Event Store   │
                       └─────────────────┘
```

---

# 4. 初始研判流程

初始研判是系统的核心主动推理流程。

```text
Question
   ↓
Question Analyzer
   ↓
Hypothesis Generator
   ↓
Evidence Planner
   ↓
Search / Query
   ↓
Evidence Evaluator
   ↓
Hypothesis Updater
   ↓
Continue?
   ├── YES → Evidence Planner
   │
   └── NO
        ↓
Conclusion Generator
        ↓
Research State
```

## 4.1 问题分析

对用户问题进行结构化解析：

* 问题主体
* 核心对象
* 时间范围
* 地域范围
* 需要解释的问题
* 已知事实
* 未知信息
* 潜在影响因素

输出：

```json
{
  "question": "某公司股价为什么突然上涨？",
  "objective": "分析股价上涨的主要原因",
  "known_facts": [],
  "unknowns": [
    "业绩是否改善",
    "是否存在政策利好",
    "是否存在重大事件",
    "是否存在资金集中流入"
  ]
}
```

---

# 5. 假设生成

系统不会直接搜索并根据搜索结果生成答案，而是首先提出多个可能的解释。

例如：

```text
H1：公司业绩改善导致股价上涨
H2：行业政策利好导致股价上涨
H3：重大事件导致市场预期改变
H4：资金集中流入导致股价上涨
H5：市场情绪或题材炒作导致上涨
```

每个假设需要维护：

```json
{
  "id": "H1",
  "hypothesis": "公司业绩改善导致股价上涨",
  "prior_probability": 0.25,
  "current_probability": 0.25,
  "status": "UNVERIFIED",
  "evidence_ids": [],
  "missing_evidence": []
}
```

---

# 6. 证据规划

针对不同假设，确定需要寻找什么证据。

例如：

```text
H1：业绩改善
    ↓
需要：
- 最新财报
- 业绩预告
- 盈利预测
- 分析师预期

H2：政策利好
    ↓
需要：
- 政策文件
- 官方公告
- 行业政策
- 政策发布时间

H3：重大事件
    ↓
需要：
- 公司公告
- 新闻报道
- 行业动态
```

这样搜索不是无目标搜索，而是：

> **针对假设寻找能够支持或反驳假设的证据。**

---

# 7. 信息搜索层

系统通过统一 Tool Layer 获取外部信息。

```text
                 Search Interface
                        │
       ┌────────────────┼────────────────┐
       ↓                ↓                ↓
    Web Search        ES Search        Database
       │                │                │
       ↓                ↓                ↓
    新闻网页          舆情数据          业务数据
```

可接入：

* Web Search
* Elasticsearch
* 新闻搜索
* 内部数据库
* 文档检索
* API
* 历史事件库
* 舆情数据库

Agent 不直接依赖具体搜索实现，而通过统一 Tool Interface 调用。

---

# 8. 证据评估

搜索结果不能直接作为证据。

系统需要对信息进行评估：

```text
Search Result
     ↓
相关性
     ↓
可信度
     ↓
时效性
     ↓
信息类型
     ↓
支持 / 反驳 / 中性
```

证据结构：

```json
{
  "id": "E001",
  "source": "某公司公告",
  "content": "公司发布半年报，净利润同比增长35%",
  "relevance": 0.95,
  "credibility": 0.98,
  "supports": ["H1"],
  "contradicts": [],
  "published_at": "2026-09-03"
}
```

---

# 9. 假设更新

新证据进入后，对相关假设进行增量更新。

例如：

```text
初始：

H1 业绩改善       30%
H2 政策利好       30%
H3 资金炒作       20%
H4 其他            20%
```

发现公司半年报：

```text
净利润同比增长35%
```

更新：

```text
H1 业绩改善       30% → 65%
H2 政策利好       30% → 20%
H3 资金炒作       20% → 10%
H4 其他            20% → 5%
```

随后发现政策文件：

```text
H1 业绩改善       65% → 62%
H2 政策利好       20% → 35%
H3 资金炒作       10% → 8%
H4 其他             5% → 2%
```

这里的数值可以根据实际系统设计采用概率模型、评分模型或其他证据更新算法。

核心不是具体公式，而是：

> **新证据应当改变已有状态，而不是覆盖已有状态。**

---

# 10. 持续更新机制

初始研判完成后，系统进入 Monitoring 状态。

```text
                 Research State
                       │
                       │
                ┌──────┴──────┐
                │             │
             新信息          用户追问
                │             │
                ↓             ↓
          Incremental     Re-Reasoning
             Update
                │
                ↓
         更新 Evidence
                ↓
         更新 Hypothesis
                ↓
        更新 Conclusion
                ↓
          新 Research State
```

新信息可能来自：

* 新闻
* 官方公告
* 政府通报
* 企业公告
* 社交媒体
* Elasticsearch
* 数据库
* 用户主动输入
* 外部 API

---

# 11. 新信息处理流程

新信息进入后不应该直接触发完整重跑。

首先进行：

```text
New Information
      ↓
事件相关性判断
      ↓
是否重复？
 ├── YES → 丢弃
 └── NO
      ↓
是否产生新事实？
 ├── NO → 记录但不更新
 └── YES
      ↓
关联已有假设
      ↓
证据评估
      ↓
更新 State
```

例如：

```text
新信息：
“教育部门发布调查通报，确认学校此前未及时处理相关投诉”
```

系统判断：

```text
属于当前事件：YES
重复信息：NO
产生新事实：YES

影响：
H1 学校管理问题 → 强支持
H2 学生个人冲突 → 削弱
```

最终更新：

```text
H1：70% → 88%
H2：55% → 35%
```

如果结论发生显著变化，则触发：

```text
Conclusion Update
        ↓
Notification
```

---

# 12. Research State

`Research State` 是整个系统最核心的数据对象。

建议至少包含：

```python
class ResearchState:

    event_id: str

    # 原始问题
    question: str

    # 问题分析
    problem_definition: dict

    # 当前假设
    hypotheses: list

    # 已确认事实
    facts: list

    # 全部证据
    evidences: list

    # 信息时间线
    timeline: list

    # 搜索历史
    search_history: list

    # 推理历史
    reasoning_history: list

    # 当前结论
    conclusion: str

    # 当前置信度
    confidence: float

    # 状态版本
    version: int

    # 最后更新时间
    updated_at: datetime

    # 监控状态
    monitoring_status: str
```

---

# 13. 时间线

系统应保存事件随时间的发展过程。

```text
T1
事件首次出现
    ↓
建立 H1/H2/H3

T2
官方通报
    ↓
H1 ↑

T3
当事人回应
    ↓
H2 ↑

T4
调查结果
    ↓
H1 ↑↑
H2 ↓

T5
最终结论
```

这样可以实现：

> **不仅知道当前结论，还知道结论为什么发生变化。**

---

# 14. 推理历史

每次状态变化都应该留下记录。

```json
{
  "version": 8,
  "trigger": "new_information",
  "information_id": "E021",
  "changes": [
    {
      "hypothesis": "H1",
      "old_score": 0.72,
      "new_score": 0.88,
      "reason": "官方调查结果直接支持该假设"
    }
  ],
  "conclusion_changed": true,
  "timestamp": "2026-09-04T09:20:00"
}
```

因此系统具备完整的：

> **Evidence → Reasoning → State Change**

链路。

---

# 15. Agent 与 Workflow 的关系

不建议设计成多个 Agent 自由对话。

推荐：

```text
                 LangGraph
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Node        Node        Node
        │           │           │
        ↓           ↓           ↓
      LLM         LLM         Tool
```

核心节点：

```text
Question Analyzer
Hypothesis Generator
Evidence Planner
Evidence Evaluator
Hypothesis Updater
Conclusion Generator
```

其中部分节点可以使用同一个 LLM。

Agent 负责：

> **推理**

LangGraph 负责：

> **状态、流程、循环、分支、恢复**

Tools 负责：

> **获取外部信息**

State 负责：

> **保存长期研判状态**

---

# 16. 推荐技术架构

```text
┌─────────────────────────────────────────────┐
│                   API Layer                 │
│              FastAPI / REST API             │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│              Research Orchestrator          │
│                  LangGraph                  │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│                Reasoning Layer              │
│                                             │
│ Question │ Hypothesis │ Evidence │ Update   │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│                  Tool Layer                 │
│                                             │
│ Web │ ES │ DB │ News │ Documents │ API     │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│                 State Layer                 │
│                                             │
│ Event Store │ Evidence Store │ Timeline     │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│              Monitoring Layer               │
│                                             │
│ Scheduler │ Message Queue │ Event Trigger   │
└─────────────────────────────────────────────┘
```

---

# 17. 推荐项目目录

```text
research-agent/
│
├── app/
│   ├── main.py
│   │
│   ├── graph/
│   │   ├── workflow.py
│   │   ├── state.py
│   │   └── router.py
│   │
│   ├── agents/
│   │   ├── question_analyzer.py
│   │   ├── hypothesis_generator.py
│   │   ├── evidence_planner.py
│   │   ├── evidence_evaluator.py
│   │   ├── hypothesis_updater.py
│   │   └── conclusion_generator.py
│   │
│   ├── tools/
│   │   ├── web_search.py
│   │   ├── es_search.py
│   │   ├── database.py
│   │   ├── news_search.py
│   │   └── document_search.py
│   │
│   ├── models/
│   │   ├── llm.py
│   │   └── embeddings.py
│   │
│   ├── prompts/
│   │   ├── question.txt
│   │   ├── hypothesis.txt
│   │   ├── evidence.txt
│   │   ├── updater.txt
│   │   └── conclusion.txt
│   │
│   ├── storage/
│   │   ├── event_store.py
│   │   ├── evidence_store.py
│   │   └── state_store.py
│   │
│   └── monitoring/
│       ├── scheduler.py
│       ├── listener.py
│       └── dispatcher.py
│
├── tests/
├── scripts/
├── requirements.txt
└── README.md
```

---

# 18. 两条核心执行路径

系统最终只需要重点维护两条路径。

### 初始研判

```text
Question
 → Analyze
 → Hypotheses
 → Evidence Plan
 → Search
 → Evaluate
 → Update
 → Search Again
 → Conclusion
 → Save State
```

### 增量更新

```text
New Information
 → Match Event
 → Deduplicate
 → Extract Facts
 → Evaluate Evidence
 → Update Hypotheses
 → Update Conclusion
 → Save State
 → Notify
```

二者最终汇聚到同一个：

```text
Research State
```

---

# 19. 系统核心原则

### 原则一：先假设，后搜索

不能：

```text
搜索结果
 ↓
模型看到什么就解释什么
```

而应该：

```text
问题
 ↓
候选假设
 ↓
针对假设寻找证据
```

---

### 原则二：新信息增量更新

不能每来一条信息：

```text
重新搜索
重新分析
重新生成全部结果
```

而应该：

```text
Existing State
 +
New Evidence
 ↓
Incremental Update
```

---

### 原则三：保留历史状态

不能只保存最终结论。

必须保存：

```text
问题
 ↓
假设
 ↓
证据
 ↓
推理
 ↓
状态变化
 ↓
当前结论
```

---

### 原则四：结论必须可追溯

任何结论都应该能够回答：

> **为什么得到这个结论？**

并能够反向追溯：

```text
Conclusion
    ↓
Hypothesis
    ↓
Evidence
    ↓
Source
```

---

### 原则五：只有有效新信息才触发状态变化

普通重复信息：

```text
New Information
     ↓
Duplicate
     ↓
Ignore
```

高价值新信息：

```text
New Information
     ↓
New Fact
     ↓
Evidence Update
     ↓
Hypothesis Update
     ↓
Conclusion Update
```

---

# 20. 最终系统抽象

整个系统可以最终抽象成：

```text
              ┌───────────────┐
              │    Question   │
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │  Hypotheses   │
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │   Evidence    │
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │   Reasoning   │
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │ Research State│
              └───────┬───────┘
                      ↑
                      │
                New Information
                      │
                      └──────────────
```

核心闭环：

> **Question → Hypothesis → Evidence → Reasoning → State → New Information → Update**

因此，该系统本质上不是一个“一问一答”的 Agent，而是一个**以 Research State 为核心、支持持续证据更新的动态智能研判系统**。
