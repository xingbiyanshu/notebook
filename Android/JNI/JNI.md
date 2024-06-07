# 简介

JNI直译是“本机编程接口”，是一套标准、规范。它允许JVM中的代码和本地代码（C/C++，甚至是汇编）以一套标准的方式互操作。
本地环境各有差异，如Windows，Linux，JVM也有不同实现，所以标准化很重要，JNI就是标准。

Java开发一般在三种场景会用到JNI（通过调用native方法）：
某些平台特定的功能，不能由java来实现，需要C/C++实现；
已有的C/C++库想要利用；
某些效率非常敏感的模块需要用C/C++来实现；

# 总体设计

## JNI Interface Functions and Pointers
JNI interface pointer就是JNIEnv *env，它是一个jni函数列表。
native代码通过它使用jvm的功能。
JNIEnv是thread local的，所以不能跨线程传递。native线程想要获取JNIEnv需要先attach到jvm。

## Compiling, Loading and Linking Native Methods

java中加载本地方法通过 System.loadLibrary(“$nativeLib”);省略lib及后缀名，平台会自动补全。
JVM会为每个class loader记录加载的native库，需要保证需要使用该库的java代码和该库是同一个class loader加载的。
native库可以用动态库也可以用静态库（我们一般是用动态库）。对于静态库L，JVM会先尝试找 JNI_OnLoad_L方法，找到了就认为
是静态加载，否则找 JNI_OnLoad。（静态库从JNI_VERSION_1_8开始支持，所以JNI_OnLoad_L必须返回>=JNI_VERSION_1_8的值）

### native函数命名规则
native层和java层的函数/方法映射中，习惯用“native函数”表示C/C++侧的实现，“native方法”java侧的声明。
Java_$类全名_$方法名_参数列表 ， 例如：
Java_com_kedacom_kdv_mt_mtapi_ImCtrl_IMAddMemberInfoReq__ILjava_lang_StringBuffer_2I
( JNIEnv *env, jclass jcls, jint jnHandle, jobject jstrbufImMemberInfo, jint jnSSID ){
}
类全名中的路径分割符/使用_替代，如果方法有重载，则方法参数前使用两个下划线"__"，_1替代_,_2替代;，_3替代[，上面的Ljava_lang_StringBuffer_2I中的"_2"就是";"
Ljava_lang_StringBuffer_2表示"Ljava/lang/StringBuffer;"，StringBuffer类

java中的native方法定义可以理解为native方法在java层的头文件。

### native方法参数
第一个参数是JNIEnv*，第二个参数是定义该native方法的java对象jobject（非static的情况）或者class对象jclass（static的情况）。
其他参数对应普通的java参数。

## native引用java对象
基本数据类型如整数、字符直接在java和native之间拷贝，而java对象则通过引用传递给native。
JVM必须要跟踪这些传递到native层的引用，这样才不会错误的释放他们的引用，native层也需要在使用完后释放。
引用分两类：global和local。local只在native函数内有效，完事会自动释放，一般不需要native层手动释放（个别情况如引用了一个很重的java对象，
然后半途不需要了又去做了很多其他事情，这时就不要等函数执行完了，直接手动释放更好。又如大循环创建了很多临时的java对象，最好也是用完立即释放）。
local引用仅在当前线程有效，不能跨线程使用。
对于需要函数返回后仍然有效的引用则使用global，JNI支持从local创建global引用。

## native访问java对象

### 访问基本类型数组
native层访问JVM中的基本类型数组，如整数数组、字符串数组，非常低效。
一个解决的办法是让JVM把这些数组pinned-down下来，然后native直接访问这些数组的底层实现——直接访问他们的内存地址。但是这需要垃圾回收器支持，还需要JVM的实现要把数组保存在连续空间。
（大多数情况下满足这些条件）
最终JNI提供了两种方案的接口：一是直接访问pinned-down的数组，二是把java的数组在native层拷贝一份访问（这种比较低效）；

### 访问方法和字段
调用JNI函数：
jmethodID mid = env->GetMethodID(cls, “f”, “(ILjava/lang/String;)D”);
jdouble result = env->CallDoubleMethod(obj, mid, 10, str);
如果使用这些jmethodID过程中对应的java类被unload了，则该jmethodID会失效，native层需要处理这种情况。（也可以即拿即用，需要的时候重新再拿）

## 错误处理
JNI并不处理类似“从java层传递null到native层，在该传递object的地方传递了class对象”之类的问题，
一来对每个native接口做这类校验太损耗性能，二来有时这也不是一件容易办到的事。
开发者需要保证严格遵循了JNI规范。

### JAVA异常
native层可以抛出java异常，若native层未处理该异常，则会一直上抛给JVM。
JNI提供了ExceptionCheck判断是否存在java异常，ExceptionOccurred获取java异常实例。
native层调用java层方法后，如果依赖该调用的结果继续执行，则往往需要检查java异常。

异常可能发生在另外的线程，称为“异步异常”。异步异常不会影响当前native线程的执行，直到：
native代码调用了一个可能引发异常的JNIEnv函数，或者显式调用ExceptionOccurred检查（同步异步的异常都能检查到）。
如果native方法执行过程较长，它应该在适当的位置做这样的检查。

native函数可以选择不处理java异常，这样异常会抛给调用该native函数的java方法。
或者可以调用ExceptionClear清除掉java异常，然后自行处理掉该异常。
如果java异常发生了，native代码必须先处理掉该异常，然后才能调用其他JNIEnv函数。但有一些函数例外：
ExceptionOccurred()
ExceptionClear()
ExceptionCheck()
...
ReleaseStringChars()
...
DeleteGlobalRef()
...
详见官方文档。


# JNI Types and Data Structures

## 基本类型
Java Type 	Native Type 	Description
boolean 	jboolean 	unsigned 8 bits
byte 	jbyte 	signed 8 bits
char 	jchar 	unsigned 16 bits
short 	jshort 	signed 16 bits
int 	jint 	signed 32 bits
long 	jlong 	signed 64 bits
float 	jfloat 	32 bits
double 	jdouble 	64 bits
void 	void 	not applicable

## 引用类型
jobject
    jclass (java.lang.Class objects)
    jstring (java.lang.String objects)
    jarray (arrays)
        jobjectArray (object arrays)
        jbooleanArray (boolean arrays)
        jbyteArray (byte arrays)
        jcharArray (char arrays)
        jshortArray (short arrays)
        jintArray (int arrays)
        jlongArray (long arrays)
        jfloatArray (float arrays)
        jdoubleArray (double arrays)
    jthrowable (java.lang.Throwable objects)

jmethodID和jfieldID是普通的C结构体指针

## 类型签名
JNI使用JVM内部的签名方式
Type Signature              Java Type
Z 	                        boolean
B 	                        byte
C 	                        char
S 	                        short
I 	                        int
J 	                        long
F 	                        float
D 	                        double
L fully-qualified-class ; 	fully-qualified-class
[ type 	                    type[]
( arg-types ) ret-type 	    method type 

例如：
long f (int n, String s, int[] arr);
签名是这样的
(ILjava/lang/String;[I)J

## Modified UTF-8 Strings

JNI字符串使用的是“Modified UTF-8 Strings”变种UTF-8，主要有两个区别：
1、'\u0000'即NUL（它的名称，注意不是NULL)被编码为了双字节11000000 10000000(110xxxxx 10yyyyyy)
java字符串是unicode编码的，并且由于java字符串是对象有length属性，它并不需要类似c语言'\0'这种结束符，
所以'\u0000'是合法的java字符串内容（打印出来是个空格，但它并非空格字符），其标准的UTF-8编码是0x00，也即'\0'，
然而对于C来说，'\0'是字符串的结尾。 也就是说java传来一个字符串='A'+'\u0000'+'B'，C会自动把它截断为A、B两部分，进而导致异常。
变种UTF-8将0x00编码为了双字节：11000000 10000000，这样java传下来的字符串中就不存在'\0'了。 （110xxxxx 10yyyyyy这种是标准的UTF-8编码方式，第一个字节有几个1就表示该字符是几个字节表示的）
2、只有1、2、3字节的UTF-8被使用了，JVM不识别4字节的，需要使用两组3字节的UTF-8表示。
这个在处理emoji表情时会遇到相关的问题

# JNI Functions

## Interface Function Table
JNI函数都定义在JNIEnv 中，JNIEnv 实际是一个函数指针列表。
JNI函数不同版本可能不同，native层必须通过JNI_Onload返回值返回当前使用的版本。
运行中可通过GetVersion获取版本。
常用的函数有：
RegisterNatives
GetJavaVM
FindClass
ExceptionOccurred
NewGlobalRef
DeleteGlobalRef
GetObjectClass
GetMethodID
CallXXXMethod
GetXXXField
SetXXXField
NewStringUTF
GetStringUTFChars
ReleaseStringUTFChars
NewXXXArray
GetXXXArrayElement
SetXXXArrayElement
ReleaseXXXArrayElements
GetXXXArrayRegion
SetXXXArrayRegion
NewDirectByteBuffer

完整列表及详细说明请参考：
https://docs.oracle.com/javase%2F8%2Fdocs%2Ftechnotes%2Fguides%2Fjni%2F%2F/spec/functions.html


# The Invocation API

（对应 JavaVM指针，除了JNI_GetDefaultJavaVMInitArgs(), JNI_GetCreatedJavaVMs(), JNI_CreateJavaVM()其他的Invocation API都在JavaVM中）

Invocation API允许native应用将JVM加载其中，若JVM不存在可以在native层create，若已存在可以attach。然后就可以查找java类及方法调用。

    #include <jni.h>       /* where everything is defined */
    ...
    JavaVM *jvm;       /* denotes a Java VM */
    JNIEnv *env;       /* pointer to native method interface */
    JavaVMInitArgs vm_args; /* JDK/JRE 6 VM initialization arguments */
    JavaVMOption* options = new JavaVMOption[1];
    options[0].optionString = "-Djava.class.path=/usr/lib/java";
    vm_args.version = JNI_VERSION_1_6;
    vm_args.nOptions = 1;
    vm_args.options = options;
    vm_args.ignoreUnrecognized = false;
    /* load and initialize a Java VM, return a JNI interface
     * pointer in env */
    JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args);
    delete options;
    /* invoke the Main.test method using the JNI */
    jclass cls = env->FindClass("Main");
    jmethodID mid = env->GetStaticMethodID(cls, "test", "(I)V");
    env->CallStaticVoidMethod(cls, mid, 100);
    /* We are done. */
    jvm->DestroyJavaVM();

