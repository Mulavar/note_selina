## 1. 执行方式

1. source

    新开子进程执行，子进程中设置的变量不会传回到父进程

2. bash

    在父进程中直接执行，小数点方式同理



## 2. 常用语法

### 2.1 默认变量

$#：代表后接参数个数

$@：全部参数内容

$0：表示脚本文件名

$1、$2...：第一、第二个参数



### 2.2 条件表达式

```bash
if [ 判断条件 ]; then
	#do something
elif [ 判断条件 ]; then
	#do something
else
	#do something
fi
```

**注意：判断条件的每个组件都需要用空格分隔**



```bash
case $变量 in
	变量1)
	#do something
	;;
	变量2)
	#do something
	;;
	*)
	#do something
case
```



### 2.3 函数

需要写在bash文件最前面，也使用$1、\$2获取参数，但不是命令行的参数。

```bash
function func_name() {
}
```



### 2.4 循环

```bash
#方式1
while [ condition ] 
do
	#do sth.
done

#方式2 与 while 相反
until [ condition ]
do
	#do sth.
done

#方式3
for var in con1 con2 con3...
do
	#do sth.
done

#方式4
for ( ( init; cond; step ) )
do
	#do sth.
done
```



[[ expression ]]

根据执行结果返回 0 或 1，0 表示成功，1 表示有问题。经常用于组合表达式，判断某个字符串或文件是否存在，来决定是否执行后面的命令（ `[[ expression ]] && expression1` ）。
       

-a file
	True if file exists

-b file

​	True if file exists and is a block special file.

-c file

​	True if file exists and is a character special file.

-d file
              True if file exists and is a directory.
-e file
              True if file exists.
-f file
              True if file exists and is a regular file.
-g file
              True if file exists and is set-group-id.
-h file
              True if file exists and is a symbolic link.
-k file
              True if file exists and its sticky'' bit is set.
-p file

 	True if file exists and is a named pipe (FIFO).

-r file
              True if file exists and is readable.
-s file
              True if file exists and has a size greater than zero.

-t fd 

​	True if file descriptor fd is open and refers to a terminal.

-u file
		True if file exists and its set-user-id bit is set.

-w file
    True if file exists and is writable.

-x file

​	True if file exists and is executable.

-O file

​	True if file exists and is owned by the effective user id.

-G file

​	True if file exists and is owned by the effective group id.

-L file

​	True if file exists and is a symbolic link.

-S file

​	True if file exists and is a socket.

-N file

​	True if file exists and has been modified since it was last read.

file1 -nt file2

​	True  if file1 is newer (according to modification date) than file2, or if file1 exists and file2 does not.

file1 -ot file2

​	True if file1 is older than file2, or if file2 exists and file1 does not.

file1 -ef file2

​	True  if file1 and file2 refer to the same device and inode numbers.

​	True  if file1 and file2 refer to the same device and inode numbers.

-o optname

​	True if shell option optname is enabled.  See the  list  of  options under the description of the -o option to the set builtin below.

-z string	

​	True：字符串长度为 0，空字符串不行，如 [[ -z $NO_EXIST_VARIABLE ]]  去检测一个不存在的环境变量会报错，需要加上 "" 变成 `[[ -z "$NO_EXIST_VARIABLE" ]]

-n string

​	True if the length of string is non-zero.