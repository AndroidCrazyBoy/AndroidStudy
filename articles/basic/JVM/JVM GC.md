# Java 虚拟机垃圾回收机制



#### **概述**

垃圾回收是一种自动的存储管理机制。 当一些被占用的内存不再需要时，就应该予以释放，以让出空间，这种存储资源管理，称为垃圾回收（Garbage Collection）。 垃圾回收器可以让程序员减轻许多负担，也减少程序员犯错的机会。



#### **JVM判断对象是否存活的算法**

GC 的存活标准知道哪些区域的内存需要被回收之后，我们自然而然地想到了，如何去判断一个对象需要被回收呢？对于如何判断对象是否可以回收，有两种比较经典的判断策略。

- 引用计数算法

> 一个对象被创建之后，系统会给这个对象初始化一个引用计数器，当这个对象被引用了，则计数器 +1，而当该引用失效后，计数器便 -1，直到计数器为 0，意味着该对象不再被使用了，则可以将其进行回收了。
> 这种算法其实很好用，判定比较简单，效率也很高，但是却有一个很致命的缺点，就是它无法避免循环引用，即两个对象之间循环引用的时候，各自的计数器始终不会变成 0，所以 引用计数算法 只出现在了早期的 JVM 中，现在基本不再使用了。

- 可达性分析算法

> 根搜索算法的中心思想，就是从某一些指定的根对象（GC Roots）出发，一步步遍历找到和这个根对象具有引用关系的对象，然后再从这些对象开始继续寻找，从而形成一个个的引用链（其实就和图论的思想一致），然后不在这些引用链上面的对象便被标识为引用不可达对象，也就是我们说的“垃圾”，这些对象便需要回收掉。这种算法很好地解决了上面 引用计数算法 的循环引用的问题了。



#### GC Root主要包括以下几类元素

**1、虚拟机栈中引用的对象**
比如：各个线程被调用的方法中使用到的参数、局部变量等。

**2、本地方法栈内JNI（通常说的本地方法）引用的对象**

**3、方法区中类静态属性引用的对象**
比如：Java类的引用类型静态变量

**4、方法区中常量引用的对象**
比如：字符串常量池（string Table） 里的引用

**5、所有被同步锁synchronized持有的对象**

**6、Java虚拟机内部的引用。**
基本数据类型对应的Class对象，一些常驻的异常对象（如：
NullPointerException、OutOfMemoryError） ，系统类加载器。

**7、反映java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等**

**8、除了这些固定的GCRoots集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象“临时性”地加入，共同构成完整GC Roots集合。比如：分代收集和局部回收（Partial GC）。**
如果只针对Java堆中的某一块区域进行垃圾回收（比如：典型的只针对新生代），必须考虑到内存区域是虚拟机自己的实现细节，更不是孤立封闭的，这个区域的对象完全有可能被其他区域的对象所引用，这时候就需要一并将关联的区域对象也加入GC Roots集
合中去考虑，才能保证可达性分析的准确性。

**小技巧：由于Root采用栈方式存放变量和指针，所以如果一个指针，它保存了堆内存里面的对象，但是自己又不存放在堆内存里面，那它就是一个Root**



#### **常用回收算法**

  标记-清除  ( mark-sweep )

```
优点：开销低，速度快
缺点： 原地清理所以无法避免碎片问题
```

  标记-复制  ( mark-copy )

```
优点：GC后的内存空间是连续的缺点： 可用内存空间减半
```

  标记-整理  ( mark-compact)

```
优点：无碎片问题，内存空间可用大小不减半
缺点：效率低，开销大
```

  引用-计数  (reference counting)

```
前三种垃圾回收算法都是间接式的，它们都需要从已知的根集合出发对存活对象图进行遍历，进而才能确定所有的存活对象。
在引用计数中，对象的存活性可以通过引用关系的创建或删除直接判定，从而无须像追踪式回收器那样先通过堆遍历找出所有的存活对象，然后再反向确定出未遍历的垃圾对象。

优点：直接遍历，速度快
缺点：无法解决环形引用问题
```



#### **分代回收**

![jvm_gc_heap_structure.jpeg](https://github.com/AndroidCrazyBoy/AndroidStudy/blob/main/resource/images/JVM/jvm_gc_heap_structure.jpeg?raw=true)

- **新生代**

  * 所有新 new 出来的对象都会最先出现在新生代中，当新生代这部分内存满了之后，就会发起一次垃圾收集事件，这种发生在新生代的垃圾收集称为 Minor collections。 这种收集通常比较快，因为新生代的大部分对象都是需要回收的，那些暂时无法回收的就会被移动到老年代。

    全局暂停事件（Stop the World）：所有小收集（minor garbage collections）都是全局暂停事件，也就是意味着所有的应用线程都需要停止，直到垃圾回收的操作全部完成。类似于“你妈妈在给你打扫房间的时候，肯定也会让你老老实实地在椅子上或者房间外待着，如果她一边打扫，你一边乱扔纸屑，这房间还能打扫完？”

  - 新生代（Young Generation）的回收算法

    - 所有新生成的对象首先都是放在新生代的。新生代的目标就是尽可能快速的收集掉那些生命周期短的对象。
    - 新生代内存按照`8:1:1`的比例分为一个 **eden区和两个survivor(survivor0,survivor1)** 区。大部分对象在Eden区中生成，回收时先将eden区存活对象复制到一个`survivor0`区，然后清空eden区。当这个survivor0区也存放满了时，则**将eden区和survivor0区存活对象复制到另一个survivor1区**，然后清空eden和这个survivor0区，此时survivor0区是空的然后**将survivor0区和survivor1区交换**，即保持survivor1区为空，如此往复。
    - 当survivor1区不足以存放 eden和survivor0的存活对象时，就将存活对象直接存放到老年代。若是老年代也满了就会触发一次`Full GC`，也就是新生代、老年代都进行回收。
    - 新生代发生的GC也叫做`Minor GC`，MinorGC发生频率比较高(不一定等Eden区满了才触发)。

    

- **老年代**

  * 老年代用来存储那些存活时间较长的对象。 一般来说，我们会给新生代的对象限定一个存活的时间，当达到这个时间还没有被收集的时候就会被移动到老年代中。随着时间的推移，老年代也会被填满，最终导致老年代也要进行垃圾回收。这个事件叫做大收集(major garbage collection)。

    大收集也是全局暂停事件。通常大收集比较慢，因为它涉及到所有的存活对象。所以，对于对相应时间要求高的应用，应该将大收集最小化。此外，对于大收集，全局暂停事件的暂停时长会受到用于老年代的垃圾回收器的影响。

  - 老年代（Old Generation）的回收算法
    - 在新生代中经历了N次垃圾回收后仍然存活的对象，就会被放到老年代中。因此，可以认为**老年代中存放的都是一些生命周期较长的对象**。
    - 内存比新生代也大很多(大概比例是1:2)，当老年代内存满时触发`Major GC`即`Full GC`，`Full GC`发生频率比较低，老年代对象存活时间比较长，存活率标记高。

- **空间担保**

  * 在发生Minor GC之前，虚拟机会**检查老年代最大可用的连续空间是否大于新生代所有对象的总空间**。

    - 如果大于，则此次Minor GC是安全的
    - 如果小于，则虚拟机会查看-X:HandlePromotionFailure设置值是否允许担保失败。如果HandlePromotionFailure=true,那么会继续检查老年代最大可用连续空间是否大于历次晋升到老年代的对象的平均大小。
       ————>  如果大于，则尝试进行一次Minor GC,但这次Minor GC依然是有风险的：
       ————>  如果小于，则改为进行一次Full GC。
       如果HandlePromotionFailure=false,则改为进行一次Full GC。

    JDK6 Update 24之后的规则变为**只要老年代的连续空间大于新生代对象总大小**或者**大于历次晋升的平均大小就会进行Minor GC**,否则将进行Full GC。

- **动态对象年龄判断**

  * 如果Survivor区中相同年龄的所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

* **gc流程图**

![jvm_gc.png](https://github.com/AndroidCrazyBoy/AndroidStudy/blob/main/resource/images/JVM/jvm_gc.png?raw=true)



#### **常见垃圾搜集器**

* **新生代收集器：** Serial、ParNew、Parallel Scavenge

* **老年代收集器：** Serial Old、CMS、Parallel Old

* **新生代和老年代收集器：** G1、ZGC、Shenandoah

每种垃圾回收器之间不是独立操作的，下图表示垃圾回收器之间有连线表示，可以协作使用：

![jvm_gc_collector.png](https://github.com/AndroidCrazyBoy/AndroidStudy/blob/main/resource/images/JVM/jvm_gc_collector.png?raw=true)

**Serial**

```
单线程串行  复制算法
```

**Serial Old**

```
老年代的收集器，与Serial一样是单线程，不同的是算法用的是标记-整理（Mark-Compact）
```

**Parallel Scavenge**

```
Serial 收集器的多线程版本，并行收集器。 复制算法
```

**Parallel Old**

```
老年代的收集器，是Parallel Scavenge老年代的版本。其中的算法替换成 Mark-Compact。
```

**ParNew**

```
跟Parallel类似，专门为了配合cms使用。 复制算法。 新生代并行收集器。
```

**CMS**

```
concurrent mark sweep 并发标记清除，以获取最短回收停顿时间为目标。 老年代并发回收器。
```



#### **GC时为什么要暂停用户线程(Stop the world)？**

首先，如果不暂停用户线程，就意味着期间会不断有垃圾产生，永远也清理不干净。
其次，用户线程的运行必然会导致对象的引用关系发生改变，这就会导致两种情况：漏标和错标。

**漏标**

```
原本不是垃圾，但是GC的过程中，用户线程将其引用关系修改，导致GC Roots不可达，成为了垃圾。这种情况还好一点，无非就是产生了一些浮动垃圾，下次GC再清理就好了。
```

**错标**

```
原本是垃圾，但是GC的过程中，用户线程将引用重新指向了它，这时如果GC一旦将其回收，将会导致程序运行错误。
```



#### **三色标记算法**

为什么CMS的GC线程可以和用户线程一起工作 ? 采用三色标记解决。

缺点：

![jvm_gc_cms_disadvantages.png](https://github.com/AndroidCrazyBoy/AndroidStudy/blob/main/resource/images/JVM/jvm_gc_cms_disadvantages.png?raw=true)



#### **一张图让你看懂JVM之垃圾回收算法详解**

![jvm_gc_all.jpeg](https://github.com/AndroidCrazyBoy/AndroidStudy/blob/main/resource/images/JVM/jvm_gc_all.jpeg?raw=true)

 #### 四大引用

**SoftReference**： 垃圾回收器会根据内存需求酌情回收软引用指向的对象。普通的 GC 并不会回收软引用，只有在即将 OOM 的时候(也就是最后一次 Full GC)如果被引用的对象只有 SoftReference 指向的引用，才会回收。

**WeakReference**： 当发生 GC 时，如果被引用的对象只有 WeakReference 指向的引用，就会被回收。

**PhantomReference**： 其是一种特殊的引用类型，不能通过虚引用获取到其关联的对象，但当 GC 时如果其引用的对象被回收，这个事件程序可以感知，这样我们可以做相应的处理。

**Reference 的核心处理流程**， JVM 在 GC 时如果当前对象只被 Reference 对象引用，JVM 会根据 Reference 具体类型与堆内存的使用情况决定是否把对应的 Reference 对象加入到一个由 Reference 构成的 pending 链表上，如果能加入 pending 链表 JVM 同时会通知 ReferenceHandler 线程进行处理。ReferenceHandler 线程收到通知后会调用 Cleaner#clean 或 ReferenceQueue#enqueue 方法进行处理。如果引用当前对象的 Reference 类型为 WeakReference 且堆内存不足，那么 JVM 就会把 WeakReference 加入到 pending-Reference 链表上，然后 ReferenceHandler 线程收到通知后会异步地做入队列操作。而我们的应用程序中的线程便可以不断地去拉取 ReferenceQueue 中的元素来感知 JVM 的堆内存是否出现了不足的情况，最终达到根据堆内存的情况来做一些处理的操作。



##### 参考文章

[GC过程中需要stop the world的原因是什么](https://www.yisu.com/zixun/541057.html)

[Java Reference 核心原理分析](https://xie.infoq.cn/article/d3420eea3b45b8499d8505a95)