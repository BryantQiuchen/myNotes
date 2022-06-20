# vector

声明：

```cpp
#include <vector> 	// 头文件
vector<int> a;		// 相当于一个长度动态变化的int数组
vector<int> b[233];	// 相当于第一维长233，第二位长度动态变化的int数组
struct rec{…};
vector<rec> c;		// 自定义的结构体类型也可以保存在vector中
```

操作：

- `size()`返回`vector`的实际长度，`empty()`返回一个`bool`类型，表明`vector`是否为空，二者复杂度都是$O(1)$，所有 STL 容器都支持。
- `clear()`函数，把`vector`清空

迭代器：STL 容器的“指针”，可以用星号 "*" 操作符解除引用。

保存`int`的`vector`迭代器的声明方法：

```cpp
vector<int>::iterator it;
```

`vector`的迭代器是“随机访问迭代器”，可以把`vector`的迭代器与一个整数相加减，其行为和指针的移动类似。可以把`vector`的两个迭代器相减，其结果也和指针相减类似，得到两个迭代器对应下标之间的距离。

- `begin()`返回指向`vector`中的第一个元素的迭代器，例如：a是一个非空的`vector`，`*a.beign()`和`a[0]`作用相同
- 所有容器都可以视作一个“前闭后开”的结构，`end()`返回第n个元素再往后的边界。`*a.end()`和`a[n]`都是越界访问，下面代码都遍历了`vector<int> a`

```cpp
for (int i = 0; i < a.size(); i ++) 
    cout << a[i] << endl;

for (vector<int>::iterator it = a.begin(); it != a.end(); it ++) 
    cout << *it << endl;
```

- `front/back`
    - `front()`返回 `vector`的第一个元素，等价于`*a.beign()`和`a[0]`
    - `back()`返回`vector`的最后一个元素，等价于`a[a.size() - 1]`
- `push_back()和pop_back()`
    - `a.push_back(x)` 把元素x插入到`vector a`的尾部。
    -  `b.pop_back() `删除`vector a`的最后一个元素。

支持比较运算

```cpp
vector<int> a(4, 3), b(3, 4);
// a < b 字典序排序
```

# pair<T, T>

- `first`，第一个元素
- `second`，第二个元素
- 支持比较运算，以`first`为第一关键字，以`second`为第二关键字（字典序）

```cpp
// 初始化
pair<int, string> p1;
p2 = make_pair(10, "dd");
p3 = {20, "aa"}
```

# queue

头文件 queue 主要包括循环队列 queue 和优先队列 priority_queue 两个容器。

声明：

```cpp
queue<int> q;
struct rec{…}; queue<rec> q; 	//结构体rec中必须定义小于号
priority_queue<int> q;		// 大根堆
priority_queue<int, vector<int>, greater<int> q;	// 小根堆
priority_queue<pair<int, int>>q;
```

循环队列 queue

- `push(x)` 从队尾插入元素x
- `pop()`从队头弹出
- `front()`返回队头元素
- `back()`返回队尾元素

优先队列 priority_queue

- `push(x)`把元素x插入堆
- `pop()`删除队顶元素
- `top()`查询队顶元素（即最大值）

构建小根堆：

```cpp
priority_queue<int, vector<int>, greater<int>> heap;
```

# stack

头文件stack包含栈。声明和前面的容器类似。

`push(x)` 向栈顶插入x

`pop()`弹出栈顶元素

# deque

双端队列 duque 是一个支持在两端高效插入和删除元素的连续线性存储空间，是vector和 queue 的结合。

与 vector 相比，deque 在头部增删元素仅需要 $O(1)$ 的时间；

与 queue 相比，deque 像数组一样支持随机访问。

- `[]`随机访问
- `begin/end`，返回deque的头/尾迭代器
- `front/back` 队头/队尾元素
- `push_back` 从队尾入队
- `push_front` 从队头入队
- `pop_back` 从队尾出队
- `pop_front` 从队头出队
- `clear` 清空队列

# set

头文件set主要包括 set 和 multiset 两个容器，分别是“有序集合”和“有序多重集合”，即前者的元素不能重复，而后者可以包含若干个相等的元素。 set 和 multiset 的内部实现是一棵红黑树，它们支持的函数基本相同。

声明：
```cpp
#include <set> 	// 头文件
set<int> s;
struct rec{…}; set<rec> s;	// 结构体rec中必须定义小于号
multiset<double> s;
```

`size / empty / clear` 与 vector 类似

迭代器：

- set 和 multiset不支持 “随机访问”，支持 (*) 解除引用。
- `begin() / end()`返回首尾迭代器，时间复杂度 $O(1)$
    - `s.begin()`指向集合中最小元素的迭代器
    - `s.end()`指向集合中最大元素的下一位置的迭代器，`-- s.end()`指向集合中最大元素的迭代器。

操作：

- `s.insert(x)`把一个元素 x 插入到集合 s 中，时间复杂度为$O(logn)$。在 set 中，若元素已存在，则不会重复插入该元素，对集合的状态无影响。

- `s.find(x) `在集合s中查找等于x的元素，并返回指向该元素的迭代器。**若不存在，则返回`s.end()`**。时间复杂度为$O(logn)$。

- `lower_bound/upper_bound`与`find`类似，查找条件略不同，时间复杂度$O(logn)$

    - `s.lower_bound(x)` 查找大于等于 x 的元素中最小的一个，并返回指向该元素的迭代器。
    - `s.upper_bound(x) `查找大于 x 的元素中最小的一个，并返回指向该元素的迭代器。

- 设`it`是一个迭代器，`s.earse(it)`从 s 中删除迭代器 it 指向的元素，时间复杂度$O(logn)$

    设 x 是一个元素，`s.earse(x)`从 s 中删除所有等于 x 的元素，时间复杂度为$O(k + logn)$，其中 k 是被删除的元素个数

- `s.count(x)` 返回集合 s 中等于 x 的元素个数，时间复杂度为 $O(k +logn)$，其中 k 为元素 x 的个数。由于 set 容器中各元素的值是唯一的，因此该函数的返回值最大为 1。

# map

map容器是一个键值对key-value的映射，其内部实现是一棵以key为关键码的红黑树。Map的key和value可以是任意类型，其中key必须定义小于号运算符。

声明：

```cpp
#inculde <map>  	//头文件
map<key_type, value_type> name;
// 例如：
map<long, long, bool> vis;
map<string, int> hash;
map<pair<int, int>, vector<int>> test;
```

- `size/empty/clear/begin/end`均与 set类 似。
- `insert / earse`与 set 类似，但其参数均是 `pair<key_type, value_type>`
- `h.find(x) `在变量名为 h 的 map 中查找 key 为 x 的二元组。
- []操作符，`h[key]` 返回key映射的value的引用，时间复杂度为$O(logn)$。可以通过`h[key]`得到`key`对应的`value`，还可以对`h[key]`进行赋值操作，改变`key`对应的`value`
- `lower_bound/upper_bound`

# unordered

`unordered_set` `unordered_map` `unordered_multiset` `unordered_multimap` 哈希表

增删改查的时间复杂度$O(1)$，不支持`lower_bound/upper_bound`，迭代器的 `++  --`

# bitset（压位）

声明

```cpp
bitset(10000) s;
```

- `count()`返回有多少个1
- `any()`判断是否至少有一个1
- `none()` 判断是否全为0
- `set()` 把所有位置变成1
- `set(k, v)` 把第k位变成v
- `reset()` 把所有位变成0
- `flip()` 等价与~  `flip(k)`第k位

