# 1. Spring 常用注解

## 1.1 自动装配

### 1.1.1 @Configuration

`Spring 3.0`之后开始支持了一种更优的配置方式：**基于 Java 类的配置方式**。通过增加 `@Configuration` 注解表明这是一个配置类，其底层为 `@Component`（**即该类的bean对象也会被Spring容器接管**）。

`proxybeanMethods` 属性作用：**指定被@Bean注解修饰的方法是否被代理**，true表示该方法被代理（底层cglib，即该方法不能为private、final修饰），每次对该方法进行调用将会直接获得容器中的bean对象，而非真正执行该方法；false表示该方法不被代理，每次调用时都将获得一个新的对象（**对应 full 和 lite 模式**）。



### 1.1.2 @Bean

配合 `@Configuration` 可以实现 XML 配置实现注入的功能，效果与

```xml
<beans>
	<bean>...</bean>
    <bean>...</bean>
</beans>
```

类似，在使用 `@Bean` 注解时，表示将该方法返回的对象加载进 Spring 容器，在使用 `@Bean` 修饰方法时需要注意：

1. 方法带返回值，且返回类型为想要被管理的 bean 的类型；
2. 方法名即为默认的 bean name ，可使用 `@Bean` 的 `name` 属性自定义；
3. 方法可带参数，则此时传入的参数也必须是被 Spring 容器管理的 bean 。



### 1.1.3 @Value

该注解的作用是从配置文件中读取属性，如 application.yaml 中有个配置

```yaml
server:
	port: 8000
```

我们使用该注解修饰方法参数 `public void test(@Value("${server.port}") address)`，则该属性 8000 会自动被注入到 address 变量中。

@Value 定位属性有两种方式：

- `${property: default_value}：读取配置文件（如 *.properties, *.yaml 中的值`

- `#{obj.property?: default_value}`：读取容器中管理的 bean 对象的属性值。


假设有一个对象声明如下：

```java
@Component
public class ServerProperty {
    @Value("${server.url}")
    private String url;

    // getter and setter
    // ...
}

@Component
public class Server {
	@Value("#{serverProperty.url}")
	private String url;
}
```

第一个 `@Value` 指代从配置文件中读取属性注入到名为 serverProperty 的 bean 对象中，而第二个 @Value 则是从名为 serverProperty 的 bean 对象中读取数据并注入到 server 中。

注：@Value 想设置 null 默认值给 String 类型的变量时需如此 `@Value("${server.url: #{null}}")`。



### 1.1.4 @ConfigurationProperties

其作用是**将 Java 类属性的值与配置文件进行映射**，即通过`@ConfigurationProperties`注解，我们可以使用 properties 或者 yaml 等配置文件对所有属性值进行管理，再映射加载到 Java 对象中。该注解的 prefix 属性则指定了配置文件中的唯一标识名。



### 1.1.5 @EnableConfigurationProperties

`@ConfigurationProperties`注解的作用仅仅是将配置文件的值映射载入到 Java 对象中，但该对象与普通 Java 对象无区别，并没有被 Spring 容器进行管理，`@EnableConfigurationProperties`的作用则是，使得这些被`@ConfigurationProperties`修饰的类转为被 Spring 容器管理的bean对象，在`@ConfigurationProperties`修饰的类上加`@Component`也可以使得其被 Spring 容器管理。



### 1.1,6 @Conditional

Spring4.0开始提供了 `@Conditional` 注解，提供了 `if...then...else` 的语义来管理 Bean 对象的注册。

使用 `@Conditional` 注解，需要与一个名为 `Condition` 的接口进行结合才能够发挥效果。`Condition` 接口只有一个方法`matches`：

```java
public interface Condition {
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

这个matches方法里即是对if...then...else语义逻辑的具体代码实现。

@Conditional注解有一个属性value，是一个类数组`Class<? extends Condition>[]`，其逻辑为，当value里所有元素类的macthes方法执行结果均为true时，同意注册Bean对象。

@Conditional可用于修饰类（@Conponent和@Configuration等）或方法（@Bean），并有许多拓展，如

- @ConditionalOnBean：容器中存在指定 Bean 对象；
- @ConditionalOnMissingBean：容器中不存在指定 Bean 对象；
- ...



### 1.1.7 @Import 

该注解用来整合所有在 @Configuration 注解中定义的 bean 配置，作用类似我们将多个 XML 配置文件导入到单个 XML 文件中。

`@Import` 结合 `ImportSelector` 接口，可以实现动态选择导入的配置类（ `@Configuration` 修饰类）。其使用与 `@Conditional` 和 `Condition` 接口的组合方式有点相似，如 SpringBoot 中正是使用 `@Import(AutoConfigurationImportSelector.class)` 加载了所有自动配置类。`ImportSelector` 接口提供如下方法：

```java
public interface ImportSelector {
	// ...
    
	String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```

该方法 `selectImports` 返回的字符串数组，即是 `@Import` 注解准备导入的类。



### 1.1.8 @EnableAutoConfiguration

该注解被`@Import(AutoConfigurationImportSelector.class)`修饰，指定将要被自动装载到Spring容器中的一系列组件。



## 1.2 Web

### 1.2.1 @RequestMapping

将 HTTP 请求映射到该注解修饰的处理方法上，衍生出的注解有：

- @GetMapping：本质是 @RequestMapping 指定了 GET 方法，接收 HTTP 的 GET 请求；
- @PostMapping：本质是 @RequestMapping 指定了 POST 方法，接收 HTTP 的 POST 请求；
- @PutMapping：本质是 @RequestMapping 指定了 PUT 方法，接收 HTTP 的 PUT 请求；
- @DeleteMapping：本质是 @RequestMapping 指定了 DELETE 方法，接收 HTTP 的 DELETE 请求；
- ...



### 1.2.2 @RequestBody

该注解用于读取 Http 请求的请求体数据（当 `content-type` 为 `application/json` 时），然后将这部分数据绑定到该注解修饰的方法参数上，在同一个方法里最多只有一个参数能够被 `@RequestBody` 修饰。

注：无法配合 `GET` 请求使用。



### 1.2.3 @ResponseBody

该注解可修饰方法和其他注解，表示将方法返回的对象转换成一定格式，并写入响应报文的响应体中，一般用于返回 `Json` 数据。



### 1.2.4 RestController

该注解一般用于修饰类，本质是 @Controller + @ResponseBody ，因此聚合了 @Controller 的路由功能和 @ResponseBody 的功能—类上的所有方法都会返回 Json 格式数据。



### 1.2.5 @RequestParam

该注解用于接收 `Http` 请求的查询参数，来源可分为两部分：

- `GET` 方法的 `URL`；
- `POST` 方法（当 `content-type` 是 `application/x-www-form-urlencoded` 时）的 `form data`。



### 1.2.6 @PathVariable

该注解用于解析 URL 模板，假如我们设计 REST 风格的 URL 如下：

```
@RequestMapping("xxx.com/items/{id}")
public String getItemById(@PathVariable("id") String id) {}
```

当访问 "xxx.com/items/100" 时，方法 `getItemById` 的方法参数可以直接解析得到 "100"。

# 2. Spring Boot 自动装配原理

Spring Boot 在启动时会自动装配一系列组件（ bean 对象），由于其自动加载这些组件，开发者无需做什么配置即可使用 Spring Boot 提供的功能和服务，且可以在 application.properties 中覆盖默认配置。其实现的原理主要是通过 `JavaConfig` 和几个上述提到的常用注解。

## 2.1 如何进行自动装配

Spring Boot 的启动类上有一个组合注解 `@SpringBootApplication`：

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
- `@EnableAutoConfiguration`：**将符合条件的 bean 自动注入到 Spring 容器中**；
- `@ComponentScan`：指定需要扫描包的路径。

自动装配的核心主要是通过`@EnableAutoConfiguration`来实现，该注解的源码如下：

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ...
}
```



这里有一个 `@Import` 和 `ImportSelector` 接口实现类的组合，查看 `AutoConfigurationImportSelector.class` 的 `selectImports` 方法：

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

在`getAutoConfigurationEntry`方法中，调用了`getCandidateConfigurations`方法，其中使用了`SpringFactoriesLoader`（基于**Java SPI** 机制拓展）读取 spring-boot-autoconfigure-xxx.jar 包中的 META-INF/spring.factories 文件内容，得到自动装配类的全限定名并进行加载。

spring.factories文件的内容如下，`SpringFactoriesLoader`将其解析为一个`Map<String, List<String>>`结构，其中 key 根据组件作用分类，而 value 则是对应的一系列自动装配类的全限定名。

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

由于`@SpringBootApplication`被`@EnableAutoConfiguration`注解修饰，因此这里会加载`EnableAutoConfiguration`对应的所有自动装配类（`XXXXAutoConfiguration.class`）注入到 Spring 容器中。



## 2.2 自动装配类的属性如何注入

开发者在 application.properties 中覆盖默认配置的功能其实就是通过**将配置文件的内容映射到自动装配类上的成员**来实现的，自动装配类（`XXXXAutoConfiguration.class`）注入属性的过程利用了上述提到的几个注解：`@ConfigurationProperties` 和 `@EnableConfigurationProperties`。以 `KafkaAutoConfiguration.class` 为例：

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

该类自身是一个配置类，内部有许多被 `@Bean` 注解修饰的方法用于生成被Spring容器管理的组件，其被 `@EnableConfigurationProperties` 修饰，注入了 `KafkaProperties.class` 的 bean 对象，而 `KafkaProperties.class` 又被 `@ConfigurationProperties `修饰，即所有属性最终都由配置文件提供，这也解释了开发者为什么能够在 application.properties 中覆盖默认配置。

```java
@ConfigurationProperties(prefix = "spring.kafka")
public class KafkaProperties {
    // ...
}
```



# 3. 参考资料

1. [一文了解 @Conditional 注解说明和使用](https://www.cnblogs.com/cxuanBlog/p/10960575.html)
2. [@Import、ImportSelector注解使用及源码分析](https://juejin.cn/post/6844903925242396686)
3. [@ResponseBody与@RestController的作用与区别](https://blog.csdn.net/alan_liuyue/article/details/53837127)
4. [springboot(四)——@EnableConfigurationProperties是如何起作用的你知道吗](https://segmentfault.com/a/1190000018987185)

