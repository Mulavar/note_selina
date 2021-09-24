### 1. getName(), getCanonicalName() 和getSimpleName()的区别

```java
public class ExternalClassConfig {
 
	private String desc;
 
	//    ...
    
    public static class InternalConfig {
    	//      ...
    }
 
    
}

@Test
public void testclassName() {
	System.out.println();
    // 外部类表示
    // getName()
	System.out.println("getName            " + ExternalClassConfig.class.getName());
    // getCanonicalName()
	System.out.println("getCanonicalName   " + ExternalClassConfig.class.getCanonicalName());
    // getSimpleName()
	System.out.println("getSimpleName      " + ExternalClassConfig.class.getSimpleName());
	
    // 内部类表示
    // getName()
	System.out.println("getName            " + InternalConfig.class.getName());
	// getCanonicalName()
    System.out.println("getCanonicalName   " +InternalConfig.class.getCanonicalName());
	// getSimpleName()
    System.out.println("getSimpleName      " +  InternalConfig.class.getSimpleName());
	
	System.out.println();
}
```
**输出结果**     

| 方法             | 值                                                  |
| ---------------- | --------------------------------------------------- |
| getName          | my.ExternalClassConfig                              |
| getName          | my.ExternalClassConfig$InternalConfig // 虚拟机表示 |
| getCanonicalName | my.ExternalClassConfig                              |
| getCanonicalName | my.ExternalClassConfig.InternalConfig // 更清晰表示 |
| getSimpleName    | ExternalClassConfig                                 |
| getSimpleName    | InternalConfig // 类名                              |

主要区别在表示数组的时候

| 方法             | 值                                       |
| ---------------- | ---------------------------------------- |
| getName          | [Ljava.lang.String; // 前面JNI字段描述符 |
| getCanonicalName | java.lang.String[]                       |
| getSimpleName    | String[]                                 |



### 2. 访问权限修饰符

1.  private: 被其修饰的类、属性以及方法只能被该类的对象访问，其子类不能访问**，更不能允许跨包访问。
2.  default：即不加任何访问修饰符（默认的访问权限），该模式下，只允许在同一个包中进行访问。
3.  protect: 被其修饰的类、属性以及方法**只能被类本身的方法及子类访问，即使子类在不同的包中也可以访问**，一般用于继承。
4.  public：可以随意访问。

#### 2.1 继承时的访问权限

继承时子类的访问权限不能小于父类的访问权限，比如：

> 父类的访问权限为protected时，子类必须是public或者protected

理解：假设多态的场景下，父类声明调用子类成员，**首先必须保证拥有访问父类该成员的权限，其次保证可以调用子类实际实现**，所以子类方法的访问权限不能小于父类。

