BeanDefinition 是 Spring Framework 中定义 Bean 的配置元信息接口，包含：

- Bean 的名称和类型信息；
- Bean 行为配置，作用域、auto-wiring 模式、生命周期中一些回调行为；
- 其他 Bean 引用，即依赖关系；
- 配置信息，比如 Bena 的属性。

比较重要的 BeanDefinition 元信息有：

| 属性                     | 说明                                                 |
| ------------------------ | ---------------------------------------------------- |
| Class                    | 类名，必须是具体类                                   |
| Name                     | Bean 的名称（即 id ）                                |
| Scope                    | 作用域（ singleton、prototype 等、默认为 singleton） |
| Constructor arguments    | Bean 构造器参数                                      |
| Properties               | Bean 属性设置                                        |
| Autowiring mode          | 自动绑定模式（byName、byType）                       |
| Laze initialization mode | 延迟初始化模式（实时和延迟）                         |
| Initialization mode      | Bean 初始化回调方法                                  |
| Destruction mode         | Bena 销毁回调方法                                    |

