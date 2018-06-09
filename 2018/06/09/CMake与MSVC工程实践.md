# CMake与MSVC工程化实践

## CMake基础
cmake无疑是最流行的c++跨平台构建工具之一，关于cmake入门指南这里不再赘述，官方文档是最好的参考，这里通过一个例子简述构建一个工程常用的函数和变量。

假设此项目有三个文件hello.h、hello.cpp、main.cpp，hello.h和hello.cpp导出一个`void hello();`的函数，在main.cpp中使用，CMakeList.txt如下：

```CMake
# 指明当前CMake的版本不能小于指定版本
cmake_minimum_required (VERSION 3.4)
# 工程名称，如果生成msvc工程，这个同样是解决方案名称
project(MyProject)

# 将hello.h、hello.cpp保存到${HELLO_SRC}变量中，作为文件列表
file(GLOB HELLO_SRC
    hello.h
    hello.cpp
)
file(GLOB MAIN_SRC
    main.cpp
)

# ${HELLO_SRC}生成为一个共享库，在windows系统自然是dll模块
add_library(hello SHARED ${HELLO_SRC})

# ${MAIN_SRC}生成一个可执行程序main.exe/main.out
add_executable(main ${MAIN_SRC})

# 生产目标main的时候指定链接器链接hello模块
target_link_libraries(main hello)

```
这样便完成了一个非常简单的CMake工程，同时它也包含了我们工程最关注的一些东西，包括生成库文件，生成可执行文件，如何链接库等等。

## Debug与Release
有时候我们希望生成带调试信息的库或者可执行文件与正式发布的文件进行区分，最常见的就是把带调试信息的文件后缀加'd'，同时为了目录结构的整齐，需要把它们的生成目录进行区分，那么可以参考以下的CMake命令：
```CMake
# 取消CMake默认生成的工程选项，仅保留Debug与Release（只对msvc这样的多样化构建ide有效）
if(CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_CONFIGURATION_TYPES Debug Release)
    set(CMAKE_CONFIGURATION_TYPES "${CMAKE_CONFIGURATION_TYPES}" CACHE STRING
        "Reset the configurations to what we need"
        FORCE)
endif()

# 可执行文件生成目录
# ${PROJECT_BINARY_DIR}运行CMake命令的所在目录（用于CMake的分离式编译）
# ${PROJECT_SOURCE_DIR}是工程的根目录（根CMakeList.txt的所在路径）
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${PROJECT_BINARY_DIR}/bin/Debug)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${PROJECT_BINARY_DIR}/bin/Release)

# 库文件生成目录
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${PROJECT_BINARY_DIR}/lib)

# 可执行文件后缀
set_target_properties(${TARGET_NAME} PROPERTIES DEBUG_POSTFIX "d")

# 库文件后缀
set(CMAKE_DEBUG_POSTFIX "d")

```

这时候会发现一些有争议的地方，如果库文件带上了'd'后缀，那么`target_link_libraries`链接库的时候如何分别链接Debug的库和Release的库的呢。可以分两种情况来说明，如果是上文中类似hello库这种在工程内部'声明'的库，那么我们可以不用考虑，仍然链接hello即可，在生成实际的msvc工程时cmake会为我们自动进行区分；如果使用第三方库，就需要特别的选项去区分：
```CMake
target_link_libraries(main 
    debug hello
    optimized hellod
)
```
简单的理解debug选项后面表明带调试信息的库就去搜索hellod,而不带调试信息的库就去搜索hello，特别的对于第三方库，需要注意声明库目录，可以参考`link_directories`

## 生成MSVC工程
当编写好一个基本的CMake工程后如何与Visual Studio结合使用呢，可以参考`cmake -G`命令：
```
# 在执行CMake命令的地方生成visual studio 2017的解决方案（32位构建选项）
cmake -G 'Visual Studio 15 2017' ${PROJECT_SOURCE_DIR}
# 生成vs2015的64位构建方案
cmake -G 'Visual Studio 14 2015 Win64' ${PROJECT_SOURCE_DIR}
# 根据你计算机内最新的visual studio版本生成解决方案
cmake ${PROJECT_SOURCE_DIR}
```
打开生成的解决方案会发现除了所有的库的可执行文件对应一个project外，还有ALL_BUILD和一个ZERO_CHECK项目，ALL_BUILD顾名思义就是构建所有的项目了。而构建ZERO_CHECK会触发cmake重新检查CMakeList.txt并且重新加载解决方案，这样就避免了修改CMakeList.txt后重新执行命令生成msvc工程的麻烦了，同时还能保留设置的断点、书签等等。

## 结束语
上文所述的仅仅是CMake的冰山一角，还有许许多多的功能和选项可以方便我们进行工程文件的管理。

CMake不仅仅是跨平台构建工程的不二选择，就是仅仅使用它来构建windows工程也提供了许多便利，尤其是鉴于visual studio更新比较频繁，如果我们使用cmake管理我们的工程，同时使用标准的c++去开发，那么visual studio版本的迁移也不是一件头疼的事情了~