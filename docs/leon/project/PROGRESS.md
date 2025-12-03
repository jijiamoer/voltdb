# EE White-box Testing Progress

## 0. 背景快照（Snapshot of Context）
- 个人背景：非计算机专业，在职学习 C++ / Java / 软件测试
- 项目范围：VoltDB Execution Engine（`src/ee`）白盒测试框架与用例
- 当前阶段：阶段 C —— 围绕 JSON 执行计划的 plan-driven 白盒测试
- 环境：开发机为 arm64 Mac，计划使用远程 x86_64 环境运行 C++ 测试

## 1. 里程碑（Milestones）
- [ ] M1: 读完 `tests/ee` 主要测试文件与工具类，并完成一份框架说明
- [ ] M2: 对领导提供的 JSON 执行计划完成结构分析与测试设计
- [ ] M3: 在 `tests/ee` 中落地第一个基于 JSON 计划的白盒测试用例
- [ ] M4: 在 x86_64 环境成功运行该测试并通过
- [ ] M5: 完成首轮 Engine-level API 测试（`UniqueEngine` / `VoltDBEngine`）并沉淀文档

## 2. 周进展（Weekly Log）

### 2025-12-??（Week ?）
- **Done**
  - 初始化 TODO/PROGRESS 文档结构，明确任务背景与总体目标
- **In Progress**
  - 梳理 `tests/ee` 目录结构与核心测试基类（ExecutionEngineTest, PlanTestingBaseClass, UniqueEngine 等）
  - 收集并整理领导提供的 JSON 执行计划
- **Next**
  - 选定首个 JSON 计划，设计对应的白盒测试场景（输入 / 输出 / 不变式）

> 后续周进展可以按类似格式追加，保持「Done / In Progress / Next」三栏结构。

## 3. 关键决策与开放问题（Decisions & Open Questions）

### 3.1 已做出的关键决策（Decisions）
- 决定采用「先 C 后 B」的学习与实现路线：
  - 先基于 JSON 执行计划设计 plan-driven 白盒测试；
  - 再深入 `UniqueEngine` / `VoltDBEngine` 级别的 API 测试。

### 3.2 当前开放问题（Open Questions）
- OQ1: 远程 x86_64 环境的最终方案（公司统一开发机 vs 自建 VM）尚待确认。
- OQ2: 领导提供的 JSON 计划是否会持续新增，是否需要一套自动生成 / 管理测试计划的机制。
- OQ3: 针对哪些执行引擎功能（存储、索引、执行器、快照等）优先补充白盒测试，需要与主管进一步对齐优先级。
