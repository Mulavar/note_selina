### foreach

`$(foreach var, text, command)`

var：局部变量

text: 文件列表

command: 操作

return：每次操作的结果，以空格隔开



### wildcard

`$(wildcard pattern)`

使用通配符得到满足该模式的文件列表（普通函数引用情况下通配符会失效）

`$(wildcard *.c)`：获取当前目录下所有.c文件



### notdir

把展开的文件去除路径信息

`$(notdir a.c b.c ./sub/sa.c ./sub/sb.c)`：得到a.c b.c sa.c sb.c



### patsubst

`$(patsubst %.c,%.o,$(dir))`：将$(dir)中变量后缀为.c的都替换成.o

