在虚拟机层面上，是没有泛型这一概念的——所有对象都属于普通类。在编译时，所有的泛型类都会被视为原生态类型，**原生态类型的含义是不带任何实际参数的泛型名称**。

三种类型的区别：

+ `List`：原生态类型
+ `List<Object>`：表示List中可以容纳任意类型对象
+ `List<?>`：表示只能包含某一种未知对象类型

**由于泛型不可协变，虽然String是Object的子类，但 `List<String>` 并不是 `List<Object>` 的子类（不能互相转换）。**



泛型有三种使用方式：

+ 泛型类
+ 泛型接口
+ 泛型方法



### 泛型类

```java
class 类名称 <泛型标识：可以随便写任意标识号，标识指定的泛型的类型>{
  	private 泛型标识 /*（成员变量类型）*/ var; 
  		.....
	}
}
```



### 泛型接口

```java
//定义一个泛型接口
public interface Generator<T> {
    public T next();
}
```



### 泛型方法

```java
**
 * 泛型方法的基本介绍
 * @param tClass 传入的泛型实参
 * @return T 返回值为T类型
 * 说明：
 *     1）public 与 返回值中间<T>非常重要，可以理解为声明此方法为泛型方法。
 *     2）只有声明了<T>的方法才是泛型方法，泛型类中的使用了泛型的成员方法并不是泛型方法。
 *     3）<T>表明该方法将使用泛型类型T，此时才可以在方法中使用泛型类型T。
 *     4）与泛型类的定义一样，此处T可以随便写为任意标识，常见的如T、E、K、V等形式的参数常用于表示泛型。
 */
public <T> T genericMethod(Class<T> tClass)throws InstantiationException ,
  IllegalAccessException{
        T instance = tClass.newInstance();
        return instance;
}
```





**QA**：

1. **泛型类，是在实例化类的时候指明泛型的具体类型；泛型方法，是在调用方法的时候指明泛型的具体类型** 。

2. **如果静态方法要使用泛型的话，必须将静态方法也定义成泛型方法** 。