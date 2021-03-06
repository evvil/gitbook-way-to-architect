## 判断对象的状态

---

在垃圾收集器对堆进行回收前，要先确定对象的状态：是还存活着，还是已经死去。

* **引用计数算法**  
  做法：给对象一个引用计数器，每当有对象引用它时，计数器就加1；当引用失效时，计数器就减1；任何时刻计数器为0的对象就是不可能再被使用的。  
  引用计数算法实现简单，判定效率高，在大部分情况下都是一个不错的算法。但主流的Java虚拟机里都没有选用该算法来管理内存：它很难解决对象之间的相互循环引用的问题。

* **可达性分析算法**  
  该算法的基本思想是：通过一系列的称为`GC Roots`的对象作为起始点，从这些节点向下搜索，搜索所走过的路径称为引用链（`Reference Chain`）。当一个对象到`GC Roots`没有任何引用链相连（从`GC Roots`到这个对象不可达），则证明该对象不可用。  
  在Java中，可作为`GC Roots`的对象包括下面几种：

  * 虚拟机栈（栈帧中的本地变量表）中引用的对象（即当前正在运行的代码用到的对象）
  * 方法区中静态属性引用的对象
  * 方法区中常量引用的对象
  * 本地方法栈中JNI（一般说的Native方法）引用的对象

* **方法区的回收**  
  方法区中垃圾回收的性价比较低（很少可以被回收）。HotSpot虚拟机中方法区使用永久代来实现的。  
  永久代的垃圾回收主要包括两部分：废弃常量和无用的类。  
  对于废弃常量的判断与Java堆中的对象判断类似。  
  而类则要同时满足下面的3个条件才能算是无用的类：  
  a.该类的所有实例都已经被回收，也就是堆中不存在该类的任何实例  
  b.加载该类的ClassLoader已经被回收  
  c.该类对应的`java.lang.Class`对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。  
  满足上面3个条件后，并不一定会回收，只是可以回收了。是否对类进行回收，HotSpot提供了-Xnoclassgc参数进行控制。

---

### 二、垃圾收集算法

* **标记-清除算法**  
  该算法分为两个阶段：标记和清除，首先标记出所有需要回收的对象，在标记完成之后，统一回收所有被标记的对象。  
  不足之处：一是效率问题，标记和清除两个过程的效率都不高；二是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后再程序中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾回收动作。

* **复制算法**  
  复制算法将内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块内存用完，就将还存活的对象复制到另一块上，然后再把已使用的内存空间一次清理掉。这样，每次回收只是针对整个半区，且不会产生内存碎片问题。  
  该算法实现简单，运行高效，但代价是将内存缩小了一半。  
  现在商业虚拟机均采用该算法对新生代进行回收，但并不按照1：1的比例划分内存。而是将内存分为一块较大的Eden空间和两块较小的Survivor空间。每次使用Eden空间和其中一块Survivor空间。当回收时，将Eden空间和Survivor空间中还存活的对象复制到另一块Survivor空间中，最后清理掉Eden空间和之前用过的Survivor空间。（**适用于对象存活率较低的场景**）  
  HotSpot中默认`Eden：Survivor：Survivor = 8：1：1`。  
  如果在回收时，有大于10%时的对象都存活，即预留的Survivor空间不够用，就要依赖其他内存（老年代）进行分配担保：直接进入老年代。

* **标记-整理算法**  
  针对老年代的一种算法。该算法在标记阶段与标记-清除算法一致，但后续并不对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

* **分代收集算法**  
  顾名思义，根据对象的存活周期的不同将内存划分为几块，一般是将堆分为新生代和老年代，然后根据各个年代的特点采用最适当的收集算法。  
  在新生代，每次垃圾回收时都发现有大批对象死去，只有少量存活，就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。  
  在老年代，由于对象存活率高、没有额外空间对它进行分配担保，就必须使用标记-清除或者标记-整理算法进行回收。

## 垃圾收集器

---

收集算法是内存回收的方法论，而垃圾收集器则是内存回收的具体实现。没有万能的收集器，没有最好的收集器，只是对具体应用选择最合适的收集器。

* **Serial收集器**

  * 单线程的新生代收集器，在它进行垃圾收集时，必须暂停其他所有工作线程，直到它收集结束。
  * 采用复制算法
  * 简单而高效（与其他收集器的单线程比），对于限定的单个CPU的环境下，没有线程交互的开销，专心做垃圾回收操作。
  * 是虚拟机运行在Client模式下的默认新生代收集器。

* **ParNew收集器**

  * 其实就是Serial收集器的多线程版本，其他与Serial收集器完全一样（比如控制参数、收集算法、Stop The World、对象分配规则、回收策略） 。默认开启的收集线程与CPU数量相同，在CPU非常多的环境下，可使用`-XX:ParallelGCThreads`参数来限制垃圾收集的线程数。
  * 是很多运行在Server模式下的虚拟机中首选的新生代收集器：除了Serial收集器，只有它能与CMS收集器配合工作。它也是使用CMS收集器时默认的新生代收集器。

* **Parallel Scavenge收集器**

  * 新生代收集器，使用复制算法，并行的多线程收集器。
  * 该收集器的目标是达到一个可控制的吞吐量（CPU用于运行用户代码的时间与CPU总消耗时间的比值），`吞吐量=运行用户代码时间/（运行用户代码时间 + 垃圾收集时间）`，被称为吞吐量优先的收集器。
  * 高吞吐量可以高效率地利用CPU时间，尽快完成程序的运算任务，适合在后台运算而不需要太多交互的任务。
  * 设定两个参数来控制吞吐量：最大垃圾收集停顿时间`-XX:MaxGCPauseMills`和吞吐量大小`-XX:GCTimeRatio`。
  * **自适应调节策略**：通过参数`-XX:+UseAdaptiveSizePolicy`，开启之后，就不需要手工指定新生代大小、Eden与Survivor区的比例、晋升老年代对象的大小等细节参数，虚拟机会根据当前系统运行情况进行GC自适应调节。

* **Serial Old收集器**

  * Serial收集器的老年代版本，单线程，使用标记-整理算法。
  * 主要意义是在给定Client模式下的虚拟机使用。在Server模式下，有两种用途：①在JDK1.5以及之前的版本中与Parallel Scavenge收集器搭配使用；②作为CMS收集器的后备预案，在并发收集发生`Concurrent Mode Failure`时使用。

* **Parallel Old收集器**

  * Parallel Scavenge收集器的老年代版本，多线程，使用标记-整理算法。
  * JDK1.6中才开始提供该收集器。在此之前，新生代如果选择了Parallel Scavenge收集器，老年代只能选择Serial Old收集器。Parallel Old收集器的出现，使得“吞吐量优先收集器”有了比较名副其实的组合：`Parallel Scavenge + Parallel Old`。

* **CMS收集器**

  * 老年代收集器 
  * CMS（`Concurrent Mark Sweep`）收集器是一种获取最短回收停顿时间为目标的收集器。优点是**并发、低停顿**。在互联网网站或者B/S系统的服务端上的Java应用，注重服务的响应时间，希望系统停顿时间最短，以带给用户较好的体验，CMS收集器非常适合这类应用的需求。
  * **基于标记-清除算法**，运作过程分为4个步骤：  
    a.**初始标记**：需要Stop The World，标记GC Roots能直接关联到的对象，速度很快。  
    b.**并发标记**：与用户程序一起运行，进行GC Roots Tracing的过程。  
    c.**重新标记**：需要Stop The World，修正并发标记阶段因用户程序继续运作而导致标记产生变化的那一部分对象的标记记录。该阶段的停顿时间比初始阶段稍长，但远小于并发标记的停顿时间。  
    d.**并发清除**：与用户程序一起运行，进行清除工作。

  * CMS远达不到完美的程度，它有4个明显的缺点：  
    a.**对CPU非常敏感**：因占用CPU资源导致应用程序变慢，总吞吐量降低。CMS默认的垃圾收集线程数为（CPU数量 + 3）/4。当CPU在4个以上时，垃圾回收线程会至少占用25%的CPU资源，并会随着CPU数量增加而下降。当CPU不足4个时，CMS对用户程序的影响可能就非常大。  
    b.**CMS无法处理浮动垃圾**。由于CMS并发清理阶段用户线程还在运行，可能会出现新的垃圾，CMS无法在本次收集时处理它们，只能等下一次GC再清理。这部分垃圾就是浮动垃圾。  
    c.由于并发收集，在收集时还要预留内存空间供用户程序使用，因此CMS收集器无法像其他收集器一样等到老年代几乎被完全填充时再进行收集。在JDK1.5的默认设置下，当老年代使用了68%的时候，就会进行垃圾收集。如果应用中老年代增长不快，可适当提高该值。JDK1.6将该值提升到92%。但不能设置的太高：如果在CMS运行期间预留的内存空间无法满足程序需要，就会出现`Concurrent Mode Failure`，这时虚拟机将会启动后备预案：临时启用Serial Old收集器重新进行老年代的垃圾收集，导致停顿时间很长。可以通过参数`-XX:CMSInitiatingOccupancyFraction`设置该值。  
    d.基于标记-清除算法，容易造成内存碎片，往往出现老年代还有很大空间剩余，但无法找到足够大的连续空间来分配当前对象，不得不出发一次Full GC。可以通过参数`-XX:UseCMSCompactAtFullCollection`（默认开启），用于在CMS顶不住要进行一次FullGC时开启内存碎片整理过程（该过程无法并发，需要停顿）。还可以使用参数`-XX:CMSFullGCsBeforeCompaction`设定执行多少次不压缩的FullGC后，跟着来一次带压缩的（默认为0，即每次进入FullGC时都进行碎片压缩）。

* **G1收集器**

  * Garbage-First是当今收集器技术发展的最前沿成果之一。
  * G1是一款面向服务器端应用的垃圾收集器，相比于CMS，优点是：
    ①并行与并发：缩短Stop The World的时间；
    ②分代收集：将堆划分为多个大小相等的独立区域Region，虽然还保留新生代和老年代的概念，但新生代和老年代不再是物理隔离了，他们都是一部分Region的集合（不需要连续）；
    ③空间整合：从整体上看基于标记-整理算法，从局部（两个Region）看是基于复制算法，不会产生内存碎片；
    ④可预测的停顿：可以指定在一个长度为M毫秒的时间段内，消耗在垃圾收集上的时间不得超过N毫秒。

## 内存分配与回收策略

---

对象的内存分配，就是在堆上分配，对象主要分配在新生代的Eden区域，如果启动了本地线程分配缓冲，将按线程优先在TLAB上分配。少数情况也可能会直接分配在老年代中。  
分配的规则不是百分百固定的。其细节取决于当前使用的是哪一种垃圾收集器组合，还有虚拟机中与内存相关的参数的配置。  
最普遍的内存分配规则有以下几点：

* **对象优先在Eden分配**

* **大对象直接进入老年代**  
  通过设置参数`-XX:PretenureSizeThreshold`参数，令大于该设置值的对象直接在老年代分配。这样做的目的是为了避免在Eden区和两个Suvivor区之间发生大量的内存复制。

* **长期存活的对象将进入老年代**  
  虚拟机给每个对象定义一个对象年龄计数器（在对象头中）。如果对象在Eden出生并且经过第一次Monior GC后仍然存活，并且被Survivor容纳的话，就将该对象的年龄设为1。对象在Survivor区中每熬过依次Monior GC，年龄就增加1。当该对象的年龄增加到一定程度（默认为15），就会进入老年代。可以通过参数`-XX:MaxTenuringThreshold`设置年龄阈值。

  * 动态对象年龄判定
    如果在Survivor区间中相同年龄的所有对象大小的总和大于Survivor空间的一半，年龄&gt;=该年龄的对象就可以直接进入老年代，无需等到参数`-XX:MaxTenuringThreshold`设置的年龄。

* **空间分配担保**  
  在发生Monior GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果大于，则Monior GC可以确保是安全的。如果小于，则虚拟机会查看`HandlePromotionFailure`设置值是否允许担保失败，如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，就尝试进行一次Monior GC（有风险），如果小于或者`HandlePromotionFailure`设置不允许冒险，则此时改为进行一次Full GC。

![空间分配担保.png](http://upload-images.jianshu.io/upload_images/1932449-4f29db4475552299.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)

---

概念补充

> **Minor GC**：清理年轻代（包括 Eden 和 Survivor 区域），所有的 Minor GC 都会触发stop-the-world。  
> **Major GC**：清理老年代。  
> **Full GC**：清理整个堆空间—包括年轻代和老年代。  
> 参考自：[Minor GC、Major GC和Full GC之间的区别](http://www.importnew.com/15820.html)

---

内容摘抄自《深入理解Java虚拟机》

