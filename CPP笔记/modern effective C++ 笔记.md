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
