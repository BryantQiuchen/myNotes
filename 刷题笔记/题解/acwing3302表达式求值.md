# 题目

给定一个表达式，其中运算符仅包含 `+,-,*,/`（加 减 乘 整除），可能包含括号，请你求出表达式的最终值。

**注意：**

- 数据保证给定的表达式合法。
- 题目保证符号 `-` 只作为减号出现，不会作为负号出现，例如，`-1+2`,`(2+2)*(-(1+1)+2)` 之类表达式均不会出现。
- 题目保证表达式中所有数字均为正整数。
- 题目保证表达式在中间计算过程以及结果中，均不超过 $2^{32}-1$。
- 题目中的整除是指向 $0$ 取整，也就是说对于大于 $0$ 的结果向下取整，例如 $5/3=1$，对于小于 $0$ 的结果向上取整，例如 $5/(1-4) = -1$。
- C++和Java中的整除默认是向零取整；Python中的整除`//`默认向下取整，因此Python的`eval()`函数中的整除也是向下取整，在本题中不能直接使用。

**输入格式**

共一行，为给定表达式。

**输出格式**

共一行，为表达式的结果。

**数据范围**

表达式的长度不超过 $10^{5}$。

**输入样例**：

```
(2+2)*(1+1)
```

**输出样例**：

```
8
```

# 思路

“表达式求值”问题，两个核心关键点：

1. 双栈，一个操作数栈(num)，一个运算符栈(op)；

2. 运算符优先级，栈顶运算符和即将入栈的运算符的优先级比较：

    如果栈顶运算符的优先级高，需要先计算，再进行入栈；

    如果栈顶运算符的优先级低，则直接入栈。

对于 `(` 和 `)`，在碰到左括号时，直接入栈(op)，碰到右括号时，就需要判断运算符栈的栈顶是否是 `(` 不是的话就需要计算直到栈顶碰到 `(` 为止。

在循环结束后，如果运算符栈还是不空的，则还要进行计算，最后得到的结果存在操作数栈(num)的栈顶

关键是计算（注意 b 和 a）的函数 `eval()`：

```cpp
void eval(stack<char> &op, stack<int> &num) {
    auto b = num.top(); num.pop();
    auto a = num.top(); num.pop();
    auto c = op.top(); op.pop();
    int x;
    if (c == '+') x = a + b;
    else if (c == '-') x = a - b;
    else if (c == '*') x = a * b;
    else x = a / b;
    num.push(x);
}
```

# 代码

```cpp
#include <iostream>
#include <stack>
#include <string>
#include <unordered_map>
#include <algorithm>

using namespace std;

stack<char> op;
stack<int> num;

void eval(stack<char> &op, stack<int> &num) {
    auto b = num.top(); num.pop(); // 第二个操作数
    auto a = num.top(); num.pop(); // 第一个操作数
    auto c = op.top(); op.pop(); // 运算符
    int x;
    // 计算结果
    if (c == '+') x = a + b;
    else if (c == '-') x = a - b;
    else if (c == '*') x = a * b;
    else x = a / b;
    
    num.push(x); //最终结果入栈
}

int main() {
    string s;
    cin >> s;
    int n = s.size();

    // 运算优先级
    unordered_map<char, int> pr{{'+', 1}, {'-', 1}, {'*', 2}, {'/', 2}};
    for (int i = 0; i < n; ++i) {
        auto c = s[i];
        if (isdigit(c)) {
            // 计算数字的过程，数字可能不止有一位
            int x = 0, j = i;
            while (j < n && isdigit(s[j])) {
                x = x * 10 + s[j++] - '0';
            }
            i = j - 1; // 注意
            num.push(x);
        } else if (c == '(') {
            // 左括号无优先级直接入栈
            op.push(c);
        } else if (c == ')') {
            // 右括号，需要一直计算到左括号
            while (op.top() != '(') eval(op, num);
            op.pop();
        } else {
            // 遍历到操作符，需要判断优先级
            while (!op.empty() && op.top() != '(' 
                   && pr[op.top()] >= pr[c]) eval(op, num);
            op.push(c);
        }
    }
    while (!op.empty()) {
        eval(op, num);
    }
    cout << num.top() << endl;
    return 0;
}
```

