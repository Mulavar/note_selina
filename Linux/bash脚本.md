### 执行方式

1. source

    新开子进程执行，子进程中设置的变量不会传回到父进程

2. bash

    在父进程中直接执行，小数点方式同理



### 常用语法

#### 默认变量

$#：代表后接参数个数

$@：全部参数内容

$0：表示脚本文件名

$1、$2...：第一、第二个参数



#### 条件表达式

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



#### 函数

需要写在bash文件最前面，也使用$1、\$2获取参数，但不是命令行的参数。

```bash
function func_name() {
}
```



#### 循环

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

