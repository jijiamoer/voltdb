# Rosetta 2 编译日志

> 目标：在 ARM64 Mac 上通过 Rosetta 2 编译运行 `engine_test.cpp`

## 环境信息

- **系统**: macOS ARM64 (Apple Silicon)
- **编译模式**: x86_64 via Rosetta 2
- **CMake**: `/tmp/cmake-3.31.2-macos-universal/CMake.app/Contents/bin/cmake`
- **编译器**: `/usr/bin/clang` (Universal Binary)
- **Java**: OpenJDK 24.0.2 (Temurin)
- **分支**: `fix/rosetta-compilation`

## 当前状态

**编译进度**: ~36% (后台命令 ID: 418)

## 已解决的问题

### 1. CMake 版本和策略

| 文件 | 修改 |
|------|------|
| `CMakeLists.txt` | 更新 `cmake_minimum_required(VERSION 3.5)` |
| `src/ee/CMakeLists.txt` | 更新 CMake 版本，移除 CMP0042 |
| `tests/ee/CMakeLists.txt` | 更新 CMake 版本，移除 CMP0042 |
| `third_party/cpp/google-s2-geometry/CMakeLists.txt` | 更新版本，移除 CMP0037 |

### 2. 架构相关编译标志

**文件**: `tools/VoltDBCompilation.cmake`

```cmake
# 条件化 x86 SIMD 标志
IF (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64|i686")
  VOLTDB_ADD_COMPILE_OPTIONS(-mmmx -msse -msse2 -msse3)
ENDIF()
```

### 3. Clang 17+ 警告抑制

**文件**: `tools/VoltDBCompilation.cmake`

```cmake
# Clang 15+ Boost 兼容性
IF ("15.0.0" VERSION_LESS ${CMAKE_CXX_COMPILER_VERSION} )
  VOLTDB_ADD_COMPILE_OPTIONS(-Wno-deprecated-builtins -Wno-enum-constexpr-conversion)
ENDIF()
# VLA 扩展
VOLTDB_ADD_COMPILE_OPTIONS(-Wno-vla-cxx-extension)
```

### 4. CRC32C ARM64 兼容

**文件**: `third_party/cpp/crc/crc32c.cc`

- 用 `#if defined(__x86_64__) || defined(__i386__)` 包装 x86 汇编代码
- 非 x86 平台使用软件实现

### 5. JNI 头文件

**文件**: `src/ee/execution/org_voltdb_jni_ExecutionEngine.h` (新建)

- 创建存根头文件，包含 JNI 类型定义
- **TODO**: 使用 Java 生成真正的 JNI 头文件

## 待解决问题

### 高优先级

1. **生成真正的 JNI 头文件**
   - Java 24 已安装
   - 需要找到 `org.voltdb.jni.ExecutionEngine` 类
   - 使用 `javac -h` 生成头文件

2. **完成编译**
   - 当前进度 ~36%
   - 可能还有其他编译错误

### 中优先级

3. **测试生成**
   - `tests/ee/CMakeLists.txt` 中的测试生成需要 Java
   - 当前跳过了 Java 依赖的测试生成

## 编译命令

```bash
# 配置
arch -x86_64 /tmp/cmake-3.31.2-macos-universal/CMake.app/Contents/bin/cmake \
  -DVOLTDB_BUILD_TYPE=release \
  -DCMAKE_C_COMPILER=/usr/bin/clang \
  -DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  ../..

# 编译
arch -x86_64 make build-engine_test -j4
```

## 更新日志

| 时间 | 事件 |
|------|------|
| 2025-12-03 | 初始化分支 `fix/rosetta-compilation` |
| 2025-12-03 | 解决 CMake 策略警告 |
| 2025-12-03 | 添加 Clang 17 警告抑制 |
| 2025-12-03 | 创建 JNI 存根头文件 |
| 2025-12-03 | 编译进度达到 ~36% |
