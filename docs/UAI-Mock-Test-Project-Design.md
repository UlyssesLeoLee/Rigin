# UAI Mock 测试项目设计书（UAI-MOCK-001）

版本: v0.1 (Draft)
文档状态: PoC 配套项目设计 — 定义"Mock 测试项目"（为验证最小闭环而构建的 Provider 模拟层与测试资产集）的需求/基本设计/详细设计三级内容
作者: AI Research / Architecture (assisted)
日期: 2026-08-18
文档标准: 参照 IPA 共通フレーム2013 惯例，采用与正式三级文档（`UAI-REQ-001`/`UAI-BD-001`/`UAI-DD-001`）一致的编号与追踪方法，但因这是一个**范围受限的配套验证项目**（而非产品本体），三级内容合并为一份文档，以 Part A/B/C 区分层级。

上游依据：`UAI-REQ-001` v0.3、`UAI-BD-001` v0.3、`UAI-DD-001` v0.3、`UAI-PoC-Development-Plan.md`（UAI-POC-001，定义本项目存在的目的——最小闭环验证）。

---

## 0. 文档管理信息

### 0.1 改订履历

| 版本 | 日期 | 变更内容 | 作成者 |
|---|---|---|---|
| v0.1 | 2026-08-18 | 初版发布 | AI Research / Architecture |

### 0.2 定位说明

Mock 测试项目**不是**产品功能的一部分，不会随产品发布。它的唯一目的是：在真实 LLM/Physics/Vector/Graph Provider 尚未接入之前，让 UAI 核心闭环（详细设计书 §5.1 主时序）可以被端到端运行、验证与回归测试。所有 Mock 组件在真实 Provider 接入后应被弃用而非长期维护为生产路径。

---

# Part A. 需求（对应 UAI-REQ-001 层级）

## A.1 目的

为 PoC 最小闭环开发计划（`UAI-POC-001`）提供：
1. 四类 Provider（LLM/Physics/Vector/Graph）的确定性、可配置 Mock 实现；
2. 一组覆盖已知类别、未知类别、反例（高 Bottleneck 非关节）的最小测试资产集；
3. 一套将 PoC 计划书 §5 退出准则转化为可运行断言的集成测试规格。

## A.2 需求边界

**范围内**：Mock Provider 的行为契约定义、测试资产规格、断言规格。
**范围外**：Mock Provider 的具体实现代码（属于 Part C 详细设计与后续实装阶段）、真实 Provider 的选型与接入（不在本文档范围，属于 PoC 计划书 §7 后续路线）。

## A.3 功能需求

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-MOCK-FR-001 | LLM Mock 三模式 | Mock LLM 须支持「正常返回多假设」「超时/失败」「返回不可解析 DSL」三种可配置响应模式，覆盖 UAI-DD-001 §5.2 降级路径全部触发条件 | MUST |
| UAI-MOCK-FR-002 | Physics Mock 可控分数 | Mock Physics 须支持按测试用例预置 Penetration/Stretching/Joint Instability 分数，包括至少一个"验证失败"用例 | MUST |
| UAI-MOCK-FR-003 | Vector/Graph Mock 预置相似案例 | Mock Vector/Graph 须返回预置的 Top-K 相似结构候选，用于验证 §8.5 接口契约而非算法效果 | MUST |
| UAI-MOCK-FR-004 | Provider 契约一致性 | 全部 Mock 实现须严格实现 UAI-DD-001 §8.5 定义的接口签名，替换为真实 Provider 时调用方代码不需修改 | MUST |
| UAI-MOCK-FR-005 | 测试资产集 | 须提供至少 5 个静态网格测试资产：≥1 个人形类、≥1 个非人形已知类（如四足/机械臂类）、≥1 个未知/虚构类别、≥1 个用于反例验证的"高瓶颈非关节"专用构造资产（如哑铃形状的单一刚体） | MUST |
| UAI-MOCK-FR-006 | 断言规格覆盖退出准则 | 每条 PoC 计划书 §5 退出准则须对应至少一个可自动执行的断言 | MUST |

## A.4 验收标准

- 全部 Mock Provider 可在无网络、无真实外部服务的环境下独立运行。
- 测试资产集含元数据标注（类别、是否为反例、期望的定性结果方向），供断言比对使用，元数据本身不作为 AI 输入。
- Mock 行为模式可通过配置切换，不需要重新编译测试代码本体（复用部署方案书 §5.1 的配置切换约束）。

---

# Part B. 基本设计（对应 UAI-BD-001 层级）

## B.1 系统构成

```
tests/mock-harness/
  ├── mock_llm/          Mock LLM Provider 实现
  ├── mock_physics/       Mock Physics Provider 实现
  ├── mock_vector/         Mock Vector Store 实现
  ├── mock_graph/           Mock Graph Store 实现
  ├── fixtures/              测试资产集 + 元数据
  └── scenarios/              §A.4 断言规格对应的集成测试场景
```

## B.2 模块设计

| 设计 ID | 模块 | 职责 | 对应 Part A |
|---|---|---|---|
| UAI-MOCK-BD-001 | MockLlmProvider | 实现 UAI-DD-001 §8.5 LLM Provider 接口；内部维护「场景 → 响应」查找表，按测试用例 ID 或输入证据摘要哈希匹配预置响应 | UAI-MOCK-FR-001 |
| UAI-MOCK-BD-002 | MockPhysicsProvider | 实现 Physics Provider 接口；接收 Rig Hypothesis 引用，返回按测试用例预置的 ValidationMetrics | UAI-MOCK-FR-002 |
| UAI-MOCK-BD-003 | MockVectorStore / MockGraphStore | 实现 Vector/Graph Provider 接口；返回固定的 Top-K 候选列表（不做真实相似度计算） | UAI-MOCK-FR-003 |
| UAI-MOCK-BD-004 | FixtureCatalog | 测试资产的加载与元数据管理，供 scenarios 引用 | UAI-MOCK-FR-005 |
| UAI-MOCK-BD-005 | ScenarioRunner | 驱动"资产 + Mock 配置 → 运行 CLI 最小子集 → 断言输出"的集成测试执行 | UAI-MOCK-FR-006 |

## B.3 与产品核心模块的接口关系

Mock 模块不修改、不依赖 UAI 核心模块（UAI-BD-ARC-001~013）的内部实现，只作为 §7.5 Provider 抽象接口的一种实现体注入。核心模块通过配置（部署方案书 §5.1）决定装配 Mock 还是未来的真实 Provider，二者对核心模块而言类型等价。

## B.4 测试资产规格

| 资产 ID | 类别标注 | 用途 |
|---|---|---|
| FIX-001 | 人形（已知类别） | 验证常规分层拆分与对称性检测 |
| FIX-002 | 四足动物（已知类别） | 验证非人形模板下的拆分泛化 |
| FIX-003 | 未知/虚构类别（如多足混合结构） | 对应需求 KPI: Unknown Category Generalization，验证 UAI-FR-DEC-003/004 |
| FIX-004 | 哑铃形单一刚体（人工构造反例） | 高 Topological Bottleneck 但物理上不可动——对应 UAI-FR-TOP-006、AC-02、PoC EXIT-002 |
| FIX-005 | 简单铰接双段结构（正例） | 作为 FIX-004 反例的对照组，验证真实关节应获得更高置信度 |

---

# Part C. 详细设计（对应 UAI-DD-001 层级）

## C.1 MockLlmProvider 内部设计

字段级配置：

| 字段 | 类型 | 说明 |
|---|---|---|
| scenario_id | String | 测试场景标识，用于查表 |
| mode | Enum{Normal, Timeout, UnparsableDsl} | 响应模式 |
| preset_hypotheses | List<DslCandidate> \| null | `mode=Normal` 时的预置 DSL 候选（含主假设与备选假设，落实 UAI-FR-AI-003） |
| preset_self_confidence | Float32 \| null | 预置的 LLM 自报置信度 |

处理契约：
```
invoke(evidence_summary, timeout_ms) -> Result<List<DslCandidate>, LlmError>
  按 scenario_id 查表：
    mode=Normal        → 立即返回 preset_hypotheses
    mode=Timeout        → 阻塞至 timeout_ms 后返回 LlmError::Timeout
    mode=UnparsableDsl  → 返回语法非法的 DSL 文本，交由 DslValidator 触发解析失败
```

## C.2 MockPhysicsProvider 内部设计

字段级配置：

| 字段 | 类型 | 说明 |
|---|---|---|
| scenario_id | String | — |
| preset_penetration | Bool | — |
| preset_stretching_score | Float32 | — |
| preset_joint_instability_score | Float32 | — |
| preset_overall_pass | Bool | — |

处理契约：
```
simulate(rig_hypothesis, motion_probes, timeout_ms) -> Result<ValidationMetrics, PhysicsError>
  按 scenario_id 查表，直接返回预置 ValidationMetrics，不执行真实仿真
```

## C.3 MockVectorStore / MockGraphStore 内部设计

```
query(vector, top_k) -> List<{ref, score}>
  返回按 scenario_id 预置的固定候选列表（长度可小于 top_k，用于验证候选不足时的下游处理）

match_pattern(query_subgraph) -> List<{matched_subgraph_ref, score}>
  返回预置匹配结果
```

## C.4 ScenarioRunner 处理时序

```
for each scenario in scenarios/:
    load fixture (Part B.4) + mock provider 配置 (C.1~C.3)
    调用 CLI 最小子集：submit(fixture) → status(run_id) 轮询至终态 → result(run_id)
    对照 §A.4 / PoC 计划书 §5 退出准则执行断言：
        - EXIT-001: result 非空且 run 状态非 Failed
        - EXIT-002: FIX-004 的 final_confidence 显著低于 FIX-005 同类关节候选
        - EXIT-003: mode=Timeout 场景下 degraded_mode_flags 含 LlmDegraded
        - EXIT-004: preset_overall_pass=false 场景下对应假设 rank 靠后
        - EXIT-005: mode=UnparsableDsl 场景下返回 CompileError，不产出部分 Rig
        - EXIT-006: stage_timeline 覆盖全部实际执行阶段
    记录断言结果 → 测试报告
```

## C.5 错误码与断言失败的区分

Mock 测试项目自身的断言失败（测试不通过）须与 UAI 核心的运行时错误码（`UAI-ERRC-*`，详细设计书 §9）在报告中分层展示：断言失败属于"测试项目层"，`UAI-ERRC-*` 属于"被测系统层"——一个场景可能是"系统正确返回了预期的 `UAI-ERRC-2002`（DSL 编译失败），因此测试断言通过"，避免二者混淆。

---

## 自审

| 检查项 | 状态 | 说明 |
|---|---|---|
| Mock 项目是否被误设计为长期生产路径 | ✅ 未误设计 | §0.2 明确定位为配套验证项目，真实接入后应弃用 |
| Mock 实现是否可能弱化 Bottleneck≠Joint 约束的验证 | ✅ 未弱化 | FIX-004/FIX-005 对照组 + EXIT-002 断言直接承接该约束 |
| Provider 契约是否与 UAI-DD-001 §8.5 一致 | ✅ 一致 | Part C 全部处理契约签名对齐 §8.5 |
| 断言是否覆盖 PoC 计划书全部退出准则 | ✅ 覆盖 EXIT-001~006 | EXIT-007/008 为架构性准则（单测独立性、契约一致性），由 CI 结构与代码评审保证，非本项目断言职责 |

---

*本文档与 `UAI-PoC-Development-Plan.md`、`UAI-Deployment-Plan.md` 配套使用，共同构成最小闭环 PoC 的完整可执行依据。*
