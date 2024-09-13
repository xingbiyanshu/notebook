
xapm中新建一个app模块使用hprofclipper
真机上验证效果
集成到项目验证效果（可能不能在崩溃时导？）

-----------------------------------------------------
~~fd类型 pipe一般是怎么创建的~~
asan使用
代码导出javaheap
libunwind使用
profiler cpu使用
~~gc信息写入日志~~
systrace
profiler使用详解
android hook机制

koom研究

基于koom裁剪出一个最小的hprofclipper, unwind，上面的实现自己弄。
使用prefab？引用android-base

尽管线下有多种手段可以排查问题，但线上的问题我们主要还是依靠日志，所以把尽可能多有用的信息在适当时机写入日志对定位问题有很大帮助。

# 项目必备

## 问题排查手段

### 日志系统
除了常规的打印，还需包括各种辅助定位信息，如崩溃栈、内存、cpu、线程、fd使用情况，gc情况等。

### javaheap导出
裁剪

### native内存问题


## 升级手段
整包推送
热修复框架

