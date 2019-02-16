# Useful  characteristics in modern cpp

### 问题的来源

管理对象的拷贝时机是影响性能的一个很重要的因素

### 语义和拷贝

对于c++来说，一个对象的行为像什么，区分了它的语义是值语义还是引用语义。

举一些具体的例子，比如说常见的标准库中的各种容器，都是值语义的。所谓值语义，就是像语言内置的基本类型一样，它的任何内容都是这个对象所拥有的，当这个对象被拷贝的时候，它所拥有的所有资源也都会被拷贝一遍。例如：

```c++
#include <vector>
#include <cassert>

int main() {
    std::vector<int> a{1, 2, 3};
	auto b = a;
	b.push_back(4);
	assert(a != b);
}
```

但是在编程中，引用语义的对象也非常常见，比如著名的库OpenCV中的一个核心类`cv::Mat`的对象，就是引用语义的对象。所谓的引用语义，就是对象本身并不拥有它所引用的对象，当对象被拷贝的时候，它所引用的资源并未被拷贝。对象本身的含义，只是它所引用资源的一个引用而已。当被拷贝出去的对象被修改的时候，原来的对象也会同时被修改，因为它们引用的是同一份资源，这就被称为浅拷贝。同时，这样的对象往往提供了clone一类的方法来使得库的用户可以获得引用对象的新的拷贝，这称为深拷贝。下面的代码展示了深浅拷贝的区别。

```c++
#include <opencv2/opencv.hpp>
#include <cassert>

bool matEqual(const cv::Mat& img0, const cv::Mat& img1) {
    return cv::norm(img0, img1, cv::NORM_L1) == 0;
}

int main() {
    cv::Mat A(2, 2, CV_8UC3, cv::Scalar(0,0,255));
    cv::Mat B(A);							// 浅拷贝
    cv::Mat C = A.clone();					// 深拷贝
    
    //修改B
    B.at<cv::Vec3b>(0, 0)[0] = 255;
    
   	//A和B引用的是一个份资源所以修改了B之后和A仍是一样的
    assert(matEqual(A, B));
    //C和原来的A一样，和现在的A已经不一样了
    assert(!matEqual(A, C));
}
```

### 值语义还是引用语义

其实从标准库的偏好，我们已经可以初见端倪，标准库中，几乎所有的容器类都是实现成为值语义的。至于为什么这么做，原因其实很简单，c++的所有资源需要一个明确的释放时机。值语义对象的释放时机是非常好确定的，每个对象拥有一份自己的资源，在析构时释放掉即可，但是引用语义的实现则没有那么简单。比如下面，看起来实现了引用语义，但实际上将会出现`double free`的问题：

```c++
#include <cstdio>
#include <cstring>

class Reference
{
public:
    Reference(const char* str) : data(new char[std::strlen(str) + 1]){
        std::strcpy(data, str);
    }
    
    Reference(const Reference& x) : data(x.data) {}
    
    Reference& operator=(const Reference& x) {
        data = x.data;
        return *this;
    }
    
    ~Reference(){
        printf("deconstructor: delete at %p, value is %s\n", data, data);
        delete[] data;
    }
private:
    char* data;
};

int main() {
    Reference A = Reference("Hello");
    Reference B(A);
    // 将会出现double free的现象
}
```

在我的电脑上运行的结果为：

```
deconstructor: delete at 0x557e33e01e70, value is Hello
deconstructor: delete at 0x557e33e01e70, value is 
```

因为B在离开它的作用域时，将它从A那里获得的引用资源释放掉了（B是建立在内存栈上的资源，离开作用域（那个大括号`{}`）后，就会被析构，A也一样）。后面main函数结束的时候，B会先析构掉其指向的资源，然后再是A析构掉其指定的资源，而A和B因为指向同一资源，这个资源被释放了2次，这就被称作double free，是很严重的内存错误。

既然这样地释放资源会导致double free，那能不能不释放资源呢，这就更不用说了，这是明显会造成**内存泄露**的做法（即申请了一块内存却不在用完后回收，而内存又是有限的）。

实际上，在c++中实现这样的引用语义对象，需要额外的操作例如**引用计数**(可以通过智能指针解决)，比如修改上面的实现为：

```c++
#include <cstdio>
#include <iostream>
#include <cstring>

class Reference
{
public:
    Reference(const char* str) : data(new char[std::strlen(str) + 1]), count(new size_t(1)){
        std::strcpy(data, str);
    }

    Reference() : data(new char[1]), count(new size_t(1)){}

    Reference(const Reference& x) : data(x.data), count(x.count) {
        ++(*count);
    }

    Reference& operator=(const Reference& x) {
        if (--(*count) == 0) {// 先析构再赋值/引用
            delete[] data;
        }
        data = x.data;
        count = x.count;
        ++(*count);
        return *this;
    }

    ~Reference(){
        if (--(*count) == 0) {
            printf("deconstructor: delete at %p, value is %s\n", data, data);
            delete[] data;
        }
    }
private:
    char* data;
    size_t* count;
};

int main() {
    Reference A = Reference("Hello");
    Reference B;
    B = A;
}
```

在我的电脑上运行的结果是：

```
deconstructor: delete at 0x557ec7392e70, value is Hello
```

但是上面的实现是线程不安全的，如果要保证线程安全，那么还要在类内的加锁，这样，在单线程下，就会造成不必要的性能损失。`cv::Mat`在实现的时候就使用了类似的方法。

但是，总的来说，这种做法不太符合c++的zero-overhead principle。出于这些原因，在c++这门语言中，值语意是更加受到青睐的。

### 值语义在一种场景下的问题

当需要保存一个值语义的对象的时候，问题就来了。大部分情况下，事情是很简单的，我们将需要保存的对象进行拷贝，就可以了，这总是可以正确的工作的。问题在于，有时候，我们刚刚拷贝完了一份内容，原有的内容就紧接着被析构了。当要保存的东西是引用语义的时候，这样的情景不构成问题，本身对象的拷贝就是浅拷贝，这并没有性能问题。但是比如说当我们想保存的东西是一个很大的vector的时候，问题就来了。将一个vector拷贝一份再析构一遍，消耗的时间很多，并且最关键的是，这样的操作毫无意义，如果能直接接管原有vector的内容就好了。但同时，我们又不能接管所有vector的内容，因为更多的vector不是即将要被析构的。

那如何区分这两种情形呢。事实上，在c++11之前，这是没有办法做到的。但是从c++11开始，我们有了非常有力的工具，那就是右值引用。

因为所有右值都是即将被析构的值，即使不是右值，也可以通过类型转换(`std::move`)，将其转换为右值引用类型来显示的表明用户保证不再使用这个值。这个时候，可以接收右值引用的函数，就可以将原来的对象做出类似清空的操作，以保证它可以安全的析构，并放心地接管原有对象的内容。这就是所谓的**移动语义**。

![img](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1550295836027&di=aaa0572e3cbcf1fab5e7174c6dca44ee&imgtype=0&src=http%3A%2F%2Fimage.bubuko.com%2Finfo%2F201710%2F20180111005357283810.png)

> [图源](http://www.bubuko.com/infodetail-2367943.html)

```c++
#include <string>
#include <iostream>

class A;

std::ostream& operator << (std::ostream& os, const A& value);

class A {
    friend std::ostream& operator<<(std::ostream& os, const A& value);
public:
    A(const char* str): p(new std::string(str)) {}
    
    A(const A& x): p(new std::string(*(x.p))) {
        std::cout << "copy constructor" << std::endl;
    }
    
    A(A&& x) noexcept: p(x.p) {
        x.p = nullptr;
        std::cout << "move constructor" << std::endl;
    }
    
    A& operator=(A x) {
        auto temp = x.p;
        x.p = p;
        p = temp;
        return *this;
    }
    
    bool empty() const noexcept {
        return !p;
    }
    
    ~A() { delete p; }
private:
	std::string *p;
};

std::ostream& operator<<(std::ostream& os, const A& value) {
   	os << *value.p;
    return os;
}
```

这是一个典型的使用右值引用实现了移动语义的类。那么对于deque来说，它的push_back有两个签名如下的重载

```c++
template<class T, class Allocator = std::allocator<T>>
void deque<T, Allocator>::push_back(const T& value);

template<class T, class Allocator = std::allocator<T>>
void deque<T, Allocator>::push_back(T&& value);
```

第一个重载会调用T类型的拷贝构造函数，而第二个重载会调用T类型的移动构造函数，来为容器添加成员。示例如下

```c++
#include <deque>
#include <cassert>
#include <utility>

int main() {
    std::deque<A> a;
    A obj("Hello");
    a.push_back(obj);
    a.push_back(std::move(obj));
    a.push_back(A("World"));
    assert(obj.empty());
    for (const auto &v: a) {
        std::cout << v << std::endl;
    }
}
```

输出的结果即为

```
copy constructor
move constructor
move constructor
Hello
Hello
World
```

第一次push_back被调用的时候，obj是左值，无法匹配第二个重载，只能绑定到const引用，push_back的第一个重载使用了A的拷贝构造函数，而A是被实现为值语意的所以A中成员p所指向的内容也被拷贝了。第二次调用push_back时，我们使用std::move函数将obj转换为右值引用类型，此时既可以绑定到const引用，亦可以绑定到右值引用，但是右值引用是更精确地匹配，因此使用了push_back的第二个重载，这个重载使用了A的移动构造函数，接管了obj原有的资源，最后一次时，A("World")是一个右值，同样更精确地匹配到了push_back的第二个重载，deque就接管了这个临时对象的资源，而并未产生拷贝。

通过assert我们可以看到，被接管后，obj已经被清空了，obj是一个不指向任何资源的无效值，无法再被使用，但是可以被安全的析构。这也是为什么右值引用没有const符号的原因，因为要被移动的对象绝不能是const的，const的对象是无法被移动的。

最后，我们看到无论是拷贝还是移动，deque最后都准确的获得了其所需要的对象。这是由A的正确实现所保证的，A的拷贝和移动语义都要实现作者希望其实现的功能，这个类才能被正确的使用。

