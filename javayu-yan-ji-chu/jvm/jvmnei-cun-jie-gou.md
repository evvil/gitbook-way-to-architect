## JVM内存结构

---

![](/assets/1932449-630b7fbee6be7f7c %281%29.png)

* **堆**

  * 存放对象，可分为新生代和老年代；
  * 线程共享

* **栈**

  * 描述Java方法运行过程的内存模型，由一个个栈帧组成，而每个栈帧中都拥有：局部变量表、操作数栈、动态链接、方法出口信息
  * 线程私有

* **方法区**

  * 方法区中存放已经被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等；
  * 方法区中包含一个运行时常量池（我们一般在一个类中通过public static final来声明一个常量。这个类被编译后便生成Class文件，这个类的所有信息都存储在这个class文件中。当这个类被Java虚拟机加载后，class文件中的常量就存放在方法区的运行时常量池中。）
  * 线程私有

* **本地方法栈**

  * 与栈实现的功能类似，只不过本地方法区是本地方法运行的内存模型。
  * 本地方法被执行的时候，在本地方法栈也会创建一个栈帧，用于存放该本地方法的局部变量表、操作数栈、动态链接、出口信息。
  * 方法执行完毕后相应的栈帧也会出栈并释放内存空间。
  * 线程共享

* **程序计数器**

  * 当前线程正在执行的字节码的行号指示器；
  * 线程私有

* **直接内存**

  * 直接内存是除Java虚拟机之外的内存，但也有可能被Java使用。
  * 在NIO中引入了一种基于通道和缓冲的IO方式。它可以通过调用本地方法直接分配Java虚拟机之外的内存，然后通过一个存储在Java堆中的DirectByteBuffer对象直接操作该内存，而无需先将外面内存中的数据复制到堆中再操作，从而提升了数据操作的效率。
  * 直接内存的大小不受Java虚拟机控制，但既然是内存，当内存不足时就会抛出OOM异常。


