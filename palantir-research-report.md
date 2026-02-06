# Palantir Foundry 能力调研报告

**调研目标：** 深入了解 Palantir Foundry 的 Actions、Rules 和 AIP Logic 能力，并与 `workflow-design.md` 的设计进行对比分析。

**调研日期：** 2026-02-05

**调研方法：** 官方文档分析（通过 web_fetch 获取）

---

## 1. 执行摘要

本报告深入调研了 Palantir Foundry 的三大核心能力：

1. **Foundry Actions** - 面向 Ontology 的声明式事务操作能力
2. **Foundry Rules** - 内置于 Actions 的规则引擎
3. **AIP Logic** - 无代码 LLM 函数构建环境

**核心发现：**

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **输入源** | 结构化数据（Ontology 对象） | 非结构化文档 |
| **生成方式** | 无代码 UI 手工配置 | 系统自动抽取 + 人工校验 |
| **执行模型** | 事务性 Actions + Automate 触发器 | DAG + 状态机 + 静态校验 |
| **人工介入** | Agent Studio / Automate 审批 | 显式 human_input 节点 |
| **LLM 集成** | AIP Logic（无代码函数） | 通过 Agent 调用 Workflow |

**关键借鉴点：**

- ✅ **无代码 Logic 编辑器**：降低规则编写门槛
- ✅ **AIP Evals 评估框架**：LLM 函数的质量保证
- ✅ **四层自动化模型**：从即时分析到全自动化
- ✅ **Platform-wide Observability**：统一的可观测性体系
- ⚠️ **Rules 作为 Actions 的一部分**：规则与操作深度绑定

---

## 2. Palantir Foundry 核心能力详解

### 2.1 Foundry Actions

#### 2.1.1 核心概念

**Actions** 是 Foundry 平台上操作 Ontology 对象的核心机制：

- **定义**：一个事务性操作，根据用户定义的逻辑修改一个或多个对象的属性和链接
- **定位**：作为 Ontology 的一等公民，用户可以通过 UI 应用 actions
- **安全模型**：完全集成平台的用户权限和函数权限体系

#### 2.1.2 Actions 的组成

一个 Action Type 包含以下组件：

| 组件 | 描述 | 对应 workflow-design.md |
|------|------|------------------------|
| **Parameters** | 定义 action 的输入参数 | Node.inputs_schema |
| **Rules** | 定义 action 的执行逻辑 | Node 的实现 |
| **Submission Criteria** | 定义何时允许提交 | Node.execution_policy |
| **Validation Logic** | 定义数据的校验规则 | 静态校验器 |
| **Side Effects** | 定义副作用行为（通知、Webhooks） | Node.capabilities.side_effect |

#### 2.1.3 Rules（规则引擎）

Rules 是 Actions 的核心逻辑引擎，分为两类：

**Ontology Rules：** 直接操作 Ontology 对象
- `Create object` - 创建预定义类型的对象
- `Modify object(s)` - 修改现有对象
- `Create or modify object(s)` - 智能创建或修改
- `Delete object(s)` - 删除现有对象
- `Create link(s)` - 创建多对多链接
- `Delete link` - 删除链接
- `Function rule` - 引用 Ontology 编辑函数
- `Create/Modify/Delete object(s) of interface` - 针对接口的操作

**Other Rules：** 执行副作用行为
- `Notification` - 发送平台通知
- `Webhooks` - 调用外部 HTTP 端点

#### 2.1.4 Values & Parameters

Rules 支持从多种来源获取值：

- **参数引用**：`$param.name`
- **对象参数属性**：`$objectParam.property`
- **静态值**：直接硬编码
- **当前用户/时间**：`$currentUser`, `$currentTime`
- **函数调用**：通过 Function rule

**关键特性：**
- 支持同时创建对象和多对多链接
- 有明确的无效组合规则（如创建对象后立即删除）
- 支持参数类型验证

---

### 2.2 AIP (Artificial Intelligence Platform) Logic

#### 2.2.1 核心概念

**AIP Logic** 是 Foundry 的无代码 LLM 函数构建环境：

- **定位**：无代码开发环境，连接非结构化输入与 Ontology 数据
- **能力**：基于大语言模型的函数构建与部署
- **安全**：完全遵守平台的用户和函数权限治理

#### 2.2.2 典型应用场景

AIP Logic 主要用于：

- 连接非结构化输入（PDF、邮件、聊天记录）与 Ontology 对象
- 解决调度冲突（如智能排班）
- 优化资源分配（如产能规划）
- 处理供应链中断
- 自动化关键业务流程

#### 2.2.3 AIP Agent Studio（AI 代理构建）

Agent Studio 提供交互式助手创建能力，基于：

- **大语言模型**：核心推理能力
- **Ontology**：结构化数据与知识
- **文档**：非结构化知识库

**四层自动化模型：**

| 层级 | 描述 | 示例 |
|------|------|------|
| **即时分析** | 单次查询，人机协作 | 分析这个合同的风险 |
| **任务代理** | 执行特定任务 | 提取这个发票的信息 |
| **流程代理** | 执行多步流程 | 完成供应商准入流程 |
| **工作代理** | 全自动化 | 自主处理一类业务问题 |

**借鉴价值：**
- 提供了清晰的自动化程度分级标准
- 将 LLM 能力与结构化数据深度集成
- 支持从简单到复杂的渐进式自动化

---

### 2.3 AIP Evals（评估框架）

AIP Evals 是 Foundry 的 LLM 函数质量评估框架：

- **目标**：确保 LLM 函数的输出质量和一致性
- **方式**：通过测试用例、评估指标、黄金标准进行自动化评估
- **价值**：为 LLM 函数提供持续的质量监控

**借鉴价值：**
- workflow-design.md 的 decision nodes 可以引入类似的评估机制
- 为从文档抽取的 Workflow 建立质量保证体系

---

### 2.4 Automate（自动化触发器）

Automate 是 Foundry 的业务自动化产品：

- **定义**：通过定义持续或定期检查的条件，在满足条件时自动执行效果
- **触发方式**：
  - **时间条件**：按时间调度（每天、每周、CRON）
  - **对象条件**：基于 Ontology 对象的状态变化
- **执行效果**：
  - Foundry Actions
  - AIP Logic 函数
  - 平台/邮件通知
- **使用场景**：
  - 自动异常处理
  - 工作流自动化
  - 持续监控与响应

**对比 workflow-design.md：**

| 维度 | Automate | workflow-design.md |
|------|----------|-------------------|
| **触发机制** | 时间 + 对象状态变化 | 主动调用 + 事件触发 |
| **执行方式** | 后台持续监控 | 单次执行 |
| **人工介入** | 通过审批 action | 显式 human_input 节点 |

---

## 3. 与 workflow-design.md 的深度对比

### 3.1 整体架构对比

#### 3.1.1 表达模型

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **流程表达** | Actions（声明式） | DAG + 状态机 |
| **规则表达** | 嵌入 Actions 的 Rules | 独立的 decision.* Nodes |
| **数据模型** | Ontology 对象 + 属性 + 链接 | Node I/O Schema + 上下文 |
| **版本管理** | Action Types 版本化 | Workflow Definition 版本化 |

**分析：**
- workflow-design.md 的 DAG 模型更适合表达**多步骤复杂流程**
- Foundry 的 Actions 模型更适合表达**单次事务性操作**
- 两者都强调**声明式**而非命令式

#### 3.1.2 执行模型

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **执行边界** | 单个 Action 事务 | 整个 Workflow 实例 |
| **状态管理** | Ontology 对象状态 | Workflow Execution 状态 |
| **并发控制** | 平台事务管理 | DAG 调度器 + 节点编排 |
| **错误处理** | Action 级重试 + 回滚 | Node 级策略 + 人工介入 |

**分析：**
- workflow-design.md 的**执行前静态校验**是显著优势
- Foundry 的**事务性**更强（原子性保证）
- workflow-design.md 的**人工介入**更明确（显式节点）

---

### 3.2 核心概念映射

#### 3.2.1 Node vs Action Rule

**workflow-design.md Node：**
```
Node {
  "type": "ontology.identify_substance",
  "inputs": { "substance_name": "$.inputs.substance_name" },
  "outputs": { "cas_number", "is_hazardous" },
  "policy": { "risk_level": "low" }
}
```

**Palantir Foundry Rule：**
```
Rule {
  "type": "Create object",
  "objectType": "Substance",
  "properties": {
    "name": "$param.substanceName",
    "casNumber": "$lookup.casNumber"
  }
}
```

**对比分析：**

| 特性 | Node | Action Rule |
|------|------|-------------|
| **输入来源** | Workflow 上下文引用 | Action Parameters |
| **输出方式** | 结构化 outputs | 修改 Ontology 对象 |
| **治理声明** | 显式 policy 字段 | 隐式集成平台安全 |
| **复用性** | 跨 Workflow 复用 | 跨 Action 复用（通过 Functions） |
| **可测试性** | 独立单元测试 | 通过 AIP Evals |

**结论：**
- Node 的**输入引用模型**更灵活（支持跨节点引用）
- Action Rule 的**事务性**更强（原子性保证）
- 两者都支持**复用**，但机制不同

#### 3.2.2 Decision Node vs AIP Logic

**workflow-design.md Decision Node：**
```
decision.check_transport_mode {
  "inputs_schema": {
    "hazard_level": { "type": "integer" },
    "transport_mode": { "type": "string" }
  },
  "outputs_schema": {
    "allowed": { "type": "boolean" },
    "required_checks": { "type": "array" },
    "reason_code": { "type": "string" },
    "evidence": { ... }
  }
}
```

**Palantir AIP Logic：**
- 无代码 UI 编辑器
- 直接连接 LLM
- 输出可直接写入 Ontology 对象

**对比分析：**

| 特性 | Decision Node | AIP Logic |
|------|---------------|-----------|
| **实现方式** | 代码 + Schema | 无代码 UI + LLM |
| **输入模式** | 结构化 inputs | 支持非结构化输入 |
| **输出模式** | 结构化 outputs | 任意（含对象修改） |
| **治理** | 显式 policy（risk_level） | 隐式平台权限 |
| **质量保证** | （未定义） | AIP Evals 评估框架 |
| **适用场景** | 确定性规则 | 不确定性、推理任务 |

**借鉴价值：**
- AIP Logic 的**无代码编辑器**可以降低决策节点编写门槛
- AIP Evals 的**评估框架**可以引入 decision nodes
- Decision Node 的**evidence 溯源**更精细

---

### 3.3 人工介入对比

#### 3.3.1 workflow-design.md 的显式人工节点

```json
{
  "nodes": {
    "human_input_1": {
      "type": "human_input",
      "requested_fields": {
        "amount": { "type": "number" }
      }
    },
    "human_approval": {
      "type": "human_approval",
      "approval_required": true
    }
  }
}
```

**特点：**
- 人工节点是**一等语义**
- 在 DSL 中**显式声明**
- 支持补参（human_input）和审批（human_approval）
- 状态机明确：`running → waiting_input → running`

#### 3.3.2 Foundry 的人工介入机制

**方式 1：Agent Studio**
- 通过交互式对话收集用户输入
- LLM 主导，人机协作
- 不需要在流程中显式声明

**方式 2：Automate 审批 Actions**
- 创建审批类型的 Action
- 由用户手动触发或 Automate 触发
- 集成平台权限体系

**对比分析：**

| 维度 | workflow-design.md | Foundry |
|------|-------------------|---------|
| **声明方式** | 显式节点（JSON DSL） | 隐式（Agent 对话）或 Action |
| **状态可见性** | 高（状态机明确） | 中（依赖用户交互） |
| **审计性** | 强（执行历史记录） | 强（平台审计日志） |
| **灵活性** | 中（预先定义） | 高（动态对话） |
| **适用场景** | 合规、严苛场景 | 协作、灵活场景 |

**结论：**
- workflow-design.md 的**显式声明**更适合**合规/审计**场景
- Foundry 的**对话式**更适合**协作/灵活**场景
- 两者不矛盾，可以互补

---

### 3.4 治理与安全对比

#### 3.4.1 workflow-design.md 的治理体系

**Workflow 级策略：**
```json
{
  "execution_policy": {
    "risk_level": "high",
    "allow_auto": false,
    "require_approval": true
  }
}
```

**Node 级策略：**
```json
{
  "policy": {
    "risk_level": "medium",
    "idempotent": false
  }
}
```

**特点：**
- **执行前置校验**：在执行前即可完整校验
- **声明式策略**：所有治理逻辑在 DSL 中显式声明
- **分层治理**：Workflow 级 + Node 级双层
- **静态分析**：支持 DAG 检测、I/O 校验、分支校验

#### 3.4.2 Foundry 的治理体系

**Platform-wide Security：**
- 集成用户权限和函数权限
- Action Types 的访问控制
- Ontology 对象的行级权限
- 审计日志全面记录

**特点：**
- **运行时治理**：通过平台权限体系拦截
- **细粒度权限**：用户 + 函数 + 对象三层
- **隐式保证**：治理逻辑嵌入平台能力中

**对比分析：**

| 维度 | workflow-design.md | Foundry |
|------|-------------------|---------|
| **治理时机** | 执行前 + 执行中 | 执行中（运行时） |
| **策略表达** | 显式（DSL） | 隐式（平台） |
| **风险分级** | 显式 risk_level | 隐式（权限体系） |
| **静态分析** | 强（前置校验） | 弱（运行时拦截） |
| **审计能力** | 强（Execution trace） | 强（平台审计日志） |

**借鉴价值：**
- workflow-design.md 的**前置校验**是显著优势
- Foundry 的**细粒度权限**可以参考
- 两者都强调**审计可回放**

---

## 4. 借鉴建议

### 4.1 短期借鉴（1-2个月内可实施）

#### 4.1.1 引入 AIP Evals 评估框架

**目标：** 为 decision nodes 和抽取类节点建立质量保证

**实施建议：**
```
Node Schema 扩展：
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

**价值：**
- 为从文档抽取的规则建立自动化测试
- 降低人工校验的成本

#### 4.1.2 扩展 Node Catalog 的分类

借鉴 Foundry 的分类思路：

| Foundry 分类 | workflow-design.md 对应 | 建议 |
|--------------|----------------------|------|
| Ontology Rules | ontology.* Nodes | 增加 `ontology` 标签 |
| Other Rules | io.* Nodes | 增加 `io` 标签 |
| Functions | utility.* Nodes | 增加 `utility` 标签 |
| AIP Logic | （缺失） | 新增 `llm` 分类 |

#### 4.1.3 建立 Platform-wide Observability

借鉴 Foundry 的统一可观测性：

```
Observability Dashboard：
├── Workflow Metrics
│   ├── 成功率 / 失败率
│   ├── 人工介入率
│   ├── 平均执行时长
├── Node Metrics
│   ├── 各 Node 的调用次数
│   ├── 各 Node 的错误率
│   ├── 各 Node 的平均耗时
└── Audit Trail
    ├── 执行历史
    ├── 人工操作记录
    ├── 错误追踪
```

---

### 4.2 中期借鉴（3-6个月）

#### 4.2.1 引入无代码 Logic 编辑器

**目标：** 降低 decision nodes 和复杂节点的编写门槛

**技术方案：**
```
UI 编辑器 → JSON DSL → 校验 → 注册到 Catalog
```

**功能：**
- 可视化输入输出 Schema 定义
- 可视化规则编写（支持 LLM 辅助）
- 一键注册到 Node Catalog
- 内置 AIP Evals 测试

#### 4.2.2 建立 Automate 类触发器

**目标：** 支持事件驱动的 Workflow 执行

**技术方案：**
```
Trigger 定义：
{
  "workflow_id": "hazard_compliance",
  "trigger_type": "object_changed",
  "condition": {
    "object_type": "Shipment",
    "field": "status",
    "from": "draft",
    "to": "submitted"
  },
  "action": "execute_workflow"
}
```

**价值：**
- 支持持续监控场景
- 扩大 Workflow 的应用范围

#### 4.2.3 引入四层自动化模型

**目标：** 建立 Workflow 自动化程度的分级标准

**定义：**
| 层级 | 描述 | 示例 |
|------|------|------|
| **即时分析** | 单次执行，人工决策 | 合规判定（需审批） |
| **任务代理** | 自动执行特定任务 | 数据抽取 |
| **流程代理** | 自动执行多步流程 | 完整合规检查 |
| **工作代理** | 全自动，无人值守 | 批量合规审核 |

**价值：**
- 为 Workflow 设计提供清晰的目标
- 支持渐进式自动化演进

---

### 4.3 长期借鉴（6个月以上）

#### 4.3.1 建立 Ontology 深度集成

**目标：** 将 Workflow 与 Ontology 深度集成

**技术方案：**
```
Workflow ↔ Ontology 双向绑定：
├── Workflow 输入/输出直接映射到 Ontology 对象
├── Node Schema 引用 Ontology 类型定义
├── Workflow 执行结果自动写入 Ontology
└── Ontology 变化自动触发 Workflow
```

**价值：**
- 提升数据一致性
- 减少数据转换成本
- 支持跨 Workflow 的语义统一

#### 4.3.2 建立 Agent Studio 类能力

**目标：** 支持基于 LLM 的交互式代理

**技术方案：**
```
Agent Studio：
├── 基于 LLM 的对话引擎
├── 集成 Workflow 执行能力
├── 集成 Ontology 数据查询
├── 支持文档检索（RAG）
└── 显式的人工介入控制
```

**价值：**
- 提升用户体验
- 降低 Workflow 使用门槛
- 支持复杂的多轮交互场景

---

## 5. 关键差异总结

### 5.1 核心定位差异

| 维度 | Palantir Foundry | workflow-design.md |
|------|------------------|-------------------|
| **核心输入** | 结构化数据（Ontology） | 非结构化文档 |
| **核心输出** | 对象修改 + 数据联动 | 结构化判定 + 证据链 |
| **主要场景** | 数据操作 + 协作 | 合规判定 + 审计 |
| **自动化程度** | 中（强调人机协作） | 高（强调自动执行） |

### 5.2 核心优势对比

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
- ✅ 精细的 evidence 溯源

---

## 6. 结论

### 6.1 总体评价

Palantir Foundry 是一个成熟的**数据平台级自动化能力**，专注于**结构化数据操作和人机协作**。

workflow-design.md 是一个创新的**文档驱动自动化能力**，专注于**非结构化文档流程抽取和合规审计**。

两者在**核心定位**上不同，但在**技术实现**上有大量可互相借鉴的地方。

### 6.2 关键建议

**优先级 1（必做）：**
1. 引入 **AIP Evals 评估框架**，为 decision nodes 建立质量保证
2. 建立 **Platform-wide Observability**，统一监控与审计
3. 完善 **Node Catalog 分类**，建立清晰的领域划分

**优先级 2（应该做）：**
1. 引入 **无代码 Logic 编辑器**，降低节点编写门槛
2. 建立 **Automate 类触发器**，支持事件驱动执行
3. 定义 **四层自动化模型**，为设计提供目标

**优先级 3（长期）：**
1. 建立 **Ontology 深度集成**，提升数据一致性
2. 引入 **Agent Studio 类能力**，支持复杂交互

### 6.3 最终定位

**不要试图成为 Palantir Foundry 的竞品。**

**应该定位为：**
- Foundry 的**互补能力**（非结构化 → 结构化）
- 合规/审计领域的**垂直解决方案**
- Agent 驱动的 **Workflow 编排平台**

**核心差异化：**
- 文档 → Workflow 的**自动生成**
- 执行前的**静态校验与治理**
- 显式的**人工介入与审计语义**

---

## 7. 附录：参考资料

### 7.1 Palantir Foundry 官方文档

- Foundry 概览：https://www.palantir.com/docs/foundry/
- AIP 概览：https://www.palantir.com/docs/foundry/aip/overview/
- Ontology 概览：https://www.palantir.com/docs/foundry/ontology/overview/
- Functions 概览：https://www.palantir.com/docs/foundry/functions/overview/
- Action Types 概览：https://www.palantir.com/docs/foundry/action-types/overview/
- Logic 概览：https://www.palantir.com/docs/foundry/logic/overview/
- Automate 概览：https://www.palantir.com/docs/foundry/automate/overview/
- Agent Studio 概览：https://www.palantir.com/docs/foundry/agent-studio/overview/

### 7.2 workflow-design.md 核心章节

- 第 2 章：术语与边界
- 第 3 章：端到端场景示例（危险品运输）
- 第 4 章：Workflow DSL 与整体结构设计
- 第 5 章：Node 设计
- 第 6 章：Workflow 与 Node 的生命周期

---

**报告版本：** v1.0
**作者：** OpenClaw Agent
**日期：** 2026-02-05
