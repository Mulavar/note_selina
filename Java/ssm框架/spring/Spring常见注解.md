# @Configuration

`Spring 3.0`之后开始支持了一种更优的配置方式：基于Java类的配置方式。通过增加@Configuration注解表明这是一个配置类，其底层为@Component（即该类的bean对象也会被Spring容器接管）。

proxybeanMethods属性作用：指定被@Bean注解修饰的方法是否被代理，true表示该方法被代理（底层cglib，即该方法不能为private、final修饰），每次对该方法进行调用将会直接获得容器中的bean对象，而非真正执行该方法；false表示该方法不被代理，每次调用时都将获得一个新的对象（对应full和lite模式）。



# @Bean

配合@Configuration可以实现XML配置实现注入的功能，效果与

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



# @ConfigurationProperties

其作用是**将Java类属性的值与配置文件进行映射**，即通过@ConfigurationProperties注解，我们可以使用properties或者yml等配置文件对所有属性值进行管理，再映射加载到Java对象中。该注解的prefix属性则指定了配置文件中的唯一标识名。



# @EnableConfigurationProperties

@ConfigurationProperties注解的作用仅仅是将配置文件的值映射载入到Java对象中，但该对象与普通Java对象无区别，并没有被Spring容器进行管理，@EnableConfigurationProperties的作用则是，使得这些被@ConfigurationProperties修饰的类转为bean，在@ConfigurationProperties修饰的类上加@Component也可以使得其被Spring容器管理。



# @Conditional

Spring4.0开始提供了[@Conditional](https://www.cnblogs.com/cxuanBlog/p/10960575.html)注解，提供了if...then...else的语义来管理Bean对象的注册。

使用@Conditional注解，需要与一个名为Condition的接口进行结合才能够发挥效果。Condition接口只有一个方法`matches`：

```java
public interface Condition {
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

这个matches方法里即是对if...then...else语义逻辑的具体代码实现。

@Conditional注解有一个属性value，是一个类数组`Class<? extends Condition>[]`，其逻辑为，当value里所有元素类的macthes方法执行结果均为true时，同意注册Bean对象。

@Conditional可用于修饰类（@Conponent和@Configuration等）或方法（@Bean），并有许多拓展，如

- @ConditionalOnBean：容器中存在指定Bean对象；
- @ConditionalOnMissingBean：容器中不存在指定Bean对象；
- ...



# @Import 

[@Import](https://juejin.cn/post/6844903925242396686) 注解用来整合所有在@Configuration注解中定义的bean配置，作用类似我们将多个XML配置文件导入到单个XML文件中。

### ImportSelector.class

@Import结合ImportSelector接口，可以实现动态选择导入的配置类（@Configuration修饰类）。其使用与@Conditional和Condition接口的组合方式有点相似，如SpringBoot中正是使用`@Import(AutoConfigurationImportSelector.class)`加载了所有自动配置类。ImportSelector接口提供如下方法：

```java
public interface ImportSelector {
	// ...
    
	String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```

该方法selectImports返回的字符串数组，即是@Import注解准备导入的类。



