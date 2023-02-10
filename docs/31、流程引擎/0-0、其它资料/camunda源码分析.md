## 启动过程

* camunda-bpm-spring-boot-starter-webapp-core

  springbootstart自动配置

  org.camunda.bpm.spring.boot.starter.webapp.CamundaBpmWebappInitializer



* org.camunda.bpm.engine.impl.cfg.ProcessEngineConfigurationImpl

  相关的配置类

  ```
    protected void init() {
      invokePreInit();
      initDefaultCharset();
      initHistoryLevel();
      initHistoryEventProducer();
      initCmmnHistoryEventProducer();
      initDmnHistoryEventProducer();
      initHistoryEventHandler();
      initExpressionManager();
  ```

  ​    invokePreInit();

  ​	//会挨个执行各个插件的init的方法，最终会触发org.camunda.bpm.spring.boot.starter.configuration.impl.DefaultDatasourceConfiguration

  的执行，将spring的datasorce和camunda的datasource打通

  ```
  
      if (camundaDataSource == null) {
        configuration.setDataSource(dataSource);
      } else {
        configuration.setDataSource(camundaDataSource);
      }
  ```

  

