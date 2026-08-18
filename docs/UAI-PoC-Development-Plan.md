# Universal Articulation Intelligence — PoC 最小闭环开发计划书

版本: v0.1 (Draft)
文档状态: PoC 实装计划阶段 — 定义 MVP 最小闭环的开发范围、任务分解、里程碑、退出准则、人员/依赖假设
作者: AI Research / Architecture (assisted)
日期: 2026-08-18
文档标准: 参照日本 IPA 共通フレーム2013（SLCP-JCF2013）之「開発プロセス」惯例编制（開発計画書一般构成：目的/範囲/体制/WBS/スケジュール/成果物/リスク）

上游依据文档：`UAI-REQ-001`（需求定义书 v0.3）、`UAI-BD-001`（基本设计书 v0.3）、`UAI-DD-001`（详细设计书 v0.3）。本文档不重新定义需求或设计，只定义"如何在最短闭环内把已确定的设计落地为可运行代码并验证"。

---

## 0. 文档管理信息

### 0.1 改订履历

| 版本 | 日期 | 变更内容 | 作成者 |
|---|---|---|---|
| v0.1 | 2026-08-18 | 初版发布 | AI Research / Architecture |

### 0.2 承认体系

| 角色 | 姓名 | 承认状态 | 日期 |
|---|---|---|---|
| 项目负责人 | （待指定） | 未承认 | — |
| 技术负责人 | （待指定） | 未承认 | — |

### 0.3 参照文档一览

| 文档 ID | 文档名 | 关系 |
|---|---|---|
| UAI-REQ-001 | 需求定义书 v0.3 | 依据 |
| UAI-BD-001 | 基本设计书 v0.3 | 依据 |
| UAI-DD-001 | 详细设计书 v0.3 | 依据 |
| UAI-MOCK-001 | 《UAI Mock 测试项目设计书》v0.1（本次同批产出） | 并行依据——PoC 验证载体的设计规格 |
| UAI-DEPLOY-001 | 《UAI 部署方案书》v0.1（本次同批产出） | 并行依据——PoC 运行环境的部署规格 |

---

## 1. 目的与基本方针

本计划的目标不是实现需求定义书 §39 MVP 范围的全部功能，而是实现一个**更小的、可在最短时间内跑通并证明架构假设成立的闭环**（下称"最小闭环"）。最小闭环是 MVP 的真子集，二者关系为：

```
最小闭环 ⊂ MVP（需求 §39）⊂ V1 ⊂ Research
```

基本方针：
- **先跑通，再补全**：最小闭环阶段允许所有外部依赖（LLM/Physics/Vector/Graph）使用 Mock 实现（见 `UAI-MOCK-001`），只要 Rust Core 领域逻辑（模块 1-4, 6, 8, 10-11, 13）是真实实现。
- **不跳过风险验证点**：需求文档标注为核心研究风险的两项（R-01 LLM 可靠性、R-07 未知类别泛化）即使在最小闭环阶段也不得用"假装通过"的方式回避——Mock LLM 必须能模拟"失败/低置信度"分支，最小闭环必须包含至少一个非训练模板类别的测试资产。
- **单元与集成解耦**：模块内部逻辑（Geometry/Topology/Decomposition/Compiler）先各自独立可测试，再组装为端到端链路，避免"先端到端、后补单测"的债务积累。

---

## 2. 最小闭环范围定义（UAI-POC-SCOPE-*）

### 2.1 范围内

| 编号 | 内容 | 对应需求/设计 |
|---|---|---|
| UAI-POC-SCOPE-001 | 单一静态网格输入（内置 3-5 个测试资产，覆盖：人形、非人形已知类别、至少 1 个未知/虚构类别） | UAI-FR-DEC-004, KPI: Unknown Category Generalization |
| UAI-POC-SCOPE-002 | Geometry + Topology 特征抽取（真实实现，非 Mock） | UAI-BD-ARC-002/003, UAI-DD-MOD-002/003 |
| UAI-POC-SCOPE-003 | 层级化结构拆分 + 最小 Structural Grammar 规则子集（至少实现 Bottleneck Rule、Chain Rule、Hub Rule 三条，其余规则以"待启用"占位） | UAI-BD-ARC-004, UAI-DD-MOD-004 |
| UAI-POC-SCOPE-004 | Structural DSL 语法解析与确定性 Compiler（真实实现） | UAI-BD-ARC-006/008, UAI-DD-MOD-006/008 |
| UAI-POC-SCOPE-005 | Retrieval（Vector/Graph）—— **Mock 实现**，返回预置的相似案例，验证接口契约而非算法效果 | UAI-DD-001 §8.5 Provider 契约 |
| UAI-POC-SCOPE-006 | LLM Reasoning Adapter —— **Mock 实现**，支持三种预置行为模式：正常返回多假设、超时/失败（验证降级路径）、返回不可解析 DSL（验证 DslValidator） | UAI-DD-MOD-007, §5.2 降级路径 |
| UAI-POC-SCOPE-007 | Physics Validation —— **Mock 实现**，返回预置的 Penetration/Stretching/Joint Instability 分数（含至少一个"验证失败"用例，驱动假设重排序） | UAI-DD-MOD-009 |
| UAI-POC-SCOPE-008 | Confidence & Explanation（真实实现） | UAI-DD-MOD-010 |
| UAI-POC-SCOPE-009 | CLI 最小子集：`submit` / `status` / `result`（真实实现，无需 API Gateway 网络层，本地进程内调用即可） | UAI-DD-IF-004 |
| UAI-POC-SCOPE-010 | 端到端可追溯性：`ProcessingRun.stage_timeline` 真实记录 | UAI-DD-001 §3.7 |

### 2.2 明确排除（本轮不做）

- Human-in-the-loop 修正流程（模块 011）——闭环验证的是"AI 判断链路"，人工修正留到下一轮。
- Learning Loop / Pattern Mining（模块 012）——无足够 Experience Graph 数据支撑，无意义。
- 网络层 API Gateway（真实 HTTP/gRPC 服务）——CLI 直接进程内调用即可验证核心逻辑。
- 任何真实的 LLM/Physics/Vector/Graph Provider 接入——全部 Mock，真实接入是下一阶段任务（见 §7 后续路线）。
- DCC/RPA 集成层（详细设计书 §11.5）——明确标注为 Research/V1+，不在最小闭环内。

### 2.3 与需求 MVP 范围的差异说明

需求定义书 §39 MVP 范围本身已经是克制的范围，但仍要求"Graph/Vector Retrieval 最小子集"与"LLM Structural Hypothesis"为真实调用。最小闭环进一步收窄为全 Mock 外部依赖，原因：
1. 真实 Provider 选型/账号/密钥等属于 §6 部署方案范畴，会显著拉长本阶段周期且与架构验证目标无关；
2. 先验证"Provider 抽象接口契约是否好用"，比"先验证某个具体 LLM 输出质量"更符合当前阶段的架构验证目的；
3. 该收窄有明确退出路径（§7），不改变最终 MVP 范围本身。

---

## 3. 工作分解结构（WBS，UAI-POC-WBS-*）

| WBS | 任务 | 依赖 | 交付物 | 对应模块 |
|---|---|---|---|---|
| WBS-1 | 内部数据结构落地（InternalMesh / GeometryEvidence / TopologyEvidence / ArticulationHypothesis / ValidationReport / ProcessingRun，按 UAI-DD-001 §3 字段表） | 无 | 可编译的核心数据类型 | 全模块基础 |
| WBS-2 | Ingestion & Validation（模块 001） | WBS-1 | `ingest()` 可用，含完整性校验 | MOD-001 |
| WBS-3 | Geometry Feature Engine（模块 002） | WBS-1 | 曲率/厚度/伸长度/对称性/测地距离/中轴 抽取可用 | MOD-002 |
| WBS-4 | Topology Feature Engine（模块 003） | WBS-1 | 连通性/边界/分支/瓶颈评分/邻接图 可用 | MOD-003 |
| WBS-5 | Structural Decomposition Engine + 最小 Grammar 规则集（模块 004） | WBS-3, WBS-4 | 可产出 Structural Part Graph | MOD-004 |
| WBS-6 | Structural DSL 语法与 Validator（模块 006） | WBS-1 | DSL 文本可被解析与语义校验 | MOD-006 |
| WBS-7 | Rig Compiler（模块 008，四段编译链） | WBS-6 | DSL → Rig Hypothesis 可用，含编译失败路径 | MOD-008 |
| WBS-8 | Mock Provider 四件套（LLM/Physics/Vector/Graph） | WBS-1 | 见 `UAI-MOCK-001` | Provider 抽象层 |
| WBS-9 | Retrieval Coordinator（模块 005，接 Mock Vector/Graph） | WBS-8 | `retrieve()` 端到端可用 | MOD-005 |
| WBS-10 | LLM Reasoning Adapter（模块 007，接 Mock LLM） | WBS-6, WBS-8 | Evidence→DSL 假设 端到端可用，含降级路径 | MOD-007 |
| WBS-11 | Physics Validation Coordinator（模块 009，接 Mock Physics） | WBS-7, WBS-8 | ValidationReport 产出可用 | MOD-009 |
| WBS-12 | Confidence & Explanation Service（模块 010） | WBS-9, WBS-10, WBS-11 | 最终 Confidence/Evidence/Alternative 输出 | MOD-010 |
| WBS-13 | CLI 最小子集（submit/status/result） | WBS-2, WBS-12 | 可从命令行跑通全链路 | MOD-013（子集） |
| WBS-14 | 端到端集成测试与回归用例（含 UAI-BD-TEST-002 反例集） | 全部 | 测试报告 + CI 集成 | 跨模块 |
| WBS-15 | 最小闭环验收演练（对照 §5 退出准则逐条验证） | WBS-14 | 验收记录 | — |

### 3.1 建议执行顺序（非严格串行，标注可并行段）

```
阶段 A（可并行）：WBS-1 → { WBS-2, WBS-3, WBS-4, WBS-8 }
阶段 B：{ WBS-3, WBS-4 } → WBS-5
阶段 B'（可并行于阶段 B）：WBS-6 → WBS-7
阶段 C：{ WBS-8, WBS-6 } → WBS-9, WBS-10（可并行）
阶段 C'：{ WBS-7, WBS-8 } → WBS-11
阶段 D：{ WBS-9, WBS-10, WBS-11 } → WBS-12
阶段 E：{ WBS-2, WBS-12 } → WBS-13 → WBS-14 → WBS-15
```

---

## 4. 里程碑（UAI-POC-MS-*）

| 里程碑 | 达成标志 | 依赖 WBS |
|---|---|---|
| M1 单元级基础可用 | WBS-1~4 全部完成，Geometry/Topology 单元测试通过（对应 UAI-BD-TEST-001） | WBS-1~4 |
| M2 结构拆分可用 | 可对至少 3 个测试资产产出非空 Structural Part Graph | WBS-5 |
| M3 DSL/Compiler 闭环 | 可手写 DSL 文本，经 Validator+Compiler 产出 Rig Hypothesis，含至少一个编译失败用例验证 | WBS-6, WBS-7 |
| M4 Mock Provider 就绪 | 四类 Mock Provider 均实现 §8.5 契约，含正常/失败/边界三类响应模式 | WBS-8 |
| M5 端到端链路打通 | CLI `submit` 到 `result` 可返回含 Confidence 的完整结果（对应需求 AC-08） | WBS-9~13 |
| M6 最小闭环验收 | §5 全部退出准则通过，含反例测试集（UAI-FR-TOP-006 相关） | WBS-14, WBS-15 |

里程碑之间不设固定日历日期（本文档不承诺具体交付日期，由实际执行团队在启动时补充排期）；但要求严格保持 M1→M6 的依赖顺序，不允许在 M2 未达成前开始 WBS-9/10/11 的集成联调，以避免过早引入跨模块调试成本。

---

## 5. 退出准则（Exit Criteria，UAI-POC-EXIT-*）

最小闭环视为完成，须同时满足：

| 编号 | 准则 | 验证方式 |
|---|---|---|
| UAI-POC-EXIT-001 | 端到端链路可对全部内置测试资产（含未知类别）产出非空结果，不崩溃、不挂起 | 自动化集成测试 |
| UAI-POC-EXIT-002 | 至少 1 个"高 Bottleneck 但非关节"反例资产，其最终 Confidence 明显低于真实关节案例（UAI-BD-TEST-002） | 回归测试用例 + 人工复核 |
| UAI-POC-EXIT-003 | Mock LLM 返回失败/超时时，链路进入降级路径而非崩溃，`degraded_mode_flags` 正确记录 | 集成测试 |
| UAI-POC-EXIT-004 | Mock Physics 返回"验证失败"时，能观察到假设置信度相应降低（排序变化） | 集成测试 |
| UAI-POC-EXIT-005 | DSL 编译失败用例不产出部分 Rig，返回结构化 `CompileError` | 单元测试 |
| UAI-POC-EXIT-006 | `ProcessingRun.stage_timeline` 完整记录全部阶段（1-10 中实际参与的阶段），可用于事后追溯 | 人工检查一次真实运行输出 |
| UAI-POC-EXIT-007 | Geometry/Topology 模块可独立于 Decomposition 单独运行单元测试（对应 UAI-BD-TEST-001） | CI 测试报告 |
| UAI-POC-EXIT-008 | 全部 Mock Provider 严格实现 `UAI-MOCK-001` 定义的契约，替换为真实 Provider 时核心模块代码不需要修改（仅配置切换） | 架构评审 + 接口测试 |

不满足任一项，最小闭环不视为完成，不得进入 §7 下一阶段。

---

## 6. 团队与资源假设（UAI-POC-RES-*）

> 本节为规划假设，非组织承诺；实际人员由执行团队按此结构对照调整。

| 角色 | 建议人数 | 主要职责 |
|---|---|---|
| Rust Core 工程师 | 2-3 | WBS-1~7, 9, 11~13 |
| Mock/测试基础设施工程师 | 1 | WBS-8, WBS-14（可与 Rust Core 工程师合并） |
| 架构/技术负责人 | 1（兼职） | 里程碑评审、退出准则裁决、跨模块契约仲裁 |
| QA | 1（可兼职） | WBS-14/15，回归用例设计 |

---

## 7. 后续路线（超出最小闭环范围，仅作路线标注，非本计划承诺范围）

最小闭环完成后，建议按以下顺序推进（具体范围与验收标准留待各自阶段的计划文档）：

1. 真实 Provider 逐一替换 Mock（建议顺序：Vector/Graph → LLM → Physics，从低风险到高风险）。
2. 补全需求 §39 MVP 范围中被本轮排除的部分（Human-in-the-loop、Learning Loop、API Gateway 网络层）。
3. 补全需求定义书附录 A 中标注为 V1 的输入/输出类型。
4. 视核心研究风险（R-01/R-07，需求 §41）验证结果决定是否继续投入详细设计书 §11.5 的 DCC/RPA 集成层。

---

## 8. 风险（承接需求 §41，聚焦本阶段新增风险）

| 编号 | 风险 | 应对 |
|---|---|---|
| UAI-POC-RISK-001 | Mock Provider 契约与未来真实 Provider 实际行为差异过大，导致"Mock 通过但真实接入失败" | §5 EXIT-008 强制要求契约一致性架构评审；Mock 行为模式须覆盖正常/失败/边界三类，不能只覆盖"理想路径" |
| UAI-POC-RISK-002 | 为求"跑通"而弱化 UAI-FR-TOP-006（Bottleneck≠Joint）约束，用简化逻辑直接判定 | EXIT-002 强制反例测试作为验收门槛，不可跳过 |
| UAI-POC-RISK-003 | 最小闭环范围被隐性放大（Scope Creep），实际做成了完整 MVP 才交付 | §2.2 明确排除清单具有一票否决效力，评审时逐条核对 |

---

## 9. 自审

| 检查项 | 状态 | 说明 |
|---|---|---|
| 范围是否严格小于需求 §39 MVP | ✅ | §2.1/§2.2 明确排除模块 011/012/网络层/真实 Provider |
| 是否规避了核心研究风险验证 | ✅ 未规避 | §5 EXIT-002/003/004 强制覆盖 R-01/R-02 相关验证点 |
| WBS 依赖关系是否可执行（无循环依赖） | ✅ | §3.1 依赖图为有向无环 |
| 退出准则是否可客观判定 | ✅ | 全部准则均可通过自动化测试或明确人工检查动作验证 |
| 是否给出不可执行的具体日期承诺 | ✅ 未给出 | §4 明确不设日历日期，避免规划不确定性伪装成承诺 |

---

*本文档为 PoC 阶段开发计划，与 `UAI-MOCK-001`（Mock 测试项目设计书）、`UAI-DEPLOY-001`（部署方案书）配套使用。*
