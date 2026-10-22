# C++ 基础学习笔记

## 知识点总览

| 板块 | 主要知识点 | 典型问题 |
| --- | --- | --- |
| 程序基础 | 头文件、命名空间、`main` 函数 | 头文件后误加分号 |
| 数据与流程 | 变量、输入输出、条件判断 | `=` 和 `==` 混淆 |
| 循环 | `for` 循环、嵌套循环、数量规律 | 图形无法居中 |
| 函数 | 值传递、引用传递、地址传递 | 修改没有影响原变量 |
| 复合数据 | 数组、结构体、结构体数组 | `stu` 与 `stuArray` 混用 |
| 面向对象 | 类、封装、`set`/`get` 函数 | 私有成员无法直接访问 |
| 字符串算法 | 字符转换、计数数组 | 重复数字被重复统计 |
| 调试 | 编译错误、运行错误、边界测试 | 程序一闪而过或没有输出 |

## 1. 一个完整程序的结构

> **本节知识点：** 头文件、命名空间、程序入口、语句结束符、大小写规则。

```cpp
#include <iostream>
#include <string>

using namespace std;

int main()
{
    cout << "Hello, C++!" << endl;
    return 0;
}
```

- `#include <iostream>` 提供 `cin`、`cout` 和 `endl`。
- `#include <string>` 提供 `string` 类型。
- 头文件后面不能加分号，所以不能写成 `#include <string>;`。
- `using namespace std;` 可以省去重复的 `std::`，适合目前的小型练习。
- C++ 区分大小写，`Student` 和 `student` 是两个不同的名字。

在较大的项目中通常会直接写 `std::cout`、`std::string`，避免不同库中的同名内容发生冲突。

## 2. 变量、输入和输出

> **本节知识点：** 基本数据类型、变量初始化、`cin`、`cout`、`getline` 和输入缓冲区。

```cpp
#include <iostream>
#include <string>

using namespace std;

int main()
{
    string name;
    int age = 0;
    double score = 0.0;

    cout << "请输入姓名：";
    cin >> name;

    cout << "请输入年龄和成绩：";
    cin >> age >> score;

    cout << "姓名：" << name << endl;
    cout << "年龄：" << age << endl;
    cout << "成绩：" << score << endl;

    return 0;
}
```

常见类型：

| 类型 | 用途 | 示例 |
| --- | --- | --- |
| `int` | 整数 | `18` |
| `double` | 小数 | `92.5` |
| `char` | 单个字符 | `'A'` |
| `bool` | 真或假 | `true` |
| `string` | 字符串 | `"张三"` |

如果姓名或句子中可能有空格，应使用 `getline(cin, text)`。当它前面刚使用过 `cin >>` 时，要先用 `cin.ignore()` 清理输入缓冲区中的换行符。

## 3. 条件判断

> **本节知识点：** `if`、`else if`、`else`、关系运算符和逻辑运算符。

```cpp
if (score >= 90)
{
    cout << "优秀" << endl;
}
else if (score >= 60)
{
    cout << "及格" << endl;
}
else
{
    cout << "不及格" << endl;
}
```

容易混淆的符号：

- `=` 表示赋值，例如 `score = 90;`。
- `==` 表示判断是否相等，例如 `score == 90`。
- `!=` 表示不等于。
- `&&` 表示并且，`||` 表示或者，`!` 表示取反。

## 4. 循环和图形输出

> **本节知识点：** `for` 循环、嵌套循环、循环变量、前导空格和奇数数量规律。

`for` 循环适合已知循环次数的情况：

```cpp
for (int i = 0; i < 5; ++i)
{
    cout << i << endl;
}
```

这里 `i` 从 `0` 开始，是因为数组下标也从 `0` 开始。若只是控制“第几行”，从 `1` 开始通常更直观。

### 输出居中的数字三角形

```cpp
#include <iostream>

using namespace std;

int main()
{
    int n = 0;
    cin >> n;

    for (int i = 1; i <= n; ++i)
    {
        for (int j = 1; j <= n - i; ++j)
        {
            cout << ' ';
        }

        for (int j = 1; j <= 2 * i - 1; ++j)
        {
            cout << i;
        }

        cout << endl;
    }

    return 0;
}
```

第 `i` 行分为两部分：

1. 先输出 `n - i` 个前导空格；
2. 再连续输出 `2 * i - 1` 个数字。

数字之间没有空格，所以图形才会形成连续且轴对称的三角形。若写成 `cout << i << ' ';`，每个数字占两个字符，前导空格也必须相应调整。

### 圣诞树树冠的规律

如果树冠一共有 `n` 行，第 `i` 行通常输出：

- `n - i` 个空格；
- `2 * i - 1` 个星号。

```cpp
for (int i = 1; i <= n; ++i)
{
    for (int j = 1; j <= n - i; ++j)
    {
        cout << ' ';
    }

    for (int j = 1; j <= 2 * i - 1; ++j)
    {
        cout << '*';
    }

    cout << endl;
}
```

内层循环变量 `j` 每一行都重新从 `1` 开始计数；外层变量 `i` 决定当前行数。变量从 `0` 还是从 `1` 开始并没有固定答案，关键是循环次数和公式必须对应。

## 5. 函数

> **本节知识点：** 函数定义与调用、返回值、形参和实参、值传递、引用传递、地址传递。

函数把一段功能独立出来，便于重复调用和检查。

```cpp
int add(int a, int b)
{
    return a + b;
}
```

### 值传递

值传递会复制一份参数，函数内的修改不会影响原变量。

```cpp
void addOne(int number)
{
    ++number;
}
```

### 引用传递

引用是原变量的别名，函数内的修改会直接影响原变量。

```cpp
void addOne(int& number)
{
    ++number;
}
```

### 地址传递

地址传递使用指针，调用时传入变量地址。

```cpp
void addOne(int* number)
{
    if (number != nullptr)
    {
        ++(*number);
    }
}

int main()
{
    int value = 10;
    addOne(&value);
    return 0;
}
```

只读取而不修改的大对象可以使用常量引用，既避免复制，也防止误修改：

```cpp
void printName(const string& name)
{
    cout << name << endl;
}
```

## 6. 数组和结构体

> **本节知识点：** 数组下标、结构体定义、成员访问、结构体数组和结构体函数参数。

数组用于保存多个相同类型的数据，结构体用于把一个对象的不同属性组合起来。

```cpp
struct Student
{
    string name;
    int age;
    int score;
};
```

结构体名称必须前后一致。定义的是 `Student`，创建变量时也要写 `Student s1;`，不能误写成 `student s1;`。

### 结构体数组

```cpp
#include <iostream>
#include <string>

using namespace std;

struct Student
{
    string name;
    int age;
    int score;
};

int main()
{
    Student stuArray[3] = {
        {"张三", 18, 90},
        {"李四", 19, 80},
        {"王五", 20, 70}
    };

    stuArray[2].name = "赵六";
    stuArray[2].age = 21;
    stuArray[2].score = 60;

    for (int i = 0; i < 3; ++i)
    {
        cout << "姓名：" << stuArray[i].name
             << "，年龄：" << stuArray[i].age
             << "，成绩：" << stuArray[i].score << endl;
    }

    return 0;
}
```

以前出错的原因是数组定义为 `stu`，后面却访问 `stuArray`。变量名只要有一个字符不同，编译器就会把它当成另一个变量。

### 结构体作为函数参数

```cpp
void printStudent(const Student& student)
{
    cout << "姓名：" << student.name << endl;
    cout << "年龄：" << student.age << endl;
    cout << "分数：" << student.score << endl;
}
```

这里使用 `const Student&`，可以避免复制整个结构体，而且保证打印函数不会修改学生信息。

## 7. 类、private 和 set/get 函数

> **本节知识点：** 类与对象、访问权限、封装、数据校验、常成员函数和类型转换。

类可以把数据和操作数据的函数放在一起。成员变量设为 `private` 后，类外不能随意修改，只能通过公开函数访问。

```cpp
class Student
{
private:
    string name;
    int score = 0;

public:
    void setName(const string& newName)
    {
        name = newName;
    }

    void setScore(int newScore)
    {
        if (newScore >= 0 && newScore <= 100)
        {
            score = newScore;
        }
    }

    string getName() const
    {
        return name;
    }

    int getScore() const
    {
        return score;
    }
};
```

`set` 函数不仅用于赋值，还可以检查数据是否合法。`get` 函数用于读取私有成员。成员函数后面的 `const` 表示该函数不会修改对象。

### 计算平均分

```cpp
double calculateAverageScore(const Student students[], int length)
{
    if (length <= 0)
    {
        return 0.0;
    }

    int totalScore = 0;

    for (int i = 0; i < length; ++i)
    {
        totalScore += students[i].getScore();
    }

    return static_cast<double>(totalScore) / length;
}
```

`static_cast<double>` 把总分转换成小数，避免两个整数相除时丢失小数部分。

如果题目没有强制要求动态内存，直接使用对象数组通常比 `new` 和 `delete` 更简单，也不容易忘记释放内存。

## 8. 字符串与 Bulls and Cows

> **本节知识点：** `string` 下标访问、字符数字转换、`vector` 计数、重复元素处理。

“猜数字”题需要分别统计位置和数字都正确的 Bulls，以及数字正确但位置错误的 Cows。处理重复数字时，不能简单地判断某个数字是否出现过，而要统计每个数字剩余的次数。

```cpp
#include <algorithm>
#include <string>
#include <vector>

using namespace std;

string getHint(const string& secret, const string& guess)
{
    int bulls = 0;
    vector<int> secretCount(10, 0);
    vector<int> guessCount(10, 0);

    for (int i = 0; i < static_cast<int>(secret.length()); ++i)
    {
        if (secret[i] == guess[i])
        {
            ++bulls;
        }
        else
        {
            ++secretCount[secret[i] - '0'];
            ++guessCount[guess[i] - '0'];
        }
    }

    int cows = 0;

    for (int i = 0; i < 10; ++i)
    {
        cows += min(secretCount[i], guessCount[i]);
    }

    return to_string(bulls) + "A" + to_string(cows) + "B";
}
```

`secret[i] - '0'` 的作用是把字符数字转换成整数下标，例如 `'7' - '0'` 等于 `7`。已经计入 Bulls 的位置不能再次计入 Cows，所以只有位置不相同时才进入计数数组。

## 9. 常见错误和排查顺序

> **本节知识点：** 编译错误、运行错误、控制台输出、图形对齐和编译器差异。

### 编译错误

优先检查：

1. 头文件后是否误加分号；
2. 大括号、圆括号和分号是否缺失；
3. 类名、结构体名和变量名的大小写是否一致；
4. 使用变量前是否已经定义；
5. `main` 函数外是否误放了可执行语句。

### 能编译但没有输出

检查程序是否在等待 `cin` 输入，输出语句是否真的执行，以及控制台窗口是否运行后立即关闭。在 Visual Studio 中应使用“开始执行（不调试）”，也可以在程序末尾临时加入：

```cpp
cin.get();
```

不建议把 `system("pause")` 当成跨平台方案，因为它依赖 Windows 命令。

### 图形没有居中

先把每一行拆成“前导空格”和“实际字符”两部分，分别列出数量规律。还要确认实际字符之间是否有空格，因为一个额外空格会改变整行宽度。

### 编译器差异

Visual Studio 2022 和旧版 Dev-C++ 可能使用不同的编译器和语言标准。若 Visual Studio 能运行而 Dev-C++ 不能，应检查 Dev-C++ 的编译器路径、项目类型和 C++ 标准设置。旧版编译器也可能不完整支持较新的语法。

## 10. 调试检查清单

> **本节知识点：** 手算验证、最小测试、错误定位、边界数据和调试输出。

写完代码后按下面的顺序检查：

1. 先手算一组简单输入的预期结果；
2. 编译并阅读第一条错误信息；
3. 从变量定义处核对名字和类型；
4. 在循环中打印 `i`、`j` 等关键变量；
5. 测试最小值、最大值和重复数据；
6. 确认无误后再删除临时调试输出。
