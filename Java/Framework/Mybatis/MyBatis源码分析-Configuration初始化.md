MyBatis初始化主要工作是加载并解析配置文件、Mapper映射配置文件以及相关的配置信息。初始化入口是`SqlSessionFactoryBuilder#build(Configuration)`。

```java
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```



主要逻辑是：

首先，根据配置文件的输入流、环境ID、传入属性，初始化XMLConfigBuilder对象，

再调用XMLConfigBuilder#parse方法解析配置文件内容，返回Configuration对象。

最后，通过SqlSessionFactoryBuilder#build(Configuration)方法，返回DefaultSqlSessionFactory对象。

并在finally代码块中关闭配置文件的输入流对象。

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
        XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
        return build(parser.parse());
    } catch (Exception e) {
        ...
    } finally {
            inputStream.close();        
    }
}

public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```



`XMLConfigBuilder`继承于`BaseBuilder`，而`BaseBuilder`是负责配置文件解析的父类。

属性Configuration保存MyBatis全局配置信息，

TypeAliasRegistry是类型别名注册器，负责注册与转换类型别名与类型全限定类名

TypeHandlerRegistry是类型转换注册器，负责注册与转换JDBC类型与Java类型

```java
public abstract class BaseBuilder {
  protected final Configuration configuration;
  protected final TypeAliasRegistry typeAliasRegistry;
  protected final TypeHandlerRegistry typeHandlerRegistry;

  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
    ...
}
```



`XMLConfigBuilder`的初始化：

```java
public class XMLConfigBuilder extends BaseBuilder {
    
    private final XPathParser parser;
    private String environment;
    private final ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();

    public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
        this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
    }

    private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
        //XMLConfigBuilder继承于BaseBuilder，初始使用new Configuration()去初始化
        super(new Configuration());
        //保存方法传入的属性集合，用于覆盖解析配置文件<properties>节点的属性
        this.configuration.setVariables(props);
        this.environment = environment;
        this.parser = parser;
    }
    ...
}
```



`XMLConfigBuilder#parse`方法

主要逻辑：按照配置文件`configuration`元素下各个元素逐一解析，将解析后的信息保存到全局配置信息对象configuration中。

```java
public Configuration parse() {
  parseConfiguration(parser.evalNode("/configuration"));
  return configuration;
}

private void parseConfiguration(XNode root) {
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    //设置Configuration#vfsImpl为setting元素vfsImpl属性指定的类
    loadCustomVfs(settings);
    //设置Configuration#logImpl为setting元素logImpl属性指定的类
    loadCustomLogImpl(settings);
    
    typeAliasesElement(root.evalNode("typeAliases"));
    pluginElement(root.evalNode("plugins"));
    //设置Configuration#objectFactory为objectFactory元素指定的类
    objectFactoryElement(root.evalNode("objectFactory"));
    //设置Configuration#objectWrapperFactory为objectWrapperFactory元素指定的类
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    //设置Configuration#reflectorFactory为reflectorFactory元素指定的类
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    
    //使用settings的键值或默认值初始化configuration对象的属性
    settingsElement(settings);
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    typeHandlerElement(root.evalNode("typeHandlers"));
    mapperElement(root.evalNode("mappers"));
}
```



解析`properties`的主要逻辑：

首先，解析property的键值，接着，读取resource或url属性指定路径的prop文件中的键值，再使用方法入参传递的键值，最后，将解析后的键值保存到configuration#variables中。

如果属性在不只一处被指定，则后者会覆盖前者，可以看出属性设置的优先级：通过方法参数传递的属性具有最高优先级，resource/url 属性中指定的配置文件次之，最低优先级的则是 properties 元素中指定的属性。

```java
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        //首先，解析property元素的键值对
        Properties defaults = context.getChildrenAsProperties();
        String resource = context.getStringAttribute("resource");
        String url = context.getStringAttribute("url");
        if (resource != null && url != null) {
            throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.");
        }
        //接着，读取resource或url属性指定路径的prop文件中的键值对
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
        } else if (url != null) {
            defaults.putAll(Resources.getUrlAsProperties(url));
        }
        //最后，使用方法调用时传入的prop，由于使用Properties保存，可以看出优先级：方法调用传入 > resource/url指定文件设置 > <property>设置 
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        //保存到xPathParser与configuration对象中
        parser.setVariables(defaults);
        configuration.setVariables(defaults);
    }
}
```



解析`settings`的主要逻辑：

解析setting的键值，并检查setting的键是否为Configuration类的可写属性

```java
private Properties settingsAsProperties(XNode context) {
  if (context == null) {
    return new Properties();
  }
  Properties props = context.getChildrenAsProperties();
  // 通过hasSetter检查键是否为Configuration的属性
  MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
  for (Object key : props.keySet()) {
    if (!metaConfig.hasSetter(String.valueOf(key))) {
      throw new BuilderException("The setting " + key + " is not known.");
    }
  }
  return props;
}
```



解析`typeAliases`的主要逻辑：

首先，读取package指定包下的所有JavaBean类，以及指定type的JavaBean类

然后，读取alias指定的别名，如果为空则查找JavaBean的@Alias注解指定的别名，否则，使用简单类名作为别名

最后，通过别名的小写格式与JavaBean类的Class，注册到TypeAliasRegistry#typeAliases中。

```java
private void typeAliasesElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      if ("package".equals(child.getName())) {
        String typeAliasPackage = child.getStringAttribute("name");
        configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
      } else {
        String alias = child.getStringAttribute("alias");
        String type = child.getStringAttribute("type");
        try {
          Class<?> clazz = Resources.classForName(type);
          if (alias == null) {
            typeAliasRegistry.registerAlias(clazz);
          } else {
            typeAliasRegistry.registerAlias(alias, clazz);
          }
        } catch (ClassNotFoundException e) {
          throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
        }
      }
    }
  }
}
```



`TypeAliasRegistry` 类型别名注册器，负责注册转换Java类型与别名的映射。其中无参构造方法已注册了自带的类型别名。

```java
public class TypeAliasRegistry {
  private final Map<String, Class<?>> typeAliases = new HashMap<>();
    ...
}
```





解析`plugins`的主要逻辑：

解析后并实例化拦截器对象，并将拦截器对象添加到Configuration#interceptorChain中。

```java
private void pluginElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      String interceptor = child.getStringAttribute("interceptor");
      Properties properties = child.getChildrenAsProperties();
      Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor().newInstance();
      interceptorInstance.setProperties(properties);
      configuration.addInterceptor(interceptorInstance);
    }
  }
}
```



`InterceptorChain`拦截器链，保存着一系列拦截器引用，拦截器在映射语句执行过程对指定方法签名拦截以动态代理方法加强逻辑。

```java
public class InterceptorChain {
  private final List<Interceptor> interceptors = new ArrayList<>();
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }
   ...
}

public interface Interceptor {
  Object intercept(Invocation invocation) throws Throwable;
  default Object plugin(Object target) {
      //Plugin类及该方法后续在补充具体分析
    return Plugin.wrap(target, this);
  }
  default void setProperties(Properties properties) { }
}
```



解析`environments`的主要逻辑：

首先，如果方法入参未传入环境ID，则设置全局配置的环境ID为"default"

然后，根据环境ID，解析`environment`的事务管理器工厂与数据源的信息，用于初始化Environment对象。

最后，将Environment对象保存到Configuration#environment中。

```java
private void environmentsElement(XNode context) throws Exception {
  if (context != null) {
    if (environment == null) {
      environment = context.getStringAttribute("default");
    }
    for (XNode child : context.getChildren()) {
      String id = child.getStringAttribute("id");
      if (isSpecifiedEnvironment(id)) {
        TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
        DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
        DataSource dataSource = dsFactory.getDataSource();
        Environment.Builder environmentBuilder = new Environment.Builder(id)
            .transactionFactory(txFactory)
            .dataSource(dataSource);
        configuration.setEnvironment(environmentBuilder.build());
      }
    }
  }
}
```



`Environment`环境信息，保存环境ID、事务管理器工厂、数据源的信息。

```java
public final class Environment {
  private final String id;
  private final TransactionFactory transactionFactory;
  private final DataSource dataSource;
}
```