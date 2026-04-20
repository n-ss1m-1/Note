好的！下面我系统地总结 CMake 的用法，包括基础语法、核心命令、最佳实践和实用示例。

## 📚 一、CMake 基础语法

### 📌 注释

cmake

```
# 行注释

#[[
块注释
可以跨越多行
]]
```

### 📌 变量引用

cmake

```
${VAR_NAME}  # 引用变量
```

### 📌 命令语法

cmake

```
COMMAND(参数1 参数2 参数3)  # 参数间用空格或分号分隔
```

## 📚 二、CMake 基本结构

### 📌 最小 CMakeLists.txt

cmake

```
cmake_minimum_required(VERSION 3.10)  # CMake最低版本要求
project(MyProject)                    # 项目名称

# 添加可执行文件
add_executable(myapp main.cpp)
```

### 📌 完整构建流程

bash

运行

```
# 1. 创建构建目录（推荐）
mkdir build && cd build

# 2. 生成Makefile
cmake ..  # .. 表示CMakeLists.txt在上一级目录

# 3. 编译
make

# 4. 运行
./myapp
```

## 📚 三、核心命令详解

### 📌 1. 项目配置

#### cmake_minimum_required()

cmake

```
cmake_minimum_required(VERSION 3.10)
```

指定 CMake 最低版本要求

推荐使用 3.10 或更高版本

#### project()

cmake

```
project(MyProject)                          # 基本用法
project(MyProject VERSION 1.0.0)            # 指定版本
project(MyProject LANGUAGES CXX)            # 指定语言
project(MyProject VERSION 1.0.0 LANGUAGES CXX C)  # 完整用法
```

#### set()

cmake

```
# 定义变量
set(VAR_NAME value)
set(SRC_LIST main.cpp util.cpp helper.cpp)

# 设置C++标准
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)  # 要求必须支持该标准
set(CMAKE_CXX_EXTENSIONS OFF)        # 禁用编译器扩展

# 设置输出路径
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lib)
```

### 📌 2. 可执行文件和库

#### add_executable()

cmake

```
# 基本用法
add_executable(myapp main.cpp util.cpp)

# 使用变量
set(SRC_LIST main.cpp util.cpp)
add_executable(myapp ${SRC_LIST})

# 多个源文件可以用空格或分号分隔
add_executable(myapp main.cpp;util.cpp;helper.cpp)
```

#### add_library()

cmake

```
# 静态库
add_library(mylib STATIC src1.cpp src2.cpp)

# 动态库
add_library(mylib SHARED src1.cpp src2.cpp)

# 接口库（仅头文件）
add_library(mylib INTERFACE)

# 对象库
add_library(mylib OBJECT src1.cpp src2.cpp)
```

### 📌 3. 源文件管理

#### aux_source_directory()

cmake

```
# 搜索指定目录下的所有源文件
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src SRC_LIST)
add_executable(myapp ${SRC_LIST})
```

#### file(GLOB/GLOB_RECURSE)

cmake

```
# 搜索指定模式的文件
file(GLOB SRC_FILES "*.cpp" "*.cc")
file(GLOB HEADER_FILES "include/*.h")

# 递归搜索子目录
file(GLOB_RECURSE ALL_SOURCES "src/*.cpp" "src/*.cc")

add_executable(myapp ${ALL_SOURCES})
```

#### list () 操作

cmake

```
# 追加元素
list(APPEND SRC_LIST file1.cpp file2.cpp)

# 移除元素
list(REMOVE_ITEM SRC_LIST file1.cpp)

# 移除重复元素
list(REMOVE_DUPLICATES SRC_LIST)

# 反转列表
list(REVERSE SRC_LIST)

# 获取列表长度
list(LENGTH SRC_LIST LEN)
message("Length: ${LEN}")

# 获取指定位置的元素
list(GET SRC_LIST 0 FIRST_FILE)
```

### 📌 4. 头文件包含

#### include_directories()

cmake

```
# 添加头文件搜索路径
include_directories(include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(/usr/local/include)

# 可以一次添加多个路径
include_directories(
    include
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    /usr/local/include
)
```

#### target_include_directories () (推荐)

cmake

```
# 为特定目标添加头文件路径
add_executable(myapp main.cpp)
target_include_directories(myapp PRIVATE include)

# PUBLIC: 传递给依赖此目标的其他目标
# PRIVATE: 仅此目标使用
# INTERFACE: 仅用于接口（头文件库）
target_include_directories(myapp
    PUBLIC include/public
    PRIVATE include/private
)
```

### 📌 5. 库的链接

#### link_directories()

cmake

```
# 添加库文件搜索路径
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/lib)
link_directories(/usr/local/lib)
```

#### link_libraries()

cmake

```
# 链接库（全局）
link_libraries(pthread dl)
```

#### target_link_libraries () (推荐)

cmake

```
# 基本用法
add_executable(myapp main.cpp)
target_link_libraries(myapp mylib pthread)

# 指定链接类型
target_link_libraries(myapp
    PRIVATE mylib      # 私有依赖
    PUBLIC pthread     # 公共依赖
    INTERFACE dl       # 接口依赖
)

# 链接系统库
target_link_libraries(myapp
    PRIVATE
        pthread
        dl
        m
)

# 链接第三方库
target_link_libraries(myapp
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/libmylib.a
)
```

### 📌 6. 编译选项

#### target_compile_options()

cmake

```
add_executable(myapp main.cpp)

# 添加编译选项
target_compile_options(myapp PRIVATE
    -Wall           # 所有警告
    -Wextra         # 额外警告
    -Werror         # 警告视为错误
    -O2             # 优化级别
    -g              # 调试信息
)

# 条件编译选项
target_compile_options(myapp PRIVATE
    $<$<CONFIG:Debug>:-O0 -g>      # Debug配置
    $<$<CONFIG:Release>:-O3>       # Release配置
)
```

#### target_compile_definitions()

cmake

```
add_executable(myapp main.cpp)

# 添加预处理器定义
target_compile_definitions(myapp PRIVATE
    DEBUG_MODE
    MAX_SIZE=1024
    VERSION="1.0.0"
)

# 条件定义
target_compile_definitions(myapp PRIVATE
    $<$<CONFIG:Debug>:DEBUG>
    $<$<CONFIG:Release>:NDEBUG>
)
```

### 📌 7. 嵌套 CMake

#### add_subdirectory()

cmake

```
# 主CMakeLists.txt
cmake_minimum_required(VERSION 3.10)
project(MyProject)

# 添加子目录
add_subdirectory(src)      # src目录包含CMakeLists.txt
add_subdirectory(lib)      # lib目录包含CMakeLists.txt
add_subdirectory(tests)    # tests目录包含CMakeLists.txt
```

子目录 CMakeLists.txt 示例

cmake

```
# src/CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

# 包含头文件
include_directories(${PROJECT_SOURCE_DIR}/include)

# 搜寻源文件
file(GLOB SRC_FILES "*.cpp")

# 创建库
add_library(mylib STATIC ${SRC_FILES})

# 设置输出路径
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
```

## 📚 四、CMake 常用变量

### 📌 路径相关变量

cmake

```
${CMAKE_CURRENT_SOURCE_DIR}  # 当前CMakeLists.txt所在目录
${CMAKE_CURRENT_BINARY_DIR}  # 当前构建输出目录
${PROJECT_SOURCE_DIR}        # 项目根目录（最外层CMakeLists.txt）
${PROJECT_BINARY_DIR}        # 项目构建输出根目录
${CMAKE_SOURCE_DIR}          # CMakeLists.txt根目录
${CMAKE_BINARY_DIR}          # 构建输出根目录
```

### 📌 配置相关变量

cmake

```
${CMAKE_BUILD_TYPE}          # 构建类型：Debug/Release/RelWithDebInfo/MinSizeRel
${CMAKE_CXX_COMPILER}        # C++编译器路径
${CMAKE_C_COMPILER}          # C编译器路径
${CMAKE_SYSTEM_NAME}         # 系统名称
```

## 📚 五、实用示例

### 💡 示例 1：简单项目

cmake

```
cmake_minimum_required(VERSION 3.10)
project(SimpleApp VERSION 1.0.0)

# 设置C++标准
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 搜寻源文件
file(GLOB SOURCES "src/*.cpp")

# 添加可执行文件
add_executable(simple_app ${SOURCES})

# 包含头文件
target_include_directories(simple_app PRIVATE include)

# 链接库
target_link_libraries(simple_app PRIVATE pthread)

# 设置输出路径
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
```

### 💡 示例 2：静态库项目

cmake

```
cmake_minimum_required(VERSION 3.10)
project(MyLibrary VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 搜寻源文件（排除main.cpp）
file(GLOB SOURCES "src/*.cpp")
list(REMOVE_ITEM SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cpp)

# 创建静态库
add_library(mylib STATIC ${SOURCES})

# 包含头文件
target_include_directories(mylib
    PUBLIC include          # 公共头文件
    PRIVATE src             # 私有头文件
)

# 设置库输出路径
set(LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lib)

# 创建可执行文件（测试用）
add_executable(test_app src/main.cpp)
target_link_libraries(test_app PRIVATE mylib pthread)
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
```

### 💡 示例 3：动态库项目

cmake

```
cmake_minimum_required(VERSION 3.10)
project(MySharedLib VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 搜寻源文件
file(GLOB SOURCES "src/*.cpp")
list(REMOVE_ITEM SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cpp)

# 创建动态库
add_library(mylib SHARED ${SOURCES})

# 包含头文件
target_include_directories(mylib
    PUBLIC include
    PRIVATE src
)

# 设置库输出路径
set(LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lib)

# 创建可执行文件
add_executable(myapp src/main.cpp)
target_include_directories(myapp PRIVATE include)
target_link_libraries(myapp PRIVATE mylib pthread)
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
```

### 💡 示例 4：多目录项目（嵌套 CMake）

cmake

```
# 根目录 CMakeLists.txt
cmake_minimum_required(VERSION 3.10)
project(MyProject VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 设置全局路径
set(LIB_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
set(EXEC_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
set(HEADER_PATH ${PROJECT_SOURCE_DIR}/include)

# 添加子目录
add_subdirectory(src)
add_subdirectory(lib)
add_subdirectory(apps)
```

cmake

```
# src/CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

# 搜寻源文件
file(GLOB SRC_SOURCES "*.cpp")

# 创建库
add_library(core STATIC ${SRC_SOURCES})

# 包含头文件
target_include_directories(core PUBLIC ${HEADER_PATH})

# 设置输出路径
set(LIBRARY_OUTPUT_PATH ${LIB_OUTPUT_PATH})
```

cmake

```
# lib/CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

# 搜寻源文件
file(GLOB LIB_SOURCES "*.cpp")

# 创建库
add_library(utils STATIC ${LIB_SOURCES})

# 包含头文件
target_include_directories(utils PUBLIC ${HEADER_PATH})

# 设置输出路径
set(LIBRARY_OUTPUT_PATH ${LIB_OUTPUT_PATH})
```

cmake

```
# apps/CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

# 搜寻源文件
file(GLOB APP_SOURCES "*.cpp")

# 创建可执行文件
add_executable(myapp ${APP_SOURCES})

# 包含头文件
target_include_directories(myapp PRIVATE ${HEADER_PATH})

# 链接库
target_link_libraries(myapp PRIVATE core utils pthread)

# 设置输出路径
set(EXECUTABLE_OUTPUT_PATH ${EXEC_OUTPUT_PATH})
```

### 💡 示例 5：带测试的项目

cmake

```
cmake_minimum_required(VERSION 3.10)
project(MyProject VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 搜寻源文件
file(GLOB SOURCES "src/*.cpp")

# 创建库
add_library(mylib STATIC ${SOURCES})
target_include_directories(mylib PUBLIC include)

# 创建主程序
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE mylib pthread)

# 创建测试程序
enable_testing()  # 启用测试

add_executable(test_app tests/test_main.cpp)
target_include_directories(test_app PRIVATE include)
target_link_libraries(test_app PRIVATE mylib pthread)

# 添加测试
add_test(NAME unit_tests COMMAND test_app)

# 设置输出路径
set(LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/lib)
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
```

### 💡 示例 6：跨平台项目

cmake

```
cmake_minimum_required(VERSION 3.10)
project(CrossPlatformApp VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 搜寻源文件
file(GLOB SOURCES "src/*.cpp")

# 根据平台添加不同的源文件
if(WIN32)
    list(APPEND SOURCES "src/windows_specific.cpp")
elseif(UNIX AND NOT APPLE)
    list(APPEND SOURCES "src/linux_specific.cpp")
elseif(APPLE)
    list(APPEND SOURCES "src/macos_specific.cpp")
endif()

# 创建可执行文件
add_executable(myapp ${SOURCES})

# 根据平台链接不同的库
if(WIN32)
    target_link_libraries(myapp PRIVATE ws2_32)
elseif(UNIX)
    target_link_libraries(myapp PRIVATE pthread dl)
endif()

# 包含头文件
target_include_directories(myapp PRIVATE include)

# 设置输出路径
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
```

## 📚 六、CMake 最佳实践

### ✅ 推荐做法

**使用 target_* 命令而不是全局命令**

cmake

```
# ✅ 推荐
target_include_directories(myapp PRIVATE include)
target_link_libraries(myapp PRIVATE mylib)

# ❌ 不推荐
include_directories(include)
link_libraries(mylib)
```

**使用 target_compile_features () 指定 C++ 特性**

cmake

```
add_executable(myapp main.cpp)
target_compile_features(myapp PRIVATE cxx_std_11)
```

**使用 find_package () 查找第三方库**

cmake

```
find_package(OpenSSL REQUIRED)
target_link_libraries(myapp PRIVATE OpenSSL::SSL)
```

**使用 option () 提供配置选项**

cmake

```
option(BUILD_TESTS "Build test programs" ON)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

if(BUILD_TESTS)
    add_subdirectory(tests)
endif()
```

**使用 install () 命令安装文件**

cmake

```
install(TARGETS myapp DESTINATION bin)
install(FILES ${HEADER_FILES} DESTINATION include)
```

### ⚠️ 常见错误和注意事项

**路径使用绝对路径或 CMake 变量**

cmake

```
# ✅ 正确
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)

# ❌ 错误（相对路径可能不明确）
set(EXECUTABLE_OUTPUT_PATH ./bin)
```

**GLOB 不会自动检测新文件**

cmake

```
# 使用GLOB时，添加新文件后需要重新运行cmake
file(GLOB SOURCES "*.cpp")
```

**区分大小写**

cmake

```
# CMake命令不区分大小写，但变量名区分大小写
SET(MY_VAR value)  # 设置变量
message(${my_var}) # ❌ 找不到（应该是MY_VAR）
```

**列表元素用分号分隔**

cmake

```
# 列表元素之间用分号或空格
set(MY_LIST item1;item2;item3)
set(MY_LIST item1 item2 item3)
```

## 📚 七、CMake 命令速查表

表格

|命令|用途|示例|
|---|---|---|
|cmake_minimum_required()|指定 CMake 最低版本|`cmake_minimum_required(VERSION 3.10)`|
|project()|定义项目|`project(MyProject VERSION 1.0.0)`|
|set()|设置变量|`set(VAR value)`|
|add_executable()|添加可执行文件|`add_executable(app main.cpp)`|
|add_library()|添加库|`add_library(lib STATIC src.cpp)`|
|include_directories()|添加头文件路径|`include_directories(include)`|
|target_include_directories()|为目标添加头文件路径|`target_include_directories(app PRIVATE include)`|
|link_directories()|添加库搜索路径|`link_directories(lib)`|
|target_link_libraries()|链接库|`target_link_libraries(app lib pthread)`|
|file(GLOB)|搜寻文件|`file(GLOB SRC "*.cpp")`|
|list()|列表操作|`list(APPEND SRC file.cpp)`|
|add_subdirectory()|添加子目录|`add_subdirectory(src)`|
|target_compile_options()|添加编译选项|`target_compile_options(app PRIVATE -Wall)`|
|target_compile_definitions()|添加预处理器定义|`target_compile_definitions(app PRIVATE DEBUG)`|
|install()|安装文件|`install(TARGETS app DESTINATION bin)`|

## 📚 八、CMake 构建类型

bash

运行

```
# Debug：包含调试信息，无优化
cmake -DCMAKE_BUILD_TYPE=Debug ..

# Release：优化，无调试信息
cmake -DCMAKE_BUILD_TYPE=Release ..

# RelWithDebInfo：优化+调试信息
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..

# MinSizeRel：最小化大小的优化
cmake -DCMAKE_BUILD_TYPE=MinSizeRel ..
```

希望这份 CMake 用法总结对你有帮助！涵盖了从基础到高级的各个方面，包括实用示例和最佳实践。如有任何问题，随时告诉我。