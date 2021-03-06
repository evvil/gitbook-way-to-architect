## Spring中的属性编辑器PropertyEditor

---

![](/assets/屏幕快照 2018-10-21 下午5.17.46.png)

**PropertyEditor**

一个Bean可能有很多属性需要设置，而配置时，我们只是写的字符串，Spring需要将字符串转换为我们定义的类型，比如：

```java
public class Person{
    private String name;
    private int age;

    //省略getter/setter
}
```

```xml
<bean id="person" class="com.maxwell.learning.spring.Person">
    <property name="name" value="maxwell" />
    <property name="age" value="20" />
</bean>
```

属性编辑器PropertyEditor工作就是：**将字符串转换为任意的Java类型**（如基本数据类型，我们写的Class，以及它们组成的Collection和Map等）。

PropertyEditor和PropertyEditorSupport都是java.bean中定义的接口/类，Spring则针对不同的情况进行了实现。

![](/assets/屏幕快照 2018-10-21 下午5.18.09.png)

**PropertyEditorRegister**

管理PropertyEditor，定义属性编辑器PropertyEditor的注册和查找：`void registerCustomEditor`和`PropertyEditor findCustomEditor`。

 **PropertyEditorRegisterSupport**

PropertyEditorRegister的默认实现。

既然是Register，内部肯定也是使用Map来实现：Map&lt;Class&lt;?&gt;, PropertyEditor&gt; customEditors。

除此之外，内部还持有一个`ConversionService conversionService`对象。

**TypeConverter**

PropertyEditor负责将字符串转换为任意Java类型，但有时候并不一定拿字符串去转换，有可能是其他类型，TypeConverter就是负责进行类型转换的，可以看做是PropertyEditor的升级版：PropertyEditor\(String -&gt; anyType\)，TypeConverter\(anyType-&gt;anyType\)。

**TypeConverterSupport**

对 TypeConverter的默认实现，其采用代理模式，交由TypeConverterDelegate实现，TypeConverterDelegate内部持有PropertyEditorRegisterSupport，转换策略为：先使用PropertyEditor转换器器转换，如果没找到对应的转换器器，会⽤ConversionService来进⾏行行对象转换。

**PropertyAccessor**

即前面获取到了某个字段的值，PropertyAccessor负责将这些具体的值设置到Bean的属性上或者说设置到对象的字段上，主要方法：

```java
//获取属性值
Object getPropertyValue(String propertyName)；
//设置属性值
void setPropertyValue(String propertyName, Object value)；
```

AbstractPropertyAccessor是它的实现类，AbstractNestablePropertyAccessor则支持嵌套结构的属性设置。

此外，Spring还使用PropertyValue来对&lt;propertyName, PropertyValue&gt;进行了包装，即PropertyAccessor中还有类似如下的定义：

```java
void setPropertyValue(PropertyValue pv)
```

**BeanWrapper**

BeanWrapper继承自ConfigurablePropertyAccessor，相当于Bean属性操作的总接口，包含了属性编辑、类型转换，属性设置的全部功能，并额外定义了获取包装对象的方法：

```java
//Return the bean instance wrapped by this object.
Object getWrappedInstance();

//Return the type of the wrapped bean instance.
Class<?> getWrappedClass();

//Obtain the PropertyDescriptors for the wrapped object
PropertyDescriptor[] getPropertyDescriptors();
```

**BeanWrapperImpl**

BeanWrapper的默认实现，通常是由PropertyAccessorFactory工厂来创建BeanWrapperImpl的实例：

```java
 public abstract class PropertyAccessorFactory {

    public static BeanWrapper forBeanPropertyAccess(Object target) {
        return new BeanWrapperImpl(target);
    }

    public static ConfigurablePropertyAccessor forDirectFieldAccess(Object target) {
        return new DirectFieldAccessor(target);
    }
}
```

Spring中的ConversionService

 

## 参考

---

[spring笔记-PropertyAccessor](https://www.jianshu.com/p/c2c419cb5bea)

[Spring3.1.0实现原理分析\(四\).属性访问器\(PropertyAccessor\)](https://blog.csdn.net/roberts939299/article/details/70599506)

