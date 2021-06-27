好莱坞原则



IoC 容器有两种实现方式

- 依赖注入：Spring 提供了 auto-wiring 方式，帮助用户自动将依赖对象注入到对象中，无需手动 set。

- 依赖查找：依赖在IoC容器中，用户需要显示获取并注入到自己的对象中。

依赖注入和依赖查找的依赖来源不同，依赖来源可以分为三类：

- 自定义 bean
- 内建 bean：如 Environment（StandardEnvironment）对象
- 内建依赖：如 BeanFactory（DefaultListableBeanFactory）对象

依赖注入可以在这三种依赖来源里进行搜索并注入，而依赖查找的依赖来源只有前两部分。

内建的 Bean 是普通的 Spring Bean，包括 BeanDefinitions 和 Singleton Objects，而内建依赖则是通过 AutowireCapableBeanFactory 中的 resolveDependency 方法来注册，这并非是一个 Spring Bean，无法通过依赖查找获取。



BeanFactory 和 ApplicationContext

Spring IoC 底层容器就是指的 BeanFactory 的实现类，大多数情况是 DefaultListableBeanFactory 这个类，它来管理 Spring Beans，而 ApplicationContext 通常为开发人员接触到的 IoC 容器，它是一个 Facade，Wrap 了 BeanFactory 的实现。

当我们通过 ApplicationContext 去查找 Bean 时，本质是使用代理模式，通过它包装的 BeanFactory 进行查找。



Spring IoC 配置元信息有哪些？

- Bean 定义配置
  - XML
  - Properties
  - Java 注解
  - Java APi
- IoC 容器配置：
  - XML
  - Java 注解 
  - Java API
- 外部化属性配置
  - 基于 Java 注解