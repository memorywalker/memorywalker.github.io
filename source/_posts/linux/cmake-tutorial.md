---
title: CMake Tutorial
date: 2023-05-07 10:11:49
categories:
- program
tags:
- cmake
---

## CMake 基本使用 
[Mastering CMake](https://cmake.org/cmake/help/book/mastering-cmake/chapter/Getting%20Started.html) 这个也是官网文档，比官方教程内容更好理解。

CMakeLists.txt是cmake的工程配置文件，一般把CMakeLists.txt文件放在工程根目录，同时新建一个Build目录，所有生成的工程文件都放在Build目录中，清除工程文件时，直接删除Build目录中的内容。

文档中一个相对完整的[教程](https://cmake.org/cmake/help/book/mastering-cmake/cmake/Help/guide/tutorial/index.html)，对应的[源代码](https://cmake.org/cmake/help/latest/_downloads/b8a65b498d06de3ba4a7dc1199af2298/cmake-3.26.3-tutorial-source.zip)

### 基本步骤

1. 给工程定义一个或多个CMakeLists.txt文件
2. 使用cmake命令生成目标工程文件vcproject/makefile
3. 使用工程文件编译工程

#### CMakeLists.txt

`CMakeLists.txt`是cmake的主文件，其中定义兼容的最小版本，工程的基本信息.这个文件一般在工程根目录。

```cmake
# always first line
cmake_minimum_required (VERSION 3.19)

# Projcet name and version
project (Test)

# output and dependency
add_executable(Test main.cpp)
```

#### 生成目标工程

在工程的目录新建build目录，到build目录中执行`cmake ..`生成工程文件。前两步也可以使用cmake自带的gui工具，linux平台依赖Curses进程名为ccmake。生成的工程文件会在build目录中，如果要清理工程，只需要把build目录清空即可。

```powershell
PS E:\code\rust\cargo_demo\src\build> cmake ..
-- Building for: Visual Studio 16 2019
-- Selecting Windows SDK version 10.0.18362.0 to target Windows 6.1.7601.
-- The C compiler identification is MSVC 19.26.28806.0
-- The CXX compiler identification is MSVC 19.26.28806.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: D:/Program Files (x86)/Microsoft Visual Studio/2019/Community/VC/Tools/MSVC/14.26.28801/bin/Hostx64/x64/cl.exe - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: D:/Program Files (x86)/Microsoft Visual Studio/2019/Community/VC/Tools/MSVC/14.26.28801/bin/Hostx64/x64/cl.exe - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: E:/code/rust/cargo_demo/src/build
```

![cmake_gui](../../uploads/linux/cmake_gui.png)
![cmake_gui](/uploads/linux/cmake_gui.png)

#### 编译工程

在build目录中执行`cmake --build .`编译当前生成的工程。生成的目标程序默认在Debug目录

```powershell
PS E:\code\rust\cargo_demo\src\build> cmake --build .
Microsoft (R) Build Engine version 16.6.0+5ff7b0c9e for .NET Framework
Copyright (C) Microsoft Corporation. All rights reserved.

  Checking Build System
  Building Custom Rule E:/code/rust/cargo_demo/src/CMakeLists.txt
  main.cpp
  Test.vcxproj -> E:\code\rust\cargo_demo\src\build\Debug\Test.exe
  Building Custom Rule E:/code/rust/cargo_demo/src/CMakeLists.txt
```

### CMake配置

#### 编译器配置

编译器配置有三种方式，优先推荐Generator的方式

* 使用Generator
* 使用环境变量
* 使用cache entry

##### Generator

使用`cmake -G`可以查看当前cmake支持的Generator。cmake会根据不同的Generator遵循对应的编译惯例

##### 环境变量

`CMAKE_C_COMPILER`指定C的编译器

`CMAKE_CXX_COMPILER`指定C++的编译器

#### 配置文件

使用配置文件可以让cmake根据配置生成一些配置头文件供工程的源程序代码使用，例如版本号信息

在工程根目录新建一个`TestConfig.h.in`的配置文件，cmake会把工程配置文件中的变量替换配置文件中的变量

```cmake
// the configured options and settings for Test, 
// CMake configures this header file the values for 
// @Test_VERSION_MAJOR@ and @Test_VERSION_MINOR@ will be replaced
#define Test_VERSION_MAJOR @Test_VERSION_MAJOR@
#define Test_VERSION_MINOR @Test_VERSION_MINOR@

#cmakedefine USE_MYMATH
```

cmake会在build目录生成`TestCongfig.h`，所以如果代码中要使用这里定义的变量，需要把build目录添加到include的目录中。这三行是有顺序要求的。

```cmake
# configure a header file to pass some of the CMake settings to the source code
configure_file(TestConfig.h.in TestConfig.h)

# output and dependency
add_executable(Test main.cpp)

# add the binary tree to the search path for include files
# so that we will find TestConfig.h
target_include_directories(Test PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
```

自动生成的`TestCongfig.h`头文件，

```c++
// the configured options and settings for Test, 
// CMake configures this header file the values for 
// 1 and 0 will be replaced
#define Test_VERSION_MAJOR 1
#define Test_VERSION_MINOR 0

#define USE_MYMATH
```

可以在代码中使用这些宏或变量声明

```c++
#include "TestConfig.h"
.....
    if (argc < 2) 
    {
        // report version
        std::cout << argv[0] << " Version " << Test_VERSION_MAJOR << "."
                << Test_VERSION_MINOR << std::endl;
        std::cout << "Usage: " << argv[0] << " number" << std::endl;
        return 1;
    } 
```

#### 使用依赖库

在库的源代码目录中新增库的CMakeLists.txt文件，其中INTERFACE说明库的使用者都要include库的源代码目录，有了这个INTERFACE的声明后，就可以不用在主程序的cmake中include库的源代码目录了

```cmake
# Add a library called FunLibs
add_library(FunLibs mysqrt.cxx)
target_include_directories(FunLibs
          INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
          )
```

在应用的CMakeLists.txt文件中配置库的编译和引用，因为库声明了INTERFACE要求，所以这里不需要include库的目录了，只是说明要链接库FunLibs。

```cmake
if(USE_MYMATH)
  add_subdirectory(FunLibs)
  list(APPEND EXTRA_LIBS FunLibs)
endif()

# set using the lib
target_link_libraries(Test PUBLIC ${EXTRA_LIBS})
```

#### CMAKE生成宏

可以根据条件来指定工程使用系统库还是自定义的库，或者一些特殊的配置，类似条件编译

1. 在cmake文件中使用`option`声明宏并定义宏的默认值
2. 在配置文件`TestConfig.h.in`中增加一句`#cmakedefine USE_MYMATH`，用来在配置头文件中生成宏，以便在代码中使用这个宏
3. cmake的配置文件中，可以使用这个宏来决定是否使用一些配置

下面的例子声明了`USE_MYMATH`宏，这个宏的默认是开，可以在cmakelists文件中使用，当这个宏开时，使用自己实现的库，而不用系统库。

同时配置文件中也会根据这里定义宏的值在`TestConfig.h`来定义宏 `#define USE_MYMATH`，

当不想配置这个宏时，可以在执行`cmake .. -DUSE_MYMATH=OFF`关闭这个宏，这样生成的头文件中，`USE_MYMATH`就是未定义状态`/* #undef USE_MYMATH */`。

需要注意的是宏的值会`CMakeCache.txt`被缓存，所以需要删除这个文件重新生成工程。

```cmake
# always first line
cmake_minimum_required (VERSION 3.19)

# Projcet name and version
project(Test VERSION 1.0)

# specify the C++ standard,  above the call to add_executable
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# This option will be displayed in the cmake-gui and ccmake with a default value of ON
option(USE_MYMATH "Use tutorial provided math implementation" ON)

# configure a header file to pass some of the CMake settings to the source code
configure_file(TestConfig.h.in TestConfig.h)

# add the library path
# add_subdirectory(FunLibs)

# use libs by options
if(USE_MYMATH)
  add_subdirectory(FunLibs)
  list(APPEND EXTRA_LIBS FunLibs)
  #list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/FunLibs")
endif()

set(SOURCE_FILES  
        main.cpp
        mode.cpp
)

# output and dependency
add_executable(Test ${SOURCE_FILES})

# set using the lib
#target_link_libraries(Test PUBLIC FunLibs)
target_link_libraries(Test PUBLIC ${EXTRA_LIBS})

# add the binary tree to the search path for include files
# so that we will find TestConfig.h
target_include_directories(Test PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           #${EXTRA_INCLUDES}
                           )
```

c++程序

```c++
#include <cmath>
#include <iostream>
#include <string>
#include "TestConfig.h"
#include "Mode.h"

#ifdef USE_MYMATH
# include "MathFunctions.h"
#endif

using namespace std;

int main(int argc, char* argv[])
{
    if (argc < 2) 
    {
        // report version
        std::cout << argv[0] << " Version " << Test_VERSION_MAJOR << "."
                << Test_VERSION_MINOR << std::endl;
        std::cout << "Usage: " << argv[0] << " number" << std::endl;
        return 1;
    }    

    // convert input to double
    const double inputValue = std::stod(argv[1]);

    #ifdef USE_MYMATH
    const double outputValue = mysqrt(inputValue);
    #else
    const double outputValue = sqrt(inputValue);
    #endif

    std::cout << "The square root of " << inputValue << " is " << outputValue
                << std::endl;

    CMode* mode = new CMode;
    if (mode)
    {
        mode->Display();
    }

    delete mode;
    mode = nullptr;   
    
    return 0;
}
```

#### 自定义命令

可以在编译完成后执行一些自定义的命令，例如在编译完成后，把生成的可执行文件拷贝到某个目录。这里的目录都需要使用绝对路径。

```cmake
add_custom_command(
  TARGET Test
  POST_BUILD
  COMMAND ${CMAKE_COMMAND}
  ARGS -E copy $<TARGET_FILE:Test> ${PROJECT_SOURCE_DIR}
  )
```

#### 交叉编译

cmake默认都是编译native的工程，交叉编译其他平台的程序时，需要额外信息告诉cmake编译器和运行库等。

交叉编译中，执行编译系统称为Host，运行程序的系统称为Target

##### 工具链配置

交叉编译需要指定交叉编译工具链，一般可以通过单独的一个toolchain文件说明目标程序的编译器，依赖库目录等。

例如创建一个`toolchain.cmake`文件用来编译运行在RaspberryPi的程序。

```cmake
# the name of the target operating system
set(CMAKE_SYSTEM_NAME linux)
# This variable is optional,当对不同的处理器需要配置不同的编译选项时，才需要配置
set(CMAKE_SYSTEM_PROCESSOR arm)

# which compilers to use for C and C++
set(CMAKE_C_COMPILER   "D:/SysGCC/raspberry/bin/arm-linux-gnueabihf-gcc.exe")
set(CMAKE_CXX_COMPILER "D:/SysGCC/raspberry/bin/arm-linux-gnueabihf-g++.exe")

# adjust the default behavior of the FIND_XXX() commands:
# search programs in the host environment
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)

# search headers and libraries in the target environment
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

指定编译器时最好用引号括起来，windows的目录需要使用`/`不能使用`\`会被解析为转义字符，这样这个工具链配置文件就固定生成给RaspberryPi使用的程序。工具链文件可以放在一个公共目录下，这样所有的工程都可以复用这个工具链配置

##### 生成工程文件 

```powershell
cmake -G"Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE=../toolchain.cmake -DCMAKE_BUILD_TYPE=Debug ..
```

其中使用`-DCMAKE_TOOLCHAIN_FILE`指定工具链文件，`-G"Unix Makefiles"`说明生成makefile类型的工程

```powershell
E:\code\rust\cargo_demo\src\build_linux>cmake -G"Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE=
../toolchain.cmake -DCMAKE_BUILD_TYPE=Debug ..
-- The C compiler identification is GNU 10.2.1
-- The CXX compiler identification is GNU 10.2.1
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: D:/SysGCC/raspberry/bin/arm-linux-gnueabihf-gcc.exe - s
kipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: D:/SysGCC/raspberry/bin/arm-linux-gnueabihf-g++.exe -
 skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: E:/code/rust/cargo_demo/src/build_linux
```

生成makefile文件之后，可以在build_linux目录中执行`cmake --build .`来生成最终的目标程序

```powershell
E:\code\rust\cargo_demo\src\build_linux>cmake --build .
Scanning dependencies of target Test
[ 50%] Building CXX object CMakeFiles/Test.dir/main.cpp.o
[100%] Linking CXX executable Test
[100%] Built target Test
```

把生成的Test程序传到之前的RaspberryPi的虚拟机中可以正常执行。

```shell
pi@raspberrypi:~ $ chmod +x Test
pi@raspberrypi:~ $ ./Test
The final price is: 8.4
```

#### 单元测试

在CMakeLists.txt中可以配置单元测试，编译程序后执行`ctest -C Debug -VV，对于MSVC需要指定测试的类型是Debug还是Release。对于GNU的，执行`ctest -N`或`ctest -VV`，N选项简化输出，VV选项详细输出

`add_test(NAME 用例名称 COMMAND 执行的命令和参数)`添加一个测试用例

还可以定义一个函数把测试的代码封装起来，下例中的`do_test`函数，其中使用了正则表达式进行匹配结果

在CMakeLists.txt最后添加

```cmake
# enable testing
enable_testing()

# does the application run
add_test(NAME Runs COMMAND Test 100)

# does the usage message work?
add_test(NAME Usage COMMAND Test)
set_tests_properties(Usage
    PROPERTIES PASS_REGULAR_EXPRESSION "Usage:.*number"
    )

# define a function to simplify adding tests
function(do_test target arg result)
    add_test(NAME Comp${arg} COMMAND ${target} ${arg})
    set_tests_properties(Comp${arg}
    PROPERTIES PASS_REGULAR_EXPRESSION ${result}
    )
endfunction(do_test)

# do a bunch of result based tests
do_test(Test 4 "4 is 2")
do_test(Test 9 "9 is 3")
do_test(Test 5 "5 is 2.236")
do_test(Test 7 "7 is 2.645")
do_test(Test 25 "25 is 5")
do_test(Test -25 "-25 is [-nan|nan|0]")
do_test(Test 0.0001 "0.0001 is 0.01")
```

输出如下

```powershell
PS E:\code\rust\cargo_demo\src\build> ctest -C Debug
Test project E:/code/rust/cargo_demo/src/build
    Start 1: Runs
1/9 Test #1: Runs .............................   Passed    0.01 sec
    Start 2: Usage
2/9 Test #2: Usage ............................   Passed    0.01 sec
    Start 3: Comp4
3/9 Test #3: Comp4 ............................   Passed    0.01 sec
    Start 4: Comp9
4/9 Test #4: Comp9 ............................   Passed    0.02 sec
    Start 5: Comp5
5/9 Test #5: Comp5 ............................   Passed    0.01 sec
    Start 6: Comp7
6/9 Test #6: Comp7 ............................   Passed    0.01 sec
    Start 7: Comp25
7/9 Test #7: Comp25 ...........................   Passed    0.02 sec
    Start 8: Comp-25
8/9 Test #8: Comp-25 ..........................   Passed    0.02 sec
    Start 9: Comp0.0001
9/9 Test #9: Comp0.0001 .......................   Passed    0.01 sec

100% tests passed, 0 tests failed out of 9

Total Test time (real) =   0.17 sec
```



