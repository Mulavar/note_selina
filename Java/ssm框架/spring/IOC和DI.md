## IOC和DI

IOC：控制反转，是一种设计思想。即将对象创建的控制权转移给了Spring容器。

假设我们创建了类A，类A中有个成员变量类B。

- 传统开发：我们需要手动创建类B的对象，将他赋给类A的对象，这叫主动控制对象的创建；
- IOC：Spring创建好B对象，然后存储到一个容器里面，当A对象需要使用B对象时，Spring就从存放对象的那个容器里面取出A要使用的那个B对象。

使用传统方式，A对B产生了依赖，也就是A和B之间存在一种耦合关系，而通过IOC的思想，这种依赖关系就被容器取代了，所有对象的创建依赖于容器。

DI：依赖注入，是IOC的一种实现方式。







#### [IOC和DI](https://blog.csdn.net/bestone0213/article/details/47424255)

#### [Spring入门](https://blog.csdn.net/gavin_john/article/details/79517418)



