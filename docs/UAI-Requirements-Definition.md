# Universal Articulation Intelligence — 需求定义书

版本: v0.1 (Draft)
文档状态: 需求阶段 (Requirements Definition Only) — 禁止在本阶段进行详细设计、类图、数据库物理设计或代码实现
作者: AI Research / Architecture (assisted)
日期: 2026-08-18

---

## 1. 文档目的

本文档定义 **Universal Articulation Intelligence（UAI）** 系统的正式需求边界。目的是为后续「基本设计 → PoC → 实验验证」阶段提供一份完整、可追踪、可验收的需求基线。本文档只回答"系统应该做什么、为什么、验收标准是什么"，不回答"系统具体怎么实现"。

本文档禁止包含：实现代码、详细设计、类图、数据库物理表设计、微服务拆分代码。任何此类内容如出现均视为文档缺陷。

---

## 2. 项目背景

传统 Auto-Rigging 工具的共同假设是：给定一个已知类别（人形、四足动物等）的网格，将预定义模板（Human Template、Dog Template 等）拟合到网格上，产出骨骼。这类方法本质上是**模板匹配**，无法处理：

- 未知类别的对象（怪物、AI 生成角色、非对称生物、混合机械结构、从未存在过的虚构对象）；
- 需要根据几何、拓扑、运动、物理证据**推导**其结构与关节关系的对象，而非查表；
- 结构可能不是传统骨骼形式（软体、腱、弹性链、程序化形变）的对象。

行业中同时缺少一个统一框架，把 Geometry、Topology、Motion、Semantics、Physics、历史结构经验综合起来，进行可解释、可验证、可自我修正的结构推理。

## 3. 问题定义

核心问题不是"骨头应该放在哪里"，而是：

> 一个三维结构应该如何拆分、如何连接、如何运动，以及为什么能够如此运动？

Skeleton 只是该问题的一种可能输出，而非问题本身。系统必须先建立对"结构"与"关节可能性"的理解，再将该理解编译为可执行的 Rig 表示。

## 4. 产品愿景

构建一个能够理解任意三维对象结构组成、连接方式、可动部件及其运动依据，并将这种理解转换为可执行 Rig 的系统。系统以 Rust 为核心运行时，具备多假设推理、物理验证、经验学习闭环与人机协作能力，目标是持续降低专业 Rig 美术师将 AI 输出打磨到生产可用状态所需的人工修正时间。

---

## 5. 术语定义

| 术语 | 定义 |
|---|---|
| Structural Evidence | 由 Geometry / Topology / Motion / Physics / 历史模式抽取出的、可供推理使用的量化或符号化特征集合 |
| Structural Decomposition | 将三维对象递归拆分为 Region / Part / Sub-part / Segment 的层级化过程 |
| Articulation Hypothesis | AI 对某个结构区域是否存在关节、关节类型及运动方式提出的候选判断，附带置信度 |
| Topological Bottleneck | 网格连通结构中出现的局部收缩/狭窄区域，是判断关节位置的证据之一，**不等价于关节** |
| Structural Grammar / Articulation Grammar | 描述"结构为何应当如此拆分/连接"的规则集合（如 Bottleneck Rule、Chain Rule 等） |
| Structural DSL | 一种领域特定语言，用于表达高层结构描述，而非直接的骨骼坐标 |
| Instance Graph | 存储单个资产实际结构关系的图 |
| Experience Graph | 存储 AI 判断、验证结果、人工修正等历史处理记录的图 |
| Pattern / Rule Graph | 从大量实例中抽象出的、可复用结构规律的图，附带 Confidence/Support/Evidence |
| Universal Rig Graph | 能够表达 Bone、Hinge、Ball Joint、Slider、Spring、Muscle、Tendon、Cable、Soft Structure 等多种运动/连接原语的统一 Rig 表示 |
| Physics-in-the-loop | 结构假设生成后经物理仿真验证并反馈修正的闭环机制 |
| Confidence Calibration | AI 输出的置信度与实际正确率之间的一致程度 |
| Human Correction Time | 专业 Rig Artist 将 AI 输出修正至生产可用状态所需的时间，核心商业指标 |

---

## 6. 项目目标

- G1: 建立可处理未知类别三维对象的结构理解与关节发现能力（不依赖硬编码模板）。
- G2: 建立 Geometry / Topology / Motion / Physics / Semantics / 历史知识联合推理的架构基础。
- G3: 建立 Rust 为核心运行时的领域逻辑、数据契约与计算管线。
- G4: 建立多假设 + 物理验证 + 置信度输出的可解释推理流程。
- G5: 建立经验学习闭环，使成功与失败案例均可沉淀为可复用规律。
- G6: 建立人机协作机制，使人工修正被结构化记录并反馈进系统。
- G7: 以 MVP 严格范围验证核心假设的技术可行性，为后续版本扩展打基础。
- G8: 降低专业 Rig Artist 的人工修正时间（Human Correction Time）作为核心商业指标。

## 7. 非目标（Non-Goals，阶段性）

- 本阶段不追求完全替代人工 Rig Artist。
- 本阶段不追求支持所有输入类型（见 §18 输入范围分级）。
- 本阶段不追求完整 DCC（Maya/Blender/Houdini）插件生态集成。
- 本阶段不追求 Muscle/Cloth/Fluid 等高级形变系统的生产级实现。
- 本阶段不追求云端多租户 SaaS 化部署（Local-first 优先）。
- 本阶段不追求锁定具体第三方库/框架的最终选型。

---

## 8. Stakeholder

| Stakeholder | 关注点 |
|---|---|
| Rig Artist / TD | 减少手工装配时间，得到可解释、可修正的结构假设 |
| Technical Director / Pipeline Owner | 系统可集成进现有资产管线，性能与可扩展性 |
| AI Research Team | 验证通用结构推理与运动规律学习的研究假设 |
| Product / Business Owner | Human Correction Time 降低带来的成本效益 |
| Platform/Infra Engineer | 部署形态、可替换 Backend、可观测性 |
| 最终购买/使用工作室 | 生产可用性、结果可靠性、失败可追溯 |

## 9. 核心用户

- **专业 Rig Artist**：使用系统生成初始结构假设与 Rig 草案，进行审查与修正。
- **技术美术（TA）**：定制/审核 Structural Grammar 规则，调优置信度阈值。
- **Pipeline 工程师**：将系统接入资产处理管线，批量处理资产。
- **AI 研究人员**：分析 Experience Graph / Pattern Graph，评估结构学习效果。

---

## 10. User Journey（示例）

### Journey A：未知类别静态网格的初次装配

1. Artist 导入一个从未见过类别的静态网格（如虚构怪物）。
2. 系统抽取 Geometry / Topology 特征，构建 Region Graph。
3. 系统执行层级化结构拆分，产出 Structural Part Graph。
4. 系统检索历史相似结构（Vector + Graph Retrieval）。
5. LLM 基于 Structural Evidence 生成多个 Articulation Hypothesis。
6. 系统对每个假设执行简化物理验证，排序并输出置信度。
7. 系统产出 Joint Candidate 列表与 Universal Rig Graph 草案。
8. Artist 审查并修正低置信度关节；系统记录修正。
9. 修正结果连同验证分数写入 Experience Graph，供未来 Pattern Mining 使用。

### Journey B：具备运动证据的资产

1. Artist 提供网格序列或多视角视频作为运动证据。
2. 系统执行 Motion Segmentation，识别刚性簇与相对运动。
3. 系统将运动证据与几何/拓扑证据融合，生成更高置信度的 Articulation Hypothesis。
4. 后续流程与 Journey A 一致。

---

## 11. Use Case 列表（概要）

| ID | 用例 | 触发者 | 概述 |
|---|---|---|---|
| UC-01 | 静态网格结构分解 | Artist / Pipeline | 输入 Mesh，输出 Structural Part Graph |
| UC-02 | 关节假设生成 | 系统（自动） | 基于证据生成多假设 Articulation Hypothesis |
| UC-03 | 物理验证 | 系统（自动） | 对 Rig Hypothesis 执行 Motion Probe 与仿真校验 |
| UC-04 | 相似结构检索 | 系统（自动） | Vector + Graph 检索历史相似案例 |
| UC-05 | 人工修正录入 | Artist | 修改 AI 判断，系统记录修正原因与结果 |
| UC-06 | 经验回写与规律挖掘 | 系统（离线批处理） | 将成功/失败案例沉淀为 Pattern/Rule |
| UC-07 | 运动证据驱动结构发现 | Artist / Pipeline | 基于视频/序列证据推导结构与关节 |
| UC-08 | 置信度与解释查询 | Artist | 查看某关节判断的证据、置信度、备选假设 |
| UC-09 | 批量资产处理 | Pipeline 工程师 | 批量提交资产，异步获取结构与 Rig 结果 |

---

## 12. 系统边界

**系统内**：几何/拓扑特征抽取、结构拆分、Structural DSL 与 Compiler、图/向量知识库、LLM 结构推理协调层、物理验证协调（调用物理引擎，不自研物理引擎内核）、Universal Rig Graph 生成、置信度与解释输出、人工修正记录、经验学习闭环、CLI/API 接口。

**系统外（依赖外部能力，不在本系统内自研）**：底层物理仿真引擎内核、底层图数据库/向量数据库引擎本身、底层 LLM 模型训练、DCC 软件本体、渲染引擎。

**明确排除**：本系统不是通用 3D 建模工具，不是动画制作工具，不是纹理/材质系统。

---

## 13. 输入数据需求

见 §18（按 MVP / 后续版本 / 研究性分级）。

## 14. 输出数据需求

见 §19（按 MVP / V1 / Research 分级）。

---

## 15. Geometry Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-GEO-001 | 曲率特征抽取 | 系统须能计算网格表面曲率（Curvature）分布作为结构证据输入 | MUST |
| UAI-FR-GEO-002 | 厚度/体积特征抽取 | 系统须能估计局部厚度（Thickness）与体积（Volume）分布 | MUST |
| UAI-FR-GEO-003 | 形状与伸长度特征 | 系统须能计算 Elongation、Shape 描述子，用于识别"肢体状"结构 | MUST |
| UAI-FR-GEO-004 | 对称性检测 | 系统须能检测部件间的 Symmetry 关系 | SHOULD |
| UAI-FR-GEO-005 | 空间关系与测地距离 | 系统须能计算 Spatial Relation 与 Geodesic Distance 作为区域邻接证据 | MUST |
| UAI-FR-GEO-006 | 中轴/骨架结构提取 | 系统须能提取 Medial Structure（中轴面/骨架线）辅助结构分解 | SHOULD |

验收标准：以上特征须以结构化 Structural Evidence 形式输出，且可独立于 Topology 模块单独测试（见 §28 自审第 4 条）。

## 16. Topology Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-TOP-001 | 连通性与组件分析 | 系统须能计算 Connectivity、Component 划分 | MUST |
| UAI-FR-TOP-002 | 边界与缝合线检测 | 系统须能识别 Boundary、Seam | SHOULD |
| UAI-FR-TOP-003 | 分支与环路检测 | 系统须能识别 Branch、Loop 结构 | MUST |
| UAI-FR-TOP-004 | 瓶颈检测 | 系统须能检测 Topological Bottleneck 作为关节候选证据之一 | MUST |
| UAI-FR-TOP-005 | 区域邻接分析 | 系统须能构建 Region Adjacency / Structural Connectivity 图 | MUST |
| UAI-FR-TOP-006 | 瓶颈非关节等价约束 | 系统禁止将 Bottleneck 直接判定为 Joint；瓶颈须与其他证据（Motion/Physics/Grammar）联合判断 | MUST |

验收标准：任何"仅凭 Topological Bottleneck 直接输出 Joint"的实现路径视为不合规，须在测试用例中构造"存在瓶颈但非关节"的反例验证。

---

## 17. Structural Decomposition Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-DEC-001 | 层级化递归拆分 | 系统须支持 Object → Major Part → Sub Part → Segment 的递归结构拆分 | MUST |
| UAI-FR-DEC-002 | 拆分终止判断 | 每一层拆分须显式判断"当前 Region 是否应继续拆分"，而非固定深度 | MUST |
| UAI-FR-DEC-003 | 非模板依赖 | 拆分逻辑不得仅依赖 Human/Dog/Robot 等预定义模板作为唯一判据 | MUST |
| UAI-FR-DEC-004 | 未知类别支持 | 系统须能对训练/规则库中未出现过的类别对象执行拆分（见 KPI: Unknown Category Generalization） | MUST |
| UAI-FR-GRAM-001 | Structural Grammar 规则集 | 系统须定义并应用 Bottleneck Rule、Motion Independence Rule、Deformation Continuity Rule、Symmetry Rule、Repetition Rule、Chain Rule、Hub Rule、Physics Override Rule 作为拆分/连接判据 | MUST |
| UAI-FR-GRAM-002 | 规则可解释性 | 每条拆分/连接决策须能追溯到触发它的具体 Grammar 规则与证据 | MUST |
| UAI-FR-GRAM-003 | 规则可扩展性 | Structural Grammar 规则集须可增量扩展，不要求重新设计核心拆分引擎 | SHOULD |

## 18. Structural DSL Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-DSL-001 | 高层结构描述语言 | 系统须提供 Structural DSL，使 AI 输出高层结构描述而非直接骨骼坐标 | MUST |
| UAI-FR-DSL-002 | DSL 语义覆盖 | DSL 须能表达 PART、ROLE（如 hub）、SEGMENT、JOINT 类型等结构元素 | MUST |
| UAI-FR-DSL-003 | 确定性编译 | 系统须提供由 Rust 实现的确定性 Compiler，将 DSL 编译为 Structural Graph / Articulation Graph / Rig Graph / Skeleton / Constraints / Skinning Regions | MUST |
| UAI-FR-DSL-004 | DSL 与 LLM 解耦 | LLM 仅负责生成/选择 DSL 层描述，不直接生成最终坐标级 Rig 数据 | MUST |

业务意义：DSL 作为 LLM（不确定性系统）与 Compiler（确定性系统）之间的契约边界，使得"结构决策"可解释、可版本化、可回归测试，同时把坐标级精度计算交给确定性 Rust 代码，降低 LLM 数值幻觉风险。

---

## 19. Graph Knowledge Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-GRAPH-001 | Instance Graph | 系统须为每个资产维护 Instance Graph，记录 CONNECTS / PART_OF / SYMMETRIC_WITH / MOVES_RELATIVE_TO / ATTACHED_VIA 等关系 | MUST |
| UAI-FR-GRAPH-002 | Experience Graph | 系统须记录 AI 判断、Physics Validation 结果、人工修正、成功/失败标记，形成可查询历史 | MUST |
| UAI-FR-GRAPH-003 | Pattern/Rule Graph | 系统须能从大量实例抽象出结构规律，并保存 Confidence、Support、Evidence、Counterexample、Version | SHOULD (MVP 可用最小子集验证，见 §24) |
| UAI-FR-GRAPH-004 | 图数据库职责边界 | 图数据库不得用于存储原始 Mesh 顶点数据；仅存储结构关系、特征摘要引用与经验/规律数据 | MUST |
| UAI-FR-GRAPH-005 | 失败案例保留 | 失败的 Articulation Hypothesis 及其验证结果不得被丢弃，须写入 Experience Graph | MUST |

## 20. Vector Retrieval Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-VEC-001 | 多类型向量 | 系统须支持 GeometryVector、TopologyVector、MotionVector、PhysicsVector、SemanticVector 等节点向量 | MUST（MVP 至少 Geometry+Topology） |
| UAI-FR-VEC-002 | 相似结构检索 | 向量检索用于寻找几何/拓扑/运动相似的历史结构，不承担关系模式判断职责 | MUST |
| UAI-FR-VEC-003 | Vector→Graph 联动 | 检索流程须遵循 Unknown Structure → Vector Retrieval → Similar Historical Structures → Graph Matching → Structural Patterns → AI Reasoning | MUST |
| UAI-FR-VEC-004 | 职责边界声明 | 系统文档与实现须明确 Vector DB 不是知识系统本身，仅为相似性索引 | MUST |

---

## 21. LLM Reasoning Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-AI-001 | 输入为 Structural Evidence | LLM 输入须为结构化 Structural Evidence（几何/拓扑/运动/物理摘要 + 历史相似度），禁止直接输入原始顶点/百万级几何数据 | MUST |
| UAI-FR-AI-002 | 角色限定为结构推理者 | LLM 职责限定为 Evidence → Rule Selection → Structural Hypothesis → Alternative Hypothesis → Confidence，不承担 Geometry Engine 职责 | MUST |
| UAI-FR-AI-003 | 多假设输出 | LLM/推理层须输出至少一个主假设与至少一个备选假设（Alternative Hypothesis） | MUST |
| UAI-FR-AI-004 | 可解释输出 | 每个假设须附带触发它的证据列表与规则引用 | MUST |
| UAI-FR-AI-005 | LLM Provider 可替换 | LLM 推理层须通过抽象接口调用，不得与具体厂商 API 强绑定在核心领域逻辑中 | MUST |
| UAI-FR-AI-006 | 禁止默认正确假设 | 系统不得假设 LLM 输出默认正确；所有输出须经后续物理验证或规则校验 | MUST |

## 22. Physics Validation Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-PHY-001 | Physics-in-the-loop 流程 | 系统须实现 Rig Hypothesis → Motion Probe → Physics Simulation → Deformation → Validation → Correction 闭环 | MUST |
| UAI-FR-PHY-002 | 穿透检测 | 验证须覆盖 Penetration 检测 | MUST |
| UAI-FR-PHY-003 | 拉伸/体积检测 | 验证须覆盖 Stretching、Volume Loss 检测 | MUST |
| UAI-FR-PHY-004 | 关节稳定性检测 | 验证须覆盖 Joint Instability、Constraint Violation | MUST |
| UAI-FR-PHY-005 | 不可能运动检测 | 验证须覆盖 Impossible Motion 检测 | SHOULD |
| UAI-FR-PHY-006 | 权重不连续检测 | 验证须覆盖 Weight Discontinuity 检测 | COULD（V1） |
| UAI-FR-PHY-007 | 结构性坍塌检测 | 验证须覆盖 Structural Collapse 检测 | COULD（V1） |
| UAI-FR-PHY-008 | 验证驱动修正 | 验证失败须能反馈触发假设重排序或修正，而非仅报告失败 | MUST |
| UAI-FR-PHY-009 | 物理引擎可替换 | 物理仿真通过抽象接口调用，核心领域逻辑不得与具体物理引擎强绑定 | MUST |

---

## 23. Rig Generation Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-RIG-001 | Universal Rig Graph 表示 | 系统须提供能够抽象 Bone、Rigid、Hinge、Ball Joint、Slider、Spring、Muscle、Tendon、Cable、Elastic Chain、Soft Structure、Procedural Deformation 的统一图表示 | MUST（MVP 覆盖 Bone/Rigid/Hinge/Ball Joint 子集，其余 V1/Research，见 §19） |
| UAI-FR-RIG-002 | 非默认骨骼假设 | 系统不得默认所有对象必须输出传统 Skeleton；Skeleton 为 Universal Rig 的一种导出形式 | MUST |
| UAI-FR-RIG-003 | Skeleton 导出 | 系统须能将 Universal Rig Graph 导出为传统 Skeleton + Joint + Axis + DOF + Range of Motion 表示 | MUST |
| UAI-FR-RIG-004 | Skinning 区域与权重 | 系统须能产出 Skinning Region 划分与 Skin Weights（MVP 可用简化算法） | SHOULD（MVP 简化版，V1 生产级） |
| UAI-FR-RIG-005 | 物理约束导出 | 系统须能导出 Physics Constraint 描述 | SHOULD |
| UAI-FR-RIG-006 | Retarget 映射 | 系统须能产出 Retarget Mapping | COULD（V1/Research） |

## 24. Confidence Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-CONF-001 | 结构化置信度输出 | 每个重要 AI 判断须返回 Confidence、Evidence、Alternative、Reason、Validation Result | MUST |
| UAI-FR-CONF-002 | 正负证据区分 | 置信度说明须区分支持证据（Positive Evidence）与不利证据（Negative Evidence） | MUST |
| UAI-FR-CONF-003 | 置信度可校准性 | 系统须支持对 Confidence Calibration 进行离线评估（预测置信度 vs 实际正确率） | SHOULD |

## 25. Human-in-the-loop Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-HIL-001 | 人工修正录入 | 系统须支持记录 AI 原始判断与 Artist 修正结果 | MUST |
| UAI-FR-HIL-002 | 修正原因记录 | 修正记录须包含修改原因（结构化或自由文本） | SHOULD |
| UAI-FR-HIL-003 | 修正与验证分数关联 | 修正记录须关联 Physics Score 与 Final Accepted Result | MUST |
| UAI-FR-HIL-004 | 修正反哺学习闭环 | 人工修正记录须能进入 Experience Graph 供后续 Pattern Mining 使用 | MUST |
| UAI-FR-HIL-005 | 聚焦不确定区域 | 系统须能标出低置信度区域，引导人工优先处理，而非要求逐一全量审查 | SHOULD |

## 26. Learning Loop Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-FR-LEARN-001 | 闭环链路完整性 | 系统须实现 Structural Evidence → Graph/Vector Retrieval → AI Reasoning → Rig Hypothesis → Physics Validation → Success/Failure → Experience Graph → Pattern Mining → Rule Graph → Future Reasoning 完整链路 | MUST（MVP 可用最小闭环验证，见 §24） |
| UAI-FR-LEARN-002 | 失败不丢弃 | 失败案例须保留并参与后续规律挖掘，不得静默丢弃 | MUST |
| UAI-FR-LEARN-003 | 规律版本化 | Pattern/Rule Graph 中的规律须支持版本管理，允许规律随证据积累演化 | SHOULD |

---

## 27. Functional Requirements 汇总

功能需求已按领域分布于 §15–§26（编号前缀 UAI-FR-*）。本节仅作交叉索引，详见 §44 Requirement Traceability Matrix。

## 28. Non-functional Requirements

### 28.1 通用非功能需求

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-GEN-001 | 核心领域逻辑语言约束 | 核心业务逻辑、几何/拓扑处理、图结构处理、Rig 数据处理、Structural DSL Parser、Rig Compiler、物理验证协调、网络服务、Worker、CLI、数据流水线须使用 Rust 实现 | MUST |
| UAI-NFR-GEN-002 | Python 使用边界 | 仅当 Rust 生态缺少可用实现、某 AI/ML 框架必须通过 Python 调用、研究模型仅有 Python 官方实现，或引入 Python 能显著降低研究风险时，方可引入 Python，且仅限 AI 推理/训练适配层 | MUST |
| UAI-NFR-GEN-003 | Python 不得承载核心领域逻辑 | 核心领域模型、规则、数据契约、图结构、DSL、验证逻辑必须保留在 Rust 侧；Python 组件须通过明确边界接口（Adapter）与 Rust Core 交互 | MUST |

### 28.2 可扩展性

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-EXT-001 | Backend 可替换性 | LLM Provider、Geometry Backend、Graph Database、Vector Database、Physics Engine、GPU Runtime、AI Model 须通过抽象接口接入，核心领域模型不得与具体 Provider 强绑定 | MUST |

### 28.3 部署约束

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-DEPLOY-001 | Local-first | 系统首个可用形态须支持本地单机部署，无需依赖云服务 | MUST |
| UAI-NFR-DEPLOY-002 | 云端部署预留 | 架构须为未来 Cloud/k3s 部署预留扩展空间，但不得在需求阶段强制过早微服务化 | SHOULD |
| UAI-NFR-DEPLOY-003 | 部署形态分级 | 需求须区分 Library / Process / Worker / Service / GPU Worker 五种部署单元角色，具体拆分留待设计阶段 | MUST |

## 29. Performance Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-PERF-001 | 大网格处理能力 | 系统须能处理生产级规模网格（具体阈值在设计阶段基于基准测试确定） | MUST |
| UAI-NFR-PERF-002 | 批量资产处理 | 系统须支持批量资产异步处理（Batch Asset Processing） | SHOULD |
| UAI-NFR-PERF-003 | 并行几何处理 | 几何/拓扑特征抽取须利用并行计算（CPU 多核，必要时 GPU） | SHOULD |
| UAI-NFR-PERF-004 | 图查询性能 | Graph Query 须在交互式响应时间内完成典型查询（具体 SLA 留待设计阶段定义） | SHOULD |
| UAI-NFR-PERF-005 | Rust 性能特性利用 | 实现须优先利用 Memory Safety、Zero-cost Abstraction、Parallelism、SIMD、Async 等 Rust 特性 | SHOULD |

## 30. Security Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-SEC-001 | 输入校验 | 所有外部输入（资产文件、DSL 文本、API 请求）须在边界处校验，防止畸形数据导致崩溃或未定义行为 | MUST |
| UAI-NFR-SEC-002 | 密钥与凭据隔离 | LLM/云服务调用密钥须与核心领域数据隔离存储，不写入版本库或日志 | MUST |
| UAI-NFR-SEC-003 | 本地数据默认不外传 | Local-first 模式下，资产几何数据默认不应上传至外部服务，除非用户显式配置云端 LLM/服务 | MUST |

## 31. Observability Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-OBS-001 | 结构化日志 | 关键处理阶段（拆分、检索、推理、验证）须输出结构化日志，可关联到具体资产与假设 ID | MUST |
| UAI-NFR-OBS-002 | 可追溯的推理链 | 每个最终 Rig 结果须能追溯回其证据、假设、验证过程与（如有）人工修正 | MUST |
| UAI-NFR-OBS-003 | 指标采集 | 系统须能采集 §37 KPI 相关的评估指标数据 | SHOULD |

---

## 32. Rust Technical Constraints

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-RUST-001 | Rust 为核心运行时 | 领域逻辑、DSL Parser、Compiler、图处理、API 服务均以 Rust 实现 | MUST |
| UAI-NFR-RUST-002 | 能力方向而非库锁定 | 需求阶段只描述能力要求（异步运行时能力、Web 服务能力、序列化能力、数据并行能力、数学/几何计算能力、图结构能力、GPU 抽象能力、跨语言 FFI 能力），不锁定具体第三方库选型 | MUST |
| UAI-NFR-RUST-003 | Python 边界隔离 | 若引入 Python AI Adapter，须通过进程边界或明确 FFI/IPC 契约与 Rust Core 通信，不得共享内存态领域模型 | MUST |

## 33. Deployment Constraints

已在 §28.3 定义；补充如下：

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-NFR-DEPLOY-004 | 避免过早服务拆分 | 需求阶段不得预先规定具体微服务边界数量或名称；该决策留待基本设计阶段 | MUST |

---

## 34. Data Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-DATA-001 | 资产原始数据存储职责 | 原始 Mesh/点云/序列数据由文件存储或专用几何存储承载，不进入图数据库 | MUST |
| UAI-DATA-002 | 结构化证据的可持久化 | Structural Evidence 须可序列化并持久化，支持复现推理过程 | MUST |
| UAI-DATA-003 | 版本化数据契约 | Instance Graph / Experience Graph / Pattern Graph 的 Schema 须版本化，允许向后兼容演进 | SHOULD |
| UAI-DATA-004 | 数据保留策略 | Experience Graph 中的失败案例须有明确保留策略（不因存储成本被自动清除，除非显式归档） | SHOULD |

## 35. API / Integration Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-API-001 | 资产提交接口 | 系统须提供提交资产（含类型标注）并触发处理流程的接口能力 | MUST |
| UAI-API-002 | 结果查询接口 | 系统须提供查询处理结果（Structural Part Graph、Rig Graph、Confidence 数据）的接口能力 | MUST |
| UAI-API-003 | 人工修正提交接口 | 系统须提供提交人工修正并触发经验记录的接口能力 | MUST |
| UAI-API-004 | CLI 支持 | 系统须提供 CLI 形态以支持批处理与脚本化调用 | MUST（MVP） |
| UAI-API-005 | DCC 集成接口预留 | 需求阶段仅预留未来 DCC 插件对接的接口能力方向，不在 MVP 中实现 | COULD（Research/V1+） |

## 36. Error Handling Requirements

| ID | 标题 | 描述 | 优先级 |
|---|---|---|---|
| UAI-ERR-001 | 分级失败处理 | 系统须区分输入错误、内部处理失败、验证未通过三类结果，并分别给出可操作的反馈 | MUST |
| UAI-ERR-002 | 部分失败可恢复 | 单个 Region/Hypothesis 处理失败不得导致整体资产处理流程崩溃 | MUST |
| UAI-ERR-003 | 失败可追溯 | 所有失败须记录足够上下文（输入摘要、失败阶段、错误原因）用于事后分析 | MUST |

---

## 37. Acceptance Criteria（总体）

系统级验收标准（细则见各需求条目及 §44 追踪矩阵）：

1. AC-01：对至少一个训练库外的未知类别资产，系统能产出非空 Structural Part Graph 与至少一个 Articulation Hypothesis。
2. AC-02：对存在 Topological Bottleneck 但已知非关节的构造测试用例，系统不得将其误判为 Joint（结合其他证据后置信度应显著低于真实关节案例）。
3. AC-03：每个输出的 Joint Candidate 均附带 Confidence、Evidence、Alternative。
4. AC-04：至少 60%（MVP 目标，具体数值随基准调整）的 Rig Hypothesis 能通过基础 Physics Validation（无严重穿透/关节失稳）。
5. AC-05：人工修正可被提交、存储，并可在 Experience Graph 中查询到。
6. AC-06：核心领域逻辑代码（几何/拓扑/图/DSL/Compiler/验证协调）以 Rust 实现，Python 代码（如有）仅存在于隔离的 AI Adapter 层，可通过代码审查验证。
7. AC-07：失败案例在系统中可查询，不因处理失败而被丢弃。
8. AC-08：MVP 全链路（Static Mesh → Geometry/Topology 特征 → Region Graph → 层级拆分 → 检索 → LLM 假设 → Joint Candidate → 简化物理验证）可端到端跑通并产出结果。

## 38. KPI

| KPI | 说明 |
|---|---|
| Structural Decomposition Accuracy | 拆分结果与标注/专家判断的一致程度 |
| Joint Discovery Accuracy | 关节存在性判断的准确率/召回率 |
| Joint Position Error | 关节位置与真值的空间误差 |
| Joint Axis Error | 关节轴向与真值的角度误差 |
| DOF Accuracy | 自由度判断准确率 |
| Skinning Quality | 蒙皮权重质量（如与参考权重的差异度量） |
| Deformation Error | 形变结果与参考形变的误差 |
| Penetration Rate | 验证阶段检测到的穿透发生率 |
| Volume Preservation | 形变过程中体积保持程度 |
| Physics Stability | 物理验证通过率/稳定性评分 |
| Unknown Category Generalization | 对未见类别的处理成功率，衡量泛化能力 |
| Confidence Calibration | 预测置信度与实际正确率的一致性（如 ECE 等度量，具体方法留待设计阶段） |
| Human Correction Time | 核心商业指标：Artist 将 AI 输出修正至生产可用所需时间 |

---

## 39. MVP Scope

**MVP 必须支持的链路：**

Static Mesh → Geometry Feature Extraction → Topology Feature Extraction → Region Graph → Hierarchical Structural Decomposition → Graph / Vector Retrieval（最小子集）→ LLM Structural Hypothesis → Joint Candidate → Simple Physics Validation

**MVP 输入范围**：仅 Static Mesh（单一网格）。

**MVP 输出范围**：Structural Part Graph、Articulation Graph（简化版）、Joint Candidate（含 Confidence/Evidence）、Skeleton（基础 Bone/Hinge/Ball Joint 子集导出）、Validation Report（简化：Penetration/Stretching/Joint Instability 三项）。

**MVP 暂不强制**：

- Muscle / Cloth / Fluid 等高级形变系统
- Dynamic 3DGS 输入
- 全自动动画生成（Full Auto Animation）
- 完整 Retarget 系统
- 完整 DCC 集成
- Pattern/Rule Graph 的自动挖掘（MVP 可用人工/半自动初始化最小规则集，自动挖掘留待 V1）

## 40. Out of Scope（本阶段/文档整体）

- 具体第三方库/框架最终选型
- 数据库物理表结构设计
- 微服务具体拆分与服务间协议细节
- 生产级 UI/UX 设计
- 云端多租户账号体系与计费系统
- 完整 DCC 插件实现

---

## 41. Risk

| ID | 风险 | 影响 | 应对方向 |
|---|---|---|---|
| R-01 | LLM 输出不稳定/幻觉，导致结构假设不可靠 | 高 | 强制物理验证闭环 + 多假设 + 置信度机制；DSL 边界隔离 LLM 与坐标计算 |
| R-02 | Topological Bottleneck 被误用为关节充分条件 | 高 | 明确规则约束（UAI-FR-TOP-006）+ 反例测试集 |
| R-03 | 图数据库退化为存储原始几何数据，导致性能与职责失焦 | 中 | 明确职责边界需求（UAI-FR-GRAPH-004）与设计评审 checkpoint |
| R-04 | Python AI 生态的开发便利性诱导核心逻辑迁移出 Rust | 高 | 架构约束条款（UAI-NFR-GEN-001~003）+ 代码审查门禁 |
| R-05 | MVP 范围蔓延（Scope Creep），一次性追求过多输入/输出类型 | 高 | 严格 MVP 边界（§39）+ 版本化路线图 |
| R-06 | 物理仿真验证成本过高，影响迭代速度 | 中 | 简化验证子集优先（MVP 三项），复杂验证延后至 V1 |
| R-07 | 未知类别泛化能力不足，实际表现接近模板匹配 | 高（研究风险） | 专设 KPI（Unknown Category Generalization）与专门测试集，作为核心创新验证点 |
| R-08 | Structural Grammar 规则集主观性强，缺乏客观校准 | 中 | 规则须关联 Pattern Graph 中的 Confidence/Support/Counterexample 数据驱动演化 |
| R-09 | 人机协作数据未被有效利用，学习闭环形同虚设 | 中 | 明确人工修正须写入 Experience Graph 并有 Pattern Mining 消费路径的验收标准 |

## 42. Open Issues

- OI-01：性能 SLA 的具体数值阈值（网格规模、批处理吞吐、图查询延迟）尚待基准测试确定。
- OI-02：Confidence Calibration 的具体评估方法学尚待研究阶段确定。
- OI-03：Structural Grammar 规则集的初始版本来源（专家编写 vs 数据挖掘）尚待决定。
- OI-04：MVP 阶段 LLM Provider 的选择原则（本地 vs 云端）尚待补充非功能约束细化。
- OI-05：Universal Rig Graph 与主流 DCC 骨骼格式（如 FBX Skeleton）之间转换的信息损失边界尚待研究。

## 43. Assumptions

- A-01：至少存在一个可通过 API 调用的 LLM，具备基本结构化推理能力。
- A-02：至少存在一个可通过 Rust FFI 或独立进程调用的物理仿真能力（自研或第三方）。
- A-03：初期资产规模以中小型生产资产为主，超大规模场景优化留待后续版本。
- A-04：初期用户为具备专业 Rig 知识的 Artist/TD，能够理解并使用置信度与证据信息。

## 44. Future Expansion

- 支持 Point Cloud、3DGS、Dynamic 3DGS、Multi-view Video、Mocap 等更多输入类型（见 §18 分级）。
- Muscle/Tendon/Cable/Soft Structure 等高级 Universal Rig 原语的生产级支持。
- 完整 Retarget Mapping 与跨骨架动作迁移。
- 云端/k3s 部署形态与批量资产处理平台化。
- Pattern/Rule Graph 的全自动持续学习与规则自我演化。
- 完整 DCC（Maya/Blender/Houdini）插件集成。

---

## 附：§18 输入范围分级 与 §19 输出范围分级

### 输入范围分级

| 输入类型 | 分级 |
|---|---|
| Static Mesh | MVP |
| Existing Skeleton（作为参考输入辅助验证） | V1 |
| Existing Animation | V1 |
| Physics Metadata（如密度/材质提示） | V1 |
| Multi Mesh | V1 |
| Point Cloud | V1 |
| Mesh Sequence | V1 |
| Video / Multi-view Video | Research → V1 |
| Mocap | Research → V1 |
| 3DGS（静态） | Research |
| Dynamic 3DGS | Research |

### 输出范围分级

| 输出类型 | 分级 |
|---|---|
| Structural Part Graph | MVP |
| Articulation Graph（简化） | MVP |
| Joint / Joint Candidate | MVP |
| Confidence Data | MVP |
| Validation Report（简化三项） | MVP |
| Skeleton（基础子集） | MVP |
| Joint Axis / DOF / Range of Motion | V1 |
| Skinning Region / Skin Weights（生产级） | V1 |
| Universal Rig Graph（完整原语集） | V1 → Research |
| Physics Constraint（完整） | V1 |
| Retarget Mapping | Research → V1 |

---

## 45. Requirement Traceability Matrix

| 需求 ID | 章节来源 | 关联 Use Case | 关联 KPI | 关联风险 |
|---|---|---|---|---|
| UAI-FR-GEO-001~006 | §15 | UC-01 | Structural Decomposition Accuracy | R-07 |
| UAI-FR-TOP-001~006 | §16 | UC-01 | Joint Discovery Accuracy | R-02 |
| UAI-FR-DEC-001~004 | §17 | UC-01 | Structural Decomposition Accuracy, Unknown Category Generalization | R-05, R-07 |
| UAI-FR-GRAM-001~003 | §17 | UC-01 | Structural Decomposition Accuracy | R-08 |
| UAI-FR-DSL-001~004 | §18 | UC-01, UC-02 | — | R-01 |
| UAI-FR-GRAPH-001~005 | §19 | UC-04, UC-06 | — | R-03, R-09 |
| UAI-FR-VEC-001~004 | §20 | UC-04 | — | R-03 |
| UAI-FR-AI-001~006 | §21 | UC-02 | Confidence Calibration | R-01 |
| UAI-FR-PHY-001~009 | §22 | UC-03 | Penetration Rate, Physics Stability, Volume Preservation | R-06 |
| UAI-FR-RIG-001~006 | §23 | UC-01 | Joint Position Error, Joint Axis Error, DOF Accuracy, Skinning Quality | — |
| UAI-FR-CONF-001~003 | §24 | UC-08 | Confidence Calibration | R-01 |
| UAI-FR-HIL-001~005 | §25 | UC-05 | Human Correction Time | R-09 |
| UAI-FR-LEARN-001~003 | §26 | UC-06 | Human Correction Time | R-09 |
| UAI-NFR-GEN-001~003 | §28.1 | — | — | R-04 |
| UAI-NFR-EXT-001 | §28.2 | UC-09 | — | — |
| UAI-NFR-DEPLOY-001~004 | §28.3, §33 | — | — | — |
| UAI-NFR-PERF-001~005 | §29 | UC-09 | — | — |
| UAI-NFR-SEC-001~003 | §30 | — | — | — |
| UAI-NFR-OBS-001~003 | §31 | UC-08 | Human Correction Time | — |
| UAI-NFR-RUST-001~003 | §32 | — | — | R-04 |
| UAI-DATA-001~004 | §34 | — | — | R-03 |
| UAI-API-001~005 | §35 | UC-09 | — | — |
| UAI-ERR-001~003 | §36 | — | — | — |

---

## 46. 需求成熟度评估（Self-Review）

按 §28（自审要求）执行系统性自审后，评估如下（1～5 分，5 为最佳）：

| 维度 | 评分 | 说明 |
|---|---|---|
| 需求完整度 | 4 | 覆盖 44 个章节要求的全部主题，编号体系完整；但具体性能 SLA 数值、置信度校准方法学等留有 Open Issues，尚未量化 |
| 需求一致性 | 4 | Geometry/Topology 职责分离明确（§15/§16），Graph/Vector 职责边界显式声明（UAI-FR-GRAPH-004、UAI-FR-VEC-004），LLM 角色被限定为结构推理者而非几何引擎；Bottleneck≠Joint 约束在需求与验收标准中重复出现，降低被忽视风险 |
| 技术可行性 | 3 | Rust 核心 + Python AI Adapter 的架构原则清晰可执行；但 Physics-in-the-loop 的仿真成本、图数据库三层知识体系的工程复杂度在 MVP 阶段仍具挑战，需 PoC 验证 |
| 研究风险 | 2 | 核心创新（未知类别泛化、Structural Grammar 的可学习性、多证据融合推理）本质上是未验证的研究假设；Unknown Category Generalization KPI 的达成难度未知，属于文档中风险最高的部分（对应 R-07） |
| MVP 可实现性 | 4 | MVP 范围已收敛至单一输入类型（Static Mesh）+ 简化验证三项 + 简化 Rig 原语子集，范围克制，具备可实现性；但 Structural Grammar 初始规则集来源（OI-03）仍待确定，可能影响启动速度 |
| 核心创新清晰度 | 4 | "Structural Evidence 驱动的多假设推理 + Physics-in-the-loop + 经验学习闭环"这一核心创新链路在文档中反复被结构化表达（§3、§4、§15-26 各节及 §45 追踪矩阵），区别于模板匹配类 Auto-Rig 工具的定位清晰 |
| Rust 实现可行性 | 4 | Rust 生态在图结构、并行计算、Web 服务、序列化等方向均有成熟能力，架构约束未锁定具体库，为设计阶段留出选型空间；主要不确定性在于 GPU/物理仿真与 AI 推理边界的 FFI/IPC 设计，需在基本设计阶段专项验证 |

**主要缺口与后续行动建议：**

1. 需在 PoC 阶段优先验证 R-07（未知类别泛化）与 R-01（LLM 输出可靠性）这两个决定项目研究成败的核心假设，建议作为 PoC 第一批实验目标。
2. 需在基本设计阶段将 OI-01（性能 SLA）、OI-02（置信度校准方法学）、OI-03（Grammar 规则集来源）转化为可执行的设计决策。
3. 建议 PoC 阶段构造"高 Bottleneck 但非关节"的最小反例测试集，作为 UAI-FR-TOP-006 与 AC-02 的可执行验收基础。
4. 建议在基本设计阶段明确 Rust Core 与 Python AI Adapter 之间的进程/内存边界（对应 UAI-NFR-RUST-003），作为防止 R-04 发生的架构守门检查点。

---

*本文档为需求定义阶段产出，禁止作为实现依据直接编码；后续须经基本设计书细化后方可进入 PoC 阶段。*
