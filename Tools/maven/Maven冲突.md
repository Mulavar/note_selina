## Maven依赖冲突

如依赖关系为

project->A->common-1.0

project->B->tmp->common2.0

由于maven采用最近路径优先，则2.0会被自动屏蔽，想要显式使用2.0，有两种方式：

1. 在**project中显式声明**使用common版本2.0；
2. 在**project依赖A的声明中**去掉对common-1.0的依赖；
3. 在**A的依赖中**声明common-1.0的范围为optional；

gradle采用最新版本优先。

