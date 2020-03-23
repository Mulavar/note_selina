## 常用变量

### 常用路径

`CMAKE_BINARY_DIR` / `PROJECT_BINARY_DIR`：全路径/build

`CMAKE_SOURCE_DIR` / `PROJECT_SOURCE_DIR`：全路径/src

`CMAKE_INCLUDE_PATH`：包含头文件目录，环境变量，非cmake变量。

`CMAKE_LIBRARY_PATH`：链接库目录，环境变量。

`CMAKE_CURRENT_SOURCE_DIR`：当前处理的CMakeLists.txt所在的路径。

`CMAKE_CURRENT_BINARY_DIR`：如果是 in-source 编译，它跟 `CMAKE_CURRENT_SOURCE_DIR` 一致，如果是 out-of-source 编译，target编译目录。



- in-source：所有临时文件直接创在当前目录下。
- out-of-source：新建build目录，`cd build` 后进行 cmake 编译，所有的临时文件都会放在build目录下不会和source有任何的关联。



### 设定输出目录

EXECUTABLE_OUTPUT_PATH：`set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)`，可执行文件输出目录。

CMAKE_ARCHIVE_OUTPUT_DIRECTORY：`set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)`，设置静态库输出目录。

CMAKE_LIBRARY_OUTPUT_DIRECTORY：`set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)`，编译时lib库输出目录。

CMAKE_RUNTIME_OUTPUT_DIRECTORY：`set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)`，动态库输出目录。





## 常用函数

### add_library(将指定的源文件生成链接文件，然后添加到工程中去)

```cmake
add_library(<name> [STATIC | SHARED | MODULE]
            [EXCLUDE_FROM_ALL]
            [source1] [source2] [...])
```



### TARGET_LINK_LIBRARIES(设置要链接的库文件的名称)

`TARGET_LINK_LIBRARIES(myProject hello)`，连接libhello.so库

`TARGET_LINK_LIBRARIES(myProject libhello.so)`，连接libhello.so库

`TARGET_LINK_LIBRARIES(myProject libhello.a)`，连接libhello.a库

