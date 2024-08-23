
# 内存相关问题

## 内存泄漏

内存泄漏是指按预期应该释放/回收的对象（生命周期已结束的对象）未能及时释放/回收，导致内存资源浪费。
本质是长生命周期对象持有了短生命周期对象引用，未及时删除。
内存泄漏是Bug，最终结果是OOM。

### 常见场景

- 申请了资源忘记释放。如打开File/Stream/Socket/Cursor等用完忘记关闭，malloc了忘记free等。
- 注册了监听器忘记注销/删除。如注册了广播监听器、回调等，在Activity#onDestroy时未删除。
- Activity、Service等短生命周期对象作为Context传给了长生命周期对象，如单例对象，没有及时删除，或不能删除（此种情形为误用，应传Application）。
  如Toast、Animation等都需要context参数，但其底层将该context关联到了长生命周期对象，如不及时删除(在onDestroy中调用cancel)则可能造成泄漏。
- 非静态内部类（常见各种匿名内部类）持有了外部类如Activity的引用，自身又被长生命周期对象引用，进而导致外部类不能及时释放。
  如Activity中定义的非静态Handler，引用链是：MessageQueue->Message->Handler(通过message.target)->Activity（通过handler.this$0），其中MessageQueue是
  长生命周期对象，导致了Activity泄漏。
- Service中的自定义binder要在对端binder gc时才会被回收，所以binder如果引用了service则可能导致service泄漏。
- 文件描述符(FD)溢出。linux一切皆文件，所以Android中许多操作都会创建FD，如创建HandlerThread、打开Stream、Cursor生成、WindowManager.addView等。
  而每个进程可创建的FD是有限的，具体的值可通过命令"ulimit -n"查看(Android 9之前是1024，9以及之后扩展到了32768)，超过该值则导致OOM崩溃。
  *注：FD溢出导致的崩溃，由于崩溃点并非崩溃的真实原因，所以崩溃栈往往看起来没啥意义（内存溢出导致的崩溃同理），
  并且此时内存状况往往也正常，所以往往没什么头绪，当出现此种状况时可考虑是否有FD泄漏。
  FD溢出时logcat往往会打印出"Too many open files"。
  除了FD外，系统对进程可申请的其它资源均有约束，超过均会导致OOM，约束详情定义在文件“/proc/pid/limits”*
- Thread溢出。类似FD，系统对进程可创建的线程数也是有约束的。
  若没有系统的约束，进程可创建的线程数理论上=进程的虚拟地址空间大小/线程的栈大小。32位虚拟空间=4G-1G内核=3G，线程帧大小默认8M（可通过"adb shell ulimit -a"查看）。
  所以32位一个进程大概可以创建3G/8M=384个线程，64位的话可以认为无上限了，只受制于系统约束。
  当线程数超出系统约束或超出进程空间可承载的极限时（64位上基本不可能），则会报OOM。

### 规避/修复手段

- 资源对象用完要关闭。Java中可以用try-finally或者try-with-resources。
- 注册/注销监听器要成对出现。可以利用Lifecycle相关组件将监听器生命周期和LifecycleOwner生命周期绑定以实现自动注销。
- context使用要注意目标对象的生命周期，如果目标的生命周期较长可考虑使用ApplicationContext。
- 使用静态内部类避免隐式持有外部类引用。
- 使用弱引用或软引用（不会立即被回收，但在内存紧张时会优先被回收，缓存对象池可以考虑使用）。
  *注：可以使用ReferenceQueue监控弱引用/软引用回收情况，被回收的会被添加到ReferenceQueue。此机制可以用来监控内存泄漏：
  将目标对象包装为弱引用，当预期目标对象已被释放时检查ReferenceQueue中是否存在包装的弱引用，若不存在则目标对象已泄漏。
  此即为LeakCanary的工作原理。*

### 检测手段

内存泄漏处理的最大难点是检测到内存泄漏，所以检测手段至关重要。

- 观察logcat日志。
  内存泄漏最终可能导致OOM，发生OOM时可以从logcat看到相关信息：
  - java堆内存不够用了
    "java.lang.OutOfMemoryError: Failed to allocate a xx byte allocation with xx free bytes and xx until OOM"
  - java线程以及attach到jvm的native线程超限了
    "java.lang.OutOfMemoryError: Could not allocate JNI Env"
    JVM中的线程走JNI层的都需要创建一个JNIEnv
  - fd超限了
    "Too many open files"

  但是大多数情况下，内存泄漏无法在logcat中找到异常。
  
- AS自带的Memory Profiler。（需要debug版本，意味着线上版本不能用）
  最直观边操作边观察内存使用情况变化，也可以dump出hprof文件分析具体泄漏的对象。
- adb shell dumpsys系列命令（不需要安装AS，不限于debug版本，但仅能看到概览）
  adb shell dumpsys activity activities $package 可以查看app的task/activity详情
  注意：如果activity泄漏了（走到了onDestroy但实际未能回收成功），dumpsys activity看不出异常。
  即dump结果中没有该泄漏了的activity，此时用命令"dumpsys meminfo"可以看到app中实际存在的activity数量。
  对比"dumpsys activity"中的结果或UI观察的结果可知道泄漏了多少activity。
  周期性执行“dumpsys meminfo”对比数据，也可发现端倪。
- adb shell am dumpheap <PID> savePath （手机上的path而非电脑，如 /data/local/tmp/dump.hprof） 导出hprof文件。
  也可以在app中添加调试按钮导出hprof文件。
  然后导入AS的memory profiler分析。（需要debug版本，意味着线上版本不能用）
- 在日志中周期性打印内存使用情况。（Debug版本中使用）
- LeakCanary（Debug版本中使用，需要项目集成）
  默认仅针对Android的几个组件Activity,Fragment,Service等，要针对其他对象需要写代码Watch。

- adb shell "ps -T -p <PID>" 查看进程下的所有线程，观察是否数量是否异常，是否不断增加
  adb shell "ps |grep <appId>"， 拿到进程ID，再通过adb shell "ps -T -p <PID>"查看其下所有线程。
  另外 adb shell "cat /proc/<PID>/status" 可以查看进程详情。
  在AS自带的CPU Profiler中也可以查看所有线程。
  adb shell "top -H -p <PID>"可以查看线程动态信息。

- adb shell ls -l /proc/<PID>/fd/ 查看进程打开的所有文件，观察是否数量是否异常，是否不断增加。
  上面的命令可能因为权限无法执行，可以在代码中编码获取，周期性打印到logcat日志。
  另外logcat中有如下打印均有fd泄漏的嫌疑：
  “Too many open files”\
  “Could not allocate JNI Env”\
  “Could not allocate dup blob fd”\
  “Could not read input channel file descriptors from parcel”\
  "pthread_create * "\
  “InputChannel is not initialized”\
  “Could not open input channel pair”\
  “FORTIFY: FD_SET: file descriptor >= FD_SETSIZE” 
- dumpsys window ，查看是否有异常window。用于解决 InputChannel 相关的泄漏问题（fd泄漏）
  
- AddressSanitizer/HWAddressSanitizer
  ASan/HWASan用来检测native层内存问题。它可以检测：
  堆/栈溢出；
  堆指针free后非法使用；
  栈变量释放后非法使用；
  重复free，非法free；

  ASan使用详情后面单独介绍。

## 内存溢出

内存溢出按字面理解是指程序申请的内存超出系统允许的阈值。如Android限制每个App的JVM内存256M，超过该值则报OOM。
不过实际开发中经常会遇到内存状况良好却报OOM的情况，所以此时的“内存溢出”可以理解为“进程资源超限”。进程资源除了内存外，还包括
文件句柄、线程数、信号数、消息队列大小等，这些资源超限均会报OOM。
内存溢出可能是内存泄漏导致的，也可能是由于需求实现方案不合理导致。前者是Bug，后者则需要重新评估方案。如数据量过大时可以分级加载
不要一次性把所有数据加载到内存，如文档阅读器逐页加载，组织架构逐级加载。

毫无头绪的崩溃，如崩溃栈全是系统代码不涉及App代码时可怀疑是内存溢出方面的问题。




线上监控：
①上报app使用期间待机内存、重点模块内存、OOM率
②上报整体及重点模块的GC次数，GC时间
// ③使用LeakCannery自动化内存泄漏分析（默认只debug版本）
总结：
上线前重点在于线下监控，把问题在上线前解决；上线后运营阶段重点做线上监控，结合一定的预警策略及时处理
4、真的出现低内存，设置一个兜底策略
低内存状态回调，根据不同的内存等级做一些事情，比如在最严重的等级清空所有的bitmap，关掉所有界面，直接强制把app跳转到主界面，
相当于app重新启动了一次一样，这样就避免了系统Kill应用进程，与其让系统kill进程还不如浪费一些用户体验，自己主动回收内存。
