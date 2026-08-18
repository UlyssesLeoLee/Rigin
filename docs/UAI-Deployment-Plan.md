# Universal Articulation Intelligence — 部署方案书

版本: v0.1 (Draft)
文档状态: PoC/MVP 部署设计阶段 — 定义运行环境、进程拓扑、配置管理、构建发布流程、监控与回滚方针
作者: AI Research / Architecture (assisted)
日期: 2026-08-18
文档标准: 参照 IPA 共通フレーム2013 之「導入・運用プロセス」惯例编制，聚焦本阶段（PoC/MVP）实际需要的部署决策，不预先设计 V1+ 的云端/k3s 拓扑细节

上游依据文档：`UAI-REQ-001` v0.3（§28.3/§33 部署约束）、`UAI-BD-001` v0.3（§4.3/§9.3 部署设计方针）、`UAI-DD-001` v0.3、`UAI-PoC-Development-Plan.md`（本次同批产出，定义最小闭环范围）。

---

## 0. 文档管理信息

### 0.1 改订履历

| 版本 | 日期 | 变更内容 | 作成者 |
|---|---|---|---|
| v0.1 | 2026-08-18 | 初版发布 | AI Research / Architecture |

### 0.2 参照文档一览

| 文档 ID | 文档名 |
|---|---|
| UAI-REQ-001 | 需求定义书 v0.3 |
| UAI-BD-001 | 基本设计书 v0.3 |
| UAI-DD-001 | 详细设计书 v0.3 |
| UAI-POC-001 | PoC 最小闭环开发计划书 v0.1 |
| UAI-MOCK-001 | Mock 测试项目设计书 v0.1（本次同批产出） |

---

## 1. 部署范围与分级

沿用需求 §28.3 的部署单元角色划分（Library/Process/Worker/Service/GPU Worker）与 §33 的"避免过早服务拆分"约束。本文档只定义 **PoC/MVP 阶段（Local-first）** 的部署方案；V1+ 云端/k3s 拓扑不在本文档范围内，仅在 §7 做方向性预留说明。

| 阶段 | 部署形态 | 状态 |
|---|---|---|
| PoC（最小闭环） | 单机单进程 + Mock Provider（进程内） | 本文档主要范围 |
| MVP（需求 §39） | 单机单进程 + 可切换真实/Mock Provider（本地或远程调用） | 本文档次要范围，标注差异点 |
| V1+ | 部分模块拆分为独立 Service/GPU Worker，可选 k3s | 仅方向性说明（§7），不展开设计 |

---

## 2. 运行环境需求（UAI-DEPLOY-ENV-*）

| 编号 | 项目 | 要求 |
|---|---|---|
| UAI-DEPLOY-ENV-001 | 操作系统 | 主流 Linux 发行版（x86_64/aarch64）为首要目标；macOS/Windows 作为开发环境支持，非本阶段部署验证目标 |
| UAI-DEPLOY-ENV-002 | Rust 工具链 | 稳定版 Rust（具体最低版本留待实装时依据所选 crate 依赖确定，不在本文档锁定） |
| UAI-DEPLOY-ENV-003 | GPU（可选） | PoC 阶段不要求 GPU（Mock Physics/几何计算走 CPU 路径）；预留 GPU 抽象接口以便 V1+ 启用 |
| UAI-DEPLOY-ENV-004 | 存储 | 本地文件系统用于原始 Mesh/资产存储；Graph/Vector Store 的 PoC 阶段实现方式见 §3.3 |
| UAI-DEPLOY-ENV-005 | 网络 | PoC 阶段默认不要求出网；仅当配置切换为真实云端 LLM Provider 时才需要出网权限（对应需求 UAI-NFR-SEC-003） |

---

## 3. 进程拓扑设计（UAI-DEPLOY-TOPO-*）

### 3.1 PoC（最小闭环）拓扑

```
┌─────────────────────────────────────────────┐
│           uai-cli （单一本地进程）              │
│                                               │
│  Ingestion → Geometry → Topology → Decomp    │
│      → DSL Layer → Rig Compiler              │
│      → Retrieval(Mock) → LLM Adapter(Mock)   │
│      → Physics(Mock) → Confidence            │
│                                               │
│  本地文件：                                    │
│   - 资产文件（输入 Mesh）                       │
│   - ProcessingRun / Hypothesis 等结构化输出     │
│     （PoC 阶段落盘为本地文件，见 §3.3）           │
└─────────────────────────────────────────────┘
```

设计要点（UAI-DEPLOY-TOPO-001）：PoC 阶段不引入网络服务、不引入独立 Graph/Vector Store 进程，所有 Provider（含 Mock）均以进程内 trait/接口调用形式存在，避免过早引入进程间通信复杂度（呼应需求 UAI-NFR-DEPLOY-004）。

### 3.2 MVP 拓扑（在 PoC 基础上的差异点）

```
┌─────────────────────────────────────────────┐
│         uai-service （本地长驻进程/CLI 均可）    │
│  模块 1-4, 6, 8, 10, 11, 13：进程内             │
│                                               │
│  模块 5 Retrieval  ──────▶ 可配置：             │
│                             本地嵌入式 Vector/   │
│                             Graph 存储，或        │
│                             远程 Provider         │
│  模块 7 LLM Adapter ─────▶ 可配置：本地/远程 LLM  │
│  模块 9 Physics     ─────▶ 可配置：本地/远程物理   │
│                             引擎                  │
│  模块 12 Learning Loop ──▶ 独立异步 Worker 进程    │
│                            （可选，MVP 阶段可禁用） │
└─────────────────────────────────────────────┘
```

差异点（UAI-DEPLOY-TOPO-002）：MVP 阶段 Provider 从"全 Mock"变为"可配置 Mock/真实"，通过 §5 配置管理统一切换，核心模块代码不感知差异（对应 PoC 计划书 EXIT-008 的架构验证目标）。

### 3.3 数据存储的 PoC 简化方案

| 数据类别 | PoC 阶段实现方式 | 依据 |
|---|---|---|
| 原始 Mesh/资产 | 本地文件系统目录 | UAI-DD-DATA §3.1 |
| Instance Graph（Region/RelationEdge/Hypothesis 等） | 进程内内存结构 + 运行结束后序列化为本地 JSON/文件（不引入真实图数据库） | 简化自 UAI-BD-001 §6，符合"图数据库不存原始几何"的边界（本阶段图数据库本身也用文件模拟） |
| Experience/Pattern Graph | PoC 阶段不落地（模块 011/012 不在最小闭环范围内，见 PoC 计划书 §2.2） | — |

> 明确声明：本阶段"用本地文件模拟图存储"是刻意的范围收窄，不是最终架构决策；真实图数据库/向量数据库的选型与拓扑设计留待 V1 部署方案书。

---

## 4. 构建与发布流程（UAI-DEPLOY-BUILD-*）

| 编号 | 项目 | 方针 |
|---|---|---|
| UAI-DEPLOY-BUILD-001 | 构建产物 | 单一可执行文件（CLI），静态或最小动态依赖，便于本地分发 |
| UAI-DEPLOY-BUILD-002 | CI 构建门禁 | 每次提交须通过：编译检查、单元测试（对应 UAI-BD-TEST-001）、集成测试（对应 PoC 计划书 §5 退出准则对应的自动化用例） |
| UAI-DEPLOY-BUILD-003 | 版本标识 | 可执行文件须内嵌版本号与构建时间，便于 `ProcessingRun` 记录中标注产出版本（呼应 UAI-NFR-OBS-002 可追溯性） |
| UAI-DEPLOY-BUILD-004 | 发布物范围 | PoC 阶段不建立正式发布/分发渠道，构建产物仅供内部验证使用 |

---

## 5. 配置管理（UAI-DEPLOY-CONF-*）

### 5.1 Provider 切换配置

落实 UAI-BD-001 §7.5 与 UAI-DD-001 §8.5 的 Provider 抽象契约，配置须能独立控制四类 Provider 的实现选择：

```
[providers]
llm      = "mock" | "<future: real provider identifier>"
physics  = "mock" | "<future: real provider identifier>"
vector   = "mock" | "<future: real provider identifier>"
graph    = "mock" | "<future: real provider identifier>"
```

设计约束（UAI-DEPLOY-CONF-001）：切换配置项不得要求重新编译核心模块代码——Provider 选择是运行时/构建时的组装决策，不是核心逻辑的分支逻辑（对应 UAI-NFR-EXT-001）。

### 5.2 密钥与敏感配置

| 编号 | 方针 |
|---|---|
| UAI-DEPLOY-CONF-002 | 真实 Provider 的密钥/凭据通过独立配置源（环境变量或本地密钥文件，不提交版本库）注入，落实 UAI-BD-NFR-008 / UAI-DD-NFR-004 |
| UAI-DEPLOY-CONF-003 | PoC 阶段（全 Mock）不涉及任何真实密钥，`.env`/密钥文件示例应以占位符形式提供，不包含真实凭据 |

---

## 6. 监控与运维方针（UAI-DEPLOY-OPS-*，PoC 阶段简化版）

| 编号 | 项目 | 方针 |
|---|---|---|
| UAI-DEPLOY-OPS-001 | 日志 | 结构化日志输出到本地文件/标准输出，落实 UAI-NFR-OBS-001；PoC 阶段不要求集中式日志收集 |
| UAI-DEPLOY-OPS-002 | 健康检查 | PoC 阶段为 CLI 形态，无需常驻健康检查端点；MVP 阶段若转为长驻进程，须提供最小健康检查接口 |
| UAI-DEPLOY-OPS-003 | 回滚方针 | PoC/MVP 阶段回滚等价于"切换回上一构建版本的可执行文件"，不涉及数据库迁移回滚（本阶段无真实数据库） |
| UAI-DEPLOY-OPS-004 | 故障响应 | 参照详细设计书 §5.2 降级路径与 §9 错误码体系（`UAI-ERRC-*`）；运维人员通过 `ProcessingRun.degraded_mode_flags` 与错误码定位问题类别 |

---

## 7. V1+ 方向性预留（不展开设计，仅标注）

- 模块 5（Retrieval）、7（LLM Adapter）、9（Physics）可拆分为独立 Service，经网络接口调用，具体协议（REST/gRPC）留待届时选型（呼应需求 UAI-NFR-DEPLOY-004，避免本阶段过早决定）。
- 引入真实 Graph/Vector 数据库产品后，§3.3 的本地文件模拟方案将被替换，需另行制定数据迁移方案（对应基本设计书 §13 移行方针的实际触发时机）。
- 云端/k3s 部署（需求 UAI-NFR-DEPLOY-002）的拓扑、弹性伸缩、多租户等设计留待专门的 V1 部署方案书，不在本文档预先设计。

---

## 8. 自审

| 检查项 | 状态 | 说明 |
|---|---|---|
| 是否遵循 Local-first 优先原则 | ✅ | §1/§3.1 明确 PoC 为单机单进程 |
| 是否过早设计云端/k3s 拓扑（违反需求 UAI-NFR-DEPLOY-004） | ✅ 未过早设计 | §7 仅方向性标注，不展开 |
| Provider 切换是否需要改核心代码 | ✅ 不需要 | §5.1 UAI-DEPLOY-CONF-001 明确约束 |
| 是否涉及真实凭据 | ✅ 未涉及 | §5.2 PoC 阶段全 Mock，无真实密钥 |
| 是否与 PoC 开发计划书范围一致 | ✅ | §1/§3.1 与 `UAI-POC-001` §2 范围对齐 |

---

*本文档聚焦 PoC/MVP 阶段的可执行部署决策；V1+ 云端拓扑设计留待后续独立文档。*
