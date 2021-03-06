# 原型模式

原型模式属于创建者模式的一种。原型模式会要求对象实现一个克隆自身的接口，这样就可以t通过拷贝一个对象实例本身来新建一个实例，这样就无需关心对象的类型，对象创建的逻辑等。只要它实现了克隆自身的方法，就可以通过克隆方法来获取新的对象，而无需使用new。

**原型模式步骤：**

①定义一个接口，该接口中定义克隆方法

②创建具体的类（原型类），实现该接口及方法

③客户端使用②中的实例的clone方法，即可创建新的实例

在`Java`具体实现时，只需要让原型类实现`cloneable`接口，实现其中的`Object clone()`方法即可。

关于浅拷贝和深拷贝的问题，请参考：[理解Java中的深拷贝和浅拷贝](https://www.jianshu.com/p/beeba83fe503)

| 浅拷贝 | 深拷贝 |
| :--- | :--- |
| 原对象和克隆对象并不是100%无关联 | 原对象和克隆对象100%无关联 |
| 对克隆对象的任何改变都会反映在原对象中，反之亦然 | 克隆对象的改变不会反映在原对象中，反之亦然 |
| 默认的clone\(\)方法创建的是浅拷贝 | 要实现深拷贝，必须重写clone\(\)方法 |
| 如果一个对象中字段只有基本类型，推荐浅拷贝 | 如果一个对象中字段存在其他对象的引用类型，推荐深拷贝 |
| 浅拷贝速度快，代价小 | 深拷贝相对较慢，代价大 |

**优势**

将对象的创建过程封装起来，客户端不需要了解对象的具体创建流程。

**缺点**  
每一个类必须都有一个clone方法，如果这个类的组成不太复杂的话还比较好，如果类的组成很复杂的话，如果想实现深度复制就非常困难了。

TODO：找到原型模式在一些类库中的应用（2018-05-03）

## 参考

[原型模式](http://www.runoob.com/design-pattern/prototype-pattern.html)  
[设计模式（六）原型模式](https://www.kancloud.cn/digest/xing-designpattern/143727)

《研磨设计模式--第九章：原型模式》

