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

## Item24:区分通用引用和右值引用

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

## Item25:对右值引用使用 `std::move`，对通用引用使用 `std::forward`