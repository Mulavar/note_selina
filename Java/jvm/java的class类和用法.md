Java程序在运行时，Java运行时系统一直对所有的对象进行所谓的运行时类型标识，即所谓的**RTTI（Run-Time Type Identification）**。

> 这项信息**纪录了每个对象所属的类**。虚拟机通常使用运行时类型信息选准正确方法去执行，用来**保存这些类型信息的类是Class类**。**Class类封装一个对象和接口运行时的状态**，当装载类时，Class类型的对象自动创建。

我们自己无法生成一个Class对象（构造函数为private)，而 **这个Class类的对象是在当各类被调入时，由 Java 虚拟机自动创建 Class 对象，或通过类装载器中的 defineClass 方法生成**。

```java
//通过该方法可以动态地将字节码转为一个Class类对象
protected final Class<?> defineClass(String name, byte[] b, int off, int len) throws ClassFormatError
{
    return defineClass(name, b, off, len, null);
}
```

