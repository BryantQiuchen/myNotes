

# 类型推导

## Item 1: 模板类型推断机制

`auto` 推断的基础是模板类型推断机制，但部分特殊情况下，模板推断机制不适用于 `auto`

```c++
template <typename T>
void f(ParamType param);  // ParamType 即 param 的类型
```

进行调用

```c++
f(expr);
```

编译期间，编译器用 expr 推断 T 和 ParamType，实际上两者通常不一致，比如

```c++
template <typename T>
void f(const T& param);

int x = 0;
f(x);   // T 被推断为 int，ParamType 被推断为 const int&，T 的推断类型与 expr 和 ParamType 相关
```

`T`的类型推导不仅取决于 `expr` 的类型，也取决于 `ParamType` 的类型。

下面的情形都基于这个模版：

```c++
template<typename T>
void f(ParamType param);

f(expr);                        // 从 expr 中推导 T 和 ParamType
```

### 情形 1: `ParamType` 不是引用或指针

丢弃 `expr` 的 cv 限定符（`const` 和 `volatile`）和引用限定符，最后得到的 `expr` 类型就是 T 和 `ParamType` 类型：

```c++
// 方便写，未定义，下同
template <typename T>
void f(T param);

int a;
const int b;
const int& c;

int* p1;
const int* p2;
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";

// 以下情况 T 和 ParamType 都是 int
f(a);
f(b);
f(c);
// 指针类型丢弃的是 top-level const（即指针本身的 const）
// low-level const（即所指对象的 const）会保留
f(p1);  // T 和 ParamType 都是 int*
f(p2);  // T 和 ParamType 都是 const int*
f(p3);  // T 和 ParamType 都是 int*
f(p4);  // T 和 ParamType 都是 const int*
// char 数组会退化为指针
f(s1);  // T 和 ParamType 都是 char*
f(s2);  // T 和 ParamType 都是 const char*
```

### 情形 2: `ParamType` 是引用类型

如果 `expr` 的类型是引用，保留 cv 限定符，ParamType 一定是左值引用类型，ParamType 去掉引用符就是 T 的类型，即 T 一定不是引用类型。

```c++
template <typename T>
void f(T& param);

int a;
int& b;
int&& c;
const int d;
const int& e;

int* p1;
const int* p2;
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";

f(a);  // ParamType 是 int&，T 是 int
f(b);  // ParamType 是 int&，T 是 int
f(c);  // ParamType 是 int&，T是int
f(d);  // ParamType 是 const int&，T 是 const int
f(e);  // ParamType 是 const int&，T 是 const int
// 因为 top-level const 和 low-level const 都保留
// 对于指针只要记住 const 的情况和实参类型一样
f(p1);  // ParamType 是 int* &，T 是 int*
f(p2);  // ParamType 是 const int* &，T 是const int*
f(p3);  // ParamType 是 int* const&，T 是 int* const
f(p4);  // ParamType 是 const int* const &，T 是 const int* const
// 数组类型对于 T& 的情况比较特殊，不会退化到指针
f(s1);  // ParamType 是 char(&)[9]，T 是 char[9]
f(s2);  // ParamType 是 const char(&)[9]，T 是 const char[9]
```

如果把 ParamType 从 T& 改为 const T&，区别只是 ParamType 一定为 top-level const，ParamType 去掉 top-level const 和引用符就是 T 类型，T 不一定是 top-level const 引用类型：

```c++
template <typename T>
void f(const T& param);

int a;
int& b;
int&& c;
const int d;
const int& e;

int* p1;
const int* p2;
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";

// 以下情况 ParamType 都是 const int&，T 都是 int
f(a);
f(b);
f(c);
f(d);
f(e);
// 数组类型类似
f(s1);  // ParamType 是 const char(&)[9]，T 是 char[9]
f(s2);  // ParamType 是 const char(&)[9]，T 是 char[9]
// 对于指针只要记住，T的指针符后一定无const
f(p1);  // ParamType 是 int* const &，T 是 int*
f(p2);  // ParamType 是 const int* const &，T 是 const int*
f(p3);  // ParamType 是 int* const&，T 是 int*
f(p4);  // ParamType 是 const int* const &，T 是 const int*
```

### 情形 3: `ParamType` 是指针类型

同情形 2 类似，ParamType 一定是 non-const 指针（传参时忽略 top-level const）类型，去掉指针符就是 T 的类型，即 T 一定不为指针类型

```c++
template <typename T>
void f(T* param);

int a;
const int b;

int* p1;
const int* p2;
int* const p3;        // 传参时与 p1 类型一致
const int* const p4;  // 传参时与 p2 类型一致

char s1[] = "downdemo";
const char s2[] = "downdemo";

f(&a);  // ParamType 是 int*，T 是 int
f(&b);  // ParamType 是const int*，T 是 const int

f(p1);  // ParamType 是 int*，T 是int
f(p2);  // ParamType 是 const int*，T 是 const int
f(p3);  // ParamType 是 int*，T 是 int
f(p4);  // ParamType 是 const int*，T 是 const int

// 数组类型会转为指针类型
f(s1);  // ParamType 是 char*，T 是 char
f(s2);  // ParamType 是 const char*，T 是 const char
```

如果 ParamType 是 const-pointer，ParamType 会多出 top-level const，T 不变

```c++
template <typename T>
void f(T* const param);

int a;
const int b;

int* p1;        // 传参时与 p3 类型一致
const int* p2;  // 传参时与 p4 类型一致
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";

f(&a);  // ParamType 是 int* const，T 是 int
f(&b);  // ParamType 是 const int* const，T 是 const int

f(p1);  // ParamType 是 int* const，T 是 int
f(p2);  // ParamType 是 const int* const，T 是 const int
f(p3);  // ParamType 是 int* const，T 是 int
f(p4);  // ParamType 是 const int* const，T 是 const int

f(s1);  // ParamType 是 char* const，T 是 char
f(s2);  // ParamType 是 const char* const，T 是 const char
```

如果 ParamType 是 pointer to const，则只有一种结果，T 一定是不带 const 的非指针类型

```c++
template <typename T>
void f(const T* param);

template <typename T>
void g(const T* const param);

int a;
const int b;

int* p1;
const int* p2;
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";

// 以下情况ParamType 都是 const int*，T 都是 int
f(&a);
f(&b);
f(p1);
f(p2);
f(p3);
f(p4);
// 以下情况ParamType 都是 const int* const，T 都是 int
g(&a);
g(&b);
g(p1);
g(p2);
g(p3);
g(p4);
// 以下情况ParamType 都是 const char*，T 都是 char
f(s1);
f(s2);
g(s1);
g(s2);
```

### 情形 4: `ParamType` 是通用引用

如果 expr 是左值，T 和 ParamType 都推断为左值引用，这有两点非常特殊：

- 这是 T 被推断为引用的唯一情形
- ParamType 使用右值引用语法，却被推断为左值引用

如果 expr 是右值，则 ParamType 推断为右值引用类型，去掉 && 就是 T 的类型，即 T 一定不为引用类型。

```c++
template <typename T>
void f(T&& x);

int a;
const int b;
const int& c;
int&& d = 1;  // d 是右值引用，也是左值，右值引用是只能绑定右值的引用而不是右值

char s1[] = "downdemo";
const char s2[] = "downdemo";

f(a);  // ParamType 和 T 都是 int&
f(b);  // ParamType 和 T 都是 const int&
f(c);  // ParamType 和 T 都是 const int&
f(d);  // ParamType 和 T 都是 const int&
f(1);  // ParamType 是 int&&，T 是 int

f(s1);  // ParamType 和 T 都是 char(&)[9]
f(s2);  // ParamType 和 T 都是 const char(&)[9]
```

### 特殊: expr 是函数名

```c++
template <typename T>
void f1(T x);

template <typename T>
void f2(T& x);

template <typename T>
void f3(T&& x);

void g(int);

f1(g);  // T 和 ParamType 都是 void(*)(int)
f2(g);  // ParamType 是 void(&)(int)，T 是 void()(int)
f3(g);  // T 和 ParamType 都是 void(&)(int)
```

## Item 2: `auto` 类型推断机制



## Item 3: 理解 `decltype`



## Item 4: 学会查看类型推导结果

# `auto`

# 移步现代 C++

## Item 7: 区别使用 () 和 {} 创建对象

值初始化方式：

```c++
int a(0);
int b = 0;
int c{0};
int d = {0};  // 按 int d{0} 处理，后续讨论将忽略这种用法
```

使用等号不一定是赋值，也可能是拷贝，尤其是对于用户自定义的类型

```c++
Widget w1;              //调用默认构造函数
Widget w2 = w1;         //不是赋值运算，调用拷贝构造函数
w1 = w2;                //是赋值运算，调用拷贝赋值运算符（copy operator=）
```

C++ 11 使用统一初始化，大括号初始化可以方便地为容器指定初始元素：

```c++
std::vector<int> v{1, 2, 3};
```

对于不可拷贝的对象可以使用小括号或者大括号初始化，不能用“=”初始化：

```c++
std::atomic<int> ai1{ 0 };      //没问题
std::atomic<int> ai2(0);        //没问题
std::atomic<int> ai3 = 0;       //错误！
```

括号表达式有一个异常的特性，它不允许内置类型间隐式的变窄转换（*narrowing conversion*），使用小括号和"="的初始化不检查是否转换为变窄转换。

```c++
double x = 1.1;
double y = 2.2;
int a{x + y};  // 错误：大括号初始化不允许 double 到 int 的收缩转换
int b(x + y);   // OK：double 被截断为 int
int c = x + y;  // OK：double 被截断为 int
```

大括号初始化不存在 C++ 最令人苦恼的解析：

```c++
struct A {
  A() { std::cout << 1; }
};

struct B {
  B(std::string) { std::cout << 2; }
};

A a();  // 不调用 A 的构造函数，而是被解析成一个函数声明：A a();
std::string s{"hi"};
B b(std::string(s));  // 不构造 B，被解析成函数声明 B b(std::string)
A a2{};               // 构造 A
B b2{std::string(s)};  // 构造 B

// C++11 之前的解决办法
A a3;
B b3((std::string(s)));
```

但大括号存在缺陷，只要类型转换后可以匹配，大括号初始化总会优先匹配参数类型为 `std::initializer_list` 的构造函数，即使收缩转换会导致调用错误

```c++
class A {
 public:
  A(int) { std::cout << 1; }
  A(std::string) { std::cout << 2; }
  A(std::initializer_list<int>) { std::cout << 3; }
};

A a{0};     // 3
A b{3.14};  // 调用第三个，但会发生错误：大括号初始化不允许 double 到 int 的收缩转换
A c{"hi"};  // 2
```

但特殊的是，参数为空的大括号初始化只会调用默认构造函数。如果想传入真正的空 `std::initializer_list`，需要额外加一层小括号或大括号

```c++
class A {
 public:
  A() { std::cout << 1; }
  A(std::initializer_list<int>) { std::cout << 2; }
};

A a{};    // 1
A b{{}};  // 2
A c({});  // 2
```

上述问题带来的实际影响很大，`std::vector`有一个非`std::initializer_list`构造函数允许你去指定容器的初始大小，以及使用一个值填满你的容器。但它也有一个`std::initializer_list`构造函数允许你使用花括号里面的值初始化容器。如果你创建一个数值类型的`std::vector`（比如`std::vector<int>`），然后你传递两个实参，把这两个实参放到小括号和放到花括号中大不相同：

```c++
std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                //创建一个包含10个元素的std::vector，
                                //所有的元素的值都是20
std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                //创建包含两个元素的std::vector，
                                //元素的值为10和20
```

编写模版时，类似 `std::vector` 这种问题应当被避免，`std::initializer_list`重载不会和其他重载函数比较，它直接盖过了其它重载函数，其它重载函数几乎不会被考虑。所以如果你要加入`std::initializer_list`构造函数，请三思而后行。

标准库函数`std::make_unique`和`std::make_shared` 解决方法是使用小括号，并被记录在文档中作为接口的一部分。

```c++
auto p = std::make_shared<std::vector<int>>(3, 6);
for (auto x : *p) {
  std::cout << x;  // 666
}
```

## Item 8: 优先考虑 `nullptr` 而非 0 或 `NULL`

字面值 0 本质是 int 而非指针，只有在使用指针的语境中发现 0 才会解释为空指针

## Item 9: 优先考虑别名声明而非 `typedef`

一般来说别名声明和 `typedef` 做的是完全一样的事情，但声明一个函数指针时，别名声明更容易理解：

```c++
// FP是一个指向函数的指针的同义词，它指向的函数带有
// int和const std::string&形参，不返回任何东西
typedef void (*FP)(int, const std::string&);
using FP = void (*)(int, const std::string&);
```

特别的，C++11 引入了别名模版，它只能使用 `using` 别名声明

```c++
template <typename T>
using Vector = std::vector<T>;  // Vector<int> 等价于 std::vector<int>

Vector<int> lw;		// 用户代码

// C++11 之前的做法是在模板内部 typedef
template <typename T>
struct V {  // V<int>::type 等价于 std::vector<int>
  typedef std::vector<T> type;
};

V<int>::type ll;		// 用户代码

// 在其他类模板中使用这两个别名的方式
template <typename T>
struct A {
  Vector<T> a;
  typename V<T>::type b;
};
```

C++14 提供了 C++11 所有 type traits 转换的别名声明版本

```c++
template <typename T>
struct remove_reference {
  using type = T;
};

template <typename T>
struct remove_reference<T&> {
  using type = T;
};

template <typename T>
struct remove_reference<T&&> {
  using type = T;
};

template <typename T>
using remove_reference_t = typename remove_reference<T>::type;
```

## Item 10: 用 `enum class` 代替 `enum`

一般大括号中声明的名称，只会限制在花括号之内，但对 `enum` 不成立，`enum` 成员属于 `enum` 所在的作用域，作用域内不能出现同名实例：

```c++
enum X { a, b, c };
int a = 1;  // 错误：a 已在作用域内声明过
```

这些枚举名的名字泄漏进它们所被定义的`enum`在的那个作用域，这个事实有一个官方的术语：未限域枚举(*unscoped `enum`*)。在C++11中它们有一个相似物，限域枚举(*scoped `enum`*)，它不会导致枚举名泄漏，使用 `enum class` 表示：

```c++
enum class X { a, b, c };
int a = 1;   // OK
X x = X::a;  // OK
X y = b;     // 错误
```

使用限域 `enum` 来减少命名空间的污染，这是一个足够合理使用它而不是未限域 `enum` 的理由，除此之外，在它的作用域中，枚举名是强类型，不会进行隐式转换：

```C++
enum X { a, b, c };
X x = a;
if (x < 3.14) {  // 不应该将枚举与浮点数进行比较，但这里合法
}

enum class Y { a, b, c };
Y y = Y::a;
if (x < 3.14) {  // 错误：不允许比较
}
if (static_cast<double>(x) < 3.14) {  // OK：enum class 允许强制转换为其他类型
}
```

限域 `enum` 可以被前置声明，可以不指定枚举名直接声明：

```c++
enum Color;         //错误！
enum class Color;   //没问题
```

C++11 之前不能前置声明 `enum` 的原因是，编译器为了节省内存，要在 `enum` 被使用前选择一个足够容纳成员取值的最小整型作为底层类型：

```c++
enum X { a, b, c };  // 编译器选择底层类型为 char
enum Status {        // 编译器选择比 char 更大的底层类型
  good = 0,
  failed = 1,
  incomplete = 100,
  corrupt = 200,
  indeterminate = 0xFFFFFFFF
};
```

不能前置声明的一个弊端是，由于编译依赖关系，在 `enum` 中仅仅添加一个成员可能就要重新编译整个系统。如果在头文件中包含前置声明，修改 `enum class` 的定义时就不需要重新编译整个系统，如果 `enum class` 的修改不影响函数的行为，则函数的实现也不需要重新编译。

C++11 支持前置声明的原因很简单，底层类型是已知的，用 `std::underlying_type` 即可获取。也可以指定枚举的底层类型，如果不指定，`enum class` 默认为 `int`，`enum` 则不存在默认类型，可以重写：

```c++
enum class Status: std::uint32_t; // Status的底层类型是std::uint32_t
```

非限域的 `enum` 在某些情况是很有用的，当牵扯到 C++11 的 `std::tuple` 的时候，能够利用到非限域 `enum` 的隐式转换时：

```c++
enum X { name, age, number };
auto t = std::make_tuple("downdemo", 6, "42");
auto x = std::get<name>(t);  // name 可隐式转换为 get 的模板参数类型 size_t
```

如果使用限域 `enum class` 就比较麻烦。

## Item 11: 用 `=delete` 替代 `private` 作用域来禁用函数

C++11 之前的禁用拷贝的方式时将拷贝构造函数和拷贝赋值运算符声明在 `private` 作用域中

```c++
class A {
 private:
  A(const A&);  // 不需要定义
  A& operator(const A&);
};
```

C++11 中可以直接将要删除的函数用 `=delete` 声明，习惯上会声明在 `public` 作用域中，这样在使用删除的函数时，会先检查访问权再检查删除状态，出错时能得到更明确的诊断信息

```c++
class A {
 public:
  A(const A&) = delete;
  A& operator(const A&) = delete;
};
```

`private` 作用域中的函数还可以被成员和友元调用，而 `=delete` 是真正禁用了函数，无法通过任何方法调用，任何函数都可以用 `=delete` 声明，比如函数不想接受某种类型的参数，就可以删除对应类型的重载：

```c++
void f(int);
void f(double) = delete;  // 拒绝 double 和 float 类型参数

f(3.14);  // 错误
```

`=delete` 还可以禁止一些模版的实例化，假如要求一个模版仅支持原生指针：

```c++
template<typename T>
void processPointer(T* ptr);
```

在指针的世界里有两种特殊情况。一是 `void*` 指针，因为没办法对它们进行解引用，或者加加减减等。另一种指针是 `char*`，因为它们通常代表C风格的字符串，而不是正常意义下指向单个字符的指针。这两种情况要特殊处理，在 `processPointer` 模板里面，我们假设正确的函数应该拒绝这些类型。也即是说，`processPointer` 不能被 `void*` 和 `char*` 调用。要想确保这个很容易，使用 `delete` 标注模板实例：

```c++
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;
```

如果类中有一个函数模版，可以使用 `private`（经典的 C++98 惯例）来禁止这些函数模板实例化，但是不能这样做，因为不能给特化的成员模板函数指定一个不同于主函数模板的访问级别。模版特例化必须位于一个命名空间作用域，而不是类作用域，因为它不需要一个不同的访问级别，且可以在类外被删除：

```c++
class Widget {
public:
    …
    template<typename T>
    void processPointer(T* ptr)
    { … }
    …

};

template<>                                          //还是public，
void Widget::processPointer<void>(void*) = delete;  //但是已经被删除了
```

## Item 12: 使用 `override` 声明重写函数

虚函数的重写（override）很容易出错，因为要在派生类中重写虚函数，必须满足一系列要求

- 基类中必须有此虚函数
- 基类和派生类的函数名相同（析构函数除外）
- 函数参数类型相同
- const 属性相同
- 函数返回值和异常说明相同

C++11 还多了一条要求，就是引用修饰符相同。引用修饰符的作用就是：指定成员函数仅在对象为左值（成员函数标记为 &）或右值（成员函数标记为 &&）时可用。

对于如此多的要求，难以面面俱到，下面的代码没有任何的重写，但却可以通过编译，大部分的编译器也不会发出 warnings 信息。

```c++
struct A {
 public:
  virtual void f1() const;
  virtual void f2(int x);
  virtual void f3() &;
  void f4() const;
};

struct B : A {
  virtual void f1();
  virtual void f2(unsigned int x);
  virtual void f3() &&;
  void f4() const;
};
```

为了保证正确性，C++11 提供了 `override` 来标记要重写的虚函数，如果未重写就不能通过编译（`override` 只有出现在成员函数声明末尾才有意义）

```c++
struct A {
  virtual void f1() const;
  virtual void f2(int x);
  virtual void f3() &;
  virtual void f4() const;
};

struct B : A {
  virtual void f1() const override;
  virtual void f2(int x) override;
  virtual void f3() & override;
  void f4() const override;
};
```

除此之外，还提供了另一个关键字：`final`，它可以用来制定虚函数禁止被重写：

```c++
struct A {
  virtual void f() final;
  void g() final;  // 错误：final 只能用于指定虚函数
};

struct B : A {
  virtual void f() override;  // 错误：f 不可重写
};
```

`final` 还可以用于指定某个类禁止被继承：

```c++
struct A final {};
struct B : A {}; // 错误，A 禁止被继承
```

## Item 13: 优先考虑 const_iterator 而非 iterator

STL 中 `const_iterator` 等价于指向常量的指针，都指向不能被修改的值。当我们需要一个迭代器，且不需要修改迭代器指向的值时，应当使用 `const_iterator`

需要迭代器但不修改值时就应该使用 `const_iterator`，获取和使用 `const_iterator` 十分简单

```C++
std::vector<int> v{2, 3};
auto it = std::find(std::cbegin(v), std::cend(v), 2);  // C++14
v.insert(it, 1);
```

上述功能很容易扩展成模板

```C++
template <typename C, typename T>
void findAndInsert(C& c, const T& x, const T& y) {
  auto it = std::find(std::cbegin(c), std::cend(c), x);
  c.insert(it, y);
}
```

C++11 没有 `std::begin()` 和 `std::send()`，手动实现即可

```C++
template <typename C>
auto cbegin(const C& c) -> decltype(std::begin(c)) {
  return std::begin(c);  // c 是 const 所以返回 const_iterator
}
```

## Item 14: 用 `noexcept` 标记不抛出异常的函数

C++98 中，必须指出一个函数可能抛出的所有异常类型，如果函数有所改动那么 **exception specification** 也要修改，而这可能破坏代码，因为调用者可能依赖于原本的 **exception specification**。

在 C++11 中，达成了共识，真正需要关心的是一个函数会不会抛出异常：一个函数要么可能抛出异常，要么绝不抛出异常，这种 maybe-or-never 形成了 C++11 **exception specification** 的基础，C++98 的 **exception specification** 在 C++17中被移除。

函数是否要加上 `noexcept` 声明与接口设计相关，调用者可以查询函数的 `noexcept` 状态，查询结果将影响代码的异常安全性和执行效率。因此函数是否要声明为 `noexcept` 就和成员函数是否要声明为 `const` 一样重要，如果一个函数不抛异常却不为其声明 `noexcept`，这就是接口规范缺陷。

`noexcept` 的一个额外优点是，它可以让编译器生成更好的目标代码。为了理解原因只需要考虑 C++98 和 C++11 表达函数不抛异常的区别

```c++
int f(int x) throw();   // C++98
int f(int x) noexcept;  // C++11
```

`noexcept` 声明的函数中，如果异常传出函数，优化器不需要保持栈在运行期的展开状态，也不需要在异常逃出时，保证其中所有的对象按构造顺序的逆序析构。而声明为 `throw()` 的函数就没有这样的优化灵活性。

## Item 15: 使用 `constexpr` 表示编译器常量

`constexpr` 用于对象时就是一个加强版的 `const`，表面看 `constexpr` 表示的值是 `const`，且在编译器已知，但用于函数则有不同的意义。编译期已知的值可能被放进只读内存，这对嵌入式开发是一个很重要的语法特性，`constexpr` 在调用时如果传入的编译期常量，则产出编译期常量，传入运行期才知道的值，则产出运行期值。

## Item 16: 使用 `std::mutex` 或 `std::atomic` 保证 `const` 成员函数线程安全

`const` 成员函数不会修改成员变量，即对变量进行只读操作，但是即使是只读，函数也不是线程安全的，假设有一个表示多项式的类，它包含一个返回根的 `const` 成员函数：

```c++
class Polynomial {
 public:
  std::vector<double> roots() const {
    if (!roots_are_valid_) {
      ... // 计算 root_vals_
      roots_are_valid_ = true;
    }
    return root_vals_;
  }

 private:
  mutable bool roots_are_valid_{false};
  mutable std::vector<double> root_vals_{};
};
```

从概念上讲，`roots` 不改变它所操作的 `Polynomial` 对象，但是作为缓存的一部分，它也许会改变 `roots_vals_` 和 `roots_are_valid` 的值。假设此时有两个线程对同一个对象调用成员函数，虽然函数声明为 `const`，但由于函数内部修改了数据成员，就可能产生数据竞争，最简单的方法是引入 `std::mutex`：

```c++
class Polynomial {
 public:
  std::vector<double> roots() const {
    std::lock_guard<std::mutex> lk{m_};
    if (!roots_are_valid_) {
      ... // 计算 root_vals_
      roots_are_valid_ = true;
    }
    return root_vals_;
  }

 private:
  mutable std::mutex m_;
  mutable bool roots_are_valid_{false};
  mutable std::vector<double> root_vals_{};
};
```

对一些简单的情况，使用原子变量 `std::atomic` 开销更低（取决于机器以及 `std::mutex` 的实现）

```c++
class Point {
 public:
  double distance_from_origin() const noexcept {
    ++call_count_;  // 计算调用次数
    return std::sqrt((x_ * x_) + (y_ * y_));
  }

 private:
  mutable std::atomic<unsigned> call_count_{0};
  double x_;
  double y_;
};
```

因为原子变量 `std::atomic` 开销更低，容易想到用多个原子变量来进同步：

```c++
class A {
 public:
  int f() const {
    if (flag_) {
      return res_;
    } else {
      auto x = expensive_computation1();
      auto y = expensive_computation2();
      res_ = x + y;   // 1
      flag_ = true;  // 2
      return res_;
    }
  }

 private:
  mutable std::atomic<bool> flag_{false};
  mutable std::atomic<int> res_;
};
```

这样做可行，但如果多个线程同时观察到标记值为 false，每个线程都要继续进行运算，这个标记反而没起到作用，造成重复运算。

先设置标记再计算可以消除这个问题（交换1和2的顺序），但会引起一个更大的问题：假设线程1刚设置好标记，线程2正好检查到标记值为 true 并直接返回数据值，接着线程1继续计算结果，这样线程2的返回值就是错误的。

因此，同步多个变量或内存区，最好还是使用 `std::mutex`

```c++
class A {
 public:
  int f() const {
    std::lock_guard<std::mutex> lk{m_};
    if (flag_) {
      return res_;
    } else {
      auto x = expensive_computation1();
      auto y = expensive_computation2();
      res_ = x + y;
      flag_ = true;
      return res_;
    }
  }

 private:
  mutable std::mutex m_;
  mutable bool flag_{false};
  mutable int res_;
};
```

## Item 17: 理解特殊成员函数的生成

特殊成员函数是指 C++ 自己生成的函数，C++98 有四个：默认构造函数，析构函数，拷贝构造函数，拷贝赋值运算符，这些函数仅在需要时生成，比如某个代码使用它们但是它们没有在类中声明。

C++11 的特殊成员函数多了两个：移动构造函数和移动赋值运算符：

```c++
struct A {
  A(A&& rhs);             // 移动构造函数
  A& operator=(A&& rhs);  // 移动赋值运算符
};
```

移动操作同样会在需要时生成，执行的是对 non-static 成员的移动操作，另外它们也会对基类部分执行移动操作。

- 两个拷贝操作是独立的（拷贝构造函数和拷贝复制运算符）：声明一个不会限制编译器生成另一个。

- 两个移动操作不是相互独立的。如果你声明了其中一个，编译器就不再生成另一个。理由是如果声明了移动构造函数，可能意味着实现上与编译器默认按成员移动的移动构造函数有所不同，从而可以推断移动赋值操作也应该与默认行为不同。

- 显式声明拷贝操作（即使声明为 =delete）会阻止自动生成移动操作（但声明为 =default 不阻止生成）。理由类似上条，声明拷贝操作可能意味着默认的拷贝方式不适用，从而推断移动操作也应该会默认行为不同。
- 声明移动操作（构造或赋值）使得编译器禁用拷贝操作。（禁用的是自动生成的拷贝操作，对用户声明的拷贝操作不受影响）
- C++11 规定，显式声明析构函数会阻止生成移动操作。这个规定源于 Rule of Three，即两种拷贝函数和析构函数应该一起声明。这个规则的推论是，如果声明了析构函数，则说明默认的拷贝操作也不适用，但 C++98 中没有重视这个推论，因此仍可以生成拷贝操作，而在 C++11 中为了保持不破坏遗留代码，保留了这个规则。

所以仅当下面条件成立时才会生成移动操作（当需要时）：

- 类中没有拷贝操作
- 类中没有移动操作
- 类中没有用户定义的析构

# 智能指针

# 右值引用、移动语义、完美转发

**移动语义**使得编译器有可能用廉价的移动操作来代替昂贵的拷贝操作。正如拷贝构造函数和拷贝赋值操作符给了你控制拷贝语义的权力，移动构造函数和移动赋值操作符也给了你控制移动语义的权力。移动语义也允许创建只可移动（*move-only*）的类型，例如 `std::unique_ptr`，`std::future` 和 `std::thread`。

**完美转发**使接收任意数量实参的函数模版成为可能，它可以将任意实参传递转发到其他函数，使目标函数接收到的实参与被传递给转发函数的实参保持一致。

**右值引用**是连接这两个截然不同的概念的胶合剂，它是使移动语义和完美转发变得可能的基础语言机制。

**牢记形参永远是左值，即使它的类型是一个右值引用**：

```cpp
void f(Widget &&w);
```

形参 `w` 是一个左值，即使它的类型是一个 rvalue-reference-to-`Widget`。

## Item 23: 理解 `std::move` 和 `std::forward` 只是一种强制类型转换

`std::move` 不移动任何东西，`std::forward` 也不转发任何东西。 `std::move` 和 `std::forward` 仅仅是执行转换（cast）的函数，`std::move` 无条件的将它的实参转换为右值，而 `std::forward` 只在特定情况满足时下进行转换。

`std::move` 的一个不完全标准实现：

```c++
template<typename T>                            //在std命名空间
typename remove_reference<T>::type&&
move(T&& param)
{
    using ReturnType =                          //别名声明，见条款9
        typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}
```

`std::move` 接受一个对象的引用（准确的说是一个通用引用），返回一个指向同对象的引用，右值本来就是移动操作的候选者，所以对一个对象使用 `std::move` 就是告诉编译器，这个对象很适合被移动。所以这就是为什么 `std::move` 叫现在的名字：更容易指定可以被移动的对象。

可能会出现，传入一个右值，但却执行拷贝操作：

```c++
class Annotation {
public:
    explicit Annotation(const std::string text)
    ：value(std::move(text))    // 转为 const std::string&&
    { … }                       // 调用 std::string(const std::string&)
    
    …

private:
    std::string value;
};
```

`std::string` 的构造函数被调用时，有两种可能性：

```c++
class string {
public:
    ...
    string(const string& rhs);	// 拷贝构造函数
    string(string&& rhs);		// 移动构造函数		
}
```

在类 `Annotation` 的构造函数的成员初始化列表中，`std::move(text) `的结果是一个 `const std::string `的右值。这个右值不能被传递给 `std::string` 的移动构造函数，因为移动构造函数只接受一个指向 **non-const** 的 `std::string ` 的右值引用。然而，该右值却可以被传递给 `std::string` 的拷贝构造函数，因为 lvalue-reference-to-const 允许被绑定到一个 `const` 右值上。因此，`std::string` 在成员初始化的过程中调用了拷贝构造函数，即使 `text` 已经被转换成了右值。这样是为了确保维持 `const` 属性的正确性。从一个对象中移动出某个值通常代表着修改该对象，所以语言不允许 `const` 对象被传递给可以修改他们的函数（例如移动构造函数）。

因此，可以做出总结：

- 不要在希望能移动对象时，声明为 `const`，对 `const` 对象的移动请求会悄无声息的被转化为拷贝操作；
- `std::move` 不仅不移动任何东西，也不保证它执行转换的对象可以被移动。

关于 `std::move`，你能确保的唯一一件事就是将它应用到一个对象上，你能够得到一个右值。

`std::forward` 是有条件的转换，最常见的就是一个模版函数，接收一个通用引用形参，并把它传递给另外的函数：

```c++
void process(const Widget& lvalArg);        //处理左值
void process(Widget&& rvalArg);             //处理右值

template<typename T>                        //用以转发param到process的模板
void logAndProcess(T&& param)
{
    auto now =                              //获取现在时间
        std::chrono::system_clock::now();
    
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
}
```

对 `logAndProcess` 的调用，一次左值为实参，一次右值为实参：

```c++
Widget w;

logAndProcess(w);               //用左值调用
logAndProcess(std::move(w));    //用右值调用
```

`logAndProcess` 和所有的其他函数行参都一样，是一个左值，每次内部在调用 `process` 时，都会调用此函数的左值重载版本，因此需要一个机制：当且仅当传递给函数 `logAndProcess` 的用以初始化 `param` 的实参是一个右值，`param` 会被转换一个右值，这就是 `std::forward` 做的事情：它的实参用右值初始化时，转换为一个右值。

`std::move` 的使用代表着无条件向右值的转换，而使用 `std::forward` 只对绑定了右值的引用进行到右值转换。这是两种完全不同的动作。前者是典型地为了移动操作，而后者只是传递（亦为转发）一个对象到另外一个函数，保留它原有的左值属性或右值属性。因为这些动作实在是差异太大，所以我们拥有两个不同的函数（以及函数名）来区分这些动作。

**关键**：

- `std::move` 执行到右值的无条件的转换，但就自身而言，它不移动任何东西；
- `std::forward` 只有当它的参数被绑定到一个右值时，才将参数转换为右值；
- `std::move` 和 `std::forward` 在运行期什么也不做。

## Item 24: 区分通用引用和右值引用

带 `&&` 的不一定是右值引用，这种不确定类型的引用被称为转发引用：

```c++
void f(Widget&& param);             //右值引用
Widget&& var1 = Widget();           //右值引用
auto&& var2 = var1;                 //不是右值引用

template<typename T>
void f(std::vector<T>&& param);     //右值引用

template<typename T>
void f(T&& param);                  //不是右值引用
```

`T&&` 有两种不同的意思：

- 右值引用。它们只能绑定到右值上，并且它们主要的存在原因就是为了识别可以移动操作的对象；
- 既可以是右值引用，也可以是左值引用。这种引用在源码里看起来像右值引用（即 `T&&`），但是它们可以表现得像是左值引用（即 `T&` ）。它们的二重性使它们既可以绑定到右值上（就像右值引用），也可以绑定到左值上（就像左值引用）。它们可以绑定到几乎任何东西。这种空前灵活的引用值得拥有自己的名字。我把它叫做**通用引用**（*universal references*）。

两种情况下会出现通用引用：

- 函数模版形参

```c++
template<typename T>
void f(T&& param);                  //param是一个通用引用
```

- `auto` 声明符

```c++
auto&& var2 = var1;					//var2是一个通用引用
```

两种情况共同情况都是存在类型推导（type deduction）。如果 `T&&` 不带有类型推导，那么看到的就是一个右值引用，通用引用必须严格按照 `T&&` 的形式：

```c++
template<typename T>
void f(std::vector<T>&& param); 	//param是一个右值引用

std::vector<int> v;
f(v);								//报错，不能将左值绑定到右值引用
```

`T&&` 在模板内部并不保证一定会发生类型推导：

```c++
template <class T, class Allocator = allocator<T>>
class vector {
 public:
  void push_back(T&& x);  // 右值引用

  template <class... Args>
  void emplace_back(Args&&... args);  // 通用引用
};

std::vector<A> v;  // 实例化指定了 T

// 对应的实例化为
class vector<A, allocator<A>> {
 public:
  void push_back(A&& x);  // 不涉及类型推断，右值引用

  template <class... Args>
  void emplace_back(Args&&... args);  // 通用引用
};
```

函数 `push_back` 不包含任何类型推导。`push_back `对于 `vector<T>` 而言（有两个函数——它被重载了）总是声明了一个类型为 rvalue-reference-to-T 的形参。

而 `emplace_back`，包含类型推导，类型参数 `Args` 是独立于 `vector` 的类型参数 `T`，所以 `Args` 会在每次 `emplace_back` 被调用的时候被推导。

声明为 `auto&&` 的变量是通用引用，因为一定会发生类型推导。完美转发中，如果想要在转发前修改要转发的值，可以用 `auto&&` 存储结果，修改后再转发：

```c++
template <typename T>
void f(T x) {
  auto&& res = doSomething(x);
  doSomethingElse(res);
  set(std::forward<decltype(res)>(res));
}
```

**关键：**

- 如果一个函数模板形参的类型为 `T&&`，并且 `T` 需要被推导得知，或者如果一个对象被声明为 `auto&&`，这个形参或者对象就是一个通用引用。
- 如果类型声明的形式不是标准的 `type&&`，或者如果类型推导没有发生，那么 `type&&` 代表一个右值引用。
- 通用引用，如果它被右值初始化，就会对应地成为右值引用；如果它被左值初始化，就会成为左值引用。

## Item 25: 对右值引用使用 `std::move`，对通用引用使用 `std::forward`

右值引用仅绑定可以移动的对象，使用 `std::move`，而通用引用使用右值初始化时，使用 `std::forward`

```c++
class A {
 public:
  A(A&& rhs) : s(std::move(rhs.s)), p(std::move(rhs.p)) {}

  template <typename T>
  void f(T&& x) {
    s = std::forward<T>(x);
  }

 private:
  std::string s;
  std::shared_ptr<int> p;
};
```

- 当把右值引用转发给其他函数时，右值引用应该被**无条件转换**为右值（通过 `std::move`），因为它们**总是**绑定到右值；

- 当转发通用引用时，通用引用应该**有条件地转换**为右值（通过 `std::forward`），因为它们只是**有时**绑定到右值。

如果希望只有在移动构造函数保证不抛出异常时，才转为右值，可以使用 `std::move_if_noexcept` 替代 `std::move`。

```c++
class A {
 public:
  A() = default;
  A(const A&) { std::cout << 1; }
  A(A&&) { std::cout << 2; }
};

class B {
 public:
  B() {}
  B(const B&) noexcept { std::cout << 3; }
  B(B&&) noexcept { std::cout << 4; }
};

int main() {
  A a;
  A a2 = std::move_if_noexcept(a);  // 1
  B b;
  B b2 = std::move_if_noexcept(b);  // 4
}
```

如果返回对象传入时是右值引用或转发引用，在返回时要用 `std::move` 或 `std::forward` 转换，返回类型不需要声明为引用，按值传递即可：

```c++
A f(A&& a) {
  doSomething(a);
  return std::move(a);
}

template <typename T>
A g(T&& x) {
  doSomething(x);
  return std::forward<T>(x);
}
```

返回局部变量时，不需要使用 `std::move` 来进行优化，局部变量会直接创建在为返回值分配的内存上，从而避免拷贝，这就是 C++ 标准诞生时就有的 RVO（return value optimization），RVO 的要求十分严谨，要求：

1. 局部对象与函数返回值的类型相同；
2. 返回的就是局部对象本身。

如果对从按值返回的函数返回来的局部对象使用 `std::move`，你并不能帮助编译器（如果不能实行拷贝消除的话，他们必须把局部对象看做右值），而是阻碍其执行优化选项（通过阻止 RVO）。在某些情况下，将 `std::move` 应用于局部变量可能是一件合理的事（即，你把一个变量传给函数，并且知道不会再用这个变量），但是满足 RVO 的 `return` 语句或者返回一个传值形参并不在此列。

使用 `std::move` 反而不满足 RVO 的要求，此外 RVO 只是一种优化，即使编译器不省略拷贝，返回对象也会被作为右值处理：

```c++
A makeA() { return A{}; }

auto x = makeA();  // 只需要调用一次 A 的默认构造函数
```

**关键：**

- 最后一次使用时，在右值引用上使用 `std::move`，在通用引用上使用 `std::forward`。
- 对按值返回的函数要返回的右值引用和通用引用，执行相同的操作。
- 如果局部对象可以被返回值优化消除，就绝不使用 `std::move` 或者 `std::forward`。

## Item 26: 避免重载通用引用

```c++
std::multiset<std::string> names;           //全局数据结构
void logAndAdd(const std::string& name)
{
    auto now =                              //获取当前时间
        std::chrono::system_clock::now();
    log(now, "logAndAdd");                 
    names.emplace(name);                    //把name加到全局数据结构中；
}

std::string petName("Darla");
logAndAdd(petName);                     //传递左值std::string
logAndAdd(std::string("Persephone"));	//传递右值std::string
logAndAdd("Patty Dog");                 //传递字符串字面值
```

第一个调用中，形参`name` 绑定到变量 `petName`，`name` 是左值，会拷贝到 `names` 中，没有办法避免拷贝。

第二个调用，形参`name` 绑定到右值，原则上可以被移动到 `names` 中，但本次调用时，仍然执行的是拷贝。

第三个调用中，仍有一个 `std::string` 的拷贝开销。

按照 Item 25 的说法，通过 `std::forward` 转发这个引用到 `emplace`：

```c++
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");           
logAndAdd(petName);                     //跟之前一样，拷贝左值到multiset
logAndAdd(std::string("Persephone"));	//移动右值而不是拷贝它
logAndAdd("Patty Dog");                 //在multiset直接创建std::string
                                        //而不是拷贝一个临时std::string
```

这样就效率就高了，但如果需要对 `logAndAdd` 进行重载：

```c++
std::string nameFromIdx(int idx);   //返回idx对应的名字

void logAndAdd(int idx)             //新的重载
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
```

之后调用都按照预期工作：

```c++
std::string petName("Darla");           //跟之前一样

logAndAdd(petName);                     //跟之前一样，
logAndAdd(std::string("Persephone")); 	//这些调用都去调用
logAndAdd("Patty Dog");                 //T&&重载版本

logAndAdd(22);                          //调用int重载版本
```

但如果传递给一个 `short` 类型：

```c++
short nameIdx;
...								//给nameIdx一个值
logAndAdd(nameIdx);				//错误
```

有两个重载的 `logAndAdd`。使用通用引用的那个推导出 `T` 的类型是 `short`，因此可以精确匹配。对于 `int` 类型参数的重载也可以在 `short` 类型提升后匹配成功。根据正常的重载解决规则，精确匹配优先于类型提升的匹配，所以被调用的是通用引用的重载。原因是对于 `short` 类型通用引用重载优先于 `int` 类型的重载。

通用引用几乎可以匹配任何类型，因此应避免对其进行重载，如果在构造函数中使用通用引用，会导致拷贝构造函数不能被正确匹配。

在适当的条件下，C++ 会生成拷贝和移动构造函数，即使类包含了模板化的构造函数，模板函数能实例化产生与拷贝和移动构造函数一样的签名，也在合适的条件范围内。如果拷贝和移动构造被生成：

```c++
class Person {
public:
    template<typename T>            //完美转发的构造函数
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}

    explicit Person(int idx);       //int的构造函数

    Person(const Person& rhs);      //拷贝构造函数（编译器生成）
    Person(Person&& rhs);           //移动构造函数（编译器生成）
    …
};

Person p("Nancy"); 
auto cloneOfP(p);                   //从p创建新Person；这通不过编译！
```

试图通过一个 `Person` 实例去创建另一个 `Person`，显然应该调用拷贝构造即可，但是这份代码不是调用拷贝构造函数，而是调用完美转发构造函数。然后，完美转发的函数将尝试使用 `Person` 对象 `p` 初始化 `Person` 的 `std::string` 数据成员，编译器就会报错。其中 `p` 被传递给拷贝构造函数或者完美转发构造函数。调用拷贝构造函数要求在 `p` 前加上 `const` 的约束来满足函数形参的类型，而调用完美转发构造不需要加这些东西。如果传递的对象改为 `const`，会得到不同的结果。

对于继承操作：
```c++
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) //拷贝构造函数，调用基类的
    : Person(rhs)                           //完美转发构造函数！
    { … }

    SpecialPerson(SpecialPerson&& rhs)      //移动构造函数，调用基类的
    : Person(std::move(rhs))                //完美转发构造函数！
    { … }
};
```

派生类的拷贝和移动构造函数没有调用基类的拷贝和移动构造函数，而是调用了基类的完美转发构造函数！为了理解原因，要知道派生类将 `SpecialPerson` 类型的实参传递给其基类，然后通过模板实例化和重载解析规则作用于基类 `Person`。最终，代码无法编译，因为 `std::string` 没有接受一个 `SpecialPerson` 的构造函数。

**关键：**

- 对通用引用形参的函数进行重载，通用引用函数的调用机会几乎总会比你期望的多得多。
- 完美转发构造函数是糟糕的实现，因为对于 non-`const `左值，它们比拷贝构造函数而更匹配，而且会劫持派生类对于基类的拷贝和移动构造函数的调用。

## Item 27: 重载通用引用的替代方案

- **不使用通用引用，改用 `const T&`**

- **传值**：通常在不增加复杂性的情况下提高性能的一种方法是，将按传引用形参替换为按值传递。

```c++
class Person {
public:
    explicit Person(std::string n)  //代替T&&构造函数，
    : name(std::move(n)) {}         //std::move的使用见条款41
  
    explicit Person(int idx)        //同之前一样
    : name(nameFromIdx(idx)) {}
    …

private:
    std::string name;
};
```

- 标签分派（tag dispatching)：额外引入一个参数来打破转发引用的万能匹配，*tag dispatch*的重要之处在于它可以允许我们组合重载和通用引用使用，没有 Item 26 中提到的问题。

```c++
std::multiset<std::string> names;           	//全局数据结构
template<typename T>                            //非整型实参：添加到全局数据结构中
void logAndAddImpl(T&& name, std::false_type)	//高亮std::false_type
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(
        std::forward<T>(name),
        std::is_integral<typename std::remove_reference<T>::type>()
    );
}

std::string nameFromIdx(int idx);        //与条款26一样，整型实参：查找名字并用它调用logAndAdd
void logAndAddImpl(int idx, std::true_type) //高亮std::true_type
{
  logAndAdd(nameFromIdx(idx)); 
}
```

- 使用 `std::enable_if` 在特定条件下禁用模版，

```c++
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_same<Person, typename std::decay<T>::type>::value>::type
    >
    explicit Person(T&& n);
    …
};


Person p("Nancy");
auto cloneOfP(p);       //用左值初始化
```

## Item 28: 理解引用折叠

引用折叠是 `std::forward` 工作的关键机制，经常能看到这样的使用：

```c++
template<typename T>
void f(T&& fParam)
{
    …                                   //做些工作
    someFunc(std::forward<T>(fParam));  //转发fParam到someFunc
}
```

这里，`fParam` 是通用引用，`std::forward` 的作用是当	且仅当传给 `f` 的实参为右值时，即 `T` 为非引用类型，才将 `fParam` （左值）转化为一个右值。

引用折叠发现在四种情况下：

- 模版实例化；
- auto 类型推断；
- decltype 类型推断；
- typedef 或 using 别名声明

存在两种类型的引用（左值和右值），所以有四种可能的引用组合（左值的左值，左值的右值，右值的右值，右值的左值）如果一个上下文中允许引用的引用存在（比如，模板的实例化），引用根据规则**折叠**为单个引用：

```
& + & → &
& + && → &
&& + & → &
&& + && → &&
```

- 引用折叠发生在四种情况下：模板实例化，`auto `类型推导，`typedef` 与别名声明的创建和使用，`decltype`；
- 当编译器在引用折叠环境中生成了引用的引用时，结果就是单个引用。有左值引用折叠结果就是左值引用，否则就是右值引用；
- 通用引用就是在特定上下文的右值引用，上下文是通过类型推导区分左值还是右值，并且发生引用折叠的那些地方。

## Item 29: 移动操作的缺点

在下面几种情况下，C++ 11 的移动语义并无优势：

- **没有移动操作**：要移动的对象没有提供移动操作，所以移动的写法也会变成复制操作。
- **移动不会更快**：要移动的对象提供的移动操作并不比复制速度更快。
- **移动不可用**：进行移动的上下文要求移动操作不会抛出异常，但是该操作没有被声明为 `noexcept`。

移动不一定比拷贝的代价小的多，比如 `std::array` 实际是带 STL 接口的内置数组，与其他容器不同，其他容器把元素存在堆上，自身只持有一个指向堆内存的指针，移动内存时只需要移动指针，在常数时间就可以完成移动，而 `std::array` 自身存储了内容，没有这样的指针，移动或拷贝对元素逐个执行，需要线性的复杂度，移动并不比拷贝快。

## Item 30: 完美转发失败的情况

完美转发的含义：“转发”仅表示一个函数的形参传递给另一个函数，对于被传递的那个函数目标是能收到与执行传递的函数完全相同的对象。这规则排除了按值传递的形参，因为它们是原始调用者传入内容的**拷贝**。我们希望被转发的函数能够使用最开始传进来的那些对象。指针形参也被排除在外，因为我们不想强迫调用者传入指针。关于通常目的的转发，我们将处理引用形参。

完美转发意味着不仅需转发对象，**还需要转发显著特征**：类型时左值还是右值，是 `const` 还是 `volatile`。

无法完美转发的情况：

- 大括号初始化

```c++
void f(const std::vector<int>& v) {}

template <typename T>
void fwd(T&& x) {
  f(std::forward<T>(x));
}

f({1, 2, 3});    // OK，{1, 2, 3} 隐式转换为 std::vector<int>
fwd({1, 2, 3});  // 无法推断 T，导致编译错误
```

当通过调用函数模版 `fwd` 间接调用 `f` 时，编译器不把传入给 `fwd` 的实参和 `f`  的声明中行参类型进行比较，而是推导传入 `fwd` 的实参类型，然后比较推导后的实参类型和 `f` 的形参声明类型。

解决：借用 `auto` 推断出 `std::initializer_list` 类型再转发：
```c++
auto x = {1, 2, 3};
fwd(x);  // OK
```

- 0 或者 NULL 作为空指针传递给模版时，会推断为 `int` 而非指针类型

```c++
void f(int*) {}

template <typename T>
void fwd(T&& x) {
  f(std::forward<T>(x));
}

fwd(NULL);  // T 推断为 int，转发失败
```

- 只声明不定义的整型 `static const` 数据成员

类内的 `static` 成员的声明不是定义，如果 `static` 声明为 `const`，则编译器会为这些成员执行 const propagation，从而不需要为它们保留内存，对整型 `static const` 成员取址可以通过编译，但会导致链接期的错误。转发引用也是引用，在编译器生成的机器代码中，引用一般会被当成指针处理。

```c++
class A {
 public:
  static const int n = 1;  // 仅声明
};

void f(int) {}

template <typename T>
void fwd(T&& x) {
  f(std::forward<T>(x));
}

f(A::n);    // OK：等价于 f(1)
fwd(A::n);  // 错误：fwd 形参是转发引用，需要取址，无法链接
```

- 重载函数名称和模版名称，要转发的函数名对应多个重载函数，则无法转发，因为模板无法从单独的函数名推断出函数类型，转发函数模版名称也会出问题，因为函数模版可以看成一批重载函数。

```c++
void g(int) {}
void g(int, int) {}

void f(void (*)(int)) {}

template <typename T>
void fwd(T&& x) {
  f(std::forward<T>(x));
}

f(g);    // OK
fwd(g);  // 错误：不知道转发的是哪一个函数指针
```

​	要让转发函数接受重载函数名称或模板名称，只能手动指定需要转发的重载版本或模板实例。

```c++
template <typename T>
void g(T x) {
  std::cout << x;
}

void f(void (*pf)(int)) { pf(1); }

template <typename T>
void fwd(T&& x) {
  f(std::forward<T>(x));
}

using PF = void (*)(int);
PF p = g;
fwd(p);                   // OK
fwd(static_cast<PF>(g));  // OK
```

- 位域：转发引用也是引用，实际上需要取址，但位域不允许直接取址，实际上接受位域实参的函数也只能收到位域值的拷贝，因此不需要使用完美转发，换用传值或传 `const` 引用即可。完美转发中也可以通过强制转换解决此问题，虽然转换的结果是一个临时对象的拷贝而非原有对象，但位域本来就无法做到真正的完美转发。

```c++
struct A {
  int a : 1;
  int b : 1;
};

void f(int) {}

template <typename T>
void fwd(T&& x) {
  f(std::forward<T>(x));
}

A x{};
f(x.a);    // OK
fwd(x.a);  // 错误

fwd(static_cast<int>(x.a));  // OK
```

# Lambda 表达式

lambda 通常被用来创建闭包，该闭包仅用作函数的实参。

```c++
{
    int x;                                  //x是局部对象
    …

    auto c1 =                               //c1是lambda产生的闭包的副本
        [x](int y) { return x * y > 55; };

    auto c2 = c1;                           //c2是c1的拷贝

    auto c3 = c2;                           //c3是c2的拷贝
    …
}
```

`c1`，`c2`，`c3`都是*lambda*产生的闭包的副本。

## Item 31: 避免使用默认捕获模式

C++11 中有两种默认的捕获模式：按引用捕获和按值捕获。但默认按引用捕获模式可能会带来悬空引用的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）。

按引用捕获会导致闭包中包含了对某个局部变量或者形参的引用，变量或形参只在定义 lambda 的作用域中可用。如果该 lambda 创建的闭包生命周期超过了局部变量或者形参的生命周期，那么闭包中的引用将会变成悬空引用。
