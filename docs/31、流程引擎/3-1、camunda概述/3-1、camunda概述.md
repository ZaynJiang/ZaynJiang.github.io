## 1. 流程引擎概述

### 1.1. 什么是工作流

工作流起源于办公自动化，是对软件中业务流程的抽象。通过工作流将各个任务有序组织起来形成一个完整的业务闭环。   比如：请假审批流程，网上购物流程，银行贷款流程；

![image-20221206143636084](image-20221206143636084.png) 

### 1.2. 应用场景

- 传统办公自动化业务；
- 微服务业务组装器；
- 各种能力层（如图像识别，离线分析，指纹识别，鉴黄鉴暴）组合业务；
- 低代码平台，标准化输入输出，功能独立化的业务组件，可以通过拖拽改变流程组成新的业务；
- 微服务状态机；

### 1.3. 常见工作流

#### 1.3.1. 工作流对比

最常用业务的工作流组件包括jbpm 、Activity、flowable 、camunda

功能最强大的属flowable   camunda  ，都从activity发展而来，性能，功能都比activity强大。

![image-20221206143742063](image-20221206143742063.png) 

PS：流程虚拟机 （Process Virtual Machine - PVM）

* Activiti

  Activiti 由 Alfresco 公司开发，目前最高版本为 Activiti cloud 7.1.0。其中 activiti5 和 [activiti6](https://www.zhihu.com/search?q=activiti6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2279723965}) 的核心 leader 是 Tijs Rademakers，由于团队内部分歧，2017 年 Tijs Rademakers 离开团队，创建了后来的 Flowable。activiti6 以及 activiti5 代码则交接给 Salaboy 团队维护，activiti6 以及 activiti5 的代码官方已经暂停维护。往后 Salaboy 团队开发了 activiti7 框架，[activiti7](https://www.zhihu.com/search?q=activiti7&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2279723965}) 内核使用的还是 activiti6，并没有为引擎注入更多的新特性，只是在 Activiti 之外的上层封装了一些应用。直到 Activiti cloud 7.1.0 版本，Activiti cloud 将系统拆分为 Runtime Bundle、Audit Service、Query Service、Cloud Connectors、Application Service、Notification Service。这些工作的主要目的其实就是为了上云，减少对 Activiti 依赖的耦合，需要使用 Activiti 的系统只需要通过调用 http 接口的方式来实现工作流能力的整合，将工作流业务托管上云。

* Flowable

  Flowable 是基于 activiti6 衍生出来的版本，目前最新版本是 v6.7.0。开发团队是从 Activiti 中分裂出来的，修复了一众 activiti6 的 bug，并在其基础上实现了 DMN 支持，BPEL 支持等。相对开源版，其商业版的功能会更强大。Flowable 是一个使用 Java 编写的轻量级业务流程引擎，使用 Apache V2 license 协议开源。2016 年 10 月，Activiti 工作流引擎的主要开发者离开 Alfresco 公司并在 Activiti 分支基础上开启了 Flowable 开源项目。Flowable 项目中包括 BPMN（Business Process Model and Notation）引擎、CMMN（Case Management Model and Notation）引擎、DMN（Decision Model and Notation）引擎和表单引擎（Form Engine）等模块。

* Camunda

  Camunda 基于 activiti5，所以其保留了 PVM，最新版本 Camunda7.17，开发团队也是从 activiti 中分裂出来的，发展轨迹与 Flowable 相似。通过压力测试验证 Camunda BPMN 引擎性能和稳定性更好。功能比较完善，除了 BPMN，Camunda 还支持 CMMN（案例管理）和 DMN（决策自动化）。Camunda 不仅带有引擎，还带有非常强大的工具]，用于建模、任务管理、操作监控和用户管理。

* jBPM

  jBPM 由 JBoss 公司开发，目前最高版本 7.61.0.Final，不过从  jBPM5 开始已经跟之前不是同一个产品了，jBPM5 的代码基础不是 jBPM4，而是从 Drools Flow 重新开始，基于 Drools Flow 技术在国内市场上用的很少，jBPM4 诞生的比较早，后来 jBPM4 创建者 Tom Baeyens 离开 JBoss 后，加入 Alfresco 后很快推出了新的基于 jBPM4 的开源工作流系统 Activiti，另外 jBPM 以 Hibernate 作为数据持久化 ORM，而 Hibernate 也已不是主流技术

* osworkflow

  osworkflow 是一个轻量化的流程引擎，基于状态机机制，数据库表很少，osworkflow 提供的工作流构成元素有：步骤（step）、条件（conditions）、循环（loops）、分支（spilts）、合并（joins）等，但不支持会签、跳转、退回、加签等这些操作，需要自己扩展开发，有一定难度。如果流程比较简单，osworkflow 是很好的选择

#### 1.3.2. 流程编辑器

* bpmn-js

  bpmn-js 是 BPMN 2.0 渲染工具包和 Web 模型。bpmn-js 正在努力成为 Camunda BPM的一部分。bpmn-js 使用 Web 建模工具可以很方便的构建 BPMN 图表，可以把 BPMN 图表嵌入到你的项目中，容易扩展。bpmn-js 是基于原生 js 开发，支持集成到 vue、react 等开源框架中。

* mxGraph

  mxGraph 是一个强大的 JavaScript 流程图前端库，可以快速创建交互式图表和图表应用程序，国内外著名的 ProcessOn和 都是使用该库创建的强大的在线流程图绘制网站。由于 mxGraph 是一个开放的 js 绘图开发框架，可以开发出很炫的样式，或者完全按照项目需求定制。

* activiti-modeler

  Activiti 开源版本中带了 web 版流程设计器，在 Activiti-explorer 项目中有 activiti-modeler，优点是集成简单，开发工作量小，缺点是界面不美观，用户体验差。

* flowable-modeler

  Flowable 开源版本中带了 web 版流程设计器，展示风格和功能基本跟 activiti-modeler 一样，优点是集成简单，开发工作量小，缺点是界面不美观，用户体验差。

* react-flow

  react-flow 是一个用于构建基于节点的应用程序的库。这些可以是简单的静态图或复杂的基于节点的编辑器。同时 react-flow 支持自定义节点类型和边线类型，并且它附带一些组件，可以查看缩略图的 Mini Map 和悬浮控制器 Controls。基于 react 生态开发，图形高度自定义，任何 ReactElement 都可以作为节点，但是不支持 bpmn 图表。

## 2. camunda概述

camunda工作流源自activity5，是德国一家工作流程自动化软件开发商提供的，现在同时提供camunda7（组件方式）与camunda8(基于云原生)两种平台

### 2.1. 协议支持：

- BPMN2（Business Process Model and Notation业务流程模型标记）
- CMMN ( Case Management Model and Notation.案例管理模型标记)
- DMN（Decision Model and Notation决策模型标记）

### 2.2. 优势

- 高性能（乐观锁，缓存机制）
- 高扩展性
- 高稳定性
- **独有的外部任务模式**
- 完善rest api
- 支持多租户
- 优秀的流程设计器

### 2.3. camunda现状

一年发形两个版本
五年内保证，camunda7 与camunda8并行开发与维护。
https://camunda.com/about/customers/ 官网在册企业

![image-20221206145646127](image-20221206145646127.png) 

### 2.4. 文档资料

官方文档：https://docs.camunda.org/manual/7.17/
论坛：https://forum.camunda.io/ 有问题可以在论坛提问或搜索，会有人回答
github社区：https://github.com/camunda-community-hub
github官方开源库：https://github.com/camunda
国内开源社区：https://www.oschina.net/informat/camunda

## 3. 安装与部署

camunda有多种使用方式，此处先使用独立平台的方式部署引擎，这种用法适合分布式，微服务化，多语言异构系统的使用方式。是官方推荐的使用方式。
camunda-platform-run支持的运行方式有多种，包括：

- Apache tomcat
- JBoss EAP/wildfly
- IBM WebShpere
- Oracle WebLogic
- Docker
- Spring boot启动

### 3.1. springboot安装

参考文档：https://docs.camunda.org/manual/latest/user-guide/spring-boot-integration/
**前提条件：**
**1.安装jdk1.8以上，camunda是java语言开发的，需要有jdk环境才能运行。（jdk安装与环境变量配置自己百度 ）**
参考：https://blog.csdn.net/bestsongs/article/details/104905060
**2.安装maven3.6以上，springboot项目默认使用maven进行构建，需要maven环境。（maven安装与环境配置自行百度）**
参考：https://www.cnblogs.com/lanrenka/p/15903261.html



#### 3.1.1. 新建springboot工程

利用camnunda提供的工具快速创建springboot工程，

https://camunda.com/download/

![image-20221206163826479](image-20221206163826479.png) 

点击绿色链接，跳转到camunda springboot工程初始化页面，将各项选项改成自己想要。和原生spring boot工程初始化界面类似。这里设置管理员帐号admin, 密码123456。点击GENERATE PROJECT按钮下载工程代码并解压

#### 3.1.2. maven依赖

配置界面默认是H2内存数据库，这里我需要改成mysql数据库，并添加mysql驱动与连接信息。
用ideal打开工程，没装ideal用任何文本编辑器打开都行。

打开pom.xml，去掉h2驱动，添加mysql5驱动

```
 <dependencyManagement>
    <dependencies>
<!--      <dependency>-->
<!--        <groupId>org.springframework.boot</groupId>-->
<!--        <artifactId>spring-boot-dependencies</artifactId>-->
<!--        <version>2.6.4</version>-->
<!--        <type>pom</type>-->
<!--        <scope>import</scope>-->
<!--      </dependency>-->

      <dependency>
        <groupId>org.camunda.bpm</groupId>
        <artifactId>camunda-bom</artifactId>
        <version>7.17.0</version>
        <scope>import</scope>
        <type>pom</type>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.camunda.bpm.springboot</groupId>
      <artifactId>camunda-bpm-spring-boot-starter-rest</artifactId>
    </dependency>

    <dependency>
      <groupId>org.camunda.bpm.springboot</groupId>
      <artifactId>camunda-bpm-spring-boot-starter-webapp</artifactId>
    </dependency>

    <dependency>
      <groupId>org.camunda.bpm</groupId>
      <artifactId>camunda-engine-plugin-spin</artifactId>
    </dependency>

    <dependency>
      <groupId>org.camunda.spin</groupId>
      <artifactId>camunda-spin-dataformat-all</artifactId>
    </dependency>

    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.47</version>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.codehaus.groovy/groovy-all -->
    <dependency>
      <groupId>org.codehaus.groovy</groupId>
      <artifactId>groovy-all</artifactId>
      <version>3.0.10</version>
      <type>pom</type>
    </dependency>

    <!--groovy发送Http请求，底层是对HTTPClient封装 HttpBuilder-->
    <dependency>
      <groupId>org.codehaus.groovy.modules.http-builder</groupId>
      <artifactId>http-builder</artifactId>
      <version>0.7.1</version>
    </dependency>
    <dependency>
      <groupId>org.camunda.bpm</groupId>
      <artifactId>camunda-engine-plugin-connect</artifactId>
      <version>7.17.0</version>
    </dependency>

    <dependency>
      <groupId>org.camunda.connect</groupId>
      <artifactId>camunda-connect-connectors-all</artifactId>
      <version>1.5.2</version>
    </dependency>

  </dependencies>
```



#### 3.1.3. 配置

```
spring:
  datasource:
    url: jdbc:mysql://10.130.36.244:3306/camunda?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&useSSL=false&nullCatalogMeansCurrent=true
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: tech789
camunda.bpm.admin-user:
  id: admin
  password: 123456
camunda:
  bpm:
    auto-deployment-enabled: true
    history-level: audit
    authorization:
      tenant-check-enabled: true
    default-number-of-retries: 4
```

#### 3.1.4. 打包部署

```
 mvn clean package -Dmaven.test.skip=true
```

```
 java -jar camunda-engine-1.0.0-SNAPSHOT.jar
```

#### 3.1.5.  页面验证

 localhost:8080

![image-20221206164034898](image-20221206164034898.png) 



### 3.2. docker安装

### 3.3.  tomcat安装

下载war包

https://docs.camunda.org/manual/7.17/installation/standalone-webapplication/

### 3.4. 设计器

地址：

https://github.com/camunda/camunda-modeler/releases

https://camunda.com/download/modeler/

#### 3.4.1. 下载

根据自己操作系统选择下载，我这里选择windows 64位 v5.2.0版本

#### 3.4.2.  解压安装

#### 3.4.3.  创建流程

使用快捷方式ctrl+p 可以切换右侧属性面板，ctrl+ shift+p 重置属性面板

![image-20221206170532302](image-20221206170532302.png) 

#### 3.4.4. 部署流程

将设计好的流程加载到流程引擎中。

![image-20221206170521228](image-20221206170521228.png)

#### 3.4.5. 启动实例

流程部署到引擎中之后，可以通过设计器直接启动流程实例。
流程与流程实例的关系可以参数面向对象语言的类与对象之前的关系，或者模具与模具生产出来的实体。

![image-20221206170550661](image-20221206170550661.png) 