# Multi-thread

注意区分多线程，并发和并行的区别，多线程并不完全是并行，反而偏向于并发。通过多线程技术，不仅可以完成一些“看起来是同时进行”的功能，更可以更加充分地利用多核CPU的计算资源，达到加速的效果。具体的线程相关的知识会在操作系统中涉及。我也看过一些比较有意思的理解：[通俗的理解 by zh](<https://www.zhihu.com/question/28550867/answer/450069610>)。

## 快速上车

### 基本使用

一般套路

- 创建线程
- 发配任务
- 阻塞/分离线程

```c++
#include <thread>
#include <iostream>

using namespace std;

void say_sth(const string& sth)
{
    cout << sth << endl;
}

int main()
{
    thread new_thread(say_sth, "Hello thread"); // 函数名+该函数需要的参数
    new_thread.join(); // 等待new_thread执行完，再和主线程汇总。
//    new_thread.detach(); // 与主线程分离，不再等待new_thread，而是继续做后面的事情。
}
```

这里要注意的一点是，<u>线程也是**对象**，脱离了作用域，就会被析构</u>，对于被`detach`的线程，如果在起离开作用域之前没能把事情干完，就会发生一些意想不到的错误。对于这样的问题，我们可以定义全局线程对象。

除此之外，线程可以被move，不可以被copy。（每个线程都由单独对应的id，不可被复制）

```c++
auto mvd_thr(std::move(new_thread));
// 类似的movable not copyable的还有：
// std::ifstream std::unique_ptr等
```

### 其他

```c++
#include <thread>
#include <iostream>
#include <chrono>

using namespace std;

void sleep()
{
    this_thread::sleep_for(chrono::seconds(1)); // 线程休眠1秒
}

int main()
{
    thread new_thread(sleep);
    new_thread.join();
    
    cout << new_thread.get_id() << endl; // 输出线程id
    cout << thread::hardware_concurrency() << endl; // 输出cpu核心数量
}
```

## 细节

### class based callable object

请先自行google什么是`callable object`。在使用基于类的callable obj的时候我们可以这么写：

```c++
#include <iostream>
#include <thread>

using namespace std;

class Foo
{
public:
    void operator ()() const
    {
        cout << "Call me maybe." << endl;
    }
};

int main()
{
    Foo f;
    f(); // 对象名+()即可
    // 同理，当将其用在线程中时：
    thread trd(f);
    trd.join();
}
```

但请注意，不要直接将构造函数作为实参传入

```c++
thread trd(Foo()); // Don't do like this.
```

这样做会被视为一个返回值为`thread`类型，接受`Foo`对象函数指针作为参数的**函数声明**。

如果要达到你的目的，你可以：

```c++
thread trd((Foo())); // 补小括号
thread trd{Foo()};   // 大括号
```

### `join` & `detach`

在一个线程析构（逃离作用域）之前，一定得使用`join/detach`，否则程序会奔溃。

而一个线程一旦开始建立并赋予函数，就会被加入调度系统并开始运行。

- `join`：查看对应线程是否干完活了，若干完了则继续走，如果没干完则等到其把活干完再继续（阻塞）。
- `detach`：不管你干没干完，直接往后走。
- `joinable`：判断改线程可否被`join`；
- `detach`之后`joinable`会变成`false`。

### 异常情况

对于一个`thread`，我们要在其析构之前使用`join/detach`。如果该线程在使用`join`前，主线程抛出了异常，那么就意味着`join`会被跳过（而上面我们说了，一个线程，无论如何都要让他被`join`或`detach`）。

通常，当倾向于在无异常的情况下使用join()时，需要在异常处理过程中调用join()，从而避免生命周期的问题。

```c++
std::thread t(some_func);
try
{
    // do sth.
}
catch(...)
{
    t.join();	// 在异常处理过程中调用join()，避免生命周期问题
    throw;
}

t.join();		// 无异常时调用join
```

### 利用RAII的思想保证线程最终能被`join/detach`

总之，我们的目的就是让线程在析构前，确保`join/detach`的使用。

为了say no to麻烦的异常处理，这里我们可以用到RAII的技术。可以自己写一个`thread_guard`，也可以撸一个智能指针版本。

```c++
// 手写版本
#include <iostream>
#include <thread>

class thread_guard
{
    std::thread t;
public:
    explicit thread_guard(std::thread& t_) : t(std::move(t_)){}
    ~thread_guard()
    {
        if(t.joinable()) t.join();
        std::cout << __FUNCTION__ << " deconstructed" << std::endl;
    }
    thread_guard(const thread_guard&) = delete; // 禁用常引用
    thread_guard& operator = (const thread_guard&) = delete;
};

struct func
{
    int &i; // 悬空的引用
    func(int& i_):i(i_){}
    void operator ()()
    {
        for (int j = 0; j < 10; ++j) i++;
    }
};

int main()
{
    int some_local_state = 0;
    std::thread trd{func(some_local_state)};

    thread_guard g(trd); // 默认在析构的时候join，无需手动join。
    // No join/detach: OK!
}
```

```c++
// 智能指针版本
#include <iostream>
#include <thread>
#include <memory>
#include <functional>

struct func
{
    int &i; // 悬空的引用
    func(int& i_):i(i_){}
    void operator ()()
    {
        for (int j = 0; j < 10; ++j) i++;
    }
};

int main()
{
    int some_local_state = 0;

    std::thread trd{func(some_local_state)};
    
    using unique_thread_ptr = std::unique_ptr<std::thread, std::function<void(std::thread*)>>;
    unique_thread_ptr trd_up(&trd, [](std::thread* pt) { 
         if(pt->joinable()) pt->join(); 
         std::cout << "Joined!" << std::endl; });
    // 不用额外delete，人家是遵循RAII写的。
    // No join/detach: OK!
}
```

> 同样，如果我们想直接往`thread_guard`中丢一个构造函数产生的临时对象，那么把`thread_guard(std::thread& t_)`中的`&`去掉即可。
>
> 我们知道`thread`不能直接copy，只能move，但对于临时对象（构造函数产生），`=`即是`move`。

### 线程转移

上面说到我们可以用`move`的方法转移线程。

考虑这样的情况：线程`t1`在执行`f1`，突然被`t2`通过`move`的方法赋值了。

```c++
std::thread t1(f1), t2;
t1 = std::move(t2);
```

这个时候，即使`t1`仍有任务在执行，`t1`中的任务也会被`std::terminate()`终结而继续运行。(`terminate`是一个不会抛异常的函数(`noexcept`))

### 批量产生线程

```c++
// 不在乎顺序
#include <iostream>
#include <thread>
#include <vector>
#include <functional>

using namespace std;

void say_i(int i)
{
    cout << i;
}

int main()
{
    vector<thread> threads;
    for (int i = 0; i < 10; ++i)
    {
        threads.emplace_back(thread(say_i, i));
    }
    for_each(threads.begin(), threads.end(), mem_fn(&thread::join));
}
// 若要求顺序是0123...9的话，只需要emplace_back后立即join。
/*
int main()
{
    vector<thread> threads;
    for (int i = 0; i < 10; ++i)
    {
        threads.emplace_back(thread(say_id, i));
        threads[i].join();
    }
}
*/

// 另外简单介绍一下std::mem_fn，其将`成员函数`(在这里是thread::join)转化为`函数`。
```

### 写个并行版本的`std::accumulate`

```c++
#include <numeric>
#include <algorithm>
#include <thread>
#include <vector>
#include <functional>
#include <chrono>

#include <iostream>

template<typename Iter, typename T>
void accumulate_block(Iter beg, Iter end, T& result)
{
    result = std::accumulate(beg, end, result);
}

template <typename Iter, typename T>
T parallel_accumulate(Iter beg, Iter end, T init)
{
    auto len = std::distance(beg, end);

    if ( !len ) { return init; }

    constexpr uint32_t len4per_thread = 1000; // 25 numbers at least for each thread.

    const uint32_t max_thread_num = len/len4per_thread + 1;
    const static uint32_t hardware_cpu_num = std::thread::hardware_concurrency();
    const uint32_t num_threads = std::min(max_thread_num, std::max<uint32_t>(hardware_cpu_num, 2));
    // When the CPU info couldn't be given, std::thread::hardware_concurrency() == 0, which will result in a bug.
    // So we use 2 as the minimum `num_threads`.
    // Use std::thread::hardware_concurrency() as the upper limit to avoid oversubscription.

    const uint32_t block_size = len/num_threads;

    std::vector<T> results(num_threads);
    std::vector<std::thread> threads(num_threads-1);
    // '-1', cause there's already one main thread here.

    for (int i = 0; i < num_threads-1; ++i)
    {
        threads[i] = std::thread(
                accumulate_block<Iter, T>,
                        beg+block_size*i,
                        beg+block_size*(i+1),
                        std::ref(results[i])
                );
    }
    accumulate_block<Iter, T>(end-block_size, end, results[num_threads-1]);

    std::for_each(threads.begin(), threads.end(), std::mem_fn(&std::thread::join));
    return std::accumulate(results.begin(), results.end(), init);
}

int main()
{
    size_t sz;
    std::cin >> sz;
    std::vector<int> vec(sz, 4);

    auto time_beg = std::chrono::steady_clock::now();
    auto ans1 = std::accumulate(vec.begin(), vec.end(), 0);
    auto time_mid = std::chrono::steady_clock::now();
    auto ans2 = parallel_accumulate(vec.begin(), vec.end(), 0);
    auto time_end = std::chrono::steady_clock::now();

    std::cout <<
              ">>> Non-parellel version: \n ans -> " << ans1 <<
              "\n time ->" << std::chrono::duration<double, std::milli>(time_mid-time_beg).count() << std::endl;

    std::cout <<
              ">>> Parellel version: \n ans -> " << ans2 <<
              "\n time ->" << std::chrono::duration<double, std::milli>(time_end-time_mid).count() << std::endl;
}
```



## OpenMP

一个简单易用的多线程并发API。OpenMP只是一个工具，至于如何使用，可以参考以下资源：

- [IBM](https://www.ibm.com/developerworks/cn/aix/library/au-aix-openmp-framework/index.html)

- [wiki](https://zh.wikipedia.org/wiki/OpenMP)（梯子 is required）。
- [小土刀](https://wdxtub.com/2016/03/20/openmp-guide/)

#### Ref

> 《c++并发编程实战》（除了翻译有点难受，其他都很好，请至少阅读前3章，其中的基础知识补充的比较详细）
>
> > 不喜欢纸质书翻译的可以移步[here](https://chenxiaowei.gitbooks.io/cpp_concurrency_in_action/content/content/about_this_book/about_this_book-chinese.html).
>
> 《深入应用c++11》第五章（讲得很简明）

#### 其他

> [这个repo](https://github.com/forhappy/Cplusplus-Concurrency-In-Practice)我大概瞄了一眼，感觉写的也不错。