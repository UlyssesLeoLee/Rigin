# Universal Articulation Intelligence — 基本设计书

版本: v0.3 (Draft)
文档状态: 基本设计阶段（Basic Design / 外部設計）— 定义系统构成、模块职责、数据概念模型、接口规格与处理流程；**禁止**在本文档中包含：具体实现代码、类图/详细类设计、数据库物理表设计（DDL）、具体第三方库版本锁定
作者: AI Research / Architecture (assisted)
日期: 2026-08-18
文档标准: 参照日本 IPA（情報処理推進機構）共通フレーム2013（SLCP-JCF2013）之「基本設計プロセス（外部設計）」惯例编制，章节结构对齐典型基本设计书（システム構成／機能設計／データ設計／インターフェース設計／非機能設計／移行設計／テスト方針）

上游依据文档：`UAI-REQ-001`（《Universal Articulation Intelligence 需求定义书》v0.3）。本文档中所有设计项均以设计 ID（`UAI-BD-*`）标注，并在 §17 追踪矩阵中与需求 ID 对应。

---

## 0. 文档管理信息

### 0.1 改订履历

| 版本 | 日期 | 变更内容 | 作成者 |
|---|---|---|---|
| v0.1 | 2026-08-18 | 初版发布 | AI Research / Architecture |
| v0.2 | 2026-08-18 | 三级文档交叉审核修订：(1) §17 追踪矩阵拆分 UAI-API-001~004 与 UAI-API-005（DCC 集成预留），避免过度声称本文档已设计 DCC 接口；(2) 澄清 UAI-BD-ARC-014 编号仅限部署配置设计方针，DCC 相关正式编号回填时须使用 UAI-BD-ARC-015 以避免冲突；(3) §6.1/§6.2 实体命名由 ValidationRecord 统一改为 ValidationReport，与详细设计书 §3.4/§4 字段级设计保持术语一致 | AI Research / Architecture |
| v0.3 | 2026-08-18 | 第二轮三级文档交叉审核（详见详细设计书 §13.0.2）：修正 §4.1 逻辑构成图与 §4.2 模块一览表的编号不一致——LLM Reasoning Adapter 独立编号为 [7]、Rig Compiler 改为 [8]、Physics Validation Coordinator 改为 [9]，并删除模块表中不存在的幻影模块「Universal Rig Graph Store」；§5.11 输出分级引用改指需求定义书附录 A.2 | AI Research / Architecture |

### 0.2 承认体系（Approval Matrix）

| 角色 | 姓名 | 承认状态 | 日期 |
|---|---|---|---|
| 需求负责人（Requirements Owner） | （待指定） | 未承认 | — |
| 技术负责人（Technical Lead） | （待指定） | 未承认 | — |
| AI 研究负责人 | （待指定） | 未承认 | — |

### 0.3 参照文档一览

| 文档 ID | 文档名 | 关系 |
|---|---|---|
| UAI-REQ-001 | 《Universal Articulation Intelligence 需求定义书》v0.3 | 上游依据 |
| UAI-BD-001 | 本文档 | 自身 |
| （未定） | 《Universal Articulation Intelligence 详细设计书》 | 下游文档，本文档进入下一阶段的输入 |

### 0.4 适用范围与读者

面向：架构师、Rust/AI 工程师、Pipeline 工程师、评审委员会。本文档描述系统的外部设计（模块划分、接口、数据概念模型、处理流程），不描述内部类结构、算法实现细节或数据库物理设计，这些内容留待详细设计阶段。

---

## 1. 文档目的

本文档基于 `UAI-REQ-001` 需求定义书，将需求转化为系统的基本设计（外部设计），确定：系统整体构成、子系统/模块划分与职责边界、数据概念模型（非物理表设计）、外部接口规格（API/CLI 契约层面，非实现）、核心处理流程、非功能设计方针。目的是为详细设计与 PoC 实现提供可执行、可评审、可追踪的设计基线。

## 2. 基本设计范围与非目标

**范围内**：逻辑系统构成、模块划分与职责、模块间数据流、Structural DSL 的语法层设计方针（非 Parser 实现）、图/向量知识库的概念模型（Instance/Experience/Pattern 三层的实体与关系设计）、核心处理流程（拆分/检索/推理/验证/学习闭环）的时序设计、外部接口的请求/响应契约（Schema 层面）、非功能设计方针（性能/可扩展/部署/可观测性的设计对策）。

**范围外（非目标）**：具体编程语言级类/结构体设计、具体第三方库版本选型（仅给出能力方向，与需求文档一致）、数据库物理表 DDL、UI 视觉设计、具体算法伪代码之外的实现细节、部署脚本/IaC 配置。

---

## 3. 系统概要

UAI 系统接收三维资产（MVP：Static Mesh）与可选运动证据，经由 Rust 核心运行时执行几何/拓扑特征抽取、层级化结构拆分、图/向量联合检索、LLM 结构推理（经 Structural DSL 中介）、物理验证闭环，最终产出 Universal Rig Graph 及其派生表示（Skeleton/Skinning/Constraints），并将处理过程与人工修正结果沉淀至三层知识图谱，形成自学习闭环。

---

## 4. 系统构成设计（UAI-BD-ARC-*）

### 4.1 逻辑构成图

```
                        ┌─────────────────────────────────────────┐
                        │              UAI Rust Core                │
                        │                                           │
  Asset Input ────────▶ │  [1] Ingestion & Validation               │
                        │        │                                  │
                        │        ▼                                  │
                        │  [2] Geometry Feature Engine               │
                        │  [3] Topology Feature Engine               │
                        │        │                                  │
                        │        ▼                                  │
                        │  [4] Structural Decomposition Engine       │
                        │        (Structural Grammar Rule Set)       │
                        │        │                                  │
                        │        ▼                                  │
                        │  [5] Retrieval Coordinator ──────────────┼──▶ Vector Store (外部/可替换)
                        │        │                                  │
                        │        │                                  ├──▶ Graph Store (外部/可替换)
                        │        ▼                                  │      Instance / Experience / Pattern Graph
                        │  [6] Structural DSL Layer                  │
                        │        │                                  │
                        │        ▼                                  │
                        │  [7] LLM Reasoning Adapter ───────────────┼──▶ LLM Provider (外部/可替换)
                        │        │                                  │
                        │        ▼                                  │
                        │  [8] Rig Compiler (deterministic)          │
                        │        │                                  │
                        │        ▼                                  │
                        │  [9] Physics Validation Coordinator ───────┼──▶ Physics Engine (外部/可替换)
                        │        │                                  │
                        │        ▼                                  │
                        │ [10] Confidence & Explanation Service       │
                        │        │                                  │
                        │        ▼                                  │
                        │ [11] Human-in-the-loop Correction Service   │
                        │        │                                  │
                        │        ▼                                  │
                        │ [12] Learning Loop / Pattern Mining Worker  │
                        │                                           │
                        │  [13] API Gateway / CLI                    │
                        └─────────────────────────────────────────┘
```

### 4.2 モジュール一覧（模块一览）

| 设计 ID | 模块名 | 职责 | 部署单元角色（对应需求 UAI-NFR-DEPLOY-003） | 对应需求 |
|---|---|---|---|---|
| UAI-BD-ARC-001 | Ingestion & Validation | 校验输入资产格式与完整性，转换为内部统一表示 | Library / Process 入口 | UAI-NFR-SEC-001, UAI-ERR-001 |
| UAI-BD-ARC-002 | Geometry Feature Engine | 计算 Curvature/Thickness/Volume/Elongation/Symmetry/Spatial Relation/Geodesic/Medial | Library（可并行 Worker 化） | UAI-FR-GEO-001~006 |
| UAI-BD-ARC-003 | Topology Feature Engine | 计算 Connectivity/Component/Boundary/Seam/Branch/Loop/Bottleneck/Adjacency | Library | UAI-FR-TOP-001~006 |
| UAI-BD-ARC-004 | Structural Decomposition Engine | 执行层级化递归拆分，应用 Structural Grammar 规则集，判断拆分终止 | Library | UAI-FR-DEC-001~004, UAI-FR-GRAM-001~003 |
| UAI-BD-ARC-005 | Retrieval Coordinator | 协调 Vector 检索与 Graph 匹配，产出候选相似结构与模式 | Service（可远程调用 Vector/Graph Store） | UAI-FR-VEC-001~004, UAI-FR-GRAPH-001~003 |
| UAI-BD-ARC-006 | Structural DSL Layer | 定义 DSL 语法与语义，承载 LLM 输出的高层结构描述，作为 LLM 与 Compiler 之间的契约边界 | Library | UAI-FR-DSL-001~004 |
| UAI-BD-ARC-007 | LLM Reasoning Adapter | 封装 LLM Provider 调用，输入 Structural Evidence，输出 DSL 层多假设 + 置信度 | Service（外部 Provider 抽象层，含 Python Adapter 可能性） | UAI-FR-AI-001~006, UAI-NFR-EXT-001 |
| UAI-BD-ARC-008 | Rig Compiler | 确定性地将 DSL 编译为 Structural Graph / Articulation Graph / Rig Graph / Skeleton / Constraints / Skinning Regions | Library | UAI-FR-DSL-003, UAI-FR-RIG-001~006 |
| UAI-BD-ARC-009 | Physics Validation Coordinator | 协调外部物理引擎执行 Motion Probe/仿真，汇总验证结果，驱动假设重排序 | Service / GPU Worker | UAI-FR-PHY-001~009 |
| UAI-BD-ARC-010 | Confidence & Explanation Service | 汇总证据、假设、验证结果，产出结构化置信度与解释输出 | Library/Service | UAI-FR-CONF-001~003 |
| UAI-BD-ARC-011 | Human-in-the-loop Correction Service | 接收人工修正，关联原始判断/修正结果/验证分数，写入 Experience Graph | Service | UAI-FR-HIL-001~005 |
| UAI-BD-ARC-012 | Learning Loop / Pattern Mining Worker | 离线/异步从 Experience Graph 挖掘 Pattern/Rule，回写 Pattern Graph | Worker（异步批处理） | UAI-FR-LEARN-001~003, UAI-FR-GRAPH-003 |
| UAI-BD-ARC-013 | API Gateway / CLI | 对外暴露资产提交/结果查询/修正提交接口；CLI 为 MVP 主要交互形态 | Service + CLI | UAI-API-001~004 |

### 4.3 配置构成设计方针（UAI-BD-ARC-014）

MVP 阶段（Local-first，对应 UAI-NFR-DEPLOY-001）：模块 1-4, 6, 8, 10-11, 13 以单一本地进程（Library 集合）形式运行；模块 5（Retrieval）、7（LLM Adapter）、9（Physics）作为可插拔外部依赖，通过抽象接口调用（本地或远程均可）；模块 12（Learning Loop）作为独立异步 Worker 进程，可离线运行。

V1+ 阶段：模块 5/7/9/12 可独立拆分为 Service/GPU Worker 部署单元，具体拆分粒度与通信协议留待详细设计阶段确定，遵循需求 UAI-NFR-DEPLOY-004（避免过早服务拆分）。

---

## 5. 机能设计（UAI-BD-FUNC-*）

对每个核心处理阶段给出：输入、处理概要、输出、异常处理方针。（详细算法留待详细设计阶段）

### 5.1 UAI-BD-FUNC-001 几何特征抽取

- 输入：Ingestion 后的内部统一网格表示
- 处理概要：并行计算曲率/厚度/体积/伸长度/对称性/空间关系/测地距离/中轴结构，产出 GeometryEvidence 记录
- 输出：GeometryEvidence（关联至 Region ID）
- 异常处理：退化网格（非流形/孤立顶点）须被 Ingestion 阶段提前拦截（UAI-BD-FUNC-013）；若局部特征计算失败，标记该 Region 特征缺失，不阻断整体流程（对应 UAI-ERR-002）
- 对应需求：UAI-FR-GEO-001~006

### 5.2 UAI-BD-FUNC-002 拓扑特征抽取

- 输入：内部统一网格表示
- 处理概要：计算连通性/组件/边界/缝合线/分支/环路/瓶颈/区域邻接，产出 TopologyEvidence 与 Region Adjacency Graph
- 输出：TopologyEvidence、Region Adjacency Graph
- 设计约束：瓶颈检测结果须以"证据强度分数"形式输出，不得直接输出布尔型"是否为关节"结论（对应 UAI-FR-TOP-006 的设计落地）
- 对应需求：UAI-FR-TOP-001~006

### 5.3 UAI-BD-FUNC-003 层级化结构拆分

- 输入：GeometryEvidence、TopologyEvidence、Region Adjacency Graph
- 处理概要：自顶向下递归应用 Structural Grammar 规则集（Bottleneck/Motion Independence/Deformation Continuity/Symmetry/Repetition/Chain/Hub/Physics Override Rule），对每个 Region 输出"继续拆分 / 终止"判定及触发规则引用
- 输出：Structural Part Graph（层级树 + 每层判定依据）
- 对应需求：UAI-FR-DEC-001~004、UAI-FR-GRAM-001~003

### 5.4 UAI-BD-FUNC-004 向量+图联合检索

- 输入：Structural Part Graph 中各 Region 的特征向量（Geometry/Topology Vector，MVP 子集）
- 处理概要：先执行向量近似最近邻检索获得候选相似历史 Region 集合，再对候选集合执行 Graph Matching（比对 Instance Graph 关系模式），输出结构化 Similar Pattern 候选列表
- 输出：Similar Pattern 候选（含相似度分数、来源 Instance/Pattern 引用）
- 对应需求：UAI-FR-VEC-001~004、UAI-FR-GRAPH-001~003

### 5.5 UAI-BD-FUNC-005 LLM 结构假设生成

- 输入：Structural Evidence（Geometry+Topology+可选 Motion/Physics 摘要）+ Similar Pattern 候选
- 处理概要：调用 LLM Reasoning Adapter，输入结构化证据摘要（非原始几何），输出至少一个主假设与一个备选假设，每个假设以 Structural DSL 表达，附带触发规则/证据引用
- 输出：DSL 假设列表（主 + 备选），每项含 Confidence（初步，未经物理验证）
- 异常处理：LLM 调用失败或返回不可解析 DSL 时，须触发降级路径（如回退至纯规则驱动假设），不得让整体流程失败
- 对应需求：UAI-FR-AI-001~006、UAI-FR-DSL-001~002

### 5.6 UAI-BD-FUNC-006 DSL 编译

- 输入：DSL 假设
- 处理概要：确定性编译为 Structural Graph / Articulation Graph / Rig Graph（Universal Rig Graph 子集）/ Skeleton / Constraints / Skinning Regions
- 输出：编译后的 Rig Hypothesis（结构化，非 DSL 文本）
- 异常处理：DSL 语法/语义非法时拒绝编译并返回结构化错误（阶段+原因），不得静默产出不完整 Rig
- 对应需求：UAI-FR-DSL-003、UAI-FR-RIG-001~003

### 5.7 UAI-BD-FUNC-007 物理验证

- 输入：Rig Hypothesis（一个或多个）
- 处理概要：对每个假设执行 Motion Probe，驱动物理仿真，检测 Penetration/Stretching/Volume Loss/Joint Instability/Constraint Violation（MVP 三项：Penetration/Stretching/Joint Instability，见需求 §39）
- 输出：Validation Report（每假设一份，含通过/失败项与量化分数）
- 处理：验证结果反馈至假设排序（重排序或触发修正建议）
- 对应需求：UAI-FR-PHY-001~009

### 5.8 UAI-BD-FUNC-008 置信度与解释汇总

- 输入：Rig Hypothesis 列表 + Validation Report + Similar Pattern 引用
- 处理概要：综合规则触发证据、检索相似度、物理验证结果，计算最终 Confidence，生成正/负证据说明与备选假设摘要
- 输出：Confidence Data（对应需求结构：Confidence/Evidence/Alternative/Reason/Validation Result）
- 对应需求：UAI-FR-CONF-001~003

### 5.9 UAI-BD-FUNC-009 人工修正录入

- 输入：Artist 提交的修正（针对特定 Joint Candidate / Region）
- 处理概要：记录原始 AI 判断快照、修正结果、修正原因（可选结构化标签+自由文本）、当前 Physics Score，标记 Final Accepted Result
- 输出：Correction Record（写入 Experience Graph）
- 对应需求：UAI-FR-HIL-001~005

### 5.10 UAI-BD-FUNC-010 经验回写与规律挖掘

- 输入：Experience Graph 中新增的处理记录（含成功/失败/人工修正）
- 处理概要：异步批处理任务定期（或触发式）扫描新增记录，聚合相似证据模式，更新或新增 Pattern/Rule（含 Confidence/Support/Evidence/Counterexample/Version）
- 输出：Pattern Graph 更新
- 对应需求：UAI-FR-LEARN-001~003、UAI-FR-GRAPH-003、UAI-FR-GRAPH-005

### 5.11 UAI-BD-FUNC-011 Rig 结果导出

- 输入：最终确认（或高置信度未修正）的 Rig Hypothesis
- 处理概要：将 Universal Rig Graph 导出为 Skeleton（含 Joint/Axis/DOF/Range of Motion，MVP 子集）
- 输出：导出结果（对应需求定义书 附录 A.2 输出范围分级）
- 对应需求：UAI-FR-RIG-003~006

### 5.12 UAI-BD-FUNC-012 API/CLI 请求处理

- 输入：外部请求（资产提交/结果查询/修正提交）
- 处理概要：请求校验 → 路由至对应内部服务 → 结构化响应
- 输出：API/CLI 响应（见 §7 接口设计）
- 对应需求：UAI-API-001~005

### 5.13 UAI-BD-FUNC-013 输入校验（Ingestion）

- 输入：原始资产文件
- 处理概要：格式校验、非流形/退化几何检测、大小限制校验
- 输出：内部统一表示 或 结构化拒绝原因
- 对应需求：UAI-NFR-SEC-001、UAI-ERR-001

---

## 6. 数据设计（概念模型，UAI-BD-DATA-*）

> 本节仅定义概念实体与关系（Conceptual/Logical Data Model），不涉及具体数据库物理表结构、字段类型、索引设计（留待详细设计阶段）。

### 6.1 核心概念实体

| 实体 | 说明 | 所属知识层（对应需求） |
|---|---|---|
| Asset | 一个被处理的三维资产（含类型、来源、处理状态） | Instance Graph |
| Region | 结构拆分产生的一个区域节点（含层级关系） | Instance Graph |
| GeometryEvidence / TopologyEvidence | 与 Region 关联的特征证据记录 | Instance Graph（引用，非存储原始几何） |
| RelationEdge | Region 间关系（CONNECTS/PART_OF/SYMMETRIC_WITH/MOVES_RELATIVE_TO/ATTACHED_VIA） | Instance Graph |
| ArticulationHypothesis | 一个结构假设（含 DSL 引用、Confidence、触发规则引用） | Instance Graph（当前资产内） |
| ValidationReport | 一次物理验证结果 | Experience Graph |
| CorrectionRecord | 一次人工修正记录 | Experience Graph |
| Pattern / Rule | 抽象结构规律（含 Confidence/Support/Evidence/Counterexample/Version） | Pattern / Rule Graph |
| ProcessingRun | 一次完整处理流程的执行记录（用于可追溯性，对应 UAI-NFR-OBS-002） | Experience Graph |

### 6.2 核心关系设计（概念级 ER，非物理表）

```
Asset 1—* Region
Region *—* Region        : RelationEdge（CONNECTS/PART_OF/SYMMETRIC_WITH/MOVES_RELATIVE_TO/ATTACHED_VIA）
Region 1—* ArticulationHypothesis
ArticulationHypothesis 1—* ValidationReport
ArticulationHypothesis 0..1—* CorrectionRecord
ProcessingRun 1—* ArticulationHypothesis   （用于追溯一次运行产出的全部假设）
Pattern *—* ArticulationHypothesis         : 引用关系（该假设由哪些历史 Pattern 支持）
Pattern 1—* Evidence/Counterexample（内嵌属性，非独立强实体，视存储实现而定）
```

### 6.3 数据职责边界设计（落实需求 UAI-FR-GRAPH-004）

- 原始 Mesh 顶点/面数据：存储于文件系统或专用几何存储，图数据库仅保存**引用（Reference/Pointer）**，不存储原始几何。
- 向量数据（GeometryVector/TopologyVector/MotionVector/PhysicsVector/SemanticVector）：存储于 Vector Store，图节点仅保存向量 ID 引用。
- 图数据库承载：Region 层级/关系、Hypothesis、Validation/Correction 记录、Pattern/Rule。

### 6.4 数据版本化设计方针（对应 UAI-DATA-003）

Instance/Experience/Pattern 三层 Schema 均须携带 `schema_version` 字段；Pattern/Rule 实体须携带独立 `pattern_version`，允许同一逻辑规律的多个版本并存并被追溯比较。

---

## 7. 外部接口设计（UAI-BD-IF-*）

> 本节定义接口契约层面的请求/响应结构方向，不锁定具体序列化格式或传输协议实现细节（如是否为 REST/gRPC，留待详细设计阶段依据 UAI-NFR-RUST-002 的能力方向选型）。

### 7.1 UAI-BD-IF-001 资产提交接口

- 对应需求：UAI-API-001
- 请求概念结构：资产文件引用/内容、资产类型标注（MVP: Static Mesh）、可选处理参数（如是否启用某些非 MUST 验证项）
- 响应概念结构：`ProcessingRun` ID、初始状态（Accepted/Rejected + 原因）

### 7.2 UAI-BD-IF-002 结果查询接口

- 对应需求：UAI-API-002
- 请求概念结构：`ProcessingRun` ID 或 `Asset` ID
- 响应概念结构：Structural Part Graph、Rig Graph（Universal + 导出的 Skeleton 子集）、Confidence Data 列表、Validation Report

### 7.3 UAI-BD-IF-003 人工修正提交接口

- 对应需求：UAI-API-003
- 请求概念结构：目标 `ArticulationHypothesis`/Joint 引用、修正内容、修正原因（可选）
- 响应概念结构：`CorrectionRecord` ID、确认状态

### 7.4 UAI-BD-IF-004 CLI 设计方针

- 对应需求：UAI-API-004
- CLI 须覆盖：提交资产（同步/异步模式）、查询状态与结果、批量提交（对应 UAI-NFR-PERF-002）、导出结果到本地文件
- CLI 为 MVP 的主要交互入口，API 为其底层能力的可编程访问方式

### 7.5 UAI-BD-IF-005 内部 Provider 抽象接口设计方针（对应 UAI-NFR-EXT-001）

以下四类外部依赖须通过统一抽象接口接入，接口契约（请求/响应形状）在本阶段定义，具体 Provider 实现留待详细设计：

| Provider 类别 | 抽象接口职责 | 对应需求 |
|---|---|---|
| LLM Provider | 输入 Structural Evidence（结构化文本/JSON），输出 DSL 假设 + 初步置信度 | UAI-FR-AI-005 |
| Physics Engine | 输入 Rig Hypothesis + Motion Probe 定义，输出 Validation Report | UAI-FR-PHY-009 |
| Vector Store | 输入查询向量，输出 Top-K 相似节点引用 | UAI-FR-VEC-001 |
| Graph Store | 输入图查询（关系模式匹配），输出匹配子图 | UAI-FR-GRAPH-004 |

---

## 8. 处理流程设计（UAI-BD-FLOW-*）

### 8.1 主处理时序（UAI-BD-FLOW-001，对应需求 Journey A/B、UC-01~UC-03）

```
Client → [13 API/CLI] → [1 Ingestion]
  → [2 Geometry] ∥ [3 Topology]（并行）
  → [4 Structural Decomposition]（消费 2/3 输出，循环至每个 Region 判定终止）
  → [5 Retrieval Coordinator]（Vector → Graph Matching）
  → [6 DSL Layer] ⇄ [7 LLM Adapter]（生成多假设 DSL）
  → [8 Rig Compiler]（DSL → Rig Hypothesis，确定性）
  → [9 Physics Validation]（对每个 Hypothesis 独立验证，可并行）
  → [10 Confidence & Explanation]（汇总排序）
  → 返回结果给 Client（经 [13]）
  → （可选）[11 Human Correction] → 写入 Experience Graph
  → （异步）[12 Learning Loop] 定期消费 Experience Graph → 更新 Pattern Graph
```

### 8.2 失败/降级路径设计（对应 UAI-ERR-001~003）

- Ingestion 校验失败 → 立即返回结构化拒绝原因，不进入后续阶段。
- 单 Region 特征计算失败 → 标记该 Region 为 `evidence_incomplete`，不阻断其余 Region 处理（UAI-ERR-002）。
- LLM 调用失败/超时 → 触发规则驱动降级假设生成（不依赖 LLM 的最小可用路径），并在 Confidence 输出中标注"降级模式"。
- 物理验证引擎不可用 → 假设仍可返回，但 Validation Report 标记为 `validation_skipped`，Confidence 计算须相应降权，不得虚报高置信度。
- 所有失败/降级须记录至 `ProcessingRun` 的执行日志，满足 UAI-NFR-OBS-002 可追溯性要求。

---

## 9. 非功能设计（UAI-BD-NFR-*）

### 9.1 性能设计方针（对应 §29）

- UAI-BD-NFR-001：Geometry/Topology 特征抽取阶段设计为可并行任务单元（按 Region 或按网格分块并行），以利用多核（对应 UAI-NFR-PERF-003）。
- UAI-BD-NFR-002：批量资产处理设计为异步队列模式（提交即返回 `ProcessingRun` ID，结果异步查询），避免同步阻塞（对应 UAI-NFR-PERF-002）。
- UAI-BD-NFR-003：具体性能 SLA 数值（吞吐、延迟阈值）留待详细设计阶段基于基准测试确定（对应需求 Open Issue OI-01），本阶段仅确定"异步 + 并行"的设计方向。

### 9.2 可扩展性设计（对应 §28.2）

- UAI-BD-NFR-004：LLM/Physics/Vector/Graph 四类外部依赖均通过 §7.5 定义的抽象接口接入，核心模块（4/6/8/10）不得直接依赖具体 Provider 的 SDK 类型，只依赖抽象接口类型。

### 9.3 部署设计方针（对应 §28.3、§33）

- UAI-BD-NFR-005：MVP 部署为单机本地进程（含内嵌或本地可选 Vector/Graph Store），符合 Local-first 要求（UAI-NFR-DEPLOY-001）。
- UAI-BD-NFR-006：模块间通信在 MVP 阶段可为进程内函数调用；服务化边界（若干模块拆分为独立 Service）在详细设计阶段依据实际性能瓶颈决定，本阶段不预先锁定（对应 UAI-NFR-DEPLOY-004）。

### 9.4 安全设计方针（对应 §30）

- UAI-BD-NFR-007：Ingestion 阶段（UAI-BD-FUNC-013）承担输入校验职责，是唯一允许接受外部不可信输入的边界模块。
- UAI-BD-NFR-008：LLM/云服务调用凭据须通过独立配置/密钥管理机制注入，不与领域数据混合存储或记录在日志中。

### 9.5 可观测性设计方针（对应 §31）

- UAI-BD-NFR-009：每个 `ProcessingRun` 须记录各阶段（1-10）的开始/结束时间、状态、关键中间结果引用，形成可追溯执行轨迹。
- UAI-BD-NFR-010：Confidence & Explanation Service（模块10）的输出须包含足够信息以回答"某关节判断的证据链是什么"（对应 UAI-NFR-OBS-002）。

---

## 10. Rust 技术约束落实设计（UAI-BD-RUST-*）

- UAI-BD-RUST-001：模块 1-4, 6, 8, 10, 11, 13 须以 Rust 实现（对应 UAI-NFR-RUST-001）。
- UAI-BD-RUST-002：模块 7（LLM Reasoning Adapter）若需引入 Python（如调用特定研究模型的官方 Python 实现），须以独立进程/子服务形式存在，与 Rust Core 之间仅通过 §7.5 定义的抽象接口契约（结构化数据，非共享内存对象）通信，符合 UAI-NFR-RUST-003。
- UAI-BD-RUST-003：模块 9（Physics Validation Coordinator）中，Rust 侧负责协调与结果解析，具体物理仿真内核可为外部引擎（Rust 或非 Rust 均可，通过抽象接口调用），不违反 Rust Core 原则（物理引擎内核本身在系统边界外，见需求 §12）。
- UAI-BD-RUST-004：本阶段不锁定具体 crate（如是否用 petgraph/nalgebra 等），仅确认能力方向：异步运行时能力、Web 服务能力、序列化能力、数据并行能力、数学/几何计算能力、图结构能力、GPU 抽象能力、FFI 能力（对应 UAI-NFR-RUST-002）。

---

## 11. Structural DSL 语法设计方针（UAI-BD-DSL-*，对应 UAI-FR-DSL-001~002）

> 仅给出语法层设计方向，不给出 Parser/Compiler 实现代码。

- UAI-BD-DSL-001：DSL 顶层结构为 `STRUCTURE <name> { PART ... }` 嵌套形式，PART 可嵌套 SEGMENT/JOINT 子元素。
- UAI-BD-DSL-002：PART 须支持 `role` 属性（如 `hub`、`appendage`），用于承载 Structural Grammar 中 Hub Rule 等规则的语义。
- UAI-BD-DSL-003：JOINT 元素须支持类型标注（如 `rotational`），作为 Rig Compiler 映射至 Universal Rig Graph 关节原语（Hinge/Ball Joint 等）的依据。
- UAI-BD-DSL-004：DSL 文本须可无损映射回其触发的 Structural Grammar 规则引用，保证 UAI-FR-GRAM-002（可解释性）在编译后依然可追溯。

---

## 12. 错误处理设计（UAI-BD-ERR-*，对应 §36）

| 设计 ID | 错误类别 | 设计对策 |
|---|---|---|
| UAI-BD-ERR-001 | 输入错误 | Ingestion 阶段统一拦截，返回结构化错误码 + 原因，不进入处理管线 |
| UAI-BD-ERR-002 | 内部处理失败（单元级） | 单 Region/单假设失败须被隔离，不影响其余单元；`ProcessingRun` 整体状态标记为"部分完成"而非整体失败 |
| UAI-BD-ERR-003 | 验证未通过 | 视为正常业务结果（非系统错误），通过 Validation Report 正常返回，驱动假设降权/重排序 |

---

## 13. 移行方针（Migration，对应共通フレーム惯例，本项目为新建系统）

本系统为新建（Green-field）项目，不涉及既有系统数据迁移。若未来对接既有 Rig 资产库（如导入 Existing Skeleton/Animation，对应需求输入分级 V1），须在对应版本单独制定数据迁移设计，本阶段不展开。

## 14. 测试方针（结合レベル，Integration-level，UAI-BD-TEST-*）

- UAI-BD-TEST-001：模块 2/3（Geometry/Topology）须有独立于 Structural Decomposition 的单元级验证入口，满足需求 §15 验收标准"可独立测试"要求。
- UAI-BD-TEST-002：须构造"高 Bottleneck 但非关节"反例资产集合，作为模块 4 与模块 6 集成测试的回归用例（对应需求 UAI-FR-TOP-006、AC-02）。
- UAI-BD-TEST-003：端到端集成测试须覆盖 §8.1 全链路（Static Mesh 输入 → 最终 Rig 输出），作为 MVP 验收（对应需求 AC-08）的技术验证载体。
- UAI-BD-TEST-004：LLM/Physics/Vector/Graph 四类外部依赖须可替换为测试替身（Test Double/Mock），保证核心链路可在无外部服务条件下进行确定性回归测试。

具体测试用例设计、覆盖率目标留待详细设计/测试设计阶段。

---

## 15. 风险与设计层应对（UAI-BD-RISK-*，承接需求 §41）

| 设计 ID | 承接风险 | 设计层应对 |
|---|---|---|
| UAI-BD-RISK-001 | R-01（LLM 不可靠） | §5.5 降级路径设计 + §8.2 失败路径设计，确保 LLM 故障不阻断整体流程且不虚报置信度 |
| UAI-BD-RISK-002 | R-02（Bottleneck 误判为 Joint） | §5.2 设计约束（瓶颈仅输出证据强度，非布尔结论）+ §14 反例回归测试 |
| UAI-BD-RISK-003 | R-03（图数据库职责蔓延） | §6.3 数据职责边界设计，原始几何/向量数据不进入图存储 |
| UAI-BD-RISK-004 | R-04（Python 侵入核心） | §10 Rust 技术约束落实设计，明确 Python 仅存在于模块 7 的独立进程边界 |
| UAI-BD-RISK-005 | R-06（物理验证成本高） | §8.2 验证不可用时的降级路径 + MVP 阶段仅验证三项（对应需求 §39） |

---

## 16. 用语表

沿用需求定义书 `UAI-REQ-001` §5 术语定义；本文档新增以下设计层术语：

| 术语 | 定义 |
|---|---|
| ProcessingRun | 一次完整的资产处理流程执行实例，是可追溯性设计的核心锚点 |
| Test Double | 测试替身，用于在无外部依赖条件下进行确定性回归测试 |
| 降级模式（Degraded Mode） | 当外部依赖（LLM/Physics）不可用时，系统切换至规则驱动或标记结果为部分可信的运行模式 |

---

## 17. 需求-设计追踪矩阵（Requirement-to-Design Traceability Matrix）

| 需求 ID（UAI-REQ-001） | 对应设计 ID（本文档） |
|---|---|
| UAI-FR-GEO-001~006 | UAI-BD-ARC-002, UAI-BD-FUNC-001 |
| UAI-FR-TOP-001~006 | UAI-BD-ARC-003, UAI-BD-FUNC-002 |
| UAI-FR-DEC-001~004, UAI-FR-GRAM-001~003 | UAI-BD-ARC-004, UAI-BD-FUNC-003 |
| UAI-FR-DSL-001~004 | UAI-BD-ARC-006, UAI-BD-ARC-008, UAI-BD-DSL-001~004, UAI-BD-FUNC-006 |
| UAI-FR-GRAPH-001~005 | UAI-BD-ARC-005, UAI-BD-DATA-001~004, UAI-BD-FUNC-004, UAI-BD-FUNC-010 |
| UAI-FR-VEC-001~004 | UAI-BD-ARC-005, UAI-BD-FUNC-004, UAI-BD-IF-005 |
| UAI-FR-AI-001~006 | UAI-BD-ARC-007, UAI-BD-FUNC-005, UAI-BD-IF-005 |
| UAI-FR-PHY-001~009 | UAI-BD-ARC-009, UAI-BD-FUNC-007, UAI-BD-IF-005 |
| UAI-FR-RIG-001~006 | UAI-BD-ARC-008, UAI-BD-FUNC-006, UAI-BD-FUNC-011 |
| UAI-FR-CONF-001~003 | UAI-BD-ARC-010, UAI-BD-FUNC-008 |
| UAI-FR-HIL-001~005 | UAI-BD-ARC-011, UAI-BD-FUNC-009 |
| UAI-FR-LEARN-001~003 | UAI-BD-ARC-012, UAI-BD-FUNC-010 |
| UAI-NFR-GEN-001~003, UAI-NFR-RUST-001~003 | UAI-BD-RUST-001~004 |
| UAI-NFR-EXT-001 | UAI-BD-NFR-004, UAI-BD-IF-005 |
| UAI-NFR-DEPLOY-001~004 | UAI-BD-ARC-014, UAI-BD-NFR-005~006 |
| UAI-NFR-PERF-001~005 | UAI-BD-NFR-001~003 |
| UAI-NFR-SEC-001~003 | UAI-BD-FUNC-013, UAI-BD-NFR-007~008 |
| UAI-NFR-OBS-001~003 | UAI-BD-NFR-009~010 |
| UAI-DATA-001~004 | UAI-BD-DATA-001~004 |
| UAI-API-001~004 | UAI-BD-IF-001~004 |
| UAI-API-005（DCC 集成接口预留） | 本文档未设计具体接口（仅需求侧预留），后续 DCC 通用集成设计见 `UAI-DD-001` §11.5，正式编号回填留待需求/基本设计下一版本（见该文档 Open Issue OI-06） |
| UAI-ERR-001~003 | UAI-BD-ERR-001~003, §8.2 |
| R-01~R-09 | UAI-BD-RISK-001~005（覆盖主要研究/架构风险；R-05/R-07/R-08/R-09 的设计层应对分别体现于 §2 范围控制、§14 测试方针、§5.3 规则可追溯设计、§5.9~5.10） |

---

## 18. 自审（Self-Review）

对照日本 IPA 共通フレーム基本設計プロセス惯例（システム構成／機能設計／データ設計／インターフェース設計／非機能設計／移行方針／テスト方針的完整性）与需求追踪完整性，检查如下：

| 检查项 | 状态 | 说明 |
|---|---|---|
| 文档管理信息/改订履历/承认体系 | ✅ | §0 |
| 系统构成图与模块一览 | ✅ | §4 |
| 每个模块均可追溯至需求 | ✅ | §4.2 表格右列 |
| 数据设计止步于概念/逻辑模型，未涉及物理表 | ✅ | §6 明确声明范围 |
| 接口设计止步于契约层面，未锁定协议实现 | ✅ | §7 明确声明 |
| 处理流程含正常与失败/降级路径 | ✅ | §8.1/§8.2 |
| 非功能需求逐项有设计层落地 | ✅ | §9 |
| Rust/Python 边界在设计层可执行 | ✅ | §10 |
| 测试方针覆盖核心风险回归用例 | ✅ | §14，尤其 UAI-BD-TEST-002 |
| 需求-设计双向追踪矩阵完整 | ✅ | §17 |
| 是否提前进入详细设计/代码/物理表 | ✅ 未越界 | 全文未出现类图、DDL、实现代码 |

**遗留缺口（留待详细设计阶段解决，非本阶段遗漏）**：具体协议选型（REST vs gRPC）、具体 Vector/Graph Store 产品选型、具体并发/并行调度策略数值参数、具体 Pattern Mining 算法。这些均属于详细设计范畴，在需求与本设计文档中已作为"能力方向 + 待定"显式标注，符合 IPA 基本设计阶段"不过早锁定实现细节"的惯例。

---

*本文档为基本设计阶段产出，禁止作为详细设计或实现依据直接编码；后续须编制详细设计书（含具体接口协议、数据库物理设计、算法细节）后方可进入 PoC 实现。*
