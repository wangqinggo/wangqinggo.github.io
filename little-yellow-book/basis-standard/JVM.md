# JVM

## 架构

### 内存区域划分（todo）

### 运行时栈帧结构（todo）

## 类加载

- static 语句块，只能访问到定义在 static 语句块之前的变量

- JVM 会保证在子类的初始化方法执行之前，父类的初始化方法已经执行完毕

### 类加载器（todo）

#### 双亲委派机制

- 除了顶层的启动类加载器以外，其余的类加载器，在加载之前，都会委派给它的父加载器进行加载。这样一层层向上传递，直到祖先们都无法胜任，它才会真正的加载

#### 打破双亲委派机制

##### tomcat(加载自己定制的目录)（todo）

##### SPI

> 线程上下文的类加载器

##### OSGi

#### 如何替换 JDK 的类

- [endorsed 技术](https://developer.aliyun.com/article/330559)

## 垃圾回收算法

### 引用级别

#### 强引用

- 即使程序会异常终止，这种对象也不会被回收。这种引用属于最普通最强硬的一种存在，只有在和 GC Roots 断绝关系时，才会被消灭掉

#### 软引用

- 软引用用于维护一些可有可无的对象。在内存足够的时候，软引用对象不会被回收，只有在内存不足时，系统则会回收软引用对象，如果回收了软引用对象之后仍然没有足够的内存，才会抛出内存溢出异常。

- 这种特性非常适合用在缓存技术上

#### 弱引用

- 当 JVM 进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。弱引用拥有更短的生命周期，在 Java 中，用 java.lang.ref.WeakReference 类来表示。

- 它的应用场景和软引用类似，可以在一些对内存更加敏感的系统里采用

#### 虚引用

- 这是一种形同虚设的引用，在现实场景中用的不是很多。虚引用必须和引用队列（ReferenceQueue）联合使用。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。

- 虚引用主要用来跟踪对象被垃圾回收的活动。

### OOM 场景

|    区域    | 是否线程私有 | 是否会发生OOM |
| :-------: | :-------: | :------: |
| 程序计数器 |      是      |      否      |
|  虚拟机栈  |      是      |      是      |
| 本地方法栈 |      是      |      是      |
|  方法区  |      否      |      是      |
|  直接内存  |      否      |      是      |
|    堆    |      否      |      是      |

- 内存的容量太小了，需要扩容，或者需要调整堆的空间。

- 错误的引用方式，发生了内存泄漏。没有及时的切断与 GC Roots 的关系。比如线程池里的线程，在复用的情况下忘记清理 ThreadLocal 的内容。

- 接口没有进行范围校验，外部传参超出范围。比如数据库查询时的每页条数等。

- 对堆外内存无限制的使用。这种情况一旦发生更加严重，会造成操作系统内存耗尽。

### 朴素算法

#### 复制算法（Copy）

- 复制算法是所有算法里面效率最高的，缺点是会造成一定的空间浪费。

#### 标记-清除（Mark-Sweep）

- 效率一般，缺点是会造成内存碎片问题。

#### 标记-整理（Mark-Compact）

### [分代垃圾回收](https://juejin.im/post/6844903953004494856)

- https://github.com/cncounter/gc-handbook

#### 弱代假设

> 大部分对象的生命周期都很短，其他对象则很可能会存活很长时间

##### 分代垃圾回收

- 根据不同的对象生命周期选择不同的GC算法

  - 年轻代选择复制算法

  - 老年代选择标记-清除、标记-整理算法

#### [年轻代](https://tech.meituan.com/2017/12/29/jvm-optimize.html)

- 1 Eden + 2 Survivor

  - 比例 -XX:SurvivorRatio=8

- TLAB 的全称是 Thread Local Allocation Buffer，JVM 默认给每个线程开辟一个 buffer 区域，用来加速对象分配。

#### 老年代

- 提升（Promotion）

  - 参数‐XX:+MaxTenuringThreshold设置对象的年龄

- 分配担保

  - 当 Survivor 空间不够，就需要依赖其他内存（指老年代）进行分配担保

- 大对象直接在老年代分配

  - 超出某个大小的对象将直接在老年代分配

  - 通过参数 -XX:PretenureSizeThreshold 进行配置的。默认为 0，意思是全部首选 Eden 区进行分配。

- 动态对象年龄判定

  - 如果幸存区中相同年龄对象大小的和，大于幸存区的一半，大于或等于 age 的对象将会直接进入老年代

## 垃圾回收器

### 年轻代垃圾回收器

#### Serial 垃圾收集器Serial 垃圾收集器

- 处理 GC 的只有一条线程，并且在垃圾回收的过程中暂停一切用户线程

#### ParNew 垃圾收集器

- ParNew 是 Serial 的多线程版本。由多条 GC 线程并行地进行垃圾清理。清理过程依然要停止用户线程。

- ParNew：追求降低用户停顿时间，适合交互式应用。强交互弱计算。

#### Parallel Scavenge 垃圾收集器

- 另一个多线程版本的垃圾回收器

- Parallel Scavenge：追求 CPU 吞吐量，能够在较短时间内完成指定任务，适合没有交互的后台计算。弱交互强计算。

### 老年代垃圾收集器

#### Serial Old 垃圾收集器

- 与年轻代的 Serial 垃圾收集器对应，都是单线程版本

#### Parallel Old

- Parallel Old 收集器是 Parallel Scavenge 的老年代版本，追求 CPU 吞吐量。

#### **CMS 垃圾收集器**

- CMS（Concurrent Mark Sweep）收集器是以获取最短 GC 停顿时间为目标的收集器，它在垃圾收集时使得用户线程和 GC 线程能够并发执行，因此在垃圾收集过程中用户也不会感到明显的卡顿

- 它在年轻代使用复制算法，而对老年代使用标记-清除算法

##### CMS 回收过程

###### 初始标记（Initial Mark）（todo）

###### 并发标记（Concurrent Mark）（todo）

###### 并发预清理（Concurrent Preclean）

###### 并发可取消的预清理（Concurrent Abortable Preclean）

###### 最终标记（Final Remark）

###### 并发清除（Concurrent Sweep）（todo）

###### 并发重置（Concurrent Reset）

##### 内存碎片

- 部分空间预留

  - 一般在 30% 左右即可，那么能用的大概只有 70%。参数 -XX:CMSInitiatingOccupancyFraction 用来配置这个比例（记得要首先开启参数UseCMSInitiatingOccupancyOnly）。也就是说，当老年代的使用率达到 70%，就会触发 GC 了。如果你的系统老年代增长不是太快，可以调高这个参数，降低内存回收的次数。

- UseCMSCompactAtFullCollection（默认开启），表示在要进行 Full GC 的时候，进行内存碎片整理。内存整理的过程是无法并发的，所以停顿时间会变长

##### CMS 中都会有哪些停顿

- 初始标记，这部分的停顿时间较短；

- Minor GC（可选），在预处理阶段对年轻代的回收，停顿由年轻代决定；

- 重新标记，由于 preclaen 阶段的介入，这部分停顿也较短；

- Serial-Old 收集老年代的停顿，主要发生在预留空间不足的情况下，时间会持续很长；

- Full GC，永久代空间耗尽时的操作，由于会有整理阶段，持续时间较长。

##### 优势

- 低延迟，尤其对于大堆来说。大部分垃圾回收过程并发执行

##### 劣势

- 内存碎片问题。Full GC 的整理阶段，会造成较长时间的停顿。

- 需要预留空间，用来分配收集阶段产生的“浮动垃圾”。

- 使用更多的 CPU 资源，在应用运行的同时进行堆扫描。

##### 极端场景

- 在发生 Minor GC 时，由于 Survivor 区已经放不下了，多出的对象只能提升（promotion）到老年代。但是此时老年代因为空间碎片的缘故，会发生 concurrent mode failure 的错误。这个时候，就需要降级为 Serail Old 垃圾回收器进行收集。这就是比 concurrent mode failure 更加严重的 promotion failed 问题。

### G1垃圾回收器

#### 目标: 停顿时间可控

- -XX:MaxGCPauseMillis=10

- 不建议设置 -Xmn 或者 -XX:NewRatio去控制年轻代的比例

#### 内存结构

- 有 Eden 区和 Survivor 区的概念的，但内存不是连续的

- 固定大小的区域，名字叫作小堆区（Region）

  - -XX:G1HeapRegionSize=<N>M

- 大小超过 Region 50% 的对象会分配在Humongous Region

#### G1 的回收过程

- G1“年轻代”的垃圾回收，同样叫 Minor GC，这个过程和我们前面描述的类似，发生时机就是 Eden 区满的时候

- 老年代的垃圾收集，严格上来说其实不算是收集，它是一个“并发标记”的过程，顺便清理了一点点对象

- 真正的清理，发生在“混合模式”，它不止清理年轻代，还会将老年代的一部分区域进行清理

#### RSet

- 全称是 Remembered Set，用于记录和维护 Region 之间的对象引用关系

- RSet 与 Card Table 有些不同的地方。Card Table 是一种 points-out（我引用了谁的对象）的结构。而 RSet 记录了其他 Region 中的对象引用本 Region 中对象的关系，属于 points-into 结构（谁引用了我的对象），有点倒排索引的味道

- RSet 通常会占用很大的空间，大约 5% 或者更高。不仅仅是空间方面，很多计算开销也是比较大的

- 事实上，为了维护 RSet，程序运行的过程中，写入某个字段就会产生一个 post-write barrier 。为了减少这个开销，将内容放入 RSet 的过程是异步的，而且经过了很多的优化：Write Barrier 把脏卡信息存放到本地缓冲区（local buffer），有专门的 GC 线程负责收集，并将相关信息传给被引用 Region 的 RSet

#### CSet

- Collection Set，即收集集合，保存一次 GC 中将执行垃圾回收的区间（Region）

### ZGC

#### 设计目标

- 停顿时间不会超过 10ms

- 停顿时间不会随着堆的增大而增大（不管多大的堆都能保持在 10ms 以下）

- 可支持几百 M，甚至几 T 的堆大小（最大支持 4T）

### 配置参数

- -XX:+UseSerialGC 年轻代和老年代都用串行收集器

- -XX:+UseParNewGC 年轻代使用 ParNew，老年代使用 Serial Old

- -XX:+UseParallelGC 年轻代使用 ParallerGC，老年代使用 Serial Old

- -XX:+UseParallelOldGC 新生代和老年代都使用并行收集器

- -XX:+UseConcMarkSweepGC，表示年轻代使用 ParNew，老年代的用 CMS

- -XX:+UseG1GC 使用 G1垃圾回收器

- -XX:+UseZGC 使用 ZGC 垃圾回收器

## [JIT](https://developer.ibm.com/zh/articles/j-lo-just-in-time/)

### 热点代码

> 被频繁调用的代码，会被编译成机器码缓存起来，以备下次使用

- -XX:ReservedCodeCacheSize 限制codecache的大小

- JIT日志参数-XX:+PrintAssembly -XX:+LogCompilation -XX:LogFile=jitdemo.log

- 查看工具JITWatch

### 分层编译

> 在不启用分层编译的情况下(Mixed混合模式)，当方法的调用次数和循环回边的次数总和，超过由参数 -XX:CompileThreshold 指定的阈值时，便会触发即时编译；当启用分层编译时，这个参数将会失效，会采用动态调整的方式进行

#### 五个层级

- 字节码的解释执行

- 执行不带 profiling 的 C1 代码

- 执行仅带方法调用次数，以及循环执行次数 profiling 的 C1 代码

- 执行带所有 profiling 的 C1 代码

- 执行 C2 代码

#### 避免冷启动后的CPU高负载

- C1就是client, C2就是server

- 开启参数-XX:+TieredCompilation

### 优化手段

- 公共子表达式消除

- 数组范围检查消除

#### 方法内联

- C2 支持的内联层次不超过 9 层（-XX:MaxInlineLevel），最多1层的直接递归调用（-XX:MaxRecursIninleLevel）

- 如果方法不是经常执行的，默认情况下，方法大小小于35字节才会进行内联（-XX:InlineSmallCode）

- 如果方法是经常执行的，默认情况下，方法大小小于325字节的都会进行内联（ -XX:MaxFreqInlineSize）

- 特殊注解： @ForceInline @DontInline

#### 逃逸分析

- 通过逃逸分析，JVM 能够分析出一个新的对象使用范围，从而决定是否要将这个对象分配到堆上

- 默认开启(-XX:+DoEscapeAnalysis)

- 是否逃逸的依据

  - 对象被赋值给堆中对象的字段和类的静态变量

  - 对象被传进了不确定的代码中去运行

- 好处

  - 同步省略

  - 栈上分配

  - 分离对象或标量替换

#### intrinsic

- 注解： @HotSpotIntrinsicCandidate