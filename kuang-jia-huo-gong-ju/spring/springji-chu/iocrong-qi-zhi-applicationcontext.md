# ApplicationContext

---

ApplicationContext首先是一个容器，具备BeanFactory的所有的功能，但扩展了BeanFactory的功能，如下是ApplicaitonContext的继承关系图。

![](/assets/屏幕快照 2018-10-29 下午10.27.55.png)

**ApplicationEventPublisher**

事件发布器，定义了`void publishEvent(Object event);`方法。

所以，ApplicationContext具备了事件发布的功能。

**EnvironmentCapable**

该接口定义的方法了一个方法：`Environment getEnvironment();`，结合其名字中的capable，可知实现者可具备环境属性。

所以，ApplicationContext具备了区分不同环境的能力或者说拥有了环境这种属性。

**ResourceLoader**

资源加载器，ResourcePatternResolver则实现了能够识别特定前缀和Ant风格的路径。

**MessageSource**

ApplicationContext继承了上述这些接口，那么这些接口所定义的内容或者功能就是ApplicationContext在BeanFactory的基础之上所扩展的地方，ApplicationContext接口定义的方法主要都是描述自身信息的东西，没有功能扩展的方法。

**ConfigurableApplicationContext**

除了通过继承其他接口来扩展功能，ApplicationContext其他主要的功能扩展的方法都在这个接口中：

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
    //设置父ApplicationContext
    void setParent(@Nullable ApplicationContext parent);
    //设置environment
    void setEnvironment(ConfigurableEnvironment environment);
    //添加BeanFactoryPostProcessor
    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);
    //添加应用监听器
    void addApplicationListener(ApplicationListener<?> listener);
    //添加协议解析器
    void addProtocolResolver(ProtocolResolver resolver);
    //真正的应用启动的方法
    void refresh() throws BeansException, IllegalStateException;
    //注册一个shutdown的钩子
    void registerShutdownHook();
    //该ApplicationContext是否已经refresh并且没有没有被close掉
    boolean isActive();
    //该ApplicationContext内部的工厂实例
    ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
}
```

而其继承的Lifecycle和Closeable分别如下：

```java
public interface Lifecycle {

    void start();

    void stop();

    boolean isRunning();
}

public interface Closeable extends AutoCloseable {
    public void close() throws IOException;
}
```

通过ConfigurableApplicationContext的加强，ApplicationContext才更加与我们实际中的应用相像：有了启动、停止、关闭、刷新等方法。

其中的refresh\(\)方法非常重要：它是真正的启动方法，这里的启动是说启动应用。

**AbstractApplicationContext**

实现了Context的基础功能，使用模板设计模式，让子类来实现抽象方法。它实现了refresh\(\)方法，即应用启动的逻辑。

**AbstractRefreshableApplicationContext**

允许多次调用refresh\(\)方法，子类只需要实现loadBeanDefinitions方法即可，通常具体的实现类要将BeanDefinitions传递给BeanFactory，这个工作一般会委托给各种具体的BeanDefinitionReader来完成 

**AbstractRefreshableConfigApplicationContext**

在 AbstractRefreshableApplicationContext的基础上，增加了处理不同location的能力，即根据不同location去加载BeanDefinitions。

* FileSystemXmlApplicationContext：从XML文件加载被定义的bean，访问路径为系统文件路径如C://beans.xml
* ClassPathXmlApplicationContext：从XML文件加载被定义的bean，提供Classpath路径的ApplicationContex
* WebXmlApplicationContext：从XML文件加载被定义的bean，加载路径基于整个Web工程  

以上的几种AbstractApplicationContext实现的主要区别在于加载Xml定义文件的路径不同，但是加载bean的format都是XML的，且文件路径的表示方式单一，不能灵活改动。

**GenericApplicationContext**

这个ApplicationContext与上面介绍的几个实现类不同的是：它可以允许读取任意格式定义的BeanDefinition，并且内部的BeanFactory总是同一个实例。

允许读取任意格式定义的BeanDefinition，是说不仅仅支持xml格式的，也支持其他格式的，借助其他的BeanDefinitionReader读取到BeanDefinition后，直接注册到BeanDefinitionRegistry中即可。

内部的BeanFactory总是同一个实例，这句话与前面的几个实现类对比来说的，上面的几个实现类，每次执行refresh的时候，都是将之前的内部BeanFactory销毁，再重新创建\(new\)一个，而GenericApplicationContext则是在构造时就创建出内部BeanFactory，且每次执行refresh的时候，都是使用的同一个内部BeanFactory实例。

**AnnotationConfigApplicationContext**

 它是GenericApplicationContext的子类，上面说GenericApplicationContext可以允许读取任意格式定义的BeanDefinition，那么从名字上就可以看出，AnnotationConfigApplicationContext是读取以注解的形式定义的BeanDefinition，使用AnnotatedBeanDefinitionReader进行读取。

这些注解包括@Configuration、@Bean、@Compoment等等。

比如xml形式：

```xml
<beans>
    <bean id="course" class="demo.Course">
        <property name="module" ref="module"/>
    </bean>
     
    <bean id="module" class="demo.Module">
        <property name="assignment" ref="assignment"/>
    </bean>
</beans>
```

而使用AnnotationConfigApplicationContext，则无需xml文件：

```java
@Configuration
public class AppContext {
    
    @Bean
    public Course course() {
        Course course = new Course();
        course.setModule(module());
        return course;
    }
 
    @Bean
    public Module module() {
        Module module = new Module();
        module.setAssignment(assignment());
        return module;
    }
 
    @Bean
    public Assignment assignment() {
        return new Assignment();
    }
}
```

 是不是有点SpringBoot的感觉。没错，SpringBoot就是使用这个类（非Web应用）。

## 参考

---

[简析GenericApplicationContext](https://www.jianshu.com/p/524d62ee91fb)

[spring注解开发AnnotationConfigApplicationContext的使用](https://www.cnblogs.com/kaituorensheng/p/8024199.html) 

[使用 Java 配置进行 Spring bean 管理](https://www.ibm.com/developerworks/cn/webservices/ws-springjava/index.html)

[Spring boot源码分析-AnnotationConfigApplicationContext非web环境下的启动容器（2）](https://blog.csdn.net/jamet/article/details/77417997)

[Spring核心类 - AnnotationConfigApplicationContext](https://www.zybuluo.com/yulongsun/note/1139397)

[Spring 官方文档：SpringApplication  
](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-spring-application.html)

 

