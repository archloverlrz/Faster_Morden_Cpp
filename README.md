# Faster Cpp

> A primary guidance to help coder write **std c++**(mainly) and **OpenCV Lib**  in an effective and efficient way.

## 前言

- Author：[**ganler**](https://github.com/ganler),  [**archloverlrz**](https://github.com/archloverlrz).
- prerequisites（请尝试解决下列问题）:
  - 用Google搜索任意关键字；
  - 写出任意一个bug-free的模板类（oop & template）;
  - 有哪些位运算，它们是如何计算的（CS）；
  - 线程，进程，并发，并行（CS - OS）；
  - 列举3个编译器的名字；
  - 使用命令行编译源文件；
  - 什么是内存泄漏，什么是double free；
  - 用c++做一个benchmark；
  - OpenCV的`cv::mat`的遍历；
  - ~~描述一下lrz有多nb。~~
- You will learn:

  - 以一种「高速度/开发效率」的模式用`modern c++`，更好地解决上述问题。
  - 想要深入了解？有哪些高质量的资料可供阅读和学习。
  - 一定程度的工程应用（避免如何堆💩山）。
  - 没有一定要写c++的目的，那么就转去写Rust吧@alanlee.（~~没事别写西++~~）
- Other:
  - 2位作者是现役Tongji University的大二学生，自入校以来研究西++已有2个春秋，平时积累了大量笔记，de过很多bug，也是效率偏执狂（经常一言不合就写benchmark互吹自己的code）。写这个东西主要是引导小白们写出bug-free并且高性能&~~高开发效率~~的modern c++，培养大家少做玩具，多写产品的想法。
  - 当然限于作者经历有限，所写的内容不能保证完全**准确**，相反，更多的时候走的是“易于理解”的风格。并且，所写的内容更多的是用于一般情况。

## 内容

#### Fast basis

快速上车。

#### IO operation

关于快速输入输出，以及iostream的无奈之处。

#### Tricks for bit operation

简单的位运算的应用，以及告诫大家别没事位运算，位运算并不一定更快（尤其是在release模式下）。

#### Useful  characteristics in modern c++

谈一谈编程的语义。

#### Memory management

RAII & 智能指针。

#### Cpp useful tools

茶余饭后学习使用STL中的小工具。

#### Multi-thread

多线程嘛...不会自己封装那就用API咯，API都不会就直接OpenMP咯。

```c++
// TODO:
// * some tips about OpenCV
// * Deep learning deployment with Cpp
```