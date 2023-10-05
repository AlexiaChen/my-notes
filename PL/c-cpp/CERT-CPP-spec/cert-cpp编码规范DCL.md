
# 声明

本文翻译自[CERT](http://www.cert.org/)(计算机安全应急响应组)提供的C++安全编码规范（2016版），与其他规范不一样的是，该规范侧重软件安全的编码规范。翻译不完全逐字段翻译，可能有简化删改。

# 正文

## 声明与初始化（DCL）

### DCL50-CPP 不要定义C风格的可变参函数

严重等级：高

- C风格的可变参函数的参数没有类型安全保证，也就是说，编译器不检查参数的类型是否匹配，如果类型不匹配这样会导致运行时出现未定义行为。这样的未定义行为可以让黑客很容易构造非法运行的代码（exploit）。

#### 不允许的代码样例
这个函数读取参数直到0才会终止循环，如果不以0终止参数，那么这个函数就会出现未定义行为。如果不小心传递成了其他非int类型的参数，那么编译阶段不会报错，运行时将出现未定义行为。
``` cpp
#include <cstdarg>

int add(int first, int second, ...)
{
    int r = first + second;
    va_list va;
    va_start(va,second);
    while(int v = va_arg(va, int))
    {
        r += v;
    }
    va_end(va);
    return r;
}

add(2,3,5,0); // print 10
add(2,3,5); //can be any number or even crash your PC
```

#### 允许的代码样例（递归包扩展）
``` cpp
#include <type_traits>

template <typename Arg, typename 
std::enable_if<std::is_integral<Arg>::value>::type * = nullptr>
int add(Arg f, Arg s)
{
    return f + s;
}

template <typename Arg, typename... Ts, typename
std::enable_if<std::is_integral<Arg>::value>::type * = nullptr>
int add(Arg f, Ts... tail)
{
    return f + add(tail...);
}

add<>(3,4,5); //print 12
add<>(3, 4, 5, 'c'); // compile error

```

以上的样例利用了std::enable_if来保证任何非整型的参数会导致ill-formed的程序。

#### 允许的代码样例（大括号初始化列表扩展）

``` cpp
template <typename Arg, typename... Ts, typename
std::enable_if<std::is_integral<Arg>::value>::type * = nullptr>
int add(Arg i, Arg j, Ts... all)
{
    int values[] = { i, j, all... };
    int r = 0;
    for (auto v : values)
    {
        r += v;
    }
    return r;
}

add<>(3,4,5); //print 12
add<>(3, 4, 5, 'c'); // compile error
```

### DCL51-CPP 不要声明或定义保留标识符

严重等级：低
- 避免与编译器的符号冲突

#### 代码样例对比（Header Guard）
``` cpp
//bad
#ifndef _MY_HEADER_H_
#define _MY_HEADER_H_

#endif // _MY_HEADER_H_

```

以上样例，大部分C++ 头文件是这样写法，自己写的头文件加前缀后缀下划线容易冲突

``` cpp
//good
#ifndef MY_HEADER_H
#define MY_HEADER_H

#endif // MY_HEADER_H

```



#### 代码样例对比（自定义字面量）
``` cpp
//bad
#include <cstddef>
unsigned int operator"" x(const char *, std::size_t);

```

``` cpp
//good
#include <cstddef>
unsigned int operator"" _x(const char *, std::size_t);

```



#### 代码样例对比（文件作用域对象）
``` cpp
//bad
#include <cstddef> // std::for size_t

static const std::size_t _max_limit = 1024;
std::size_t _limit = 100;

unsigned int get_value(unsigned int count) {
  return count < _limit ? count : _limit;
}

```

``` cpp
//good
#include <cstddef> // for size_t

static const std::size_t max_limit = 1024;
std::size_t limit = 100;

unsigned int get_value(unsigned int count) {
  return count < limit ? count : limit;
}
```

#### 代码样例对比（保留的宏）
``` cpp
//bad
#include <cinttypes> // for int_fast16_t

void f(std::int_fast16_t val) {
  enum { MAX_SIZE = 80 };
// ...
  }
}

```

``` cpp
//good
#include <cinttypes> // for std::int_fast16_t

void f(std::int_fast16_t val) {
  enum { BufferSize = 80 };
  // ...
  }
}

```



### DCL52-CPP 不要用const或volatile限定一个引用类型

严重等级：低

- C++会阻止或忽略这样的限定，只有非引用类型的值才能这样限定。

#### 代码样例对比
``` cpp
//bad
#include <iostream>
void f(char c) {
char &const p = c; // plz instead of char const &p;  Or: const char &p;
p = 'p';
std::cout << c << std::endl;
}
```

以上代码，会导致未定义行为，在VS2013 VS2015下，这段代码编译时会有警告warning C4227: anachronism used : qualifiers on reference are ignored。运行结果会是p，const 没有产生任何期望的作用。

在Clang 3.9下，这段代码直接产生编译错误error: 'const' qualifier may not be applied to a reference

``` cpp
//good
#include <iostream>

void f(char c) {
  const char &p = c;
  p = 'p'; //产生期望结果，编译错误，及时发现
  std::cout << c << std::endl;
}
```


### DCL53-CPP 不要写在语法上引起歧义的声明

严重等级：低

- 因为依赖编译器的起义规则来决定该声明的语义结果

#### 代码样例对比
``` cpp
//bad
#include <mutex>
static std::mutex m;
static int shared_resource;

void increment_by_42() {
  std::unique_lock<std::mutex>(m);
  shared_resource += 42;
}

```

以上样例是一个匿名的std::unique_lock对象，这样的语法歧义可能会被编译器解释成：
- 声明一个匿名std::unique_lock对象，并且调用自身的单参转化构造函数
- 声明了一个名为m的std::unique_lock对象，然后调用默认的构造函数构造

如果是情况2，那么m这个互斥量就永远不会被锁上了。

``` cpp
//good
#include <mutex>
static std::mutex m;
static int shared_resource;

void increment_by_42() {
  std::unique_lock<std::mutex> lock(m); //一定要加上对象名字
  shared_resource += 42;
}
```

#### 代码样例对比
``` cpp
#include <iostream>

struct Widget {
  Widget() { std::cout << "Constructed" << std::endl; }
};

void f() {
  Widget w(); // 有歧义
}

```
以上的代码本意是想声明一个类型为Widget局部变量w，然后执行它的默认构造函数。然而，这个声明有语法歧义，还可能是一个函数声明，这个函数是一个无参并且返回类型为Widget的函数。

所以，如果是第二种，那么程序根本就不会如你预想的输出Constructed文本。

``` cpp
#include <iostream>
struct Widget {
Widget() { std::cout << "Constructed" << std::endl; }
};
void f() {
Widget w1; // Elide the parentheses
Widget w2{}; // Use direct initialization
}

```
以上使用正确的初始化方法才能如预想工作。

#### 代码样例对比

``` cpp

#include <iostream>

struct Widget {
  explicit Widget(int i) { std::cout << "Widget constructed" << std::endl; }
};

struct Gadget {
  explicit Gadget(Widget wid) { std::cout << "Gadget constructed" << std::endl; }
};

void f() {
  int i = 3;
  Gadget g(Widget(i));// 该声明有歧义
  std::cout << i << std::endl;
}
```
以上的声明有歧义，不会被解释为一个Gadget类型的对象g，而是被解释成函数g，返回类型为Gadget。

``` cpp

#include <iostream>

struct Widget {
  explicit Widget(int i) {
    std::cout << "Widget constructed" << std::endl;
  }
};

struct Gadget {
  explicit Gadget(Widget wid) {
    std::cout << "Gadget constructed" << std::endl;
  }
};

void f() {
  int i = 3;
  Gadget g1((Widget(i))); // Use extra parentheses
  Gadget g2{Widget(i)}; // Use direct initialization
  std::cout << i << std::endl;
}

```
以上的代码才会正确输出：

``` txt
Widget constructed
Gadget constructed
Widget constructed
Gadget constructed
3
```

### DCL54-CPP 在同一个作用域同时重载alloc和dealloc的函数

严重等级：低

- 比如，一个重载的alloc函数使用私有堆来创建它的分配，传递进去的指针的值被默认的dealloc函数返回可能会导致未定义行为。

#### 代码样例对比

``` cpp

#include <Windows.h>
#include <new>

void *operator new(std::size_t size) noexcept(false) {
  // Private, expandable heap.
  static HANDLE h = ::HeapCreate(0, 0, 0);
  if (h) {
    return ::HeapAlloc(h, 0, size);
  }
  throw std::bad_alloc();
}
// No corresponding global delete operator defined.
``` 

以上样例中，alloc函数是在全局作用域中重载的，然而，与之匹配的dealloc函数却没有声明，如果有一个对象是这个重载的alloc函数分配的，那么用默认的delete 删除对象的时候会导致未定义行为

``` cpp

#include <Windows.h>
#include <new>

class HeapAllocator {
  static HANDLE h;
  static bool init;
public:
  static void *alloc(std::size_t size) noexcept(false) {
    if (!init) {
      h = ::HeapCreate(0, 0, 0); // Private, expandable heap.
      init = true;
    }

    if (h) {
      return ::HeapAlloc(h, 0, size);
    }
      throw std::bad_alloc();
  }

  static void dealloc(void *ptr) noexcept {
    if (h) {
      (void)::HeapFree(h, 0, ptr);
    }
  }
};

HANDLE HeapAllocator::h = nullptr;
bool HeapAllocator::init = false;

void *operator new(std::size_t size) noexcept(false) {
  return HeapAllocator::alloc(size);
}

void operator delete(void *ptr) noexcept {
  return HeapAllocator::dealloc(ptr);
}
```


#### 代码样例对比

``` cpp
#include <new>

extern "C++" void update_bookkeeping(void *allocated_ptr,
std::size_t size, bool alloc);

struct S {
  void *operator new(std::size_t size) noexcept(false) {
    void *ptr = ::operator new(size);
    update_bookkeeping(ptr, size, true);
    return ptr;
  }
};
```
以上代码，operator new() 实在类作用域重载的，但是却没有与之匹配的在类作用域重载的delete，所以当然分配好的类S的对象，需要删除的时候，调用的是全局默认的delete函数，这样就会导致程序处于一个未确定状态（未定义行为）。

``` cpp
#include <new>

extern "C++" void update_bookkeeping(void *allocated_ptr,
std::size_t size, bool alloc);

struct S {
  void *operator new(std::size_t size) noexcept(false) {
    void *ptr = ::operator new(size);
    update_bookkeeping(ptr, size, true);
    return ptr;
  }
  
  //需要一个与之匹配的delete
  void operator delete(void *ptr, std::size_t size) noexcept {
    ::operator delete(ptr);
    update_bookkeeping(ptr, size, false);
  }
};
```

### DCL55-CPP 当传递一个类对象需要跨边界的时候要避免信息泄露

严重等级：低

C++标准对于非union类的非静态数据成员的布局如下所述：

> Nonstatic data members of a (non-union) class with the same access control are
allocated so that later members have higher addresses within a class object. The order
of allocation of non-static data members with different access control is unspecified.
Implementation alignment requirements might cause two adjacent members not to be
allocated immediately after each other; so might requirements for space for managing
virtual functions and virtual base classes.

还有，关于类位域（bit-fields）的声明如下：

> Allocation of bit-fields within a class object is implementation-defined. Alignment of bitfields is implementation-defined. Bit-fields are packed into some addressable allocation
unit.

因此 padding bits可能出现在类对象实例中的任何内存地址上，这就导致其中可能包含一些敏感信息。

#### 代码样例对比

``` cpp
//bad
#include <cstddef>

struct test {
  int a;
  char b;
  int c;
};

// Safely copy bytes to user space
extern int copy_to_user(void *dest, void *src, std::size_t size);

void do_stuff(void *usr_buf) {
  test arg{1, 2, 3};
  copy_to_user(usr_buf, &arg, sizeof(arg));
}
```

如果以上代码运行在操作系统的内核空间，它把数据arg拷贝到用户空间，然而，这个test类型的对象可能有padding-bits（内存对齐），比如，为了保证类数据成员的对齐，这个padding-bits可能包含敏感信息，然后就导致这些信息从内核空间随着拷贝泄露到用户空间了。

``` cpp

#include <cstddef>

struct test {
  int a;
  char b;
  int c;
};

// Safely copy bytes to user space
extern int copy_to_user(void *dest, void *src, std::size_t size);

void do_stuff(void *usr_buf) {
  test arg{};
  arg.a = 1;
  arg.b = 2; // 关键在这里
  arg.c = 3;
  copy_to_user(usr_buf, &arg, sizeof(arg));
}

```

以上这段代码倒是在使用前用初始化保证了arg对象的字段都初始化为0了，用memset也可以达到同样效果。但是编译器对arg.b = 2 这个赋值的实现是自由的，也就是说，各个厂商的编译器实现可能不一样。可能有些编译器仅仅只是把最低8位赋值为2，高24位（padding-bits）没有变化，最后就导致高位字节随着拷贝泄漏到用户空间了。

其实可以给出以下两种方案解决这个信息泄露问题

``` cpp
#include <cstddef>
#include <cstring>

struct test {
  int a;
  char b;
  int c;
};

// Safely copy bytes to user space.
extern int copy_to_user(void *dest, void *src, std::size_t size);

void do_stuff(void *usr_buf) {
  test arg{1, 2, 3};
  // May be larger than strictly needed.
  unsigned char buf[sizeof(arg)];
  std::size_t offset = 0;
  std::memcpy(buf + offset, &arg.a, sizeof(arg.a));
  offset += sizeof(arg.a);
  std::memcpy(buf + offset, &arg.b, sizeof(arg.b));
  offset += sizeof(arg.b);
  std::memcpy(buf + offset, &arg.c, sizeof(arg.c));
  offset += sizeof(arg.c);
  copy_to_user(usr_buf, buf, offset /* size of info copied */);
}
```

以上这段代码就保证了未初始化的padding-bits不会被拷贝到用户空间去。

``` cpp
#include <cstddef>

struct test {
  int a;
  char b;
  char padding_1, padding_2, padding_3;
  int c;
  test(int a, char b, int c) : a(a), b(b),
  padding_1(0), padding_2(0), padding_3(0),
  c(c) {}
};

// Ensure c is the next byte after the last padding byte.
static_assert(offsetof(test, c) == offsetof(test, padding_3) + 1,
"Object contains intermediate padding");

// Ensure there is no trailing padding.
static_assert(sizeof(test) == offsetof(test, c) + sizeof(int),
"Object contains trailing padding");

// Safely copy bytes to user space.
extern int copy_to_user(void *dest, void *src, std::size_t size);

void do_stuff(void *usr_buf) {
  test arg{1, 2, 3};
  copy_to_user(usr_buf, &arg, sizeof(arg));
}
```

以上代码是通过显式声明padding-bits，但是这个方案不具有可移植性，因为依赖目标内存的架构与实现。以上代码是限定于x86-32的平台。

#### 代码样例对比

``` cpp

#include <cstddef>
class base {
public:
  virtual ~base() = default;
};

class test : public virtual base {
  alignas(32) double h;
  char i;
  unsigned j : 80;
protected:
  unsigned k;
  unsigned l : 4;
  unsigned short m : 3;
public:
  char n;
  double o;
  test(double h, char i, unsigned j, unsigned k, unsigned l,unsigned short m, char n, double o) :h(h), i(i), j(j), k(k), l(l), m(m), n(n), o(o) {}
  virtual void foo();
};

// Safely copy bytes to user space.
extern int copy_to_user(void *dest, void *src, std::size_t size);

void do_stuff(void *usr_buf) {
  test arg{0.0, 1, 2, 3, 4, 5, 6, 7.0};
  copy_to_user(usr_buf, &arg, sizeof(arg));
}
```

以上代码可能还是会把padding-bits泄露到用户空间中，因为padding-bits是implementation-defined，所以对象内存布局在各个编译器下不一样。可能出现以下的情况：

- 为了对齐产生的padding-bits在虚函数表之后，或者在虚基类的数据之后。之后才是各种数据成员

- padding-bits可能在各个拥有不同访问控制权限的数据成员之间

- 可能编译器会在类的实例中开辟出一个专门存放padding-bits的数组，这个数组位置也是implementation-defined

以下给出解决方案：

``` cpp
#include <cstddef>
#include <cstring>

class base {
public:
  virtual ~base() = default;
};

class test : public virtual base {
  alignas(32) double h;
  char i;
  unsigned j : 80;
protected:
  unsigned k;
  unsigned l : 4;
  unsigned short m : 3;
public:
  char n;
  double o;
  test(double h, char i, unsigned j, unsigned k, unsigned l,unsigned short m, char n, double o) :h(h), i(i), j(j), k(k), l(l), m(m), n(n), o(o) {}
  virtual void foo();
  bool serialize(unsigned char *buffer, std::size_t &size) {
    if (size < sizeof(test)) {
      return false;
    }
    std::size_t offset = 0;
    std::memcpy(buffer + offset, &h, sizeof(h));
    offset += sizeof(h);
    std::memcpy(buffer + offset, &i, sizeof(i));
    offset += sizeof(i);
    // Only sizeof(unsigned) bits are valid, so the following is
    // not narrowing.
    unsigned loc_j = j;
    std::memcpy(buffer + offset, &loc_j, sizeof(loc_j));
    offset += sizeof(loc_j);
    std::memcpy(buffer + offset, &k, sizeof(k));
    offset += sizeof(k);
    unsigned char loc_l = l & 0b1111;
    std::memcpy(buffer + offset, &loc_l, sizeof(loc_l));
    offset += sizeof(loc_l);
    unsigned short loc_m = m & 0b111;
    std::memcpy(buffer + offset, &loc_m, sizeof(loc_m));
    offset += sizeof(loc_m);
    std::memcpy(buffer + offset, &n, sizeof(n));
    offset += sizeof(n);
    std::memcpy(buffer + offset, &o, sizeof(o));
    offset += sizeof(o);
    size -= offset;
    return true;
  }
};

// Safely copy bytes to user space.
  extern int copy_to_user(void *dest, void *src, size_t size);

  void do_stuff(void *usr_buf) {
    test arg{0.0, 1, 2, 3, 4, 5, 6, 7.0};
    // May be larger than strictly needed, will be updated by
    // calling serialize() to the size of the buffer remaining.
    std::size_t size = sizeof(arg);
    unsigned char buf[sizeof(arg)];
    if (arg.serialize(buf, size)) {
      copy_to_user(usr_buf, buf, sizeof(test) - size);
    } else {
      // Handle error
    }
}
```

以上代码就手工上保证没有初始化的padding-bits不会泄漏到用户空间了。


### DCL56-CPP 静态对象的初始化期间避免循环初始化

严重等级：低

- 可能导致未指定行为（unspecified behavior）
- 可能导致死锁

####  代码样例对比

``` cpp
#include <stdexcept>

int fact(int i) noexcept(false) {
  if (i < 0) {
  // Negative factorials are undefined.
    throw std::domain_error("i must be >= 0");
  }

  static const int cache[] = {
    fact(0), fact(1), fact(2), fact(3), fact(4), fact(5),
    fact(6), fact(7), fact(8), fact(9), fact(10), fact(11),
    fact(12), fact(13), fact(14), fact(15), fact(16)
  };

  if (i < (sizeof(cache) / sizeof(int))) {
    return cache[i];
  }
    return i > 0 ? i * fact(i - 1) : 1;
}
```

以上的代码本意是想用cache的思想实现一个高效的求阶乘的函数，但是静态数组cache的初始化包含了递归，这个行为是未定义的，即使这个递归是有边界的。

从实现上看，VS2015和GCC 6.1.0 中，这个静态数组的初始化在线程安全的方式下可能死锁。而且还不一定能算出正确结果。

下面给出解决方案

``` cpp
#include <stdexcept>

int fact(int i) noexcept(false) {
  if (i < 0) {
    // Negative factorials are undefined.
    throw std::domain_error("i must be >= 0");
  }

  // Use the lazy-initialized cache.
  static int cache[17];
  
  if (i < (sizeof(cache) / sizeof(int))) {
    if (0 == cache[i]) {
      cache[i] = i > 0 ? i * fact(i - 1) : 1;
    }
    return cache[i];
  }
  return i > 0 ? i * fact(i - 1) : 1;
}
```

上面代码使用了懒初始化的方式，本质上是把静态初始化变成赋值了。完全可以计算出正确结果。

#### 代码样例对比

``` cpp
// file.h
#ifndef FILE_H
#define FILE_H

class Car {
  int numWheels;
public:
  Car() : numWheels(4) {}
  explicit Car(int numWheels) : numWheels(numWheels) {}
  int get_num_wheels() const { return numWheels; }
};
#endif // FILE_H

// file1.cpp
#include "file.h"
#include <iostream>

extern Car c;
int numWheels = c.get_num_wheels();

int main() {
  std::cout << numWheels << std::endl; // 不一定输出6
}

// file2.cpp
#include "file.h"

Car get_default_car() { return Car(6); }
Car c = get_default_car();
```
file1.cpp中numWheels的值