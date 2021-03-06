第3步：prepareBeanFactory\(beanFactory\)，为该Context中持有的beanFactory实例做一些准备工作。

该方法的实现都在AbstractApplicationContext中，逻辑很清晰，我们一步一步分析它做了什么。

第一段

```java
// Tell the internal bean factory to use the context's class loader etc.
beanFactory.setBeanClassLoader(getClassLoader());
beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
```

setBeanClassLoader：设置加载Bean用哪个ClassLoader。

setBeanExpressionResolver：设置用来解析BeanDefinition中“EL表达式”的解析器，即写的那些形如`#{...}`的Spring EL表达式。

addPropertyEditorRegistrar：设置属性编辑器，相当于registerCustomEditor的便捷操作，具体看到底做了什么：

```java
public ResourceEditorRegistrar(ResourceLoader resourceLoader, PropertyResolver propertyResolver) {
    this.resourceLoader = resourceLoader;
    this.propertyResolver = propertyResolver;
}
```

 由于AbstractApplicationContext继承自DefaultResourceLoader，而Environment的真正propertyResolve的功能是在PropertySourcesPropertyResolver中，所以这里的resourceLoader是DefaultResourceLoader，而propertyResolver是PropertySourcesPropertyResolver。

第二段

```java
// Configure the bean factory with context callbacks.
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

beanFactory.addBeanPostProcessor：添加了一个ApplicationContextAwareProcessor，对于BeanFactory添加的BeanPostProcessor，BeanFactory生产的每个Bean都会在对应的时机回调这个BeanPostProcessor。这个ApplicationContextAwareProcessor是在Bean完成初始化之后执行的，负责回调Bean实现的一些Aware接口方法。

beanFactory.ignoreDependencyInterface\(Class&lt;?&gt; ifc\)：告诉beanFactory，如果某个Bean要自动注入某个Bean时，如果要注入的这个Bean的Class类型为ifc，则不进行注入，这个方法仅在允许在Spring内部中使用，这么做有什么意义呢？TODO，等到后面看什么时候使用了这个属性。

第三段

```java
// BeanFactory interface not registered as resolvable type in a plain factory.
// MessageSource registered (and found for autowiring) as a bean.
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

 beanFactory.registerResolvableDependency\(dependencyType, autowiredValue\)表示：当程序中需要某个dependencyType，给用定的这个autowiredValue来装配，主要是针对那些需要被装配但又没有注册的Bean，这类Bean主要就是beanFactory或者context中的引用，如上所示。

第四段

```java
// Register early post-processor for detecting inner beans as ApplicationListeners.
beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

// Detect a LoadTimeWeaver and prepare for weaving, if found.
if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    // Set a temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
}
```

ApplicationListenerDetector实现了DestructionAwareBeanPostProcessor和MergedBeanDefinitionPostProcessor，在postProcessMergedBeanDefinition中它会记录bean是不是单例，然后在postProcessAfterInitialization中先判断bean是不是ApplicationListener，如果是则再判断是不是单例，如果是单例就将其添加到该context中，所以它的功能就如同它的名字一样：探测应用监听器。

beanFactory.containsBean\(LOAD\_TIME\_WEAVER\_BEAN\_NAME\)是判断beanFactory是否包含LoadTimeWeaver。在AOP中，织入切面的时机有：编译期织入、类加载期织入和运行期织入。编译期织入是指在Java编译期，采用特殊的编译器，将切面织入到Java类中；而类加载期织入则指通过特殊的类加载器，在类字节码加载到JVM时，织入切面；运行期织入则是采用CGLib工具或JDK动态代理进行切面的织入。这个LoadTimeWeaver则是在类加载期间的织入，如果设置了织入器，则需要将beanFactory的类加载器由默认的替换为这个ContextTypeMatchClassLoader这个特殊的类加载器。一般我们都会使用JDK动态代理或者Cglib这种运行期织入的方式，所以这里不再赘述。

第五段

```java
//String ENVIRONMENT_BEAN_NAME = "environment";
//String SYSTEM_PROPERTIES_BEAN_NAME = "systemProperties";
//String SYSTEM_ENVIRONMENT_BEAN_NAME = "systemEnvironment";
if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
}
if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
}
if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
}
```

由于是第一次refresh，所以这些if判断都是true，即将这3个bean添加到beanFactory。environment即为第一步创建的StandardEnvironment，systemProperties是系统属性，结构是Map&lt;String, Object&gt;，比如os.version、user.name、java.version等等信息，而systemEnvironment则是系统的环境变量，，结构也是Map&lt;String, Object&gt;，如JAVA\_HOME等等。

总结这一步的工作：

①设置beanFactory加载Bean的ClassLoader；

②创建Spring EL表达式的解析器StandardBeanExpressionResolver；

③设置beanFactory的资源加载器DefaultResourceLoader，设置beanFactory的属性解析器PropertySourcesPropertyResolver；

④创建进行Aware接口回调的BeanPostProcessor\(ApplicationContextAwareProcessor\)，并添加到beanFactory中；

⑤加载系统属性和系统变量到beanFactory，并将Environment、systemProperties、systemEnvironment注册到beanFactory。

































参考

[Spring之LoadTimeWeaver——一个需求引发的思考](http://sexycoding.iteye.com/blog/1062372)

 

