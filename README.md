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

三份文档之间以唯一编号（`UAI-FR-*` / `UAI-BD-*` / `UAI-DD-*` 等）互相追踪，详见各文档末尾的追踪矩阵章节。

## 文档阅读顺序

1. 先读《需求定义书》了解系统要解决的问题、边界与验收标准。
2. 再读《基本设计书》了解系统如何在模块层面组织以满足这些需求。
3. 最后读《详细设计书》了解每个模块的内部构造、数据结构与处理细节，以及通用 DCC 集成层的设计方向。

## 项目状态

当前处于文档化的需求/设计阶段，尚未进入 PoC 与代码实现阶段。后续版本将围绕 MVP 范围（详见需求定义书 §39）展开 PoC 验证。
