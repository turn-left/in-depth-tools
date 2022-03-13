## 工具篇：maven实战与原理深入分析

#### 1. 核心功能

- 编译
- 打包
- 测试
- 依赖管理

#### 2. 一个设计原则

- 约定优于配置（Convention Over Configuration）

#### 3. maven依赖优先原则

- 最短路径原则
- 相同路径下配置在前优先

#### 4. 依赖范围(scope)

- **compile**(默认)

编译范围，编译和打包都会依赖。

- **provided**

提供范围，编译时依赖，但不会打包进去。如：`servlet-api.jar`

- **runtime**

运行时范围，打包时依赖，编译不会。如：`mysql-connector-java.jar`

- **test**

测试范围，编译运行测试用例依赖，不会打包进去。如：`junit.jar`

- **system**

表示由系统中 CLASSPATH 指定。编译时依赖，不会打包进去。配合`<systemPath>` 一起使用。

#### 5. 项目聚合与继承

- **聚合**

将多个模块聚合在一起，统一构建，避免一个一个的构建。聚合需要一个父工程，然后使用`<modules>`标签配置其中子工程相对路径。

```xml
<modules>
    <module>spring-in-action</module>
    <module>spring-source-analysis</module>
    <module>spring-boot-in-action</module>
    <module>spring-boot-source-analysis</module>
</modules>
```

- **继承**

指子工程直接继承父工程中的属性、依赖、插件等配置，避免重复配置。子工程如果进行重写，则以子工程为准！

1. 属性继承

2. 依赖继承

3. 插件继承

#### 6. 项目构建