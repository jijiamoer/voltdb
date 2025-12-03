# VoltDB EE 白盒测试素材生成链路解析

本文档记录了如何从 Java Planner 测试生成 C++ 执行引擎白盒测试所需的 Catalog 和 Plan JSON。
基于对 `tests/frontend/org/voltdb/planner/TestPlansENG10022.java` 和 `engine_test.cpp` 的分析。

## 1. 核心流程概览

整个白盒测试的数据流向是一个标准的“产销模型”：
- **生产者 (Java)**: 输入 SQL，产出 Catalog 和 JSON Plan。
- **消费者 (C++)**: 消费 Catalog 和 JSON Plan，在 EE 中执行并验证。

```mermaid
graph LR
    SQL[SQL文件\n(testplans-eng10022.sql)] -->|setupSchema| Java[Java Planner测试\n(TestPlansENG10022.java)]
    Java -->|编译| Catalog[Catalog String\n(机器指令单)]
    Java -->|compile(SELECT...)| JSON[JSON Execution Plan\n(执行计划)]
    Catalog -->|粘贴/注入| CPP[C++ 白盒测试\n(engine_test.cpp)]
    JSON -->|粘贴/注入| CPP
```

## 2. 详细步骤拆解

### Step 1: 定义数据结构 (SQL)
- **源文件**: `testplans-eng10022.sql`
- **内容**: `CREATE TABLE`, `CREATE INDEX` 等语句。
- **作用**: 定义测试所需的表结构。

### Step 2: 生成 Catalog (Java -> Catalog String)
- **相关代码**: `TestPlansENG10022.java` 中的 `setUp()`
  ```java
  setupSchema(getClass().getResource("testplans-eng10022.sql"), "ENG-10022", false);
  ```
- **原理**: 调用 VoltCompiler 将 SQL 编译为 Catalog。
- **产物**:
  - **方法 A (推荐)**: 直接在 Java 测试中调用 `getCatalogString()` 获取 C++ 所需的 catalog 字符串。
  - **方法 B (备选)**: 编译生成的 JAR 包内的 `catalog.txt` 文件内容。

### Step 3: 生成执行计划 (Java -> JSON Plan)
- **相关代码**: `TestPlansENG10022.java` 中的测试方法
  ```java
  compile("select r_customerid as cid, 2*r_customerid as cid2 from r_customer;");
  ```
- **调试模式开启**:
  - **方式 (系统属性)**: 使用 `-Dorg.voltdb.compilerdebug=true` 运行测试。
  - *注：PlannerTestCase 不使用 ProjectBuilder，因此代码中设置 PrintStream 的方式无效。*
- **原理**: Planner 对具体 SQL 进行解析和优化，生成物理执行计划。
- **产物**: 开启 debug 模式后，JSON 文件会生成在 `$TEST_DIR/debugoutput/statement-plans/` 目录下（例如 `debugoutput/statement-plans/ENG-10022-stmt-0_json.txt`）。

### Step 4: 组装 C++ 测试 (C++ Engine Test)
- **相关代码**: `engine_test.cpp`
- **操作**:
  1. 将 Step 2 的 Catalog String 赋值给 `catalog_string` 变量。
  2. 将 Step 3 的 JSON 内容赋值给 `plan` 变量。
  3. **关键步骤**: 调用 `initialize(catalog_string)` 启动引擎。
  4. **关键步骤**: 调用 `m_topend->addPlan(100, plan)` 将 JSON 注册到 Topend（Fragment ID 设为 100）。
  5. 调用 `m_engine->executePlanFragments(..., &fragmentId, ...)` 执行（传入 ID 100）。
  6. 使用 `ASSERT_TRUE` 验证结果。

## 3. 如何复用于新任务？

如果你接到了一个新的白盒测试任务（例如新的 JSON 计划），标准操作路径如下：

1. **准备 SQL**: 拿到新计划对应的建表语句（SQL 文件）。
2. **新建 Java 测试**: 仿照 `TestPlansENG10022.java` 写一个新的 Planner 测试类，继承 `PlannerTestCase`。
3. **运行生成**:
   - 在 Java 测试中调用 `getCatalogString()` 打印出 catalog 字符串。
   - 开启 debug 模式 (`-Dorg.voltdb.compilerdebug=true`)，运行测试后在 `debugoutput/statement-plans/` 目录获取 JSON Plan。
4. **编写 C++ 测试**:
   - 在 `tests/ee` 下新建测试文件。
   - 定义测试类继承 `PlanTestingBaseClass<EngineTestTopend>`。
   - 填入 Catalog 和 Plan。
   - 记得调用 `addPlan(id, plan)` 注册计划。
   - 编写断言逻辑验证执行结果。
