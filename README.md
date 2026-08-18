# Rigin

## Universal Articulation Intelligence（UAI）

Rigin 是 **Universal Articulation Intelligence** 项目的代码与文档仓库。该系统的目标不是又一个模板匹配式的 Auto-Rigging 工具，而是让机器理解任意三维对象由哪些结构组成、这些结构如何连接、哪些部分能够发生相对运动、为什么能够如此运动，并最终将这种理解转换为可执行的 Rig。

核心问题不是"骨头应该放在哪里"，而是：

> 一个三维结构应该如何拆分、如何连接、如何运动？

系统综合 Geometry、Topology、Motion、Semantics、Physics 与历史结构知识，推导出结构证据、结构拆分、关节假设、Universal Rig 表示，并经物理验证闭环形成可自我修正的 Rig 结果；Skeleton 只是最终输出形式之一。

系统核心运行时以 **Rust** 为主，仅在 AI 推理/训练适配层等边界场景按需引入 Python（作为隔离适配层，不承载核心领域逻辑）。

## 项目文档

本项目遵循「需求定义 → 基本设计 → 详细设计 → PoC / 实装」的阶段化流程，文档结构参照日本 IPA（情報処理推進機構）共通フレーム2013（SLCP-JCF2013）惯例编制，每份文档均含版本管理、改订履历、承认体系与需求/设计追踪矩阵。

| 阶段 | 文档 | 说明 |
|---|---|---|
| 需求定义 | [《Universal Articulation Intelligence 需求定义书》](docs/UAI-Requirements-Definition.md) | 定义需求边界、用户场景、功能/非功能需求、数据/接口需求、验收标准、风险与约束，并附需求成熟度自审 |
| 基本设计 | [《Universal Articulation Intelligence 基本设计书》](docs/UAI-Basic-Design.md) | 定义系统构成、模块划分与职责、数据概念模型、外部接口契约、核心处理流程与非功能设计方针 |
| 详细设计 | [《Universal Articulation Intelligence 详细设计书》](docs/UAI-Detailed-Design.md) | 定义模块内部构造、字段级数据结构、字段级接口契约、处理时序、算法设计方针、错误码体系，并包含面向常规 3D 软件的通用 DCC / RPA 式集成层设计 |

四份正式设计文档之间以唯一编号（`UAI-FR-*` / `UAI-BD-*` / `UAI-DD-*` 等）互相追踪，详见各文档末尾的追踪矩阵章节。

### PoC 配套文档

三份需求/设计文档确定了系统的完整目标，但直接实现全部 MVP 范围成本过高。以下三份文档定义了一个**范围更小、可最快跑通的"最小闭环"**：核心 Rust 领域逻辑为真实实现，LLM/Physics/Vector/Graph 四类外部依赖全部替换为可配置的 Mock，用于尽早验证架构假设（尤其是 Provider 抽象契约与"Bottleneck ≠ Joint"等核心设计约束），再逐步替换为真实实现。

| 文档 | 说明 |
|---|---|
| [《PoC 最小闭环开发计划书》](docs/UAI-PoC-Development-Plan.md) | 定义最小闭环范围、工作分解结构（WBS）、里程碑、退出准则 |
| [《部署方案书》](docs/UAI-Deployment-Plan.md) | 定义 PoC/MVP 阶段的运行环境、进程拓扑、配置管理、构建发布与运维方针 |
| [《UAI Mock 测试项目设计书》](docs/UAI-Mock-Test-Project-Design.md) | 定义验证最小闭环所需的 Mock Provider 与测试资产集的需求/基本设计/详细设计 |

## 文档阅读顺序

1. 先读《需求定义书》了解系统要解决的问题、边界与验收标准。
2. 再读《基本设计书》了解系统如何在模块层面组织以满足这些需求。
3. 再读《详细设计书》了解每个模块的内部构造、数据结构与处理细节，以及通用 DCC 集成层的设计方向。
4. 最后读 PoC 配套文档（开发计划 → 部署方案 → Mock 测试项目设计），了解如何以最小成本验证上述设计。

## 项目状态

当前处于文档化的需求/设计阶段，尚未进入 PoC 与代码实现阶段。已制定最小闭环的 PoC 开发计划、部署方案与配套 Mock 测试项目设计，后续版本将据此展开实装与验证。
