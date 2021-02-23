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

#### 2.3.1 byName

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



#### 2.3.2 byType

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



### 2.5 JavaConfig配置

`Spring 3.0`之后开始支持了一种更优的配置方式：基于Java类的配置方式。通过增加@Configuration注解表明这是一个配置类，其底层为@Component（即该类的bean对象也会被Spring容器接管）。

@Configuration结合@Bean注解可以实现XML配置实现注入的功能，效果与

```xml
<beans>
	<bean>...</bean>
    <bean>...</bean>
</beans>
```

类似，在使用@Bean注解时，表示将该方法返回的对象加载进Spring容器，在使用@Bean修饰方法时需要注意：

1. 方法带返回值，且返回类型为想要被管理的bean的类型；
2. 方法名即为默认的bean name，可使用@Bean的name属性自定义；
3. 方法可带参数，则此时传入的参数也必须是被Spring容器管理的bean。

以创建UserService和UserDao为例：

```java
@Configuration
public class UserConfiguration {
    @Bean
    public UserService userService() {
        return new UserService();
    }
    
    @Bean
    public UserDao userDao() {
        return new UserDao();
    }
}
```

当Spring扫描该类时，会自动将UserConfiguration、UserService、UserDao的bean注入到Spring容器中进行托管。



## 3. 面向切面编程—AOP

### 3.1 基本概念

在软件开发中，我们需要编写许多的业务代码，但还有很多代码是业务无关的，如

- 日志功能
- 缓存清理
- 资源回收
- 权限检查
- …

为了实现业务代码和非业务代码的分离，Spring利用代理模式抽出了非业务代码，形成了AOP思想。

AOP全称Aspect Oriented Programming，意为面向切面编程，假设业务代码本身是一条纵向的逻辑流，我们在其中找几个点（切点，Pointcut），插入非业务代码，就像是一个个切面（Aspect），这就是一个简单的AOP例子。

切面和切点很容易被混淆，严格来说，切点是切面的一部分信息，切面主要包含两部分信息：

- 切入点；
- 切入后要执行的动作（Advice）。

而在具体的实现中，Spring为切面还加了一个信息：在切点的哪个阶段执行？由此衍生出有：

- 前置通知（Before）：在目标方法被调用之前调用通知功能；
- 后置通知（After）：在目标方法被调用之后调用通知功能；
- 返回通知(After-returning)：在目标方法成功执行之后调用通知；
- 异常通知(After-throwing)：在目标方法抛出异常后调用通知；
- 环绕通知(Around)：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为。

关于AOP在Spring的实现有三种方式：

1. 实现原生的Spring API接口
2. 自定义切面类
3. 注解



### 3.2 实现原生Spring API接口

Spring提供了诸如MethodBeforeAdvice、AfterAdvice、AfterReturningAdvice等原生接口，我们可以通过实现这些接口构建切面，如

```java
@Component
public class Log implements MethodBeforeAdvice {
    /**
     * @param method  要执行的目标对象方法
     * @param objects 参数
     * @param o       目标对象
     * @throws Throwable
     */
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println(o.getClass().getName() + "-" + method.getName() + "将要执行");
    }
}
```

该Log类实现了MethodBeforeAdvice，成为了一个在方法执行前插入的切面，其次，我们需要在XML中进行配置，设置切点并建立其和切面的连接：

```xml
<aop:config>
    <!--切入点：execution表达式，execution(要执行的方法！)-->
    <aop:pointcut id="pointcut" expression="execution(* com.dong.service.UserServiceImpl.*(..))"/>
    <aop:advisor advice-ref="log" pointcut-ref="pointcut"></aop:advisor>
</aop:config>
```

**execution表示在方法执行时触发；\*表示方法返回值可以是任意类型，com.dong.service.UserServiceImpl.\*使用全限定类名和方法名指定要添加前置通知的方法，使用\*表示对com.dong.service.UserServiceImpl类下的所有方法都执行切入；(..)表示方法的参数列表，使用(..)表示方法的入参可以是任意类型。**

需要注意，Log必须是一个被Spring容器管理的bean对象，否则Spring将无法找到该切面的bean，因此使用@Component注解修饰（也可以使用XML配置等方式注入）。



### 3.3 自定义切面类

自定义切面不需要实现Spring的原生接口，首先需要定义一个切面类，并将加载给Spring容器进行托管。

```java
@Component(value="diy")
public class DiyPointCut {
    public void before() {
        System.out.println("方法执行前");
    }

    public void after() {
        System.out.println("方法执行后");
    }
}
```

与3.2相似，该方式需要通过XML配置设置切点以及其和切面的连接。

```xml
<aop:config>
    <!--自定义切面，ref：要引用的bean-->
    <aop:aspect ref="diy">
        <aop:pointcut id="pointcut" expression="execution(* com.dong.service.UserServiceImpl.*(..))"/>
        <!--指定advice和切点的关系-->
        <aop:before method="before" pointcut-ref="pointcut"></aop:before>
        <aop:after method="after" pointcut-ref="pointcut"></aop:after>
    </aop:aspect>
</aop:config>
```



### 3.4 注解

#### 3.4.1 启用注解功能

首先，使用注解有两种方式：

1. XML导入配置：

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:context="http://www.springframework.org/schema/context"
           xmlns:aop="http://www.springframework.org/schema/aop"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
            https://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            https://www.springframework.org/schema/context/spring-context.xsd
            http://www.springframework.org/schema/aop
            https://www.springframework.org/schema/aop/spring-aop.xsd">

        <!--启用注解-->
        <context:annotation-config/>、
        <!--注解包扫描-->
        <context:component-scan base-package="com.dong.dao"></context:component-scan>
        <context:component-scan base-package="com.dong.log"></context:component-scan>
        <!--启用aop注解-->
        <aop:aspectj-autoproxy proxy-target-class="true"/>
    </beans>
    ```

2. 使用`@EnableAspectJAutoProxy`注解需要被切的类

    ```java
    @Component(value = "userService")
    @EnableAspectJAutoProxy
    public class UserServiceImpl implements UserService {}
    ```
    

其次，使用注解时，需要注意两部分：

- 声明切面：声明某类是切面以及切入后实际要执行的动作（Advice）；
- 定位切点：需要通过execution表达式定位需要切入的方法。

下面介绍一下注解的使用方式。



#### 3.4.2 注解使用

只要对一个类使用@Aspect注解修饰，就表明这是一个切面。可以对其中的方法使用

- @Before
- @After
- @AfterReturning
- @Around
- @AfterThrowing

注解指定切入时机，切入点通过execution表达式指定，举例如下：

```java
@Aspect
@Component
public class CustomPointCut {
    @Before("execution(* com.dong.service.UserServiceImpl.*(..))")
    public void before() {
        System.out.println("aop:方法发生前");
    }
}
```



## 4. 参考资料

1. [ Spring最新5完整教程](https://www.bilibili.com/video/BV1WE411d7Dv)

2. [Spring入门(十)：Spring AOP使用讲解](https://juejin.cn/post/6844903925112373262)

3. [Spring AOP - 注解方式使用介绍](https://juejin.cn/post/6844903987062243341)

