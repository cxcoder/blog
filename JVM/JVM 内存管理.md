JVM 通过 *垃圾收集-GC* 自动管理内存堆中对象内存的分配和回收。JVM 通常采用**分代垃圾收集器**，以便于整理内存碎片。分代垃圾收集器就是基于对象不同生命周期，将堆分成不同的内存区域，然后组合使用不同的垃圾收集算法，可简单认为分为两部分组成：

- *Young Generation*：年轻代，由*Eden*和两个相等的*Survivor*空间组成，其中一个Survivor始终为空，用来复制Minor GC后在Eden和另一个Survivor存活的对象。
- *Old Generation*：老年代，对象生命周期比较长。


## 内存回收

内存回收主要考虑两个问题：

- **如何判断对象可被回收**，判断策略：

    - *Tracing GC*，跟踪收集，也叫**可达性分析算法**，其思想是从某些**根对象引用(GC roots)**出发总能找到一个到一组存活对象的引用链。
    - *Reference counting*，**引用计数法**，不能解决循环引用。
    - *Escape analysis*，**逃逸分析**，可以将堆分配转为栈分配，动态编译优化手段，减轻GC压力。

- **采用何种方式进行回收**，垃圾收集算法：

    - *Copying*，复制，将存活对象从一块内存复制到另一块内存，不产生内存碎片，由于要保留一个空的备份内存，所以空间利用率较低。
    - *Mark-Sweep*，标记清理，从GC roots出发标记所有存活对象，然后清理所有未标记的对象，会产生内存碎片。
    - *Mark-Compact*，标记整理，标记清除后，会压缩内存，避免内存碎片。


## 内存分配

对象实例首先在 Eden 区分配，为了快速分配，Hotspot JVM 采用的是一种 *bump-the-pointer* 的**线性分配**方法。分配时一般都有大块连续内存可用，此方法就是检查剩余内存是否足够，给对象分配内存，然后更新指针偏移量和初始化对象。


线性分配效率固然高，但对多线程程序来说，分配内存的操作必须是线程安全的。可以使用全局锁但会影响性能，Hotspot JVM 采用的是一种 *Thread-Local Allocation Buffers (TLABs)* 的方法，为每个线程分配一个缓冲区，当TLAB满了，再加锁去申请，在线程内部就能使用*bump-the-pointer*，进而提高分配的吞吐量。


分配内存时，一些大对象有可能直接在老年代分配，在年轻代经过几轮Minor GC存活，达到一定年龄的对象，会被提升复制到老年代。


## Hotspot JVM

Hotspot JVM 自JDK 6u23开始支持逃逸分析，采用 *Tracing GC* 追踪堆中存活对象，显然，GC roots就是堆外的**对象引用**：

- 栈帧中局部变量或方法参数的对象引用
- 类引用类型静态成员变量
- JNI 本地方法局部变量，参数和JNI 全局引用


需要注意*GC roots*是一组**对象引用**而不是**引用对象**。


Hotspot JVM 提供了多个垃圾收集器让分代收集器组合使用：

- **串行收集器**，单个GC线程执行所有垃圾收集工作
    - *Serial*，年轻代，使用*复制算法*，*DefNew*
    - *Serial Old*，老年代，使用*标记压缩整理算法*，*Tenured*

- **并行收集器**，吞吐量收集器，多个GC线程加速垃圾回收
    - *ParNew*，年轻代，使用*复制算法*，可看做并行的Serial，*ParNew*
    - *Parallel Scavenge*，年轻代，使用*复制算法*，与 ParNew 的区别，它可以动态调节最大停顿时间和吞吐量，*PSYoungGen*
    - *Parallel Old*，老年代，使用*标记压缩整理算法*，通常与*Parallel Scavenge*配合使用，*ParOldGen*

- **并发收集器**，看重响应时间而不是吞吐量
    - *CMS, Concurrent Mark Sweep*，低停顿，采用*标记清除算法*，会有内存碎片，Java堆空间需求比较大，*CMS*
    - *G1, Garbage-First*，相比CMS，压缩内存，*G1*

常用JVM参数指定GC组合：
- -XX:+UseSerialGC：Serial + Serial Old
- -XX:+UseParNewGC：ParNew + Serial Old
- -XX:+UseParallelGC：Parallel Scavenge + Serial Old
- -XX:+UseParallelOldGC：Parallel Scavenge + Parallel Old
- -XX:+UseConcMarkSweepGC：ParNew + CMS + Serial Old
- -XX:+UseG1GC：G1


### 64位JVM 默认配置，以JDK 8为例

使用命令`java -XX:+PrintFlagsFinal -version`或者`jmap -heap <pid>`可以查看默认配置。


JVM堆最小为物理内存的 1/64，64位至少2G，即最小32M；最大为物理内存的 1/4，如128G内存，默认JVM最大32G，可以使用`-Xms`和`-Xmx`指定初始和最大大小。年轻代默认比例为`NewRatio=2`，即占总堆的 1/3，可使用`-Xmn`指定大小，Survivor空间比例为`SurvivorRatio=8`，即每个Survivor占Eden的 1/8，因为有两个，所以占年轻代的 1/10。


JDK 8 默认垃圾收集器是ParallelOldGC，其中Parallel Scavenge默认打开AdaptiveSizePolicy，自适应调整各种参数，默认的SurvivorRatio 配置需要手动指定才能生效。比如指定堆大小为270M，那么各区域大小如下：

![JVM Heap](https://github.com/cxcoder/blog/blob/master/Image/Heap.png)

图 1 并行垃圾收集，指定SurvivorRatio=8，堆空间大小

### 永久代, Permanent Generation

JVM 中方法区的实现，不同版本之间存储内容存在差别：

- JDK 6：类元数据、类静态变量、intern字符串
- JDK 7：将intern字符串移到堆中
- JDK 8：移除永久代，新增一个元空间存储类元数据，将类静态变量和intern字符串移到堆中


永久代逻辑上是堆的组成部分，64位JVM默认大小为82M，但最大大小难以估计，因为程序里面有多少类，有多少方法以及常量池大小都很难估算，此外永久代垃圾收集与老年代绑定，任一区域满了都会触发Full GC，所以对永久代的调优很困难。


在JDK 8中，移除永久代，类元数据分配在本地内存中，默认情况下，可用内存不受限制，可使用`MaxMetaspaceSize`设置可用的内存上限。Hotspot JVM 显式管理这部分内存，从OS申请空间，然后分成块，一个块绑定到一个特定的类加载器，类元数据就在这些块中分配，当类卸载或加载器标记被回收，这些块被释放重用或返回OS。


## 小结

JDK 8中 GC cause有：年轻代内存分配失败，引起的Minor GC；年轻代提升到老年代，而老年代空间不足，引起的Full GC；元数据空间达到阈值，引起的GC。JDK 7之前，由于永久代空间不足引起的Full GC。不管是永久代或者元数据空间，也都会存在内存泄漏，比如web应用，当应用程序被卸载，那么它的war包中的所有类都是垃圾，如果不移除就会有内存泄漏。


在分析内存回收时，一定要谨记 GC roots 是一组对象引用或者说指针，拿废弃常量来说，完全可以通过GC root来识别，进而回收。


## 参考

- [Memory Management Whitepaper](http://www.oracle.com/technetwork/java/javase/tech/memorymanagement-whitepaper-1-150020.pdf)
- [Our Collectors](https://blogs.oracle.com/jonthecollector/entry/our_collectors)
- [HotSpot Virtual Machine Garbage Collection Tuning Guide](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)
- [JEP 122: Remove the Permanent Generation](http://openjdk.java.net/jeps/122)
- [Java HotSpot VM Options](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
- [String.intern in Java 6, 7 and 8 – string pooling](http://java-performance.info/string-intern-in-java-6-7-8/)
- [Java永久代去哪儿了](http://www.infoq.com/cn/articles/Java-PERMGEN-Removed)
