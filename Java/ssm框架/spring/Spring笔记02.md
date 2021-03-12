# 1. Spring常用注解



# 1.1 @Configuration

`Spring 3.0`之后开始支持了一种更优的配置方式：基于Java类的配置方式。通过增加@Configuration注解表明这是一个配置类，其底层为@Component（即该类的bean对象也会被Spring容器接管）。

proxybeanMethods属性作用：指定被@Bean注解修饰的方法是否被代理，true表示该方法被代理（底层cglib，即该方法不能为private、final修饰），每次对该方法进行调用将会直接获得容器中的bean对象，而非真正执行该方法；false表示该方法不被代理，每次调用时都将获得一个新的对象（对应full和lite模式）。



# 1.2 @Bean

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



# 1.3 @ConfigurationProperties

其作用是**将Java类属性的值与配置文件进行映射**，即通过`@ConfigurationProperties`注解，我们可以使用properties或者yml等配置文件对所有属性值进行管理，再映射加载到Java对象中。该注解的prefix属性则指定了配置文件中的唯一标识名。



# 1.4 @EnableConfigurationProperties

`@ConfigurationProperties`注解的作用仅仅是将配置文件的值映射载入到Java对象中，但该对象与普通Java对象无区别，并没有被Spring容器进行管理，`@EnableConfigurationProperties`的作用则是，使得这些被`@ConfigurationProperties`修饰的类转为被Spring容器管理的bean对象，在`@ConfigurationProperties`修饰的类上加`@Component`也可以使得其被Spring容器管理。



# 1.5 @Conditional

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



# 1.6 @Import 

[@Import](https://juejin.cn/post/6844903925242396686) 注解用来整合所有在@Configuration注解中定义的bean配置，作用类似我们将多个XML配置文件导入到单个XML文件中。

### 1.6.1 ImportSelector.class接口

@Import结合ImportSelector接口，可以实现动态选择导入的配置类（@Configuration修饰类）。其使用与@Conditional和Condition接口的组合方式有点相似，如SpringBoot中正是使用`@Import(AutoConfigurationImportSelector.class)`加载了所有自动配置类。ImportSelector接口提供如下方法：

```java
public interface ImportSelector {
	// ...
    
	String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```

该方法selectImports返回的字符串数组，即是@Import注解准备导入的类。



# @EnableAutoConfiguration

该注解被`@Import(AutoConfigurationImportSelector.class)`修饰，指定将要被自动装载到Spring容器中的一系列组件。



# 2. Spring Boot自动装配原理

Spring Boot在启动时会自动装配一系列组件（bean对象），由于其自动加载这些组件，开发者无需做什么配置即可使用Spring Boot提供的功能和服务，且可以在application.properties中覆盖默认配置。其实现的原理主要是通过JavaConfig和几个上述提到的常用注解。

### 2.1 如何进行自动装配

Spring Boot的启动类上有一个组合注解`@SpringBootApplication`：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    // ...
}
```

忽略Java的元注解之后，主要剩下这三个注解：

- `@SpringBootConfiguration`：标注这是一个配置类，本质是一个`@Configuration`；
- `@EnableAutoConfiguration`：**将符合条件的bean自动注入到Spring容器中**；
- `@ComponentScan`：指定需要扫描包的路径。

自动装配的核心主要是通过`@EnableAutoConfiguration`来实现，该注解的源码如下：

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ...
}
```



这里有一个@Import和ImportSelector接口实现类的组合，查看AutoConfigurationImportSelector.class的selectImports方法：

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    // 查看自动装配是否被禁用，即"spring.boot.enableautoconfiguration"的值是否为true
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

在`getAutoConfigurationEntry`方法中，调用了`getCandidateConfigurations`方法，其中使用了`SpringFactoriesLoader`（基于Java SPI机制拓展）读取spring-boot-autoconfigure-xxx.jar包中的META-INF/spring.factories文件内容，得到自动装配类的全限定名并进行加载。

spring.factories文件的内容如下，`SpringFactoriesLoader`将其解析为一个`Map<String, List<String>>`结构，其中key根据组件作用分类，而value则是对应的一系列自动装配类的全限定名。

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
...

# Failure analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.autoconfigure.data.redis.RedisUrlSyntaxFailureAnalyzer,\
...

# Template availability providers
org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerTemplateAvailabilityProvider,\
...
```

由于`@SpringBootApplication`被`@EnableAutoConfiguration`注解修饰，因此这里会加载`EnableAutoConfiguration`对应的所有自动装配类（XXXXAutoConfiguration.class）注入到Spring容器中。



### 2.2 自动装配类的属性如何注入

开发者在application.properties中覆盖默认配置的功能其实就是通过**将配置文件的内容映射到自动装配类上的成员**来实现的，自动装配类（XXXXAutoConfiguration.class）注入属性的过程利用了上述提到的几个注解：`@ConfigurationProperties`和`@EnableConfigurationProperties`。以`KafkaAutoConfiguration.class`为例：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(KafkaTemplate.class)
@EnableConfigurationProperties(KafkaProperties.class)
@Import({ KafkaAnnotationDrivenConfiguration.class, KafkaStreamsAnnotationDrivenConfiguration.class })
public class KafkaAutoConfiguration {
    private final KafkaProperties properties;
    // ...
}
```

该类自身是一个配置类，内部有许多被@Bean注解修饰的方法用于生成被Spring容器管理的组件，其被`@EnableConfigurationProperties`修饰，注入了`KafkaProperties.class`的bean对象，而`KafkaProperties.class`又被`@ConfigurationProperties`修饰，即所有属性最终都由配置文件提供，这也解释了开发者为什么能够在application.properties中覆盖默认配置。

```java
@ConfigurationProperties(prefix = "spring.kafka")
public class KafkaProperties {
}
```