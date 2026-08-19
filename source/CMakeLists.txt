cmake_minimum_required(VERSION 3.20)

project(rpc_protos LANGUAGES CXX)

# ==================== vcpkg 集成 ====================
# 如果通过 -DCMAKE_TOOLCHAIN_FILE 指定了 vcpkg，则自动使用
if(NOT DEFINED CMAKE_TOOLCHAIN_FILE AND EXISTS "C:/vcpkg/scripts/buildsystems/vcpkg.cmake")
    message(STATUS "vcpkg toolchain available at C:/vcpkg/scripts/buildsystems/vcpkg.cmake")
endif()

# 查找 protobuf（通过 vcpkg 提供）
find_package(protobuf CONFIG REQUIRED)

# ==================== 自动收集 .proto 文件 ====================
# 递归扫描 source 目录下所有 .proto 文件
file(GLOB_RECURSE PROTO_FILES "${CMAKE_CURRENT_SOURCE_DIR}/*.proto")

if(NOT PROTO_FILES)
    message(FATAL_ERROR "No .proto files found in ${CMAKE_CURRENT_SOURCE_DIR}")
endif()

message(STATUS "Found proto files:")
foreach(PROTO_FILE ${PROTO_FILES})
    message(STATUS "  ${PROTO_FILE}")
endforeach()

# protobuf 生成的代码输出目录（build/generated）
set(PROTO_GEN_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")

# ==================== 创建静态库目标 ====================
add_library(rpc_protos STATIC ${PROTO_FILES})

# 自动调用 protoc 生成 .pb.h/.pb.cc，并附加到目标
protobuf_generate(
    TARGET rpc_protos
    LANGUAGE cpp
    PROTOC_OUT_DIR "${PROTO_GEN_DIR}"
    IMPORT_DIRS "${CMAKE_CURRENT_SOURCE_DIR}"
    PROTOS ${PROTO_FILES}
)

# 包含生成的头文件目录
target_include_directories(rpc_protos PUBLIC
    "${PROTO_GEN_DIR}"
    "${CMAKE_CURRENT_SOURCE_DIR}"
)

# 链接 protobuf 库
target_link_libraries(rpc_protos PUBLIC
    protobuf::libprotobuf
)

# C++17 标准
target_compile_features(rpc_protos PUBLIC cxx_std_17)