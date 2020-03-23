#### 注解的创建

```java
public @interface TestAnnotation {
	// 创建了一个名为 TestAnnotation 的注解
}
```



#### 元注解

元注解是可以注解到注解上的注解，是一种基本注解。

java的元注解有五种：

- @Retention：保留该注解的存活时间。

    - RetentionPolicy.SOURCE 注解只在**源码阶段保留**，在编译器进行编译时它将被丢弃忽视。
    - RetentionPolicy.CLASS 注解只被保留到**编译进行的时候**，它并不会被加载到 JVM 中。
    - RetentionPolicy.RUNTIME 注解可以**保留到程序运行**的时候，它会被加载进入到 JVM 中，所以在程序运行时可以获取到它们。

- @Documented：将注解的元素包含到javadoc中

- @Target：指定了注解的运用场景。

    - ElementType.ANNOTATION_TYPE 可以给一个注解进行注解
    - ElementType.CONSTRUCTOR 可以给构造方法进行注解
    - ElementType.FIELD 可以给属性进行注解
    - ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
    - ElementType.METHOD 可以给方法进行注解
    - ElementType.PACKAGE 可以给一个包进行注解
    - ElementType.PARAMETER 可以给一个方法内的参数进行注解
    - ElementType.TYPE 可以给一个类型进行注解，比如类、接口、枚举

- @Inherited：例子如下

- ```java
    @Inherited
    @Retention(RetentionPolicy.RUNTIME)
    @interface Test {}
    
    @Test
    public class A {}
    
    public class B extends A {}
    ```

    若类B没有被其他的注解应有，则类B自动继承Test这个注解。

- @Repeatable （1.8中加入）





#### 注解的属性

注解的属性也叫做成员变量。注解**只有成员变量，没有方法**。注解的成员变量在注解的定义中以“**无形参的方法**”形式来声明，其**方法名定义了该成员变量的名字**，其**返回值定义了该成员变量的类型**。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
	// 可以设置默认值，如果设置则为 int id() default -1;
    int id();
    String msg();

}
```

上面代码定义了 TestAnnotation 这个注解中拥有 id 和 msg 两个属性。

注：在注解中定义属性时它的类型必须是 **8 种基本数据类型外加 String、类、接口、注解及它们的数组**。





#### 注解的提取（通过反射）

可应用该方法判断是否应用某注解，

```java
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
```

再通过 `getAnnotation()` 方法来获取 `Annotation` 对象。

```java
public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}

// 或者是 getAnnotations() 方法。
public Annotation[] getAnnotations() {}
```

之后就可以利用判断注解做自己想做的事。