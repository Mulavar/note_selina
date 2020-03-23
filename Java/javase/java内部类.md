内部类有四种：

+ 成员内部类

+ 局部内部类
+ 静态内部类（实际是静态嵌套类，是一个静态类，只是恰好位置位于某个类内部）
+ 匿名内部类





### 成员内部类

1. 不能拥有 `static` 的成员

2. 需要由外部类的实例创建

3. 外部类仍然能访问内部类的 `private` 成员

    ```java
    public class OuterClass {
        private String name = "outerclass";
    
        class InnerClass {
            void show() {
                System.out.println("InnerClass is invoting OutClass's fields: " + name);
            }
        }
    
        public static void main(String[] args) {
            OuterClass outerClass = new OuterClass();
            // 利用外部类实例创建
            InnerClass innerClass = outerClass.new InnerClass();
            innerClass.show();
        }
    }
    
    ```

### 局部内部类

定义并作用在方法内。

### 匿名内部类（编译时仍会生成class文件）

1. 匿名内部类没有访问修饰符
2. 匿名内部类**new的类**类型必须存在(可以是接口)
3. 当所在方法的形参需要被匿名内部类使用，那么这个形参就必须为final
4. 匿名内部类无构造方法

### 静态内部类

1. 创建不依赖外部类实例
2. 不能使用外部类的任何非**static**方法

```java
public class StaticClass {
    String name = "Static Class";
    static int num = 10;

    static class InnerClass {
        void show() {
            // 引用非static变量会出错
            // System.out.println(name);
            System.out.println(num);
        }
    }
}
```

创建静态内部类实例与成员内部类不同，如上述代码创建静态内部类是使用

`StaticClass.InnerClass innerClass = new StaticClass.InnerClass();`