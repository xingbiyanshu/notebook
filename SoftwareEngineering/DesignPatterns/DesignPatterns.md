# 六大原则（SOLID原则）

## 单一职责原则(SRP——Single Responsibility Principle)
简单说就是提高内聚性，一个类、接口只做一件事或一组相关性较高的事，而非出于种种原因把相关性不高的事丢在一起。

## 开闭原则（OCP——Open Close Principle）
一个类应该对于扩展是开放的，但是对于修改是封闭的。
在一开始编写代码时，就应该注意尽量通过扩展的方式实现新的功能，而不是通过修改已有的代码实现，否则容易破坏原有的系统，也可能带来新的问题，如果发现没办法通过扩展来实现，应该考虑是否是代码结构上的问题，通过重构等方式进行解决。

## 里氏替换原则（LSP——Liskov Substitution Principle）
所有引用基类的地方必须能透明地使用其子类对象。

## 迪米特原则（LoD——Law of Demeter （ Least Knowledge Principle））
一个对象只需要知道它该知道的（完成它的职责所需要知道的），不相干尽量少了解。
就像地下情报工作者尽量只知道自己该知道的，整个组织就不容易出大事。

## 接口隔离原则（ISP——Interface Segregation Principle）
类之间的依赖关系应该建立在最小的接口上。
其原则是将非常庞大的、臃肿的接口拆分成更小的更具体的接口。
跟“单一职责”、“迪米特”有相通性。

## 依赖倒置原则（DIP——Dependency Inversion Principle）
一般我们模块之间的关系是上层模块依赖下层模块，使用下层模块提供的接口完成某些功能，而下层模块提供的这些接口往往是具体实现相关的，并不稳定。

而这个原则就是要求改变这种“自然而然”的依赖关系。 变成：
具体的依赖抽象的（代码层面往往是接口）——
高层模块不应该依赖底层模块（具体实现），二者都应该依赖其抽象（抽象类或接口），比如MVP模式中的VP之间的依赖就是通过接口（抽象的）。

变化的依赖稳定的——
具体实现是可能随时变化的，接口是稳定的

次要的依赖主要的——
业务模块往往是一个产品最重要的模块，所以它应该尽量保持独立性，不应该依赖其他模块的具体实现细节。
比如我一个视频会议业务模块，既可以在android平台用，也可以在ios模块用（假设不考虑语言差异），那该模块就不应该依赖android平台或ios平台相关的东西。
那样无法保持独立性了。而应该由该模块自己定义接口，不同平台提供不同的实现该接口的适配器来接入该业务模块。就像OS和驱动之间的关系。尽管驱动在OS下层，但他们之间的接口是由OS定义的。
更详细的可参考Clean架构。

面向接口编程，提取出事务的本质和共性。

## 高内聚低耦合
所有以上原则其实都是“高内聚低耦合”的不同形式的表现。

## 控制反转
所谓控制反转就是反转获取依赖对象的过程。传统的是自己需要用到某个对象时自己“主动”去创建，控制反转就是不需要自己主动去创建，而是“被动”接收，
往往是通过方法参数传入。常见的具体实现有两种：DI(Dependency Inject)依赖注入（如android的Dagger框架）和“依赖查找”（Dependency Lookup）。



# 设计模式
设计模式是对SOLID原则的实践。
设计模式大致分三类：（类对象的）创建型、（类之间的）结构型、（类之间的）行为型。


## 创建型

创建型封装了对象的创建细节，使得使用者创建符合需求的对象更加便捷。

### 简单工厂

简单工厂模式又叫做静态工厂方法模式（static Factory Method pattern）,它是通过使用静态方法接收不同的参数来返回不同的实例对象。
实例：

````

/**产品接口**/

public interface Product {
    void doSomething();
}

/**具体产品实现**/
class ProductA implements Product{
    public void doSomething() {
        System.out.println("我是ProductA");
    }
}

class ProductB implements Product{
    public void doSomething() {
        System.out.println("我是ProductB");
    }
}

/**简单工厂**/
public class SimpleFactory { // 就一个工厂
    /**工厂类创建产品静态方法**/
    public static Product createProduct(String productName) {  // 一般是静态方法，返回的是接口对象而非具体的Product
        Product instance = null;
        switch (productName){
            case "A":
                instance = new ProductA();
                break;
            case "B":
                instance = new ProductB();
                break;
        }
        return instance;
    }

    /**客户端(client)调用工厂类**/
    public static void main(String[] args) {
        Product product = SimpleFactory.createProduct("A"); // 客户端不需要了解创建对象的具体细节，甚至不需要知道具体的对象。（它只需知道是个product）
        product.doSomething();
        product = SimpleFactory.createProduct("B");
        product.doSomething();
    }
}

````
client不直接new对象而是通过工厂创建对象，自然是为了封装client不想关注的细节。
简单工厂的client只需要知道它要创建的对象的标识符（如这里的“A”、“B”）以及他需要的是一个product（具体是什么product不关注），这样至少代码上有个显而易见的好处就是依赖的类数量减少了。
缺点：违背开闭原则，如果需要新增其他产品类，就必须在工厂类中新增if-else逻辑判断（可以通过配置文件来改进）。
    但是整体来说，系统扩展还是相对其他工厂模式要困难。

Android的getSystemService(String serviceName)就是简单工厂模式的例子。

### 工厂方法模式
工厂方法模式对比简单工厂模式，多了工厂的接口和各个不同的工厂。原本一个工厂的任务拆分到各自不同工厂了。

````

/**产品相关的定义同简单工厂模式**/
.....


/**多了工厂接口**/
public interface AbstractFactory {
	/**创建Product方法,区别与工厂模式的静态方法**/
    public Product createProduct();
}



/**多了具体工厂实现**/
class FactoryA implements AbstractFactory{
    public Product createProduct() {
        return new ProductA();
    }
}



class FactoryA implements AbstractFactory{
    public Product createProduct() {
        return new ProductA();
    }
}



class FactoryA implements AbstractFactory{
    public Product createProduct() {
        return new ProductA();
    }
}

/**客户端调用工厂**/
public class Client {
    public static void main(String[] args) {
        Product productA = new FactoryA().createProduct(); // 和简单工厂方法一样，客户端仍然无需关心创建对象的具体过程，甚至具体的对象是什么。
                                                            但是就像前面需要知道对象标识符“A”一样，client需要知道产品A对应的工厂是FactoryA
        productA.doSomething();
        Product productB = new FactoryB().createProduct();
        productB.doSomething();
    }
}

````
现在要新增产品可以不用修改已有代码了，而是新增一种工厂。符合了“开-闭原则”。
但是每新增一种产品就要新增一个具体的Product类和Factory类。

### 抽象工厂模式

抽象工厂模式适用于生产“产品族”的场景。

比如数据协作的PaintBord/Painter/PaintFactory就是抽象工厂模式实现的。
PaintBord和Painter都是产品，他们是配套使用的，是一个产品族。一个factory要同时生产这两类产品，也就是生产一个产品族。
每个产品都有自己的接口定义，每个工厂也有自己的接口定义，这点和工厂方法一样。但是多了一个“产品族”的概念，就是一次不是只生产一类产品，
而是生产一个产品族。

上面列举了三种工厂模式，他们的初衷都是一样的，即对用户隐藏创建对象的具体细节，只不过他们适用的场景不一样。
如针对产品族的场景适用抽象工厂。 用户甚至不知道创建的具体对象是什么，他只知道对象的抽象接口类型，他是通过接口引用对象的，也是通过接口引用工厂的。
这样的好处就是扩展性可维护性很好，当用户想要替换另一个产品时只需要在代码中换用另一工厂，可能就是改一行代码的事。更进一步，用户可以通过配置文件
来配置要使用的工厂，这样完全不必改代码，只需要改配置文件，等程序重新价值配置文件，工厂也更新了。

### 单例模式

有的类我们只需要一个实例，多个实例不仅没必要有时可能还会造成混乱或资源浪费，这种情况下我们采取一些手段让客户只能创建出一个类实例，这种模式就是单例模式。
单例模式的使用场景有很多，Android的众多系统服务都是单例的，比如AM，WM，PM等。
WindowManager wm = (WindowManager)getSystemService(getApplication().WINDOW_SERVICE);
很多语言都内置了语言机制来支持这种模式，比如kotlin的object类。这样省去了用户自己实现（什么懒汉模式、线程锁之类的细节）。

### 生成器（Builder）模式
有时候一个对象非常复杂，成员变量都有几十个。这时要构造这个对象，调用构造方法传递几十个参数显然不可取，即便可以多重载几个构造函数针对不同场景
内部初始化一些参数，这也非常麻烦。这个时候就可以考虑使用Builder模式了。
比如android中的
AlertDialog.Builer builder=new AlertDialog.Builder(context);
builder.setIcon(R.drawable.icon)
.setTitle("title")
.setMessage("message")
.setPositiveButton("Button1",
new DialogInterface.OnclickListener(){
public void onClick(DialogInterface dialog,int whichButton){
setTitle("click");
}   
})
.create()
.show();

Builder模式可以将复杂的构造过程，拆分成一个个的set，一般支持链式调用，也可以在代码的不同阶段set不同的字段，而不必像调用构造时一股脑全部传入。
只有等到最终调用构建方法时才会真正执行对象构造。如上面的show()，一般是build()方法。

### 原型模式
原型模式，即Prototype，是指创建新对象的时候，根据现有的一个原型来创建。
我们举个例子：如果我们已经有了一个String[]数组，想再创建一个一模一样的String[]数组，怎么写？
实际上创建过程很简单，就是把现有数组的元素复制到新数组。如果我们把这个创建过程封装一下，就成了原型模式。用代码实现如下：
// 原型:
String[] original = { "Apple", "Pear", "Banana" };
// 新对象:
String[] copy = Arrays.copyOf(original, original.length);
对于普通类，我们如何实现原型拷贝？Java的Object提供了一个clone()方法，
它的意图就是复制一个新的对象出来，我们需要实现一个Cloneable接口来标识一个对象是“可复制”的。

通过“复制”一个已经存在的实例来返回新的实例,而不是新建实例。被复制的实例就是我们所称的“原型”，这个原型是可定制的。
原型模式多用于创建复杂的或者耗时的实例，因为这种情况下，复制一个已经存在的实例使程序运行更高效；或者创建值相等，只是命名不一样的同类数据。
原型模式是在内存二进制流的拷贝， 要比直接new一个对象性能好很多， 特别是要在一个循环体内产生大量的对象时， 原型模式可以更好地体现其优点。
原型模式还可以避免构造函数的约束 ，这既是它的优点也是缺点，直接在内存中拷贝，构造函数是不会执行的 。 优点就是减少了约束， 缺点也是减少了约束， 需要大家在实际应用时考虑。
这里需要注意的一点是“深拷贝”和“浅拷贝”的区别，只有深拷贝出来的才是完全不同的对象。
Java的Object中提供了clone方法，这是原型模式的一个例子。

实际上，创建对象包含的申请内存、给成员变量赋值这一过程，本身并不会花费太多时间，或者说对于大部分业务系统来说，这点时间完全是可以忽略的。
应用一个复杂的模式，只得到一点点的性能提升，这就是所谓的过度设计，得不偿失。这种情况，不建议使用原型模式。
如果对象中的数据需要经过复杂的计算才能得到（比如排序、计算哈希值），或者需要从 RPC、网络、数据库、文件系统等非常慢速的 IO 中读取，
这种情况下，我们就可以利用原型模式，从其他已有对象中直接拷贝得到，而不用每次在创建新对象的时候，都重复执行这些耗时的操作。
此时，建议使用原型模式，如数据库连接池、线程池等各种资源池。

很多实例会持有类似文件、Socket这样的资源，这些资源是无法复制给另一个对象共享的。


## 结构型

### 适配器模式

#### 使用场景
需要使用现有类，但其接口不符合系统需求，而我们无法修改它或者修改它不合适时。（往往用于事后补救，已经存在了某个功能类，想使用但接口又不符合需求）；
需要一个统一的输出接口，而输入类型不可预知（往往在类体系结构设计时就包含了Adapter。如Android的各种集合View和Adapter。因为View加载数据的方式相对统一，但数据的来源以及获取方式却多种多样无法预估）；

#### 角色组成
目标接口（Target）：客户期望的接口。
适配者类（Adaptee）：一个已存在的类，提供了客户需要的功能但接口不是客户期望的。
适配器类（Adapter）：实现了Target接口，因而客户可以直接和其打交道，同时通过组合或继承的方式将客户的请求转交给Adaptee处理。

#### 优缺点
优点：
    可以复用已有类；
    允许类的设计更加灵活同时又不降低复用性。Adapter不一定是在需要复用已有类的时候才考虑添加的，也可能是在设计类体系结构的时候就考虑添加Adapter这一环了。
比如，Android的ListView和Adapter是一开始就设计好的（adapter是listview的一个成员变量）。
可能成为Adaptee的类（可能是一整套）在设计的时候可能无法照顾到所有的使用场景，
但它仍然可以按照已有的一套标准来设计，标准以外的情况可以通过适配器来处理。如电器，国外标准的电器到中国来尽管电气参数不同仍然可以通过各种适配器使用。

缺点：
过度使用适配器可能导致系统结构混乱，难以理解和维护。

#### 代码实现
如：在生活中手机充电器, 需要将家用220V的交流电 转换为 5V的直流电后, 才能对手机充电。
手机充电器 相当于 Adapter适配器
220V的交流电 相当于 Adaptee 被适配者
5V的直流电 相当于 Target目标
手机 相当于 客户

````
public class Phone {
    public void charging(Target voltage) { // charging接受的参数是5V的Target，所以下面的Adaptee没法直接使用
        if (voltage.output5V() == 5) {
            System.out.println("电压适配为5V，可以充电");
        } else if (voltage.output5V() > 5) {
            System.out.println("电压大于5V，不能充电～");
        }
    }
}

public interface Target { // Phone期望的电压
    public int output5V();
}

public class Adaptee { // 已有了Adaptee，想利用但是不符合Phone#charging的入参要求
    public int output220V() {
        System.out.println("正常220V电压");
        return 220;
    }
}

// 于是定义适配器完成转换

// // 类适配器模式
// public class Adapter extends Adaptee implements Target { // adaptee是通过继承的方式（JAVA不支持多重继承，所以Target得是接口才可实现类适配器模式）
//     @Override
//     public int output5V() {
//         // 获取到220V的电压
//         int a = output220V();
//         // 处理电压，转成5V
//         int b = a / 44;
//         return b;
//     }
// }

// 对象适配器模式（优先使用这种方式）
public class Adapter implements Target {
    private Adaptee adaptee; // adaptee是通过组合的方式而非继承

    public Adapter(Adaptee adaptee) {
        this.adaptee = adaptee;
    }

    @Override
    public int output5V() {
        int dst = 0;
        if (adaptee != null) {
            int src = adaptee.output220V();
            System.out.println("使用对象适配器进行适配");
            dst = src / 44;  // 转换电压
            System.out.println("适配完成，输出电压为：" + dst);
        }
        return dst;
    }
}


public class Test {
    public static void main(String[] args) {
        Phone phone = new Phone();
        phone.charging(new Adapter(new Adaptee()));
    }
}

````