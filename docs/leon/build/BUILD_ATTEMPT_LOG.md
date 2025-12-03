# VoltDB ARM64 构建尝试记录

## 目标
使用 Python 脚本构建 VoltDB 项目，生成 `voltdbipcrun` 可执行文件到 `target/compiled/prod` 或 `obj/release/ee/prod/`

## 构建环境
- **系统**: macOS (ARM64/Apple Silicon)
- **编译器**: AppleClang 17.0.0.17000404
- **CMake**: 版本 4.1 (通过 uv/Python)
- **构建类型**: Release
- **第三方库路径**: `/Volumes/leon01_ssd/MyFiles/200_work/2025_mywork/2025_10/shuangzhao/project/my_volt/voltdb/third_party`

## 发现的关键信息

### 1. 构建流程依赖关系
- VoltDB 构建分为两部分：Java 部分（使用 Ant）和 C++ 部分（使用 CMake）
- C++ 部分依赖 Java 生成的 catalog 文件
- catalog 文件生成脚本位于 `src/catgen/catalog.py` 和 `src/catgen/install.py`
- 必须在 CMake 配置前运行 catalog 生成脚本

### 2. 构建系统文件结构
- 主 CMakeLists.txt: `/voltdb/CMakeLists.txt`
- EE CMakeLists.txt: `/voltdb/src/ee/CMakeLists.txt`
- 测试 CMakeLists.txt: `/voltdb/tests/ee/CMakeLists.txt`
- 编译配置: `/voltdb/tools/VoltDBCompilation.cmake`
- 第三方库 S2GEO: `/voltdb/third_party/cpp/google-s2-geometry/CMakeLists.txt`

### 3. 第三方依赖库
- OpenSSL 1.0.2t
- PCRE2 10.33
- Google S2 Geometry
- Boost (位于 third_party/cpp/boost)
- CRC32C (位于 third_party/cpp/crc)
- TTMath (位于 third_party/cpp/ttmath)

## 遇到的错误序列

### 错误 1: Python 包构建失败
**错误信息**:
```
ValueError: Unable to determine which files to ship inside the wheel
```
**发生位置**: `pyproject.toml` 配置
**发现**: hatchling 构建系统需要明确指定要包含的文件

### 错误 2: Ant 工具缺失
**错误信息**:
```
Apache Ant 未找到
```
**发现**: 系统未安装 Ant，但 C++ 部分可以独立构建

### 错误 3: CMake Policy CMP0042 不兼容
**错误信息**:
```
Policy CMP0042 may not be set to OLD behavior because this version of CMake no longer supports it
```
**发生位置**: 
- `/voltdb/CMakeLists.txt` 第 191-197 行
- `/voltdb/src/ee/CMakeLists.txt` 第 76-80 行
- `/voltdb/tests/ee/CMakeLists.txt` 第 44-48 行
**发现**: 旧版 CMake policy 设置为 OLD，但新版 CMake 不再支持

### 错误 4: Java 测试文件缺失
**错误信息**:
```
错误: 找不到或无法加载主类 org.voltdb.planner.eegentests.GenerateEETests
FILE failed to open for reading ... generated_tests.txt
```
**发现**: CMake 尝试构建测试，但测试依赖 Java 生成的文件

### 错误 5: CMake 最低版本过旧
**错误信息**:
```
Compatibility with CMake < 3.5 has been removed from CMake
```
**发生位置**: 
- `/voltdb/CMakeLists.txt` 第 188 行: `CMAKE_MINIMUM_REQUIRED (VERSION 2.8.11 FATAL_ERROR)`
- `/voltdb/src/ee/CMakeLists.txt` 第 51 行
- `/voltdb/tests/ee/CMakeLists.txt` 第 36 行
- `/voltdb/third_party/cpp/google-s2-geometry/CMakeLists.txt` 第 1 行: `cmake_minimum_required(VERSION 2.8)`

### 错误 6: x86 SSE 指令在 ARM64 上不支持
**错误信息**:
```
clang++: error: unsupported option '-mmmx' for target 'arm64-apple-darwin25.0.0'
clang++: error: unsupported option '-msse' for target 'arm64-apple-darwin25.0.0'
clang++: error: unsupported option '-msse2' for target 'arm64-apple-darwin25.0.0'
clang++: error: unsupported option '-msse3' for target 'arm64-apple-darwin25.0.0'
```
**发生位置**: `/voltdb/tools/VoltDBCompilation.cmake` 第 56 行
**发现**: Release 构建类型硬编码了 x86 特定的 SIMD 指令集选项

### 错误 7: CRC32C 使用 x86 汇编指令
**错误信息**:
```
error: invalid output constraint '=a' in asm
error: value size does not match register size specified by the constraint and modifier
```
**发生位置**: `/voltdb/third_party/cpp/crc/crc32c.cc`
**涉及函数**:
- `cpuid()` 第 29 行: 使用 x86 `cpuid` 指令
- `_mm_crc32_u64()` 第 105 行: 使用 `crc32q` 指令
- `_mm_crc32_u32()` 第 110 行: 使用 `crc32l` 指令
- `_mm_crc32_u16()` 第 115 行: 使用 `crc32w` 指令
- `_mm_crc32_u8()` 第 120 行: 使用 `crc32b` 指令
- `crc32cHardware64()` 第 127 行: 硬件加速 CRC32 函数

### 错误 8: S2GEO CMake Policy CMP0037 问题
**错误信息**:
```
Policy CMP0037 may not be set to OLD behavior
```
**发生位置**: `/voltdb/third_party/cpp/google-s2-geometry/CMakeLists.txt` 第 92 行

### 错误 9: S2GEO CMAKE_BUILD_TYPE 变量为空
**错误信息**:
```
STRING no output variable specified
if given arguments: "NOT" "(" "STREQUAL" "debug" ")" ...
```
**发生位置**: `/voltdb/CMakeLists.txt` 第 213 行
**发现**: S2GEO 作为 ExternalProject 构建时，接收到的 `CMAKE_BUILD_TYPE` 值不正确（小写 "release" 而非首字母大写 "Release"）

### 错误 10: Boost 库使用废弃的编译器内建函数
**错误信息**:
```
error: builtin __has_trivial_copy is deprecated; use __is_trivially_copyable instead
error: builtin __has_trivial_destructor is deprecated; use __is_trivially_destructible instead
error: builtin __has_nothrow_assign is deprecated; use __is_nothrow_assignable instead
```
**发生位置**: 
- `/voltdb/third_party/cpp/boost/type_traits/has_trivial_copy.hpp` 第 34 行
- `/voltdb/third_party/cpp/boost/type_traits/has_trivial_destructor.hpp` 第 30 行
- `/voltdb/third_party/cpp/boost/type_traits/has_nothrow_assign.hpp` 第 65 行
**发现**: Boost 库版本较旧，使用了已被 Clang 17 标记为废弃的内建函数

### 错误 11: TTMath 库使用 x86 汇编指令
**错误信息**:
```
error: value size does not match register size specified by the constraint and modifier [-Werror,-Wasm-operand-widths]
error: invalid output constraint '=a' in asm
```
**发生位置**: `/voltdb/third_party/cpp/ttmath/ttmathuint_x86.h`
**涉及行号**:
- 第 1291 行: `movl` 指令约束问题
- 第 1292 行: `bsrl` 指令约束问题
- 第 1293 行: `cmovz` 指令约束问题
- 第 1349 行: 无效的输出约束 `=a`
- 第 1406 行: 无效的输出约束 `=a`
- 第 1470 行: 无效的输出约束 `=a`
**发现**: TTMath 库包含大量 x86 特定的内联汇编代码

## 尝试的构建步骤

### 尝试 1: 创建 Python 构建脚本
- 创建了 `build_voltdb.py` 脚本
- 创建了 `pyproject.toml` 配置文件
- 使用 `uv run` 执行 Python 脚本

### 尝试 2: 生成 Catalog 文件
- 在 CMake 配置前运行 `python3 catalog.py`
- 在 CMake 配置前运行 `python3 install.py`
- 成功生成了 catalog 相关的 C++ 文件到 `src/ee/catalog/` 目录

### 尝试 3: 禁用测试构建
- 在根 CMakeLists.txt 添加 `VOLTDB_BUILD_TESTS` 选项
- 条件编译测试相关代码块
- 在 CMake 命令行传递 `-DVOLTDB_BUILD_TESTS=OFF`

### 尝试 4: 更新 CMake 最低版本要求
- 将所有 CMakeLists.txt 中的最低版本从 2.8.x 更新到 3.10

### 尝试 5: 修复 CMake Policy
- 将 CMP0042 从 OLD 改为 NEW（3 个文件）
- 将 S2GEO 的 CMP0037 从 OLD 改为 NEW

### 尝试 6: 架构特定编译选项
- 在 VoltDBCompilation.cmake 中添加架构检测
- 对 ARM64 架构移除 x86 SSE 指令选项（`-mmmx -msse -msse2 -msse3`）

### 尝试 7: CRC32C 架构条件编译
- 为 x86 特定代码添加 `#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)` 预处理器条件
- 为 ARM64 提供软件实现的回退路径（使用 `crc32cSlicingBy8`）

### 尝试 8: 修复 S2GEO 构建配置
- 在根 CMakeLists.txt 中为 S2GEO ExternalProject 同时传递 `CMAKE_BUILD_TYPE` 和 `VOLTDB_BUILD_TYPE`

### 尝试 9: 添加编译器警告抑制
- 添加 `-Wno-deprecated-builtins` 以忽略 Boost 废弃内建函数警告
- 添加 `-Wno-asm-operand-widths` 以忽略汇编操作数宽度警告

## 构建进度观察

### 成功编译的部分
- Catalog 代码生成：✅ 成功
- CMake 配置阶段：✅ 成功
- OpenSSL (crypto)：✅ 成功构建
- PCRE2：✅ 成功构建
- third_party_objs (CRC32C)：✅ 成功编译
- S2GEO 配置：✅ 成功

### 失败的部分
- voltdbobjs 目标：❌ 编译失败
- 具体失败的源文件包括：
  - `common/LargeTempTableBlockCache.cpp`
  - `common/LoadTableCaller.cpp`
  - `common/NValue.cpp`
  - `catalog/catalog.cpp`
  - `catalog/catalogtype.cpp`
  - `catalog/cluster.cpp`

## 关键发现

1. **架构不兼容**: VoltDB 代码库针对 x86_64 架构优化，包含大量 x86 特定的汇编代码
2. **第三方库问题**: 多个第三方库（CRC32C、TTMath）直接使用 x86 汇编
3. **编译器版本**: Clang 17 对废弃特性的检查更严格，与旧版 Boost 不兼容
4. **构建系统复杂性**: 构建过程涉及 Python、Ant、CMake 多个工具链
5. **测试依赖**: 测试构建依赖 Java 编译输出，需要完整的 Java 工具链

## 最后的构建状态
- 构建命令: `make -j10 voltdbipc`
- 退出码: 2
- 主要错误: TTMath 库中的 x86 汇编代码在 ARM64 上无法编译
- 错误类型: `-Werror,-Wasm-operand-widths` 和 `invalid output constraint`
