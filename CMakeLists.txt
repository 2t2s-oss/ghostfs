cmake_minimum_required(VERSION 2.8.3)

project(GhostFS)

# 定义 CMake 模块路径
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/modules/")

# 使用 ccache（可选）
find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
    message(STATUS "Using ccache")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif(CCACHE_FOUND)

# 修正：一次性指定 Boost 组件
find_package(fuse REQUIRED)
find_package(CURL REQUIRED)
find_package(Boost REQUIRED COMPONENTS system filesystem)  # 合并调用
find_package(PythonLibs 2.7 REQUIRED)

# 修正：明确指定库路径和名称
set(
    GHOST_LIBRARIES
    ${FUSE_LIBRARIES}          # FUSE 库（已包含正确路径）
    ${CURL_LIBRARIES}          # CURL 库
    ${PYTHON_LIBRARIES}        # Python 2.7 库
    ${Boost_SYSTEM_LIBRARY}    # Boost system 组件
    ${Boost_FILESYSTEM_LIBRARY} # Boost filesystem 组件
    pthread                   # 显式链接 pthread 库
    dl                        # 保留动态链接库（如需要）
)

# 编译选项（添加 -pthread 确保线程支持）
set(CMAKE_CXX_FLAGS         "${CMAKE_CXX_FLAGS} --std=c++11 -Wall -pthread -D_FILE_OFFSET_BITS=64")
set(CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_DEBUG} -fno-inline -ggdb -DDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_RELEASE} -O2 -DNDEBUG")

# 安装路径配置
set(INSTALL_BIN_DIR /usr/bin)
set(INSTALL_LIB_DIR /usr/lib)

# 包含头文件路径
include_directories(
    ${FUSE_INCLUDE_DIRS}
    ${CURL_INCLUDE_DIRS}
    ${PYTHON_INCLUDE_DIR}
    ${CMAKE_SOURCE_DIR}
)

# 构建静态库和可执行文件
set(BUILD_SHARED_LIBS FALSE)
add_library(ghostfs_lib
    ghost_file.cc
    block_info.cc
    cache.cc
    ghost_fs.cc
    utils.cc

    protocol/base_protocol.cc
    protocol/http_protocol.cc
    protocol/load_drivers.cc
    protocol/python_driver.cc
)

add_executable(ghostfs
    main.cc
)

# 修正：调整链接顺序（先链接静态库，再链接其他库）
target_link_libraries(
    ghostfs
    ghostfs_lib           # 先链接静态库，确保符号优先解析
    ${GHOST_LIBRARIES}    # 再链接其他依赖库
)

# 安装配置（保留原有内容）
install(
    TARGETS ghostfs
    DESTINATION "${INSTALL_BIN_DIR}"
    COMPONENT application
)

install(
    TARGETS ghostfs_lib
    DESTINATION "${INSTALL_LIB_DIR}"
)

install(
    FILES ${CMAKE_CURRENT_SOURCE_DIR}/gmount
    DESTINATION "${INSTALL_BIN_DIR}"
    PERMISSIONS
        OWNER_READ OWNER_WRITE OWNER_EXECUTE
        GROUP_READ GROUP_WRITE GROUP_EXECUTE
        WORLD_READ WORLD_WRITE WORLD_EXECUTE
)

# 打包配置（保留原有内容）
set(ALLOW_BUILD_PACKAGE ON CACHE BOOL "checkbox")
if (ALLOW_BUILD_PACKAGE)
    # ... 原有打包配置 ...
endif (ALLOW_BUILD_PACKAGE)
