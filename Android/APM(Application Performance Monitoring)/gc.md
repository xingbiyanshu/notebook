# JVM垃圾回收详解

## 堆空间的基本结构

从垃圾回收的角度来说，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆被划分为了几个不同的区域，这样我们就可以根据各个区域的特点选择合适的垃圾收集算法。
在 JDK 7 版本及 JDK 7 版本之前，堆内存被通常分为下面三部分：新生代内存(Young Generation)老生代(Old Generation)永久代(Permanent Generation)
下图所示的 Eden 区、两个 Survivor 区 S0 和 S1 都属于新生代，中间一层属于老年代，最下面一层属于永久代。堆内存结构JDK 8 版本之后 PermGen(永久) 已被
Metaspace(元空间) 取代，元空间使用的是直接内存 。

## 内存分配和回收原则

### 对象优先在 Eden 区分配

大多数情况下，对象在新生代中 Eden 区分配（大对象直接在老年代）。当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC（不会中断线程执行）。

### 大对象直接进入老年代

大对象就是需要大量连续内存空间的对象（比如：字符串、数组）。大对象直接进入老年代的行为是由虚拟机动态决定的，它与具体使用的
垃圾回收器和相关参数有关。大对象直接进入老年代是一种优化策略，旨在避免将大对象放入新生代，从而减少新生代的垃圾回收频率和成本。

### 长期存活的对象将进入老年代

既然虚拟机采用了分代收集的思想来管理内存，那么内存回收时就必须能识别哪些对象应放在新生代，哪些对象应放在老年代中。为了做到
这一点，虚拟机给每个对象一个对象年龄（Age）计数器。大部分情况，对象都会首先在 Eden 区域分配。如果对象在 Eden 出生并经过
第一次 Minor GC 后仍然能够存活，并且能被 Survivor 容纳的话，将被移动到 Survivor 空间（s0 或者 s1）中，并将对象年龄
设为 1(Eden 区->Survivor 区后对象的初始年龄变为 1)。对象在 Survivor 中每熬过一次 MinorGC,年龄就增加 1 岁，当它的
年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。
对象晋升到老年代的年龄阈值，可以通过参数 -XX:MaxTenuringThreshold 来设置。

### 回收

针对 HotSpot VM 的实现，它里面的 GC 其实准确分类只有两大种：
- 部分收集 (Partial GC)：
    - 新生代收集（Minor GC / Young GC）：只对新生代进行垃圾收集；
    - 老年代收集（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
    - 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。
- 整堆收集 (Full GC)：收集整个 Java 堆和方法区

### 回收的原因/场景

#### Concurrent

并发GC不会挂起应用程序线程。该GC在后台线程中运行，并且不会阻止分配。此为最常见的场景。

#### Alloc

应用尝试创建对象内存不够时触发。

#### Explicit

应用显式调用了gc()接口（但调用了gc()不一定触发Explicit，Android 14上实测是“Background concurrent copying GC”或“Background young concurrent copying GC”）。
注意：调用gc()接口只是给JVM提出建议，并不意味着JVM会立马执行，也不意味JVM会执行每一次gc()请求。现象就是短时间内几十次调用gc()接口，实际
要过一会且只触发了几次gc。

#### NativeAlloc

该次回收是由于本地内存分配引发的。

### 何时触发回收

这涉及到回收算法，回收算法不断演进，目前大体有以下几种场景会触发回收：

- 年轻代满了（Young Generation Full）
  当年轻代的Eden区满了时，会触发Minor GC（也称为Young GC）。这是因为新创建的对象首先分配在Eden区，当Eden区没有足够的
  空间来分配新对象时，就会进行Minor GC。
- 年老代满了（Old Generation Full）
  当年老代的空间不足以容纳新的对象时，会触发Major GC（也称为Full GC）。因为大对象会直接分配在年老代上，且年老代需要容纳年轻代
  变老的对象。Major GC不仅回收年老代，还可能包括年轻代的回收。相比于Minor GC，Major GC的开销更大，会导致应用的停顿时间更长。
- 元空间满了（Metaspace Full）
  在Java 8之前，持久代（Permanent Generation）用于存储类的元数据，Java 8之后改为元空间（Metaspace）。当元空间满了时，也会触发Full GC。
- 显式调用System.gc()
  尽管不推荐在生产环境中使用，调用System.gc()方法会建议JVM进行垃圾回收。不过，具体是否回收以及何时回收由JVM决定。
- 分配失败（Allocation Failure）
  当JVM尝试分配内存而没有足够空间时，会触发GC。JVM会尝试通过GC回收未使用的内存来满足分配请求。如果回收后仍然没有足够的内存，可能会导致OutOfMemoryError异常。
- 到达一定阈值时
  对于G1垃圾回收器，当年轻代和年老代的内存使用达到一定阈值时，会触发混合回收（Mixed GC），即回收年轻代和部分年老代的区域，以优化内存使用和应用停顿时间。
  对于CMS垃圾回收器，当老年代使用达到一定比例时，会触发并发标记和清理过程，以避免Full GC的长时间停顿。

### 死亡对象判断方法

#### 引用计数法

给对象中添加一个引用计数器：
每当有一个地方引用它，计数器就加 1；
当引用失效，计数器就减 1；
任何时候计数器为 0 的对象就是不可能再被使用的。
这个方法实现简单，效率高，但是目前主流的虚拟机中并没有选择这个算法来管理内存，其最主要的原因是它很难解决对象之间循环引用的问题。

#### 可达性分析算法

这个算法的基本思想就是通过一系列的称为 “GC Roots” 的对象作为起点，从这些节点开始向下搜索，节点所走过的路径称为引用链，
当一个对象到 GC Roots 没有任何引用链相连的话，则证明此对象是不可用的，需要被回收。
哪些对象可以作为 GC Roots 呢？
- 虚拟机栈(栈帧中的局部变量表)中引用的对象
- 本地方法栈(Native 方法)中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 所有被同步锁持有的对象
- JNI（Java Native Interface）引用的对象

### 引用类型总结

无论是通过引用计数法判断对象引用数量，还是通过可达性分析法判断对象的引用链是否可达，判定对象的存活都与“引用”有关。

1．强引用（StrongReference）以前我们使用的大部分引用实际上都是强引用，这是使用最普遍的引用。
如果一个对象具有强引用，那就类似于必不可少的生活用品，垃圾回收器绝不会回收它。当内存空间不足，Java 虚拟机宁愿抛出 
OutOfMemoryError 错误， 使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题。

2．软引用（SoftReference）如果一个对象只具有软引用，那就类似于可有可无的生活用品。
如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，
该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。软引用可以和一个引用队列（ReferenceQueue）联合使用，
如果软引用所引用的对象被垃圾回收，JAVA 虚拟机就会把这个软引用加入到与之关联的引用队列中。

3．弱引用（WeakReference）如果一个对象只具有弱引用，那就类似于可有可无的生活用品。
弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，
一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 
因此不一定会很快发现那些只具有弱引用的对象。弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，
Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。

4．虚引用（PhantomReference）"虚引用"顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。
如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。虚引用主要用来跟踪对象被垃圾回收的活动。
虚引用与软引用和弱引用的一个区别在于： 虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，
如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经
加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的
内存被回收之前采取必要的行动。
特别注意，在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为软引用可以加速 JVM 对垃圾内存的回收速度，
可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生。

### 如何判断一个常量是废弃常量？

运行时常量池主要回收的是废弃的常量。那么，我们如何判断一个常量是废弃常量呢？
假如在字符串常量池中存在字符串 "abc"，如果当前没有任何 String 对象引用该字符串常量的话，就说明常量 "abc" 就是废弃常量，
如果这时发生内存回收的话而且有必要的话，"abc" 就会被系统清理出常量池了。

### 如何判断一个类是无用的类？

方法区主要回收的是无用的类，那么如何判断一个类是无用的类的呢？判定一个常量是否是“废弃常量”比较简单，而要判定一个类是否是“无用的类”的条件则相对苛刻许多。
类需要同时满足下面 3 个条件才能算是 “无用的类”：
- 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。
- 加载该类的 ClassLoader 已经被回收。
- 该类对应的 java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。
虚拟机可以对满足上述 3 个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样不使用了就会必然被回收。

### 垃圾收集算法

#### 标记-清除算法（mark--sweep）

标记-清除（Mark-and-Sweep）算法分为“标记（Mark）”和“清除（Sweep）”阶段：
首先标记出所有不需要回收的对象，在标记完成后统一回收掉所有没有被标记的对象。它是最基础的收集算法，后续的算法都是对其不足进行改进得到。
这种垃圾收集算法会带来两个明显的问题：效率问题：标记和清除两个过程效率都不高。空间问题：标记清除后会产生大量不连续的内存碎片。

#### 复制（copying）

此算法把内存空间划分为两个相等的区域，每次只使用其中的一个区域。垃圾回收时，遍历当前使用区域的所有对象，将正在使用中的对象复制到另一个区域。此算法每次只处理使用中的对象，因此复制成本比较小，同时复制过去之后还能进行相应的内存整理，所以不会出现碎片问题。
不足：缺点也很明显就是需要两倍的内存空间。以空间换取时间。

#### 标记--整理（mark--compact）

此算法结合了标记--清除和复制的有点，也是分两个阶段，第一阶段是从根节点遍历所有对象标记所有能被引用的对象，第二阶段遍历整个堆中的对象，
清除所有未被标记的对象，并把所有存活对象“压缩”到堆的其中一块，按顺序排放。此算法避免了“标记--清除”的碎片问题，也没有“复制”的空间问题。
由于多了整理这一步，因此效率也不高。

#### 分代收集算法

当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。
一般将 Java 堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。比如在新生代中，每次收集都会有大量对象死去，
所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，
而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。

### 垃圾收集器

如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。虽然我们对各个收集器进行比较，但并非要挑选出一个最好的收集器。
因为直到现在为止还没有最好的垃圾收集器出现，更加没有万能的垃圾收集器，我们能做的就是根据具体应用场景选择适合自己的垃圾收集器。

#### Serial 收集器

Serial（串行）收集器是最基本、历史最悠久的垃圾收集器了。大家看名字就知道这个收集器是一个单线程收集器了。它的 “单线程” 
的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的
工作线程（ "Stop The World" ），直到它收集结束。
Serial 收集器有没有优于其他垃圾收集器的地方呢？当然有，它简单而高效（与其他收集器的单线程相比）。
Serial 收集器由于没有线程交互的开销，自然可以获得很高的单线程收集效率。

#### ParNew 收集器

ParNew 收集器其实就是 Serial 收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为（控制参数、收集算法、回收策略等等）
和 Serial 收集器完全一样。

#### Parallel Scavenge 收集器（JDK1.8 默认收集器）

Parallel Scavenge 收集器也是使用标记-复制算法的多线程收集器，它看上去几乎和 ParNew 都一样。 那么它有什么特别之处呢？
Parallel Scavenge 收集器关注点是吞吐量（高效率的利用 CPU）。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。
所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值。 Parallel Scavenge 收集器提供了很多参数供用户找到最合适的
停顿时间或最大吞吐量，如果对于收集器运作不太了解，手工优化存在困难的时候，使用 Parallel Scavenge 收集器配合自适应调节策略，
把内存管理优化交给虚拟机去完成也是一个不错的选择。

#### CMS 收集器（Android8前的默认收集器）

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用。
CMS是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。

从名字中的Mark Sweep这两个词可以看出，CMS 收集器是一种 “标记-清除”算法实现的，它的运作过程相比于前面几种垃圾收集器来说更加复杂一些。
整个过程分为四个步骤：
初始标记： 暂停所有的其他线程，并记录下直接与 root 相连的对象，速度很快 ；
并发标记： 同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。
        因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。
重新标记： 重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，
        这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短。
并发清除： 开启用户线程，同时 GC 线程开始对未标记的区域做清扫。

CMS 垃圾回收器在 Java 9 中已经被标记为过时(deprecated)，并在 Java 14 中被移除。

#### CC（Concurrent Copying，Android8开始默认垃圾收集器）

CC 支持使用名为“RegionTLAB”的触碰指针分配器。此分配器可以向每个应用线程分配一个线程本地分配缓冲区 (TLAB)，这样，应用线程只需触碰“栈顶”指针，而无需任何同步操作，即可从其 TLAB 中将对象分配出去。
CC 通过在不暂停应用线程的情况下并发复制对象来执行堆碎片整理。这是在读取屏障的帮助下实现的，读取屏障会拦截来自堆的引用读取，无需应用开发者进行任何干预。
GC 只有一次很短的暂停，对于堆大小而言，该次暂停在时间上是一个常量。
在 Android 10 及更高版本中，CC 会扩展为分代 GC。它支持轻松回收存留期较短的对象，这类对象通常很快便会无法访问。这有助于提高 GC 吞吐量，并显著延迟执行全堆 GC 的需要。

#### G1 收集器

G1 (Garbage-First) 是一款面向服务器的垃圾收集器,主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足 GC 停顿时间要求的同时,还具备高吞吐量性能特征.
从 JDK9 开始，G1 垃圾收集器成为了默认的垃圾收集器。


## Use SIGQUIT to get GC performance info
To get GC performance timings for apps, send SIGQUIT to already running apps or pass in
-XX:DumpGCPerformanceOnShutdown to dalvikvm when starting a command line program. When an app gets 
the ANR request signal (SIGQUIT), it dumps information related to its locks, thread stacks, and GC performance.
To get GC timing dumps, use:
adb shell kill -s QUIT PID
This creates a file (with the date and time in the name such as anr_2020-07-13-19-23-39-817) in /data/anr/. 
This file contains some ANR dumps as well as GC timings. You can locate the GC timings by searching for 
Dumping cumulative Gc timings. These timings show a few things that may be of interest, including the 
histogram info for each GC type's phases and pauses. The pauses are usually more important to look at. For example:
young concurrent copying paused:	Sum: 5.491ms 99% C.I. 1.464ms-2.133ms Avg: 1.830ms Max: 2.133ms
This shows that the average pause was 1.83 ms, which should be low enough that it doesn't cause missed frames in most apps and shouldn't be a concern.
Another area of interest is time to suspend, which measures how long it takes a thread to reach a 
suspend point after the GC requests that it suspends. This time is included in the GC pauses, 
so it's useful to determine if long pauses are caused by the GC being slow or the thread suspending slowly.
Here's an example of a normal time to suspend on a Nexus 5:
suspend all histogram:	Sum: 1.513ms 99% C.I. 3us-546.560us Avg: 47.281us Max: 601us
There are other areas of interest, including total time spent and GC throughput. Examples:
Total time spent in GC: 502.251ms
Mean GC size throughput: 92MB/s
Mean GC object throughput: 1.54702e+06 objects/s

## logcat中gc日志

如，超大循环创建新对象，logcat中出现如下日志（Android 14）：
Alloc young concurrent copying GC freed 376258(12MB) AllocSpace objects, 0(0B) LOS objects, 6% free, 323MB/346MB, paused 39us,15us total 60.043ms

格式=触发原因 + 回收器的名称（一般说明了回收的算法）+ 年轻代释放的对象数量和大小 + 年老代（LOS-Large Object Space, 大对象区域，大对象处于年老代）+堆当前状况（空闲，已用/总共）+ GC导致的应用暂停时间 + GC总耗时
     Alloc    young concurrent copying GC     376258(12MB)                   0(0B)                                            6% free, 323MB/346MB      39us,15us          total 60.043ms
“GC导致的应用暂停时间”若过长则会导致应用卡顿。

## 代码中获取gc统计



