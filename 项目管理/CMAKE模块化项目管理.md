项目组织：
- 项目名/include/项目名/模块名.h
- 项目名/src/模块名.cpp
在CMakeLists.txt中写：
```cmake
target_include_directories(项目名 PUBLIC include)
```
在源码中编写：
```cpp
#include <项目名/模块名.h>
项目名::函数名();
```
# find_package用法举例
- `find_package(OpenCV)`：查找名为OpenCV的包，找不到不报错，事后可以通过`${OpenCV_FOUND}` 查询是否找到。
- `find_package(OpenCV QUIET)`：查找名为OpenCV的包，找不到不报错，也不打印任何信息。
- `find_package(OpenCV REQUIRED)` ：查找名为OpenCV的包，找不到报错
- `find_package(OpenCV REQUIRED COMPONENTS core videoio)`：查找名为OpenCV的包，找不到报错，且必须具有`OpenCV::core`和`OpenCV::videoio` 这两个组件，如果没有这两个组件也会报错。
- `find_package(OpenCV REQUIRED OPTIONAL_COMPONENTS core videoio)`：查找名为OpenCV的包，找不到报错，可具有`OpenCV::core`和`OpenCV::videoio` 这两个组件，如果没有这两个组件不会报错，通过`${OpenCV_core_FOUND}` 查询是否找到core组件。
`find_package(OpenCV)`实际上是在找一个名为`OpenCVConfig.cmake`的文件（出于历史兼容性考虑，除了`OpenCVConfig.cmake`以外`OpenCV-config.cmake`也会被CMake识别）
同理，`find_package(Qt5)`则会去找名为`Qt5Config.cmake` 的文件(`Qt5_DIR`)。
`Qt5Config.cmake`是在安装Qt5时，随`libQt5Core.so` 等实际的库文件一起安装到系统中去的。包配置文件位于`/usr/lib/cmake/Qt5/Qt5Config.cmake`。实际的动态库文件位于`/usr/lib/libQt5Core.so`。
