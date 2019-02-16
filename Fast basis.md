# Fast basis

### 原则

- 不加速只会使用个别次数的操作；
- 要权衡代码效率和开发效率&可读性；
- 对于新手，尽量使用第三方库和标准库的东西；
- 一切以测试结果为准。

### 测试

#### 可参考的基本框架

> 这里给出一个纯标准库的基本的框架。

```c++
#include <iostream>
#include <chrono>
#include <iomanip>

#define TEST_SIZE 1000000
#define PRECISION 5

using namespace std;

int main()
{
    int i = 0;
    auto beg = chrono::steady_clock::now();
    
    for(int i=0; i < TEST_SIZE; ++i)
    {
        // YOUR CODE
    }
    
    auto end = chrono::steady_clock::now();
    auto diff = (end-beg);

    cout << "[PRE::ADD]: " << endl << "for " << TEST_SIZE << " tests:" << endl;
    cout << ">>> time cost: " << fixed << setprecision(PRECISION)
         << chrono::duration<double, milli>(diff).count() << "ms" << endl << endl;
}
```

#### 随机数

```c++
#include <iostream>
#include <array>
#include <chrono>
#include <iomanip>
#include <random>

constexpr uint32_t TEST_SIZE = 100000;
constexpr uint32_t PRECISION = 6;

using namespace std;

int main()
{
    default_random_engine e(0);
    uniform_int_distribution<int> u{0, 100000}; // 0~100000

    array<int, TEST_SIZE> data;
    for (int i = 0; i < TEST_SIZE; ++i)
        data[i] = u(e);
}
```

```c++
// 如果需要从dev/random中拿“真随机数”
random_device rd;
default_random_engine e{rd()};
uniform_int_distribution<int> u{0, 100000};
```

> 具体可参考我N年前写的[一篇文章](https://ganler.github.io/2018/07/18/c-%E7%9A%84%E9%9A%8F%E6%9C%BA%E6%95%B0%EF%BC%9A%E4%BD%A0%E8%BF%98%E5%9C%A8%E7%94%A8srand%E5%90%97/)。

### 优化选项

> 一般的编译器都会默认进行`O2`优化。一般`O2`已经做的够好且能保证高度安全（就是不容易崩），`O3`采用积极内联，产生的二进制程序往往更大。也有人在stackoverflow上喷O3的程序会崩而且有的时候还比O2慢。
>
> 如O3，`g++ -O3 main.cpp`
>
> > 细节可以参考[here](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)。

### 常量表达式constexpr

> 有助于编译时提前计算一些固定的结果。

```c++
// https://heavywatal.github.io/cxx/speed.html
// 掛け算も割り算も毎回計算
return 4.0 * M_PI * std::pow(radius, 3) / 3.0;

// コンパイル時定数を掛けるだけ
constexpr double c = 4.0 * M_PI / 3.0; // 编译时计算
return c * std::pow(radius, 3);
```

### 多使用STL提供的工具

> 写过一些测试，总体发现STL在release模式下的性能非常恐怖。自己手动实现的一些tricks很难超越。所以，尽量使用STL。当然，以测试结果为准。

```c++
// https://heavywatal.github.io/cxx/speed.html
const double threshold = 3.14;
std::transform(
        v.begin(), v.end(), result.begin(), 
        [threshold](double x) -> bool{ return x > threshold; }
        );
```

### 传值和传址

> 结论：
>
> - 有按址操作的需求 -> 传址；
> - 无按址操作的需求：
>   - 大型数据（至少得大于8字节（64位系统）/4字节（32位系统））-> 传常引用；
> - 其他：传值（因为对小型数据传指针还得有解引用的开销，此外指针是4/8字节的）

```c++
void print(const std::tuple<int, double, float>& t_)
{
    auto [a, b, c] = t_;
    // 当然，这个地方快一点的方法是直接输出get<index>(t_);
    // auto [a, b, c] = t_的方法本质是初始化。
    std::cout << '(' << a << ',' << b << ',' << c << ')';
}
```

### 其他

> - Intel的cpu可以考虑使用Intel家的编译器icc。（还没用过，准备在寒假的机器人上试一试）
> - 有的包（比如OpenCV）有支持Intel的CPU的cmake加速选项。
> - 关注一些高性能的包（比如矩阵运算中的eigen3）。

### 参考

> [heavy watal](https://heavywatal.github.io/cxx/speed.html)