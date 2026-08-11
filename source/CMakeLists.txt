cmake_minimum_required(VERSION 3.14)

project(rpc_protos LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 查找 Protobuf 库和 protoc 编译器
find_package(Protobuf REQUIRED)

# proto 仓库根目录（proto/）
get_filename_component(PROTO_ROOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/.." ABSOLUTE)

# 输出目录统一放在 proto/build 下
set(GENERATED_DIR "${PROTO_ROOT_DIR}/build/generated")
set(LIB_DIR "${PROTO_ROOT_DIR}/build/lib")
set(BIN_DIR "${PROTO_ROOT_DIR}/build/bin")

# 创建输出目录
file(MAKE_DIRECTORY "${GENERATED_DIR}" "${LIB_DIR}" "${BIN_DIR}")

# 编译产物输出路径
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${LIB_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${LIB_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${BIN_DIR}")

# Proto 源文件根目录（本文件所在目录）
set(PROTO_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")

# 递归收集 proto/source 下所有子目录中的 .proto 文件
file(GLOB_RECURSE PROTO_FILES
    CONFIGURE_DEPENDS
    "${PROTO_SOURCE_DIR}/*.proto"
)

if(NOT PROTO_FILES)
    message(FATAL_ERROR "No .proto files found under ${PROTO_SOURCE_DIR}")
endif()

message(STATUS "Found proto files:")
foreach(PROTO_FILE ${PROTO_FILES})
    message(STATUS "  ${PROTO_FILE}")
endforeach()

# 选择 protoc 可执行程序
if(TARGET Protobuf::protoc)
    set(PROTOC_EXECUTABLE Protobuf::protoc)
else()
    set(PROTOC_EXECUTABLE "${Protobuf_PROTOC_EXECUTABLE}")
endif()


# 设置导入路径，支持 proto 之间的 import
set(PROTO_IMPORT_DIRS "${PROTO_SOURCE_DIR}")

# 逐个调用 protoc 生成 C++ 代码到 proto/build/generated
set(ALL_PROTO_SRCS "")
set(ALL_PROTO_HDRS "")

foreach(PROTO_FILE ${PROTO_FILES})
    # 计算 proto 文件相对于 PROTO_SOURCE_DIR 的相对路径，如 mysql/MySQL.proto
    file(RELATIVE_PATH PROTO_REL_PATH "${PROTO_SOURCE_DIR}" "${PROTO_FILE}")

    # 去掉 .proto 后缀得到 basename，如 mysql/MySQL
    string(REGEX REPLACE "\\.proto$" "" PROTO_BASENAME "${PROTO_REL_PATH}")

    set(PROTO_HDR "${GENERATED_DIR}/${PROTO_BASENAME}.pb.h")
    set(PROTO_SRC "${GENERATED_DIR}/${PROTO_BASENAME}.pb.cc")

    add_custom_command(
        OUTPUT "${PROTO_HDR}" "${PROTO_SRC}"
        COMMAND "${PROTOC_EXECUTABLE}"
        ARGS -I${PROTO_SOURCE_DIR} --cpp_out=${GENERATED_DIR} ${PROTO_FILE}
        DEPENDS "${PROTO_FILE}"
        COMMENT "Generating C++ code for ${PROTO_FILE}"
        VERBATIM
    )

    list(APPEND ALL_PROTO_SRCS "${PROTO_SRC}")
    list(APPEND ALL_PROTO_HDRS "${PROTO_HDR}")
endforeach()

# 将所有生成的代码编译为静态库
add_library(${PROJECT_NAME} STATIC ${ALL_PROTO_SRCS} ${ALL_PROTO_HDRS})

target_include_directories(${PROJECT_NAME} PUBLIC
    "${GENERATED_DIR}"
    "${PROTO_SOURCE_DIR}"
    ${Protobuf_INCLUDE_DIRS}
)

target_link_libraries(${PROJECT_NAME} PUBLIC protobuf::libprotobuf)

# 安装生成的头文件和库文件
install(TARGETS ${PROJECT_NAME} ARCHIVE DESTINATION "${LIB_DIR}")
install(FILES ${ALL_PROTO_HDRS} DESTINATION "${GENERATED_DIR}")