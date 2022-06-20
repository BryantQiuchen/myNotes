# **C++ programs 代码基本形式**

![截屏2022-04-01 上午10.42.18](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 上午10.42.18.png)

**Header（头文件）的布局**

![截屏2022-04-01 上午10.54.42](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 上午10.54.42.png)

# **class 的声明（declaration）**

```cpp
class complex
{
public: // 访问级别
	complex (double r = 0, double i = 0) : re(r), im(i) { }	// inline 内联函数
    // initialization list (初值列，初始列)
    complex &operator += (const complex&);
    double real () const { return re; } // 内联函数
    double imag () const { return im; } // 内联函数
private:
	double re, im;
    friend complex & __doapl (complex*, const complex&);
    // 友元
};
// 自由取得 friend 的 private 成员
inline complex& __doapl (complex *ths, const complex &r)
{
    ths->re += r.re;
    ths->im += r.im;
    return *ths;
}

{
    complex c1(2, 1); // 绑定为double
    complex c2;
}
```

## **constructor（ctor，构造函数）**

![截屏2022-04-01 上午11.24.42](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 上午11.24.42.png)

函数名和 class 名相同，不需要有返回类型

initialization list 构造函数特有的写法 `re(r), im(i)`

`complex c2` 没有给定参数，调用默认实参

## **ctor可以有多个 —— overloading（重载）**

![截屏2022-04-01 上午11.32.16](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 上午11.32.16.png)

黄色部分的重载是否可行？

不可行，因为第一个有默认参数，黄色部分没有给定值，编译器在调用时不知道该调用哪一个构造函数

```cpp
{
	complex c1;
    complex c2();
}
```

## ctor 被放在 ==private== 区

不允许外界创建，外界调用只能通过`A::getInstance().setup();`

![截屏2022-04-01 上午11.36.51](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 上午11.36.51.png)

## 参数传递：pass by value vs. pass by reference (to const)

养成习惯，尽量 pass by reference，速度比 pass by value 快

![截屏2022-04-01 上午11.42.30](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 上午11.42.30.png)

## 返回值传递：return by value vs. return by refrerence(to const)

![截屏2022-04-01 上午11.46.49](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 上午11.46.49.png)

## 相同 class 的各个objects 互为 friends（友元）

`c2.func(c1)` 两个为相同的 class，可以直接调用

![截屏2022-04-01 下午4.24.53](侯捷C++面向对象高级开发.assets/截屏2022-04-01 下午4.24.53.png)

## operator overloading（操作符重载1，成员函数） ==this==

![截屏2022-04-01 下午4.53.35](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 下午4.53.35.png)

`this` 不能写到形参里，编译器默认其存在的，但在函数里面可以直接用

## operator overloading（操作符重载2，非成员函数） ==无this==

![截屏2022-04-01 下午5.19.22](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 下午5.19.22.png)

不可以 `return by reference`，因为它们返回的是个 **local object**

`typename()` 创建临时对象

![截屏2022-04-01 下午5.33.27](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-01 下午5.33.27.png)

不能用 `void` 返回后还是 `ostream`

## **ctor 和 dtor （构造函数和析构函数）**

![截屏2022-04-02 下午3.22.58](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-02 下午3.22.58.png)

## copy ctor（拷贝构造函数）

```cpp
inline
    String::String(const String &str)
{
	m_data = new char[strlen(str.m_data) + 1]; // 直接取另一个 object 的 private data
    strcpy(m_data, str.m_data);
}

{
    String s1("hello");
    String s2(s1);
    String s2 = s1;
}
```

## copy assignment opeator（拷贝赋值函数）

先杀掉自己的内存，再开辟拷贝需要的内存（需要+1），再进行拷贝

![截屏2022-04-02 下午3.31.44](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-02 下午3.31.44.png)

==一定要在operator中检测是否 self assignment==（如果本来指向的就是同一块内存，不检测自我赋值，直接杀掉，就会产生 undefined behavior）

## stack（栈）和 heap（堆）

stack objects的生命期

```cpp
class Complex {...};
...
{
    Complex c1(1,2);
} 
// 生命在作用域（scope）结束时就结束了，这种作用域内的object，又称为 auto object（因为会被自动清理）
```

static local objects 的生命期

```cpp
class Complex {...};
...
{	
    static Complex c1(1,2);
}
// 生命在作用域结束后仍存在，直到整个程序结束
```

global objects 的生命期

```cpp
class Complex {...};
...
Complex c3(1,2);
int main()
{
    ...
}
// c3是global object，其生命在整个程序结束之后才结束，也可以视为是一个static object
```

heap objects的生命期

```cpp
class Complex {...};
...
{	
    Complex *p = new Complex;
    ...
    delete p;
}
// 生命在被 delete 时结束，如果不进行 delete，会出现内存泄露（memory leak）当作用域结束，p 所指的heap object 仍存在，但指针 p 的生命却结束了，作用域之外再也看不到 p
```

## new ：先分配 memory，再调用 ctor

![截屏2022-04-02 下午3.57.44](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-02 下午3.57.44.png)

## delete : 先调用 dtor，再释放 memory

![截屏2022-04-02 下午3.57.12](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-02 下午3.57.12.png)

## static

![截屏2022-04-06 下午1.34.22](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午1.34.22.png)

静态函数没有 this pointer

调用 static 函数方式：

- 通过 object 调用
- 通过 class name 调用

## function template 函数模版

![截屏2022-04-06 下午1.46.59](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午1.46.59.png)

编译器会做实参推导（argument duduction）

# Object Oriented Programming, Object Oriented Design (OOP, OOD)

- Inheritance（继承)
- Composition（复合）
- Delegation（委托）

## Composition (复合)，表示 has -a

![截屏2022-04-06 下午2.01.31](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午2.01.31.png)

![截屏2022-04-06 下午2.01.49](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午2.01.49.png)

queue 里面有一个（或多个）deque，queue 里面的功能，都通过调用 deque 来实现

设计模式：Adapter

## Composition 关系下的构造和析构

![截屏2022-04-06 下午2.09.27](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午2.09.27.png)

（红色部分，都是编译器帮忙加上的，在调用构造函数时，编译器调用 ==Component 的默认构造函数==）

先后关系：==构造由内而外，析构由外而内==

## Delegation（委托）Composition by reference

用指针相连两个类，左边对外，所有的操作都在右边实现（编译防火墙）

![截屏2022-04-06 下午2.25.29](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午2.25.29.png)

## Inheritance （继承），表示 is -a

![截屏2022-04-06 下午2.32.53](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午2.32.53.png)

## Inheritance 关系下的构造和析构

子类对象有父类的成分

![截屏2022-04-06 下午2.36.05](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午2.36.05.png)

## Inheritance with ==virtual== functions（虚函数）

![截屏2022-04-06 下午5.59.32](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午5.59.32.png)

![截屏2022-04-06 下午6.09.46](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午6.09.46.png)

通过子类对象调用父类函数（在父类不做定义，父类定义为 virtual ，子类去定义 Serialize() 函数）

## Inheritance + Composition 关系下的构造和析构

![截屏2022-04-06 下午6.18.18](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午6.18.18.png)

## Delegation + Inheritance

![截屏2022-04-06 下午7.26.04](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-06 下午7.26.04.png)

为 Primititve 和 Composite 写一个父类 Component，这样 vector 中既可以放左边的东西，也可以放右边的东西，add 函数同理。

# 兼谈对象模型（II）

## conversersion function 转换函数

![截屏2022-04-12 上午10.38.07](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 上午10.38.07.png)

`Fraction` 转为 `double` 

## conversion function vs. non-explicit-one-argument ctor

![截屏2022-04-12 上午10.46.58](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 上午10.46.58.png)

error 原因：有两条可行的方法，编译器不知道应该走哪条路，会出现二义

## explicit-one-argument ctor

![截屏2022-04-12 上午10.50.45](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 上午10.50.45.png)

## function template 函数模版

![截屏2022-04-12 上午11.29.26](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 上午11.29.26.png)

编译器会对 function template 进行实参推导（argument deduction）

## member template 成员模版

```cpp
template <class T1, class T2>
struct pair {
    typedef T1 first_type;
    typedef T2 second_type;
    
    T1 first;
    T2 second;
    
    pair() : first(T1()), second(T2()) {}
    pair(const T1 &a, const T2 &b) : first(a), second(b) {}
    
    // 成员模版
    template <class U1, class U2>
    pair(const pair<U1, U2> &p) : first(p.first), second(p.second) {}
};
```

![截屏2022-04-12 上午11.41.45](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 上午11.41.45.png)

把一个鲫鱼➕麻雀构成的 pair，放进一个（拷贝到）一个由鱼类和鸟类构成的 pair 中，可以吗？反之？

可以，反之不可以

![截屏2022-04-12 下午3.12.16](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 下午3.12.16.png)

定义一个指针指向父类，给他设初值，指向子类 （up-cast）

## specialization 模版特化

对于独特的类型做特殊的设计

泛化与特化：

![截屏2022-04-12 下午3.17.07](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 下午3.17.07.png)

## partial specialization 模版偏特化

- **个数的偏**

- **范围的偏**

## template template parameter 模版模版参数

![截屏2022-04-12 下午3.25.20](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 下午3.25.20.png)

必须加上中间两行才能通过

## reference

![截屏2022-04-12 下午4.26.05](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 下午4.26.05.png)

reference 的常见用途：

通常不用于声明变量，而用于参数类型（parameters type）和返回类型（return type）的描述

![截屏2022-04-12 下午4.43.53](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-12 下午4.43.53.png)

## vptr 和 vtbl

![截屏2022-04-13 下午3.09.02](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-13 下午3.09.02.png)

子类对象有父类的成分，B、C继承了A中的函数和data1、data2

p 动态绑定

## const

![截屏2022-04-13 下午3.39.36](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-13 下午3.39.36.png)

const属于函数签名的一部分，故加和不加可以同时存在，不会出现二义

## 重载 member operator new/delete

![截屏2022-04-13 下午3.58.39](/Users/ryan/Documents/notes/images/cppNotes/截屏2022-04-13 下午3.58.39.png)