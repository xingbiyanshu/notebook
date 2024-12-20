# FootPrint

## c++枚举转字符串
（枚举定义消息，用于打印消息交互）
使用map。定义在manager中static msgNames，通过接口设置给下层的会话管理器。

## 原来的jni层拍扁放sdk
Caster接口
新增command


## 消息如何定义
Request
Command
Response
Notify
Message
Session

如何像原来一样可以在一个地方看到message相关所有定义？
定义static消息结构体list，延迟初始化以节约内存（调用init函数时触发初始化）

## 定义上帝对象

## 匿名内部类用于传监听器参数


StartupManager->Caster{
    overload
    list<Request> requests(){
    }

    overload
    list<Response> responses(){
    }

    overload
    list<Notification> notifications(){
    }
}