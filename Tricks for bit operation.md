# Tricks for bit operation

> 本篇虽然讲位运算的技巧，但我并不建议大家深刻&认真的学习这些技巧，而是把它当小说一样快速阅读。更重要的是，希望大家看清楚位运算这个东西到底有没有用，到底什么时候更有用。（btw，一切基于benchmark）

### Basic

#### and

> `x and 1` 等价于 `is_1(x)`

对一个二进制整数的末位bit使用此技巧，可以判断其奇偶性。(把这里的x想象成二进制码，把1想象成`11...11`)

#### or

> `x or 1` 等价于 `x = 1`
>
> `(x or 1) - 1` 等价于 `x = 0`

可以把一个数变成最接近的奇数/偶数。

#### xor

> 取反操作。可逆。
>
> `x xor 1` 等价于 `x = (x==0) ? (1):(0);`
>
> `x xor 0` 等价于 `do_nothing`

> 关于可逆运算和`swap`：
>
> 假设`#`和`@`互为可逆运算：
>
> > 即：`(x # y) @ y = x`，比如`+`和`-` 。

```c++
x = x # y; // (x # y)
y = x @ y; // (x # y) @ y = x
x = x @ y; // (x # y) @ x = y
```

```c++
// 整数swap
// void swap(); built in std.
void swap_(int& a, int& b)
{
    a += b;		// a_new = a+b
    b = a-b;	// b_new = a_new-b = a
    a -= b;		// a_new = a+b-b_new = b
}

void swap_bit(int& a, int& b)
{
    a ^= b;
    b ^= a;
    a ^= b;
}
```

> 与`std::swap`的benchmark：

```c++
/*
Compiler: g++-8
OS:       MacOS Mojave
CPU:      2.2 GHz Intel Core i7
*/
#include <iostream>

#include <array>

#include <chrono>
#include <iomanip>
#include <random>

constexpr uint32_t TEST_SIZE = 100000;
constexpr uint32_t PRECISION = 6;

using namespace std;

void swap_bit(int& a, int& b)
{
    a ^= b;
    b ^= a;
    a ^= b;
}

void swap_normal(int& a, int& b)
{
    int tmp = a;
    a = b;
    b = tmp;
}

int main()
{
    default_random_engine e(0);
    uniform_int_distribution<int> u{0, 100000};

    array<int, TEST_SIZE> data;
    for (int i = 0; i < TEST_SIZE; ++i)
        data[i] = u(e);

    auto data_ = data;
    auto _data_ = data;

    // std::swap
    auto beg = chrono::steady_clock::now();
    for (int j = 0; j < TEST_SIZE / 2; ++j)
        swap(data[j*2], data[j*2+1]);
    auto end = chrono::steady_clock::now();
    auto diff = (end-beg);

    cout << "[STD::SWAP]: " << endl << "for " << TEST_SIZE/2 << " tests:" << endl;
    cout << ">>> time cost: " << fixed << setprecision(PRECISION)
         << chrono::duration<double, milli>(diff).count() << "ms" << endl << endl;

    // bit::swap
    beg = chrono::steady_clock::now();
    for (int j = 0; j < TEST_SIZE / 2; ++j)
        swap_bit(data_[j*2], data_[j*2+1]);
    end = chrono::steady_clock::now();
    diff = (end-beg);

    cout << "[BIT::SWAP]: " << endl << "for " << TEST_SIZE/2 << " tests:" << endl;
    cout << ">>> time cost: " << fixed << setprecision(PRECISION)
         << chrono::duration<double, milli>(diff).count() << "ms" << endl << endl;

    cout << data[TEST_SIZE-1] << data_[TEST_SIZE-1] << endl;

    // normal::swap
    beg = chrono::steady_clock::now();
    for (int j = 0; j < TEST_SIZE / 2; ++j)
        swap_normal(_data_[j*2], _data_[j*2+1]);
    end = chrono::steady_clock::now();
    diff = (end-beg);

    cout << "[NORMAL::SWAP]: " << endl << "for " << TEST_SIZE/2 << " tests:" << endl;
    cout << ">>> time cost: " << fixed << setprecision(PRECISION)
         << chrono::duration<double, milli>(diff).count() << "ms" << endl << endl;

    // cout << data[TEST_SIZE-1] << data_[TEST_SIZE-1] << _data_[TEST_SIZE-1] << endl;
    // release时打开上面一句。
}
/*
debug:
    [STD::SWAP]: 
    for 50000 tests:
    >>> time cost: 1.014000ms

    [BIT::SWAP]: 
    for 50000 tests:
    >>> time cost: 0.830000ms

    [NORMAL::SWAP]: 
    for 50000 tests:
    >>> time cost: 0.857000ms

release:
    STD::SWAP]: 
    for 50000 tests:
    >>> time cost: 0.024000ms

    [BIT::SWAP]: 
    for 50000 tests:
    >>> time cost: 0.049000ms

    [NORMAL::SWAP]: 
    for 50000 tests:
    >>> time cost: 0.028000ms
*/
```

#### shr & shl

即移位运算：

```c++
a >> 1; // a /= 2; 2的一次方
a >> 2; // a /= 4; 2的二次方
a >> 3; // a /= 8; 2的三次方
```

> matrix67大佬在[Matrix67](http://www.matrix67.com/blog/archives/263)提及这种位运算方法会比传统方法快60%。而我的实践表明（mac pro 2015, clang++ 7.0.1, debug mode），对于`1e9`次除法：

```c++
/*
除2
For /= 2 -> 1911203 μs
For >> 1 -> 1893611 μs
*/

/*
除4
For /= 2 -> 1880941 μs
For >> 1 -> 1861074 μs
*/

/*
除8
For /= 2 -> 1913451 μs
For >> 1 -> 1909108 μs
*/

/*
除16
For /= 2 -> 1864361 μs
For >> 1 -> 1880314 μs
*/
```

> 我们发现两者差异并不大（位运算稍占可以忽略的优势），而且，当除数变大的时候，位运算的“优势”似乎会逐渐转变为“劣势”。

#### 小总结

上述讲到的位运算技巧，<u>都其实对应着一个可读性很高的“一般技巧”</u>，由上述内容，我们可能会认为：

- 位运算可读性低
- 位运算并没有想象的那种高性能
- 写位运算的结果是费力不讨好

至此，依照我的经验，对于绝大多数位运算操作，他们仅仅只是很fancy或者good in debug, bad in release。所以对于绝大多数情况，写出可读性高的代码，往往会更加利于编译器的优化。对于一些特别简单的操作，请不要尝试位运算，很多时候可能都是<u>费力不讨好</u>。

### 我所了解过的有用的位运算

> 在上面的章节，我强调了一点，对于简单的，一般操作就能完成的算法，不要轻易尝试位运算。但有的时候，位运算可以简单解决一些“一般操作难以完成”的算法，对于某些情况，设置还有加速的效果。

#### 减色函数

> 设计一些图像处理和位运算的知识，这里不详述，有兴趣可以看看[这本书](https://www.amazon.com/OpenCV-Computer-Application-Programming-Cookbook/dp/1786469715)。（有中文版，但不建议阅读中文版）。

```c++
#include <opencv2/opencv.hpp>
#include <iostream>
#include <chrono>

void colorReduce_normal(const cv::Mat& img, cv::Mat& result, int div = 64) {
    result.create(img.rows, img.cols, img.type());
    int rows_num = img.rows;
    for (int i = 0; i < img.rows; i++) {
        const uchar* original = img.ptr<uchar>(i); // const 对象要用const指针
        uchar* row_ptr = result.ptr<uchar>(i);
        for (int j = 0; j < img.cols*img.channels(); j++) {
            row_ptr[j] = (original[j] / div)*div + div / 2;
        }
    }
}

void colorReduce_bit(cv::Mat img, int div = 64) {
    int rows_num = img.rows;
    int cols_num = img.cols*img.channels();
    int n = static_cast<int>(log(static_cast<double>(div)) / log(2.0));
	if (img.isContinuous()) {
		cols_num = cols_num * rows_num; // 把所有像素值集于一列
		rows_num = 1; // 一行
	}
    uchar mask = 0xFF << n;// 0xFF = 11...1(8个1); mask = 11...0(n个0)
    uchar div2 = div >> 1; // div2=div/2;
    for (int i = 0; i < rows_num; i++) {
        uchar* rows_ptr = img.ptr<uchar>(i);
        for (int j = 0; j < cols_num; j++) {
            // This makes it very fast.
            *rows_ptr &= mask;
            *rows_ptr++ += div2;
        }
    }
}

int main() {
    auto path = "../NIP/test_photo.png"; // use your path.
    cv::Mat img = cv::imread(path);
    cv::Mat res;// 声明而不初始化

    auto beg = std::chrono::steady_clock::now();
    colorReduce_normal(img, res);
    auto end = std::chrono::steady_clock::now();
    std::cout << "normal\t" << std::chrono::duration<double, std::milli>(end-beg).count() << "ms" << std::endl;

    beg = std::chrono::steady_clock::now();
    colorReduce_bit(img);
    end = std::chrono::steady_clock::now();
    std::cout << "bit\t" << std::chrono::duration<double, std::milli>(end-beg).count() << "ms" << std::endl;
}

// debug
// normal 161.818ms
// bit    50.5042ms

// release
// normal 64.9445ms
// bit    1.56774ms
```

#### 判断x是不是2的N次幂

```c++
constexpr bool normal_method(uint32_t x)
{// O(log2(n))
    uint32_t i = 1;
    for(; i < x; i *= 2);
    return i == x;
}

constexpr bool bit_method(uint32_t x)
{// O(1)
    return x & (x-1) == 0;
    // x & (x-1)返回的是x消除二进制末位最后一个1的结果，2^n只有一个1，消去后为0
}
// bit is always faster about 8 times.
```

#### 数组中只有一个数出现了奇数次，其他数出现了偶数次，求这个数

```c++
int normal_method(const vector<int>& vec)
{
    // 略
    // 用哈希的话是O(max(n, max(vec)))，但空间消耗是max(vec)，且常数大。
}

int bit_method(const vector<int>& vec)
{
    int ans = 0;
	for(auto x : vec) ans ^= x;
    return ans ^= 0;
    // 用到的是我上面讲的xor的思路，对于一个可逆操作#，a # b # b = a，^就是一个可逆的操作。
}
```

#### 总结

- 对于简单的程序请不要优先考虑位运算；
- 在需求可以得到满足的情况下，尽量使用标准库提供的方法；
- 对于时间复杂度高的算法，可以考虑用位运算降低其复杂度；

> 对于具体的位运算教学可以参考Matrix67的博客，这里不再赘述。

### Ref

> [Matrix67](http://www.matrix67.com/blog/archives/263)

