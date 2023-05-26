# Maven 的生命周期

Maven 的生命周期本质是项目构建的生命周期，比如在迭代过程中我们需要清理项目环境、编译、测试、打包及部署。由于工作环境等原因，每个人的构建工作方式不同，有的以手工方式逐行命令执行，有的写了自动化脚本执行，这些方式不可重用，经常换个项目就需要从头再来。

Maven 的生命周期的初衷就是对这些项目的构建过程进行抽象和统一。Maven 总结提供了高度完善的生命周期：清除（clean）、初始化（initialize）、编译（compile）、测试（test）、打包（package）、部署（deploy）等等。这一系列

与直觉不同，Maven 共有三套生命周期，每个生命周期包含了一系列阶段（Phase），**同一个生命周期之内的阶段有顺序依赖关系（调用后面的阶段会自动触发前面的阶段），而三套生命周期之间相互独立：当我们指定其中一个生命周期的某个阶段时，不会触发另一个生命周期的执行**，比如我们执行 clean 生命周期不会触发 default。

-   clean：清除阶段
-   default
-   site

## clean

其中 clean 的生命周期较为简单，仅包含了 pre-clean、clean 和 post-clean，它用于清除项目环境，比如编译好的类、资源等。

## default

default 的生命周期定义了项目真正构建时所需要执行的步骤，其包含的阶段如下（重点的几个阶段进行了标注）：

-   validate：
-   initialize：
-   generate-sources：生成源码阶段，像一些语法分析生成器，对外会提供一个语法描述文件，然后绑定在 generate-sources 阶段来生成具体的源代码（如 Calcite）。
-   process-sources；
-   generate-resources；
-   process-resources；
-   compile：编译项目的主源码；
-   process-classes；
-   generate-test-sources；
-   process-test-sources；

-   generate-test-resources；
-   process-test-resources；
-   test-compile：编译测试项目的主源码；
-   process-test-classes；
-   test；
-   prepare-package；
-   package：将编译好的代码进行打包，最常见是打包成 jar 格式；
-   pre-integration-test；
-   integration-test；
-   post-integration-test；
-   verify；
-   install：将打好的包安装到本地仓库；
-   deploy。

## site

建立和发布项目站点，Maven 能够基于 POM 文件包含的信息，自动生成一个友好的站点。该生命周期包含了如下阶段：

-   pre-site；
-   site：生产项目站点文档；
-   post-site；
-   site-deploy：将生成的项目站点发布到服务器上。

使用命令行执行 Maven 的主要方式是指定调用的 Maven 的生命周期的某个**阶段**，比如 mvn clean，指的并不是执行 clean 这整个生命周期，而是执行 clean 生命周期中的 clean 阶段，结合之前谈到的“同一个生命周期内阶段之间有顺序关系”，因此实际执行的阶段为 pre-clean 和 clean。

再举个例子，平时我们最常用的 mvn clean package 命令，实际是分别指定执行 clean 生命周期的 clean 阶段和 default 生命周期的 package 阶段，因此最终执行的阶段为 clean 生命周期的 pre-clean 和 clean 以及 default 生命周期 validate 到 package 的所有阶段。

需要注意的是，前面提到，Maven 的生命周期其实是一个抽象概念，也就是没有任何实体行为，那我们说的通过命令行指定执行 Maven 的某个阶段是什么意思呢？这里需要引入插件（Plugin）的概念。Maven 借鉴了设计模式中的模板方法（Template Method），定义了一套抽象的生命周期阶段，然后每个阶段的具体动作由插件实现，当我们说运行某个阶段，实际是运行该阶段的所有插件。模板方法模式的优点在于，它既提供了拓展点给子类来控制实际行为，又严格控制了整体的算法流程。

# Maven 的插件机制

Maven 的每个生命周期阶段都可以绑定一个或多个插件，当运行当前生命周期阶段时，Maven 就会执行该生命周期阶段绑定的所有插件。但严格来说，Maven 的生命周期绑定的其实并不是插件，而是插件的目标（Goal）。个人认为目标是一个很好的设定，因为它将插件和 Maven 的生命周期彻底解耦了，每个插件可以有多个目标，每个目标绑定一个生命周期阶段，反过来，每个生命周期阶段也可以绑定多个插件目标，这些插件目标可以属于不同的插件。例如下图：

![Maven 插件和生命周期](https://tva1.sinaimg.cn/large/008i3skNly1gvszrrxieyj30az09y0sz.jpg)

该图显示有两个 Maven 插件，其中 plugin1 有一个目标 goal，绑定了 clean 生命周期的 clean 阶段，plugin2 有两个插件 goal1 和 goal2，分别绑定了 clean 生命周期的 clean 阶段和 default 生命周期的 package 阶段。通过这个图我们可以看出，插件本身和生命周期阶段是完全解耦的，只有当具体目标绑定了该阶段，该阶段才会有执行实体。

当然，Maven 也为大多数生命周期阶段提供了默认插件，如为 compile 阶段绑定了 maven-compile-plugin:compile 目标，针对 test 阶段提供了 maven-surefire-plugin:test 等

我们可以通过 Maven 命令指定运行某个阶段，而运行某个阶段等价于运行该阶段绑定的所有插件目标，所以很自然地，我们应当也可以直接运行某个插件模板，Maven 执行具体插件目标的格式如下：mvn $plugin-prefix:$goal。: 左边是插件前缀而非插件名，具体规律是：一般插件的命名为 maven-xxx-plugin，而这里的插件前缀就是 xxx。比如执行  maven-compile-plugin:compile 时的命令为 mvn compile:compile，而执行 maven-surefire-plugin:test 的命令为 mvn surefire:test。



