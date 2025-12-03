# VoltDB EE Rosetta 2 编译完整指南

> 在 ARM64 Mac (Apple Silicon) 上通过 Rosetta 2 编译 VoltDB 执行引擎

## 1. 概述

### 目标
在 ARM64 Mac 上编译并运行 VoltDB EE 的 `engine_test` 测试程序。

### 最终结果
```
ExecutionEngineTest:
        IndexOrder: PASSED.
        Execute_PlanFragmentInfo: PASSED.
PASSED
```

### 技术路线
使用 Rosetta 2 将编译和运行环境模拟为 x86_64，避免修改 VoltDB 核心代码中的 x86 汇编（如 TTMath）。

---

## 2. 环境准备

### 2.1 系统要求
- macOS ARM64 (Apple Silicon)
- Xcode Command Line Tools（含 Clang 17+）
- Rosetta 2（首次运行 x86_64 程序时自动安装）

### 2.2 获取 Universal Binary CMake

**方法 A：pip 安装**（推荐）
```bash
# 创建虚拟环境
python3 -m venv .venv
source .venv/bin/activate
pip install cmake

# CMake 路径
.venv/lib/python3.13/site-packages/cmake/data/bin/cmake
# 验证
file .venv/lib/python3.13/site-packages/cmake/data/bin/cmake
# 输出: Mach-O universal binary with 2 architectures: [x86_64] [arm64]
```

**方法 B：手动下载**
```bash
# 从 cmake.org 下载 macOS Universal 版本
curl -LO https://github.com/Kitware/CMake/releases/download/v3.31.2/cmake-3.31.2-macos-universal.tar.gz
tar xf cmake-3.31.2-macos-universal.tar.gz
# 路径: cmake-3.31.2-macos-universal/CMake.app/Contents/bin/cmake
```

### 2.3 验证 Rosetta 2
```bash
arch -x86_64 /usr/bin/true && echo "Rosetta 2 OK"
```

---

## 3. 代码修改清单

### 3.1 CMake 现代化

| 文件 | 修改 |
|------|------|
| `CMakeLists.txt` | `cmake_minimum_required(VERSION 3.5)` |
| `src/ee/CMakeLists.txt` | `cmake_minimum_required(VERSION 3.5)` |
| `tests/ee/CMakeLists.txt` | `cmake_minimum_required(VERSION 3.5)` |
| `third_party/cpp/google-s2-geometry/CMakeLists.txt` | `cmake_minimum_required(VERSION 3.5)` |

### 3.2 移除过时 CMake 策略

**所有 CMP0042 相关代码替换为：**
```cmake
# macOS rpath handling - 使用新版 CMake 的默认行为
IF (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    SET(CMAKE_MACOSX_RPATH ON)
ENDIF()
```

**移除 CMP0037（s2geo）：**
```cmake
# 删除这行
# cmake_policy(SET CMP0037 OLD)
# 添加注释
# CMP0037 已过时，新版 CMake 默认允许下划线目标名
```

### 3.3 Clang 17 警告抑制

**文件：** `tools/VoltDBCompilation.cmake`

```cmake
# Clang 15+ 标记旧版 Boost 库的内建函数为废弃；在 Rosetta 的 Clang 17 上还会触发 unqualified-std-cast-call
IF ("15.0.0" VERSION_LESS ${CMAKE_CXX_COMPILER_VERSION} )
  VOLTDB_ADD_COMPILE_OPTIONS(-Wno-deprecated-builtins -Wno-enum-constexpr-conversion -Wno-unqualified-std-cast-call)
ENDIF()
# VLA 是 Clang 的 C++ 扩展，VoltDB 代码中有使用
VOLTDB_ADD_COMPILE_OPTIONS(-Wno-vla-cxx-extension)
```

### 3.4 条件化 x86 SIMD 标志

**文件：** `tools/VoltDBCompilation.cmake`

```cmake
ELSEIF (VOLTDB_BUILD_TYPE STREQUAL "RELEASE")
  # x86 SIMD 标志仅在非 ARM 架构上使用
  IF (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64|i686")
    VOLTDB_ADD_COMPILE_OPTIONS(-O3 -g3 -mmmx -msse -msse2 -msse3 -DNDEBUG)
  ELSE()
    VOLTDB_ADD_COMPILE_OPTIONS(-O3 -g3 -DNDEBUG)
  ENDIF()
```

### 3.5 CRC32C ARM64 条件编译

**文件：** `third_party/cpp/crc/crc32c.cc`

```cpp
// 前向声明软件实现
uint32_t crc32cSlicingBy8(uint32_t crc, const void* data, size_t length);

#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
// x86 平台：支持硬件加速
static uint32_t crc32c_CPUDetection(...) { ... }
CRC32CFunctionPtr crc32c = crc32c_CPUDetection;
CRC32CFunctionPtr detectBestCRC32C() { ... }
#else
// 非 x86 平台（ARM64 等）：只使用软件实现
CRC32CFunctionPtr crc32c = crc32cSlicingBy8;
CRC32CFunctionPtr detectBestCRC32C() {
    return crc32cSlicingBy8;
}
#endif
```

### 3.6 JNI 存根头文件

**文件：** `src/ee/execution/org_voltdb_jni_ExecutionEngine.h`（新建）

```cpp
/* 
 * 存根文件 - 用于在没有 Java 编译的情况下编译 C++ EE
 */
#ifndef _Included_org_voltdb_jni_ExecutionEngine
#define _Included_org_voltdb_jni_ExecutionEngine

#include <stdint.h>

/* JNI 类型定义存根 */
#ifndef _JAVASOFT_JNI_H_
typedef int32_t jint;
typedef int64_t jlong;
typedef int8_t jbyte;
typedef uint8_t jboolean;
typedef uint16_t jchar;
typedef int16_t jshort;
typedef float jfloat;
typedef double jdouble;
typedef jint jsize;
#endif

#endif
```

### 3.7 TableTupleAllocator 常量转换修复

**文件：** `src/ee/storage/TableTupleAllocator.hpp:1074`

```cpp
// 修改前
static constexpr char const MASK = 1 << NthBit;
// 修改后
static constexpr unsigned char MASK = 1u << NthBit;
```

### 3.8 测试生成优雅降级

**文件：** `tests/ee/CMakeLists.txt`

添加 Java 测试生成失败时的处理逻辑，允许仅编译手动测试。

---

## 4. 错误与解决方案映射表

| 错误信息 | 触发文件 | 解决方案 |
|----------|----------|----------|
| `-mmmx: unsupported option` | VoltDBCompilation.cmake | 条件化 SIMD 标志 |
| `-Wdeprecated-builtins` | Boost headers | `-Wno-deprecated-builtins` |
| `-Wvla-cxx-extension` | VoltDB 源码 | `-Wno-vla-cxx-extension` |
| `-Wenum-constexpr-conversion` | Boost headers | `-Wno-enum-constexpr-conversion` |
| `-Wunqualified-std-cast-call` | VoltDB 源码 | `-Wno-unqualified-std-cast-call` |
| `-Wconstant-conversion` (MASK) | TableTupleAllocator.hpp | `unsigned char` + `1u` |
| `CMP0042 is deprecated` | 多个 CMakeLists | 移除，使用 `CMAKE_MACOSX_RPATH` |
| `CMP0037 is deprecated` | s2geo CMakeLists | 移除策略设置 |
| `org_voltdb_jni_ExecutionEngine.h not found` | VoltDBEngine.cpp | 创建存根头文件 |
| `undeclared identifier 'jint'` | VoltDBEngine.cpp | 在存根中添加 typedef |
| `crc32q instruction` on ARM | crc32c.cc | `#if defined(__x86_64__)` |

---

## 5. 编译步骤

### 5.1 清理（如需重建）
```bash
cd obj/release
rm -rf CMakeCache.txt CMakeFiles
```

### 5.2 配置
```bash
cd obj/release

# 设置 CMake 路径（根据获取方式调整）
CMAKE_PATH="../../.venv/lib/python3.13/site-packages/cmake/data/bin/cmake"

# 配置
arch -x86_64 $CMAKE_PATH \
  -DVOLTDB_BUILD_TYPE=release \
  -DCMAKE_C_COMPILER=/usr/bin/clang \
  -DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  ../..
```

### 5.3 编译
```bash
arch -x86_64 make build-engine_test -j4
```

### 5.4 运行测试
```bash
arch -x86_64 ./cpptests/execution/engine_test
```

---

## 6. 常见问题 FAQ

### Q: Homebrew CMake 可以用吗？
A: 不行。`/opt/homebrew/bin/cmake` 是 ARM64 原生二进制，无法在 `arch -x86_64` 下运行。

### Q: 为什么不直接原生编译？
A: VoltDB 依赖 TTMath 等库，包含大量 x86 汇编代码，原生 ARM64 编译需要大量移植工作。

### Q: 编译后的二进制可以在其他 ARM64 Mac 上运行吗？
A: 可以，只要目标机器安装了 Rosetta 2。

### Q: 这些修改可以提交到上游吗？
A: 部分可以（CMake 现代化、警告抑制），但 JNI 存根不适合上游。

---

## 7. Git 分支信息

### 分支名
`fix/rosetta-compilation`

### Commit 历史
```
098f1bbf20 fix: TableTupleAllocator 常量转换警告
aeaa630446 fix: Rosetta 2 编译兼容性修复
943b134496 docs: 添加学习笔记和进度文档
```

### 文件变更统计
```
14 files changed, 1123 insertions(+), 71 deletions(-)
```

---

## 8. 参考资料

- VoltDB GitHub: https://github.com/VoltDB/voltdb
- Rosetta 2 文档: https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment
- CMake 下载: https://cmake.org/download/

---

*最后更新: 2024-12-03*
*作者: Cascade + Codex*
