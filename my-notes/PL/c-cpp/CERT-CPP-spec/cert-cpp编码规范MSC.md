---
title: CERT C++ 编码规范翻译（MSC）
date: 2017-08-21 13:52:26
tags:
    - C/C++
---

## 其他功能杂项

### MSC50-CPP. 不要使用std::rand()生成伪随机数

严重程度： 中等。使用std::rand()函数可能直接造成可预测的随机数。

伪随机数生成器使用了数学算法来生成统计学上较好的随机数序列，但是产生的随机数序列不是真正意义上的随机。

C标准库提供的rand()函数通过C++ 标准库函数\<cstdlib\>暴露出的std::rand()对随机序列的生成没有任何质量保证。

一般对于应用程序需要一个强伪随机数生成器，需要高质量的随机数，并且需要高性能。

#### 代码样例对比

``` cpp
#include <cstdlib>
#include <string>

void f() {
  std::string id("ID"); // Holds the ID, starting with the
  // characters "ID" followed by a
  // random integer in the range [0-10000].
  id += std::to_string(std::rand() % 10000);
  // ...
}
```
以上代码显然用了std::rand()来生成随机数，但是生成的随机ID序列是可预测的，随机数质量有限。应该采用C++标准提供的随机数生成器:

``` cpp
#include <random>
#include <string>

void f() {
  std::string id("ID"); // Holds the ID, starting with the
  // characters "ID" followed by a random
  // integer in the range [0-10000].
  std::uniform_int_distribution<int> distribution(0, 10000);
  std::random_device rd;
  std::mt19937 engine(rd());
  id += std::to_string(distribution(engine));
  // ...
}
```
以上代码采用了C++标准库提供的在伪随机数生成上更加细粒度的控制机制。它把随机数生成分为了两个部分，一个是用算法提供很多随机值（引擎部分），一个是通过概率密度函数重新分布引擎提供的随机值。分布对象不是必须的，但是它可以确保这些随机值能够恰当的分布在给定的范围内，在此样例代码中，是采用[Mersenne Twister算法](https://en.wikipedia.org/wiki/Mersenne_Twister)作为引擎来生成随机值，然后通过[均匀分布](https://en.wikipedia.org/wiki/Uniform_distribution_\(continuous\))来代替取模运算生成给定范围内的随机数。

### MSC51-CPP. 确保随机数生成器被恰当的种子初始化

严重程度: 中等。

一个伪随机数生成器是使用确定性算法来生成逼近真随机数的随机数序列。每个序列可以通过伪随机数的初始状态和算法来完全确定。也就是说可以预测。大多数的伪随机数生成器都需要设置初始状态，也叫种子状态。

如果用相同的初始化状态来初始化伪随机数生成器(比如没有显式初始化或者直接初始化一个常量值)，那么最终生成的随机数序列在不同的程序上运行都是一样的结果，这样攻击者就容易预测出随机序列的值了。

所以方案就是，确保伪随机数生成器以恰当的初始种子初始化状态，这样就不可被预测和被攻击者掌控了。一个初始化正常的伪随机数生成器在每次程序运行都会生成不一样的随机数序列。

当然，不是所有的随机数生成器都需要种子初始化。真随机数生成器一般要依赖物理硬件来生成完全不可预测的随机数，这样的随机数生成器就不需要种子化。还有一些高质量的伪随机数生成器，比如在一些Unix系统上的/dev/random设备，也不需要种子化。只有完全依赖算法生成的伪随机数生成器才需要被种子初始化。

#### 代码样例对比

``` cpp
#include <random>
#include <iostream>

void f() {
  std::mt19937 engine;
  for (int i = 0; i < 10; ++i) {
    std::cout << engine() << ", ";
  }
}
```
以上代码很显然没有恰当的种子初始化伪随机数生成器，所以无论多少次执行，结果都是一样的。结果是以下:

```
1st run: 3499211612, 581869302, 3890346734, 3586334585, 545404204,
4161255391, 3922919429, 949333985, 2715962298, 1323567403,

2nd run: 3499211612, 581869302, 3890346734, 3586334585, 545404204,
4161255391, 3922919429, 949333985, 2715962298, 1323567403,

...

nth run: 3499211612, 581869302, 3890346734, 3586334585, 545404204,
4161255391, 3922919429, 949333985, 2715962298, 1323567403,
```
好吧，开发者之后又想到了一个方案，类似于C语言之前经常的种子化用法，用系统时间戳来做种初始化伪随机数生成器:

``` cpp
#include <random>
#include <iostream>

void f() {
  std::random_device dev;
  std::mt19937 engine(dev());
  
  for (int i = 0; i < 10; ++i) {
    std::cout << engine() << ", ";
  }
}
```
以上代码虽然每次运行生成出来的随机数序列都不一样了，但是一旦攻击者控制了系统时间的生成，那么它就可以控制种子的值了，还是会被攻击者攻击利用，所以需要改成以下终极方案:

``` cpp
#include <random>
#include <iostream>

void f() {
  std::random_device dev;
  std::mt19937 engine(dev());
  
  for (int i = 0; i < 10; ++i) {
    std::cout << engine() << ", ";
  }
}
```
以上代码使用std::random_device来生成一个随机的值来做种子，然后初始化Mersenne Twister算法引擎对象。std::random_device生成出来的随机值如果可能的话，那么随机值就是非确定性的，也就是不可预测，其随机强度依赖于底层的随机数生成设备，比如/dev/random。如果没有此类的设备，那么std::random_device会直接调用随机数引擎，然而，这个生成出来的随机值已经足够随机了，可以用其来做种子。

### MSC52-CPP. 如果函数是有返回值的，那么在所有函数退出路径中必须返回一个值

严重程度: 中等。如果忘记返回一个值，那么就会导致未定义行为，可能会导致数据完整性违例。

C++标准[stmt.return]有如下声明:

> Flowing off the end of a function is equivalent to a return with no value; this results in undefined behavior in a value-returning function.

对于异常处理部分，C++标准[except.handle]有如下说明:

> Flowing off the end of a function-try-block is equivalent to a return with no value; this results in undefined behavior in a value-returning function (6.6.3)

#### 代码样例对比

``` cpp
int absolute_value(int a) {
  if (a < 0) {
    return -a;
  }
}
```
以上代码显然遗漏了另一个条件，忘记返回一个int值了，一旦a等于大于0，那么会导致未定义行为，应该改成以下:

``` cpp
int absolute_value(int a) {
  if (a < 0) {
    return -a;
  }
  return a;
}
```

#### 代码样例对比

``` cpp
#include <vector>

std::size_t f(std::vector<int> &v, std::size_t s) try {
  v.resize(s);
  return s;
} catch (...) {

}
```
以上代码采用function-try-block的形式捕获异常，但是在catch块中，却没有返回一个值，一旦一个异常发生，那么就会导致未定义行为。应该改成以下:

``` cpp
#include <vector>

std::size_t f(std::vector<int> &v, std::size_t s) try {
  v.resize(s);
  return s;
} catch (...) {
  return 0;
}
```

### MSC53-CPP. 如果函数被声明为[[noreturn]]，那么就一定不能返回

严重程度： 中等。如果强制返回，那么直接会造成未定义行为，一般是数据完整性违例。

[[noreturn]]是指定一个函数没有返回，C++标准[dcl.attr.noreturn]有如下说明:

> If a function f is called where f was previously declared with the noreturn attribute and f eventually returns, the behavior is undefined.

注意，异常抛出不是返回，语义不一样。

#### 代码样例对比

``` cpp
#include <cstdlib>

[[noreturn]] void f(int i) {
  if (i > 0)
    throw "Received positive input";
  else if (i < 0)
    std::exit(0);
}
```
以上代码在i == 0的时候，不会通过任何条件，那么就会导致隐式返回，直接造成未定义行为。应该考虑所有执行路径，改成以下代码:

``` cpp
#include <cstdlib>

[[noreturn]] void f(int i) {
  if (i > 0)
    throw "Received positive input";
  std::exit(0);
}
```

### MSC54-CPP. 信号处理函数必须是一个平坦老式的函数

严重程度: 高。修复代价极高。如果不遵循条款，可能导致implementation-defined的行为，也可能是undefined-behavior 。

C++标准[support.runtime]有如下声明:

> The common subset of the C and C++ languages consists of all declarations, definitions, and expressions that may appear in a well formed C++ program and also in a conforming C program. A POF (“plain old function”) is a function that uses only features from this common subset, and that does not directly or indirectly use any function that is not a POF, except that it may use plain lock-free atomic operations. A plain lock-free atomic operation is an invocation of a function f from Clause 29, such that f is not a member function, and either f is the function atomic_is_lock_free, or for every atomic argument A passed to f, atomic_is_lock_free(A) yields true. All signal handlers shall have C linkage. The behavior of any function other than a POF used as a signal handler in a C++ program is implementation-defined.228

所有的信号处理函数必须这么做。

#### 代码样例对比

``` cpp
#include <csignal>

static void sig_handler(int sig) {
  // Implementation details elided.
}

void install_signal_handler() {
  if (SIG_ERR == std::signal(SIGTERM, sig_handler)) {
    // Handle error
  }
}
```
以上感觉信号处理函数平坦了，但是不是C风格的“老式”函数，因为在C++程序中，默认的链接性质是C++的，不是C语言的符号，所以以上代码会导致未定义行为，必须改成以下:

``` cpp
#include <csignal>

extern "C" void sig_handler(int sig) {
  // Implementation details elided.
}

void install_signal_handler() {
  if (SIG_ERR == std::signal(SIGTERM, sig_handler)) {
    // Handle error
  }
}
```

#### 代码样例对比

``` cpp
#include <csignal>

static void g() noexcept(false);

extern "C" void sig_handler(int sig) {
  try {
    g();
  } catch (...) {
    // Handle error
  }
}

void install_signal_handler() {
  if (SIG_ERR == std::signal(SIGTERM, sig_handler)) {
  // Handle error
  }
}
```
以上代码信号处理函数调用了一个可能会抛异常的函数，所以用try-catch来防止向信号处理函数外抛异常。因为异常不属于C/C++特性的公共子集范畴，所以以上代码可能会导致implementation-defined的行为。在以stack-based的架构中，信号的产生是异步的，所以可能导致栈帧没有被恰当的初始化，导致栈追踪(stack tracing)处于不稳定的状态。所以应该改成以下:

``` cpp
#include <csignal>

volatile sig_atomic_t signal_flag = 0;

static void g() noexcept(false);
  extern "C" void sig_handler(int sig) {
  signal_flag = 1;
}

void install_signal_handler() {
  if (SIG_ERR == std::signal(SIGTERM, sig_handler)) {
  // Handle error
  }
}

// Called periodically to poll the signal flag.
void poll_signal_flag() {
  if (signal_flag == 1) {
    signal_flag = 0;
    try {
      g();
    } catch(...) {
      // Handle error
    }
  }
}
```
以上代码就使用一个信号处理的flag的变量标记信号处理函数是否被调用，所以需要周期性的调用poll_signal_flag()函数来检查信号处理函数是否被调用，如果被调用，那么在来做之前的处理。