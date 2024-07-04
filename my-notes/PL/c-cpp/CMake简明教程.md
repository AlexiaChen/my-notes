#cmake
#cpp

## 前言

主要最近的换工作，完全在Linux下开发，虽然以前都接触过CMake，不过体系也是零散的，遂做了一个简短的CMake教程，以供后续快速入门。

另外，好久也没有写文章了，这份工作还是有一定的技术性，之前的那家公司是开发/维护，大部分工作都是维护，没有什么写文章的激情。

所以，今天是硬凑一篇文章。

## 正文

#### CMake

CMake是跨平台的元构建系统，也就是说，它不实际产生构建行为，它只是生成给其他构建系统使用的文件，比如Makefile，MSBuild的solution file。

CMake根据读取名为CMakeLists.txt的文件，然后生成平台特定的构建文件，但是一个很大的问题是，CMake官方提供的教程特别复杂，对于新手的个坑，很难快速入门。

这个教程会通过例子来学习怎么用CMake。以下我们提供几个C++源代码供例子使用：

- main.cpp
- vector.h
- vector.cpp
- array.h
- array.cpp

那么描述构建的CMakeLists.txt内容会是以下:

```cmake
cmake_minimum_required(VERSION 2.8)
project(pjc-lab5)

set(CMAKE_CXX_FLAGS "-std=c++14 -Wall ${CMAKE_CXX_FLAGS}")

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

add_executable(vector-test
    array.cpp
    vector.cpp
    main.cpp
)
```
以上代码很简单，但是第一个问题是，它是不可移植的，因为它没有任何逻辑判断就设置了GCC/Clang的特定编译参数。

第二个问题是，它全局改变了include的搜索路径。

CMake也要有好的书写习惯，采用更加现代的方式来写CMake文件:

```cmake
cmake_minimum_required(VERSION 3.5)
project(pjc-lab5 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)


add_executable(vector-test
    array.cpp
    vector.cpp
    main.cpp
)
```
注意了，以上代码有几点改变了：

- 强制要求了CMake的版本不得小于3.5，因为要使用CMake的一些新功能
- 直接指定该工程为C++工程。这样可以减少CMake查找tool chain的时间，它就不会去查找其他编译器了，也不会检查其他编译器是否正常了。
- 直接采用可跨平台的方式来指定采用的C++标准为C++ 14
- 打开CMAKE_CXX_STANDARD_REQUIRED开关，如果C++ 14标准不被支持，CMake会直接终止构建过程。反之，会采用老的标准来构建
- CMAKE_CXX_EXTENSIONS开关是告诉CMake采用更加通用的编译参数，比如这个开关打开，传递给GCC的参数就会是-std=c++14 而不是-std=gnu++14

然后在构建过程中，你会发现没有警告，因为CMake不会设定编译器的警告级别，需要你根据不同平台来指定相应的编译器警告参数:

```cmake
if ( CMAKE_CXX_COMPILER_ID MATCHES "Clang|AppleClang|GNU" )
    target_compile_options( vector-test PRIVATE -Wall -Wextra -Wunreachable-code -Wpedantic)
endif()
if ( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
    target_compile_options( vector-test PRIVATE -Wweak-vtables -Wexit-time-destructors -Wglobal-constructors -Wmissing-noreturn )
endif()
if ( CMAKE_CXX_COMPILER_ID MATCHES "MSVC" )
    target_compile_options( vector-test PRIVATE /W4 /w44265 /w44061 /w44062 )
endif()
```

如果就采用上述的CMake文件，那么它生成的工程文件并不好，没有预期，你会发现如果生成VS的solution，你打开工程，你会发现没有包含头文件（vector.h array.h）。因为CMake不理解C++语言，它只是构建工具。

所以CMake文件中要改变下:
```cmake
add_executable(vector-test
    array.cpp
    vector.cpp
    main.cpp
    array.h
    vector.h
)
```

当然，也可以通过CMake的source_group命令给文件归类:

```cmake
source_group("Tests" FILES main.cpp)
source_group("Implementation" FILES array.cpp vector.cpp)
```

这样VS工程下就可以看到对C++源文件分类的文件夹图标了。

#### Tests

CMake是一堆工具的集合，所以它有一个test runner，叫[CTest](https://cmake.org/cmake/help/latest/manual/ctest.1.html)。

要使用它，你需要显式指定它：
```cmake
add_test(NAME test-name COMMAND how-to-run-it)
```
测试返回0表示成功，返回其他值表示失败。

还可以自定义，通过[set_tests_properties](https://cmake.org/cmake/help/latest/command/set_tests_properties.html?highlight=set_tests_properties)来设置其[相关属性](https://cmake.org/cmake/help/v3.11/manual/cmake-properties.7.html#test-properties)。

对于我们的例子工程，我们仅仅是运行bin文件，并不做额外检查:

```cmake
include(CTest)
add_test(NAME plain-run COMMAND $<TARGET_FILE:vector-test>)
```
COMMAND后面的表达式是[generator-expression](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html)。

最后，我们的CMakeLists.txt的内容会是:

```cmake
cmake_minimum_required(VERSION 3.5)
project(pjc-lab5 
DESCRIPTION "Very nice project"
LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)


add_executable(vector-test
    array.cpp
    vector.cpp
    main.cpp
    array.h
    vector.h
)

source_group("Tests" FILES main.cpp)
source_group("Implementation" FILES array.cpp vector.cpp)


if ( CMAKE_CXX_COMPILER_ID MATCHES "Clang|AppleClang|GNU" )
    target_compile_options( vector-test PRIVATE -Wall -Wextra -Wunreachable-code -Wpedantic)
endif()
if ( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
    target_compile_options( vector-test PRIVATE -Wweak-vtables -Wexit-time-destructors -Wglobal-constructors -Wmissing-noreturn )
endif()
if ( CMAKE_CXX_COMPILER_ID MATCHES "MSVC" )
    target_compile_options( vector-test PRIVATE /W4 /w44265 /w44061 /w44062 )
endif()

include(CTest)
add_test(NAME plain-run COMMAND $<TARGET_FILE:vector-test>)
```

#### libraries

之前的教程都是很简单的例子，但是现实中的项目往往要拆分模块，链接外部的第三方库或者链接工程内的库。

如果不想阅读此章节，可以参考JetBrain的CLion提供的一个简明的[CMake教程](https://www.jetbrains.com/help/clion/quick-cmake-tutorial.html)。

里面记录如何包含链接外部的library。

构建生成一个library的cmake指令如下:

```cmake
add_library(libname [STATIC | SHARED] two.cpp three.h)
```

下面我们来写一个计算器程序的cmake,bin文件依赖了lib文件，这样看起来就像一个小工程了:

```cmake
cmake_minimum_required(VERSION 3.8)

project(Calculator LANGUAGES CXX)

add_library(calclib STATIC src/calclib.cpp include/calc/lib.hpp)
target_include_directories(calclib PUBLIC include)
target_compile_features(calclib PUBLIC cxx_std_11)

add_executable(calc apps/calc.cpp)
target_link_libraries(calc PUBLIC calclib)
```

#### 变量和缓存变量

为什么要说这个？因为如果cmake维护一个很大的工程，会有各种编译策略参数，这个就需要逻辑判断，变量这些就随之诞生了。

给一个局部变量设置一个值:

```cmake
set(MY_VARIABLE "value")
```

当然，cmake也有list类型的变量,有2种表达方式：

```cmake
set(MY_LIST "one" "two")
set(MY_LIST "one;two")
```

以上变量如果离开作用域就无效了。

下面来说下缓存变量，如果你要从命令行来设置cmake的变量，那么就需要声明缓存变量。类似于CMAKE_BUILD_TYPE这样的变量都是缓存变量：

```cmake
set(MY_CACHE_VARIABLE "VALUE" CACHE STRING "Description")
```

但是以上不会替换已经存在的值，需要按照以下这样：

```cmake
set(MY_CACHE_VARIABLE "VALUE" CACHE STRING "" FORCE)
mark_as_advanced(MY_CACHE_VARIABLE)
```

#### 环境变量

cmake可以访问环境变量：

```cmake
set(ENV{variable_name} value)
set(MY_VAR $ENV{variable_name})
```

#### 属性

cmake也会在属性中保存一些信息，这些属性有点像变量，但是这些属性的作用一般是针对一个target或者目录什么的。cmake的属性变量是以CMAKE_打头的，类似CMAKE_BUILD_TYPE，CMAKE_CXX_STANDARD这些。

比如使用CXX标准可以用设置属性的方式办到，有两种表达：

```cmake
set_property(TARGET TargetName
             PROPERTY CXX_STANDARD 11)

set_target_properties(TargetName PROPERTIES
                      CXX_STANDARD 11)
```

当然既然有set，当然有get:

```cmake
get_property(ResultVariable TARGET TargetName PROPERTY CXX_STANDARD)
```

### Cmake编程

#### 控制流

```cmake
if(variable)
    # If variable is `ON`, `YES`, `TRUE`, `Y`, or non zero number
else()
    # If variable is `0`, `OFF`, `NO`, `FALSE`, `N`, `IGNORE`, `NOTFOUND`, `""`, or ends in `-NOTFOUND`
endif()
```

```cmake
if("${variable}")
    # True if variable is not false-like
else()
    # Note that undefined variables would be `""` thus false
endif()
```

#### 宏和函数

```cmake
function(SIMPLE REQUIRED_ARG)
    message(STATUS "Simple arguments: ${REQUIRED_ARG}, followed by ${ARGV}")
    set(${REQUIRED_ARG} "From SIMPLE" PARENT_SCOPE)
endfunction()

simple(This)
message("Output: ${This}")
```

#### 与源码文件“通信”

Cmake允许源代码访问cmake的变量，这个指令就是configure_file。

这样的功能在版本管理上经常使用：

Version.h.in
```c++
#pragma once

#define MY_VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define MY_VERSION_MINOR @PROJECT_VERSION_MINOR@
#define MY_VERSION_PATCH @PROJECT_VERSION_PATCH@
#define MY_VERSION_TWEAK @PROJECT_VERSION_TWEAK@
#define MY_VERSION "@PROJECT_VERSION@"
```

```cmake
configure_file (
    "${PROJECT_SOURCE_DIR}/include/My/Version.h.in"
    "${PROJECT_BINARY_DIR}/include/My/Version.h"
)
```

可以从上面的配置看出来，本质上cmake只是把后缀in的文件，进行变量标记替换，然后再拷贝到指定目录的文件名。没有什么神奇的。

当然，cmake也可以反过来，cmake代码访问源码文件的内容：

```cmake
# Assuming the canonical version is listed in a single line
# This would be in several parts if picking up from MAJOR, MINOR, etc.
set(VERSION_REGEX "#define MY_VERSION[ \t]+\"(.+)\"")

# Read in the line containing the version
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/include/My/Version.hpp"
    VERSION_STRING REGEX ${VERSION_REGEX})

# Pick out just the version
string(REGEX REPLACE ${VERSION_REGEX} "\\1" VERSION_STRING "${VERSION_STRING}")

# Automatically getting PROJECT_VERSION_MAJOR, My_VERSION_MAJOR, etc.
project(My LANGUAGES CXX VERSION ${VERSION_STRING})
```

### 怎样规划你的工程结构

基本如下:

```txt
- project
  - .gitignore
  - README.md
  - LICENCE.md
  - CMakeLists.txt
  - cmake
    - FindSomeLib.cmake
  - include
    - project
      - lib.hpp
  - src
    - CMakeLists.txt
    - lib.cpp
  - apps
    - CMakeLists.txt
    - app.cpp
  - tests
    - testlib.cpp
  - docs
    - Doxyfile.in
  - extern
    - googletest
  - scripts
    - helper.py

```

### 运行其他的程序

编译的时候要做很多事情，比如你构建完毕要用nsis进行打包等等，有些时候不得不调用其他命令行。

#### 配置时运行一个命令

```cmake
find_package(Git QUIET)

if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
    execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                    RESULT_VARIABLE GIT_SUBMOD_RESULT)
    if(NOT GIT_SUBMOD_RESULT EQUAL "0")
        message(FATAL_ERROR "git submodule update --init failed with ${GIT_SUBMOD_RESULT}, please checkout submodules")
    endif()
endif()
```

#### 编译时运行一个命令

```cmake
find_package(PythonInterp REQUIRED)
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/include/Generated.hpp"
    COMMAND "${PYTHON_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/scripts/GenerateHeader.py" --argument
    DEPENDS some_target)

add_custom_target(generate_header ALL
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/include/Generated.hpp")

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/Generated.hpp DESTINATION include)
```

### Cmake其他的一些特性

CMAKE_BUILD_TYPE变量默认既不是Debug也不是Release。必须指定。

关于C++ 11等一些新标准，有以下配置:

```cmake
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set_target_properties(myTarget PROPERTIES
    CXX_STANDARD 11
    CXX_STANDARD_REQUIRED YES
    CXX_EXTENSIONS NO
)
```

#### 查找库

```cmake
find_library(MATH_LIBRARY m)
if(MATH_LIBRARY)
    target_link_libraries(MyTarget PUBLIC ${MATH_LIBRARY})
endif()
```