## 1. 整体架构

### 1.1. 全局架构图

![image-20221208163842364](image-20221208163842364.png) 

* modeler

  模型设计器，独立工具

* File Repository

  存放模型的仓库

* Engine

  核心引擎，将模型文件解释成对象数据，并提供相关管理；

* api 

  提供java API,与rest api进行操作

* DataBase

  数据存储，支持常用的关系型数据库

* Tasklist

  任务列表，可以从管理界面查看

* Cockpit

  流程控制台，可以从管理界面操作

* Admin

  租户，用户，用户组，权限等管理操作，可以从管理界面操作

* Job executor

  处理定时任务与异步任务相关

### 1.2. 流程引擎架构

![image-20221208164313083](image-20221208164313083.png) 

* **database**

  camunda7目前支持大多 数关系型数据库，camunda8云原生版本支持elasticsearch分布式存储，

* **public Api**

  提供JAVA API对部署，任务，流程实例等一系列操作；

* **Job Executor**

  提供对定时任务与异步任务的操作；

* **persistence** 

  数据持久化，采用mybatis框架；

* **BPMN2核心引擎：**

  加载BPMN流程



## 2. 运行态机制

### 2.2. 变量作用域

## 3. api定义

流程引擎（ProcessEngine）是整个camunda工作流的大心脏，初始化好流程引擎，就可以通过它获取各种API的操作。提供的所有API都是线程安全的

![image-20221208162515283](image-20221208162515283.png) 

### 3.1. java-api

### 3.2. rest-api

RestApi是指restful [api](https://so.csdn.net/so/search?q=api&spm=1001.2101.3001.7020)的简写，restful是一种接口设计风格，一句话就是对接口的定义提出一组标准.

* 每个URL代表一种资源，独一无二；
* 无状态，访问不同的实例结果一致；
* 遵守GET POST PUT PATCH DELETE对应相应的操作；
* 支持路径参数；
* 不用代码开发，直接就可以通过URL获取到数据；

设计完善的RestApi 最大的好处就是与语言无关。

camunda提供了非常完整的API，文档也非常齐全。
参考文档：https://docs.camunda.org/manual/7.17/reference/rest/

如果camunda提供的RestApi不满足业务要求，还可以通过使用springboot引擎端借助流程引擎API自己提供额外的rest api。
如果流程引擎API也不能满足要求，还可以通过直接通过SQL查询提供额外的Rest Api。

#### 3.2.1. RestApi调用

在实际开发中使用调用RestApi时，需要将入参request，返回response的json数据转换成对象，这些对象可能非常繁琐，如果是java客户端，可以使用以下方法。
https://camunda-community-hub.github.io/camunda-platform-7-rest-client-spring-boot/7.17.1/getting-started.html

#### 3.2.2. postman导入

#### 3.2.3. 外部任务

外部任务是camunda非常重要的特性，它是基于restapi实现的。

![image-20221212195913864](image-20221212195913864.png) 

外部任务的执行大体分三步：

* **Process Engine**: Creation of an external task instance
* **External Worker**: Fetch and lock external tasks（根据TOPIC订阅拉取）
* **External Worker & Process Engine**: Complete external task instance

当第3步当处理业务失败时，还可以上报异常给引擎，可以从控制台可视化准确定位到流程实例失败的原因，当然，上报异常时，在异常没消失前会永远卡在失败节点，重试成功后异常会消失，流程往下走

![image-20221212200106537](image-20221212200106537.png) 

**长轮询（Long Polling ）机制**

官网参考：https://docs.camunda.org/manual/7.17/user-guide/process-engine/external-tasks/
由于外部任务是通过rest api的原理实现的，并且由客户端主动拉取任务，延迟较小
如果采用普通的HTTP请求定时轮询拉取任务，摘取时间间隔慢了确实会有延迟，轮询太频繁资源开销确实很大，在高并发项目中这没法接受。

因此客户端都设计为长轮询的模式拉取任务，如果没有外部任务可用，请求会被服务器挂起并加锁，防止一个任务被 其他客户端重复消费，一旦出现新的外部任务，就重新激活请求并执行响应。请求挂起的时间可以通过订阅参数配置超时时间，超时后释放锁且不再挂起

![image-20221212200856450](image-20221212200856450.png) 



## 4. 数据模型

![image-20221208164628511](image-20221208164628511.png) 

### 4.1. 表名规则

camunda7.17.0共有49张表，都以ACT_开头，共分为五大类：

* ACT_GE_*

  GE为general缩写，表示通用数据，用于不同场景，有3张表；

- ACT_HI_*:

  HI为history缩写，表示历史操作数据相关，比如流程实例，流程变量，任务等数据，数据存储级别可以设置，共18张表；

- ACT_ID_*

   'ID’为identity缩写，表示组织用户信息，比如用户，组，租户等，共6张表；

- ACT_RE_*

  'RE’为repository缩写，表示流程资源存储，这个前缀的表包含了流程定义和流程静态资源（图片，规则等），共6张表；

- ACT_RU_*

  'RU’为runtime缩写，表示流程运行时。 这些运行时的表，包含流程实例，任务，变量，Job等运行中的数据。 Camunda只在流程实例执行过程中保存这些数据，在流程结束时就会删除这些记录， 这样运行时表的数据量最小，可以最快运行。共16张表

### 4.2. 数据表

| **分类**           | **表名**                | **职责**                                             | **是否永久保存**   |
| ------------------ | ----------------------- | ---------------------------------------------------- | ------------------ |
| 流程通用数据       | act_ge_bytearray        | 流程引擎二进制数据表                                 | 是                 |
|                    | act_ge_property         | 流程引擎属性配置表                                   | 是                 |
|                    | act_ge_schema_log       | 数据库脚本执行日志表                                 | 是                 |
| 流程历史记录       | act_hi_actinst          | 历史的活动实例表                                     | 根据配置的级别决定 |
|                    | act_hi_attachment       | 历史的流程附件表                                     |                    |
|                    | act_hi_batch            | 历史的批处理记录表                                   |                    |
|                    | act_hi_caseactinst      | 历史的CMMN活动实例表                                 |                    |
|                    | act_hi_caseinst         | 历史的CMMN实例表                                     |                    |
|                    | act_hi_comment          | 历史的流程审批意见表                                 |                    |
|                    | act_hi_dec_in           | 历史的DMN变量输入表                                  |                    |
|                    | act_hi_dec_out          | 历史的DMN变量输出表                                  |                    |
|                    | act_hi_decinst          | 历史的DMN实例表                                      |                    |
|                    | act_hi_detail           | 历史的流程运行时变量详情记录表                       |                    |
|                    | act_hi_ext_task_log     | 历史的外部任务执行记录                               |                    |
|                    | act_hi_identitylink     | 历史的流程运行过程中任务关联的用户表，主要是usertask |                    |
|                    | act_hi_incident         | 历史的流程异常事件记录表                             |                    |
|                    | act_hi_job_log          | 历史的流程作业记录表                                 |                    |
|                    | act_hi_op_log           | 历史的操作记录，主要是流程定义的变更，创建等         |                    |
|                    | act_hi_procinst         | 历史的流程实例                                       |                    |
|                    | act_hi_taskinst         | 历史的任务实例                                       |                    |
|                    | act_hi_varinst          | 历史的流程变量记录表                                 |                    |
| 用户，组，租户数据 | act_id_group            | 用户组信息表                                         | 是                 |
|                    | act_id_info             | 用户扩展信息表                                       |                    |
|                    | act_id_membership       | 用户与用户组的关系表                                 |                    |
|                    | act_id_tenant           | 租户信息表                                           |                    |
|                    | act_id_tenant_member    | 租户成员表                                           |                    |
|                    | act_id_user             | 用户信息表                                           |                    |
| 流程资源存储       | act_re_camformdef       | camunda表单定义表                                    | 是                 |
|                    | act_re_case_def         | CMMN案例管理模型定义表                               |                    |
|                    | act_re_decision_def     | DMN决策模型定义表                                    |                    |
|                    | act_re_decision_req_def | DMN决策请求定义表                                    |                    |
|                    | act_re_deployment       | 流程部署表                                           |                    |
|                    | act_re_procdef          | BPMN流程模型定义表                                   |                    |
| 流程运行时         | act_ru_authorization    | 流程运行时授权表                                     | 是                 |
|                    | act_ru_batch            | 流程执行批处理表                                     | 否                 |
|                    | act_ru_case_execution   | CMMN案例运行执行表                                   | 否                 |
|                    | act_ru_case_sentry_part | 运行时sentry日志监控表                               | 否                 |
|                    | act_ru_event_subscr     | 流程事件订阅表                                       | 是                 |
|                    | act_ru_execution        | BPMN流程运行时记录表                                 | 否                 |
|                    | act_ru_ext_task         | 外部任务执行表                                       | 否                 |
|                    | act_ru_filter           | 流程定义查询配置表                                   | 是                 |
|                    | act_ru_identitylink     | 运行时流程人员表                                     | 否                 |
|                    | act_ru_incident         | 运行时异常事件表                                     | 否                 |
|                    | act_ru_job              | 流程运行时作业表                                     | 否                 |
|                    | act_ru_jobdef           | 流程作业定义表                                       | 是                 |
|                    | act_ru_meter_log        | 流程运行时度量日志表                                 | 是                 |
|                    | act_ru_task             | 流程运行时任务表                                     | 否                 |
|                    | act_ru_variable         | 流程运行时变量表                                     | 否                 |
|                    | act_ru_task_meter_log   | 流程运行时任务度量表                                 | 是                 |

## 5. 流程事务

## 6. 高可用方案

实际生产环境中流程引擎一般处理整个业务流程的核心地位，引擎服务一量不能正常使用，整个系统将会不能正常提供服务。为此，引擎高可用方案在生产环境是必不可少的一个环节，特别是SAAS模式。嵌入使用模式，引擎生命周期与业务系统一致。

![image-20221208171108404](image-20221208171108404.png) 



## 7. 常见配置

### 7.1. 历史数据配置

camunda工作流的数据都存储在数据库中，历史数据会非常大，可以根据需要，选择存储历史数据的级别，camunda支持的级别如下：

* **full级别**

  所有历史数据都被保存，包括变量的更新。

* audit级别

  只有历史的流程实例，活动实例，表单数据等会被保存。

* auto级别

   数据之前配置的是什么级别就是用什么级别，如果没配置则是audit级别。

* none级别

  不存储历史数据。

springboot配置中可以加入:

```
camunda.bpm.history-level=audit
```

**生产环境中建议使用audit级别，开发测试环境可以使用full级别，方便调试查找问题。**

### 7.2. 自动部署

流程引擎中默认会自动对resources/bpmn目录下的bpmn进行部署。如果采用rest api或者model直接部署，可以添加配置进行关闭。
引擎端同时支持多个项目在使用时，不可能因为某个项目部署流程重启引擎，一般会通过rest api进行部署。
springboot配置中可以加入:

```
camunda.bpm.auto-deployment-enabled=false
###自动部署resources下面的bpmn文件
```

### 7.3. 接口鉴权 

https://docs.camunda.org/manual/7.17/reference/rest/overview/authentication/
前面所有的示例中外部任务订阅是通过rest api的基础上实现的，都没有带任何身份校验直接可以访问，数据案例存在很大问题，生产环境中不可接受。

#### 7.3.1. 添加权限过滤器

这个配置方法官网只给出了web.xml中配置方法，以下是工作中springboot中配置使用过的方法。
com.forestlake.camunda.AuthFilterConfig

```
@Configuration
public class AuthFilterConfig implements ServletContextInitializer {
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        FilterRegistration.Dynamic authFilter = servletContext.addFilter("camunda-auth", ProcessEngineAuthenticationFilter.class);
        authFilter.setAsyncSupported(true);
        authFilter.setInitParameter("authentication-provider","org.camunda.bpm.engine.rest.security.auth.impl.HttpBasicAuthenticationProvider");
        authFilter.addMappingForUrlPatterns(null,true,"/engine-rest/*");
    }
}
```

#### 7.3.2. 配置账号

控制台添加用户javaclient，密码123456，并加入一个组中获取组权限

![image-20221212212855555](image-20221212212855555.png) 



#### 7.3.3. 客户端配置

```
camunda.bpm.client.basic-auth.username=javaclient
camunda.bpm.client.basic-auth.password=123456
```

## 8. 多租户设计

