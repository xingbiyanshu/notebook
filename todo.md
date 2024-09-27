
~~14及以上导不出的问题~~
hprof裁剪了还未压缩
unwind

-----------------------------------------------------
~~fd类型 pipe一般是怎么创建的~~
asan使用
~~代码导出javaheap~~
libunwind使用
profiler cpu使用
~~gc信息写入日志~~
systrace
profiler使用详解
~~android hook机制~~

koom研究

基于koom裁剪出一个最小的hprofclipper, unwind，上面的实现自己弄。


尽管线下有多种手段可以排查问题，但线上的问题我们主要还是依靠日志，所以把尽可能多有用的信息在适当时机写入日志对定位问题有很大帮助。

# 项目必备

## 问题排查

### 日志系统
除了常规的打印，还需包括各种辅助定位信息，如崩溃栈、内存、cpu、线程、fd使用情况，gc情况等。

### 内存问题

#### javaheap导出
时机：利用OomMonitor自定义警戒值，触发时即dump，此时离oom系统强杀还有一定距离来得及dump。
防卡顿，提升用户体验：fork进程去dump。
裁剪：使用koom的裁剪功能。
压缩：使用lzma sdk。压缩耗内存且耗时，在内存即将溢出时容易失败，可在app重启时去做。

#### native内存问题


### 崩溃捕获

#### java层崩溃
Thread.setDefaultUncaughtExceptionHandler

#### native层崩溃
通过捕获信号捕获崩溃；
利用unwind解析出崩溃栈信息；

### 卡顿

### 电池耗用

## 升级手段
整包推送
热修复框架

