# 基础算法

## 快速排序

主要思想：分治

```cpp
void quick_sort(int q[], int l, int r) {
    if (l >= r) return;
    int i = l - 1, j = r + 1, x = q[(l+r)>>1];
    while (i < j) {
        do i++; while (q[i] < x);
        do j--; while (q[j] > x);
        if (i < j) swap(q[i], q[j]);
    }
    quick_sort(q, l, j), quick_sort(q, j + 1, r);
}
```

## 归并排序

```cpp
void merge_sort(int l, int r) {
    if (l >= r) return;
    // 确定分界点
    int mid = (l + r) / 2;
    // 递归处理
    merge_sort(l, mid), merge_sort(mid + 1, r);
    // 归并的过程，合二为一的过程
    int k = 0, i = l, j = mid + 1;
    while (i <= mid && j <= r) {
        if (q[i] <= q[j]) temp[k++] = q[i++];
        else temp[k++] = q[j++];
    }
    while (i <= mid) temp[k++] = q[i++];
    while (j <= r) temp[k++] = q[j++];
    // 搬回q[]
    for (i = l, k = 0; i <= r; i++, k++) q[i] = temp[k];
}
```

## 二分查找

### 整数二分查找

模版1：区间`[l, r]`划分成`[l, mid]`和`[mid + 1, r]`

更新操作：`r = mid` 或 `l = mid + 1`，计算`mid`时为 `mid = l + r >> 1`

```cpp
int bsearch_1(int l, int r) {
    while (l < r) {
        int mid = l + r >> 1;
        if (check(mid)) r = mid;    // check()判断mid是否满足性质
        else l = mid + 1;
    }
    return l;
}
```

模版2：区间`[l, r]`划分成`[l, mid - 1]`和`[mid, r]`

更新操作：`l = mid` 或 `r = mid - 1`，计算`mid`时为 `mid = l + r + 1 >> 1`（区别模版1）

```cpp
int bsearch_2(int l, int r) {
    while (l < r) {
        int mid = l + r + 1 >> 1;
        if (check(mid)) l = mid;
        else r = mid - 1;
    }
    return l;
}
```

### 浮点数二分

不需要判断区间，选择的 mid 是 double 型

```cpp
double bsearch_3(double l, double r) {
    const double eps = 1e-6;   // eps 表示精度，取决于题目对精度的要求，假设要求保留4位，就设为1e-6
    while (r - l > eps) {
        double mid = (l + r) / 2;
        if (check(mid)) r = mid;
        else l = mid;
    }
    return l;
}
```

## 高精度运算

### 加法运算

```cpp
// C = A + B, A >= 0, B >= 0
// 加引用可以提高效率
vector<int> big_add(vector<int> &A, vector<int> &B) {
    vector<int> C;
    int t = 0;  // 进位标识符
    for (int i = 0; i < A.size() || i < B.size(); ++i){
        if (i < A.size()) t += A[i];
        if (i < B.size()) t += B[i];
        C.push_back(t % 10);
        t /= 10;
    }
    if (t) C.push_back(t);
    return C;
}
```

### 减法运算

```cpp
// C = A - B, 满足A >= B, A >= 0, B >= 0，对与A < B的情况，在输出时进行判断即可
// 判断A和B的大小关系
bool cmp(vector<int> &A, vector<int> &B) {
    if (A.size() != B.size()) return A.size() > B.size();
    for (int i = A.size() - 1; i >= 0; --i)
        if (A[i] != B[i]) 
            return A[i] > B[i];
    return true;
}

vector<int> big_sub(vector<int> &A, vector<int> &B) {
    vector<int> C;
    for (int i = 0, t = 0; i < A.size(); ++i){
        t = A[i] - t;
        if (i < B.size()) t -= B[i];
        C.push_back((t + 10) % 10);
        if (t < 0) t = 1;
        else t = 0;
    }
	// 判断相减后前面出现多个0的情况
    while (C.size() > 1 && C.back() == 0) C.pop_back();
    return C;
}
```

### 乘法运算

```cpp
// C = A * b, A >= 0, b >= 0
vector<int> big_mul(vector<int> &A, int b) {
    vector<int> C;
    int t = 0;
    for (int i = 0; i < A.size() || t; ++i) {
        if (i < A.size()) t += A[i] * b;
        C.push_back(t % 10);
        t /= 10;
    }
    while (C.size() > 1 && C.back() == 0) C.pop_back();
    return C;
}
```

### 除法运算

```cpp
// A / b = C ... r, A >= 0, b > 0
vector<int> big_div(vector<int> &A, int b, int &r) {
    vector<int> C;
    r = 0;
    for (int i = A.size() - 1; i >= 0; --i) {
        r = r * 10 + A[i];
        C.push_back(r / b);
        r %= b;
    }
    reverse(C.begin(), C.end());
    while (C.size() > 1 && C.back() == 0) C.pop_back();
    return C;
}
```

## 前缀和和差分

### 一维前缀和

```
S[i] = a[1] + a[2] + ... a[i]
a[l] + ... + a[r] = S[r] - S[l - 1]
```

### 二维前缀和

```
S[i, j] = 第i行j列格子左上部分所有元素的和
S[i, j] = S[i - 1, j] + S[i, j - 1] - S[i - 1, j - 1] + a[i, j]
以(x1, y1)为左上角，(x2, y2)为右下角的子矩阵的和为：
S[x2, y2] - S[x1 - 1, y2] - S[x2, y1 - 1] + S[x1 - 1, y1 - 1]
```

### 一维差分

可以看作是前缀和的逆运算，核心：`insert` 操作（`给区间[l,r]之间的每个数加上c`），不需要直接构造。

```cpp
void insert(int l, int r, int c) {
    b[l] += c;
    b[r+1] -= c;
}
```

### 二维差分

给以`(x1, y1)`为左上角，`(x2, y2)`为右下角的子矩阵中的所有元素加上`c`：

```cpp
void insert(int x1, int y1, int x2, int y2, int c) {
    b[x1][y1] += c;
    b[x2+1][y1] -= c;
    b[x1][y2+1] -= c;
    b[x2+1][y2+1] += c;
}
```

## 双指针法

常见问题分类：

1. 对于一个序列，用两个指针维护一段区间
2. 对于两个序列，维护某种次序，比如归并排序中合并两个有序序列的操作

```cpp
for (int i = 0, j = 0; i < n; ++i) {
    while (j < i && check(i, j)) ++j;
    // 具体问题的逻辑
}
```

## 离散化

[Acwing802.区间和](题解/acwing802区间和.md)

```c++
vector<int> alls; // 存储所有待离散化的值
sort(alls.begin(), alls.end()); // 将所有值排序
alls.erase(unique(alls.begin(), alls.end()), alls.end());   // 去掉重复元素

// 二分求出x对应的离散化的值, 找到第一个大于等于x的位置
int find(int x) {
    int l = 0, r = alls.size() - 1;
    while (l < r) {
        int mid = l + r >> 1;
        if (alls[mid] >= x) r = mid;
        else l = mid + 1;
    }
    return r + 1; // 映射到1, 2, ...n
}
```

## 区间合并

```cpp
// 将所有存在交集的区间合并
void merge(vector<PII> &segs) {
    vector<PII> res;

    sort(segs.begin(), segs.end());

    int st = -2e9, ed = -2e9;
    for (auto seg : segs) {
        if (ed < seg.first) {
            if (st != -2e9) res.push_back({st, ed});
            st = seg.first, ed = seg.second;
        } else {
            ed = max(ed, seg.second);
        }
    }
    if (st != -2e9) res.push_back({st, ed});

    segs = res;
}
```

# 数据结构

## 链表

### 单链表

用数组构建单链表

```cpp
/**
 * @param head : 存储链表头
 * @param e[]  : 存储节点的值
 * @param ne[] : 存储节点的next指针
 * @param idx  : 表示当前可用的节点是哪个
 */
int head, e[N], ne[N], idx;

// 初始化
void init() {
    head = -1;
    idx = 0;
}

// 在链表头插入一个数a
void addToHead(int a) {
    e[idx] = a, ne[idx] = head, head = idx++;
}

// 第k个插入的数后面插入一个数x
void addTok(int k, int x) {
    e[idx] = x, ne[idx] = ne[k], ne[k] = idx++;
}

// 将头结点删除，需要保证头结点存在
void remove() {
    head = ne[head];
}

// 将下标是k的点后面的点删掉
void remove(int k) {
    ne[k] = ne[ne[k]];
}

// 打印链表
for (int i = head; i != -1; i = ne[i]) 
    cout << e[i] << ' ';
```

### 双链表

```cpp
/**
 * @param e[] : 存储节点的值
 * @param l[] : 节点的左指针
 * @param r[] : 节点的右指针
 * @param idx : 表示当前可用的节点是哪个
 */
int e[N], l[N], r[N], idx;

// 初始化，导致第a个插入元素的下标为：a + 1
void init() {
    // 0是左端点，1是右端点，两个边界
    r[0] = 1, l[1] = 0;
    idx = 2;
}

// 在下标k的右边插入一个数x
void insert(int k, int x) {
    e[idx] = x;
    l[idx] = k, r[idx] = r[k];
    l[r[k]] = idx, r[k] = idx++;
}

// 删除节点k
void remove(int k) {
    l[r[k]] = l[k];
    r[l[k]] = r[k];
}
```

## 栈

[Acwing3302.表达式求值](题解/acwing3302表达式求值.md)

### 数组模拟栈

```cpp
// tt表示栈顶
int stk[N], tt = 0;

// 向栈顶插入一个数
stk[++tt] = x;

// 从栈顶弹出一个数
tt--;

// 栈顶的值
stk[tt];

// 判断栈是否为空
if (tt > 0) not empty
else empty
```

### 单调栈

常见模型：找出每个数左边离它最近的比它大/小的数是多少。

```cpp
int tt = 0;
for (int i = 1; i <= n; ++i) {
    while (tt && check(stk[tt], i)) tt--;
    stk[++tt] = i;
}
```

## 队列

### 数组模拟队列

```cpp
// 普通队列
// hh 表示队头，tt表示队尾
int q[N], hh = 0, tt = -1;

// 向队尾插入一个数
q[++tt] = x;

// 从队头弹出一个数
hh++;

// 队头的值
q[hh];

// 队尾的值
q[tt];

// 判断队列是否为空
if (hh <= tt) not empty
else empty
```

### 循环队列

```cpp
// hh 表示队头，tt表示队尾的后一个位置
int q[N], hh = 0, tt = 0;
// 向队尾插入一个数
q[tt++] = x;
if (tt == N) tt = 0;

// 从队头弹出一个数
hh ++ ;
if (hh == N) hh = 0;

// 队头的值
q[hh];

// 判断队列是否为空
if (hh != tt) not empty
else empty
```

### 单调队列

常见模型：找出滑动窗口中的最大值/最小值

```cpp
int hh = 0, tt = -1;
for (int i = 0; i < n; ++i) {
    while (hh <= tt && check_out(q[hh])) hh++;  // 判断队头是否滑出窗口
    while (hh <= tt && check(q[tt], i)) tt--;
    q[++tt] = i;
}
```

## KMP算法

常用于字符串匹配

```cpp
// s[]是文本串，p[]是模式串，n是s的长度，m是p的长度

// 构造next[]数组，用ne[]表示，求next数组的过程：
for (int i = 2, j = 0; i <= n; ++i) {
	while (j && p[i] != p[j+1]) j = ne[j];
	if (p[i] == p[j+1]) ++j;
	ne[i] = j;
}
// 匹配过程
for (int i = 1, j = 0; i <= m; ++i) {
	while (j && s[i] != p[j+1]) j = ne[j];
	if (s[i] == p[j+1]) j++;
	if (j == n) {
		// 匹配成功
		cout << i - n << ' ';
		j = ne[j];
    }
}
```

## Trie树

[Acwing143.最大异或对](题解/acwing143最大异或对.md)

高效地存储和查找字符串集合的数据结构

```cpp
int son[N][26], cnt[N], idx;
// 0号点既是根节点，又是空节点
// son[][]存储树中每个节点的子节点(假设只有小写字母)
// cnt[]存储以每个节点结尾的单词数量

// 插入一个字符串
void insert(char str[]){
    int p = 0;
    for (int i = 0; str[i]; ++i) {
        int u = str[i] - 'a';
        if (!son[p][u]) son[p][u] = ++idx;
        p = son[p][u];
    }
    cnt[p]++;
}

// 查询字符串出现的次数
int query(char str[]) {
    int p = 0;
    for (int i = 0; str[i]; ++i) {
        int u = str[i] - 'a';
        if (!son[p][u]) return 0;
        p = son[p][u];
    }
    return cnt[p];
}
```

## 并查集

### 朴素并查集

```cpp
int p[N]; // 存储每个点的祖宗节点

// 初始化，假定节点编号是1~n
for (int i = 1; i <= n; ++i) p[i] = i;

// 返回x的祖宗节点，做了路径压缩操作
int find(int x) {
	if (p[x] != x) p[x] = find(p[x]);
	return p[x];
}

// 合并a和b所在的两个集合：
p[find(a)] = find(b);
```

### 维护size的并查集

```cpp
// p[]存储每个点的祖宗节点
// cnt[]只有祖宗节点的有意义，表示祖宗节点所在集合中的点的数量
int p[N], cnt[N]; 

// 返回x的祖宗节点
int find(int x) {
	if (p[x] != x) p[x] = find(p[x]);
    return p[x];
}

// 初始化，假定节点编号是1~n
for (int i = 1; i <= n; ++i) {
	p[i] = i;
	cnt[i] = 1;
}

// 合并a和b所在的两个集合：
cnt[find(b)] += cnt[find(a)];
p[find(a)] = find(b);
```

### 维护到祖宗节点距离的并查集

```cpp
// p[]存储每个点的祖宗节点
// d[x]存储x到p[x]的距离
int p[N], d[N];

// 返回x的祖宗节点
int find(int x) {
	if (p[x] != x) {
		int u = find(p[x]);
        d[x] += d[p[x]];
        p[x] = u;
	}
        return p[x];
}

// 初始化，假定节点编号是1~n
for (int i = 1; i <= n; ++i) {
	p[i] = i;
	d[i] = 0;
}

// 合并a和b所在的两个集合：
p[find(a)] = find(b);
d[find(a)] = distance; // 根据具体问题，初始化find(a)的偏移量
```

## 堆

小根堆：

```cpp
// h[N]存储堆中的值, h[1]是堆顶，x的左儿子是2x, 右儿子是2x + 1
int h[N], cnt;

void down(int u) {
    int t = u;
    if (u * 2 <= size && h[u*2] < h[t]) t = u * 2;
    if (u * 2 + 1 <= size && h[u*2+1] < h[t]) t = u * 2 + 1;
    if (u != t) {
        heap_swap(u, t);
        down(t);
    }
}

void up(int u) {
    while (u / 2 && h[u] < h[u/2]) {
        heap_swap(u, u / 2);
        u >>= 1;
    }
}

// O(n)建堆
for (int i = n / 2; i; --i) down(i);
```

## 哈希

### 拉链法

```cpp
int h[N], e[N], ne[N], idx;

// 向哈希表中插入一个数
void insert(int x){
    int k = (x % N + N) % N;
    e[idx] = x;
    ne[idx] = h[k];
    h[k] = idx++;
}

// 在哈希表中查询某个数是否存在
bool find(int x) {
    int k = (x % N + N) % N;
    for (int i = h[k]; i != -1; i = ne[i])
        if (e[i] == x)
			return true;

	return false;
}
```

### 开放寻址法

```cpp
int h[N];

// 如果x在哈希表中，返回x的下标；如果x不在哈希表中，返回x应该插入的位置
int find(int x) {
	int t = (x % N + N) % N;
    while (h[t] != null && h[t] != x) {
        t++;
        if (t == N) t = 0;
    }
    return t;
}
```

