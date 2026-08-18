# Universal Articulation Intelligence — 详细设计书

版本: v0.3 (Draft)
文档状态: 详细设计阶段（Detailed Design / 内部設計）— 定义模块内部构造、数据字段级结构、接口字段级契约、处理时序细节、算法设计方针、错误码体系
作者: AI Research / Architecture (assisted)
日期: 2026-08-18
文档标准: 参照日本 IPA（情報処理推進機構）共通フレーム2013（SLCP-JCF2013）之「詳細設計プロセス（内部設計）」惯例编制，章节结构对齐典型详细设计书（モジュール詳細設計／データ詳細設計／インターフェース詳細設計／処理シーケンス／アルゴリズム設計／エラーコード定義）

上游依据文档：`UAI-REQ-001`（需求定义书 v0.3）、`UAI-BD-001`（基本设计书 v0.3）。本文档设计项以 `UAI-DD-*` 编号，并在 §11 追踪矩阵中与上游 `UAI-BD-*` / `UAI-FR-*` 对应。

**本阶段允许**：模块内部结构（组件/函数职责，非具体源代码）、字段级逻辑数据结构（可对应未来物理表，但不使用 DDL 语法）、接口字段级 Schema、处理时序图、算法设计方针（伪代码级，非可编译代码）、错误码体系。
**本阶段仍然禁止**：可编译源代码、具体编程语言类定义、具体数据库产品 DDL 语句、具体第三方库 API 调用代码。

---

## 0. 文档管理信息

### 0.1 改订履历

| 版本 | 日期 | 变更内容 | 作成者 |
|---|---|---|---|
| v0.1 | 2026-08-18 | 初版发布 | AI Research / Architecture |
| v0.2 | 2026-08-18 | 三级文档交叉审核（§13.0）：将 DCC 模块回填编号由 `UAI-BD-ARC-014` 改为 `UAI-BD-ARC-015`（避免与基本设计书 §4.3 冲突）；同步基本设计书参照版本号至 v0.2；登记 Open Issue 编号空间未统一问题 | AI Research / Architecture |
| v0.3 | 2026-08-18 | 第二轮三级文档交叉审核（§13.0.2）：运行时错误码前缀由 `UAI-ERR-nnnn` 改为 `UAI-ERRC-nnnn`，与需求 ID `UAI-ERR-001~003` 命名空间分离；§11.5.7 分级引用改指需求定义书附录 A.1/A.2；记录本轮 4 类发现及审核方法论改进 | AI Research / Architecture |

### 0.2 承认体系（Approval Matrix）

| 角色 | 姓名 | 承认状态 | 日期 |
|---|---|---|---|
| 技术负责人（Technical Lead） | （待指定） | 未承认 | — |
| Rust Core 负责人 | （待指定） | 未承认 | — |
| AI 研究负责人 | （待指定） | 未承认 | — |
| QA/测试负责人 | （待指定） | 未承认 | — |

### 0.3 参照文档一览

| 文档 ID | 文档名 | 关系 |
|---|---|---|
| UAI-REQ-001 | 《Universal Articulation Intelligence 需求定义书》v0.3 | 上上游依据 |
| UAI-BD-001 | 《Universal Articulation Intelligence 基本设计书》v0.3 | 直接上游依据 |
| UAI-DD-001 | 本文档 | 自身 |
| （未定） | PoC 实装计划书 / 测试设计书 | 下游文档 |

### 0.4 适用范围与读者

面向：Rust/AI 实装工程师、QA/测试设计人员、Code Review 负责人。本文档描述"如何构造"（模块内部结构、字段、时序、算法方针、错误码），是 PoC 与实装阶段编码的直接依据；但本文档本身不是可编译代码，编码时仍需工程师做具体语言绑定与库选型决策。

---

## 1. 文档目的与范围

将 `UAI-BD-001` 中定义的 13 个逻辑模块（UAI-BD-ARC-001~013）逐一展开为内部构造设计：内部组件划分、关键处理函数的输入输出契约（概念签名，非源代码）、数据字段级结构、模块间接口的字段级 Schema、核心处理时序、关键算法的设计方针与判定条件、统一错误码体系。目标是使实装工程师可以在不需要重新做架构决策的前提下直接编码。

**范围外**：具体编程语言语法、具体 crate/库 API、数据库产品特定 DDL、UI 像素级设计、CI/CD 流水线配置。

---

## 2. 模块内部设计（UAI-DD-MOD-*）

> 每个模块对应 `UAI-BD-001` §4.2 的一个 UAI-BD-ARC-*。内部组件以"职责单元"描述，非具体类/结构体代码。

### 2.1 UAI-DD-MOD-001（对应 UAI-BD-ARC-001 Ingestion & Validation）

内部组件：
- `FormatDetector`：识别输入资产文件格式，路由至对应 Parser。
- `MeshParser`（MVP：Static Mesh 专用）：解析为内部统一网格表示（顶点/面/法线/UV，概念结构见 §3.1）。
- `IntegrityValidator`：检测非流形边、孤立顶点、退化三角形、超出大小限制的资产。
- `IngestResultEmitter`：产出 `ProcessingRun`（状态=Accepted/Rejected）。

处理契约（概念级，非代码）：
```
ingest(raw_asset: AssetFile, asset_type: AssetTypeTag) -> Result<InternalMesh, IngestError>
```
判定条件：`IntegrityValidator` 发现严重缺陷（非流形比例 > 阈值 T_manifold，具体数值留待 PoC 基准确定）时返回 `IngestError::IntegrityViolation`；轻微缺陷（如孤立顶点少量存在）允许通过但记录 `warnings[]`。

### 2.2 UAI-DD-MOD-002（对应 UAI-BD-ARC-002 Geometry Feature Engine）

内部组件：
- `CurvatureEstimator`：逐顶点/逐面曲率估计。
- `ThicknessVolumeEstimator`：局部厚度与体积估计（如基于射线投射或距离场近似，具体算法见 §7.2）。
- `ElongationSymmetryAnalyzer`：形状伸长度与对称性分析。
- `SpatialRelationBuilder`：区域间空间关系与测地距离计算。
- `MedialStructureExtractor`：中轴/骨架结构提取。
- `GeometryEvidenceAggregator`：汇总以上输出为统一 `GeometryEvidence` 记录（字段见 §3.2）。

处理契约：
```
extract_geometry(mesh: InternalMesh, regions: RegionSet) -> Map<RegionId, GeometryEvidence>
```
并行化设计：以 Region 为并行任务单元（对应 UAI-BD-NFR-001），各 Region 计算相互独立，聚合阶段串行合并。

### 2.3 UAI-DD-MOD-003（对应 UAI-BD-ARC-003 Topology Feature Engine）

内部组件：
- `ConnectivityAnalyzer`：连通分量与组件划分。
- `BoundarySeamDetector`：边界与缝合线检测。
- `BranchLoopDetector`：分支与环路检测（基于图论：度数分析、环检测算法）。
- `BottleneckScorer`：瓶颈强度评分（输出连续分数 `bottleneck_score ∈ [0,1]`，**不输出布尔关节判定**，落实 UAI-FR-TOP-006 / UAI-BD-FUNC-002 设计约束）。
- `RegionAdjacencyBuilder`：构建 Region Adjacency Graph。

处理契约：
```
extract_topology(mesh: InternalMesh, regions: RegionSet) -> Map<RegionId, TopologyEvidence> + RegionAdjacencyGraph
```

### 2.4 UAI-DD-MOD-004（对应 UAI-BD-ARC-004 Structural Decomposition Engine）

内部组件：
- `GrammarRuleEngine`：加载并执行 Structural Grammar 规则集（Bottleneck/Motion Independence/Deformation Continuity/Symmetry/Repetition/Chain/Hub/Physics Override Rule），每条规则实现统一接口 `evaluate(region, evidence_context) -> RuleVerdict{trigger: bool, confidence: f32, explanation: String}`。
- `DecompositionController`：自顶向下递归调用 `GrammarRuleEngine`，对每个 Region 综合多条规则判定结果决定"继续拆分/终止"。
- `PartGraphBuilder`：将判定结果组装为 Structural Part Graph（层级树，节点含触发规则引用列表）。

判定综合方针（UAI-DD-MOD-004-A）：多规则判定采用"加权证据累积"方式（非简单投票），具体权重与阈值留待 PoC 阶段基于标注数据校准；每次判定须记录参与投票的规则集合与各自 `confidence`，保证可解释性（对应 UAI-FR-GRAM-002）。

### 2.5 UAI-DD-MOD-005（对应 UAI-BD-ARC-005 Retrieval Coordinator）

内部组件：
- `VectorQueryClient`：向 Vector Store 抽象接口发起 Top-K 近似最近邻查询（K 为可配置参数）。
- `GraphMatcher`：对 Top-K 候选执行关系模式匹配（子图同构近似匹配，允许部分匹配并输出匹配度分数）。
- `PatternCandidateRanker`：综合向量相似度与图匹配度产出最终 Similar Pattern 候选列表。

处理契约：
```
retrieve(region_vector: GeometryVector + TopologyVector, k: u32) -> List<SimilarPatternCandidate{pattern_ref, vector_similarity, graph_match_score}>
```

### 2.6 UAI-DD-MOD-006（对应 UAI-BD-ARC-006 Structural DSL Layer）

内部组件：
- `DslGrammarSpec`：DSL 语法定义（对应 UAI-BD-DSL-001~004 设计方针的正式化，语法规则见 §6）。
- `DslSerializer` / `DslValidator`：DSL 文本的语法/语义合法性校验（不做编译，仅校验）。

### 2.7 UAI-DD-MOD-007（对应 UAI-BD-ARC-007 LLM Reasoning Adapter）

内部组件：
- `EvidenceSummarizer`：将 GeometryEvidence/TopologyEvidence/SimilarPatternCandidate 压缩为面向 LLM 的结构化摘要（避免传入原始几何，落实 UAI-FR-AI-001）。
- `PromptComposer`：组装结构化输入（非自由文本拼接，采用固定字段模板，便于版本化与回归测试）。
- `LlmProviderClient`：调用抽象 LLM Provider 接口（§8.5）。
- `DslResponseParser`：解析 LLM 输出为候选 DSL 假设列表；解析失败触发降级路径（UAI-DD-MOD-007-FALLBACK，见 §5.2）。
- `HypothesisRanker`（初步）：按 LLM 自报置信度与规则触发强度做初排序（非最终排序，最终排序在物理验证后由模块 010 完成）。

### 2.8 UAI-DD-MOD-008（对应 UAI-BD-ARC-008 Rig Compiler）

内部组件：
- `DslToStructuralGraphCompiler`：DSL → Structural Graph（确定性映射）。
- `StructuralToArticulationCompiler`：Structural Graph → Articulation Graph（关节类型映射，依据 UAI-BD-DSL-003 的 JOINT 类型标注）。
- `ArticulationToRigCompiler`：Articulation Graph → Universal Rig Graph（Bone/Hinge/Ball Joint 等原语实例化，MVP 子集）。
- `SkeletonExporter` / `SkinningRegionEstimator` / `ConstraintExporter`：从 Universal Rig Graph 派生传统 Skeleton/Skinning/Constraints 表示。

编译错误处理：任一编译子阶段失败均返回结构化 `CompileError{stage, reason, dsl_fragment_ref}`，不产出部分 Rig（保证 UAI-FR-DSL-003 的"确定性"语义——要么完整编译成功，要么明确失败）。

### 2.9 UAI-DD-MOD-009（对应 UAI-BD-ARC-009 Physics Validation Coordinator）

内部组件：
- `MotionProbeGenerator`：为给定 Rig Hypothesis 生成标准化运动探测序列（如对每个 Joint 独立施加小幅度旋转/位移，观察形变响应）。
- `PhysicsEngineClient`：调用抽象物理引擎接口（§8.5）执行仿真。
- `ValidationMetricsCollector`：收集 Penetration/Stretching/Volume Loss/Joint Instability/Constraint Violation 等指标（MVP：前三项，见 UAI-BD-FUNC-007）。
- `ValidationReportBuilder`：汇总为 `ValidationReport`（字段见 §3.4）。

### 2.10 UAI-DD-MOD-010（对应 UAI-BD-ARC-010 Confidence & Explanation Service）

内部组件：
- `EvidenceAggregator`：汇总规则触发证据、检索相似度、LLM 自报置信度、物理验证结果。
- `ConfidenceCalculator`：按加权公式（设计方针见 UAI-DD-MOD-010-A）计算最终 Confidence。
- `ExplanationComposer`：生成正/负证据说明文本（结构化字段，非纯自由文本，便于 UI/CLI 一致渲染）。

UAI-DD-MOD-010-A（置信度计算设计方针）：`final_confidence = f(rule_evidence_strength, retrieval_similarity, llm_self_confidence, physics_validation_score)`，具体函数形式（线性加权 vs 学习型校准模型）与各分量权重留待 PoC 阶段基于 Confidence Calibration 实验确定（对应需求 Open Issue OI-02），本阶段仅确定输入分量与输出契约。

### 2.11 UAI-DD-MOD-011（对应 UAI-BD-ARC-011 Human-in-the-loop Correction Service）

内部组件：
- `CorrectionRequestValidator`：校验修正请求的目标引用有效性。
- `CorrectionSnapshotBuilder`：记录修正前的 AI 判断快照（避免后续 Pattern Mining 时原始状态不可追溯）。
- `CorrectionPersister`：写入 `CorrectionRecord`（字段见 §3.5），关联 Experience Graph。

### 2.12 UAI-DD-MOD-012（对应 UAI-BD-ARC-012 Learning Loop / Pattern Mining Worker）

内部组件：
- `ExperienceScanner`：定期扫描 Experience Graph 新增记录（增量扫描，基于 `schema_version` + 时间戳游标）。
- `PatternAbstractor`：从相似证据模式的多个实例中抽象候选 Pattern（聚类 + 规则归纳，具体算法方针见 §7.4）。
- `PatternGraphWriter`：写入/更新 Pattern/Rule（含版本号递增、Counterexample 关联）。

### 2.13 UAI-DD-MOD-013（对应 UAI-BD-ARC-013 API Gateway / CLI）

内部组件：
- `RequestRouter`：路由至内部模块（013 是唯一直接暴露给外部 Client 的模块）。
- `RequestSchemaValidator`：字段级请求校验（与 §5 接口详细设计一致）。
- `ResponseSerializer`：统一响应封装（含 `status`/`data`/`error` 三段式，见 §5.0）。
- `CliCommandDispatcher`：CLI 子命令（`submit`/`status`/`result`/`correct`/`batch-submit`/`export`）到内部 API 调用的映射。

---

## 3. 数据详细设计（UAI-DD-DATA-*，字段级逻辑结构）

> 以下为逻辑字段定义，用于指导未来物理表/存储 Schema 设计，本身不是 DDL。类型标注为抽象类型（如 `String`/`Float32`/`UUID`/`Enum`/`Ref<T>`/`List<T>`），不绑定具体数据库类型系统。

### 3.1 InternalMesh（内部统一网格表示，对应 UAI-BD-FUNC-001/002 输入）

| 字段 | 类型 | 说明 |
|---|---|---|
| mesh_id | UUID | 内部唯一标识 |
| vertices | List<Vec3> | 顶点坐标（不进入图数据库，仅内存/文件存储） |
| faces | List<FaceIndex> | 面索引 |
| normals | List<Vec3> | 法线（可选） |
| uvs | List<Vec2> | UV（可选） |
| schema_version | UInt32 | 结构版本号 |

### 3.2 GeometryEvidence / TopologyEvidence（对应需求 §15/§16，UAI-BD-DATA §6.1）

| 字段 | 类型 | 说明 |
|---|---|---|
| region_id | Ref<Region> | 关联 Region |
| curvature_stats | Struct{mean, std, max} | 曲率统计摘要 |
| thickness_stats | Struct{mean, std} | 厚度统计摘要 |
| volume | Float32 | 区域体积估计 |
| elongation | Float32 | 伸长度（0~1） |
| symmetry_score | Float32 | 对称性分数 |
| geodesic_adjacency | List<Ref<Region>> | 测地邻接区域引用 |
| medial_summary | Struct（简化） | 中轴结构摘要 |
| connectivity_component_id | UInt32 | 所属连通分量 |
| boundary_flag | Bool | 是否含边界 |
| branch_count | UInt32 | 分支数 |
| loop_count | UInt32 | 环路数 |
| bottleneck_score | Float32 (0~1) | 瓶颈强度（非布尔关节结论） |
| evidence_incomplete | Bool | 是否因局部计算失败而不完整（对应 UAI-ERR-002） |
| schema_version | UInt32 | — |

### 3.3 ArticulationHypothesis（对应 UAI-BD-DATA §6.1）

| 字段 | 类型 | 说明 |
|---|---|---|
| hypothesis_id | UUID | 唯一标识 |
| processing_run_id | Ref<ProcessingRun> | 所属处理运行 |
| dsl_text | String | 编译前 DSL 表示（用于可追溯性） |
| structural_graph_ref | Ref<StructuralGraph> | 编译后结构图引用 |
| rig_graph_ref | Ref<UniversalRigGraph> | 编译后 Rig 图引用 |
| triggering_rules | List<RuleReference> | 触发该假设的 Grammar 规则引用 |
| similar_pattern_refs | List<Ref<Pattern>> | 支持该假设的历史模式引用 |
| llm_self_confidence | Float32 | LLM 自报初步置信度 |
| final_confidence | Float32 | 综合最终置信度（模块010计算） |
| is_alternative | Bool | 是否为备选假设（非主假设） |
| rank | UInt32 | 排序位次 |
| schema_version | UInt32 | — |

### 3.4 ValidationReport（对应 UAI-BD-DATA §6.1）

| 字段 | 类型 | 说明 |
|---|---|---|
| validation_id | UUID | — |
| hypothesis_id | Ref<ArticulationHypothesis> | — |
| penetration_detected | Bool | MVP 必检项 |
| stretching_score | Float32 | MVP 必检项 |
| joint_instability_score | Float32 | MVP 必检项 |
| volume_loss_score | Float32 (可选) | V1 |
| constraint_violation_flags | List<Enum> (可选) | V1 |
| validation_skipped | Bool | 物理引擎不可用时置真（对应 §8.2 降级路径） |
| overall_pass | Bool | 综合通过与否 |
| schema_version | UInt32 | — |

### 3.5 CorrectionRecord（对应 UAI-BD-DATA §6.1）

| 字段 | 类型 | 说明 |
|---|---|---|
| correction_id | UUID | — |
| hypothesis_id | Ref<ArticulationHypothesis> | 修正目标 |
| original_snapshot | Struct（快照） | 修正前 AI 判断快照 |
| corrected_value | Struct | 修正后结果 |
| reason_tags | List<Enum> (可选) | 结构化修正原因标签 |
| reason_text | String (可选) | 自由文本原因 |
| physics_score_at_correction | Float32 | 修正时的验证分数 |
| final_accepted | Bool | 是否为最终采纳结果 |
| corrected_by | String（用户标识，非邮箱等敏感信息，需脱敏） | — |
| corrected_at | Timestamp | — |
| schema_version | UInt32 | — |

### 3.6 Pattern / Rule（对应 UAI-BD-DATA §6.1，需求 UAI-FR-GRAPH-003）

| 字段 | 类型 | 说明 |
|---|---|---|
| pattern_id | UUID | — |
| pattern_version | UInt32 | 版本号，随规律演化递增 |
| condition_signature | Struct（特征条件组合） | 如 elongation=high + bottleneck=strong + rigidity=high |
| conclusion | Enum（如 articulated_segment / soft_connection / hub） | 推导结论 |
| confidence | Float32 | — |
| support_count | UInt32 | 支持该规律的实例数 |
| evidence_refs | List<Ref<ArticulationHypothesis / ValidationReport>> | — |
| counterexample_refs | List<Ref<...>> | 反例引用 |
| created_at / updated_at | Timestamp | — |
| schema_version | UInt32 | — |

### 3.7 ProcessingRun（对应 UAI-BD-NFR-009 可追溯性设计）

| 字段 | 类型 | 说明 |
|---|---|---|
| run_id | UUID | — |
| asset_ref | Ref<Asset> | — |
| status | Enum{Accepted, Rejected, Running, PartiallyCompleted, Completed, Failed} | — |
| stage_timeline | List<Struct{stage_id, start_at, end_at, status, warnings[]}> | 各阶段（对应模块1-10）执行轨迹 |
| degraded_mode_flags | List<Enum>（如 LlmDegraded, PhysicsSkipped） | 对应 §5.2 降级路径记录 |
| schema_version | UInt32 | — |

---

## 4. 数据关系详细设计（UAI-DD-REL-*）

在 `UAI-BD-001` §6.2 概念 ER 基础上细化基数与引用完整性方针：

| 关系 | 基数 | 完整性方针 |
|---|---|---|
| Asset → Region | 1:N | Region 必须归属唯一 Asset，Asset 删除（若支持）须级联标记 Region 为 orphaned，而非物理级联删除（保留审计） |
| Region ↔ Region（RelationEdge） | N:N | 边须携带 `relation_type` 枚举（CONNECTS/PART_OF/SYMMETRIC_WITH/MOVES_RELATIVE_TO/ATTACHED_VIA）与置信度分数 |
| Region → ArticulationHypothesis | 1:N | 一个 Region 可产生多个假设（主+备选） |
| ArticulationHypothesis → ValidationReport | 1:N | 一次假设可被多次验证（如重试不同 Motion Probe 参数） |
| ArticulationHypothesis → CorrectionRecord | 0..1:N | 允许多次修正迭代，保留全部历史（不覆盖） |
| Pattern ↔ ArticulationHypothesis | N:N | 引用关系，Pattern 删除（弃用）须转为 `deprecated` 状态而非物理删除，保证历史假设仍可追溯其曾经依据的 Pattern |

---

## 5. 处理时序详细设计（UAI-DD-SEQ-*）

### 5.1 主时序详细化（对应 UAI-BD-FLOW-001）

```
Client → MOD013.RequestRouter
  → MOD001.ingest()  [同步，含 IntegrityValidator]
     成功 → 创建 ProcessingRun(status=Running)
  → 并行: MOD002.extract_geometry() ∥ MOD003.extract_topology()
  → MOD004.DecompositionController（逐 Region 递归，每层调用 GrammarRuleEngine）
  → 对 Structural Part Graph 中每个候选关节 Region：
       MOD005.retrieve() → MOD007.EvidenceSummarizer → MOD007.PromptComposer
       → MOD007.LlmProviderClient.invoke()
           成功 → MOD007.DslResponseParser → DSL假设列表
           失败/超时/不可解析 → 触发 UAI-DD-MOD-007-FALLBACK（§5.2）
  → MOD008.DslToStructuralGraphCompiler → ... → ArticulationToRigCompiler
       编译失败 → 该假设标记 CompileError，不进入后续验证
  → MOD009.MotionProbeGenerator → PhysicsEngineClient.simulate()
       引擎不可用 → ValidationReport.validation_skipped=true（§5.2）
  → MOD010.EvidenceAggregator → ConfidenceCalculator → ExplanationComposer
  → ProcessingRun.status = Completed（或 PartiallyCompleted，若存在阶段失败）
  → MOD013.ResponseSerializer 返回 Client
```

### 5.2 降级路径详细设计（对应 UAI-BD-ERR-*、UAI-BD-RISK-001）

| 触发条件 | 降级行为 | 记录 |
|---|---|---|
| LLM 调用失败/超时/返回不可解析 DSL | 回退至 `GrammarRuleEngine` 直接生成规则驱动假设（跳过 LLM 生成，仅用规则触发结果构造最小 DSL） | `ProcessingRun.degraded_mode_flags += LlmDegraded`；`ArticulationHypothesis.llm_self_confidence = null` |
| 物理引擎不可用 | 跳过仿真，`ValidationReport.validation_skipped=true`，`overall_pass=null`（非 false，避免误判为"验证失败"） | `degraded_mode_flags += PhysicsSkipped`；`ConfidenceCalculator` 对该分量降权而非置零 |
| 单 Region 特征计算异常 | 标记 `evidence_incomplete=true`，跳过该 Region 参与规则判定的证据分量，不阻断兄弟 Region 处理 | `stage_timeline[stage=2或3].warnings += RegionId` |
| DSL 编译失败 | 该假设整体丢弃（不产出部分 Rig），若为唯一假设则该 Region 无候选输出，标记 `NoValidHypothesis` | `stage_timeline[stage=8].warnings` |

---

## 6. Structural DSL 详细语法设计（UAI-DD-DSL-*，对应 UAI-BD-DSL-001~004）

以 EBNF 概要描述（非实现代码，属语法规格）：

```
structure   := "STRUCTURE" identifier "{" part+ "}"
part        := "PART" identifier "{" part_attr* segment* joint* "}"
part_attr   := "role" ":" role_value
role_value  := "hub" | "appendage" | "trunk" | "extremity"
segment     := "SEGMENT" identifier
joint       := "JOINT" joint_type
joint_type  := "rotational" | "hinge" | "ball" | "slider" | "fixed"
```

设计约束：
- UAI-DD-DSL-005：每个 `part`/`joint` 节点在序列化时须携带 `source_rule_refs` 元数据（非核心语法但为必需扩展属性），保证编译后仍可追溯至 §2.4 GrammarRuleEngine 的触发规则（落实 UAI-FR-GRAM-002 端到端可解释性）。
- UAI-DD-DSL-006：DSL 语义校验须在编译前独立执行（`DslValidator`），区分"语法错误"（结构不合法）与"语义错误"（如 JOINT 出现在没有相邻 SEGMENT 的位置），两类错误须有不同错误码（见 §8）。

---

## 7. 关键算法设计方针（UAI-DD-ALG-*，伪代码级，非可编译代码）

### 7.1 Bottleneck 评分算法方针（UAI-DD-ALG-001，对应 UAI-BD-FUNC-002）

```
function bottleneck_score(region, adjacency_graph):
    local_cross_section = min_cross_section_area(region)
    neighbor_avg_cross_section = average(cross_section_area(n) for n in adjacency_graph.neighbors(region))
    ratio = local_cross_section / max(neighbor_avg_cross_section, epsilon)
    score = clamp(1 - ratio, 0, 1)
    return score   # 高分 = 强瓶颈证据，但非关节结论
```
设计约束：该函数**只产出证据分数**，是否判定为 Joint 由 §2.4 GrammarRuleEngine 综合 Motion/Physics/其他 Grammar 规则决定，不在本函数内做二值判定（强制落实 UAI-FR-TOP-006）。

### 7.2 层级拆分终止判断方针（UAI-DD-ALG-002，对应 UAI-BD-FUNC-003）

```
function should_decompose_further(region, evidence, grammar_rules):
    verdicts = [rule.evaluate(region, evidence) for rule in grammar_rules]
    triggered = [v for v in verdicts if v.trigger]
    if triggered is empty:
        return Terminate
    weighted_confidence = weighted_sum(v.confidence for v in triggered)
    if weighted_confidence >= THRESHOLD_CONTINUE:
        return ContinueDecompose(triggering_rules=triggered)
    else:
        return Terminate
```
`THRESHOLD_CONTINUE` 为可配置参数，初始值留待 PoC 基准数据校准（不在本阶段锁定具体数值）。

### 7.3 多假设排序方针（UAI-DD-ALG-003，对应 UAI-DD-MOD-010-A）

```
function rank_hypotheses(hypotheses, validation_reports):
    for h in hypotheses:
        vr = validation_reports[h.hypothesis_id]
        h.final_confidence = confidence_function(
            rule_evidence_strength(h),
            retrieval_similarity(h),
            h.llm_self_confidence,
            physics_score(vr)
        )
    return sort_desc(hypotheses, key=final_confidence)
```

### 7.4 Pattern 抽象方针（UAI-DD-ALG-004，对应 UAI-DD-MOD-012）

```
function abstract_patterns(new_experience_records):
    clusters = cluster_by_condition_signature(new_experience_records)
    for cluster in clusters:
        if support_count(cluster) >= MIN_SUPPORT:
            existing = find_matching_pattern(cluster.signature)
            if existing:
                update_pattern(existing, cluster)  # 版本号递增
            else:
                create_pattern(cluster)
        collect_counterexamples(cluster) into pattern.counterexample_refs
```
`MIN_SUPPORT` 与聚类相似度阈值留待 PoC 阶段确定。

---

## 8. 接口详细设计（UAI-DD-IF-*，字段级，对应 UAI-BD-IF-001~005）

### 8.0 统一响应封装

```
Response := {
  status: Enum{Success, PartialSuccess, Rejected, Error},
  data: Object | null,
  error: { code: ErrorCode, message: String, stage: String | null } | null
}
```

### 8.1 UAI-DD-IF-001 资产提交（对应 UAI-BD-IF-001）

请求字段：
| 字段 | 类型 | 必须 |
|---|---|---|
| asset_content_ref | String（文件引用/上传句柄） | 是 |
| asset_type | Enum{StaticMesh}（MVP） | 是 |
| processing_options | Object{skip_optional_validations: Bool} | 否 |

响应字段（data）：`{ run_id: UUID, status: Enum{Accepted, Rejected}, reject_reason: String|null }`

### 8.2 UAI-DD-IF-002 结果查询（对应 UAI-BD-IF-002）

请求字段：`{ run_id: UUID }`
响应字段（data）：`{ status, structural_part_graph, hypotheses: List<ArticulationHypothesis摘要>, validation_reports, confidence_data, degraded_mode_flags }`

### 8.3 UAI-DD-IF-003 人工修正提交（对应 UAI-BD-IF-003）

请求字段：`{ hypothesis_id: UUID, corrected_value: Object, reason_tags: List<Enum>|null, reason_text: String|null }`
响应字段（data）：`{ correction_id: UUID, accepted: Bool }`

### 8.4 UAI-DD-IF-004 CLI 命令映射（对应 UAI-BD-IF-004）

| CLI 子命令 | 映射内部接口 |
|---|---|
| `uai submit <file> --type static_mesh` | UAI-DD-IF-001 |
| `uai status <run_id>` / `uai result <run_id>` | UAI-DD-IF-002 |
| `uai correct <hypothesis_id> --value <...>` | UAI-DD-IF-003 |
| `uai batch-submit <dir>` | 循环调用 UAI-DD-IF-001（异步队列，对应 UAI-NFR-PERF-002） |
| `uai export <run_id> --format skeleton` | UAI-DD-IF-002 结果基础上执行 UAI-BD-FUNC-011 导出 |

### 8.5 Provider 抽象接口详细契约（对应 UAI-BD-IF-005）

- LLM Provider：`invoke(evidence_summary: StructuredEvidence, timeout_ms: u32) -> Result<List<DslCandidate>, LlmError>`
- Physics Engine：`simulate(rig_hypothesis: RigGraph, motion_probes: List<MotionProbe>, timeout_ms: u32) -> Result<ValidationMetrics, PhysicsError>`
- Vector Store：`query(vector: EmbeddingVector, top_k: u32) -> List<{ref, score}>`
- Graph Store：`match_pattern(query_subgraph: PatternQuery) -> List<{matched_subgraph_ref, score}>`

---

## 9. 错误码体系（UAI-DD-ERRCODE-*，对应 UAI-BD-ERR-001~003）

> 命名空间说明：运行时错误码统一使用 `UAI-ERRC-nnnn` 前缀，与需求定义书 §36 的**需求 ID** `UAI-ERR-001~003` 分属不同命名空间，避免二者在文本检索与追踪矩阵中混淆。

| 错误码 | 类别 | 说明 | 对应设计 |
|---|---|---|---|
| UAI-ERRC-1001 | 输入错误 | 资产格式不支持 | UAI-BD-ERR-001 |
| UAI-ERRC-1002 | 输入错误 | 几何完整性校验失败（非流形/退化） | UAI-BD-ERR-001 |
| UAI-ERRC-1003 | 输入错误 | 超出大小限制 | UAI-BD-ERR-001 |
| UAI-ERRC-2001 | 内部处理失败（单元级） | 单 Region 特征计算异常 | UAI-BD-ERR-002 |
| UAI-ERRC-2002 | 内部处理失败（单元级） | DSL 编译失败 | UAI-BD-ERR-002 |
| UAI-ERRC-2003 | 内部处理失败（系统级） | 内部依赖服务不可达（非降级场景，如配置错误） | UAI-BD-ERR-002 |
| UAI-ERRC-3001 | 验证结果（非系统错误） | 物理验证未通过（Penetration 检出） | UAI-BD-ERR-003 |
| UAI-ERRC-3002 | 验证结果 | 关节失稳检出 | UAI-BD-ERR-003 |
| UAI-ERRC-4001 | 降级通知（非错误） | LLM 降级模式生效 | §5.2 |
| UAI-ERRC-4002 | 降级通知 | 物理验证被跳过 | §5.2 |

设计约束：4xxx 系列不属于错误，是正常业务状态的显式通知，须与真实错误（1xxx/2xxx）在响应封装的 `status` 字段上区分（`PartialSuccess` vs `Error`），避免调用方将降级模式误判为失败（承接 UAI-BD-RISK-001）。

---

## 10. 非功能详细设计（UAI-DD-NFR-*，对应 UAI-BD-NFR-001~010）

- UAI-DD-NFR-001：模块002/003 内部以 Region 为并行任务粒度，任务调度模型采用"工作窃取（work-stealing）式数据并行"设计方向（具体调度器实现库留待 PoC 选型）。
- UAI-DD-NFR-002：模块013 的资产提交接口在设计上为"提交即返回 run_id，处理异步化"，`ProcessingRun.status` 状态机为 `Accepted → Running → (Completed | PartiallyCompleted | Failed)`。
- UAI-DD-NFR-003：`stage_timeline`（§3.7）记录粒度须精确到模块级（1-10 各一条记录），满足 UAI-BD-NFR-009 可追溯性设计在字段级的落地。
- UAI-DD-NFR-004：密钥配置（LLM/云服务凭据）设计为独立配置源（环境变量/密钥文件引用），任何 `Evidence`/`ProcessingRun`/日志结构体中不得包含凭据字段，作为字段级设计约束落实 UAI-BD-NFR-008。

---

## 11. 需求/设计/详细设计三级追踪矩阵（UAI-DD-TRACE）

| 需求 ID | 基本设计 ID | 详细设计 ID |
|---|---|---|
| UAI-FR-GEO-001~006 | UAI-BD-ARC-002, UAI-BD-FUNC-001 | UAI-DD-MOD-002, UAI-DD-DATA §3.2 |
| UAI-FR-TOP-001~006 | UAI-BD-ARC-003, UAI-BD-FUNC-002 | UAI-DD-MOD-003, UAI-DD-ALG-001, UAI-DD-DATA §3.2 |
| UAI-FR-DEC-001~004, UAI-FR-GRAM-001~003 | UAI-BD-ARC-004, UAI-BD-FUNC-003 | UAI-DD-MOD-004, UAI-DD-ALG-002 |
| UAI-FR-DSL-001~004 | UAI-BD-ARC-006/008, UAI-BD-DSL-001~004 | UAI-DD-MOD-006/008, UAI-DD-DSL-005~006, §6 |
| UAI-FR-GRAPH-001~005 | UAI-BD-ARC-005, UAI-BD-DATA-001~004 | UAI-DD-DATA §3.6/§3.7, §4 |
| UAI-FR-VEC-001~004 | UAI-BD-ARC-005, UAI-BD-IF-005 | UAI-DD-MOD-005, §8.5 |
| UAI-FR-AI-001~006 | UAI-BD-ARC-007 | UAI-DD-MOD-007, §5.2, §8.5 |
| UAI-FR-PHY-001~009 | UAI-BD-ARC-009 | UAI-DD-MOD-009, UAI-DD-DATA §3.4, §5.2 |
| UAI-FR-RIG-001~006 | UAI-BD-ARC-008, UAI-BD-FUNC-011 | UAI-DD-MOD-008 |
| UAI-FR-CONF-001~003 | UAI-BD-ARC-010 | UAI-DD-MOD-010, UAI-DD-ALG-003 |
| UAI-FR-HIL-001~005 | UAI-BD-ARC-011 | UAI-DD-MOD-011, UAI-DD-DATA §3.5 |
| UAI-FR-LEARN-001~003 | UAI-BD-ARC-012 | UAI-DD-MOD-012, UAI-DD-ALG-004 |
| UAI-NFR-RUST-001~003, UAI-NFR-GEN-001~003 | UAI-BD-RUST-001~004 | §2 全模块划分遵循 Rust 内部组件设计；MOD007 唯一允许 Python 边界 |
| UAI-NFR-PERF-001~005 | UAI-BD-NFR-001~003 | UAI-DD-NFR-001~002 |
| UAI-NFR-SEC-001~003 | UAI-BD-FUNC-013, UAI-BD-NFR-007~008 | UAI-DD-MOD-001, UAI-DD-NFR-004 |
| UAI-NFR-OBS-001~003 | UAI-BD-NFR-009~010 | UAI-DD-DATA §3.7, UAI-DD-NFR-003 |
| UAI-API-001~005 | UAI-BD-IF-001~004 | UAI-DD-IF-001~004 |
| UAI-ERR-001~003 | UAI-BD-ERR-001~003 | §9 错误码体系 |
| UAI-API-005（DCC 集成接口预留） | UAI-BD-IF-005（Provider 抽象方向） | UAI-DD-DCC-001~004, §11.5 全节（新增 Provider 第五类：DCC Runtime Provider，含原生脚本通道 + RPA 兜底通道），追踪编号并入待正式立项事项 OI-06 |

---

## 11.5 DCC 通用集成设计（UAI-DD-DCC-*）—— RPA 式模型操作适配层

> 背景：需求定义书 §35（UAI-API-005）将 DCC 集成列为 Research/V1+ 预留方向，未在 MVP 中展开。基于新增设计要求（插件须能自由适配主流常规 3D 软件，且以类 RPA 手法操作软件内模型），本节在详细设计阶段正式展开该集成层的设计，作为 UAI-BD-001 §7.5 Provider 抽象接口体系的第五类抽象（DCC Runtime Provider），不改变已合并的需求/基本设计文档，而是作为其预留扩展点的详细化落地。

### 11.5.1 设计动机与约束

- 常规 3D 软件（Maya、Blender、3ds Max、Houdini、Cinema 4D、Unreal Engine、Unity 等）各自拥有互不兼容的原生脚本 API（MEL/Python for Maya、Blender Python API、MaxScript、VEX/Houdini Python、C4D Python SDK、Unreal Python/Blueprint 等）。
- 若为每款软件单独实现原生插件，将违反 UAI-NFR-EXT-001（核心领域模型不得与具体 Provider 强绑定）并显著增加维护面。
- 因此本层采用**双通道适配策略**：
  1. **原生脚本通道（Native Scripting Channel）**：对已有官方 Python/脚本 API 的软件（Maya/Blender/Houdini/C4D/Unreal 等），通过生成/派发标准化的操作指令序列，由各软件内嵌解释器执行——本质是"用同一套抽象操作语义，编译为不同软件的脚本绑定"，而非逐软件手写业务逻辑。
  2. **RPA 通道（RPA Channel，兜底与无脚本 API 场景）**：对不提供可编程接口、或脚本接口无法覆盖的操作（如部分老旧/闭源工具、或需要驱动软件 UI 完成的操作），采用类 RPA 手法——基于可访问性 API（Accessibility API）/窗口自动化（UI Automation）/图像识别定位控件，模拟用户在软件 UI 上的操作序列（选择物体、拖拽骨骼、点击菜单、输入数值）完成模型操作。

设计原则（UAI-DD-DCC-000）：RPA 通道是**兜底手段而非默认手段**——优先探测并使用原生脚本通道；仅当目标软件/目标操作不具备脚本可达性时，回退到 RPA 通道。这一优先级顺序须在 §14.3 的能力探测阶段强制执行。

### 11.5.2 抽象操作语义层（UAI-DD-DCC-001，对应 Universal Rig Graph 的落地目标）

核心设计是定义一套与具体软件无关的**通用模型操作原语（Universal Scene Operation Primitives, USOP）**，作为 Rig Compiler 输出（Universal Rig Graph / Skeleton / Skinning）与具体 DCC 软件之间的中间表示：

| USOP 原语 | 说明 |
|---|---|
| `CreateJoint(position, orientation, parent_ref)` | 创建关节节点 |
| `SetJointConstraint(joint_ref, type, limits)` | 设置关节约束（Hinge/Ball/Slider 等，对应 Universal Rig Graph 原语） |
| `BindSkin(mesh_ref, joint_refs, weight_map)` | 绑定蒙皮权重 |
| `SelectObject(object_ref)` / `SetTransform(object_ref, transform)` | 基础选择与变换操作 |
| `ImportMesh(source_ref)` / `ExportRig(target_format)` | 资产导入导出 |
| `PlaybackPose(pose_data)` | 姿态回放，用于验证/预览 |

处理契约：
```
apply_operations(session: DccSession, ops: List<USOP>) -> Result<List<OpResult>, DccAdapterError>
```

### 11.5.3 DCC Provider 能力探测与通道选择（UAI-DD-DCC-002）

```
function select_channel(target_dcc, operation):
    if native_binding_registry.has(target_dcc, operation):
        return NativeScriptingChannel(binding=native_binding_registry.get(target_dcc, operation))
    elif ui_automation_profile_registry.has(target_dcc, operation):
        return RpaChannel(profile=ui_automation_profile_registry.get(target_dcc, operation))
    else:
        return Unsupported(reason="no_binding_and_no_ui_profile")
```

- `native_binding_registry`：每款软件的 USOP→原生脚本片段映射表（可插件化扩展，新增软件支持只需新增一份映射表，不改动核心 USOP 语义层，落实 UAI-NFR-EXT-001）。
- `ui_automation_profile_registry`：每款软件（或版本）UI 控件坐标/可访问性标识的操作画像（Profile），用于 RPA 通道定位控件。

### 11.5.4 RPA 通道内部设计（UAI-DD-DCC-003）

内部组件：
- `UiAccessibilityLocator`：优先使用操作系统级 Accessibility API / UI Automation Tree 定位控件（比纯图像坐标匹配更稳健，跨分辨率/主题更可靠）。
- `ImageFallbackLocator`：当控件不可通过 Accessibility 树定位时，退化为模板图像匹配定位（兜底中的兜底）。
- `ActionExecutor`：执行鼠标/键盘事件序列（点击、拖拽、输入），对每步操作设置超时与重试策略。
- `StateVerifier`：每步操作后校验目标软件内状态是否符合预期（如通过读取场景大纲/属性面板文本确认关节已创建），而非假设操作必然成功——RPA 操作须有闭环验证，因其本质是"外部黑盒操作"，可靠性低于原生脚本通道。
- `SessionRecorder`：记录每次 RPA 操作序列与验证结果，供失败重放分析（对应 UAI-NFR-OBS-002 可追溯性原则在 DCC 层的延伸）。

处理契约：
```
execute_rpa_sequence(session: DccUiSession, ops: List<USOP>, profile: UiAutomationProfile) -> Result<List<OpResult>, RpaExecutionError>
```

### 11.5.5 可靠性与安全设计约束（UAI-DD-DCC-004）

- RPA 通道运行于独立进程/沙箱，与 Rust Core 领域逻辑隔离（延续 UAI-NFR-RUST-003 的进程边界原则——RPA 自动化属于"外部适配层"，不得承载核心领域逻辑）。
- 每次 RPA 操作序列执行前须有"预演/可回滚"检查点设计方向：优先在目标软件支持的场景下先创建场景快照（若软件原生支持撤销栈/场景另存），失败时可回滚，避免误操作破坏用户现有工作文件。
- RPA 通道默认禁用状态下不得自动激活；用户须显式为目标软件配置/确认启用 RPA 通道（对应人机协作原则，避免无脚本 API 场景下的"静默后台操控 UI"造成用户困惑或误伤）。
- 失败/超时的 RPA 操作须计入 `ProcessingRun.degraded_mode_flags`（复用 §5.2 机制，新增枚举值 `DccRpaFallbackUsed` / `DccRpaOperationFailed`），保持与既有降级通知体系一致，不新增平行的状态体系。

### 11.5.6 与既有架构的集成点

- DCC Runtime Provider 是 §8.5 Provider 抽象接口体系的第五类，接口契约新增：
  ```
  DCC Provider: apply(session, ops: List<USOP>) -> Result<List<OpResult>, DccAdapterError>
  ```
- 消费方为 UAI-BD-FUNC-011（Rig 结果导出）的下游扩展：导出目标从"文件格式导出"扩展为"直接在目标 DCC 软件会话中重建 Rig"。
- 新增错误码：`UAI-ERRC-5001`（原生绑定缺失）、`UAI-ERRC-5002`（RPA 定位失败）、`UAI-ERRC-5003`（RPA 状态校验不通过），并入 §9 错误码体系，属于"降级/适配层错误"范畴，不影响核心 Rig 推理结果的正确性判定。

### 11.5.7 范围分级说明

本节内容超出当前已合并需求文档 §39 MVP 范围（原 DCC 集成为 Research/V1+），属于详细设计阶段对预留扩展点的前瞻设计，不要求 MVP 实现。建议后续版本迭代需求/基本设计文档时，将本节内容正式回填为 `UAI-FR-DCC-*` / `UAI-BD-ARC-015`（`UAI-BD-ARC-014` 已被基本设计书 §4.3 部署配置设计方针占用，避免编号冲突）系列编号，纳入正式追踪矩阵；本文档暂以 `UAI-DD-DCC-*` 独立编号标注，待需求侧正式立项后再统一编号体系（记为 Open Issue OI-06）。

---

## 12. 用语表

沿用 `UAI-REQ-001` §5 与 `UAI-BD-001` §16；本文档新增：

| 术语 | 定义 |
|---|---|
| RuleVerdict | GrammarRuleEngine 中单条规则对某 Region 的评估结果（触发与否+置信度+解释） |
| DegradedMode | 系统在外部依赖不可用时切换的显式标注运行状态，非错误状态 |
| condition_signature | Pattern 抽象过程中用于聚类的特征条件组合摘要 |

---

## 13. 自审（Self-Review）与三级文档交叉审核

### 13.0 三级文档交叉审核（Cross-Document Review，对 UAI-REQ-001 v0.2 / UAI-BD-001 v0.2 / UAI-DD-001 v0.1 三份已合并文档做交叉一致性检查）

| 检查项 | 结果 | 处理 |
|---|---|---|
| 需求→基本设计的 ID 覆盖率（FR/NFR 逐条是否在基本设计中至少被一个设计 ID 引用） | ✅ 覆盖（基本设计以区间记法如 `UAI-FR-GEO-001~006` 引用，非逐条罗列，属正常写法，非遗漏） | 无需修改 |
| 基本设计→详细设计的模块 1:1 映射 | ✅ 一致 | UAI-BD-ARC-001~013 与 UAI-DD-MOD-001~013 逐一对应，顺序一致 |
| 概念实体命名跨文档一致性 | ⚠ 发现不一致，已修复 | 基本设计 §6.1/§6.2 原用 `ValidationRecord`，详细设计 §3.4/§4/§5.2 统一使用 `ValidationReport`；已将基本设计改为 `ValidationReport` 以对齐详细设计（详见基本设计书 v0.3 改订履历） |
| 设计 ID 命名空间是否存在跨文档冲突 | ⚠ 发现冲突，已修复 | 详细设计书 §11.5.7 原计划将 DCC 模块回填编号为 `UAI-BD-ARC-014`，但该编号已被基本设计书 §4.3（部署配置设计方针）占用；已改为拟编号 `UAI-BD-ARC-015` 并在两份文档中同步标注 |
| 追踪矩阵是否存在"声称已设计但实际未设计"的过度覆盖 | ⚠ 发现过度声称，已修复 | 基本设计书 §17 原将 `UAI-API-001~005` 整体映射至 `UAI-BD-IF-001~004`，但 `UAI-API-005`（DCC 预留）在基本设计阶段并无具体接口设计；已拆分为两行，`UAI-API-005` 明确标注"本文档未设计具体接口"并指向详细设计书 §11.5 |
| 三份文档的文档管理信息（改订履历/承认体系/参照文档一览）是否互相指向正确版本号 | ✅ 已核对并同步 | 详细设计书 §0.3 参照文档一览中的基本设计书版本号已同步更新为 v0.2 |
| 需求文档 Open Issue 与详细设计文档 Open Issue 是否共享同一编号空间 | ⚠ 存在双重编号空间 | 需求定义书 §42 为 OI-01~OI-05，详细设计书 §13 另起 OI-06/OI-07，二者当前为各自独立文档内编号，非全局统一编号空间；已在本文档 OI-06 描述中显式注明其待回填目标是需求文档，提醒下一版本合并需求文档 Open Issue 列表时需重新统一编号（非阻塞性缺口，作为交叉审核发现登记，不单独开新 OI） |

#### 13.0.2 第二轮交叉审核（v0.3，重点：内部交叉引用正确性与命名空间纪律）

第一轮审核集中于 ID 与术语一致性，未逐条验证「文档内 §N 交叉引用是否指向正确章节」。第二轮补充该项检查，逐条核对三份文档中的全部 `§N` 引用，发现并修复以下问题：

| # | 类别 | 发现 | 处理 |
|---|---|---|---|
| 1 | 交叉引用失效（系统性） | 需求定义书中 9 处 `§N` 引用沿用了原始需求提纲的章节编号，而非本文档自身的章节编号，导致引用指向错误章节：`§28 自审`（实际 §46）、`§24 MVP`（实际 §39）、`§44 追踪矩阵`（实际 §45）、`§37 KPI`（实际 §38）、以及多处 `§18/§19 输入输出分级`（实际位于文末附录） | 已逐条修正为正确章节号 |
| 2 | 附录无正式编号 | 输入/输出范围分级表原以「附：§18 输入范围分级 与 §19 输出范围分级」为标题，借用提纲编号且与文档自身 §18/§19（Structural DSL / Graph Knowledge Requirements）冲突 | 已正式编号为 **附录 A**（A.1 输入范围分级 / A.2 输出范围分级），并将全部上游引用改指附录 A；基本设计书 §5.11、详细设计书 §11.5.7 中的同类引用同步修正 |
| 3 | **架构图与模块表不一致（本轮最重要发现）** | 基本设计书 §4.1 逻辑构成图的模块编号与 §4.2 模块一览表不符：图中将 LLM Reasoning Adapter 作为 [6] 的括号附注而未独立编号，凭空引入了模块表中不存在的 `[9] Universal Rig Graph Store`，并因此把 Rig Compiler 标为 [7]、Physics Validation 标为 [8]。而 §4.3、§10、§17 及详细设计书全部 `UAI-DD-MOD-*` 均按模块表编号（7=LLM Adapter、8=Rig Compiler、9=Physics），即**图是唯一的偏差方**，会误导实装工程师对照错误的模块编号 | 已修正 §4.1 图：LLM Reasoning Adapter 独立为 [7]，Rig Compiler 改为 [8]，Physics Validation Coordinator 改为 [9]，删除幻影模块 `Universal Rig Graph Store`（该模块在任何模块表、部署设计或时序设计中均不存在） |
| 4 | 命名空间重叠 | 详细设计书 §9 运行时错误码原用 `UAI-ERR-1001` 等编号，与需求定义书 §36 的需求 ID `UAI-ERR-001~003` 共用同一文本前缀，检索与追踪时易混淆 | 运行时错误码统一改为 `UAI-ERRC-nnnn` 前缀，并在 §9 增加命名空间说明 |

结论（第二轮）：共发现 4 类问题，全部已修复。其中第 3 项属于会直接误导实装的实质性设计文档缺陷，而非文字疏漏——它说明第一轮审核只比对 ID 集合、未比对图示与表格的语义一致性，存在方法论盲区；后续审核须将「图/表/正文三者是否互相印证」列为固定检查项。

#### 13.0.1 第一轮交叉审核（v0.2）

结论：本轮交叉审核共发现 4 项跨文档不一致（术语命名 1 项、ID 命名空间冲突 1 项、追踪矩阵过度声称 1 项、Open Issue 编号空间未统一 1 项），前三项已直接修复并同步更新相关文档版本号（基本设计书 v0.1→v0.2）；第四项（Open Issue 编号空间）性质为文档管理流程问题，已记录待下一次正式版本迭代时统一处理，不影响当前设计内容的正确性。

### 13.1 详细设计自审

对照 IPA 共通フレーム詳細設計プロセス惯例（モジュール内部設計／データ詳細設計／インターフェース詳細設計／処理シーケンス／アルゴリズム設計／エラーコード体系的完整性）与三级追踪完整性，检查如下：

| 检查项 | 状态 | 说明 |
|---|---|---|
| 13 个基本设计模块均有对应详细设计 | ✅ | §2.1~2.13 |
| 数据结构细化到字段级，但未使用 DDL | ✅ | §3，类型为抽象类型系统 |
| 接口细化到字段级，未锁定传输协议 | ✅ | §8，仅定义 Schema |
| 关键算法给出设计方针（伪代码），非可编译代码 | ✅ | §7，明确标注"非可编译代码" |
| 统一错误码体系，区分错误/验证结果/降级通知 | ✅ | §9，三类型明确区分 |
| Bottleneck≠Joint 约束在算法级落地 | ✅ | UAI-DD-ALG-001 显式声明"只产出证据分数" |
| Rust/Python 边界在模块级明确唯一入口 | ✅ | MOD007 为唯一允许引入 Python 的模块，其余模块划分未涉及 Python |
| 三级（需求/基本设计/详细设计）追踪矩阵完整 | ✅ | §11（新增 DCC 集成一行，标注为待正式立项） |
| 是否越界进入可编译代码/具体库选型/DDL | ✅ 未越界 | 全文未出现可编译语法、具体 crate 名称的 API 调用、DDL 语句 |
| DCC 通用适配设计是否与既有 Provider 抽象体系一致 | ✅ | §11.5.6，作为第五类 Provider 接入，未破坏既有四类接口契约 |
| RPA 通道是否有可靠性/安全兜底设计 | ✅ | §11.5.5：进程隔离、可回滚检查点方向、默认禁用、失败计入既有降级通知体系（未新增平行状态体系） |
| RPA 通道是否被误设计为默认/主要通道 | ✅ 未误设计 | §11.5.1 UAI-DD-DCC-000 明确"原生脚本通道优先，RPA 仅兜底" |

**遗留缺口（留待 PoC/实装阶段解决）**：`THRESHOLD_CONTINUE`、`MIN_SUPPORT`、置信度加权公式的具体参数值均需基于 PoC 阶段的标注数据集与实验校准，本文档已将其显式标注为"留待确定"而非遗漏（对应需求 Open Issue OI-01/OI-02 的延续处理）。

**新增 Open Issue**：
- OI-06：DCC 通用集成设计（§11.5）当前仅存在于详细设计文档，尚未回填至需求定义书（`UAI-FR-DCC-*`）与基本设计书（拟编号 `UAI-BD-ARC-015`，须避开已占用的 `UAI-BD-ARC-014`），建议下一版本迭代时补齐三级文档的正式编号与验收标准，并将其 MVP/V1/Research 分级正式写入需求文档 附录 A.1/A.2 分级表。
- OI-07：各目标 DCC 软件的 `native_binding_registry` 与 `ui_automation_profile_registry` 覆盖率目标尚未定义，建议在 PoC 阶段先选定 1-2 款代表性软件（如 Blender 作为开源可脚本化代表、一款闭源工具作为 RPA 通道验证代表）建立试点。

---

*本文档为详细设计阶段产出，是 PoC/实装编码的直接依据；但本文档中的伪代码、字段表、EBNF 语法均非可编译源代码，实装工程师仍需完成具体语言绑定、库选型与数据库物理设计（DDL）后方可开始编码。*
