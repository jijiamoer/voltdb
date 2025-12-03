# VoltDB EE 白盒测试学习 - AI 提问模板

这份文档整理了针对 VoltDB C++ 执行引擎关键文件的“AI 提问提示词”。
你可以直接复制这些问题，发送给 AI 助手，让它帮你快速解读代码核心逻辑。

## 阶段 1：测试框架基础 (tests/ee)

### 1. 文件：`tests/ee/test_utils/plan_testing_baseclass.h`
**推荐提问：**
> 请帮我解读 `tests/ee/test_utils/plan_testing_baseclass.h` 这个文件：
> 1. `PlanTestingBaseClass` 这个模板类主要负责管理哪些核心组件（例如 m_engine, m_topend, buffers）？
> 2. 它的 `initialize(catalog_string, random_seed)` 函数里具体做了哪些初始化工作？
> 3. 它如何配合 `EngineTestTopend` 来模拟执行计划的下发？
> 4. 继承这个基类对写测试有什么好处？

### 2. 文件：`tests/ee/test_utils/testing_topend.h` (或对应实现文件)
**推荐提问：**
> 请解读 `tests/ee/test_utils/testing_topend.h`：
> 1. `EngineTestTopend` 作为一个 Mock/Fake 对象，它内部是如何存储 plan fragment 的？
> 2. `addPlan` 方法的逻辑是什么？
> 3. 当执行引擎调用它获取 plan 时，它做了什么简化处理？

### 3. 文件：`tests/ee/test_utils/UniqueEngine.hpp`
**推荐提问：**
> 请解读 `tests/ee/test_utils/UniqueEngine.hpp`：
> 1. `UniqueEngine` 和 `UniqueEngineBuilder` 的设计目的是什么？
> 2. `build()` 函数是如何构造并初始化 `VoltDBEngine` 实例的？
> 3. 相比于 `PlanTestingBaseClass`，什么时候更适合用 `UniqueEngine` 来写测试？

---

## 阶段 2：执行引擎核心 (src/ee/execution)

### 4. 文件：`src/ee/execution/VoltDBEngine.h`
**推荐提问：**
> 请阅读 `src/ee/execution/VoltDBEngine.h`：
> 1. 请用一句话概括 `VoltDBEngine` 类在整个系统中的职责。
> 2. 请列出它最重要的 3-5 个 Public API（如 initialize, executePlanFragments, loadCatalog），并简述功能。
> 3. 它持有哪些关键的成员变量（如 catalog, tables, context），分别起什么作用？

### 5. 文件：`src/ee/execution/VoltDBEngine.cpp`
**推荐提问：**
> 请结合 `src/ee/execution/VoltDBEngine.cpp` 的实现（重点看 initialize 和 executePlanFragments）：
> 1. `initialize` 函数在启动时构建了哪些关键子系统（如 Hashinator, Executors）？
> 2. `executePlanFragments` 是如何接收 JSON plan 并调度执行的？它的参数列表里包含了哪些运行时上下文？

---

## 阶段 3：执行器与存储 (Executors & Storage)

### 6. 文件：`src/ee/executors/abstractexecutor.h`
**推荐提问：**
> 请解读 `src/ee/executors/abstractexecutor.h`：
> 1. 所有 Executor 的基类定义了哪些标准生命周期接口（如 p_execute, p_init）？
> 2. Executor 是如何通过 `ExecutorContext` 与底层数据表交互的？

### 7. 文件：`src/ee/executors/indexscanexecutor.cpp` (或 seqscanexecutor.cpp)
**推荐提问：**
> 请以 `indexscanexecutor.cpp` 为例分析：
> 1. 当它收到一个 Plan Node 时，是如何解析出查询条件的？
> 2. 它具体调用了哪个底层接口来遍历索引或数据表？
> 3. 在 `p_execute` 中，它是如何生成输出 tuple 的？

### 8. 文件：`src/ee/storage/persistenttable.h` 与 `src/ee/indexes/tableindex.h`
**推荐提问：**
> 请解读 `persistenttable.h` 和 `tableindex.h`：
> 1. `PersistentTable` 类管理了哪些核心数据结构（如行数据块、索引集合）？
> 2. `TableIndex` 定义了哪些索引操作接口（insert, delete, lookup）？
> 3. 这里的索引是否有唯一性（Unique）和非唯一性的区分？在接口上如何体现？
