# 题目

在给定的 $N$ 个整数 $A_1,A_2,...,A_N$ 中选出两个进行 `xor`（异或）运算，得到的结果最大是多少？

**输入格式**

第一行输入一个整数 $N$。

第二行输入 $N$ 个整数 $A_1,A_2,...,A_N$。

**输出格式**

输出一个整数表示答案。

**数据范围**

$1\leq N\leq 10^5,$
$0 \leq A_i \leq 2^{31}.$

**输入样例**：

```
3
1 2 3
```

**输出样例**：

```
3
```

# 思路

暴力枚举$O(n)$

```cpp
for (int i = 0; i < n; ++i) {
    for (int j = 0; j < i; ++j) {
        // 但其实 a[i] ^ a[j] == a[j] ^ a[i]
        // 所以内层循环 j < i // 因为 a[i] ^ a[i] == 0 所以事先把返回值初始化成0 不用判断相等的情况
    }
}
```

优化：trie树

数据范围是$0 \leq A_i \leq 2^{31}$，因此可以映射为31位的二进制表示，将暴力做法的内层循环用 $O(1)$ 时间来实现:

```cpp
    for (int i = 0; i < n; ++i) {
        insert(a[i]);
        res = max(res, a[i] ^ query(a[i]));
    }
```

因为 `a[i] ^ query(a[i])` 相当于内存循环的最大 `res`

为何 `query(a[i])` 与 `a[i]` 异或运算结果最大 ?

因为 `query(a[i])` 尽可能找二进制高位上与 `a[i]` 相反的元素, 于是在该位上异或结果尽可能为 `1`

**核心操作**

`insert` 操作

```cpp
void insert(int x) {
    int p = 0;
    for (int i = 30; i >= 0; --i) {
        int u = x >> i & 1; // 第i位是0还是1
        if (!son[p][u]) son[p][u] = ++idx;
        p = son[p][u];
    }
}
```

`query` 操作

```cpp
int query(int x) {
    int p = 0, res = 0;
    for (int i = 30; i >= 0; --i) {
        int u = x >> i & 1;
        if (son[p][!u]) {
            p = son[p][!u];
            res = res * 2 + !u;
        } else {
            p = son[p][u];
            res = res * 2 + u;
        }
    }
    return res;
}
```

# 代码

```cpp
#include <iostream>
#include <algorithm>

using namespace std;

const int N = 100010, M = 31 * N;

int a[N];
int son[M][2], idx;
// 本题中只有0或1两种

void insert(int x) {
    int p = 0; // 根节点
    for (int i = 30; i >= 0; --i) {
        int u = x >> i & 1; // 第i位是0还是1
        if (!son[p][u]) son[p][u] = ++idx; // 如果插入中发现没有该子节点,开出这条路
        p = son[p][u];
    }
}

int query(int x) {
    int p = 0, res = 0;
    // 从最大位开始找
    for (int i = 30; i >= 0; --i) {
        int u = x >> i & 1;
        // *2相当左移一位，然后如果找到对应位上不同的数
        if (son[p][!u]) {
            p = son[p][!u];
            res = res * 2 + !u;
        } else {
            p = son[p][u];
            res = res * 2 + u;
        }
    }
    return res;
}

int main() {
    int n;
    cin >> n;
    for (int i = 0; i < n; ++i) cin >> a[i];

    int res = 0;
    for (int i = 0; i < n; ++i) {
        insert(a[i]);
        res = max(res, a[i] ^ query(a[i]));
    }
    cout << res << endl;
    return 0;
}
```

