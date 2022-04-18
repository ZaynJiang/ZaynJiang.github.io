## 1.  源码编译

https://github.com/apache/skywalking.git

切换至6.x的分支

## 2. 客户端启动

### 2.1. premain配置初始化

org.apache.skywalking.apm.agent.SkyWalkingAgent#premain

	 public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException, IOException {
	     		//初始化
	            SnifferConfigInitializer.initialize(agentArgs);
	            
				//插件查找
	            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
	            
	            //ByteBuddy框架字节码增强
	            final ByteBuddy byteBuddy = new  ByteBuddy()
	            .with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
	            
	            //构建builder并且忽略一些不增强的类
	            AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy)
	            .ignore(
	                nameStartsWith("net.bytebuddy."
	                    .or(nameStartsWith("org.slf4j."))
	                    .or(nameStartsWith("org.groovy."))
	                    .or(nameContains("javassist"))
	                    .or(nameContains(".asm."))
	                    .or(nameContains(".reflectasm."))
	                    .or(nameStartsWith("sun.reflect"))
	                    .or(allSkyWalkingAgentExcludeToolkit())
	                    .or(ElementMatchers.<TypeDescription>isSynthetic()));
	                    
	           agentBuilder
	            //要通过插件增强的一些类，这个类可以通过pluginFinder buildMath方法查看
	            .type(pluginFinder.buildMatch()) 
	            //在class加载的时候会执行，相当于一个监听，传入pluginFinder进行增强
	            .transform(new Transformer(pluginFinder))
	            .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
	            //放了一个监听器，主要打了一些日志
	            .with(new Listener())
	            .installOn(instrumentation);
	          
	        }

做了很多工作

#### 2.1.1. 配置初始化

```
    public static void initialize(String agentOptions) {
        InputStreamReader configFileStream;
        try {
            configFileStream = loadConfig();
            Properties properties = new Properties();
            properties.load(configFileStream);
            for (String key : properties.stringPropertyNames()) {
                String value = (String)properties.get(key);
                properties.put(key, PropertyPlaceholderHelper.INSTANCE.replacePlaceholders(value, properties));
            }
            ConfigInitializer.initialize(properties, Config.class);
            .....
           overrideConfigBySystemProp();
          
             .....
            overrideConfigByAgentOptions(agentOptions);
               
            //校验必填参数
           if (StringUtil.isEmpty(Config.Agent.SERVICE_NAME)) {
                throw new ExceptionInInitializerError("`agent.service_name` is missing.");
            }
            if (StringUtil.isEmpty(Config.Collector.BACKEND_SERVICE)) {
                throw new ExceptionInInitializerError("`collector.backend_service` is missing.");
            }
            if (Config.Plugin.PEER_MAX_LENGTH <= 3) {
                logger.warn("PEER_MAX_LENGTH configuration:{} error, the default value of 200 will be used.", Config.Plugin.PEER_MAX_LENGTH);
                Config.Plugin.PEER_MAX_LENGTH = 200;
            }
```

* loadConfig()

  会从skywaking-agent.jar会截取它的全路径截取所在目录下找到/config/agent.config这个配置文件

  截取的方法为：org.apache.skywalking.apm.agent.core.boot.AgentPackagePath#findPath

  即通过agentpackagepath所在的位置jar,确定agent配置的绝对路径

* properties.load(configFileStream)

  将key-value的配置转化为propeties文件的形式

*  PropertyPlaceholderHelper

  因为skywalking的配置项为，server_name=${a1:a2}

  a1是环境变量，a2是填写的值，如果没有填写a2则使用a1环境默认的值，这个使用PropertyPlaceholderHelper帮助解析

  最终会将key在properties配置文件里覆盖掉

* 所有的配置项都是通过config类进行映射转换的

  ```
  ConfigInitializer.initialize(properties, Config.class);
  ```

* overrideConfigBySystemProp

  使用系统环境变量进行覆盖

* overrideConfigByAgentOptions

  命令行的参数的优先级最高然后覆盖掉

* 校验必填参数

#### 2.1.2. 插件查找

注意AbstractClassEnhancePluginDefine是所有插件的父类，插件查找的主要代码为：

```
pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
```

**.loadPlugins()的分析如下：**

```
 public List<AbstractClassEnhancePluginDefine> loadPlugins() throws AgentPackageNotFoundException {
       //初始化自定义类加载器
       AgentClassLoader.initDefaultLoader();
		
	   //寻找资源文件
       PluginResourcesResolver resolver = new PluginResourcesResolver();
       List<URL> resources = resolver.getResources();
       
       //封装插件类的定义对象列表
       for (URL pluginUrl : resources) {
       	   try {
       		  PluginCfg.INSTANCE.load(pluginUrl.openStream());
           } catch (Throwable t) {
              logger.error(t, "plugin file [{}] init failure.", pluginUrl);
           }
       }
        List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();
        
        //根据插件类的定义创建插件对象
    }
```

* initDefaultLoader

  将所有插件加载出来，loadPlugins()的方法如下：

  * AgentClassLoader.initDefaultLoader();

    初始化自定义类加载器

        public static void initDefaultLoader() throws AgentPackageNotFoundException {
            if (DEFAULT_LOADER == null) {
                synchronized (AgentClassLoader.class) {
                    if (DEFAULT_LOADER == null) {
                        DEFAULT_LOADER = new AgentClassLoader(PluginBootstrap.class.getClassLoader());
                    }
                }
            }
        }

    ```
    public AgentClassLoader(ClassLoader parent) throws AgentPackageNotFoundException {
        super(parent);
        File agentDictionary = AgentPackagePath.getPath();
        classpath = new LinkedList<File>();
        classpath.add(new File(agentDictionary, "plugins"));
        classpath.add(new File(agentDictionary, "activations"));
    }
    ```

    获取jar包所在的路径将plugings目录和activations目录下的jar中插件加载进来

  * tryRegisterAsParallelCapable

    解析插件中的类的方法，将classloader使用方式的方式注册成一个并行的classloader,从而提升类加载的速度，即调用registerAsParallelCapable方法实现

    ````
    public class AgentClassLoader extends ClassLoader {
        static {
            tryRegisterAsParallelCapable();
        }
       private static void tryRegisterAsParallelCapable() {
            Method[] methods = ClassLoader.class.getDeclaredMethods();
            for (int i = 0; i < methods.length; i++) {
                Method method = methods[i];
                String methodName = method.getName();
                if ("registerAsParallelCapable".equalsIgnoreCase(methodName)) {
                    try {
                        method.setAccessible(true);
                        method.invoke(null);
                    } catch (Exception e) {
                        logger.warn(e, "can not invoke ClassLoader.registerAsParallelCapable()");
                    }
                    return;
                }
            }
        }
    ````

  * getAllJars

    在classpath下找到所有的jar文件

    ```
      private List<Jar> getAllJars() {
            if (allJars == null) {
                jarScanLock.lock();
                try {
                    if (allJars == null) {
                        allJars = new LinkedList<Jar>();
                        for (File path : classpath) {
                            if (path.exists() && path.isDirectory()) {
                                String[] jarFileNames = path.list(new FilenameFilter() {
                                    @Override
                                    public boolean accept(File dir, String name) {
                                        return name.endsWith(".jar");
                                    }
                                });
                                for (String fileName : jarFileNames) {
                                    try {
                                        File file = new File(path, fileName);
                                        Jar jar = new Jar(new JarFile(file), file);
                                        allJars.add(jar);
                                        logger.info("{} loaded.", file.toString());
                                    } catch (IOException e) {
                                        logger.error(e, "{} jar file can't be resolved", fileName);
                                    }
                                }
                            }
                        }
                    }
                } finally {
                    jarScanLock.unlock();
                }
            }
    
            return allJars;
        }
    ```

    最终这个jar包会拼接成.class，调用classloader的内置方法加载

  总的来说，使用自定义的类加载器加载自己的插件，可以起到class隔离资源的功能，这个和tomcat的类似

*  resolver.getResources()

  通过classloader将jar中的skywalking-plugin.def文件

  这个文件在每个插件包都有，定义了每类插件的增强点，例如dubbo的插件的：

  ```
  dubbo=org.apache.skywalking.apm.plugin.dubbo.patch.WrapperInstrumentation
  ```

  这个将返回一个URL的list

* PluginCfg.INSTANCE.load

  ```
  void load(InputStream input) throws IOException {
          try {
              BufferedReader reader = new BufferedReader(new InputStreamReader(input));
              String pluginDefine = null;
              while ((pluginDefine = reader.readLine()) != null) {
                  try {
                      if (pluginDefine == null || pluginDefine.trim().length() == 0 || pluginDefine.startsWith("#")) {
                          continue;
                      }
                      PluginDefine plugin = PluginDefine.build(pluginDefine);
                      pluginClassList.add(plugin);
                  } catch (IllegalPluginDefineException e) {
                      logger.error(e, "Failed to format plugin({}) define.", pluginDefine);
                  }
              }
          } finally {
              input.close();
          }
      }
  ```

  一行行上一步的配置文件，然后将配置文件中的key插件名称=class名称读取出来，然后生成PluginDefine定义对象，然后添加到list之中返回

* 创建插件实例

  使用Class.forName来创建class对象，使用AgentClassLoader.getDefault()的类加载器来创建最终的实例,最终返回plugin列表

  ```
   List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<AbstractClassEnhancePluginDefine>();
          for (PluginDefine pluginDefine : pluginClassList) {
              try {
                  logger.debug("loading plugin class {}.", pluginDefine.getDefineClass());
                  AbstractClassEnhancePluginDefine plugin =
                      (AbstractClassEnhancePluginDefine)Class.forName(pluginDefine.getDefineClass(),
                          true,
                          AgentClassLoader.getDefault())
                          .newInstance();
                  plugins.add(plugin);
              } catch (Throwable t) {
                  logger.error(t, "load plugin [{}] failure.", pluginDefine.getDefineClass());
              }
          }
  
          plugins.addAll(DynamicPluginLoader.INSTANCE.load(AgentClassLoader.getDefault()));
  ```

**new PluginFinder的分析如下：**

```
public class PluginFinder {
    private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<String, LinkedList<AbstractClassEnhancePluginDefine>>();
    private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();
    private final List<AbstractClassEnhancePluginDefine> bootstrapClassMatchDefine = new ArrayList<AbstractClassEnhancePluginDefine>();   
   
   public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
        for (AbstractClassEnhancePluginDefine plugin : plugins) {
            ClassMatch match = plugin.enhanceClass();

            if (match == null) {
                continue;
            }

            if (match instanceof NameMatch) {
                NameMatch nameMatch = (NameMatch)match;
                LinkedList<AbstractClassEnhancePluginDefine> pluginDefines = nameMatchDefine.get(nameMatch.getClassName());
                if (pluginDefines == null) {
                    pluginDefines = new LinkedList<AbstractClassEnhancePluginDefine>();
                    nameMatchDefine.put(nameMatch.getClassName(), pluginDefines);
                }
                pluginDefines.add(plugin);
            } else {
                signatureMatchDefine.add(plugin);
            }

            if (plugin.isBootstrapInstrumentation()) {
                bootstrapClassMatchDefine.add(plugin);
            }
        }
    }
```

构造方法对象实际上是将pulgin分成两类

* 全匹配，根据全类名进行匹配

  NamedMath就是全类名匹配的插件。

* 间接匹配，根据注解或者通配符等其它的方式进行匹配

  如PrefixMatch为前缀匹配，MethodAnnotationMatch为注解匹配，这些匹配器都是ClassMatch的实现类

注意存放插件的容器时map时key-list结构，因为在前面插件的配置文件多个相同的key value形式，即一对多的形式。

最终我们的插件都没放到了PlginFinder之中

**new PluginFinder的find方法**:

根据给定的一个类找到所有可以使用的插件list

```
public List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription) {
    List<AbstractClassEnhancePluginDefine> matchedPlugins = new LinkedList<AbstractClassEnhancePluginDefine>();
    String typeName = typeDescription.getTypeName();
    if (nameMatchDefine.containsKey(typeName)) {
        matchedPlugins.addAll(nameMatchDefine.get(typeName));
    }

    for (AbstractClassEnhancePluginDefine pluginDefine : signatureMatchDefine) {
        IndirectMatch match = (IndirectMatch)pluginDefine.enhanceClass();
        if (match.isMatch(typeDescription)) {
            matchedPlugins.add(pluginDefine);
        }
    }

    return matchedPlugins;
}
```

* TypeDescription

  就是查找类的一个描述对象，主要是全类名即代码中的

  String typeName = typeDescription.getTypeName();

* nameMatchDefine

  先从nameMatchDefine查找是否包含，添加到结果之中

* matchedPlugins

  再通过matchedPlugins查找，添加到结果list之中

#### 2.1.3. ByteBuddy增强

* 构建AgentBuilder

  * Config.Agent.IS_OPEN_DEBUGGING_CLASS

    ByteBuddy创建时可以传入参数是否开启debug模式，如果开启的话，会将一些增强的日志放到debuging目录下

  * 忽略一些不增强的class

* agentBuilder.type(pluginFinder.buildMatch())

  传入插件需要增强的一些类，即调用pluginFinder的buildMatch();

  ```
  public ElementMatcher<? super TypeDescription> buildMatch() {
          ElementMatcher.Junction judge = new AbstractJunction<NamedElement>() {
              @Override
              public boolean matches(NamedElement target) {
                  return nameMatchDefine.containsKey(target.getActualName());
              }
          };
          judge = judge.and(not(isInterface()));
          for (AbstractClassEnhancePluginDefine define : signatureMatchDefine) {
              ClassMatch match = define.enhanceClass();
              if (match instanceof IndirectMatch) {
                  judge = judge.or(((IndirectMatch)match).buildJunction());
              }
          }
          return new ProtectiveShieldMatcher(judge);
      }
  ```

  这个方法就是将我们所有需要增强的类构造成ElementMatcher

  ProtectiveShieldMatcher继承ElementMatcher，只是做了一些封装。主要是为了避免因为一些框架也用了这个字节码增强框架导致这个增强时发生异常，这个ProtectiveShieldMatcher覆盖了方法直接接覆盖。

* transform

  在class加载的时候会执行，相当于一个监听，传入pluginFinder进行增强

* 传入linstener

  放了一个监听器，主要打了一些日志

### 2.2.  agent启动

agent的启动是在premain函数作完配置后执行的

```
try {
	ServiceManager.INSTANCE.boot();
} catch (Exception e) {
	logger.error(e, "Skywalking agent boot failure.");
}
```

#### 2.2.1. 插件架构

* agent-core看作内核
* 服务就是各种各样的插件，这些统一由agent-core进行管理

​	maven的build就是个插件架构，有做测试的，有编译源码的，有打包的plugin,它允许开发者根据它的规范开发各种插件。skywalking的也是借鉴这种插件架构模式，bootservice就是它的规范。它是一个接口。它有4个主要的方法：

```
    void prepare() throws Throwable;

    void boot() throws Throwable;

    void onComplete() throws Throwable;

    void shutdown() throws Throwable;
```

anget-core内核会在不同的时机调用不同的方法。

比如JVMService是一个实现类。它主要时读取jvm的信息，这个类定义在agent-core的包下

#### 2.2.2. boot启动

```
  public void boot() {
        bootedServices = loadAllServices();
		
		//执行准备相关的方法
        prepare();
        startup();
        onComplete();
    }
```

通过spi机制加载BootService的实现类

```
void load(List<BootService> allServices) {
//这里通过spi机制BootService的实现类放入到Iterator容器中。
    Iterator<BootService> iterator = ServiceLoader.load(BootService.class, AgentClassLoader.getDefault()).iterator();
    while (iterator.hasNext()) {
        allServices.add(iterator.next());
    }
}
```

将BootService的实现类进行处理放到boot的map之中，但是会根据实现类是否含有的注解进行不同的处理

```
private Map<Class, BootService> loadAllServices() {
	//spi加载
	load(allServices);
	Iterator<BootService> serviceIterator = allServices.iterator();
	//迭代处理
    while (serviceIterator.hasNext()) {
    	//判断是否含有DefaultImplementor注解或者OverrideImplementor注解
    	  BootService bootService = serviceIterator.next();
            Class<? extends BootService> bootServiceClass = bootService.getClass();
            boolean isDefaultImplementor = bootServiceClass.isAnnotationPresent(DefaultImplementor.class);
            if (isDefaultImplementor) {
                if (!bootedServices.containsKey(bootServiceClass)) {
                    bootedServices.put(bootServiceClass, bootService);
                } else {
                    //ignore the default service
                }
            } else {
                OverrideImplementor overrideImplementor = bootServiceClass.getAnnotation(OverrideImplementor.class);
                //没有overrideImplementor注解，则放到map之中，如果已经有了就直接报错
                if (overrideImplementor == null) {
                    if (!bootedServices.containsKey(bootServiceClass)) {
                        bootedServices.put(bootServiceClass, bootService);
                    } else {
                        throw new ServiceConflictException("Duplicate service define for :" + bootServiceClass);
                    }
                 //有overrideImplementor注解 
                } else {
                    Class<? extends BootService> targetService = overrideImplementor.value();
                    if (bootedServices.containsKey(targetService)) {
                        boolean presentDefault = bootedServices.get(targetService).getClass().isAnnotationPresent(DefaultImplementor.class);
                        if (presentDefault) {
                            bootedServices.put(targetService, bootService);
                        } else {
                            throw new ServiceConflictException("Service " + bootServiceClass + " overrides conflict, " +
                                "exist more than one service want to override :" + targetService);
                        }
                    } else {
                        bootedServices.put(targetService, bootService);
                    }
                }
            }
    }
}
```

* DefaultImplementor注解

  默认实现

  ```
  @DefaultImplementor
  public class JVMService implements BootService, Runnable {
  	。。。。。
  }
  ```

  如果判断是这个注解，就放到map之中。

* OverrideImplementor注解

  可以定义OverrideImplementor（value=“父类实现class”）

  ```
  @OverrideImplementor(ContextManagerExtendService.class)
  public class TraceIgnoreExtendService extends ContextManagerExtendService {
  	......
  }
  ```

   父类实现class表示要覆盖的父类，父类实现必须要是默认实现，并且只能是覆盖一个。

总结下来就是：

if  存在  defalut

​	if 没有加载

​		进行加载

​	else

​		忽略	

else

​	if 不存在 override

​			if 没有加载

​				 进行加载

​			else

​				报错

​	else 

​			if  指定的类实现已经加载

​					if  指定的类存在于default之中

​						覆盖

​				    else 

​						报错

​			else 

​				直接加载

### 2.3. 注册关闭函数

```
Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
    @Override public void run() {
    	ServiceManager.INSTANCE.shutdown();
   }
}
```

## 3. 总结

