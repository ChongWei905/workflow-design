# 面向非结构化文档的Workflow能力设计

# **1. 背景与目标**

## **1.1 背景**

当前部门按能力拆分为 **知识工程 / 本体 / Agent** 三个方向：

- **知识工程 组**：负责从非结构化文档（PDF / Word / HTML 等）中，通过既有 pipeline 提取信息并生成回答；
- **本体 组**：负责统一领域概念、实体、属性与关系，提供跨文档、跨流程的一致语义基础；
- **Agent 组**：负责基于 system prompt + tools 的方式进行决策、规划与执行。

现有 RAG 能力的核心形态是：

**“给定文档 + 查询 → 生成一个回答（Answer）”**

该模式在“信息查询 / 总结 / 问答”场景中表现良好，但在以下场景中逐渐暴露出明显不足：

- 需要**多步骤执行**（而非一次性回答）的业务流程；
- 需要**显式校验 / 审计 / 回放**的严苛场景（银行、工业、合规）；
- 需要 Agent **基于流程执行结果继续决策**的复杂任务；
- 需要从文档中沉淀 **可复用、可版本化的流程资产**。

换言之，现有系统能够“回答问题”，但无法“执行流程”。



## **1.2 引入Workflow能力的动机**

随着 Agent 能力的引入，系统开始具备：

- 调用工具（tool）
- 组合多个步骤
- 在执行过程中进行条件判断与分支

但如果缺乏统一的 **Workflow 抽象与执行体系**，将会出现以下问题：

- 流程逻辑散落在 agent prompt 或 tool 代码中，**不可治理**；
- 相同业务流程在不同 agent 中重复实现，**不可复用**；
- 流程执行缺乏统一的校验、审计与安全边界，**不可控**；
- 文档中蕴含的大量 SOP / 规章 / 操作说明 **无法资产化**。

因此，我们需要在现有 RAG 能力之上，引入一种新的核心能力：

**将非结构化文档中描述的流程，抽取、规范化并执行为可治理的 Workflow。**



## **1.3 目标（Goals）**

本次设计的目标是引入一套 **Workflow 能力体系**，并与现有 RAG / Ontology / Agent 架构形成清晰分工。

### **1.3.1 功能目标**

- 定义一套 **通用 Workflow 表达方式**，支持大多数常见业务流程：
  - 顺序执行
  - 条件分支
  - 并行/汇聚
  - 重试、超时、终止
  - 人工介入节点（Human-in-the-loop）
- 支持从非结构化文档中：
  - 抽取流程结构
  - 映射为标准 Workflow
  - 沉淀为可复用资产
- 提供 **Workflow 执行能力**，支持：
  - dry-run / execute
  - 可回放、可审计
  - 可中断、可恢复

### **1.3.2 架构目标**

- **Workflow 定义与执行**由 RAG 侧主导，而非嵌入 Agent prompt；
- Agent 通过 **稳定的 MCP Tool 接口** 调用 Workflow，而不直接感知流程细节；
- Workflow 能力可独立演进，不影响 Agent 规划逻辑；
- Workflow 能与 Ontology 体系对齐，实现跨文档、跨流程的一致语义。

### **1.3.3 非目标（Non-goals）**

本设计**不追求**以下内容：

- 不构建通用低代码平台或 BPMN 替代品；
- 不允许 Agent 直接“解释并执行自然语言流程文本”；
- 不试图在第一阶段覆盖所有极端复杂流程（如长事务分布式 Saga）。

# 2. 术语与边界

为避免后续设计与实现过程中的语义混乱，本章节对核心概念与职责边界进行统一定义。

## 2.1 核心术语定义

### **2.1.1 Workflow**

**Workflow** 是一个可执行的流程定义，表示为一个**有向无环图（DAG）**，由多个 Node 及其连接关系构成。

一个 Workflow 至少包含：

- 唯一标识：workflow_id
- 版本号：version
- 节点集合（Nodes）
- 节点之间的依赖关系
- 输入 / 输出定义
- 执行约束与策略（如风险等级、是否需要人工介入）

Workflow 是**一等资产**，必须支持版本化、审计与回放。

### **2.1.2 Node**

**Node** 是 Workflow 中的最小执行单元，表示一个原子步骤。

每个 Node 具备：

- 明确的 **Node Type**（如：检索、抽取、校验、调用外部系统、人工审批）
- 输入 Schema
- 输出 Schema
- 执行语义（副作用 / 是否幂等 / 是否可重试）
- 风险等级与执行限制

Node 本身不包含业务流程控制逻辑（如分支/并行），这些由 Workflow 层统一编排。

### **2.1.3 Workflow Definition**

Workflow Definition 是 Workflow 的**静态描述**，通常采用 JSON / DSL 形式，包含：

- Workflow 元信息（id / version / owner / tags）
- Node 定义与连线关系
- 输入输出 schema
- 执行策略与 guard 规则

Workflow Definition **不等于** Workflow Execution（运行时实例）。

### **2.1.4 Workflow Execution**

Workflow Execution 表示某一次具体运行，具备：

- execution_id
- 绑定的 workflow_id + version
- 实际输入
- 每个 Node 的运行状态
- 执行日志、证据、结果

Execution 是**运行时对象**，用于观测、审计与回放。

### **2.1.5 Workflow MCP Server**

Workflow MCP Server 是 **Workflow 能力的对外接口层**，负责：

- 按 MCP 协议暴露 Workflow 相关 tools；
- 接收 Agent 的调用请求；
- 驱动 Workflow Execution；
- 返回结构化执行结果。

MCP Server **不负责** Agent 的决策与规划。

## **2.2 各组职责边界**

![workflow插图整体架构](/Users/weichong/Documents/working_area/KnowledgeEngine/notes/workflow插图整体架构.jpg)

### **2.2.1 知识工程组（Workflow Owner）**

RAG 组是 **Workflow 能力的 Owner**，负责：

- Workflow DSL / Schema 设计；
- Node 类型定义与执行实现；
- 从文档中抽取、生成、校验 Workflow；
- Workflow Registry（存储、版本、生命周期）；
- Workflow 执行引擎与 MCP Server 实现；
- 执行安全、审计、回放能力。

### **2.2.2 Ontology 组（Semantic Backbone）**

Ontology 组提供：

- 统一的领域实体与属性定义；
- Workflow 输入/输出字段的语义对齐；
- 跨文档、跨 Workflow 的概念一致性；
- 校验与类型约束支持。

Ontology 不参与 Workflow 的控制逻辑，但影响其**可泛化性与可维护性**。

### **2.2.3 Agent 组（Orchestrator & Decision Maker）**

Agent 组负责：

- 理解用户目标；
- 决定是否调用 Workflow 能力；
- 选择合适的 workflow_id + version；
- 通过 MCP Tool 调用 Workflow；
- 基于执行结果进行后续决策或用户交互。

Agent **不负责** Workflow 的定义、执行细节与安全控制。



## **2.3 关键边界原则（Design Principles）**

- **Workflow 是服务端执行的受控程序，不是 prompt 中的文本**；
- **Agent 只调用 Workflow，不解释 Workflow**；
- **流程变化通过版本管理，而不是 prompt 变更**；
- **高风险动作必须由 Workflow 执行层拦截，而非 Agent 自律**。

# **3. 端到端实际场景示例：化工企业「危险品运输合规判定」**

![workflow插图示例流程](./workflow插图示例流程.jpg)

本章通过一个真实的业务场景，展示从非结构化文档到可执行 Workflow 的全链路转化过程。

## **3.1 输入材料：非结构化文档（多源）**

系统接收三份不同来源的文档作为知识底座：

1. **文档 A：《企业内部危险品运输管理制度》（PDF）**
   - **核心逻辑**：规定了危险品运输前必须确认类别；一类、二类危险品在公路运输时必须备案。
2. **文档 B：《危险化学品目录（国家标准）》（外部库/PDF）**
   - **核心内容**：包含化学品名称、CAS 编号、危险等级（1/2/3 类）。
3. **文档 C：《XX 任务运输说明》（Word/表单）**
   - **具体任务**：运输物品：硫酸；数量：5 吨；运输方式：公路；目的地：XX 工厂。

## **3.2 自然语言问题（触发点）**

用户输入查询：“这次硫酸运输是否符合公司和国家规定？如果不符合，缺少哪些条件？”

## **3.3 流程发现与 Node 抽取**

系统通过 RAG 侧的逻辑分析，从**文档 A** 中抽取出逻辑链路，并将其拆解为原子化的 **Node**：

- **Node 1: `identify_substance`**（识别物质）：判断物品是否在危险品目录中。
- **Node 2: `lookup_hazard_level`**（查询等级）：从标准库获取该物质的危险等级。
- **Node 3: `check_transport_mode`**（校验方式）：判断运输方式与等级是否匹配。
- **Node 4: `check_filing_status`**（核查备案）：针对公路运输核实业务系统中的备案记录。
- **Node 5: `compliance_summary`**（汇总结论）：聚合各步结果，生成判定。

## **3.4 Workflow 组装与人工校验**

当系统从《危险品运输管理制度》等非结构化文档中识别出逻辑意图后，并不会直接进入执行阶段，而是通过“**合成引擎（Synthesis Engine）**”生成一份可供人类审查的 Workflow 草案。该过程分为三个核心步骤：

### **3.4.1 候选原子节点召回（Candidate Node Recall）**

系统基于从文档中提取的语义标签（Semantic Tags），在全局 **Node Catalog** 中搜索匹配的执行单元。

- **语义匹配**：例如，文档 A 提到“核对危险等级”，系统自动召回具备 `ontology_query` 属性且关联 `hazard_level` 概念的候选节点。
- **硫酸案例应用**：在此阶段，系统检索并锁定以下工具：
  - `identify_substance`（用于获取 CAS 编号）
  - `lookup_hazard_level`（用于判定危险等级）
  - `check_transport_filing`（用于查询外部备案系统）

### **3.4.2 规则逻辑加载与约束解析（Rule & Constraint Mapping）**

系统从文档段落中提取**触发器（Trigger）\**与\**约束条件（Constraint）**，并将其映射为节点的输入/输出逻辑。

- **触发器加载**：识别流程启动的初始条件。在案例中，Trigger 被定义为“用户输入包含物质名称且存在运输意图”。
- **条件约束解析**：提取文档中的分支谓词。
  - *提取逻辑*：“若属于一类或二类危险品，须确认运输方式。”
  - *约束转化*：生成节点间edge的判定逻辑：当 hazard_level ∈ [1,2] 且 transport_mode=公路 时，**边**指向 filing 节点
- **数据依赖（Data Dependency）**：系统自动识别出 `check_filing` 节点必须依赖 `identify_substance` 产出的 `task_id` 和 `substance_cas`，从而确定执行顺序。

### **3.4.3 设计引子编排（Design Primer Synthesis）**

这是生成草案的最后阶段，系统利用 **Design Primer（编排引子）** 策略，将零散的节点和规则组装成符合 DAG（有向无环图）规范的结构描述。

- **拓扑排序**：基于数据依赖关系，将节点排列为：`识别` → `查询等级` → `条件分支` → `备案检查`。
- **缺省策略植入**：对于文档中未明确提及的异常路径（如：查询不到该化学品怎么办？），系统会植入标准的“**异常处理引子**”，通常表现为一个 `human_input` 节点，请求用户手动介入。
- **生成 DSL 预览**：最终合成一份 JSON 格式的 Workflow 描述文件，并通过前端 UI 转化为可视化流程图。

### **3.4.4 人工校验与语义修正（Human-in-the-loop）**

Workflow 此时处于“**Draft（草案）**”状态。业务专家可以对生成的引子进行微调：

- **案例修正**：用户发现系统自动生成的逻辑中，三类危险品也进入了备案核查分支。
- **修正操作**：用户直接在 UI 界面修改 `Condition` 约束，将备案触发条件收窄为“仅一、二类”。
- **状态固化**：用户点击“发布”后，该草案正式转化为可被 Agent 调用的受控工具（Managed Tool）。

## **3.5 Workflow 执行（结合本体库做Grounding）**

Workflow 进入执行引擎，通过 **Agent 调用 MCP Tool** 驱动：

- **执行 Step 1**：调用本体库（Ontology），将“硫酸”映射到 `CAS 7664-93-9`，确定其为危险品。
- **执行 Step 2**：从本体库检索该 CAS 对应的危险等级，得到 `Level 2`。
- **执行 Step 3**：逻辑判定 `Level 2` + `公路运输` = `需核查备案`。
- **执行 Step 4**：调用业务接口查询 `task_id`，返回 `has_filing = false`。

## **3.6 最终答案生成**

Agent 获取 Workflow 的结构化输出后，生成回复：

“本次硫酸运输**不合规**。原因是：硫酸属于二类危险品，公路运输需备案，但当前系统未查询到备案记录。建议完成备案后再行组织。”

# 4. Workflow DSL 与整体结构设计

## 4.1 设计参考与取舍

在设计 Workflow DSL 之前，我们对业界已有的 Workflow 表达与执行模型进行了系统性调研，重点参考了以下三类体系：

### **4.1.1 DAG 型工作流（Airflow / Argo）**

该类系统以 **DAG（有向无环图）** 作为流程的核心抽象：

- Node 表示原子任务
- Edge 表示依赖关系
- 执行语义清晰、可视化友好

**借鉴点：**

- DAG 是表达流程的最低共识模型；
- Node 与依赖关系分离，有利于静态分析与调度；
- 执行状态（pending/running/succeeded/failed）清晰可观测。

**不采纳点：**

- 以调度/批处理为中心的设计；
- 对条件分支、人工介入支持不足。

### **4.1.2 状态机型 DSL（AWS Step Functions）**

该类系统将 Workflow 建模为 **显式状态机（State Machine）**，以 JSON DSL 表达：

- 明确的 State 类型（Task / Choice / Parallel / Fail / Succeed）
- 显式控制结构（条件、并行、终止）
- 可由程序生成，适合自动化构建

**借鉴点（核心）：**

- Workflow ≠ Task 逻辑，而是控制结构的组合；
- 条件、并行、终止是一等能力；
- JSON DSL 非常适合从文档自动生成。

**不采纳点：**

- 对云服务调用的强绑定；
- 过于偏“服务编排”的 Node 语义。

### **4.1.3 业务流程建模（BPMN）与确定性执行（Temporal）**

BPMN 提供了极其完备的业务流程表达能力，Temporal 提供了工业级的执行语义与回放机制。

**借鉴点：**

- 人工任务（Human Task / Human Gate）是一等流程节点；
- 高风险步骤需要显式审批与中断能力；
- Workflow Definition 与 Execution 必须严格区分；
- 执行过程必须可回放、可审计。

**明确不采纳点：**

- BPMN 的完整规范与图形优先建模；
- Temporal 的代码级 DSL 与强运行时侵入。

## 4.2 设计目标

### 4.2.1 设计前提与总体约束

在本系统中，Workflow 的设计受到以下**硬约束**：

- Workflow 首先由 **系统自动生成**（文档抽取），辅助人工编写；
- Workflow 必须在 **执行前即可被完整理解、校验与治理**；
- Workflow 的执行过程必须 **确定、可回放、可审计**；
- 系统明确 **不追求最大流程表达能力**，而是追求工程闭环。

### **4.2.2 设计策略**

综合上述调研，本系统的 Workflow DSL 采用如下策略：

##### 以DAG作为结构基础：

由于agent调用mcp前需要明确确认当前流程是可执行的，且流程是有理有据的，因此要求流程可以进行静态校验、可以回放审计，且具备可解释性。

因此，不能选择允许循环/隐式跳转的表达，因为无法进行静态校验（获得入参并执行前确认流程可行）；不能允许执行行为依赖运行时状态，因为无法回放。

综上考虑，有向无环图是解决场景问题的最小正确解。

##### 以状态机思想表达控制结构：

状态机思想并非真实构建状态机，而是用状态机思想来约束Workflow的控制流，保证流程执行的每个节点都产生事件、更新状态和落盘。任意时刻的暂停或是pending都可以追溯到流程中的某一节点，并知悉其处于哪种状态。

```text
# 状态变化示例
validate → (true) → execute
validate → (false) → human_input
running → waiting_input → running
running → waiting_approval → running
```

##### 以 JSON DSL 作为表达形式：

选用JSON DSL的原因源自场景需求：**流程不是人写的，而是系统生成、系统校验、系统执行的**。

因此，代码DSL不可行，因为模型能力很难支持直接从文档抽取生成可执行可解释的代码；自然语言不可行，因为无法校验、无法编译、无法固定流程面向严肃行业场景；图形（BPMN）不可行，生成复杂、规范过重。

##### 显式支持人工接入与审计语义：

这一点同样源于场景需求，且能够产生产品的差异化。它对于模型和文档的不确定性进行了人为的补全，面向：

- 文档没写清楚的流程
- 模型无法确定的条件
- 高风险但业务必须继续的步骤

通过显式支持人工介入的方式，保证了流程的严谨性和回放/审计过程的可能性。

#### **4.2.3 设计目标**

综合上述分析，本系统的设计目标是以 DAG + 状态机思想 + JSON DSL 为核心设计策略，并将人工介入与审计作为一等语义，是为了在保证流程表达能力的同时，实现**可生成、可校验、可执行、可回放、可治理**的工程闭环，而非追求最大表达自由度。

## **4.3 Workflow 的整体抽象**

### **4.3.1 Workflow 的精确定义**

在第三章的示例中，系统面对的并不是一个“编排需求”，而是一个来自非结构化文档的**合规判定流程**：

- 是否属于危险化学品
- 危险等级是多少
- 当前运输方式是否允许
- 是否完成必要备案
- 汇总是否合规

这一流程具有以下显著特征：

- **步骤可枚举**：每一步在文档中都有对应语义；
- **依赖明确**：危险等级依赖物质识别结果；
- **条件有限**：分支条件来源于文档中的明确约束；
- **无隐式循环**：流程在有限步内必然终止；
- **可能需要人工介入**：文档规则存在歧义或缺失。

因此，在本系统中，**Workflow 是一个「声明式执行规范对象」**，它必须满足：

- 在执行前即可完全解析
- 在执行前即可回答：
  - 有哪些步骤
  - 哪些路径是合法的
  - 哪些路径需要人工
  - 是否可能自动完成

### **Workflow 的最小结构（必须存在）**

```
{
  "workflow_id": "hazard_transport_compliance",
  "version": "v1",
  "inputs": {
    "substance_name": { "type": "string" },
    "transport_mode": { "type": "string" }
  },
  "nodes": { ... },
  "edges": [ ... ],
  "execution_policy": { ... }
}
```

**其中：**

- inputs 明确了执行该流程**必须具备的外部事实**；
- nodes 定义了可被调度的原子能力；
- edges 穷举了所有合法的流程走向；
- execution_policy 则定义了流程的治理约束。

**明确禁止：**

- 在 Workflow 中嵌入任意可执行代码
- 在 Workflow 中嵌入 prompt 逻辑
- 在 Workflow 中描述“执行算法”



## **4.4 DSL 的基础结构（Concrete Skeleton）**

### **4.4.1 DSL 的角色**

DSL 是：

**Workflow 抽象在系统之间流转的唯一结构载体**

必须满足：

- LLM / pipeline 可生成
- 校验器可静态分析
- 引擎可确定性执行
- 审计系统可复原

### **4.4.2 DSL 的核心组成块（明确）**

```
{
  "metadata": {
    "risk_level": "medium",
    "description": "..."
  },
  "inputs": {
    "contract_text": { "type": "string", "required": true }
  },
  "nodes": {
    "n1": { ... },
    "n2": { ... }
  },
  "edges": [
    { "from": "n1", "to": "n2", "condition": "always" }
  ],
  "execution_policy": {
    "allow_auto": false
  }
}
```

**明确禁止**

- 运行时新增 node / edge
- edge 条件依赖不可解释的模型输出
- edge 直接引用外部状态

### 4.4.3 示例：危险品运输 Workflow DSL

```
{
  "metadata": {
    "description": "危险品运输合规判定",
    "risk_level": "high"
  },
  "inputs": {
    "substance_name": { "type": "string", "required": true },
    "transport_mode": { "type": "string", "required": true }
  },
  "nodes": {
    "identify": { "type": "identify_substance" },
    "level": { "type": "lookup_hazard_level" },
    "mode_check": { "type": "check_transport_mode" },
    "filing": { "type": "check_filing_status" },
    "summary": { "type": "compliance_summary" }
  },
  "edges": [
    { "from": "identify", "to": "level" },
    { "from": "level", "to": "mode_check" },
    {
      "from": "mode_check",
      "to": "filing",
      "condition": {
        "and": [
          { "in": ["$.outputs.level.hazard_level", [1, 2]] },
          { "eq": ["$.inputs.transport_mode", "公路"] }
        ]
      }
    },
    { "from": "mode_check", "to": "summary", "condition": "otherwise" },
    { "from": "filing", "to": "summary" }
  ]
}
```



## **4.5 Node 模型（抽象层 + 可执行边界）**

### **4.5.1 Node 的最小可执行定义**

```
Node {
  "type": "extract_amount",
  "inputs": {
    "contract_text": "$.inputs.contract_text"
  },
  "outputs": {
    "amount": { "type": "number" }
  }
}
```

Node 的职责 **仅限于**：

输入 → 确定性执行 → 输出

### **4.5.2 Node 不允许的能力（硬约束）**

Node 内**不允许**：

- if / else 决定流程
- retry / terminate 决策
- 调用人工
- 修改 workflow 结构

```
// ❌ 不允许的 Node（示意）
{
  "type": "review_contract",
  "logic": "if amount > 1M then ask human"
}
```

## **控制结构建模（状态机思想的具体落点）**

### **4.6.1 状态机思想 = 有限、显式、可枚举**

在 DSL 中，**流程控制只能通过结构出现**。

### **4.6.2 条件分支**

```
{
  "from": "risk_check",
  "to": "human_approval",
  "condition": {
    "eq": ["$.outputs.risk_check.level", "high"]
  }
},
{
  "from": "risk_check",
  "to": "auto_execute",
  "condition": {
    "eq": ["$.outputs.risk_check.level", "low"]
  }
}
```

所有分支在 DSL 中被**穷举**

### **4.6.3 人工介入**

```
"nodes": {
  "human_input_1": {
    "type": "human_input",
    "requested_fields": {
      "amount": { "type": "number" }
    }
  }
}
```

**人工 = 一个合法状态**

```
running → waiting_input → running
```

**明确禁止**

- Node 内触发人工
- Agent 临时决定“要不要人工”
- 执行引擎自动“猜测”是否需要暂停

## **4.7 输入 / 输出与数据流建模**

### **4.7.1 Workflow 输入**

```
"inputs": {
  "contract_text": {
    "type": "string",
    "required": true
  }
}
```

### **4.7.2 Node 输入映射**

```
"inputs": {
  "text": "$.inputs.contract_text"
}
```

或：

```
"inputs": {
  "amount": "$.outputs.extract_amount.amount"
}
```

**明确禁止**

- Node 隐式读取全局上下文
- Node 访问“上一步的变量”
- 动态创建输入字段



## **4.8 执行策略（声明式，而非执行算法）**

本节用于回答一个核心问题：

**“在什么前提下，一个 Workflow / Node 被系统允许执行？”**

而不是：

**“它具体是如何被执行的？”**

执行策略的设计目标，是将**风险、权限、人工介入、重试边界等治理性决策**

从运行时代码与 Agent 推理中剥离出来，

以前置、显式、可校验的方式纳入 Workflow DSL。

## **4.8.1 执行策略的角色定位**

在本系统中，存在三类不同性质的逻辑：

1. **业务逻辑**：某一步“做什么”（由 Node 承载）

2. **流程逻辑**：步骤之间“如何衔接”（由 Workflow 结构承载）

3. **治理逻辑**：

   > 在什么条件下允许执行？

   > 是否允许自动执行？

   > 是否必须人工介入？

   > 出现失败时系统能否自动重试？

**执行策略（Execution Policy）正是第三类逻辑的唯一承载位置。**

### **为什么治理逻辑不能放在别处？**

- 放在 Node 内：
  - 校验阶段不可见
  - 每个 Node 各自为政
- 放在 Agent prompt：
  - 不可审计
  - 不可强制
- 放在执行引擎代码：
  - 规则散落
  - 难以演进

因此，本系统要求：

**所有“是否允许 / 是否必须 / 是否禁止”的决策，必须以声明式策略存在于 DSL 中。**

## **4.8.2 Workflow 级执行策略**

### **4.8.2.1 定义与作用范围**

Workflow 级执行策略用于描述：

**整个 Workflow 作为一个整体，在系统治理层面具有什么约束与风险属性。**

示例：

```
"execution_policy": {
  "risk_level": "high",
  "allow_auto": false,
  "require_approval": true
}
```

### **4.8.2.2 各字段的“系统语义”（不是字面意思）**

#### **risk_level**

- 表示 **该 Workflow 在业务和合规层面的整体风险等级**
- 用于：
  - 校验阶段判断是否允许自动执行
  - 审计阶段解释为何引入人工

#### **allow_auto**

- 表示 **是否允许在无人工介入的情况下启动与推进 Workflow**
- 当 allow_auto=false 时：
  - Agent 不得直接触发执行
  - 执行引擎必须拒绝或转入人工审批路径

该字段用于防止：

- 高风险流程被 Agent 自动调用
- 不经审批的批量执行

#### **require_approval**

- 表示 **是否必须存在人工审批节点**

- 校验阶段将检查：

  - 若 require_approval=true，但流程中不存在 human_approval

    → Workflow 定义不合法

这一机制确保：

- 人工审批不是“靠约定”
- 而是被结构性强制

### **4.8.2.3 Workflow 级策略的使用阶段**

Workflow 级策略会被以下模块使用：

1. **校验器（第 5 章）**
   - 判断流程定义是否合法
   - 拦截不符合风险要求的结构
2. **执行引擎（第 6 章）**
   - 决定是否允许自动推进
   - 决定是否必须进入人工状态
3. **审计系统（第 10 章）**
   - 解释“为什么该流程必须人工介入”

## **4.8.3 Node 级执行声明**

### **4.8.3.1 Node 级声明的目的**

Node 级执行声明并不描述 Node 的实现方式，

而是声明：

**“这个 Node 在系统治理层面，具有什么不可忽略的执行属性？”**

示例：

```
{
  "type": "call_external_api",
  "risk_level": "high",
  "idempotent": false
}
```

### **4.8.3.2 常见 Node 级声明及其意义**

#### **risk_level**

- 表示 **该 Node 的执行行为本身具有风险**
- 用于校验：
  - 高风险 Node 是否必须出现在人工审批之后
  - 是否被嵌入在低风险 Workflow 中

#### **idempotent**

- 表示 **该 Node 是否允许被安全地重复执行**
- 执行引擎将依据该声明：
  - 禁止对非幂等 Node 自动重试
  - 禁止在恢复执行时重复调用

这一声明直接影响：

- retry 策略
- resume 行为
- 审计对“为何未重试”的解释

### **4.8.3.3 Node 级声明的使用方式**

Node 级声明会被用于：

1. **编译 / 校验阶段**
   - 检查 Node 的使用位置是否合法
2. **执行阶段**
   - 决定是否允许 retry / resume
3. **审计阶段**
   - 解释执行策略的选择结果

# **5. Node设计**

## **5.1 Node 的定位与边界**

### **5.1.1 Node 的定义**

Node 是 Workflow 的原子执行单元，满足“输入 → 执行 → 输出”的确定性接口契约。Node 的输出必须结构化，且可被下游节点/控制结构引用。

### **5.1.2 Node 不承载控制流（边界红线）**

为了保持 DSL 的可校验与可治理，Node **不得**内嵌以下逻辑（这些逻辑由 Workflow 层承载）：

- if/else 分支选择
- retry/terminate/跳转下一步
- 触发人工介入（审批、补参）
- 修改 workflow 结构（增删节点、改边）

原则：**Node 只产出事实或结构化判断；Workflow 负责“基于状态决定走向”。**

## **5.2 Node Type / Node Schema / Node Instance**

本系统为了实现 **可生成、可校验、可执行、可回放、可治理** 的工程闭环，将 Node 按 **Type / Schema / Instance** 三层进行建模与管理。三者分别回答不同问题：

- **Node Type**：系统提供了哪些稳定的原子能力？（能力清单）
- **Node Schema**：某个能力如何被使用？需要什么输入、产出什么输出、具有什么治理属性？（能力契约）
- **Node Instance**：在某个 Workflow 中如何具体使用该能力？（一次具体调用）

| **对象**      | **归属**          | **生命周期**               | **复用范围**                  | **典型复用方式**            |
| ------------- | ----------------- | -------------------------- | ----------------------------- | --------------------------- |
| Node Type     | 平台能力资产      | 长期存在                   | 跨 workflow / 跨文档 / 跨业务 | 多个 workflow 引用同一 type |
| Node Schema   | Type 的契约       | 长期存在（随 type 版本）   | 跨 workflow / 跨业务          | 静态校验 + 生成对齐         |
| Node Instance | workflow 流程资产 | 短期或随 workflow 版本存在 | 同一 workflow 版本内          | 多次执行同一 workflow       |

### **5.2.1 Node Type（能力类型）**

**定义**

Node Type 是平台级的能力类型条目，由 Node Catalog/Registry 管理并版本化。它代表一个稳定的、可复用的原子能力，例如：

- ontology.identify_substance
- ontology.lookup_hazard_level
- decision.check_transport_mode
- io.check_filing_status
- utility.compliance_summary

**作用**

Node Type 的存在使得 Workflow 生成与编排不需要“发明新能力”，而是在受控的能力集合中组合；同时便于做统一治理（风险、权限、白名单等）。

**约束**

- Workflow 只能引用 Catalog 中已注册的 Node Type。
- Node Type 必须具备版本信息，以支持兼容与演进。

### **5.2.2 Node Schema（能力契约）**

**定义**

Node Schema 是 Node Type 的结构化契约，定义该 Type 的输入输出结构与治理属性。Node Schema 不描述执行算法，仅用于：

- Workflow 生成阶段：确定节点需要哪些输入、可产出哪些输出
- 校验阶段：静态检查数据引用与类型匹配、约束是否满足
- 执行/回放/治理：依据幂等性、副作用、风险等级等做策略判断

**Schema 的最小要素**

Node Schema 至少应包含：

1. **输入契约**：inputs_schema
2. **输出契约**：outputs_schema
3. **治理属性**：governance/capabilities（例如 risk、side_effect、idempotent、retryable、allowlist 等）
4. **版本与分类信息**：version/category/summary

**示例**

以第三章示例中的 ontology.lookup_hazard_level 为例：

```
{
  "type": "ontology.lookup_hazard_level",
  "version": "1.0.0",
  "category": "ontology",
  "summary": "根据 CAS 编号查询危险等级",
  "inputs_schema": {
    "cas_number": { "type": "string", "required": true }
  },
  "outputs_schema": {
    "hazard_level": { "type": "integer", "required": true }
  },
  "capabilities": {
    "side_effect": false,
    "idempotent_default": true,
    "retryable_default": true
  },
  "governance": {
    "risk_level_default": "low",
    "requires_allowlist": false
  }
}
```

**约束**

- Node Schema 必须可被静态校验器解析（结构化、无自然语言歧义）。
- Node Schema 不得包含流程控制语义（如 next/branch/terminate/retry-algorithm）。
- Node Schema 的治理属性是校验与治理的输入，不应由运行时模型自行推断替代。

### **5.2.3 Node Instance（Workflow 中的节点实例）**

**定义**

Node Instance 是某个 Workflow 内对某个 Node Type 的一次具体使用。它至少包含：

- id：实例标识（workflow 内唯一）
- type：引用的 Node Type
- inputs：将 Workflow 上下文（$.inputs 与上游 $.outputs.*）映射为该节点所需输入
- policy：实例级治理声明（可选；只能收紧，不得放松）

**示例（危险品运输合规判定中的实例）**

```
{
  "id": "level",
  "type": "ontology.lookup_hazard_level",
  "inputs": {
    "cas_number": "$.outputs.identify.entity.cas_number"
  },
  "policy": {
    "risk_level": "low"
  }
}
```

**实例级策略收紧原则**

- Node Type 在 Schema 中定义默认治理属性（例如 risk_level_default=high、requires_allowlist=true）。
- Node Instance 可以在 policy 中将约束 **收紧**（例如把 risk 从 medium 提高到 high），但不得放松（例如把 requires_allowlist 从 true 改为 false）。

**约束**

- Node Instance 必须引用已注册的 Node Type。
- Node Instance 必须显式声明输入映射（除非该 Type 被定义为无输入）。
- Node Instance 不能包含流程控制逻辑（if/else 决定下一跳、循环、终止、重试算法、动态改图）。
- Node Instance 的 policy 不能放松 Node Type Schema 中的治理默认值。

## **5.3 Node Catalog**

本节给出覆盖第三章“危险品运输合规判定”的**最小 Node Type 集合**。Catalog 的目标不是枚举所有节点，而是提供一组**可复用、可校验、可治理**的原子能力，使第四章 DSL 能够稳定生成并执行。

### **5.3.1 Ontology / Grounding 类**



#### **ontology.identify_substance**

**语义**：将 substance_name grounding 为本体实体（CAS/UN 等），并提供来源证据。

**输入示例**

```
{
  "substance_name": "硫酸"
}
```

**输出示例**

```
{
  "is_hazardous": true,
  "entity": {
    "canonical_name": "硫酸",
    "cas_number": "7664-93-9",
    "un_number": "UN1830"
  },
  "evidence": {
    "source_type": "ontology",
    "source_id": "hazchem_catalog_2025Q4",
    "record_id": "7664-93-9"
  }
}
```

> 若未命中（必须显式）可返回：

```
{
  "is_hazardous": null,
  "entity": null,
  "evidence": { "source_type": "ontology", "source_id": "hazchem_catalog_2025Q4", "status": "not_found" }
}
```

#### **ontology.lookup_hazard_level**

**语义**：依据 CAS 查询危险等级/类别等结构化属性，并提供证据。

**输入示例**

```
{
  "cas_number": "7664-93-9"
}
```

**输出示例**

```
{
  "hazard_level": 2,
  "hazard_class": "腐蚀性物质",
  "regulatory_tags": ["危险化学品", "受控运输"],
  "evidence": {
    "source_type": "ontology",
    "source_id": "national_hazchem_list_GB_2023",
    "record_id": "7664-93-9"
  }
}
```

### **5.3.2 Decision / Rule Eval 类（只产出判断，不做分支）**

#### **decision.check_transport_mode**

**语义**：评估“危险等级/运输方式/数量”等是否满足运输规则，输出结构化判断与原因码。

**注意**：该节点**不决定下一跳**，仅输出 required_checks 等结构化结果，供 workflow 条件引用。

**输入示例**

```
{
  "hazard_level": 2,
  "transport_mode": "公路",
  "quantity_kg": 5000
}
```

**输出示例（允许但需要备案）**

```
{
  "allowed": true,
  "reason_code": "L12_ROAD_ALLOWED_REQUIRE_FILING",
  "required_checks": ["filing_required"],
  "evidence": {
    "source_type": "policy_doc",
    "doc_id": "hazmat_transport_policy_v3",
    "rule_id": "R-ROAD-L12-FILING",
    "span": "p3-para2"
  }
}
```

**输出示例（不允许）**

```
{
  "allowed": false,
  "reason_code": "MODE_NOT_ALLOWED",
  "required_checks": [],
  "evidence": {
    "source_type": "policy_doc",
    "doc_id": "hazmat_transport_policy_v3",
    "rule_id": "R-MODE-BAN",
    "span": "p4-para1"
  }
}
```

典型 DSL 用法：

- contains($.outputs.mode_check.required_checks, "filing_required") → 执行 io.check_filing_status
- $.outputs.mode_check.allowed == false → 直接汇总或进入人工节点（由 workflow 控制）

### 5.3.3 IO / Systems 类

#### **io.check_filing_status**

**语义**：查询运输备案状态（可能依赖业务系统/外部系统，通常需要 allowlist）。输出备案是否存在、备案号、状态与证据。

**输入示例**

```
{
  "shipment_id": "SHIP-2026-00019",
  "cas_number": "7664-93-9",
  "transport_mode": "公路",
  "quantity_kg": 5000
}
```

**输出示例（已备案）**

```
{
  "has_filing": true,
  "filing_id": "FILING-2026-103388",
  "filing_status": "approved",
  "valid_until": "2026-03-31",
  "evidence": {
    "source_type": "business_system",
    "system": "filing_registry",
    "record_id": "FILING-2026-103388"
  }
}
```

**输出示例（未备案）**

```
{
  "has_filing": false,
  "filing_id": null,
  "filing_status": "missing",
  "evidence": {
    "source_type": "business_system",
    "system": "filing_registry",
    "query_ref": "querylog:2026-01-29T10:12:45Z#89321"
  }
}
```

**输出示例（系统不可用/未知）**

```
{
  "has_filing": null,
  "filing_id": null,
  "filing_status": "unknown",
  "evidence": {
    "source_type": "business_system",
    "system": "filing_registry",
    "status": "unavailable",
    "error_code": "TIMEOUT"
  }
}
```

### **5.3.4 Utility / Summarize 类**

#### **utility.compliance_summary**

**语义**：汇总上游节点的结构化输出，形成最终“是否合规/缺失项/证据束”。

**注意**：summary 输出建议结构化（便于下游展示、统计、审计）；自然语言表述可由 Agent 层生成。

**输入示例**

```
{
  "substance_name": "硫酸",
  "transport_mode": "公路",
  "identify": {
    "is_hazardous": true,
    "entity": { "cas_number": "7664-93-9" }
  },
  "level": { "hazard_level": 2 },
  "mode_check": {
    "allowed": true,
    "required_checks": ["filing_required"],
    "reason_code": "L12_ROAD_ALLOWED_REQUIRE_FILING"
  },
  "filing": {
    "has_filing": false,
    "filing_status": "missing"
  }
}
```

**输出示例（不合规：缺备案）**

```
{
  "compliant": false,
  "decision_code": "NOT_COMPLIANT_MISSING_FILING",
  "missing_requirements": [
    {
      "code": "dangerous_goods_road_filing",
      "severity": "high",
      "message": "危险化学品公路运输备案缺失"
    }
  ],
  "facts": {
    "hazard_level": 2,
    "transport_mode": "公路",
    "filing_required": true,
    "has_filing": false
  },
  "evidence_bundle": [
    { "ref": "$.outputs.identify.evidence" },
    { "ref": "$.outputs.level.evidence" },
    { "ref": "$.outputs.mode_check.evidence" },
    { "ref": "$.outputs.filing.evidence" }
  ]
}
```

**输出示例（合规）**

```
{
  "compliant": true,
  "decision_code": "COMPLIANT",
  "missing_requirements": [],
  "facts": {
    "hazard_level": 2,
    "transport_mode": "公路",
    "filing_required": true,
    "has_filing": true
  },
  "evidence_bundle": [
    { "ref": "$.outputs.identify.evidence" },
    { "ref": "$.outputs.level.evidence" },
    { "ref": "$.outputs.mode_check.evidence" },
    { "ref": "$.outputs.filing.evidence" }
  ]
}
```

# 6. Workflow 与 Node 的生命周期

本章描述从“领域能力建设”到“单次文档分析与执行”的全流程中，**Node Type/Schema、Node Instance、Workflow（Definition/Version）** 的产生、复用与消亡边界。核心原则：

- **Node Type/Schema：平台能力资产，长期存在、跨流程复用。**
- **Node Instance：流程资产的一部分，随 Workflow 草案/版本产生与演进。**
- **Workflow Definition/Version：流程资产，可版本化、多次执行、可回放。**
- **单次执行（Execution）：运行态对象，执行完即结束，但 trace 可保留用于审计/回放。**

## **6.1 Phase 1：领域能力建设（Node Type/Schema 库）**

![workflow插图phase1](./workflow插图phase1.png)

- Node Type/Schema 在 Phase 1 形成并进入 Catalog，**不会因某次文档处理而消亡**；
- 后续 workflow 生成/校验依赖这些 schema 做静态检查与治理约束。

## **6.2 Phase 2：Workflow 资产沉淀（可选，但推荐）**

![workflow插图phase2](./workflow插图phase2.png)

- Workflow 作为资产进入 Registry 后，形成可复用版本（v1/v2…），**Node Instance 随 workflow 版本存在**；
- 流程变更通过发布新版本实现，避免运行时改图。

## **6.3 Phase 3：单次文档分析与执行（规划/选择 + 执行/产出）**

### **Phase 3A：规划/选择（历史资产 + 本次 PDF → Validated Workflow）**

![workflow插图phase3A](./workflow插图phase3A.png)

### **Phase 3B：执行/产出（Validated Workflow → Structured Result → Answer）**

- 本次 PDF 默认用于提供**事实/证据/增量约束**，而非动态生成新的 Node Type；
- 执行完成后，Execution 结束，但 trace 可长期保存以支持审计与回放。

## 7. TodoList（待详细设计）

### **Workflow 执行引擎与运行时模型（不写算法，写语义）**

- 定义 Execution/Run 的状态机：pending/running/waiting_human/completed/failed/cancelled
- 定义 Node 执行状态：scheduled/running/succeeded/failed/skipped
- 定义条件评估与分支选择语义（otherwise、优先级、互斥校验）
- 定义数据上下文模型：inputs、outputs、errors、evidence、trace 结构
- 定义幂等/副作用对 resume/retry 的影响（声明式规则）
- 定义人工介入节点的运行态行为（暂停、补参、审批、继续）

### **Planning Context（Retriever / 规则加载 / Primer）**

- 定义候选能力召回：如何从 Catalog 选子集（按意图/域/权限）
- 定义规则加载策略：只加载 trigger/constraint（来源：制度库 + 本次PDF若为policy）
- 定义 Primer：生成/编排必须遵守的建模范式（不发明 type、不在 node 内控制流等）
- 定义“本次 PDF vs 历史资产”的分型与作用（Instance facts/evidence vs Policy delta）
- 输出 Phase3 的两张简化时序图（Planning / Execution）

### **从非结构化文档到 Workflow 的生成链路**

- 定义文档解析产物（IR）：步骤、依赖、条件、约束、缺参点、证据指针
- 定义 Step-to-Type 对齐：匹配策略（语义+I/O+域约束），失败如何处理
- 定义 Workflow Draft 生成：nodes/edges/conditions/inputs 的生成规则
- 定义“需要人工校验”的触发条件（歧义、缺参、规则冲突、高风险）
- 用第三章示例走一遍：文档片段 → IR → Draft DSL

### **校验与静态分析（可执行前置关卡）**

- DSL 结构校验：DAG、入口/出口、不可达节点、环检测
- I/O 校验：引用字段存在性、类型匹配、required 缺失处理
- 分支校验：互斥性、完备性（otherwise）、优先级与冲突检测
- 治理校验：risk/allowlist/side_effect 与 execution_policy 的一致性
- 输出校验错误的标准化格式（error_code、定位到节点/字段/边）

### **治理与安全（运行前/运行中/运行后）**

- 权限模型：谁能调用哪些 Node Type / 哪些外部系统（allowlist）
- 风险策略：risk_level 与 require_approval/allow_auto 的关系
- 数据分级与脱敏：inputs/outputs/evidence 的存储与展示策略
- 变更治理：Node Type/Workflow 版本发布、弃用、兼容策略
- 审计要求：必须记录哪些关键事件（分支选择、人工介入、外部调用）

### **可观测与审计回放**

- Trace 模型：Execution、Node 输出快照、边选择、错误、人工操作事件
- 回放能力：给定 exec_id 还原执行路径与关键证据
- 指标体系：成功率、人工介入率、规则命中率、外部系统失败率、耗时分布
- 解释输出：如何从结构化结果 + evidence 生成“可审计解释”

### **资产化与演进（Catalog/Workflow 的长期维护）**

- Node Catalog 生命周期：新增/升级/弃用流程（owner、评审、版本策略）
- Workflow Registry 生命周期：draft→review→published→deprecated
- 复用策略：模板化 workflow、参数化、域隔离（不同业务线不同规则包）
- 能力缺口回流：Phase3 发现缺口 → 形成“Type 需求” → 回到 Phase1
- 最小落地里程碑：先跑通第三章示例闭环（端到端验收清单）