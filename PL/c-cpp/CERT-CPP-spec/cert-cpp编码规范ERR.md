

## 异常和错误处理

### ERR50-CPP. 不要突然的终止程序

严重程度: 低。 允许程序在运行中非正常终止会导致一些资源没有被释放，容易遭受服务拒绝攻击。

std::abort(),std::quick_exit()和std::_Exit()函数通常用来快速突然的终止程序运行。一旦它们被调用，那么是不会调用std::atexit()函数注册的exit handler的，也不会自动调用程序中各类对象的析构函数，线程对象的析构函数也不会被调用，还有网络socket连接等等资源。程序的扫尾工作不可能进行。

std::terminate()函数调用terminate_handler函数，该handler函数默认调用std::abort()函数。

C++标准定义了std::termiate()函数可能在以下几种情况被间接调用:

- When the exception handling mechanism, after completing the initialization of the exception object but before activation of a handler for the exception, calls a function that exits via an exception ([except.throw], paragraph 7)

- When a throw-expression with no operand attempts to rethrow an exception and no
exception is being handled ([except.throw], paragraph 9)

- When the exception handling mechanism cannot find a handler for a thrown exception
([except.handle], paragraph 9)

- When the search for a handler encounters the outermost block of a function with a noexceptspecification that does not allow the exception ([except.spec], paragraph 9)

- When the destruction of an object during stack unwinding terminates by throwing an
exception ([except.ctor], paragraph 3)

- When initialization of a nonlocal variable with static or thread storage duration exits via an exception ([basic.start.init], paragraph 6)

- When destruction of an object with static or thread storage duration exits via an exception ([basic.start.term], paragraph 1)

- When execution of a function registered with std::atexit() or std::at_quick_exit() exits via an exception ([support.start.term], paragraphs 8 and 12)

- When the implementation’s default unexpected exception handler is called ([except.unexpected], paragraph 2) Note that std::unexpected() is currently deprecated.

- When std::unexpected() throws an exception that is not allowed by the previously violated dynamic-exception-specification, and std::bad_exception() is not included in that dynamic-exception-specification ([except.unexpected], paragraph 3)

- When the function std::nested_exception::rethrow_nested() is called for an object that has captured no exception ([except.nested], paragraph 4)

- When execution of the initial function of a thread exits via an exception ([thread.thread.constr], paragraph 5)

- When the destructor is invoked on an object of type std::thread that refers to a joinable thread ([thread.thread.destr], paragraph 1)

- When the copy assignment operator is invoked on an object of type std::thread that refers to a joinable thread ([thread.thread.assign], paragraph 1)

- When calling condition_variable::wait(), condition_variable::wait_until(), or condition_variable::wait_for() results in a failure to meet the postcondition: lock.owns_lock() == true or lock.mutex() is not locked by the calling thread ([thread.condition.condvar], paragraphs 11, 16, 21, 28, 33, and 40)

- When calling condition_variable_any::wait(), condition_variable_any::wait_until(), or condition_variable_any::wait_for() results in a failure to meet the postcondition: lock is not locked by the calling thread ([thread.condition.condvarany], paragraphs 11, 16, and 22)

所以在以上那么多场景下，隐式调用std::terminate()直接导致调用栈不能收缩，C++标准[except.terminate]有如下声明:

> In the situation where no matching handler is found, it is implementation-defined whether or not the stack is unwound before std::terminate() is called. In the situation where the search for a handler encounters the outermost block of a function with a noexcept-specification that does not allow the exception, it is implementation-defined whether the stack is unwound, unwound partially, or not unwound at all before std::terminate() is called. In all other situations, the stack shall not be unwound before std::terminate() is called.

不要显式或隐式调用std::quick_exit(),std::abort()和std::_Exit()。当然，在恰当的情况下可以调用std::exit()来终止程序，因为该函数的调用，会安全的执行程序运行中各类对象的析构函数和恰当的清理释放资源。

#### 代码样例对比

``` cpp
#include <cstdlib>

void throwing_func() noexcept(false);

void f() { // Not invoked by the program except as an exit handler.
  throwing_func();
}

int main() {
  if (0 != std::atexit(f)) {
    // Handle error
  }
  // ...
}
```
以上代码显然有问题，因为在exit handler函数中不小心调用了可能会抛出异常的函数，那么会间接导致std::terminate()调用。所以要阻止异常向exit handler函数外抛出:

``` cpp
#include <cstdlib>

void throwing_func() noexcept(false);

void f() { // Not invoked by the program except as an exit handler.
  try {
    throwing_func();
  } catch (...) {
    // Handle error
  }
}

int main() {
  if (0 != std::atexit(f)) {
    // Handle error
  }
  // ...
}
```

当然，如果程序发生不可能恢复的错误的时候，可以用std::abort,std::_Exit和std::terminate 

``` cpp
#include <exception>

void report(const char *msg) noexcept;

[[noreturn]] void fast_fail(const char *msg) {
  // Report error message to operator
  report(msg);
  // Terminate
  std::terminate();
}

void critical_function_that_fails() noexcept(false);

void f() {
  try {
    critical_function_that_fails();
  } catch (...) {
    fast_fail("Critical function failure");
  }
}
```

### ERR51-CPP. 处理所有的异常

> *首先这个条款我持有保留意见，不应该catch所有异常，有一些异常是开发者自身的编码原因导致的，比如容器越界等，如果catch住这类异常反而让程序继续运行，在错误的道路上越走越远，不好调试分析，不如立刻让它崩溃，还可以通过coredump查看现场原因。可能是CERT C++的规范偏向于软件安全方面的考量吧*

严重程度：低。允许程序非正常终止会导致资源不能及时释放，关闭。容易遭受拒绝服务攻击。

当一个异常被抛出，那么异常会试图匹配“离它最近的”处理代码块(handler)，handler也就是try catch块，如果没有匹配的handler，那么异常会一直往调用栈的上层抛出，直到main函数或者线程的入口函数，如果还没有匹配的handler，那么结果就会如C++标准[except.handler]所描述的那样,runtime库直接自动调用std::terminate()终止程序:

> If no matching handler is found, the function std::terminate() is called; whether or not the stack is unwound before this call to std::terminate() is implementationdefined.

危害显而易见了，由于直接非正常终止程序，那么程序中的对象都不会调用析构函数，程序的外部依赖资源（Socket连接，文件句柄）都来不及释放，这样就不符合ERR50-CPP的规范。

#### 代码样例对比

``` cpp
void throwing_func() noexcept(false);

void f() {
  throwing_func();
}

int main() {
  f();
}
```
以上代码可能会抛出异常，但是没有匹配的try catch代码块(handler)，所以最终std::terminate()被调用，程序非正常终止，应该改成以下:

``` cpp
void throwing_func() noexcept(false);

void f() {
  throwing_func();
}

int main() {
  try {
    f();
  } catch (...) {
    // Handle error
  }
}
```
异常处理应该终止于main函数内，不能再向main函数外抛了。这样调用栈才来得及收缩到main上。这样也方便管理资源。

#### 代码样例对比

``` cpp
#include <thread>

void throwing_func() noexcept(false);

void thread_start() {
  throwing_func();
}

void f() {
  std::thread t(thread_start);
  t.join();
}
```

以上代码显然有问题，因为main函数的本质也是一个线程入口函数，只不过是主线程入口函数而已。所以说，以上的代码的异常没有在线程入口函数内被就阻止，还会继续往外抛，那么线程函数就被非正常终止了，最终又会调用std::terminate()。所以在线程入口函数内就要catch住异常:

``` cpp
#include <thread>

void throwing_func() noexcept(false);

void thread_start(void) {
  try {
    throwing_func();
  } catch (...) {
    // Handle error
  }
} 

void f() {
  std::thread t(thread_start);
  t.join();
}
```

### ERR52-CPP. 不要使用setjmp()或longjmp()函数

严重程度: 低。 使用这两个函数非常容易造成资源没有按预料中的被销毁，这样就容易遭受服务拒绝攻击。

C语言标准库提供的setjmp()和longjmp()函数可以用来模拟异常的throw和catch机制，然而这样在内存中的跳转，会很容易忽略遗忘的资源释放，造成未定义行为。这样资源泄漏就导致容易遭受服务拒绝攻击。

C++标准[support.runtime]作出了以下声明:

> The function signature longjmp(jmp_buf jbuf, int val) has more restricted behavior in this International Standard. A setjmp/longjmp call pair has undefined behavior if replacing the setjmp and longjmp by catch and throw would invoke any non-trivial destructors for any automatic objects.

所以，如果用C++，那么就使用try catch来代替这两个函数吧。

#### 代码样例对比

``` cpp
#include <csetjmp>
#include <iostream>

static jmp_buf env;

struct Counter {
  static int instances;
  Counter() { ++instances; }
  ~Counter() { --instances; }
};

int Counter::instances = 0;

void f() {
  Counter c;
  std::cout << "f(): Instances: "
            << Counter::instances << std::endl;
  std::longjmp(env, 1);
}

int main() {
  std::cout << "Before setjmp(): Instances: "
            << Counter::instances << std::endl;
  if (setjmp(env) == 0) {
    f();
  } else {
    std::cout << "From longjmp(): Instances: "
              << Counter::instances << std::endl;
  }

  std::cout << "After longjmp(): Instances: "
            << Counter::instances << std::endl;
}
```
这段代码有相当严重的问题，会直接造成未定义行为。因为f()中的局部对象c可能由于longjump跳转没有调用它的析构函数而造成严重问题。

以下讨论下Clang 3.8在Linux下运行这段代码，输出是:

``` 
Before setjmp(): Instances: 0
f(): Instances: 1
From longjmp(): Instances: 1
After longjmp(): Instances: 1
```
以上代码就如分析的那样，没有调用对象c的析构函数，当然，也可能会调用析构函数，因为行为是未定义的。所以应该用try catch代替:

``` cpp
#include <iostream>

struct Counter {
  static int instances;
  Counter() { ++instances; }
  ~Counter() { --instances; }
};

int Counter::instances = 0;

void f() {
  Counter c;
  std::cout << "f(): Instances: "
            << Counter::instances << std::endl;
  throw "Exception";
}

int main() {
  std::cout << "Before throw: Instances: "
            << Counter::instances << std::endl;
  try {
    f();
  } catch (const char *E) {
    std::cout << "From catch: Instances: "
              << Counter::instances << std::endl;
  }
  
  std::cout << "After catch: Instances: "
            << Counter::instances << std::endl;
}
```
以上代码以抛异常处理异常的方式来实现，这样到了catch块中，局部对象c的析构函数就能如预料中那样被调用了，不会造成未定义行为，以下是Clang 3.8在Linux上的输出以证明此观点:

```
Before throw: Instances: 0
f(): Instances: 1
From catch: Instances: 0
After catch: Instances: 0
```

### ERR53-CPP. 在构造函数和析构函数中不要在function-try-block handler中引用基类或者类的数据成员

严重程度: 低。在构造函数异常handler或者析构函数异常handler中访问非静态数据会导致未定义行为。

C++标准[except.handle]有如下声明:

> Referring to any non-static member or base class of an object in the handler for a function-try-block of a constructor or destructor for that object results in undefined behavior

#### 代码样例对比

``` cpp
#include <string>

class C {
  std::string str;
public:
  C(const std::string &s) try : str(s) {
    // ...
  } catch (...) {
    if (!str.empty()) {
       // ...
    }
  }
};
```
以上代码构造函数采用了function-try-block的形式来处理异常，然而在其中直接引用了str成员对象，造成未定义行为。需要改成以下:

``` cpp
#include <string>

class C {
  std::string str;
public:
  C(const std::string &s) try : str(s) {
    // ...
  } catch (...) {
    if (!s.empty()) { //注意，这里是构造函数的参数s，不是成员对象str
       // ...
    }
  }
};
```
以上代码就完美避免产生未定义行为了。

### ERR54-CPP. catch handler的参数类型应该采用从most derived到least derived的优先级顺序考虑，从低到高。

严重程度: 中等。当自定义的异常发生如果catch打乱了异常优先级，那么会造成无法预料程序控制流混乱。

一般来说，所有自定义的异常或者预定义的异常都必须继承同一个基类(std::exception)，这样形成一个继承链，most derived的异常一般就是直接继承std::exception的类，least derived的异常就是离std::exception继承“最远”的类。

#### 代码样例对比

``` cpp
#include <exception>
// Classes used for exception handling
class B : public std::exception {};
class D : public B {};

void f() {
  try {
    // ...
  } catch (B &b) {
    // ...
  } catch (D &d) {
    // ...
  }
}
```
以上代码自定义了两个异常类，B和D。但是catch handler却考虑优先处理B，而不是D，而D的父亲却是B，所以catch(D& d)的代码块永远不会被执行，因为如果抛出D类型的异常全部被匹配到catch(B &b)的代码块了，所以要有限匹配小范围的异常:

``` cpp
#include <exception>
// Classes used for exception handling
class B : public std::exception {};
class D : public B {};

void f() {
  try {
    // ...
  } catch (D &d) {
    // ...
  } catch (B &b) {
    // ...
  }
}
```
以上代码无论抛出什么类型的异常都可以匹配到期待的catch代码块了。

### ERR55-CPP. 遵循异常规范

严重程度: 低。 抛出一个未确定的异常导致打断控制流，容易造成程序终止和拒绝服务攻击。

C++标准[except.spec]有如下声明:

> A function is said to allow an exception of type E if the constant-expression in its noexcept-specification evaluates to false or its dynamic-exception-specification contains a type T for which a handler of type T would be a match (15.3) for an exception of type E.

如果一个函数没遵循异常规范并抛出一个异常，那么直接会导致implementation-defined类型的程序终止。

如果一个函数声明了一个动态异常规范但是又抛出了一个不遵守规范的异常，那么会导致函数std::unexpected()被调用，当然，这个函数的行为可以被覆盖，但是默认情况下会导致一个std::bad_exception()的异常被抛出，除非在异常规范里面列出了std::bad_exception()，不然，std::terminate()就会被间接调用。

所以要严格遵守异常规范。

#### 代码样例对比

``` cpp
#include <exception>
class Exception1 : public std::exception {};
class Exception2 : public std::exception {};

void foo() {
  throw Exception2{};
  // Okay because foo() promises nothing about exceptions
}

void bar() throw (Exception1) {
  foo();
  // Bad because foo() can throw Exception2
}
```
以上代码中，函数bar()明显声明了只能抛出Exception1的异常规范，但是bar()函数却因为调用foo()函数可能会抛出Exception2的异常，所以是错的，会导致程序不正常终止，应该改成以下:

``` cpp
#include <exception>
class Exception1 : public std::exception {};
class Exception2 : public std::exception {};

void foo() {
  throw Exception2{};
  // Okay because foo() promises nothing about exceptions
}

void bar() throw (Exception1) {
  try {
    foo();
  } catch (Exception2 e) {
    // Handle error without rethrowing it
  }
}
```
以上代码让Exception2的异常终止于bar()函数内部，不再让其往外抛出，这样就是正确的代码了。或者可以改成以下，新增一个异常在动态异常规范中:

``` cpp
#include <exception>
class Exception1 : public std::exception {};
class Exception2 : public std::exception {};

void foo() {
  throw Exception2{};
  // Okay because foo() promises nothing about exceptions
}

void bar() throw (Exception1, Exception2) {
  foo();
}
```

#### 代码样例对比

``` cpp
#include <cstddef>
#include <vector>

void f(std::vector<int> &v, size_t s) noexcept(true) {
  v.resize(s); // May throw
}
```
以上代码声明了函数f()不会抛出任何异常的规范，但是vector::reszie函数就可能在内存不够的时候无法分配导致抛出异常，所以该段代码会导致程序不正常终止，应该改成以下:

``` cpp
#include <cstddef>
#include <vector>

void f(std::vector<int> &v, size_t s) {
  v.resize(s); // May throw, but that is okay
}
```
以上代码移除了动态异常规范，默认情况下，该函数允许任何异常被抛出。

下面来谈点C++编译器产商对于异常的外延:

一般来说，各个C++编译器产商都会加入自己的语言扩展来支持“指定一个函数抛不抛异常”，比如，Visual Studio就提供了 __declspec(nothrow)，Clang也支持 \_\_attribute\_\_((nothrow))的形式。但是这些产商都没有文档化如果违反这些扩展的异常规范会发生什么行为，我们姑且可以认为是未定义行为。

### ERR56-CPP. 保证异常安全

> *这里提下我本人的观点，C++大型项目和基础设施中，要满足保证这个条款正确实行太艰难了，所以Google C++ Coding Style才会一律禁止使用C++异常。所以以下内容更多的是作为参考*

严重程度: 高。如果代码不是异常安全的，那么一般情况下是导致资源泄漏，导致程序处于一个不一致，未知的状态，最终在程序运行在某个时间点上发生未定义行为，代码在第一个没保证异常安全的异常抛出后，运行了几天，甚至几年以后可能才发生未定义行为或者崩溃

恰当的处理错误和异常对于一个连续运行的正确的程序是至关重要的，而用C++提供的异常机制来报告错误也比返回错误码的方式要方面简洁。连C++的标准库都大量采用了异常来报错。

因为异常是打断了程序的执行路径，所以很容易导致资源泄漏，还有程序的状态出错，然而，代码为了避免这样的无法预料的情况就叫异常安全。

#### 代码样例对比

``` cpp
#include <cstring>

class IntArray {
  int *array;
  std::size_t nElems;
public:
  // ...
  ~IntArray() {
    delete[] array;
  }

  IntArray(const IntArray& that); // nontrivial copy constructor
  IntArray& operator=(const IntArray &rhs) {
    if (this != &rhs) {
      delete[] array;
      array = nullptr;
      nElems = rhs.nElems;
      if (nElems) {
        array = new int[nElems];
        std::memcpy(array, rhs.array, nElems * sizeof(*array));
      }
    }
    return *this;
  }
  // ...
};
```
以上代码自定义了一个int类型的数组类，问题就出现在拷贝赋值函数中，如果内存不够，导致new分配符抛出异常那么后续的操作将不会被执行，这时候成员变量nElems已经在new之前被赋值了，有了状态，但是异常发生了，该状态无效。那么这个IntArray的对象将处于一个未知的状态，如果还在该对象上操作，或者该对象析构，就会导致未定义行为。所以可想而知，保证异常安全举步维艰，按理来说，以上的代码在项目中经常会出现的，开发人员很难发现问题，但是却不是无懈可击的代码，经过充分考虑后就是要保证异常发生后，对象还处于安全的状态:

``` cpp
#include <cstring>

class IntArray {
  int *array;
  std::size_t nElems;
public:
  // ...
  ~IntArray() {
    delete[] array;
  }

  IntArray(const IntArray& that); // nontrivial copy constructor
  IntArray& operator=(const IntArray &rhs) {
    int *tmp(nullptr);
    if(rhs.nElems){
      tmp = new int[rhs.nElems];
      std::memcpy(tmp,rhs.array,rhs.nElems * sizeof(*array));
    }
    delete [] array;
    array = tmp;
    nElems = rhs.nElems;
    return *this;
  }
  // ...
};
```
仔细分析以上代码，在异常安全上已经无懈可击了，当new分配失败，异常抛出，却没有影响此刻IntArray类型对象的自身的状态，等到分配完毕和拷贝数据到局部指针tmp分配的内存空间完毕，万事俱备只欠东风的时候，再来修改IntArray类型对象自身的状态，包括释放当前对象的数组空间，把指针指向tmp所分配的空间，然后再是设置当前对象的数组元素的个数，然后返回当前对象。

### ERR57-CPP. 当处理异常的时候不要泄漏资源

严重程度: 低。内存和资源泄漏一般会最终会导致程序崩溃，如果攻击者通过特殊手段制造会导致资源泄漏的异常抛出，那么会导致程序反复泄漏资源内存，攻击者就可以制造拒绝服务攻击了。

这个在ERR56-CPP规范中也有支出，其实是两个规范是有相交兼容的地方。为了防止资源泄漏，所以要经常利用C++的RAII设计模式，当对象退出作用域，自动释放资源。

然而，构造函数就不提供这样的保护。因为构造函数的设计目地本来就包含资源的分配，如果它提前结束，也不会自动释放任何资源，C++标准[except.ctor]有如下声明:

> An object of any storage duration whose initialization or destruction is terminated by an exception will have destructors executed for all of its fully constructed subobjects (excluding the variant members of a union-like class), that is, for subobjects for which the principal constructor (12.6.2) has completed execution and the destructor has not yet begun execution. Similarly, if the non-delegating constructor for an object has completed execution and a delegating constructor for that object exits with an exception, the object’s destructor will be invoked. If the object was allocated in a new-expression, the matching deallocation function (3.7.4.2, 5.3.4, 12.5), if any, is called to free the storage occupied by the object.

所以在C++编程中一般推荐在构造函数发生错误时用抛异常的方式来报错，抛异常千万不要导致资源泄漏，包括在对象构造阶段，这个与MEM51-CPP规范兼容。

#### 代码样例对比

``` cpp
#include <new>

struct SomeType {
  SomeType() noexcept; // Performs nontrivial initialization.
  ~SomeType(); // Performs nontrivial finalization.
  void process_item() noexcept(false);
};

void f() {
  SomeType *pst = new (std::nothrow) SomeType();
  if (!pst) {
    // Handle error
    return;
  }
  try {
    pst->process_item();
  } catch (...) {
    // Process error, but do not recover from it; rethrow.
    throw;
  }
  delete pst;
}
```
以上代码很显然，在处理process_item()抛出的异常的时候，忘记释放pst已经分配好的内存资源了，导致了资源泄漏。所以很好改:

``` cpp
#include <new>

struct SomeType {
  SomeType() noexcept; // Performs nontrivial initialization.
  ~SomeType(); // Performs nontrivial finalization.
  void process_item() noexcept(false);
};

void f() {
  SomeType *pst = new (std::nothrow) SomeType();
  if (!pst) {
    // Handle error
    return;
  }
  try {
    pst->process_item();
  } catch (...) {
    // Process error, but do not recover from it; rethrow.
    delete pst;
    throw;
  }
  delete pst;
}
```
当然以上代码也不是没有缺点，缺点就是当要处理的异常种类增加的时候，那么每个catch块都会要写delete ptr来清除释放资源，而且还要保证清楚释放资源的操作不能抛出异常，不然又打破规范了。所以最好使用RAII机制:

``` cpp
struct SomeType {
  SomeType() noexcept; // Performs nontrivial initialization.
  ~SomeType(); // Performs nontrivial finalization.
  void process_item() noexcept(false);
};

void f() {
  SomeType st;
  try {
    st.process_item();
  } catch (...) {
    // Process error, but do not recover from it; rethrow.
    throw;
  }
  // After re-throwing the exception, the
  // destructor is run for st.
}
// If f() exits without throwing an exception,
// the destructor is run for st.
```
以上代码代码的好处就是不会遇到new分配失败的异常了，而且无论f()函数是通过throw退出，还是正常退出，对象st都会自动调用析构函数及时释放资源。

#### 代码样例对比

``` cpp
struct A {/* ... */};
struct B {/* ... */};

class C {
  A *a;
  B *b;
protected:
  void init() noexcept(false);
public:
  C() : a(new A()), b(new B()) {
    init();
  }
};
```
以上代码问题严重，new A和new B都可能抛出异常，如果new B抛出异常，导致构造函数异常，那么对象的析构函数也不会调用，这就导致了new A不能释放，资源泄漏。如果是init()发生异常，那么new A和new B都会泄漏。所以应该改成以下:

``` cpp
struct A {/* ... */};
struct B {/* ... */};

class C {
  A *a;
  B *b;
protected:
  void init() noexcept(false);
public:
  C() : a(nullptr), b(nullptr) {
    try {
      a = new A();
      b = new B();
      init();
    } catch (...) {
      delete a;
      delete b;
      throw;
    }
  }
};
```
以上代码就没有问题了，当抛出异常，都能及时释放资源，别惊奇，delete nullptr没有任何问题。但是上面代码虽然正确，但是不美观，可以利用智能指针这样的RAII机制简化代码:

``` cpp
#include <memory>

struct A {/* ... */};
struct B {/* ... */};

class C {
  std::unique_ptr<A> a;
  std::unique_ptr<B> b;
protected:
  void init() noexcept(false);
public:
  C() : a(new A()), b(new B()) {
    init();
  }
};
```
以上代码任何地方发生异常，std::unique_ptr对象一旦退出其作用域，就会调用析构函数释放资源。

### ERR58-CPP. 在main函数开始执行之前处理所有的异常

严重程度: 低。 如果异常没有被捕获，那么直接导致程序非正常终止，间接导致服务拒绝攻击。

不是所有的异常都能够被捕获，即使小心使用function-try-block这样的机制。C++标准[except.handle]有如下声明:

> Exceptions thrown in destructors of objects with static storage duration or in constructors of namespace scope objects with static storage duration are not caught by a functiontry-block on main() . Exceptions thrown in destructors of objects with thread storage duration or in constructors of namespace-scope objects with thread storage duration are not caught by a function-try-block on the initial function of the thread

当声明一个静态对象或线程对象，而且没有在该对象上使用function-try-block这样的机制的时候，那么这个对象的类的构造函数必须声明为noexcept，必须遵守ERR55-CPP规范。另外如果没有捕获的异常在main函数开始前被抛出，或者main函数执行完毕后被抛出，那么没有任何的机会来处理异常，直接导致implementation-defined的行为，参考ERR50-CPP规范，另外，注意析构函数的异常规范，参考DCL57-CPP规范。

#### 代码样例对比

``` cpp
struct S {
  S() noexcept(false);
};

static S globalS;
```
以上的代码显然有问题，因为static类型的对象globalS是全局的，一定是在main函数开始执行就构造初始化了，但是这个类S的构造函数可能会抛出异常，如果异常发生，这个异常在main函数开始之前就被抛出了，开发者没有任何机会来catch这个异常，那么程序非正常终止。应该把代码改成以下:

``` cpp
struct S {
  S() noexcept(false);
};

S &globalS() {
  try {
    static S s;
    return s;
  } catch (...) {
    // Handle error, perhaps by logging it and gracefully terminating the application.
  }
// Unreachable.
}
```
以上代码对象s只会构造初始化一次，类似于单例模式，把对象s的初始化推迟到第一次调用globalS()函数，但是这样的方式让开发者有机会来catch处理异常了，这样就不会导致程序异常终止。

#### 代码样例对比

``` cpp
#include <string>

static const std::string global("...");

int main()
  try {
    // ...
  } catch(...) {
    // IMPORTANT: Will not catch exceptions thrown
    // from the constructor of global
  }
```
以上的代码在main函数上采用了function-try-block的机制，所以main函数没有自己的花括号。即使用这样完整的捕获方式还是有问题，因为global对象的构造函数还是会发生于main函数执行之前，一旦发生异常，没有任何方式捕获，导致runtime库自动间接调用std::terminate()，程序非正常终止。解决办法也是有:

``` cpp
static const char *global = "...";
int main() {
// ...
}
```
以上代码直接用常量指针的方式实现常量字符串，这样就不会抛任何异常，有句话说的好，绕过异常安全问题，就是不使用异常。嗯，所以很理解Google的做法。如果坚持使用异常可以自定义一个string类:

``` cpp
#include <exception>
#include <string>

namespace my {
  struct string : std::string {
    explicit string(const char *msg,const std::string::allocator_type &alloc=std::string::allocator_type{}) noexcept
    try : std::string(msg, alloc) {} catch(...) {
      extern void log_message(const char *) noexcept;
      log_message("std::string constructor threw an exception");
      std::terminate();
    }
    // ...
  };
}

static const my::string global("...");

int main() {
// ...
}
```
以上的代码自定义了一个string类，并且在string类的构造函数上使用了function-try-block机制，如果构造发生异常，直接主动调用std::terminate()在main函数执行之前终止程序。因为在构造函数中的任何异常都不会逃逸到构造函数之外，所以构造函数标记上了noexcept，这样自定义的string类，就可以全局地声明static对象了，程序不会非正常终止了。

#### 代码样例对比

``` cpp
extern int f() noexcept(false);
int i = f();
int main() {
// ...
}
```
以上代码变量i本质也是个全局变量，其初始化依赖f()函数并且在main函数执行前，所以一旦f()抛出异常，那么程序就非正常终止了。所以按照之前的思路改就可以。

``` cpp
#include <exception>

int f_helper() noexcept {
  try {
    extern int f() noexcept(false);
    return f();
  } catch (...) {
    extern void log_message(const char *) noexcept;
    log_message("f() threw an exception");
    std::terminate();
  }
// Unreachable.
}

int i = f_helper();
int main() {
// ...
}
```
以上代码在f()函数之上做了一个wrapper，在这个helper函数中catch住所有的异常，阻止异常往外逃逸，这样程序的终止就在掌控范围内了。

### ERR59-CPP. 不要跨执行边界抛异常

严重程度： 高。跨执行边界抛异常导致的后果依赖于异常处理机制的实现细节，结果可有有多种，可能是正确的，可能直接就导致未定义行为。

如果动态库和可执行文件都是由同一个编译器的同一个版本编译的，那么跨边界抛异常可能是安全的，但是最好需要编译器设置一致的编译参数和链接参数，这样才有可能二进制兼容，跨执行边界抛异常才可能是安全的。

其实以上只是理想状态，为了代码复用，不可能每个动态库都是同一个编译器的同一个版本采用一致的参数来编译链接。而恰恰执行边界的界定就是两边可执行二进制文件是用不同的编译器编译的，这里的不同编译器，包括同产商的版本的不同，不同产商的编译器。

#### 代码样例对比

``` cpp
// library.h
void func() noexcept(false); // Implemented by the library maybe DLL or so,cannot be the static lib, cause static lib only be linked into exe file by same compilor

// library.cpp
void func() noexcept(false) {
  // ...
  if (/* ... */) {
    throw 42;
  }
}

// application.cpp
#include "library.h"

void f() {
  try {
    func();
  } catch(int &e) {
    // Handle error
  }
}
```
以上代码可能由于ABI不兼容导致程序出错，其实也隐式兼容了EXP60-CPP规则。

说点以上代码在不同编译器实现上的情况:

如果库的代码library.cpp是用MinGW-w64上默认安装的GCC 4.9编译的(不含任何特殊的编译参数)，那么异常的抛出会依赖zero-cost，table-based的一种异常模型，该模型底层基于[DWARF调试标准](http://dwarfstd.org/)同时使用Itanium ABI。 如果应用可执行文件代码用Visual Studio 2013编译，那么catch代码handler块底层就是基于SEH异常处理机制还有使用Microsoft ABI。这两种异常处理格式不兼容，ABI也不兼容，最终导致程序非正常终止。特别是，这种情况下，catch代码块根本捕获不到异常，最终导致std::terminate()被间接调用。

所以代码应该改成以下，最好通过C语言接口的方式，以返回错误码的形式报错:

``` cpp
// library.h
void func() noexcept(true); // Implemented by the library maybe DLL or so,cannot be the static lib, cause static lib only be linked into exe file by same compilor

// library.cpp
void func() noexcept(true) {
  // ...
  if (/* ... */) {
    return 42;
  }
  return 0;
}

// application.cpp
#include "library.h"

void f() {
 
  if( int err = func())
  {
    // Handle Error
  }
}
```
以上代码两边可执行体都采用同样的编译器同样的版本参数保持一致编译链接就可以了，异常处理机制和API都需要保证兼容。

### ERR60-CPP. 异常对象不要以拷贝构造的形式抛出

严重程度: 低。可能最终导致程序非正常终止，这样的间接后果就是资源泄漏，容易遭受拒绝服务攻击。

当一个异常被抛出的时候，异常对象是被throw操作符拷贝到一个临时对象中的，一般这个临时对象是用来初始化Handler的，C++标准[except.throw]有如下声明:

> Throwing an exception copy-initializes a temporary object, called the exception object. The temporary is an lvalue and is used to initialize the variable declared in the matching handler.

也就是用这个临时对象来传入匹配的catch(Exception& expObj)块的参数的。

因为异常对象是被拷贝到临时对象中的，所以会调用其拷贝构造函数来初始化临时对象，如果拷贝构造函数中又抛出了异常，那么C++ runtime库会自动调用std::terminate()，这个调用会导致未定义行为，这个与ERR50-CPP规范兼容。

所以异常对象的拷贝构造函数必须声明为noexcept，包括隐式定义的拷贝构造函数，如果声明了noexcept的函数还是抛出了异常那么就违背了ERR55-CPP规范。

#### 代码样例对比

``` cpp
#include <exception>
#include <string>

class S : public std::exception {
  std::string m;
public:
  S(const char *msg) : m(msg) {}
  const char *what() const noexcept override {
    return m.c_str();
  }
};

void g() {
  // If some condition doesn't hold...
  throw S("Condition did not hold");
}

void f() {
  try {
    g();
  } catch (S &s) {
    // Handle error
  }
}
```
以上代码可以看出，异常从g()中被抛出。然而异常类S有个成员对象std::string，而std::string的拷贝构造函数没有被声明为noexcept，而且异常类S的隐式定义(编译器默认生成)的构造函数也没有声明为noexcept，在低内存的场景下，std::string的拷贝构造函数可以由于无法分配足够的内存而导致抛出异常std::bad_alloc无法完成拷贝构造，间接导致了S的拷贝构造也抛出异常，异常对象构造失败，无法catch，程序非正常终止。

所以需要改成以下代码:

``` cpp
#include <stdexcept>
#include <type_traits>

struct S : std::runtime_error {
  S(const char *msg) : std::runtime_error(msg) {}
};

static_assert(std::is_nothrow_copy_constructible<S>::value,"S must be nothrow copy constructible");

void g() {
  // If some condition doesn't hold...
  throw S("Condition did not hold");
}

void f() {
  try {
    g();
  } catch (S &s) {
    // Handle error
  }
}
```
以上代码直接让异常类S继承了std::runtime_error类，使用runtime_error类保存异常信息，就弃用std::string来保存异常信息了，然后还用static_assert在编译时刻测试异常类S的拷贝构造函数不会抛出异常。或者不要继承std::runtime_error类，直接封装该类并继承std::exception类，这样最符合规范，少用继承，多用复合类的策略:

``` cpp
#include <stdexcept>
#include <type_traits>

class S : public std::exception {
  std::runtime_error m;
public:
  S(const char *msg) : m(msg) {}
  const char *what() const noexcept override {
    return m.what();
  }
};

static_assert(std::is_nothrow_copy_constructible<S>::value,"S must be nothrow copy constructible");

void g() {
  // If some condition doesn't hold...
  throw S("Condition did not hold");
}

void f() {
  try {
    g();
  } catch (S &s) {
    // Handle error
  }
}
```

### ERR61-CPP. catch异常的时候以左值引用的方式捕获异常对象

严重程度: 低。对象分片直接导致程序不正常终止，这种情况，一般对于异常对象本身没什么问题，但是会导致未指定行为，该行为依赖于异常handler的处理方式。

当一个异常被抛出的时候，它被拷贝到一个临时匿名对象中，这个临时对象就叫异常对象，该异常对象会匹配“距离它最近”的catch代码块并传入进去，C++标准[except.handle]中有如下声明:

> The variable declared by the exception-declaration, of type cv T or cv T&, is initialized from the exception object, of type E, as follows:

> • if T is a base class of E, the variable is copy-initialized from the corresponding base class subobject of the exception object;

> • otherwise, the variable is copy-initialized from the exception object.

也就是说catch代码的输入参数可以是引用捕获，或者是值传递的捕获方式，但是值传递的话，又会调用临时异常对象的拷贝构造函数，极端情况会发生对象分片，因为异常对象是有继承关系的，这样会导致继承类的异常对象的完整状态通过值赋值拷贝的方式发生信息丢失，导致不可恢复的错误，需要遵守OOP51-CPP规范，而且以值传递会调用临时异常对象的拷贝构造函数，万一其拷贝构造函数发生异常，那么程序又非正常终止了，这样违背了ERR60-CPP规范了。

所以总应该以左值引用捕获异常对象，除非异常类的类型是trivial type的。C++标准[basic.types]对trivial types作出了如下声明:

> Arithmetic types, enumeration types, pointer types, pointer to member types,std::nullptr_t, and cv-qualified versions of these types are collectively called scalar types.... Scalar types, trivial class types, arrays of such types and cv-qualified versions of these types are collectively called trivial types.

C++标准[class]还定义了trivial class types是什么:

> A trivially copyable class is a class that:

> • has no non-trivial copy constructors,

> • has no non-trivial move constructors,

> • has no non-trivial copy assignment operators,

> • has no non-trivial move assignment operators, and

> • has a trivial destructor.

> A trivial class is a class that has a default constructor, has no non-trivial default constructors, and is trivially copyable. [Note: In particular, a trivially copyable or trivial class does not have virtual functions or virtual base classes. — end note]

#### 代码样例对比

``` cpp
#include <exception>
#include <iostream>

struct S : std::exception {
  const char *what() const noexcept override {
    return "My custom exception";
  }
};

void f() {
  try {
    throw S();
  } catch (std::exception e) {
    std::cout << e.what() << std::endl;
  }
}
```
以上代码以传值的方式捕获异常对象，所以可能导致异常对象拷贝初始化对象e的时候被分片，先拷贝的std::exception 后拷贝异常类S，导致开发者无法预料的输出(implementation-defined)，可能不会输出"My custom exception"， 而是直接调用std::exception::what()。所以为了防止这样，应该改为引用捕获:

``` cpp
#include <exception>
#include <iostream>

struct S : std::exception {
  const char *what() const noexcept override {
    return "My custom exception";
  }
};

void f() {
  try {
    throw S();
  } catch (std::exception &e) {
    std::cout << e.what() << std::endl;
  }
}
```
以上代码永远始终会调用S::what()了。

### ERR62-CPP. 当把字符串转换成一个数值时，注意检测异常

严重程度: 中等。内存数据可能会冲突，导致非法代码被植入。

由字符串转换为一个数值时，这个过程中可能会发生很多错误异常，比如字符串根本就不是一个数值，或者字符串表示的数值已经超出int或者float，double所表示的范围了，可能字符串除了数值里面还包含一些非法的字符，其中都可能导致转换失败。

#### 代码样例对比

``` cpp
#include <iostream>

void f() {
  int i, j;
  std::cin >> i >> j;
  // ...
}
```
以上代码本质就是从输入流获取的字符串转换成了多个int类型的值，问题就出在从输入流获取的字符串是不可预料的，一旦int类型不能表示，或超出表示范围，你就无法预测了。很好改:

``` cpp
#include <iostream>

void f() {
  int i, j;
  std::cin.exceptions(std::istream::failbit |
  std::istream::badbit);
  try {
    std::cin >> i >> j;
    // ...
  } catch (std::istream::failure &E) {
    // Handle error
  }
}
```
以上可以对std::cin设置检测的异常，转换过程如果失败，就立刻抛出异常。还可以向以下这么改:

``` cpp
#include <iostream>
#include <limits>

void f() {
  int i;
  std::cin >> i;
  if (std::cin.fail()) {
    // Handle failure to convert the value.
    std::cin.clear();
    std::cin.ignore(
    std::numeric_limits<std::streamsize>::max(), ' ');
  }

  int j;
  std::cin >> j;
  if (std::cin.fail()) {
    std::cin.clear();
    std::cin.ignore(
    std::numeric_limits<std::streamsize>::max(), ' ');
  }
// ...
}
```
