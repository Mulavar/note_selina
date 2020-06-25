## 1. Spring简介

一句话概括Spring，Spring是一个开源的、轻量的**控制反转(IOC)**和**面向切面编程(AOP)**的框架。

### 1.1 Spring、SpringMVC、Spring Boot和Spring Cloud的区别

SpringMVC是在Spring基础之上结合了MVC三层架构的框架，使用DispatchServlet和视图解析器做到了更方便的视图跳转和控制器调度处理，主要负责处理WEB应用开发。SpringMVC是Spring的一个子模块。

SpringBoot则使用了约定大于配置的理念，简化了Spring配置，专注于微服务的开发。

SpringCloud专注于全局微服务的整合、管理。



## 2. 控制反转—IOC

### 2.1 基本概念

IOC：控制反转，是一种设计思想。即将对象创建的控制权转移给了Spring容器。

假设我们创建了类A，类A中有个成员变量类B。

- 传统开发：我们需要手动创建类B的对象，将他赋给类A的对象，这叫主动控制对象的创建；
- IOC：Spring创建好B对象，然后存储到一个容器里面，当A对象需要使用B对象时，Spring就从存放对象的那个容器里面取出A要使用的那个B对象。

使用传统方式，A对B产生了依赖，也就是**A和B之间存在一种耦合关系（假设B的实现有了改动，我们可能还需要改动A的源码）**，而通过IOC的思想，这种依赖关系就被容器取代了，所有对象的创建依赖于容器，在启动的时候，Spring会读取配置文件，并将创建的对象先注入容器，之后我们想要使用直接从容器中取就可以。

DI：依赖注入，可以理解为是Spring实现IOC的具体方式。



### 2.2 Spring创建对象方式

```java
class UserService {
    private UserDao dao;
    
    public void setUserDao(UserDao dao) {
        this.userDao = dao;
    }
}

class UserDao {
    private String name;
    
	public void setName(String name) {
        this.name = name;
    }
}
```



#### 2.2.1 构造函数

```java
<bean id="dao" class="com.dong.dao.UserDao">
    <!--使用name定位构造函数-->
    <constructor-arg name="name" value="dongjh"></constructor-arg>
</bean>


<bean id="service" class="com.dong.service.UserService">
    <!--使用type定位构造函数-->
    <constructor-arg type="com.dong.dao.UserDao" ref="dao"></constructor-arg>
</bean>
```

注：

1. 如果某类有无参构造方法，则默认调用无参构造方法；
2. 如果某类只有一个构造方法，则默认调用该构造方法。



#### 2.2.2 property（调用set方法创建）

```xml
<bean id="dao" class="com.dong.service.UserDao">
    <!--基本类型使用value注入属性-->
	<property name="name" value="dongjh"></property>
</bean>

<bean id="service" class="com.dong.service.UserService">
    <!--引用类型使用ref属性-->
	<property name="userDao" ref="dao"></property>
</bean>
```

注：

1. 使用属性注入时，容器会**默认调用无参构造方法创建实例**，所以需要保证要创建的bean有无参的构造方法；
2. 使用属性注入等价于调用了`setXXX`方法，XXX是**property标签中的name**，假设上面的代码的`setName`方法改成了`setName1`，则第一个bean的property需这样写：

```xml
<property name="name1" value="dongjh"></property>
```



### 2.3 Bean的自动装配(autowiring)

自动装配即Spring在上下文中自动寻找，自动给bean装配属性。

bean的自动装配主要有三种策略：

1. byname：把与某bean（A）属性**具有相同名字**的其他bean（B）自动装配到该bean（A）的属性中；
2. byType：把与某bean（A）属性**具有相同类型**的其他bean（B）自动装配到该bean（A）的属性中；
3. constructor：把与某bean（A）的构造器入参**具有相同类型**的其他bean（B）自动装配到该bean（A）的属性中；

#### byName

##### xml配置方式

```xml
<!--由于该bean的id为userDao，所以会直接自动装配下面service的userDao属性中-->
<bean id="userDao" class="com.dong.service.UserDao">
	<property name="name" value="dongjh"></property>
</bean>

<!--该类有setUserDao方法，且设置了属性自动装配byName-->
<bean id="service" class="com.dong.service.UserService" autowired="byName">
</bean>
```



#### byType

##### xml配置方式

```xml
<!--该bean的类型为com.dong.service.UserDao，所以会直接自动装配下面service的userDao属性中-->
<bean id="userDao111" class="com.dong.service.UserDao">
	<property name="name" value="dongjh"></property>
</bean>

<!--该类一个com.dong.service.UserDao类型的属性，且设置了属性自动装配byType-->
<bean id="service" class="com.dong.service.UserService" autowired="byType">
</bean>
```

##### 



### 2.4 Spring注解配置

JDK1.5开始支持注解，而Spring是2.5开始支持。

使用Spring注解配置需要先导入约束: context名称空间和注解扫描。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <!--注：只会寻找在同一个应用上下文中定义的注解，
    比如将这个声明放置到DispatcherServlet的配置上下文中，
    它就只会检查controllers的上下文，而不会检查services的上下文，
    详情见https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-annotation-config-->
    <context:annotation-config/>
    <!--指定注解扫描返回-->
    <context:component-scan base-package="com.dong.dao"></context:component-scan>

</beans>
```



#### @Autowired

之后在需要自动装配的**属性或set方法**上使用@AutoWired注解，就可以实现自动装配。

```java
//详细的可修饰范围
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
```

注：@Autowired会先使用byType，再按byName的顺序自动装配。

@Autowired有一个常用属性：required，默认值为true，这个属性表示修饰的成员变量是否一定需要装配一个bean：

- 当required设置为true时，表示必须有一个可以自动装配到该成员变量的bean，否则报错；
- 当required设置为false时，则表示该成员变量可以为null；



#### @Qualifier

当有多个bean满足自动装配条件时，可以使用这个指定装配的bean，一般配合@Autowired使用。

@Qualifier有个属性value，给这个value设置想要装配的bean的id即可。



#### @Component

修饰类，表示将该类的bean对象注入到Spring容器中。



@Component有个属性value，表示注入时设置的id。

如：

```java
//等价于<bean id="user" class="User">
@Component(value="user")
public class User {}
```

@Component有一些衍生注解，如Web开发会按照MVC架构分层，根据不同的功能层，衍生出了一些注解：

- @Repository：针对dao层
- @Service：针对service层
- @Controller：针对controller层 

这四个注解功能一致，都表示将某类的bean对象注入到Spring容器中。



#### @Value

给属性注入值，主要用于给成员变量注入值，简单的可以**注入基本类型的值**，复杂时可以注入一些操作系统属性或其他bean的属性。



#### @Scope

用于设置bean的作用域，先介绍最常见的：

- singleton：全局唯一，单例；
- prototype：每次注入创建一个新的对象。



#### 注解和配置的区别

xml功能更加强大，方便管理维护。注解适用于简单的属性注入。



## 3. 面向切面编程—AOP