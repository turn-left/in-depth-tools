## 工具篇：maven实战与原理深入分析

#### 核心功能

- 编译
- 打包
- 测试
- 依赖管理

#### 设计原则

- 约定大于配置

#### 依赖优先原则

- maven依赖管理--最短路径原则
- maven依赖管理--相同路径下配置在前优先

#### 依赖范围(scope)

- **compile**(默认)

编译范围，编译和打包都会依赖。

- **provided**

提供范围，编译时依赖，但不会打包进去。如：servlet-api.jar runtime：运行时范围，打包时依赖，编译不会。如：mysql-connector-java.jar

- **test**
  
测试范围，编译运行测试用例依赖，不会打包进去。如：junit.jar

- **system**

表示由系统中 CLASSPATH 指定。编译时依赖，不会打包进去。配合<systemPath> 一起使用。
