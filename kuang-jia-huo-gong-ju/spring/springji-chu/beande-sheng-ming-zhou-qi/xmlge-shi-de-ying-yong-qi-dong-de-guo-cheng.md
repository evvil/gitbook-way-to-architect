第二步：obtainFreshBeanFactory\(\)：获取BeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

refreshBeanFactory\(\)是留给子类实现的，注释说是加载配置信息，具体到ClassPathXmlApplicationContext，它的实现其实是在父类AbstractRefreshableApplicationContext中，代码如下：

```java
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

AbstractRefreshableApplicationContext中持有一个DefaultListableBeanFactory实例beanFactory，hasBeanFactory其实是看beanFactory是否为空，如果不为空，说明之前实例化过，则进销毁之前的bean管理的Bean并关闭该beanFactory：

```java
protected void destroyBeans() {
    getBeanFactory().destroySingletons();
}
```

这里的destroySingletons\(\)其实就是从单例注册中心（理解为map&lt;beanName, bean&gt;）clearAll。即destroy并不是从JVM将之前创建的Bean销毁（其实也没有这样的方法可以达到这样的效果），只是将其从该Context脱离出来。

closeBeanFactory则是将该Context持有的DefaultListableBeanFactory实例beanFactory置为null。

现在，看createBeanFactory，由于没有父工厂，其实就是new了一个DefaultListableBeanFactory实例。

```java
protected DefaultListableBeanFactory createBeanFactory() {
    return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
```

customizeBeanFactory是进行定制化Bean工厂：

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    if (this.allowBeanDefinitionOverriding != null) {
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.allowCircularReferences != null) {
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```

 由于AbstractRefreshableApplicationContext中这两个变量初始值均为null，而最终的实现类ClassPathXmlApplicationContext也没有设置这两个值，所以这两个值均为null，则该Context中的DefaultListableBeanFactory采用默认值true，即允许覆盖之前的BeanDefinition（即添加&lt;beanName-&gt;BeanDefinition&gt;时，发现已经存在相同BeanName的&lt;beanName-&gt;BeanDefinition&gt;，则替换掉原来的），允许循环依赖。

loadBeanDefinitions则是去真正加载BeanDefinition，具体实现是在AbstractXmlApplicationContext中，由XmlBeanDefinitionReader去完成。这里流程比较复杂，暂时跳过。需要清楚的一点是，该方法是将完整的BeanDefinitions加载进BeanFactory。

这一步的主要工作：

①创建\(new\)一个DefaultListableBeanFactory实例（如果之前存在，则销毁重新new），设置为允许覆盖BeanDefinition，并允许循环依赖。

②使用XmlBeanDefinitionReader加载BeanDefinitions到DefaultListableBeanFactory。

 

