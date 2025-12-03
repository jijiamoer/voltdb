# EE White-box Testing TODO

## 0. 背景（Background）
- 角色：执行引擎（Execution Engine, `src/ee`）白盒测试学习与实践
- 个人背景：非计算机专业，正在补习 C++ / Java / 软件测试相关知识
- 环境限制：本机为 arm64 Mac，VoltDB 服务器/EE 只支持 x86_64；计划使用远程 x86_64 环境实际编译和运行 C++ 测试
- 任务来源：主管希望“搭建一套执行引擎白盒测试的框架和流程”，并利用现有 JSON 执行计划设计测试用例

## 1. 总体目标（High-level Goals）
- [ ] G1: 梳理并理解现有 C++ EE 测试框架（`tests/ee`）
- [ ] G2: 围绕领导提供的 JSON 执行计划，设计并落地首批白盒测试用例
- [ ] G3: 固化一套在 x86_64 环境中构建与运行 EE 测试的标准流程
- [ ] G4: 扩展覆盖范围，并整理成可供团队复用的文档

## 2. 阶段 C：基于 JSON 执行计划的测试（Plan-driven Tests）
- [ ] C1: 收集并规范存放所有 JSON 执行计划样本（路径、命名约定）
- [ ] C2: 分析每个 JSON 计划的大致语义（涉及的表 / 索引 / 算子）
- [ ] C3: 选定 1 个代表性 JSON 计划作为“首个白盒测试对象”
- [ ] C4: 设计对应的测试场景：
  - 表结构（是否复用现有测试表，例如 customer 表）
  - 初始化数据（随机 / 手工构造）
  - 需要验证的结果 / 不变式（例如行数、聚合值、索引命中情况）
- [ ] C5: 在 `tests/ee` 中选择 / 创建合适的 C++ 测试文件，用于计划驱动的 EE 测试
- [ ] C6: 实现第一个基于 JSON 计划的 `TEST_F` 测试用例
- [ ] C7: 在 x86_64 环境上编译并运行该测试，记录命令与结果

## 3. 阶段 B：引擎 API 级测试（Engine-level API Tests, UniqueEngine / VoltDBEngine）
- [ ] B1: 阅读 `tests/ee/test_utils/UniqueEngine.hpp`，理解如何构造测试用 VoltDBEngine
- [ ] B2: 列出 `VoltDBEngine` 若干对外 API（例如 `initialize`, `loadCatalog`, `getSnapshotSchema` 等）
- [ ] B3: 选择 1–2 个 API 设计白盒测试：输入 / 输出 / 内部不变式
- [ ] B4: 编写对应的 C++ 单元 / 组件级测试用例
- [ ] B5: 在环境就绪后运行测试，分析结果并更新文档

## 4. 环境与运行流程（Environment & Execution Flow）
- [ ] E1: 确认公司提供的 x86_64 Linux / 虚拟机环境（主机名、访问方式）
- [ ] E2: 在目标环境中完成一次 VoltDB C++ EE 构建（CMake + make）
- [ ] E3: 记录构建和运行 `tests/ee` 的完整命令序列
- [ ] E4: 尝试运行现有 `ExecutionEngineTest` 等测试，验证环境正常

## 5. 文档与知识沉淀（Documentation & Knowledge Sharing）
- [ ] D1: 创建并维护 `PROGRESS.md`，按时间记录进展
- [ ] D2: 记录遇到的主要问题、踩坑与解决方案
- [ ] D3: 将关键结论同步到 Redmine / Wiki，方便主管和同事查看
