# Palantir 概念与 Workflow Design 概念映射表

**目的:** 将 Palantir Foundry 的 Action / AIP Logic / Rules 概念与 workflow-design.md 的设计概念进行映射对比，便于理解两者在能力和设计思路上的异同。

**生成日期:** 2026-02-06
**版本:** v1.0

---

## 一、概念总览对比表

### 1.1 核心概念映射

| Palantir Foundry | workflow-design.md | 对应关系说明 |
|-----------------|-------------------|-------------|
| **Action Type** | **Workflow Definition** | 都是可复用的流程定义，支持版本化 |
| **Action Instance** | **Workflow Execution** | 都是具体运行时的执行实例 |
| **Rule** | **Node** (部分) | Rule 更接近原子操作，但 workflow-design.md 的 Node 职责更单一（输入→执行→输出，不做分支决策） |
| **Parameters** | **Workflow.inputs** + **Node.inputs** | 都是定义流程/节点的输入接口 |
| **Submission Criteria** | **Workflow.execution_policy** | 都是定义执行前的约束条件 |
| **Validation Logic** | **校验器（第 5 章）** | 都是静态校验机制 |
| **Side Effects** | **Node.capabilities.side_effect** | 都是对副作用的声明 |
| **AIP Logic** | **无直接对应（决策类 Node 可部分对应）** | AIP Logic 是无代码 LLM 函数，workflow-design.md 的 Decision Node 是结构化规则 |
| **Automate Trigger** | **（暂未设计，可作为扩展）** | Automate 是事件驱动触发，workflow-design.md 目前是主动调用 |
| **Ontology Object** | **（依赖外部 Ontology 组）** | 都是结构化数据模型 |
| **Function** | **Node Type 实现** | 都是可复用的代码逻辑 |
| **AIP Evals** | **（暂未设计，可作为扩展）** | AIP Evals 是 LLM 函数质量评估，workflow-design.md 暂无对应机制 |
| **Agent Studio** | **无直接对应（Agent 组负责）** | Agent Studio 是交互式助手，workflow-design.md 的显式人工节点更结构化 |

---

## 二、逐层详细映射

### 2.1 Action vs Workflow

#### Palantir Foundry Action

**定义：**
- Action 是操作 Ontology 对象的核心机制
- 一个事务性操作，根据逻辑修改对象的属性和链接
- 作为 Ontology 的一等公民，通过 UI 触发

**核心组件：**
```
Action Type {
  Parameters          // 输入参数
  Rules               // 执行逻辑
  Submission Criteria // 提交条件
  Validation Logic    // 校验逻辑
  Side Effects        // 副作用行为
}
```

#### workflow-design.md Workflow

**定义：**
- Workflow 是一个可执行的流程定义，表示为 DAG
- 由多个 Node 及其连接关系构成
- 作为一等资产，支持版本化、审计与回放

**核心组件：**
```
Workflow Definition {
  metadata           // 元信息（描述、风险等级）
  inputs             // 输入 schema
  nodes              // 节点集合
  edges              // 节点依赖关系
  execution_policy   // 执行策略（allow_auto、require_approval）
}
```

#### 映射分析

| Palantir Action | workflow-design.md Workflow | 对应关系 |
|----------------|---------------------------|---------|
| Parameters | inputs | ✅ 直接对应：定义输入接口 |
| Rules | nodes | ⚠️ 部分对应：Rules 包含事务性操作（Create/Modify/Delete Object），workflow-design.md 的 Node 更通用 |
| Submission Criteria | execution_policy (部分) | ⚠️ 部分对应：Submission Criteria 控制何时允许提交，execution_policy 控制整体执行策略 |
| Validation Logic | 校验器（第 5 章） | ✅ 直接对应：都是静态校验机制 |
| Side Effects | Node.capabilities.side_effect | ✅ 直接对应：都是副作用声明 |
| Action Type Version | Workflow Definition Version | ✅ 直接对应：都支持版本化 |
| Action Instance | Workflow Execution | ✅ 直接对应：都是运行时实例 |

**关键差异：**

1. **执行模型：**
   - Action: 单次事务性操作，原子性强
   - Workflow: DAG 结构的多步骤流程，可显式分支、并行

2. **输入源：**
   - Action: 主要操作 Ontology 对象（结构化数据）
   - Workflow: 可以处理非结构化文档输入，通过 RAG + LLM 抽取

3. **人工介入：**
   - Action: 隐式（通过 Agent Studio 对话或 Automate 审批）
   - Workflow: 显式（human_input / human_approval 节点）

---

### 2.2 Rule vs Node

#### Palantir Foundry Rules

**定义：**
- Rules 是 Actions 的核心逻辑引擎
- 定义如何修改 Ontology 对象或执行副作用

**规则类型：**

**Ontology Rules（直接操作对象）：**
```
Ontology Rules {
  Create object                          // 创建对象
  Modify object(s)                       // 修改对象
  Create or modify object(s)             // 创建或修改
  Delete object(s)                       // 删除对象
  Create link(s)                         // 创建多对多链接
  Delete link                            // 删除链接
  Function rule                          // 引用 Ontology 编辑函数
  Create/Modify/Delete object(s) of interface  // 接口操作
}
```

**Other Rules（执行副作用）：**
```
Other Rules {
  Notification              // 发送平台通知
  Webhooks                // 调用外部 HTTP 端点
}
```

**值来源：**
```
Values & Parameters {
  $param.name              // 参数引用
  $objectParam.property    // 对象参数属性
  $currentUser             // 当前用户
  $currentTime             // 当前时间
  静态值                  // 硬编码值
  函数调用                // 通过 Function rule
}
```

#### workflow-design.md Node

**定义：**
- Node 是 Workflow 的原子执行单元
- 满足"输入 → 执行 → 输出"的确定性接口契约
- Node 不承载控制流逻辑（if/else、retry、人工介入由 Workflow 层负责）

**Node 类型：**
```
Node Type Catalog {
  // Ontology / Grounding 类
  ontology.identify_substance           // 将物质名称 grounding 为实体
  ontology.lookup_hazard_level         // 查询危险等级

  // Decision / Rule Eval 类
  decision.check_transport_mode        // 评估运输方式是否合规

  // IO / Systems 类
  io.check_filing_status               // 查询备案状态

  // Utility / Summarize 类
  utility.compliance_summary           // 汇总合规结果
}
```

**Node Schema 结构：**
```
Node Schema {
  inputs_schema          // 输入契约
  outputs_schema         // 输出契约
  capabilities           // 能力声明（side_effect、idempotent、retryable）
  governance             // 治理属性（risk_level、requires_allowlist）
}
```

**Node Instance 结构：**
```
Node Instance {
  id                      // 实例标识
  type                    // 引用的 Node Type
  inputs                  // 将 Workflow 上下文映射为节点输入
  policy                  // 实例级治理声明（可选，只能收紧）
}
```

#### 映射分析

| Palantir Rule | workflow-design.md Node | 对应关系 |
|--------------|----------------------|---------|
| **Ontology Rules** | **ontology.* Nodes** | ✅ 直接对应：都操作 Ontology 对象 |
| Create object | 无直接对应 | ⚠️ workflow-design.md 的 Node 不直接创建对象，而是通过查询/判断产出事实 |
| Modify object(s) | 无直接对应 | ⚠️ workflow-design.md 的 Node 是只读或产生结构化输出，不修改外部对象 |
| Function rule | Node Type 实现 | ✅ 直接对应：都是可复用的代码逻辑 |
| **Other Rules** | **io.* Nodes** | ⚠️ 部分对应：Webhooks 可对应 io.* 节点，Notification 需要新增节点类型 |
| **Values & Parameters** | **inputs_schema** | ✅ 直接对应：都定义输入接口 |
| $param.name | $.inputs.* | ✅ 直接对应：都引用输入参数 |
| $objectParam.property | 无直接对应 | ⚠️ workflow-design.md 没有对象参数概念，数据通过上下文传递 |
| $currentUser / $currentTime | 无直接对应 | ⚠️ workflow-design.md 没有内置上下文变量（可作为扩展） |

**关键差异：**

1. **职责边界：**
   - Rule: 可以直接创建/修改/删除对象，事务性强
   - Node: 只产出结构化输出（事实、判断、查询结果），不直接修改外部状态

2. **控制流：**
   - Rule: 可以嵌套逻辑（如 Create object 后 Create link）
   - Node: 不包含控制流逻辑，分支由 Workflow 的 edges 决定

3. **复用机制：**
   - Rule: 在 Action 内部复用，通过 Functions 实现跨 Action 复用
   - Node: 通过 Node Type / Schema / Instance 三层模型，支持跨 Workflow 复用

---

### 2.3 AIP Logic vs Decision Node

#### Palantir AIP Logic

**定义：**
- 无代码 LLM 函数构建环境
- 连接非结构化输入（PDF、邮件、聊天记录）与 Ontology 数据
- 完全遵守平台的用户和函数权限治理

**典型应用场景：**
- 连接非结构化输入与 Ontology 对象
- 解决调度冲突
- 优化资源分配
- 处理供应链中断
- 自动化关键业务流程

**核心特性：**
- 无代码 UI 编辑器
- 直接连接 LLM
- 输出可直接写入 Ontology 对象

**四层自动化模型：**
```
四层自动化模型 {
  即时分析         // 单次查询，人机协作
  任务代理         // 执行特定任务
  流程代理         // 执行多步流程
  工作代理         // 全自动化
}
```

#### workflow-design.md Decision Node

**定义：**
- Decision Node 是产出结构化判断的 Node 类型
- 输出包含 allowed、reason_code、required_checks、evidence 等字段
- 不决定下一跳，仅输出结构化结果供 Workflow 条件引用

**示例：decision.check_transport_mode**

**输入：**
```
{
  "hazard_level": 2,
  "transport_mode": "公路",
  "quantity_kg": 5000
}
```

**输出：**
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

#### 映射分析

| Palantir AIP Logic | workflow-design.md Decision Node | 对应关系 |
|------------------|-------------------------------|---------|
| **无代码 UI 编辑器** | **（暂无对应）** | ❌ 无直接对应：workflow-design.md 的 Node 需要手动编写 |
| **连接非结构化输入** | **文档抽取 → Workflow 生成** | ⚠️ 间接对应：workflow-design.md 的流程是从文档抽取的，但不是无代码 |
| **LLM 调用** | **无直接对应** | ⚠️ workflow-design.md 的 Node 可以内部调用 LLM，但这是实现细节 |
| **输出写入 Ontology 对象** | **outputs_schema** | ⚠️ 部分对应：Decision Node 产出结构化输出，但不直接写对象 |
| **四层自动化模型** | **（暂无对应）** | ⚠️ 可作为设计参考：定义 Workflow 的自动化程度分级 |
| **AIP Evals 评估** | **（暂无对应）** | ❌ 无直接对应：workflow-design.md 暂无 Node 质量评估机制 |

**关键差异：**

1. **实现方式：**
   - AIP Logic: 无代码 UI，通过拖拽配置连接 LLM
   - Decision Node: 代码实现，需要定义 inputs_schema 和 outputs_schema

2. **不确定性处理：**
   - AIP Logic: 直接依赖 LLM 的推理能力
   - Decision Node: 产出结构化判断 + evidence，不确定的部分通过人工节点处理

3. **质量保证：**
   - AIP Logic: 通过 AIP Evals 进行自动化测试和评估
   - Decision Node: 暂无对应机制（可作为扩展）

**借鉴建议：**

- ✅ 可以引入 **无代码 Logic 编辑器**，降低 Decision Node 编写门槛
- ✅ 可以引入 **AIP Evals 评估框架**，为 Decision Node 建立质量保证
- ✅ 可以参考 **四层自动化模型**，为 Workflow 定义自动化程度分级

---

### 2.4 Automate vs (扩展点)

#### Palantir Automate

**定义：**
- 通过定义持续或定期检查的条件，在满足条件时自动执行效果
- 支持时间条件和对象状态变化触发

**触发方式：**
```
Trigger Types {
  时间条件          // 每天上午 9 点、每周一、CRON
  对象条件          // Ontology 对象的状态变化
}
```

**执行效果：**
```
Effects {
  Foundry Actions
  AIP Logic 函数
  平台/邮件通知
}
```

**典型场景：**
- 自动异常处理
- 工作流自动化
- 持续监控与响应

#### workflow-design.md (暂未设计)

workflow-design.md 目前聚焦于 **主动调用** 模式：
- Agent 通过 MCP Tool 调用 Workflow
- 用户通过界面触发 Workflow

**暂无对应：**
- ❌ 时间触发
- ❌ 对象状态变化触发
- ❌ 持续监控

#### 映射分析

| Palantir Automate | workflow-design.md | 对应关系 |
|------------------|-------------------|---------|
| 时间条件触发 | （暂无对应） | ❌ 无直接对应：可作为扩展 |
| 对象状态变化触发 | （暂无对应） | ❌ 无直接对应：可作为扩展 |
| 执行 Actions / AIP Logic | 执行 Workflow | ⚠️ 部分对应：都是执行流程，但触发方式不同 |
| 平台/邮件通知 | Node.capabilities.side_effect | ⚠️ 部分对应：side_effect 可以包含通知，但不是触发机制 |

**借鉴建议：**

- ✅ 可以引入 **事件驱动触发机制**，支持：
  - 时间触发（CRON）
  - 对象状态变化触发
  - 外部事件触发（Webhook）
- ✅ 可以定义 **Trigger DSL**，如：
  ```json
  {
    "trigger_type": "object_changed",
    "condition": {
      "object_type": "Shipment",
      "field": "status",
      "from": "draft",
      "to": "submitted"
    },
    "action": {
      "type": "execute_workflow",
      "workflow_id": "hazard_compliance"
    }
  }
  ```

---

### 2.5 AIP Evals vs (扩展点)

#### Palantir AIP Evals

**定义：**
- LLM 函数质量评估框架
- 通过测试用例、评估指标、黄金标准进行自动化评估

**核心要素：**
```
AIP Evals {
  测试用例          // 输入 + 期望输出
  评估指标          // accuracy、coverage、consistency
  黄金标准          // 专家标注的高质量样本
}
```

**价值：**
- 确保 LLM 函数的输出质量和一致性
- 为 LLM 函数提供持续的质量监控

#### workflow-design.md (暂无设计)

workflow-design.md 暂无对应的质量评估机制：
- ❌ Node 测试用例
- ❌ Node 评估指标
- ❌ Node 质量监控

#### 映射分析

| Palantir AIP Evals | workflow-design.md | 对应关系 |
|------------------|-------------------|---------|
| 测试用例 | （暂无对应） | ❌ 无直接对应：可作为扩展 |
| 评估指标 | （暂无对应） | ❌ 无直接对应：可作为扩展 |
| 黄金标准 | （暂无对应） | ❌ 无直接对应：可作为扩展 |
| 质量监控 | （暂无对应） | ❌ 无直接对应：可作为扩展 |

**借鉴建议：**

- ✅ 可以在 **Node Schema** 中引入 `evals` 字段：
  ```json
  {
    "type": "decision.check_transport_mode",
    "evals": {
      "test_cases": [
        {
          "inputs": { "hazard_level": 2, "transport_mode": "公路" },
          "expected": {
            "allowed": true,
            "required_checks": ["filing_required"]
          }
        }
      ],
      "metrics": ["accuracy", "coverage", "consistency"]
    }
  }
  ```
- ✅ 为 **从文档抽取的 Workflow** 建立质量保证机制
- ✅ 建立 **Node 质量监控面板**，跟踪成功率、错误率等指标

---

### 2.6 Agent Studio vs (扩展点)

#### Palantir Agent Studio

**定义：**
- 交互式助手构建环境
- 基于 LLM、Ontology、文档创建 AI 代理

**核心能力：**
```
Agent Studio {
  大语言模型        // 核心推理能力
  Ontology          // 结构化数据与知识
  文档              // 非结构化知识库
}
```

**特点：**
- 对话式交互
- 动态决策
- 人机协作

#### workflow-design.md (无直接对应)

workflow-design.md 的 **人工介入节点** 是结构化的、显式的：
- human_input: 收集用户输入
- human_approval: 人工审批

但 Agent Studio 是 **对话式、动态的**：
- 通过多轮对话收集信息
- LLM 自主决策何时需要人工介入

#### 映射分析

| Palantir Agent Studio | workflow-design.md | 对应关系 |
|----------------------|-------------------|---------|
| 对话式交互 | （暂无对应） | ❌ 无直接对应：人工节点是结构化的 |
| 动态决策 | （暂无对应） | ❌ 无直接对应：workflow-design.md 的流程是静态的 |
| 人机协作 | human_input / human_approval | ⚠️ 部分对应：都支持人工介入，但方式不同 |

**关键差异：**

1. **人工介入方式：**
   - Agent Studio: 对话式，动态决定何时需要人工
   - workflow-design.md: 显式节点，预先定义好人工介入点

2. **适用场景：**
   - Agent Studio: 协作、灵活、需要多轮交互的场景
   - workflow-design.md: 合规、审计、严苛场景，需要明确审计轨迹

3. **决策透明性：**
   - Agent Studio: 依赖 LLM 的推理过程
   - workflow-design.md: 流程静态可审计，每个决策都有 evidence

**借鉴建议：**

- ✅ 可以引入 **对话式人工节点**，作为显式节点的补充
- ✅ 可以定义 **两种人工介入模式**：
  - 结构化模式（当前）：适合合规场景
  - 对话式模式（扩展）：适合协作场景

---

## 三、核心设计思想对比

### 3.1 声明式 vs 命令式

| 维度 | Palantir Foundry | workflow-design.md | 共性 |
|------|------------------|-------------------|-----|
| **Action / Rule** | 声明式（UI 配置） | 声明式（JSON DSL） | ✅ 都强调声明式而非命令式 |
| **输入输出** | 显式定义 Schema | 显式定义 Schema | ✅ 都强调结构化接口 |
| **执行逻辑** | 通过 Rules 组合 | 通过 Node + Edge 组合 | ✅ 都通过原子能力组合 |

### 3.2 执行前校验 vs 运行时拦截

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **校验时机** | 主要在运行时（通过平台权限） | 执行前（静态校验器） |
| **校验方式** | 隐式（平台能力） | 显式（DSL 声明） |
| **校验内容** | 权限、对象有效性 | DAG 结构、I/O 匹配、分支完备性、治理一致性 |
| **优势** | 平台统一治理，运行时安全 | 静态可分析、执行前确定性、便于审计 |

**借鉴建议：**
- workflow-design.md 的 **执行前静态校验** 是显著优势
- 可以借鉴 Palantir 的 **细粒度权限体系**，增强运行时治理

### 3.3 人工介入模型

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **人工介入方式** | 隐式（Agent Studio 对话）或 Action 审批 | 显式（human_input / human_approval 节点） |
| **状态可见性** | 中（依赖用户交互） | 高（状态机明确） |
| **审计性** | 强（平台审计日志） | 强（Execution trace） |
| **适用场景** | 协作、灵活场景 | 合规、审计、严苛场景 |

**借鉴建议：**
- 两者不矛盾，可以互补：
  - workflow-design.md 的显式节点适合合规场景
  - Palantir 的对话式适合协作场景
- 可以提供 **两种人工介入模式** 作为可选配置

### 3.4 质量保证机制

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **质量评估** | AIP Evals（测试用例 + 指标） | （暂无） |
| **测试覆盖** | 自动化测试 | （暂无） |
| **质量监控** | 平台级监控 | （暂无） |

**借鉴建议：**
- ✅ 可以引入 **AIP Evals 评估框架**，为 Decision Node 建立质量保证
- ✅ 可以建立 **Node 质量监控面板**，跟踪成功率、错误率等指标
- ✅ 为 **从文档抽取的 Workflow** 建立自动化测试机制

---

## 四、无法对照的概念

### 4.1 Palantir 独有概念

以下 Palantir 概念在 workflow-design.md 中 **无直接对应**，且 **暂不建议直接引入**：

| 概念 | 说明 | 原因 |
|------|------|------|
| **Create/Modify/Delete Object** | 直接创建/修改/删除 Ontology 对象 | workflow-design.md 的 Node 是只读或产出结构化输出，不直接修改外部状态（符合"无副作用"原则） |
| **Create/Delete Link** | 创建/删除多对多链接 | workflow-design.md 没有对象链接概念，依赖外部 Ontology 组 |
| **Interface-based Operations** | 针对接口的操作 | workflow-design.md 没有接口概念 |
| **$objectParam** | 对象参数引用 | workflow-design.md 没有对象参数概念，数据通过上下文传递 |
| **$currentUser / $currentTime** | 内置上下文变量 | workflow-design.md 暂无内置上下文变量（可作为扩展） |

### 4.2 workflow-design.md 独有概念

以下 workflow-design.md 概念在 Palantir Foundry 中 **无直接对应**：

| 概念 | 说明 | 价值 |
|------|------|------|
| **Node Type / Schema / Instance 三层模型** | 将能力定义、能力契约、具体使用分层 | 支持跨 Workflow 复用、统一治理、版本演进 |
| **DAG + 状态机执行模型** | 以 DAG 结构表达流程，以状态机思想控制执行 | 支持静态校验、可回放、可审计 |
| **显式人工介入节点** | human_input / human_approval 作为一等语义 | 明确审计轨迹、适合合规场景 |
| **execution_policy（声明式治理）** | Workflow 级和 Node 级的执行策略声明 | 执行前静态校验、声明式治理 |
| **从非结构化文档自动生成 Workflow** | 通过 RAG + LLM 抽取流程并生成 DSL | 降低 Workflow 编写门槛、沉淀流程资产 |
| **evidence 溯源** | 每个 Node 输出包含 evidence 字段 | 支持精细的审计回放、可解释性 |

---

## 五、设计演进建议

### 5.1 短期（1-2 个月）

**优先级 1（必做）：**
1. 引入 **AIP Evals 评估框架**，为 Decision Node 建立质量保证
2. 建立 **Platform-wide Observability**，统一监控与审计
3. 完善 **Node Catalog 分类**，参考 Palantir 的分类思路

**优先级 2（应该做）：**
1. 为 **无代码 Logic 编辑器** 做技术预研，降低 Node 编写门槛
2. 评估 **事件驱动触发机制** 的可行性

### 5.2 中期（3-6 个月）

**优先级 1（必做）：**
1. 引入 **无代码 Logic 编辑器**，支持可视化 Node Schema 定义
2. 建立 **事件驱动触发机制**，支持时间触发和对象状态变化触发
3. 定义 **四层自动化模型**，为 Workflow 设计提供目标

**优先级 2（应该做）：**
1. 建立 **Node 质量监控面板**，跟踪成功率、错误率等指标
2. 评估 **对话式人工节点** 的可行性

### 5.3 长期（6 个月以上）

**优先级 1（必做）：**
1. 建立 **Ontology 深度集成**，支持 Workflow 输入/输出直接映射到 Ontology 对象
2. 引入 **对话式人工节点**，作为显式节点的补充

**优先级 2（应该做）：**
1. 建立 **Agent Studio 类能力**，支持基于 LLM 的交互式代理
2. 探索 **内置上下文变量**（$currentUser、$currentTime）的支持

---

## 六、总结

### 6.1 核心定位差异

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **核心输入** | 结构化数据（Ontology） | 非结构化文档 |
| **核心输出** | 对象修改 + 数据联动 | 结构化判定 + 证据链 |
| **主要场景** | 数据操作 + 协作 | 合规判定 + 审计 |
| **自动化程度** | 中（强调人机协作） | 高（强调自动执行） |
| **治理重点** | 运行时拦截（平台权限） | 执行前校验（静态分析） |

### 6.2 互补性分析

**Palantir Foundry 的优势：**
- ✅ 无代码编辑器（降低门槛）
- ✅ 平台级安全与权限体系
- ✅ AIP Evals 质量评估
- ✅ 四层自动化模型
- ✅ Ontology 深度集成

**workflow-design.md 的优势：**
- ✅ 非结构化文档自动抽取
- ✅ 执行前静态校验
- ✅ 显式的人工介入语义
- ✅ DAG + 状态机的执行模型
- ✅ 精细的 evidence 源源

**结论：**
- 两者不是竞品关系，而是 **互补能力**
- workflow-design.md 可以作为 Palantir Foundry 的 **非结构化 → 结构化** 桥梁
- Palantir Foundry 的无代码编辑器和质量评估机制可以为 workflow-design.md 提供借鉴

### 6.3 最终建议

**不要试图成为 Palantir Foundry 的竞品。**

**应该定位为：**
- Foundry 的 **互补能力**（非结构化 → 结构化）
- 合规/审计领域的 **垂直解决方案**
- Agent 驱动的 **Workflow 编排平台**

**核心差异化：**
- 文档 → Workflow 的 **自动生成**
- 执行前的 **静态校验与治理**
- 显式的 **人工介入与审计语义**

---

**报告版本：** v1.0
**作者：** OpenClaw Agent
**日期：** 2026-02-06
