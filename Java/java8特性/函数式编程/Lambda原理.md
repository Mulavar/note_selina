## 实现原理

java8的Lambda是通过invokedynamic和methodhandle实现的。

当我们的java文件里出现lambda表达式时，编译器会做以下事情：

**编译**

1. 为我们自动生成一个**静态私有的lambda方法实体（里面包含该lambda的实际逻辑）**，由于源码中不含该方法，因此该方法带有ACC_SYNTHETIC标记。
2. bootstrap methods属性中会添加一个LambdaMetafactory.metafactory的方法调用，用于生成CallSite。
3. 在原先声明lambda的地方添加一条invokedynamic指令，指向我们的bootstrap methods。

**运行**

1. CallSite在构建过程中会生成一个**目标类型（函数式接口）的一个匿名实现类，**该类关联编译第一步生成的lambda脱糖方法。
2. **CallSite里包含的MethodHandle指向的是一个总是返回刚刚创建的.class类的实例的方法,**由`MethodHandles.constant(samBase, inst)`完成
3. 调用该方法，创建一个匿名实现类的实例。

注意：invokedynamic**不会直接调用而是通过MethdHandle间接调用bootstrap methods。调用位置位于CallSite.makeSite()**



## 流程

所以invokedynamic指令运行的完整流程如下：

1. 启动方法指向`LambdaMetafactory.metafactory`，但是不会直接调用而是通过MethdHandle间接调用。调用位置位于`CallSite.makeSite()`；
2. `LambdaMetafactory.metafactory()`使用`InnerClassLambdaMetafactory.buildCallSite()`创建了最后的CallSite
3. `buildCallSite()`会创建一个**.class；**
4. `buildCallSite()`会向最后的CallSite里面放入一个可调用的`MethdHandle`；
5. 这个MethodHandle指向的是一个总是返回刚刚创建的.class类的实例的方法，由`MethodHandles.constant(samBase, inst)`完成
6. 最后，用invokevirtual调用CallSite里面的MethdHandle，返回.class类的示例，即`new MethodReference$$Lambda$1`

![image-20200214163849984](../../../resources/lambda执行流程.png)





## 问题

### BSM返回一个CallSite，invokedynamic如何生成对象。

CallSite里封装的MethodHandle的功能实际是创建lambda匿名类对象。

### LambdaMetafactory.metafactory()方法参数说明

```java
public static CallSite LambdaMetafactory.metafactory(MethodHandles.Lookup caller,
                                       String invokedName,
                                       MethodType invokedType,
                                       MethodType samMethodType,
                                       MethodHandle implMethod,
                                       MethodType instantiatedMethodType);
```

- `MethodHandles.Lookup caller` 表示哪个类引发了调动；
- `String invokedName` 表示生成的类的方法名，即接口的**方法名**；
- `MethodType invokedType` 表示CallSite的函数签名，其中**参数类型表示捕获变量的类型**，**返回类型是类要实现的函数式接口的名字**；
- `MethodType samMethodType` 表示**要实现的方法的函数签名和返回值**；
- `MethodHandle implMethod` 表示实现的方法里面应该调用的函数；

- `MethodType instantiatedMethodType` 表示调用方法的**运行时描述符**，如果不是泛型就和`samMethodType`一样

### 如何dump生成的lambda匿名类

在java虚拟机运行时添加选项 `-Djdk.internal.lambda.dumpProxyClasses`，如`java -Djdk.internal.lambda.dumpProxyClasses invokedynamic.CustomLmabdaDemo`。





[参考资料——invokedynamic流程和源码解析](https://cloud.tencent.com/developer/article/1395043)

[参考资料——从字节码角度分析invokedynamic](https://colobu.com/2014/11/06/secrets-of-java-8-lambda/)



