# DDD

## 业务用例

刷新组织架构；
浏览组织架构树（只需要某人简要信息（名字、图标））；展开时尽量秒刷新甚至体验不到刷新过程。可以提前加载当浏览当前部门时把下一级部门的人的简要信息加载，头像可先不加载比较大，等展开时再加载头像。
查看某人详情；
全局搜索；
基于某个节点的搜索；支持名字、拼音、缩写，部分匹配。要求边输入边展示结果，且反馈迅速。
自定义常用联系人列表，支持层级归类；

## 基于DDD的项目结构

TODO

## 概念解析

### 聚合根

### Repository

repository仅针对聚合根。这意味着：
1、只有聚合根才有对应的Repository。但并非每个聚合根都必须配备Repository，是否需要看需求；
2、Repository中的接口的参数或者返回值如果是类对象只能是聚合根对象或其对象的集合，但如果是基本类则没限制，如获取部门下所有成员，
则可以通过member repo的getAllMemberInDepartment(string depId)，这里的参数是部门id。如果需要获取聚合根的某个成员，
则通过聚合根对象遍历获取，而非通过Repository接口拿。且repo接口的实现不能引用其它聚合根对象。比如部门repo中接口getMemberCounts(depId)，
该接口内部实现去访问了Member的repo，这是不对的。这个接口应该放到service中，部门的repo只应存在getDepartmentCounts(depId)，
然后service再去member repo中去计算所有属于这些部门的人数。
3、如果一个需求跨多个聚合根，（如统计某个部门下的人数（递归），需要先通过部门聚合根的Repository获取该部门下所有子部门，
然后再通过成员聚合根的Repository获取所有部门下的人数，再累加），则应创建一个service来协调处理，而非直接让不同的
Repository中混合其它Repository的内容，也就是不能违反第2点。比如在部门repo中增加一个接口getAllMemberCounts(depId)，
然后该接口内部实现去访问了Member的repo。

领域层repository应该仅定义接口，具体实现用户可配置，这样可以让领域层和具体的持久化实现解耦。（具体实现放哪？又在哪配置？）

### 领域事件

TODO
