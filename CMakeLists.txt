cmake_minimum_required(VERSION 3.10)

# 项目信息
project(srep VERSION 3.93 LANGUAGES C CXX)

# ------------------------------------------------------------------------------
# 1. 语言标准设置
# ------------------------------------------------------------------------------
# 强制使用 C++98 标准，确保旧代码的兼容性 (避免 auto 等新关键字产生冲突)
set(CMAKE_CXX_STANDARD 98)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 默认构建类型设为 Release (开启 -O3 优化)，除非用户指定了 Debug
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to Release as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()

# ------------------------------------------------------------------------------
# 2. 源代码与依赖
# ------------------------------------------------------------------------------
# 注意：原 Makefile 采用 Unity Build 模式（srep.cpp include 了其他 .cpp），
# 我们保持这一结构以确保链接正确。
set(SOURCES
    Compression/Common.cpp
    Compression/SREP/srep.cpp
)

# 包含头文件路径
include_directories(
    Compression
    Compression/_Encryption
    Compression/_Encryption/headers
    Compression/_Encryption/hashes
)

# ------------------------------------------------------------------------------
# 3. 平台与架构检测
# ------------------------------------------------------------------------------
# 操作系统宏定义
if(WIN32)
    add_definitions(-DFREEARC_WIN)
elseif(UNIX)
    add_definitions(-DFREEARC_UNIX -D_FILE_OFFSET_BITS=64)
endif()

# 字节序检测 (Byte Order)
include(TestBigEndian)
TEST_BIG_ENDIAN(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
    message(WARNING "Big Endian system detected. SREP assumes Little Endian logic. Forcing MOTOROLA_BYTE_ORDER macro.")
    add_definitions(-DFREEARC_MOTOROLA_BYTE_ORDER)
else()
    add_definitions(-DFREEARC_INTEL_BYTE_ORDER)
endif()

# ------------------------------------------------------------------------------
# 4. 编译器标志与兼容性
# ------------------------------------------------------------------------------
include(CheckCXXCompilerFlag)

# SSE2 支持检测 (兼容老旧 x86 CPU)
check_cxx_compiler_flag("-msse2" COMPILER_SUPPORTS_SSE2)
if(COMPILER_SUPPORTS_SSE2)
    add_compile_options(-msse2)
endif()

# 循环展开优化 (原 Makefile 特性)
check_cxx_compiler_flag("-funroll-all-loops" COMPILER_SUPPORTS_UNROLL)
if(COMPILER_SUPPORTS_UNROLL)
    add_compile_options(-funroll-all-loops)
endif()

# 屏蔽干扰视线的旧代码警告 (保持构建日志清洁)
check_cxx_compiler_flag("-Wno-write-strings" W_NO_WRITE_STRINGS)
if(W_NO_WRITE_STRINGS)
    add_compile_options(-Wno-write-strings)
endif()

check_cxx_compiler_flag("-Wno-unused-result" W_NO_UNUSED_RESULT)
if(W_NO_UNUSED_RESULT)
    add_compile_options(-Wno-unused-result)
endif()

# ------------------------------------------------------------------------------
# 5. 构建目标
# ------------------------------------------------------------------------------
add_executable(srep ${SOURCES})

# 链接线程库 (Linux/Unix 需要)
if(UNIX)
    set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
    find_package(Threads REQUIRED)
    target_link_libraries(srep Threads::Threads)
endif()

# 静态链接选项 (默认关闭，使用 cmake -DSREP_STATIC_LINK=ON .. 开启)
option(SREP_STATIC_LINK "Link statically for better portability" OFF)
if(SREP_STATIC_LINK)
    if(UNIX AND NOT APPLE)
        target_link_libraries(srep -static)
    endif()
endif()

# ------------------------------------------------------------------------------
# 6. 安装规则
# ------------------------------------------------------------------------------
install(TARGETS srep DESTINATION bin)
