

## 内存管理

### MEM50-CPP. 不要访问已经释放的内存

严重程度：高。读取动态分配的内存（已经释放）会导致不正常的程序终止和拒绝服务攻击。写动态分配的内存（已释放）会导致执行任意代码和恶意的代码提权操作。

#### 代码样例对比

``` cpp
#include <new>

struct S {
  void f();
};

void g() noexcept(false) {
  S *s = new S;
  // ...
  delete s;
  // ...
  s->f();
}
```

以上代码有个明显的错误，就是s指向的对象被释放了之后，还对s指针解引用。这种问题出现在大型工程里面不好查。当然noexcept(false)是为了遵循MEM52-CPP这个规范。这个是正确的。

知道了原理可以用以下多种方法来解决:

``` cpp
#include <new>

struct S {
  void f();
};

void g() noexcept(false) {
  S *s = new S;
  // ...
  s->f();
  delete s;
}
////////////////////////////
struct S { 
  void f();
};

void g() {
  S s;
  // ...
  s.f();
}
```

#### 代码样例对比

``` cpp
#include <string>

std::string str_func();

void display_string(const char *);

void f() {
  const char *str = str_func().c_str();
  display_string(str); /* Undefined behavior */
}
```

注意，以上代码str_func()返回的是一个临时std::string对象，然后再通过c_str()获取对象底层的C字符串指针，这导致一旦该赋值语句执行完毕，临时对象立即释放，所以str指针在执行到display_string的时候访问了std::string对象的生命周期以外，所以立即导致未定义行为。所以可以推迟对象的生命周期:

``` cpp
#include <string>

std::string str_func();
void display_string(const char *s);

void f() {
  std::string str = str_func();
  const char *cstr = str.c_str();
  display_string(cstr); /* ok */
}
```

这样来看的话，那么就把临时对象赋值给了新的str对象，而str对象的生命周期是f()函数的语句块范围内，就把其生命周期推迟到了display_string函数执行完毕之后，cstr指针的指向一直是有效值。

#### 代码样例对比

``` cpp
#include <new>

void f() noexcept(false) {
  unsigned char *ptr = static_cast<unsigned char *>(::operator new(0));
  *ptr = 0;
  // ...
  ::operator delete(ptr);
}
```

以上代码是想通过new操作符分配一个0字节的内存空间，并用ptr指针指向这段内存空间。如果分配成功，那么operator new()会返回一个非null指针。然而，根据C++标准[basic.stc.dynamic.allocation]，这样通过指针解引用一个0字节的内存空间会导致未定义行为。

如果你想分配一个单个unsigned char大小的对象，可以用new操作符来直接代替operator new()。

``` cpp
void f() noexcept(false) { 
  unsigned char *ptr = new unsigned char;
  *ptr = 0;
  // ...
  delete ptr;
}
```

如果你有分配一个0字节内存空间的需求，该需求可能是需要获取一个唯一的指针值，该值直到指针没被释放都不能被其他指针复用的指针值。防止解引用该指针造成的未定义行为可以改成以下代码:

``` cpp
#include <new>

void f() noexcept(false) {
  void *ptr = ::operator new(0); // ptr的值经过多次分配，值都是随机不重复的
  // ...
  ::operator delete(ptr);
}
```

### MEM51-CPP. 恰当的释放已经动态分配的资源

严重程度：高。不采用对应的dealloc函数会导致未定义行为。

在C语言中，语言标准库提供了几种分配内存的方式:
- std::malloc
- std::calloc
- std::realloc

这三种方式都可以在C++语言中使用，然而，只能使用std::free来释放已经分配的内存。

C++语言提供了另外的分配内存方式，比如new,new[]和placement new，还有分配器对象。不像C语言，C++提供了相应的几种释放动态分配内存空间的方式，比如delete,delete[]()和分配器对象的dealloc函数。

不要调用对nullptr指针调用dealloc函数，一定要采用与分配时候相对应的dealloc函数来释放内存资源。以下是分配和释放对应的方法列表:

| Allocator                     | Deallocator   |
| -------------                 |:-------------:| 
| global operator new()/new     | global operator delete()/delete |
| global operator new[]()/new[] | global operator delete[]()/delete[] |  
| class-specific operator new()/new | class-specific operator delete()/delete |   
| class-specific operator new[]()/new[] | class-specific operator delete[]()/delete[] |
| placement operator new() | N/A |
| allocator\<T\>::allocate() | allocator\<T\>::deallocate() |
| std::malloc(), std::calloc(),std::realloc() |  std::free() |
| std::get_temporary_buffer() | std::return_temporary_buffer() |

一旦相同的内存资源释放调用的函数与分配时候的函数不匹配对应，那么立即会造成未定义行为。

C++标准[expr.delete]有如下声明：

> In the first alternative (delete object), the value of the operand of delete may be a null
pointer value, a pointer to a non-array object created by a previous new-expression, or a
pointer to a subobject (1.8) representing a base class of such an object (Clause 10). If
not, the behavior is undefined. In the second alternative (delete array), the value of the
operand of delete may be a null pointer value or a pointer value that resulted from a
previous array new-expression. If not, the behavior is undefined.

#### 代码样例对比

``` cpp
#include <iostream>
struct S {
  S() { std::cout << "S::S()" << std::endl; }
  ~S() { std::cout << "S::~S()" << std::endl; }
};

void f() {
  S s1;
  S *s2 = new (&s1) S;
  // ...
  delete s2;
}
```
以上代码f()函数的局部对象s1被传入到placement new中，然而placement new返回的指针s2最后又被传到::operator delete() 中调用。这就直接导致了未定义行为，因为::operator delete()试图释放通过::operator new()分配还没有返回的内存，简而言之，就是，还没有分配完成并返回，就开始释放了，那当然错了。

其实要知道以上代码为什么会有问题，需要知道C++ placement new是什么特性？  平时最常用的是new，它底层的流程机制是这样的：分配内存空间，在该内存空间上调用构造函数初始化对象，把对象的地址返回给指针。但是如果已知给定一段内存空间的情况下，向在这片内存空间上初始化分配一个对象，那么new是办不到的，只有用placement new，它的作用就是在特定的内存空间上(不主动开辟内存空间)调用构造函数创建并初始化一个对象。

所以以上代码出错了，因为代码意图想在s1对象所在的内存空间上又创造并初始化一个S类型的对象，但是却显式调用delete，s2所指向的对象立即被析构释放了，而s1对象确实是存在的，所以在f()函数结束时，s1对象又调用一次自身的析构函数释放，就出现问题了。

应该改成以下方式:

``` cpp
#include <iostream>

struct S {
  S() { std::cout << "S::S()" << std::endl; }
  ~S() { std::cout << "S::~S()" << std::endl; }
};

void f() {
  S s1;
  S *s2 = new (&s1) S;
  // ...
}
```

#### 代码样例对比

``` cpp
#include <new>

void f() {
  int *i1, *i2;
  try {
    i1 = new int;
    i2 = new int;
  } catch (std::bad_alloc &) {
    delete i1;
    delete i2;
  }
}
```

以上代码不仔细分析看不出个所以然，try catch块包裹代码是为了防止new失败所抛出的异常，看似没有任何问题。然而，i1，i2指针没有初始化为nullptr，所以一旦代码抛出bad_alloc的异常，如果new一旦失败，::operator new()还没有返回值并赋值给i1或i2。那么::operator delete()所传入的指针值就是不合法的，因为指针没有被初始化，它可以是任何随机值，这就立即导致了未定义行为。所以应该初始化指针:

``` cpp
#include <new>

void f() {
  int *i1 = nullptr, *i2 = nullptr;
  try {
    i1 = new int;
    i2 = new int;
  } catch (std::bad_alloc &) {
    delete i1;
    delete i2;
  }
}
```
C++标准指出，delete 空指针不会有任何问题。所以即使分配失败抛出异常，这段代码也是没有问题的。

#### 代码样例对比

``` cpp
struct P {};

class C {
  P *p;
public:
  C(P *p) : p(p) {}
  ~C() { delete p; }
  void f() {}
};

void g(C c) {
  c.f();
}

void h() {
  P *p = new P;
  C c(p);
  g(c);
}
```

这段代码仔细看看出问题了吗？显然p所指向的对象导致了重复的二次释放，来主要来源于C++的浅拷贝所带来的问题。当在函数h()中p所指向的对象的生命周期与c对象的生命周期绑定了，然而，对象c被通过传值的方式传到了g()函数中，这里会隐式调用一个C类型的拷贝构造函数重新在g()函数的作用域范围内创建一个新的对象c，也就是有两个对象内部的p指针会指向同一个P类型的对象，当函数g()退出时，g()函数内的对象c会调用析构函数delete指针所指向的对象，h()函数退出时，h()函数内的对象c又会调用自己的析构函数delete指针所指向的对象，所以导致p指针所指向的对象被重复释放两次。

C++标准[class.copy]对于类的拷贝构造函数有如下声明:

> If the class definition does not explicitly declare a copy constructor, one is declared
implicitly. If the class definition declares a move constructor or move assignment
operator, the implicitly declared copy constructor is defined as deleted; otherwise, it is
defined as defaulted (8.4). The latter case is deprecated if the class has a user-declared
copy assignment operator or a user-declared destructor.

所以知道了原因就好办了，直接强行禁止编译器默认生成的拷贝构造函数和赋值构造函数:

``` cpp
struct P {};

class C {
  P *p;
public:
  C(P *p) : p(p) {}
  C(const C&) = delete;
  ~C() { delete p; }
  void operator=(const C&) = delete;
  void f() {}
};

void g(C &c) {
c.f();
}

void h() {
  P *p = new P;
  C c(p);
  g(c);
}
```

以上代码禁止了默认的构造函数并且g()函数的调用参数改成了引用传递，这样就不会二次释放了。

#### 代码样例对比

``` cpp
void f() {
  int *array = new int[10];
  // ...
  delete array;
}
```
以上代码分配和释放方法不匹配对应，所以是错的，会导致未定义行为。

``` cpp
void f() {
  int *array = new int[10];
  // ...
  delete[] array;
}
```

#### 代码样例对比

``` cpp
#include <cstdlib>

void f() {
  int *i = static_cast<int *>(std::malloc(sizeof(int)));
  // ...
  delete i;
}

```
以上的代码也是分配和释放方法不匹配，尽管大多数的编译器产商new实现是malloc分配的内存，delete是free来进行释放内存，尽管在大多数编译器上，这段代码不会发生问题。但是这些是实现，应该尽量不依赖实现编写程序，C++的内存管理不应该依赖底层的C内存管理方法。应该针对同级抽象层级来写代码，不同层级的代码不要混用。

``` cpp
#include <cstdlib>

void f() {
  int *i = static_cast<int *>(std::malloc(sizeof(int)));
  // ...
  std::free(i);
}
```

#### 代码样例对比

``` cpp
#include <cstdlib>

struct S {
  ~S();
};

void f() {
  S *s = new S();
  // ...
  std::free(s);
}
```

以上的代码也是分配和释放方法不匹配,要注意的是，以上代码new的时候除了分配内存空间，还调用了构造函数创建了对象，当释放的时候通过s指针free了内存空间，然而，对象的析构函数没有被调用，会导致未定义行为。这个代码违反了MEM53-CPP的规则。

``` cpp
struct S {
  ~S();
};

void f() {
  S *s = new S();
  // ...
  delete s;
}
```

#### 代码样例对比

``` cpp
#include <cstdlib>
#include <new>

struct S {
  static void *operator new(std::size_t size) noexcept(true) {
    return std::malloc(size);
  }

  static void operator delete(void *ptr) noexcept(true) {
    std::free(ptr);
  }
};

void f() {
  S *s = new S;
  ::delete s;
}
```
以上代码的全局new操作符被与类特定关联的operator new()重载实现了。所以对于此类上调用new的话，实际上调用的是S::operator new()。然而，因为对象的销毁是用范围::delete操作符，所以实际上调用的全局operator delete(),而不是S::operator delete()，所以由于分配和释放的方式不匹配一致，导致出现未定义行为。所以应该改成以下代码:

``` cpp
#include <cstdlib>
#include <new>

struct S {
  static void *operator new(std::size_t size) noexcept(true) {
    return std::malloc(size);
  }

  static void operator delete(void *ptr) noexcept(true) {
    std::free(ptr);
  }
};

void f() {
  S *s = new S;
  delete s;
}
```
以上代码就采用非范围delete代替了范围::delete。所以实质上调用的是匹配的S::operator delete()。是正确的。

#### 代码样例对比

``` cpp
#include <memory>

struct S {};

void f() {
  std::unique_ptr<S> s{new S[10]};
}
```
以上代码显然有问题，经常使用智能指针的开发人员就可以看出来了，std::unique_ptr模版传入的类型有错误，s指针指向的是一个S类型的数组，所以当s对象销毁时，它的析构函数调用的是delete来销毁S类型的数组，而不是delete []，所以本质上是分配和释放方法不匹配导致未定义行为。应该改成以下:

``` cpp
#include <memory>

struct S {};

void f() {
  std::unique_ptr<S[]> s = std::make_unique<S[]>(10); //最好使用工厂函数来初始化指针指针
}

void f() {
  std::unique_ptr<S[]> s(new S[10]); //当然，这样也无所谓，没有任何问题，只是最好当没有显式出现delete操作符的时候最好不要出现new了，这两个操作符之间的出现最好一de致匹配，要不就都没有，这是个审美问题。
}
```

#### 代码样例对比

``` cpp
#include <memory>

struct S {};

void f() {
  std::shared_ptr<S> s{new S[10]};
}
```
以上代码的问题与之前的std::unique_ptr的问题一样，本质是分配和释放的方法不匹配一致，但是，std::shared_ptr模版中不能传递S[]类型，std::make_shared工厂函数也不能传递S[]类型，所以只能用点蹩脚的方式:

``` cpp
#include <memory>

struct S {};

void f() {
  std::shared_ptr<S> s{
    new S[10], [](const S *ptr) { delete [] ptr; }
  };
}
```

就是通过std::shared_ptr所提供的方式自定义std::shared_ptr的释放方式(deleter)。这样就保证对象数组是通过delete[]正确释放了。

为什么std::shared_ptr会与std::unique_ptr不一样的？为什么std::shared_ptr这么不方便的？因为std::shared_ptr所指向的对象的设计初衷就不是指向数组，std::unique_ptr更适合指向数组。

### MEM52-CPP. 检测和处理内存分配错误

严重程度：高。如果没有检测内存分配失败的错误那么可能导致程序非正常终止和拒绝服务攻击。

在C++默认的内存分配操作符中，::operator new(std::size_t)调用分配失败就会抛出std::bad_alloc的异常。因此开发者不需要检测返回的指针是否为nullptr。其有nothrow的形式，::operator new(std::size_t, const std::nothrow_t &)，也就是说该分配失败不会抛出异常，与之代替的是返回nullptr。 operator new[]也是一样的，有无异常和有异常这两种形式。

另外，默认的对象分配器(std::allocator)也直接使用了::operator new(std::size_t)来分配内存所以需要同样小心对待。

``` cpp
T *p1 = new T; // Throws std::bad_alloc if allocation fails
T *p2 = new (std::nothrow) T; // Returns nullptr if allocation fails
T *p3 = new T[1]; // Throws std::bad_alloc if the allocation fails
T *p4 = new (std::nothrow) T[1]; // Returns nullptr if the allocation fails
```

#### 代码样例对比

``` cpp
#include <cstring>

void f(const int *array, std::size_t size) noexcept {
  int *copy = new int[size];
  std::memcpy(copy, array, size * sizeof(*copy));
  // ...
  delete [] copy;
}
```
以上代码使用::operator new[](std::size_t)分配一个int数组，但是没有对分配结果做检查。函数f()又被标记为noexcept,所以调用者会假定这个函数不会抛任何异常，但是 ::operator new[](std::size_t) 一旦分配失败，就会抛出异常，这就导致程序不正常的终止了。所以可以改成nothrow版本:

``` cpp
#include <cstring>
#include <new>

void f(const int *array, std::size_t size) noexcept {
  int *copy = new (std::nothrow) int[size];
  if (!copy) {
    // Handle error
    return;
  }
  
  std::memcpy(copy, array, size * sizeof(*copy));
  // ...
  delete [] copy;
}
```
当然，也可以改成throw版本的，实现try catch来捕获异常:

``` cpp
#include <cstring>
#include <new>

void f(const int *array, std::size_t size) noexcept {
  int *copy;
  try {
    copy = new int[size];
  } catch(std::bad_alloc) {
    // Handle error
    return;
  }
  // At this point, copy has been initialized to allocated memory
  std::memcpy(copy, array, size * sizeof(*copy));
  // ...
  delete [] copy;
}
```
最后，还有个throw版本的，函数签名需要声明noexcept(false):

``` cpp
#include <cstring>

void f(const int *array, std::size_t size) noexcept(false) {
  int *copy = new int[size];
  // If the allocation fails, it will throw an exception which
  // the caller will have to handle.
  std::memcpy(copy, array, size * sizeof(*copy));
  // ...
  delete [] copy;
}
```

#### 代码样例对比

``` cpp
struct A { /* ... */ };
struct B { /* ... */ };

void g(A *, B *);

void f() {
  g(new A, new B);
}
```
以上代码在同一个表达式中执行了2个new操作这显然会发生问题。因为，一旦其中一个new操作发生了异常，另一个new就可能会发生内存泄漏。

考虑一个场景，比如A类被先分配构造，而B在分配的过程中抛出了异常。即使把g()函数的调用用try catch包裹处理，然而对于A的内存释放是不可能的，因为没有保留指向A的指针。另外这个代码与EXP50-CPP规则相违背。所以可以考虑用以下方案:

``` cpp
#include <memory>
struct A { /* ... */ };
struct B { /* ... */ };

void g(std::unique_ptr<A> a, std::unique_ptr<B> b);

void f() {
  g(std::make_unique<A>(), std::make_unique<B>());
}
```
以上代码通过RAII的方式来管理对象A和B。如果再次发生之前描述的场景，B抛出异常了，那么A由于生命周期与对象绑定了，离开了其作用域，所以也能正确调用其对象的析构函数保证内存资源不泄露。

当然，还可以用最土的办法解决:

``` cpp
struct A { /* ... */ };
struct B { /* ... */ };

void g(A &a, B &b);

void f() {
  A a;
  B b;
  g(a, b);
}
```

### MEM53-CPP. 当手动管理对象生命周期的时候一定要显式构造和析构对象

严重程度:高。可能间接导致未定义行为和信息泄漏。

C++动态分配对象的创建一般是分两步，第一步就是，分配一段足够的内存空间来存储对象，第二步就是根据要创建的对象类型来初始化这段已分配的内存。

C++动态销毁已分配的对象也是分两步，第一步就是，根据对象的类型调用对象自身的析构函数释放自身所持有的资源。第二步就是，释放对象本身所占用的那片内存空间。

C++标准[basic.life]有如下声明:

> The lifetime of an object is a runtime property of the object. An object is said to have
non-trivial initialization if it is of a class or aggregate type and it or one of its members is
initialized by a constructor other than a trivial default constructor. [Note: initialization by a
trivial copy/move constructor is non-trivial initialization. — end note] The lifetime of an
object of type T begins when:

> • storage with the proper alignment and size for type T is obtained, and

> • if the object has non-trivial initialization, its initialization is complete.
The lifetime of an object of type T ends when:

> • if T is a class type with a non-trivial destructor, the destructor call starts, or

> • the storage which the object occupies is reused or released.

当手动管理对象生命周期的时候，在对象生命周期开始的阶段构造函数必须被调用，同样，在一个对象生命周期结束的时候，析构函数必须被调用。在对象的生命周期外访问对象就会直接导致未定义行为。

#### 代码样例对比

``` cpp
#include <cstdlib>

struct S {
  S();
  void f();
};

void g() {
  S *s = static_cast<S *>(std::malloc(sizeof(S)));
  s->f();
  std::free(s);
}
```
这段代码一眼看上去貌似没问题，malloc和free都是匹配的，但是仔细看，这时候S是个类，提供了默认的构造函数，g()函数中，没有用new分配符来分配S类的内存并初始化该对象，仅仅只是分配了存放对象的内存空间，但是此刻对象没有被初始化，处于一个未知的状态，然而代码还试图调用对象的成员函数，这直接导致了未定义行为。如果非要使用malloc，可以改成以下方案:

``` cpp
#include <cstdlib>
#include <new>

struct S {
  S();
  void f();
};

void g() {
  void *ptr = std::malloc(sizeof(S));
  S *s = new (ptr) S; //此刻使用的是placement new
  s->f();
  s->~S();       
  std::free(ptr);
}
```
以上代码手动实现new/delete功能的方案。代码是完全没有任何问题的。但是工程实践中，强烈建议不这么做。

#### 代码样例对比

``` cpp
#include <memory>

template <typename T, typename Alloc = std::allocator<T>>
class Container {
  T *underlyingStorage;
  size_t numElements;
  
  void copy_elements(T *from, T *to, size_t count);
public:
  void reserve(size_t count) 
  {
    if (count > numElements) 
    {
      Alloc alloc;
      T *p = alloc.allocate(count); // Throws on failure
      try {
        copy_elements(underlyingStorage, p, numElements);
      } catch (...) {
       alloc.deallocate(p, count);
       throw;
      }
      underlyingStorage = p;
    }
    numElements = count;
  }

  T &operator[](size_t idx) { return underlyingStorage[idx]; }
  const T &operator[](size_t idx) const {
    return underlyingStorage[idx]; 
  }
};
```
以上代码复杂点，首先，代码实现了一个自定义的容器类，然后使用了std::allocator对象获取其元素类型的内存，主要看reserve函数，当copy_elements执行的时候，假定拷贝每个元素到新的已分配的空间的时候会调用元素的拷贝构造函数，然而，问题就发生在这里，没有在新分配的内存位置显示调用每个元素的拷贝构造函数，元素对象状态未知，所以一旦对其访问，立即造成未定义行为。应该改成以下显示调用元素的构造函数:

``` cpp
#include <memory>

template <typename T, typename Alloc = std::allocator<T>>
class Container {
  T *underlyingStorage;
  size_t numElements;
  
  void copy_elements(T *from, T *to, size_t count);
public:
  void reserve(size_t count) 
  {
    if (count > numElements) 
    {
      Alloc alloc;
      T *p = alloc.allocate(count); // Throws on failure
      try {
        copy_elements(underlyingStorage, p, numElements);
        for (size_t i = numElements; i < count; ++i) {
          alloc.construct(&p[i]);
        }
      } catch (...) {
       alloc.deallocate(p, count);
       throw;
      }
      underlyingStorage = p;
    }
    numElements = count;
  }

  T &operator[](size_t idx) { return underlyingStorage[idx]; }
  const T &operator[](size_t idx) const {
    return underlyingStorage[idx]; 
  }
};
```

### MEM54-CPP. 使用placement new的时候需要注意指针在内存空间上的对齐

严重程度:高。容易直接导致未定义行为，包括缓冲区溢出和程序非正常终止。

当我们调用C++默认的全局new表达式时，默认使用的是非placement的形式，它会分配一个足够的内存空间来存储对象，如果分配成功，还会返回一个根据对象类型已对齐的对象指针。然而，如果是仅仅使用placement new的话，返回给调用者的指针就无法保证是已对齐的了，而且也无法保证有足够的内存空间来构造对象，因为内存是外部给定的，无法预知。

C++标准[expr.new]有如下非正式声明:

> [Note: when the allocation function returns a value other than null, it must be a pointer to
a block of storage in which space for the object has been reserved. The block of storage
is assumed to be appropriately aligned and of the requested size. The address of the
created object will not necessarily be the same as that of the block if the object is an
array. —end note]

#### 代码样例对比

``` cpp
#include <new>

void f() {
  short s;
  long *lp = ::new (&s) long;
}
```
以上代码显然是错的，在short类型对象s开辟的内存空间上构造一个long类型的对象，在大多数平台架构上 sizeof(short) < sizeof(long)，显然内存空间是不够的，直接导致未定义行为。

``` cpp
#include <new>

void f() {
  char c; // c只是用来站位的，让代码更好看点
  unsigned char buffer[sizeof(long)];
  long *lp = ::new (buffer) long;
  // ...
}
```

以上代码换了个方式，直接开辟内存大小为long类型大小的连续char内存空间，然后在这段空间上构造long类型对象，看似合理没问题了。当然，内存空间是保证能存放long类型对象了，但是之前还提到过，placement new返回的对象指针并没保证是以long对齐的，所以首先需要强制bufffer的内存空间是以long对齐，placement new所返回的指针才能以long对齐:

``` cpp
#include <new>

void f() {
  char c; // c只是用来站位的，让代码更好看点
  alignas(long) unsigned char buffer[sizeof(long)];
  long *lp = ::new (buffer) long;
  // ...
}
```
以上代码就没有任何问题了，当然，还有以下另一种方式来保证以long类型对齐:

``` cpp
#include <new>

void f() {
  char c; // c只是用来站位的，让代码更好看点
  std::aligned_storage<sizeof(long), alignof(long)>::type buffer;
  long *lp = ::new (&buffer) long;
  // ...
}
```

#### 代码样例对比

``` cpp
#include <new>

struct S {
  S ();
  ~S ();
};

void f() {
  const unsigned N = 32;
  alignas(S) unsigned char buffer[sizeof(S) * N];
  S *sp = ::new (buffer) S[N];
  // ...
  // Destroy elements of the array.
  for (size_t i = 0; i != n; ++i) {
    sp[i].~S();
  }
}
```
经过了之前的代码样例对比，以上的代码就难以看出任何问题了，无论是内存空间大小和指针对齐都考虑了，但是还是有问题。这需要对C++的实现有一定理解，因为在有些编译器实现上可能会少计算了对象数组的overhead(可以简单理解为数组的cookie，一段不属于本身的额外内存)，这个cookie是存放对象数组元素个数的，这个有什么用呢？ 因为在用delete表达式删除对象数组的时候，需要用到对象数组的元素个数，还有就是在exception unwinding这样的机制中需要调用数组中每个成功构造了的元素的析构函数，这些都需要提前做记录，所以会有数组cookie。当然，可以强制避免编译器生成这样的cookie，所以以上代码没有任何手段来避免cookie的产生，为了避免意外情况发生，应该改成以下代码:

``` cpp
#include <cstddef>
#include <new>

#if defined(__clang__) || defined(__GNUG__)
  const size_t overhead = sizeof(size_t);
#else
  static_assert(false, "you need to determine the size of your
implementation's array overhead");
  const size_t overhead = 0;
  // Declaration prevents additional diagnostics
  // about overhead being undefined; the value used
  // does not matter.
#endif

struct S {
  S();
  ~S();
};

void *operator new[](size_t n, void *p, size_t bufsize) {
  if (n <= bufsize) 
  {
    throw std::bad_array_new_length();
  }
  return p;
}

void f() {
  const size_t n = 32;
  alignas(S) unsigned char buffer[sizeof(S) * n + overhead];
  S *sp = new (buffer, sizeof(buffer)) S[n];
  // ...
  // Destroy elements of the array.
  for (size_t i = 0; i != n; ++i) {
    sp[i].~S();
  }
}
```

以上代码只针对了clang和gcc的实现作出了overhead的判断。如果是其他编译器得稍微作出修改了，需要断定其他编译器的数组overhead。所以还是少用placement new吧。

### MEM55-CPP. 自定义动态内存管理函数时需要遵守其替换规范

严重程度:高。如果不满足替换动态内存管理函数的规范，那么自定义的内存管理函数会导致未定义行为。

C++标准是允许开发人员替换全局的动态内存分配和释放函数的。

比如有这样一个需求，开发人员想探测一个应用的动态内存使用率来判断默认的内存分配器是否对这个应用的场景是否合适，能否考虑更好的内存分配策略，C++标准[res.on.function]有如下声明:

> In certain cases (replacement functions, handler functions, operations on types used to
instantiate standard library template components), the C++ standard library depends on
components supplied by a C++ program. If these components do not meet their
requirements, the Standard places no requirements on the implementation.

还有更具体点的:

> In particular, the effects are undefined in the following cases:

> • for replacement functions, if the installed replacement function does not implement
the semantics of the applicable Required behavior: paragraph.

#### 代码样例对比

``` cpp
#include <new>

void *operator new(std::size_t size) {
  extern void *alloc_mem(std::size_t); // Implemented elsewhere;
  // may return nullptr
  return alloc_mem(size);
}

void operator delete(void *ptr) noexcept; // Defined elsewhere
void operator delete(void *ptr, std::size_t) noexcept; // Defined elsewhere
```
以上代码把全局的operator new(std::size_t)函数替换成自定义的实现了，然而，该实现却违反了替换的规范，如果自定义的分配器分配内存的时候失败了，那么该函数是返回了nullptr指针，而不是抛出std::bad_alloc的异常。所以，应该改成下面的版本:

``` cpp
#include <new>

void *operator new(std::size_t size) {
  extern void *alloc_mem(std::size_t); // Implemented elsewhere;
  // may return nullptr
  if (void *ret = alloc_mem(size)) {
    return ret;
  }
  throw std::bad_alloc();
}

void operator delete(void *ptr) noexcept; // Defined elsewhere
void operator delete(void *ptr, std::size_t) noexcept; // Defined elsewhere
```

### MEM56-CPP. 不要把一个已拥有所属权的指针的值保存进一个无关的智能指针中

严重程度：高。可能资源早已被智能指针释放，但是还有用原裸指针访问原来的资源，间接造成未定义行为。

C++中的智能指针比如std::unique_ptr和std::shared_ptr实现了指针所属权的语义，所属权是C++类型系统的一部分。它们包装了指针的值，通过operator *() 和 operator ->()成员函数提供了类似指针的语义。然后，指针所指向的对象的生命周期就与智能指针的对象的生命周期自动关联上了。

调用std::unique_ptr::release()会放弃被管理的指针值的所属权，析构，移动赋值，还有调用std::unique_ptr::reset()都会放弃之前指针的所属权。如果调用std::shared_ptr::unique()返回true，并且std::shared_ptr对象析构或者调用std::shared_ptr::reset()也会释放之前指针的所属权。

#### 代码样例对比

``` cpp
#include <memory>

void f() {
  int *i = new int;
  std::shared_ptr<int> p1(i);
  std::shared_ptr<int> p2(i);
}
```
以上两个不同的std::shared_ptr指针都包装了指针i，所以当f()函数退出的时候，p1和p2就会析构，然后就造成了两次delete i 。 所以应该改成下面:

``` cpp
#include <memory>

void f() {
  std::shared_ptr<int> p1 = std::make_shared<int>();
  std::shared_ptr<int> p2(p1);
}
```
以上代码std::shared_ptr就能维护正确的引用计数了，直到引用计数为0才会释放所指向的资源。

#### 代码样例对比

``` cpp
#include <memory>

struct B {
  virtual ~B() = default; // Polymorphic object
  // ...
};

struct D : B {};

void g(std::shared_ptr<D> derived);

void f() {
  std::shared_ptr<B> poly(new D);
  // ...
  g(std::shared_ptr<D>(dynamic_cast<D *>(poly.get())));
  // Any use of poly will now result in accessing freed memory.
}
```
以上代码poly智能指针所包装的裸指针被转换成了继承类D的指针然后又被多态类型的std::shared_ptr\<D\>所包装，然而，这样最终会导致未定义行为。因为一个同样的指针被保存到两个不同的std::shared_ptr对象中了，当g()函数退出时，std::shared_ptr\<D\>对象通过默认的deleter释放了所指向的资源，然而，等到f()也退出的时候，poly也同样释放了所指资源，同样的资源被释放两次，直接造成未定义行为。因为不同类型的std::shared_ptr不维护同样的引用计数。

所以要转换必须改成以下版本：

``` cpp
#include <memory>

struct B {
  virtual ~B() = default; // Polymorphic object
  // ...
};

struct D : B {};

void g(std::shared_ptr<D> derived);
void f() {
  std::shared_ptr<B> poly(new D);
  // ...
  g(std::dynamic_pointer_cast<D, B>(poly));
  // poly is still referring to a valid pointer value.
}
```
以上代码把dynamic_cast替换成了std::dynamic_pointer_cast()，这个函数就直接返回合法的shared pointer的多态类型，两个指针所持有的引用计数是一样的，直到引用计数为0才会释放，所以不会重复释放。

#### 代码样例对比

``` cpp
#include <memory>

struct S {
  std::shared_ptr<S> g() { return std::shared_ptr<S>(this); }
};

void f() {
  std::shared_ptr<S> s1 = std::make_shared<S>();
  // ...
  std::shared_ptr<S> s2 = s1->g();
}
```
以上代码创建了一个S类型的对象，并把对象指针保存在了对象s1中，然后，又调用了S::g()，把s1指向的裸指针的值又包装到了另一个shared pointer中。然而，s2对象没有与s1关联上，所以没有维持共同的引用计数，导致所指向的资源释放了两次。应该改成下面:

``` cpp
#include <memory>

struct S : std::enable_shared_from_this<S> {
  std::shared_ptr<S> g() { return shared_from_this(); }
};

void f() {
  std::shared_ptr<S> s1 = std::make_shared<S>();
  std::shared_ptr<S> s2 = s1->g();
}
```
以上代码用了std::enable_shared_from_this::shared_from_this()来获取shared pointer，这个获得返回的shared pointer是与之前的s1对象关联上的，所以s1和s2维持了共同的引用计数，所以不会导致重复释放了。

### MEM57-CPP. 避免使用默认的operator new来生成超出对齐大小的类型

严重程度: 中等。使用对齐有问题的指针会导致未定义行为，一般现象是直接导致程序不正常终止。

CERT有以下描述，这部分没有翻译，为了准确性:

> The non-placement new expression is specified to invoke an allocation function to allocate
storage for an object of the specified type. When successful, the allocation function, in turn, is
required to return a pointer to storage with alignment suitable for any object with a fundamental
alignment requirement. Although the global operator new, the default allocation function
invoked by the new expression, is specified by the C++ standard [ISO/IEC 14882-2014] to
allocate sufficient storage suitably aligned to represent any object of the specified size, since the
expected alignment isn’t part of the function’s interface, the most a program can safely assume is
that the storage allocated by the default operator new defined by the implementation is
aligned for an object with a fundamental alignment. In particular, it is unsafe to use the storage for
an object of a type with a stricter alignment requirement—an over-aligned type.

> Furthermore, the array form of the non-placement new expression may increase the amount of
storage it attempts to obtain by invoking the corresponding allocation function by an unspecified
amount. This amount, referred to as overhead in the C++ standard, is commonly known as a
cookie. The cookie is used to store the number of elements in the array so that the array delete
expression or the exception unwinding mechanism can invoke the type’s destructor on each
successfully constructed element of the array. While the specific conditions under which the
cookie is required by the array new expression aren’t described in the C++ standard, they may be
outlined in other specifications such as the application binary interface (ABI) document for the
target environment. For example, the Itanium C++ ABI describes the rules for computing the size
of a cookie, its location, and achieving the correct alignment of the array elements. When these
rules require that a cookie be created, it is possible to obtain a suitably aligned array of elements
of an overaligned type [CodeSourcery 2016a]. However, the rules are complex and the Itanium
C++ ABI isn’t universally applicable.

#### 代码样例对比

``` cpp
struct alignas(32) Vector {
  char elems[32];
};

Vector *f() {
  Vector *pv = new Vector;
  return pv;
}
```
以上代码使用了默认的new表达式来分配并构造自定义的类型Vector，但是这个对齐超出了基本对齐的最大实现(一般是16字节)。一般来说当传入对齐不正常的参数就会进入陷阱(trap)的SIMD向量化指令是需要这样的超出对齐大小的类型的。应该改成以下:

``` cpp
#include <cstdlib>
#include <new>

struct alignas(32) Vector {
  char elems[32];
  static void *operator new(size_t nbytes) {
    if (void *p = std::aligned_alloc(alignof(Vector), nbytes)) {
      return p;
    }
    throw std::bad_alloc();
  }

  static void operator delete(void *p) {
    free(p);
  }
};

Vector *f() {
  Vector *pv = new Vector;
  return pv;
}
```
以上代码采用了重载的operator new并通过C11的函数aligned_alloc()获取合法的对齐内存。aligned_alloc()函数不属于C++ 98, C++ 11或 C++ 14 标准的任何一部分，但是这些标准都会把它作为一个扩展通过标准库提供出来。
