# Memory management

#### RAII

RAII是C++的发明者Bjarne Stroustrup提出的概念，RAII全称是“Resource Acquisition is Initialization”，直译过来是“资源获取即初始化”，也就是说在构造函数中申请分配资源，在析构函数中释放资源。因为C++的语言机制保证了，当一个对象创建的时候，自动调用**构造函数**，当对象超出作用域的时候会自动调用**析构函数**。所以，在RAII的指导下，我们应该使用类来管理资源，将资源和对象的生命周期绑定。

最简单的例子：

```c++
auto x = new int;
delete x;
```

一般为了达到RAII，我们在构造函数中申请好自己所需的资源（内存，设备…），并在析构函数中释放它们。

```c++
class Crazy
{
public:
    Crazy() : data(new char){}
    ~Crazy(){ std::cout << "CALL DECON\n"; delete data; }
private:
    char* data;
};
```

#### 内存泄漏

`new`出来后不`delete`。(大多数人写数据结构作业时会这么干)

而重复`delete`会导致程序奔溃（大多数人写数据结构作业时不delete的原因）：

```c++
auto x = new int;
*x = 10;

auto& y = x;

delete x;
delete y;// 崩了
```

#### 智能指针

有三种智能指针：`shared_ptr`, `unique_ptr` & `weak_ptr`。

需要头文件`<memory>`。

##### 大概

C++没有自动回收内存的机制，因此每一次new出来的动态内存必须手动delete回去。而智能指针可以解决这个问题，即对于**指向动态分配堆**的对象指针，当其离开指针作用域的时候，会通过某种方式（e.g. 引用计数）销毁动态分配的对象。

> 什么是**引用计数**：
>
> 了解 Objective-C/Swift 的程序员应该知道**引用计数**的概念。引用计数这种计数是为了防止内存泄露而产生的。基本想法是对于动态分配的对象，进行引用计数，初始化的时候，计数为一，每当增加一次对同一个对象的引用，那么引用对象的引用计数就会增加一次，每删除一次引用，引用计数就会减一，<u>当一个对象的引用计数减为零时，就自动删除指向的堆内存。</u>
>
> **注意**：引用计数不是垃圾回收，引用技术能够尽快收回不再被使用的对象，同时在回收的过程中也不会造成长时间的等待，更能够清晰明确的表明资源的生命周期。

##### `std::shared_ptr`

`std::shared_ptr`使用引用计数的方式，允许有多份shared_ptr的指针的拷贝，只有当最后一个shared_ptr析构的时候，内存才会被释放。

###### 创建

```c++
std::shared_ptr<int> p_int(new int(2));	// 初始化方法1
auto p_double = std::make_shared<double>(6.6); // 初始化方法2(更加高效)

auto p_int_ref = p_int; 	// 可复制

std::shared_ptr<int> p_ready;
p_ready.reset(new int(3));	// 先声明，再用reset方法初始化并重新计数。
// 不可以直接std::shared_ptr<int> p_int = new int(2);
```

###### 获取原始指针

```c++
// 使用get()方法
int* p_raw_int = p_int.get();
```

###### 指定删除器

好比在RAII中，我们在构造函数中申请资源，在析构函数中释放资源，析构函数就是我们的说的删除器，能够自定义删除行为，而在智能指针的应用中，我们可以自定义删除器。

```c++
void delete_diy(int* p)
{
    std::cout << "Diy deleted" << std::endl;
    delete p;
}

std::shared_ptr<int> p_int(new int(1), delete_diy);
// std::shared_ptr<int> p_int(new int(1), [](int* p){delete p;});
```

###### `shared_ptr`和数组

默认删除器都是直接`delete`，这样就不支持数组对象了（不理解这句可以先看下面），所以对于数组的初始化要额外多写一点：

```c++
std::shared_ptr<int> p_int(new int[3], delete_diy);
```

> 至于为什么不支持数组对象，是因为，`auto x = new int[N]`的x的类型还是`int*`，通过这个类型，我们无法知道这到底是一个`int`的数组，还是一个简单的`int`的指针。而shared_ptr的默认删除器，是遇到`int*`的时候，一律只`delete x`，而不会加上`[]`。故我们需要手动操作一下。

```c++
// 展示x的类型
#include<iostream>

#ifndef _MSC_VER
#include<cxxabi.h>
#endif

using namespace std;

int main()
{
    auto x = new int[3];
    
#ifndef _MSC_VER
    auto print_out = abi::__cxa_demangle(typeid(x).name(), nullptr, nullptr, nullptr);
#else
    auto print_out = typeid(x).name();
#endif
    
    cout << print_out << endl;
}
```

###### 其他注意事项

- 不要用一个原始指针去初始化多个shared_ptr，而应该用原始指针初始化完一个shared_ptr后，用该shared_ptr去给新的shared_ptr复制传值。
- 不要在函数实参中创建shared_ptr

```c++
void f(std::shared_ptr<int>, std::function<void()>)
{
    // do sth.
}
int main()
{
    f(std::shared_ptr<int>(new int), g());
    // Don't do like this!!!
}
```

对于实参初始化，由于编译器版本不同，可能从左到右，也可能反过来。假设是从左到右，先`new int`，然后`g()`抛出异常，那么`new int`无人`delete`，就会导致内存泄漏。取而代之的方法可以是这样：

```c++
int main()
{
    std::shared_ptr<int> p(new int);
    f(p, g());
    // Do like this!!!
}
```

- 还有很重要，但一般很少人会犯的2条，具体可以自己google，不再赘述（过一会再赘述）：
  - 一定要通过`shared_from_this`方法返回`this`指针
  - 避免循环引用

##### `std::unique_ptr`

`std::unique_ptr`是一种独占的智能指针，它禁止其他智能指针与其共享同一个对象，从而保证代码 的安全。

使用的原则为：**不能复制，只能移动**(`std::move`)。

```c++
auto u_ptr = std::make_unique<int>(10);
// make_unique只在c++14及以上的版本才有
// 至于为什么没提供：
// C++ 标准委员会主席Herb Sutter在他的博客中提到原因是『被他们忘记了』。
```

```c++
// 自己写个make_unique
templare<typename T, typename ...Args>
std::make_unique<T> make_unique(Args&& ...args)
{
    return std::unique<T>(new T(std::forward<Args>(args)...));
}
```

```c++
// 通过move转移
auto mvd = std::move(u_ptr);
```
此外，`unique_ptr`的指定删除器的方法和`shared_ptr`不同，`shared_ptr`无需提供删除器的类型，而`unique_ptr`需要。
```c++
// 仔细看<>里的东西，shared_ptr可以省略第二项。
std::unique_ptr<int, void(*)(int*)> ptr(new int(1), [](int* p){delete p;})
```

##### `std::weak_ptr`

用于解决循环引用等问题

关于循环引用的细节，请参考[Here](https://github.com/changkun/modern-cpp-tutorial/blob/master/book/zh-cn/05-pointers.md)。

> `weak_ptr`被造出来主要是用于“监视”`shared_ptr`，即通过一些方法，来查看`shared_ptr`的状态：

```c++
shared_ptr<int> sp(new int(3));
weak_ptr<int> wp(sp);

cout << wp.use_count() << endl;
if(wp.expired())			// expired方法用来判断sp是否已被释放
    cout << "资源以及被释放" << endl;
auto x = wp.lock(); 		// lock方法返回weak_ptr对应的shared_ptr
cout << *x << endl;
```

###### 解决`return this`问题

> 上文提到“一定要通过`shared_from_this`方法返回`this`指针”，即不能直接将this指针包装成`std::shared_ptr`然后返回。

```c++
// Bad example.
#include<iostream>
#include<memory>

class BadExample
{
public:
    std::shared_ptr<BadExample> getme(){ return std::shared_ptr<BadExample>(this); }
    ~BadExample(){ std::cout << "Bad example instance ends." << std::endl; }
};

int main()
{
    std::shared_ptr<BadExample> beptr(new BadExample);
    auto copy = beptr->getme();

    std::cout << beptr.use_count() << std::endl;
    std::cout << copy.use_count() << std::endl;
}
/*
clang下的输出：
CXX_TEST(26017,0x10c0915c0) malloc: *** error for object 0x7f8dfdc00610: pointer being freed was not allocated
CXX_TEST(26017,0x10c0915c0) malloc: *** set a breakpoint in malloc_error_break to debug
1
1
Bad example instance ends.
Bad example instance ends.

Process finished with exit code 6
*/
```

上面的情况中，之所以出现了奔溃，是因为有重复删除的现象。原因很简单，copy和beptr是独立的2个智能指针，他们管理了相同的资源，所以最后还是会出现double free的现象。如果能让这两个指针互相“通信”，或者说让copy成为beptr的引用（anyway，即让copy&beptr信息合并）。那么就不会出现double free。

> 说到这里，**单个**share_ptr保证的是“<u>No double free</u>”。
>
> **引用计数**为的是“<u>确定析构时机</u>”。
>
> 上述讲到的问题是由于 多个share_ptr管理相同资源，从而导致了"double free"。

**Guidance**

- 不要对同一资源建立多个分离的智能指针；
- **一般不要使用返回this的这个功能**，对于<u>释放堆内存区域资源</u>的智能指针而言，完全可以不用返回this。

**解决方案**

```c++
// Good example.
#include<iostream>
#include<memory>

class BadExample
        : public std::enable_shared_from_this<BadExample>
{
    // 继承enable...this类，其中包含了一个weak_ptr，并且包含shared_from_this()方法，
    // 直接通过weak_ptr的lock方法返回对于this的智能指针，和原始指针say goodbye.
public:
    std::shared_ptr<BadExample> getme(){return shared_from_this();}
    ~BadExample(){std::cout << "Bad example instance ends." << std::endl; }
};

int main()
{
    std::shared_ptr<BadExample> beptr(new BadExample);
    auto copy = beptr->getme();

    std::cout << beptr.use_count() << std::endl; // 2
    std::cout << copy.use_count() << std::endl;  // 2
}
```

#### 智能指针和第三方库

很多时候，我（们）在使用第三方库的时候容易忘记释放资源，我们希望借用智能指针，自动析构。我们知道对于一个智能指针，我们能自定义删除器，对于第三方库的一个对象，就用第三方库的析构方法作为智能指针的删除器。

```c++
void* p = Obj()->create();
std::shared_ptr<void> sp(p, [this](void* p){ Obj()->release(); });
```

但是这样写很麻烦，所以可以考虑使用宏：

```c++
#define GUARD(P) std::shared_ptr<void> p##p(p, [](void* p){ Obj()->release(); });

void* p = Obj()->create();
GUARD(p);
// 不直接封装成一个函数的原因是：若封装成一个带返回值的函数，而返回值没人接，那么资源就凉了。(当然这是使用者的问题)
```



#### Ref:

> Modern-cpp-tutorial(Github)
>