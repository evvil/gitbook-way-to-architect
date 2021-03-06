仅讨论普通Java对象（不包括数组和Class对象）的创建。

## 对象的创建

---

主要分为以下几个步骤

###### 1、检查对象对应的类是否加载

虚拟机遇到new指令的时候，先检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查该符号引用代表的类是否已经被加载、解析和初始化。如果没有，则先执行类的加载过程。

###### 2、为新生对象分配内存

对象所需要的内存大小在类加载完成后便可以完全确定，此时需要从运行时数据区域中的堆中将特定大小的内存划分出来，给该对象。

* **内存划分（两种方式）**

* * **指针碰撞** 假设堆中内存绝对规整，用过的内存放一边，没用过的放一边，中间用一个指针作为分界点的指示器。分配内存仅仅是将指针向空闲空间那边挪动一段与对象大小相等的距离。
* * **空闲列表** 假设堆中内存不规整，已使用的和空闲的内存相互交错，此时，虚拟机维护一个列表，用来记录哪些内存可用。在分配内存时，从列表中找到一块足够大的空间划分给该对象实例，并更新列表上的记录。 到底使用哪种分配方式，与Java堆是否规整有关，而Java堆是否规整与所选用的垃圾收集器是否带有压缩整理功能决定。如果使用Serial、ParNew等带有Compact过程的收集器时，系统采用指针碰撞方式。如果使用CMS这种基于Mark-Sweep算法的收集器时，采用空闲列表方式。
* **划分时保证线程安全（两种方式）**

* * **对分配内存空间的动作进行同步处理** 虚拟机实际上采用CAS+失败重试的方式保证更新操作的原子性。
* * **把内存分配的动作按照线程划分在不同的空间上** 即每个线程在堆中预先分配一小块内存区域，称为本地线程分配缓冲（`Thread Local Allocation Buffer`，TLAB）。哪个线程要分配内存，就在哪个线程的TLAB上分配，只有当TLAB用完并分配新的TLAB时，才需要同步锁定。虚拟机是否使用TLAB，可以通过`-XX:+/-UseTLAB`参数设定。

###### 3、内存空间初始化

将分配到的内存空间都初始化为零值（不包括对象头），这一步操作保证了对象的实例字段在Java代码中可以不赋初值就可以直接使用，程序能够访问到这些字段的数据类型所对应的零值。

```java
public class ObjectInit {
    private String s;
    private int a;

    @org.junit.Test
    public void test(){
        int b;
        String s1;
        System.out.println(s);  ---1
        System.out.println(a);-----2
        //System.out.println(b);----3
        //System.out.println(s1);---4
    }
}
```

上面的代码可以正常运行，最终打印出null和0。假如将标注3、4的两句代码去掉注释，此时运行test\(\)方法，就会提示：~~java:可能尚未初始化变量b和s1~~。

###### 4、对象设置

例如该对象是哪个类的实例、如何才能找到类的元数据信息、对象的Hash码、对象的GC分代年龄信息等。这些信息存放在对象的对象头之中。

###### 5、对象初始化

上面的工作完成之后，一个新的对象已经产生了，但是所有的字段都还为零。一般在执行new指令之后，会接着执行&lt;init&gt;方法，把对象按照程序员的设定进行初始化。此时，一个真正可用的对象完全产生。

## 对象的内存布局（HotSpot虚拟机而言）

---

对象在内存中存储的布局可以分为3块区域：对象头、实例数据、对齐填充。

**对象头Header**

对象头包括两部分信息：Mark Word和类型指针

* **Mark Word**
  用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。这些信息是与对象自身定义的数据无关的额外的存储成本，在32位和64位的虚拟机中，该部分数据长度分别为32bit和64bit。
  根据对象的不同的状态，Mark Word中存储的内容分别如下所示：

| 状态 | 存储内容 | 标志位 |
| :--- | :--- | :--- |
| 未锁定 | 对象哈希码、对象分代年龄 | 01 |
| 轻量级锁定 | 指向锁记录的指针 | 00 |
| 重量级锁定 | 指向重量级锁的指针 | 10 |
| GC标记 | 空，不记录信息 | 11 |
| 可偏向 | 偏向线程ID、偏向时间戳、对象分代年龄 | 01 |

注：这里的锁定及偏向是属于线程竞争该对象时的概念

* **类型指针**  
  即对象指向它的类元数据的指针，虚拟机通过这个指针来确定对象是哪个类的实例。（并不是所有的虚拟机实现都必须在对象数据上保留类型指针，即查找对象实例的元数据信息并不一定要经过其本身）

**实例数据**  
实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的，还是在子类中定义的，都要记录下来。字段的存储顺序为（HotSpot默认）：`longs/doubles、ints、shorts/chars、bytes/booleans、oops(Ordinary Object Pointers)`。根据分配策略，相同宽度的字段总是被分配到一起。在满足这个前提的条件下，在父类中定义的变量总会出现在子类之前。如果CompactFields参数为true\(默认为true\)，那么子类中较窄的变量也可能会插入到父类的空隙之中。

**对齐填充**  
对齐填充并不是必然存在的，也没有特别的含有，仅仅起到占位的作用。

---

### 对象的访问定位

创建对象后，是通过栈上的reference数据来操作堆上的该具体对象。虚拟机规范中，reference类型数据为一个指向对象的引用，但该引用如何去定位、访问堆中的对象的具体位置则是由虚拟机实现而定的。  
目前主流的访问方式是使用句柄和直接指针两种。

* **使用句柄**
* * 堆中划分出一块内存作为句柄池，reference中存储的是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。
  * 这种访问方式的优势是，reference中存储的是稳定的句柄地址，在对象被移动后（比如垃圾收集时），只会改变句柄中的实例数据指针，reference本身不需要修改。
* **使用直接指针**
* * reference中存储的直接就是对象地址。
  * 使用这种访问方式的优势就是速度更快，节省了一次指针定位的时间开销。（HotSpot采用的这种方式）



内容摘抄自《深入理解Java虚拟机》

