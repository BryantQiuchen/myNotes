# 右值引用

## 概念

- 左值(lvalue)：顾名思义就是赋值符号左边的值。 左值是表达式（不一定是赋值表达式）后依然存在的持久对象。

- 右值(rvalue)：右边的值，是指表达式结束后就不再存在的临时对象。

    而 C++11 中为了引入强大的右值引用，将右值的概念进行了进一步的划分，分为：纯右值、将亡值。

    - 纯右值(prvalue)：纯粹的右值，要么是纯粹的字面量，例如 `10`, `true`；要么是求值结果相当于字面量或匿名临时对象，例如 `1+2`。非引用返回的临时变量、运算表达式产生的临时变量、原始字面量、Lambda 表达式都属于纯右值。（字面量除了字符串字面量以外，均为纯右值）
    - 将亡值(xvalue)：是 C++11 为了引入右值引用而提出的概念（传统 C++ 中，纯右值和右值是同一个概念），也就是即将被销毁、却能够被移动的值。

```cpp
std::vector<int> foo() {
    std::vector<int> temp = {1, 2, 3, 4};
    return temp;
}
std::vector<int> v = foo();
// 函数 foo 的返回值 temp 在内部创建然后被赋值给 v， 然而 v 获得这个对象时，会将整个 temp 拷贝一份，然后把 temp 销毁，如果这个 temp 非常大， 这将造成大量额外的开销。
// 在最后一行中，v 是左值、 foo() 返回的值就是右值（也是纯右值）。但是，v 可以被别的变量捕获到，而 foo() 产生的那个返回值作为一个临时值，一旦被 v 复制后，将立即被销毁，无法获取、也不能修改。 
```

在 C++11 之后，编译器为我们做了一些工作，此处的左值 `temp` 会被进行此隐式右值转换，等价于 `static_cast<std::vector<int> &&>(temp)`，进而此处的 `v` 会将 `foo` 局部返回的值进行移动。

## 右值引用和左值引用

要拿到一个将亡值，就需要用到右值引用：`T &&`。右值引用的声明让这个临时值的生命周期得以延长、只要变量还活着，那么将亡值将继续存活。C++11 提供了 `std::move` 这个方法将左值参数无条件的转换为右值，有了它我们就能够方便的获得一个右值临时对象。

```cpp
#include <iostream>
#include <string>
void reference(std::string& str) {
    std::cout << "左值" << std::endl;
}
void reference(std::string&& str) {
    std::cout << "右值" << std::endl;
}

int main() {
    std::string lv1 = "string,"; // lv1 是一个左值
    // std::string&& r1 = lv1; // 非法, 右值引用不能引用左值
    std::string&& rv1 = std::move(lv1); // 合法, std::move可以将左值转移为右值
    std::cout << rv1 << std::endl; // string,

    const std::string& lv2 = lv1 + lv1; // 合法, 常量左值引用能够延长临时变量的生命周期
    // lv2 += "Test"; // 非法, 常量引用无法被修改
    std::cout << lv2 << std::endl; // string,string,

    std::string&& rv2 = lv1 + lv2; // 合法, 右值引用延长临时对象生命周期
    rv2 += "Test"; // 合法, 非常量引用能够修改临时变量
    std::cout << rv2 << std::endl; // string,string,string,Test

    reference(rv2); // 输出左值
	// rv2 虽然引用了一个右值，但由于它是一个引用，所以 rv2 依然是一个左值。
    return 0;
}
```

## 移动语义

传统 C++ 通过拷贝构造函数和赋值操作符为类对象设计了拷贝/复制的概念，但为了实现对资源的移动操作，调用者必须使用先复制、再析构的方式，否则就需要自己实现移动对象的接口。传统的 C++ 没有区分『移动』和『拷贝』的概念，造成了大量的数据拷贝，浪费时间和空间。右值引用的出现恰好就解决了这两个概念的混淆问题。

```cpp
#include <iostream> // std::cout
#include <utility> // std::move
#include <vector> // std::vector
#include <string> // std::string

int main() {
    std::string str = "Hello world.";
    std::vector<std::string> v;

    // 将使用 push_back(const T&), 即产生拷贝行为
    v.push_back(str);
    // 将输出 "str: Hello world."
    std::cout << "str: " << str << std::endl;

    // 将使用 push_back(const T&&), 不会出现拷贝行为
    // 而整个字符串会被移动到 vector 中，所以有时候 std::move 会用来减少拷贝出现的开销
    // 这步操作后, str 中的值会变为空
    v.push_back(std::move(str));
    // 将输出 "str: "
    std::cout << "str: " << str << std::endl;

    return 0;
}
```

## 完美转发

一个声明的右值引用其实是一个左值。参数传递中会有一些问题：

```cpp
void reference(int& v) {
    std::cout << "左值" << std::endl;
}
void reference(int&& v) {
    std::cout << "右值" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << "普通传参:";
    reference(v); // 始终调用 reference(int&)
}
int main() {
    std::cout << "传递右值:" << std::endl;
    pass(1); // 1是右值, 但输出是左值

    std::cout << "传递左值:" << std::endl;
    int l = 1;
    pass(l); // l 是左值, 输出左值

    return 0;
}
```

对于 `pass(1)` 来说，虽然传递的是右值，但由于 `v` 是一个引用，所以同时也是左值。 因此 `reference(v)` 会调用 `reference(int&)`，输出『左值』。对于 `pass(l)`，`l` 是一个左值，但 C++ 由于右值引用的出现而放宽了这一做法，从而产生了引用坍缩规则，允许我们对引用进行引用，既能左引用，又能右引用。但是却遵循如下规则：

| 函数形参类型 | 实参参数类型 | 推导后函数形参类型 |
| :----------- | :----------- | :----------------- |
| T&           | 左引用       | T&                 |
| T&           | 右引用       | T&                 |
| T&&          | 左引用       | T&                 |
| T&&          | 右引用       | T&&                |

模板函数中使用 `T&&` 不一定能进行右值引用，当传入左值时，此函数的引用将被推导为左值。更准确的讲，**无论模板参数是什么类型的引用，当且仅当实参类型为右引用时，模板参数才能被推导为右引用类型**。这才使得 `v` 作为左值的成功传递。

完美转发就是基于上述规律产生的。所谓完美转发，就是为了让我们在传递参数的时候，保持原来的参数类型（左引用保持左引用，右引用保持右引用）。为了解决这个问题，我们应该使用 `std::forward` 来进行参数的转发（传递）：

```cpp
#include <iostream>
#include <utility>
void reference(int& v) {
    std::cout << "左值引用" << std::endl;
}
void reference(int&& v) {
    std::cout << "右值引用" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << "              普通传参: ";
    reference(v);
    std::cout << "       std::move 传参: ";
    reference(std::move(v));
    std::cout << "    std::forward 传参: ";
    reference(std::forward<T>(v));
    std::cout << "static_cast<T&&> 传参: ";
    reference(static_cast<T&&>(v));
}
int main() {
    std::cout << "传递右值:" << std::endl;
    pass(1);

    std::cout << "传递左值:" << std::endl;
    int v = 1;
    pass(v);

    return 0;
}
```

结果如下：

```
传递右值:
              普通传参: 左值引用
       std::move 传参: 右值引用
    std::forward 传参: 右值引用
static_cast<T&&> 传参: 右值引用
传递左值:
              普通传参: 左值引用
       std::move 传参: 右值引用
    std::forward 传参: 左值引用
static_cast<T&&> 传参: 左值引用
```

无论传递参数为左值还是右值，普通传参都会将参数作为左值进行转发，所以 `std::move` 总会接受到一个左值，从而转发调用了`reference(int&&)` 输出右值引用。但 `std::forward` 即没有造成任何多余的拷贝，同时**完美转发**(传递)了函数的实参给了内部调用的其他函数。

`std::forward` 和 `std::move` 一样，没有做任何事情，`std::move` 单纯的将左值转化为右值，`std::forward` 也只是单纯的将参数做了一个类型的转换，从现象上来看， `std::forward<T>(v)` 和 `static_cast<T&&>(v)` 是完全一样的。

# 智能指针

C++指针包括两种，原始指针（raw ptr）和智能指针。

智能指针是原始指针的封装，优点是自动分配内存，不用担心潜在的内存泄露风险。

## 独占指针：`unique_ptr`

当需要一个智能指针时，`std::unique_ptr`通常是最合适的。可以合理假设，默认情况下，`std::unique_ptr`大小等同于原始指针，而且对于大多数操作（包括取消引用），他们执行的指令完全相同。移动一个`std::unique_ptr`将所有权从源指针转移到目的指针。（源指针被设为null。）拷贝一个`std::unique_ptr`是不允许的，因为如果你能拷贝一个`std::unique_ptr`，你会得到指向相同内容的两个`std::unique_ptr`，每个都认为自己拥有（并且应当最后销毁）资源，销毁时就会出现重复销毁。因此，`std::unique_ptr`是一种只可移动类型（*move-only type*）。当析构时，一个non-null `std::unique_ptr`销毁它指向的资源。默认情况下，资源析构通过对`std::unique_ptr`里原始指针调用 `delete` 来实现。

指针超过作用域时，内存自动释放。该类型指针不能copy，只能move。

- 通过已有原始指针创建
- 通过 `new` 创建
- 通过 `std::make_unique` 创建（推荐）
- 我们可以利用 `std::move` 将其转移给其他的 `unique_ptr`

## 计数指针：`shared_ptr`

可以共享数据，`shared_ptr` 创建一个计数器，它能够记录多少个 `shared_ptr` 共同指向一个对象，与类对象所指的内存相关联，copy则计数加1，否则就减1，当引用计数变为零的时候就会将对象自动删除。`std::make_shared` 就能够用来消除显式的使用 `new`，所以`std::make_shared` 会分配创建传入参数中的对象， 并返回这个对象类型的`std::shared_ptr`指针。

`std::shared_ptr` 可以通过 `get()` 方法来获取原始指针，通过 `reset()` 来减少一个引用计数， 并通过 `use_count()` 来查看一个对象的引用计数。

```cpp
auto pointer = std::make_shared<int>(10);
auto pointer2 = pointer; // 引用计数+1
auto pointer3 = pointer; // 引用计数+1
int *p = pointer.get();  // 这样不会增加引用计数
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 3
std::cout << "pointer2.use_count() = " << pointer2.use_count() << std::endl; // 3
std::cout << "pointer3.use_count() = " << pointer3.use_count() << std::endl; // 3

pointer2.reset();
std::cout << "reset pointer2:" << std::endl;
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 2
std::cout << "pointer2.use_count() = "
          << pointer2.use_count() << std::endl;           // pointer2 已 reset; 0
std::cout << "pointer3.use_count() = " << pointer3.use_count() << std::endl; // 2
pointer3.reset();
std::cout << "reset pointer3:" << std::endl;
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 1
std::cout << "pointer2.use_count() = " << pointer2.use_count() << std::endl; // 0
std::cout << "pointer3.use_count() = "
          << pointer3.use_count() << std::endl;           // pointer3 已 reset; 0
```

- Pass by value
    - copy
    - 函数计数+1
- Pass by reference
- Return by value 链式调用

不能将 `shared_ptr` 转换为 `unique_ptr`，可以通过 `std::move` 将 `unique_ptr` 转换为 `shared_ptr`。

## `weak_ptr`

`weak_ptr `并不拥有所有权，不能调用`->`和解引用，用于解决循环依赖的问题。

可以调用 `use_count()`，但计数器不会加一。

```cpp
struct A;
struct B;

struct A {
    std::shared_ptr<B> pointer;
    ~A() {
        std::cout << "A 被销毁" << std::endl;
    }
};
struct B {
    std::shared_ptr<A> pointer;
    ~B() {
        std::cout << "B 被销毁" << std::endl;
    }
};
int main() {
    auto a = std::make_shared<A>();
    auto b = std::make_shared<B>();
    a->pointer = b;
    b->pointer = a;
}
```

运行结果是 A, B 都不会被销毁，这是因为 a,b 内部的 pointer 同时又引用了 `a,b`，这使得 `a,b` 的引用计数均变为了 2，而离开作用域时，`a,b` 智能指针被析构，却只能造成这块区域的引用计数减一，这样就导致了 `a,b` 对象指向的内存区域引用计数不为零，而外部已经没有办法找到这块区域了，也就造成了内存泄露。解决这个问题的办法就是使用弱引用指针 `std::weak_ptr`，`std::weak_ptr`是一种弱引用（相比较而言 `std::shared_ptr` 就是一种强引用）。（弱引用不会引起引用计数增加）

# 并行和并发

## 线程

`std::thread` 用于创建一个执行的线程实例，所以它是一切并发编程的基础，使用时需要包含 `<thread>` 头文件，它提供了很多基本的线程操作，例如 `get_id()` 来获取所创建线程的线程 ID，使用 `join()` 来加入一个线程。

```cpp
#include <iostream>
#include <thread>

int main() {
    std::thread t([](){
        std::cout << "hello world." << std::endl;
    });
    t.join();
    return 0;
}
```

## 互斥量和临界区

C++11 引入了 `mutex` 相关的类，其所有相关的函数都放在 `<mutex>` 头文件中。`std::mutex` 是 C++11 中最基本的 `mutex` 类，通过实例化 `std::mutex` 可以创建互斥量， 而通过其成员函数 `lock()` 可以进行上锁，`unlock()` 可以进行解锁。但是在实际编写代码的过程中，最好不去直接调用成员函数，因为调用成员函数就需要在每个临界区的出口处调用 `unlock()`，当然，还包括异常。 这时候 C++11 还为互斥量提供了一个 RAII 语法的模板类 `std::lock_guard`。

在 RAII 用法下，对于临界区的互斥量的创建只需要在作用域的开始部分，例如：

```cpp
#include <iostream>
#include <mutex>
#include <thread>

int v = 1;

void critical_section(int change_v) {
    static std::mutex mtx;
    std::lock_guard<std::mutex> lock(mtx);
    // 执行竞争操作
    v = change_v;
    // 离开此作用域后 mtx 会被释放
}

int main() {
    std::thread t1(critical_section, 2), t2(critical_section, 3);
    t1.join();
    t2.join();

    std::cout << v << std::endl;
    return 0;
}
```

由于 C++ 保证了所有栈对象在生命周期结束时会被销毁，所以这样的代码也是异常安全的。无论 `critical_section()` 正常返回、还是在中途抛出异常，都会引发堆栈回退，也就自动调用了 `unlock()`。

而 `std::unique_lock` 则是相对于 `std::lock_guard` 出现的，`std::unique_lock` 更加灵活，`std::unique_lock` 的对象会以独占所有权（没有其他的 `unique_lock` 对象同时拥有某个 `mutex` 对象的所有权）的方式管理 `mutex` 对象上的上锁和解锁的操作。所以在并发编程中，推荐使用 `std::unique_lock`。

例如：

```cpp
#include <iostream>
#include <mutex>
#include <thread>

int v = 1;

void critical_section(int change_v) {
    static std::mutex mtx;
    std::unique_lock<std::mutex> lock(mtx);
    // 执行竞争操作
    v = change_v;
    std::cout << v << std::endl;
    // 将锁进行释放
    lock.unlock();

    // 在此期间，任何人都可以抢夺 v 的持有权

    // 开始另一组竞争操作，再次加锁
    lock.lock();
    v += 1;
    std::cout << v << std::endl;
}

int main() {
    std::thread t1(critical_section, 2), t2(critical_section, 3);
    t1.join();
    t2.join();
    return 0;
}
```

## 条件变量

条件变量 `std::condition_variable` 是为了解决死锁而生，当互斥操作不够用而引入的。线程可能需要等待某个条件为真才能继续执行，而一个忙等待循环中可能会导致所有其他线程都无法进入临界区使得条件为真时，就会发生死锁。`std::condition_variable`的 `notify_one()` 用于唤醒一个线程；`notify_all()` 则是通知所有线程。下面是一个生产者和消费者模型的例子：

```cpp
#include <queue>
#include <chrono>
#include <mutex>
#include <thread>
#include <iostream>
#include <condition_variable>


int main() {
    std::queue<int> produced_nums;
    std::mutex mtx;
    std::condition_variable cv;
    bool notified = false;  // 通知信号

    // 生产者
    auto producer = [&]() {
        for (int i = 0; ; i++) {
            std::this_thread::sleep_for(std::chrono::milliseconds(900));
            std::unique_lock<std::mutex> lock(mtx);
            std::cout << "producing " << i << std::endl;
            produced_nums.push(i);
            notified = true;
            cv.notify_all(); // 此处也可以使用 notify_one
        }
    };
    // 消费者
    auto consumer = [&]() {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            while (!notified) {  // 避免虚假唤醒
                cv.wait(lock);
            }
            // 短暂取消锁，使得生产者有机会在消费者消费空前继续生产
            lock.unlock();
            // 消费者慢于生产者
            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
            lock.lock();
            while (!produced_nums.empty()) {
                std::cout << "consuming " << produced_nums.front() << std::endl;
                produced_nums.pop();
            }
            notified = false;
        }
    };

    // 分别在不同的线程中运行
    std::thread p(producer);
    std::thread cs[2];
    for (int i = 0; i < 2; ++i) {
        cs[i] = std::thread(consumer);
    }
    p.join();
    for (int i = 0; i < 2; ++i) {
        cs[i].join();
    }
    return 0;
}
```

在多消费者的情况下，我们的消费者实现中简单放弃了锁的持有，这使得可能让其他消费者争夺此锁，从而更好的利用多个消费者之间的并发。

## 原子操作与内存模型

### 原子操作

`std::mutex` 可以解决上面出现的并发读写的问题，但互斥锁是操作系统级的功能，这是因为一个互斥锁的实现通常包含两条基本原理：

1. 提供线程间自动的状态转换，即『锁住』这个状态
2. 保障在互斥锁操作期间，所操作变量的内存与临界区外进行隔离

这是一组非常强的同步条件，换句话说当最终编译为 CPU 指令时会表现为非常多的指令（我们之后再来看如何实现一个简单的互斥锁）。这对于一个仅需原子级操作（没有中间态）的变量，似乎太苛刻了。

在 C++11 中多线程下共享变量的读写这一问题上，还引入了 `std::atomic` 模板，使得我们实例化一个原子类型，将一个原子类型读写操作从一组指令，最小化到单个 CPU 指令。

```cpp
std::atomic<int> counter;
```

并非所有的类型都能提供原子操作，这是因为原子操作的可行性取决于具体的 CPU 架构，以及所实例化的类型结构是否能够满足该 CPU 架构对内存对齐条件的要求，我们可以通过 `std::atomic<T>::is_lock_free` 来检查该原子类型是否支持原子操作。

### 一致性模型

并行执行的多个线程，从宏观层面考虑，可以粗略的认为是一种分布式系统，在分布式系统中，任何通信或本地操作都需要耗费一定的时间，甚至出现不可靠通信。

从原理上看，每个线程可以对应为一个集群节点，而线程间的通信也几乎等价于集群节点间的通信。削弱进程间的同步条件，通常我们会考虑四种不同的一致性模型：

1. **线性一致性**（强一致性或原子一致性）

    它要求任何一次读操作都能读到某个数据的最近一次写的数据，并且所有线程的操作顺序与全局时钟下的顺序是一致的。

    ```
            x.store(1)      x.load()
    T1 ---------+----------------+------>
    
    
    T2 -------------------+------------->
                    x.store(2)
    ```

    在这种情况下线程 `T1`, `T2` 对 `x` 的两次写操作是原子的，且 `x.store(1)` 是严格的发生在 `x.store(2)` 之前，`x.store(2)` 严格的发生在 `x.load()` 之前。 值得一提的是，线性一致性对全局时钟的要求是难以实现的，这也是人们不断研究比这个一致性更弱条件下其他一致性的算法的原因。

2. **顺序一致性**

    同样要求任何一次读操作都能读到数据最近一次写入的数据，但未要求与全局时钟的顺序一致。

    ```
            x.store(1)  x.store(3)   x.load()
    T1 ---------+-----------+----------+----->
    
    
    T2 ---------------+---------------------->
                  x.store(2)
    
    或者
    
            x.store(1)  x.store(3)   x.load()
    T1 ---------+-----------+----------+----->
    
    
    T2 ------+------------------------------->
          x.store(2)
    ```

    在顺序一致性的要求下，`x.load()` 必须读到最近一次写入的数据，因此 `x.store(2)` 与 `x.store(1)` 并无任何先后保障，即只要 `T2` 的 `x.store(2)` 发生在 `x.store(3)` 之前即可。

3. **因果一致性**

    要求进一步降低，只需要有因果关系的操作顺序得到保障，非因果关系的操作顺序不做要求。

    ```
          a = 1      b = 2
    T1 ----+-----------+---------------------------->
    
    
    T2 ------+--------------------+--------+-------->
          x.store(3)         c = a + b    y.load()
    
    或者
    
          a = 1      b = 2
    T1 ----+-----------+---------------------------->
    
    
    T2 ------+--------------------+--------+-------->
          x.store(3)          y.load()   c = a + b
    
    亦或者
    
         b = 2       a = 1
    T1 ----+-----------+---------------------------->
    
    
    T2 ------+--------------------+--------+-------->
          y.load()            c = a + b  x.store(3)
    ```

    只有 `c` 对 `a` 和 `b` 产生依赖，而 `x` 和 `y` 并无因果关系，上面的三种例子都是属于因果一致的。

4. **最终一致性**

    是最弱的一致性要求，它只保障某个操作在未来的某个时间节点上会被观察到，但并未要求被观察到的时间。因此我们甚至可以对此条件稍作加强，例如规定某个操作被观察到的时间总是有界的。

    ```
        x.store(3)  x.store(4)
    T1 ----+-----------+-------------------------------------------->
    
    
    T2 ---------+------------+--------------------+--------+-------->
             x.read      x.read()           x.read()   x.read()
    ```

    可能会出现不限于以下的情况：

    ```
    3 4 4 4 // x 的写操作被很快观察到
    0 3 3 4 // x 的写操作被观察到的时间存在一定延迟
    0 0 0 4 // 最后一次读操作读到了 x 的最终值，但此前的变化并未观察到
    0 0 0 0 // 在当前时间段内 x 的写操作均未被观察到，
            // 但未来某个时间点上一定能观察到 x 为 4 的情况
    ```

### 内存顺序

C++11 为原子操作定义了六种不同的内存顺序 `std::memory_order` 的选项，表达了四种多线程间的同步模型：

1. **宽松模型（Relaxed ording)**

    此模型下，单个线程的原子操作都是顺序执行的，不允许指令重排，但不同线程间原子操作的顺序是任意的。类型通过 `std::memory_order_relaxed` 指定。

    如果某个操作只要求是原子操作，除此之外，不需要其它同步的保障，就可以使用 Relaxed ordering。

    ```cpp
    std::atomic<int> counter = {0};
    std::vector<std::thread> vt;
    for (int i = 0; i < 100; ++i) {
        vt.emplace_back([&](){
            counter.fetch_add(1, std::memory_order_relaxed);
        });
    }
    
    for (auto& t : vt) {
        t.join();
    }
    std::cout << "current counter:" << counter << std::endl;
    ```

2. **释放/消费模型(Release-Consume ordering)**

    在此模型中，我们开始限制进程间的操作顺序，如果某个线程需要修改某个值，但另一个线程会对该值的某次操作产生依赖，即后者依赖前者。具体而言，线程 A 完成了三次对 `x` 的写操作，线程 `B` 仅依赖其中第三次 `x` 的写操作，与 `x` 的前两次写行为无关，则当 `A` 主动 `x.release()` 时候（即使用 `std::memory_order_release`），选项 `std::memory_order_consume` 能够确保 `B` 在调用 `x.load()` 时候观察到 `A` 中第三次对 `x` 的写操作。

    ```cpp
    // 初始化为 nullptr 防止 consumer 线程从野指针进行读取
    std::atomic<int*> ptr(nullptr);
    int v;
    std::thread producer([&]() {
        int* p = new int(42);
        v = 1024;
        ptr.store(p, std::memory_order_release);
    });
    std::thread consumer([&]() {
        int* p;
        while(!(p = ptr.load(std::memory_order_consume)));
    
        std::cout << "p: " << *p << std::endl;
        std::cout << "v: " << v << std::endl;
    });
    producer.join();
    consumer.join();
    ```

3. **释放/获取模型(Release-Acquire ordering)**

    在此模型下，我们可以进一步加紧对不同线程间原子操作的顺序的限制，在释放 `std::memory_order_release` 和获取 `std::memory_order_acquire` 之间规定时序，即发生在释放（release）操作之前的**所有**写操作，对其他线程的任何获取（acquire）操作都是可见的，亦即发生顺序（happens-before）。

    可以看到，`std::memory_order_release` 确保了它之前的写操作不会发生在释放操作之后，是一个向后的屏障（backward），而 `std::memory_order_acquire` 确保了它之前的写行为不会发生在该获取操作之后，是一个向前的屏障（forward）。对于选项 `std::memory_order_acq_rel` 而言，则结合了这两者的特点，唯一确定了一个内存屏障，使得当前线程对内存的读写不会被重排并越过此操作的前后：

    ```cpp
    std::vector<int> v;
    std::atomic<int> flag = {0};
    std::thread release([&]() {
        v.push_back(42);
        flag.store(1, std::memory_order_release);
    });
    std::thread acqrel([&]() {
        int expected = 1; // must before compare_exchange_strong
        while(!flag.compare_exchange_strong(expected, 2, std::memory_order_acq_rel))
            expected = 1; // must after compare_exchange_strong
        // flag has changed to 2
    });
    std::thread acquire([&]() {
        while(flag.load(std::memory_order_acquire) < 2);
    
        std::cout << v.at(0) << std::endl; // must be 42
    });
    release.join();
    acqrel.join();
    acquire.join();
    ```

4. **顺序一致性模型(Sequentially-consistent ordering)**

    在此模型下，原子操作满足顺序一致性，进而可能对性能产生损耗。可显式的通过 `std::memory_order_seq_cst` 进行指定。

    ```cpp
    std::atomic<int> counter = {0};
    std::vector<std::thread> vt;
    for (int i = 0; i < 100; ++i) {
        vt.emplace_back([&](){
            counter.fetch_add(1, std::memory_order_seq_cst);
        });
    }
    
    for (auto& t : vt) {
        t.join();
    }
    std::cout << "current counter:" << counter << std::endl;
    ```

