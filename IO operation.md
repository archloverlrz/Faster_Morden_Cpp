# I/O operation

### printf is faster than cout

实际上，在c++中输出是一个并不慢的环节。`std::cout`其实是一个很慢的标准输出流操作，比C的`printf`慢一截，为什么这么说呢？很简单，C++为了兼容C，`std::cout`也要“兼容”其他的输出操作，比如`std::cerr`，`printf`等。这里的“兼容”其实就是与stdio的缓冲区做同步的意思，即对于不同的输出操作，`std::cout`要使之同步，即必须要让当前输出操作执行完，才能进行下一步的输出，这样的话，肯定是需要对输出操作进行检查，这就导致程序运行效率低下。同步C的输出是线程安全的，不同线程的输出可能会交错，但是不会出现资源竞争。

```c++
// base.cpp
#include <chrono>
#include <iostream>

using namespace std;
using namespace std::chrono;

class Timer
{
public:
    Timer() : clk_beg(high_resolution_clock::now()) {}
    void reset(){clk_beg = high_resolution_clock::now(); }

    template <typename Format = milliseconds>
    uint64_t time_out() const
    {
        return duration_cast<Format>(high_resolution_clock::now() - clk_beg).count();
    }

    uint64_t time_out_microseconds() const{ return time_out<microseconds>(); }
    uint64_t time_out_seconds() const{ return time_out<seconds>(); }
    uint64_t time_out_minutes() const{ return time_out<minutes>(); }
    uint64_t time_out_hours() const{ return time_out<hours>(); }

private:
    time_point<high_resolution_clock> clk_beg;
};

int main()
{
    Timer t;

    for (int i = 0; i < 10000; ++i)
        cout << "nioasdnosindoiasalkndladnzndklfnniadmnflm" << endl;

    auto ans1 = t.time_out();

    t.reset();

    for (int i = 0; i < 10000; ++i)
        printf("nioasdnosindoiasalkndladnzndklfnniadmnflm\n");

    auto ans2 = t.time_out();

    cout << "For cout -> " << ans1 << " ms" << endl << "For printf -> " << ans2 << " ms";
}
/*
// Release模式
For cout -> 37 ms
For printf -> 28 ms

*/
```



### 关闭同步

> 如果你想坚持使用`std::cout`。

这个`std::cout`同步的功能在某些场景下很有帮助，比如我们开发了一个程序，里面有各种各样的输出操作:

```c++
#include<iostream>
int main()
{// 这个的结果肯定是按顺序，我就不贴截图了。
    std::cout << "first\n";
    printf("second\n");
}
```

`std::cout`的同步功能能使之严格按顺序输出，即`first`一定会在`second`之前输出。

而如果我们关掉这个同步功能的话，就有可能出现输出顺序混乱的情况：

```c++
#include<iostream>
int main(){
/*
在不同步的情况下，printf不会等cout输出完，且
printf本来就比cout输出快，故先输出second。
*/
    std::ios_base::sync_with_stdio(false);
    std::cout << "first\n";
    printf("second\n");
}

// 在base.cpp上加入上述代码后
/*
For cout -> 34 ms(快了3ms)
For printf -> 27 ms
*/
```

![image-20181226153459315](/Users/ganler-mac/Library/Application Support/typora-user-images/image-20181226153459315.png)

当然手动使用`std::flush`和`std::endl`刷新缓冲区就可以解决这个问题。

而在我们的程序中，我们并用不着换着不同的输出方式输出，因为我们是统一的`std::cout`，这样我们就可以使用`std::ios_base::sync_with_stdio(false)`来关掉同步的功能，从而换取40%（不一定准确）输出速度的提升。

> 建议就是，如果你想用cout，就关掉同步，并且不要用printf，以及没事少`endl`。

> 在`base.cpp`上把`endl`换成`\n`后，速度显著提升，甚至比printf还快：

```c++
/* 未关闭同步
For cout -> 23 ms
For printf -> 28 ms
*/

/* 关闭同步
For cout -> 22 ms
For printf -> 28 ms
*/
```



### `cin.tie(nullptr)`

在LeetCode上很多人都会用这段代码：

```c++
static const bool io_sync_off = []()
{
    std::ios::sync_with_stdio(false);
    std::cin.tie(nullptr);
    // or std::cin.tie(NULL) or std::cin.tie(0)
    return true;
}();
// 关于具体使用，只需要在代码中加入该段程序即可（位置即一般函数的位置）。
```

> `tie`是用来绑定两个stream的方法。`std::cin.tie(xxx)`就是将`xxx`&`cin`两个流给绑定起来。`std::cin.tie()`在无参传入的情况下返回当前绑定的流的指针（默认情况下，`std::cin`与`std::cout`是绑定的）。
>
> 被`std::cin`绑定的stream（流），在每次使用了`<<`之后都会强行`std::flush`一下，即刷新缓冲区。`std::cin.tie(nullptr)`操作即让`std::cin`随便和一个家伙绑定，这样`std::cout`就不受其限制，每次输出的时候都不用再`std::flush`。于是乎更快了。

#### 总结

- anyway，减少不必要的刷新buffer操作；
- 使用上面写的lambda函数（如果你的代码不需要和别的IO方式兼容）；
- cout/cin 不和 printf/scanf 混用（为了某种意义上的安全）；
- 减少没有意义的`std::flush/std::endl`。（一般直接`'\n'`即可）。

## Reference

> [geek](https://www.geeksforgeeks.org/fast-io-for-competitive-programming/)
>
> [heavy watal](https://heavywatal.github.io/cxx/speed.html)

