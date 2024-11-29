# C++ Core Guidelines

## Interface

- For stable library ABI, consider the Pimpl idiom
- 函数参数不要超过4个，若超过了考虑拆分函数或者将参数封装为结构体
- 不要以“指针+size”的形式传递数组，用span替代
- 相同类型的参数尽量避免位置上相邻，以防止实参误传（传颠倒了）而编译器又无法检测这种错误。

## Functions

- 一般情形下使用普通指针或者引用而非智能指针作为参数，慎用智能指针做参数，仅当存在所有权语义时使用（独占或共享）
- 考虑使用匿名lambda，如果需要一个函数类型的参数
- 函数参数及返回值指导原则：
  基本类型以及unique_ptr按值传递（传入f(T), 传出T f()）；
  类类型按引用传入（f(const T&)）,按值传出(T f())；
  大对象（大型结构体或者集合类对象）按const引用传入(f(const T&))，按引用参数传出（f(T&)）；
  特殊场景下返回值可以是T&，但T一定不能是局部变量。一般为某个传入参数。
  按值传递是最简单最安全的，而且访问起来最快（因为是直接访问的，指针及引用都涉及到间接访问（解引用）），如果可以尽量选用。
  不要返回const T
  慎用右值引用参数，若要用在注释内说明理由；
- 尽量显式指定lambda的捕获参数，默认的按值捕获[=]可能出现意料之外的行为。
  
  ````c++
  class My_class {
    int x = 0;
    // ...

    void f()
    {
        int i = 0;
        // ...

        auto lambda = [=/*实际对this指针进行的按值捕获*/] { use(i, x/*此处的x并非是一份拷贝，而是this->x*/); };   // BAD: "looks like" copy/value capture

        x = 42;
        lambda(); // calls use(0, 42); 期望的是x已经复制一份了，外部x的修改不影响lambda里面的x，然而修改却影响到了。
        x = 43;
        lambda(); // calls use(0, 43);

        // ...

        auto lambda2 = [i, this/*显式指定捕获的this指针*/] { use(i, x); }; // ok, most explicit and least confusing

        // ...
    }
  };
  ````

- 不要使用C的可变形参va_arg
  C++是强类型语言，va_arg无法做类型检验，有安全隐患，避免使用。
  可以用重载、变参模板、variant参数、initializer_list（同类型数据）或者不定数量参数标记“...”替代。
- 条件检查未通过提前return，避免嵌套大量的"if else"
  
  ````c++
  void foo(){
    // 不好的示范
    //if (condition){ // 条件满足
    //    ... // do something
    //}else{
    //    return;
    //}

    // 好的示范
    if (!condition){ // 条件不满足
        return; // 提前返回
    }
    ... // do something
  }

  ````

## Classes and class hierarchies

## Naming and layout suggestions

- 尽量让代码自解释而非用冗长的注释
- 注释说明意图而非具体实现
- 标识符命名不要包含类型信息
- 作用域越大标识符长度应该越长
- 保持一贯的命名风格，如全小写加下划线、驼峰式等，ISO C++推荐全小写加下划线
  
  ````c++
  
  ````

- 仅在宏定义中使用全大写（const常量、枚举也不要用全大写）
- 类内的内容按如下顺序布置：内部类型（class、enums、aliases）、特殊函数、普通函数、数据成员。public最前，private最后
- 使用C++风格的声明格式，如 int* p; 而非int *p。
- 不要使用void参数，保持空置就好。~~void f(void); ~~  void f();
- 头文件使用.h后缀（个人倾向.hpp以区分纯c头文件），源文件使用.cpp后缀。