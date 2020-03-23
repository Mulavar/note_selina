Travis CI 只支持 Github，首先需要用github账号登录官方网站 [travis-ci.org](https://travis-ci.org/)，激活需要CI的github仓库并开始监听。

所有运行项目会经过两个流程：

+ install 阶段：安装依赖
+ script 阶段：运行脚本

### install

`install`字段用来指定安装脚本。

```
# 单条command
install: ./install-dependencies.sh

# 如果command1失败了，整个流程都会停止
install:
  - command1
  - command2
  
# 跳过install阶段
install: true
```



### script

`script`字段用来指定构建或测试脚本。

注意，`script`与`install`不一样，如果`command1`失败，`command2`会继续执行。但是，整个构建阶段的状态是失败。

```
script: bundle exec thor build
script:
  - command1
  - command2
  
# 如果command2只有在command1成功后才能执行，就要写成下面这样。
script: command1 && command2
```



### 钩子方法

Travis提供了7个钩子方法

> - before_install：install 阶段之前执行
> - before_script：script 阶段之前执行
> - after_failure：script 阶段失败时执行
> - after_success：script 阶段成功时执行
> - before_deploy：deploy 步骤之前执行
> - after_deploy：deploy 步骤之后执行
> - after_script：script 阶段之后执行

完整的生命周期，从开始到结束是下面的流程。

> 1. before_install
> 2. install
> 3. before_script
> 4. script
> 5. after*success or after*failure
> 6. [OPTIONAL] before_deploy
> 7. [OPTIONAL] deploy
> 8. [OPTIONAL] after_deploy
> 9. after_script



### 运行状态

Travis 每次运行，可能会返回四种状态。

> - passed：运行成功，所有步骤的退出码都是`0`
> - canceled：用户取消执行
> - errored：`before_install`、`install`、`before_script`有非零退出码，运行会立即停止
> - failed ：`script`有非零状态码 ，会继续运行



### 使用技巧

`.travis.yml`的`env`字段可以定义环境变量。

```
env:
  - DB=postgres
  - SH=bash
  - PACKAGE_VERSION="1.0.*"
```

