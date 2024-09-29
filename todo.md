
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

----------

书签整理，先想好怎么分类
分类原则，如果严格按功能分

文件夹 结合 tag 这样就解决了一个网页需要多个书签文件夹的问题，多个书签文件夹->一个文件夹+多个tag
结合new tab快捷键，结合tag和key功能在地址栏直接打开
Ctrl+T打开新tab，ctrl+w关闭当前tab。
Ctrl+T打开新tab后光标默认在地址栏，此时如果想快速打开书签中的某个项可以输入“*tag”然后上下键选择，或者对于常用的可以直接设置key，然后地址栏直接输入key

文件夹以使用习惯分类，不用分太细，细化使用tag，如tools可以包含ip转换这种小工具、也可以包含man手册这种大工具，
以主题分类如Android的develop查询不分在tools，而分在android下tag为tool，opensource文件夹去掉，放android下，tag添加opensourceproject或github
书签精简，若B可以由A链接，则保留A即可。
