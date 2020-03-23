### make和cmake的一些区别联系（[参考链接](https://www.zhihu.com/question/27455963)）

make工具可以看成是一个智能的批处理工具，它本身并没有编译和链接的功能，而是用类似于批处理的方式—通过**调用makefile文件中用户指定的命令来进行编译和链接**。

如果换平台的话makefile需要重新修改，于是出现cmake，**cmake可以更加简单地生成makefile文件给make调用，cmake根据CMakeLists.txt去生成makefile**。



## make([参考链接—阮一峰](http://www.ruanyifeng.com/blog/2015/02/make.html))

### makefile

makefile的文件规则如下：

```makefile
# 目标: 前置条件
<target>: <prerequisites>
# tab后 写命令
[tab] commands
```

#### 目标

目标可以是一个或多个文件名，多个文件名之间用空格分隔，还可以是某个操作的名字，这称为“伪目标”（phony target)，如：

```makefile
clean:
	rm *.o
```

clean是一个伪目标操作，如果当前目录正好有个文件名字为clean，则该命令不会被执行，可以通过声明clean是”伪目标“避免，如下：

```makefile
# 内置目标名
.PHONY: clean
clean:
	rm *.o temp
```

#### 前置条件

前置条件通常是一组文件名（也可以是伪目标），之间用空格分隔。

#### 命令

表示如何更新目标文件，由一行或多行的Shell命令组成。

每行命令之前必须有一个tab键。如果想用其他键，可以用内置变量.RECIPEPREFIX声明，如下：

```makefile
.RECIPEPREFIX = >
all:
> echo Hello, world
```

每一行命令在一个单独的shell中执行，shell之间没有继承关系，如果要执行有依赖关系的命令，需要写在一行，用分号分隔，或是在换行符前加反斜杠。

```makefile
# 方法1
var-kept:
    export foo=bar; echo "foo=[$$foo]"

# 方法2
var-kept:
    export foo=bar; \
    echo "foo=[$$foo]"
```

#### 常用命令

- \# 表示注释

- @ 关闭回声（执行每条命令前会打印）

- 通配符，与bash一致

- **Make命令允许对文件名，进行类似正则运算的匹配，主要用到的匹配符是%**。比如，假定当前目录下有 f1.c 和 f2.c 两个源码文件，需要将它们编译为对应的对象文件，如下

	```makefile
	%.o: %.c
	```

- 对变量赋值

    ```makefile
    # 调用自定义变量
    txt = Hello World
    test:
        @echo $(txt)
        
    # 调用shell变量
    test:
        @echo $$HOME
    ```

- 内置变量

    **$(CC)** 指向当前使用的编译器，**$(MAKE)** 指向当前使用的Make工具

- 自动变量



## cmake

### CMakeLists.txt编写基础

#### 单目录单源文件

```cmake
# 最小版本号
cmake_minimum_required(VERSION 3.14)
project(hello-world)
add_definitions("-Wall -std=c++11") # <= 新增的编译选项
# 添加编译文件
add_executable(hello_world main.cpp)
```



#### 单目录多源文件

```cmake
cmake_minimum_required(VERSION 3.14)

# 列举文件名 cmake会自己找依赖关系
add_executable(Main 
	main.cpp
	func1.cpp
	func2.cpp
)
```



#### 多目录编译



假设目录结构如下

- 项目名/

    - main.cpp

    - mod1/

        - func1.cpp
        - func2.cpp

    - mod2/

        - func1.cpp
        - func2.cpp

        

##### 方法一

```cmake
cmake_minimum_required(VERSION 3.14)
# 列出所有要编译的文件
add_executable(Main
  main.cpp
  mod1/func1.cpp
  mod1/func2.cpp
  mod2/func1.cpp
  mod2/func2.cpp
)
```



##### 方法二（推荐）

**每个目录都写一个**`CMakeLists.txt`

```cmake
#CMakeLists.txt
cmake_minimum_required(VERSION 2.8)
add_subdirectory(mod1) 
add_subdirectory(mod2) 
add_executable(Main main.cpp)
target_link_libraries(Main Mod1 Mod2) 
```



**mod1目录**

```cmake
#mod1/CMakeLists.txt
cmake_minimum_required(VERSION 3.14)
add_library(Mod1 STATIC
  func1.cpp
  func2.cpp
)
```



**mod2目录**

```cmake
#mod2/CMakeLists.txt
cmake_minimum_required(VERSION 3.14)
add_library(Mod2 STATIC
  func1.cpp
  func2.cpp
)
```

