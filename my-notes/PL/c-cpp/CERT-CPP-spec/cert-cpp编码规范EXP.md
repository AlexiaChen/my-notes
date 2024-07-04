
## 表达式（EXP）

### EXP50-CPP 不要依赖求值顺序所造成的副作用

严重等级：中等

关于求值顺序可能造成的副作用，C++标准[intro.execution]有说明：

> Sequenced before is an asymmetric, transitive, pair-wise relation between evaluations executed by a single thread, which induces a partial order among those evaluations. Given any two evaluations A and B, if A is sequenced before B, then the execution of A shall precede the execution of B. If A is not sequenced before B and B is not sequenced before A, then A and B are unsequenced. [Note: The execution of unsequenced evaluations can overlap. — end note] Evaluations A and B are indeterminately sequenced when either A is sequenced before B or B is sequenced before A, but it is unspecified which. [Note: Indeterminately sequenced evaluations cannot overlap, but
either could be executed first. — end note]

当然，接下来还有描述：

> Except where noted, evaluations of operands of individual operators and of subexpressions of individual expressions are unsequenced. ... The value computations of the operands of an operator are sequenced before the value computation of the result of the operator. If a side effect on a scalar object is unsequenced relative to either another side effect on the same scalar object or a value computation using the value of the same scalar object, and they are not potentially concurrent, the behavior is undefined. ... When calling a function (whether or not the function is inline), every value computation and side effect associated with any argument expression, or with the postfix expression designating the called function, is sequenced before execution of every expression or statement in the body of the called function. ... Every evaluation in the calling function (including other function calls) that is not otherwise specifically sequenced before or after the execution of the body of the called function is indeterminately sequenced with respect to the execution of the called function. Several contexts in C++ cause evaluation of a function call, even though no corresponding function call syntax appears in the translation unit. ... The sequencing constraints on the execution of the called function (as described above) are features of the function calls as evaluated, whatever the syntax of the expression that calls the function might be.

#### 代码样例对比

``` cpp
void f(int i, const int *b) {
  int a = i + b[++i];
  // ...
}
```

上面这段代码是不符合规范的，i的求值会以不定的顺序求值多次，所以表达式的行为是未定义的。比如对于函数f(5,&b)的调用。表达式i + b[++i]可能有以下二种求值顺序：

-  5 + b[++i] -> 5 + b[6] 
-  i + b[6] -> 6 + b[6]

这就导致了我们没想到的行为结果。应该避免，如下：

``` cpp

void f(int i, const int *b) {
  ++i;
  int a = i + b[i];
  // ...
}

void f(int i, const int *b) {
  int a = i + b[i + 1];
  ++i;
  // ...
}
```

#### 代码样例对比

``` cpp

extern void func(int i, int j);

void f(int i) {
  func(i++, i);
}

```

以上的代码，对于函数func的调用可能行为不一致，因为函数func的输入的参数表达式的求值顺序是不定的。对于f(5)的调用，func的参数求值有以下顺序：

- func(5,i) -> func(5,6)
- func(i++,5) -> func(5,5)

必须用以下方式来避免：

``` cpp
extern void func(int i, int j);

void f(int i) {
  i++;
  func(i, i);
}

extern void func(int i, int j);

void f(int i) {
  int j = i++;
  func(j, i);
}
```
这样以来，函数f传入任何参数，都会有一一对应的输出。不会出现副作用。

#### 代码样例对比

``` cpp
extern void c(int i, int j);
int glob = 5;

int a() {
  return glob + 10;
}

int b() {
  glob = 42;
  return glob;
}

void f() {
  c(a(), b());
}
```

上面这段代码，对函数c的调用，依赖于参数的求值顺序，也就是说，a(),b()的调用顺序是不定的。会导致未指定行为(unspecified bhavior)，不是未定义行为(undefinded behavior)。

- c(15, b()) -> c(15,42)
- c(a(),42) -> c(52,42)

需要固定函数的调用顺序，就可以了：

``` cpp
extern void c(int i, int j);
int glob = 5;

int a() {
  return glob + 10;
}

int b() {
  glob = 42;
  return glob;
}

void f() {
  int a_val = a();
  int b_val = b();
  c(a_val, b_val);
}
```

### EXP51-CPP 不要通过不正确的类型的指针来删除一个数组

严重等级：低

在C++标准中[expr.delete] 有如下声明：

> In the first alternative (delete object), if the static type of the object to be deleted is different from its dynamic type, the static type shall be a base class of the dynamic type of the object to be deleted and the static type shall have a virtual destructor or the behavior is undefined. In the second alternative (delete array) if the dynamic type of the object to be deleted differs from its static type, the behavior is undefined.

不要通过静态指针类型删除一个动态指针类型的对象数组，这样会导致未定义行为。至于后果，可能就是不正常的程序和内存泄漏。

#### 代码样例对比

``` cpp
struct Base {
  virtual ~Base() = default;
};

struct Derived final : Base {};

void f() {
  Base *b = new Derived[10];
  // ...
  delete [] b;
}

```

以上代码创建了一个Derived类型的对象数组，这个数组指针被保存在Base类型的指针中。尽管Base的析构函数被声明为virtual，但是该段代码还是会导致未定义行为。另外，企图在静态类型的Base指针上执行指针算数操作，违反了CTR56-CPP的条款。

应该改成以下代码：

``` cpp

struct Base {
  virtual ~Base() = default;
};

struct Derived final : Base {};

void f() {
  Derived *b = new Derived[10];
  // ...
  delete [] b;
}

```

以上的代码就移除了未定义行为的发生的可能。Derived对象数组的删除，就应该交由Drived类型的指针来删除。

### EXP53-CPP 不要读取未初始化的内存

严重等级： 高

局部变量，自动变量在初始化之前读取的话，它们的值都是无法预测的（随机值）。C++标准[dcl.init]有如下声明：

> If no initializer is specified for an object, the object is default-initialized. When storage for an object with automatic or dynamic storage duration is obtained, the object has an indeterminate value, and if no initialization is performed for the object, that object retains an indeterminate value until that value is replaced. If an indeterminate value is produced by an evaluation, the behavior is undefined except in the following cases:

> • If an indeterminate value of unsigned narrow character type is produced by the evaluation of:

> — the second or third operand of a conditional expression,

> — the right operand of a comma expression,

> — the operand of a cast or conversion to an unsigned narrow character type, or

> — a discarded-value expression, then the result of the operation is an indeterminate value.

> • If an indeterminate value of unsigned narrow character type is produced by the evaluation of the right operand of a simple assignment operator whose first operand is an lvalue of unsigned narrow character type, an indeterminate value replaces the value of the object referred to by the left operand.

> • If an indeterminate value of unsigned narrow character type is produced by the evaluation of the initialization expression when initializing an object of unsigned narrow character type, that object is initialized to an indeterminate value.

一个对象的默认初始化在C++标准中也有子条款描述：

> To default-initialize an object of type T means:

> • if T is a (possibly cv-qualified) class type, the default constructor for T is called (and the initialization is ill-formed if T has no default constructor or overload resolution results in an ambiguity or in a function that is deleted or inaccessible from the context of the initialization);

> • if T is an array type, each element is default-initialized;

> • otherwise, no initialization is performed.If a program calls for the default initialization of an object of a const-qualified type T, Tshall be a class type with a user-provided default constructor.

#### 代码样例对比

``` cpp
#include <iostream>

void f() {
  int i;
  std::cout << i; //导致未定义行为
}
```
``` cpp
#include <iostream>

void f() {
  int i = 0; // 打印前，先初始化
  std::cout << i; 
}
```

``` cpp
#include <iostream>

void f() {
  int *i = new int;
  std::cout << i << ", " << *i; //打印指针指向的值会导致未定义行为，打印指针本身的值是合法的
}
```

``` cpp
#include <iostream>

void f() {
  int *i = new int(12);
  std::cout << i << ", " << *i;//完全没问题了
}
```

关于new表达式初始化的方法，有以下正确姿势：

``` cpp
int *i = new int(); // zero-initializes *i
int *j = new int{}; // zero-initializes *j
int *k = new int(12); // initializes *k to 12
int *l = new int{12}; // initializes *l to 12
```

#### 代码样例对比

``` cpp
class S {
  int c;
public:
  int f(int i) const { return i + c; }
};

void f() {
  S s;
  int i = s.f(10);
}
```
以上代码中，虽然对象s被默认构造函数初始化了，成员变量c的空间也被分配了，但是变量c没有被初始化为一个值，所以，对成员函数f的调用会导致未定义行为。需要这么写：

``` cpp
class S {
  int c;
public:
  S() : c(0) {} //给个默认构造函数，把c初始化为0
  int f(int i) const { return i + c; }
};

void f() {
  S s;
  int i = s.f(10);
}
```

### EXP54-CPP 不要访问处于其生命周期之外的对象

严重程度：高

C++中每个对象都有自己的生命周期，C++程序员掌握对象的生命周期是必须要做的事。它是保证一个程序是well-defined的必要途径。如果用指针访问了超出了其声明周期的对象，那么就会导致未定义行为。

在C++标准[basic.life]中，描述指针的生命周期是这样的：

> Before the lifetime of an object has started but after the storage which the object will occupy has been allocated or, after the lifetime of an object has ended and before the storage which the object occupied is reused or released, any pointer that refers to the storage location where the object will be or was located may be used but only in limited ways. For an object under construction or destruction, see 12.7. Otherwise, such a pointer refers to allocated storage, and using the pointer as if the pointer were of type void*, is well-defined. Indirection through such a pointer is permitted but the resulting lvalue may only be used in limited ways, as described below. The program has undefined behavior if:

> • the object will be or was of a class type with a non-trivial destructor and the pointer is used as the operand of a delete-expression,

> • the pointer is used to access a non-static data member or call a non-static member
function of the object, or

> • the pointer is implicitly converted to a pointer to a virtual base class, or

> • the pointer is used as the operand of a static_cast, except when the conversion is to pointer to cv void, or to pointer to cv void and subsequently to pointer to either cv char or cv unsigned char, or

> • the pointer is used as the operand of a dynamic_cast.

以下是对非指针对象生命周期的描述：

> Similarly, before the lifetime of an object has started but after the storage which the object will occupy has been allocated or, after the lifetime of an object has ended and before the storage which the object occupied is reused or released, any glvalue that refers to the original object may be used but only in limited ways. For an object under construction or destruction, see 12.7. Otherwise, such a glvalue refers to allocated storage, and using the properties of the glvalue that do not depend on its value is welldefined.

> The program has undefined behavior if:

> • an lvalue-to-rvalue conversion is applied to such a glvalue,

> • the glvalue is used to access a non-static data member or call a non-static member function of the object, or

> • the glvalue is bound to a reference to a virtual base class, or

> • the glvalue is used as the operand of a dynamic_cast or as the operand of typeid.

值得注意的一点就是，一个对象的生命周期是从一个对象内存空间的分配完成并且初始化也完成的时候，才算正式开始。

#### 代码样例对比

``` cpp
struct S {
  void mem_fn();
};

void f() {
  S *s;
  s->mem_fn();
}
```
上面的代码S类型的对象并没有初始化，S类型指针s倒是生命周期已经开始，但是并不是对象本身，所以调用对象的mem_fn函数，就会导致未定义行为。要改成以下片段：

``` cpp
struct S {
  void mem_fn();
};

void f() {
  S *s = new S;
  s->mem_fn();
  delete s;
}
```
当然，如果以上的代码片段，如果你的编译器允许智能指针会更好。这个情景下建议使用std::unique_ptr。

#### 代码样例对比

``` cpp
struct B {};
struct D1 : virtual B {};
struct D2 : virtual B {};
struct S : D1, D2 {};

void f(const B *b) {}

void g() {
  S *s = new S;
  // Use s
  delete s;
  f(s);
}
```

上面这段代码，注意到了没？ 在调用函数f的时候，类型S的指针被隐式转换成了虚基类B的指针，并且s所指对象的生命周期结束了，被delete了。尽管在函数f的实现中没有使用该参数，但是这还是会导致未定义行为。上面的代码应该改为以下:

``` cpp
struct B {};
struct D1 : virtual B {};
struct D2 : virtual B {};
struct S : D1, D2 {};

void f(const B *b) {}

void g() {
  S *s = new S;
  // Use s
  f(s);
  delete s;
}
```
把使用s变量提前，要保证s的使用在生命周期内。

#### 代码样例对比

``` cpp
#include <initializer_list>
#include <iostream>

class C {
  std::initializer_list<int> l;
public:
  C() : l{1, 2, 3} {}
  int first() const { return *l.begin(); }
};

void f() {
  C c;
  std::cout << c.first();
}
```

上面这段代码咋看好像还没什么问题，这比较复杂，需要仔细推敲。首先需要解释下原理，由一个初始化列表构造一个std::initializer_list的对象从实现上看是需要分配一个临时数组，然后再把这个临时数组传递给std::initializer_list类对象的构造函数，让你意外的是，std::initializer_list成员对象l所持有的是这个临时数组的引用，并没有拷贝给它。而临时数组的生命周期在构造函数结束后就同样结束了，所以，以上代码，无论如何，一旦对成员l的任何元素访问，就会造成未定义行为。

``` cpp
#include <iostream>
#include <vector>

class C {
  std::vector<int> l;
public:
  C() : l{1, 2, 3} {}
  int first() const { return *l.begin(); }
};

void f() {
  C c;
  std::cout << c.first();
}
```

以上代码就规避了未定义行为。仅仅把std::initialzier_list换成了std::vector。std::vector就是把初始化列表的成员拷贝到自己的容器中了，而不是std::initialzier_list那样仅仅是依赖一个临时数组的引用委托。

#### 代码样例对比

``` cpp
#include <functional>

void f() {
  auto l = [](const int &j) { return j; };
  std::function<const int&(const int &)> fn(l);
  int i = 42;
  int j = fn(i);
}
```

以上这段代码引入了C++11的lambda表达式，一下子看上去也没什么问题，所以分析起来复杂点。
首先，代码中的一个lambda对象被存储到了一个函数对象中，待之后调用，获取一个值的常引用。lambda对象返回了一个int类型的值，这个值一般是会存储在一个临时int对象中的，然而这个临时对象的类型却被std::function函数对象的返回值const int&类型所绑定了，然后这个存储返回值的临时int对象的生命周期随着fn的调用在意料之外没有延长，反而随着fn的调用完成结束了生命周期，而std::function函数类型所指定的返回值却被常引用所绑定，一旦对其进行访问，那么就会导致未定义行为。

所以知道原理后，很简单就可以把代码改正确：

``` cpp
#include <functional>

void f() {
  auto l = [](const int &j) { return j; };
  std::function<int(const int &)> fn(l);
  int i = 42;
  int j = fn(i);
}
```
std::function函数类型所指定的返回值类型不能用引用了，改成普通的int类型即可。

#### 代码样例对比

下面一个例子就是滥用goto语句导致的问题，所以还是慎用。

``` cpp
class S {
  int v;
public:
  S() : v(12) {} // Non-trivial constructor
  void f();
};

void f() {
  // ...
  goto bad_idea;
  // ...
  S s; // Control passes over the declaration, so initialization does not take place.
bad_idea:
  s.f();
}
```

没什么难的，上面的代码不小心用goto语句jump到了bad_idea的标签位置，但是标签后恰恰紧接着调用了s对象的成员函数，而对象s的默认初始化构造被跳过去了，所以对s的引用属于生命周期以外，导致未定义行为。

很简单就可以改成下面:

``` cpp
class S {
  int v;
public:
  S() : v(12) {} // Non-trivial constructor
  void f();
};

void f() {
  S s;
  // ...
  goto bad_idea;
  // ...
bad_idea:
  s.f();
}
```

把s的初始化构造提前到goto执行之前。

#### 代码样例对比

``` cpp
#include <algorithm>
#include <cstddef>
#include <memory>
#include <type_traits>

class S {
  int i;
public:
  S() : i(0) {}
  S(int i) : i(i) {}
  S(const S&) = default;
  S& operator=(const S&) = default;
};

template <typename Iter>
void f(Iter i, Iter e) {

  static_assert(std::is_same<typename std::iterator_traits<Iter>::value_type, S>::value,
          "Expecting iterators over type S");

  ptrdiff_t count = std::distance(i, e);
  if (!count) {
    return;
  }

  // Get some temporary memory.
  auto p = std::get_temporary_buffer<S>(count);
  if (p.second < count) {
    // Handle error; memory wasn't allocated, or
    // insufficient memory was allocated.
    return;
  }

  S *vals = p.first;
  // Copy the values into the memory.
  std::copy(i, e, vals);
  // ...
  // Return the temporary memory.
  std::return_temporary_buffer(vals);
}
```

上面这段代码实现有点复杂，不管，先从函数f()的实现来看，f函数的参数是一个类型为S的迭代范围对象。然后这可迭代的一组对象被std::copy到一个temporary_buffer中。然而，std::get_temporary_buffer返回的S对象数组中的对象并没有初始化,所以std::copy函数解引用了目标迭代器，所以导致了未定义行为。归而结网的原因就是，对象空间被分配了，但是对象的构造函数和初始化并未调用。

std::get_temporary_buffer和std::copy的大致实现类似于下面的代码片段：

``` cpp
unsigned char *buffer = new (std::nothrow) unsigned char[sizeof(S) * object_count];
S *result = reinterpret_cast<S *>(buffer);

while (i != e) {
  *result = *i; // Undefined behavior
  ++result;
  ++i;
}
```

所以result所指向的类型为S的对象的生命周期还没有开始。需要做以下改动，仅仅改动std::copy那里就可以了：

``` cpp
//...
// Copy the values into the memory.
std::uninitialized_copy(i, e, vals);
// ...
```
这个std::uninitialized_copy保证了对象初始化的时候用placement new来代替解引用为初始化的内存。或者还可以改成以下代码：

``` cpp
//...
// Copy the values into the memory.
std::copy(i, e, std::raw_storage_iterator<S*, S>(vals));
// ...
```
用std::raw_storage_iterator作为目标地址迭代器与使用std::uninitialized_copy的效果是一样的，都是well-defined的。


### EXP55-CPP 不要通过cv-unqualified的类型来访问一个cv-qualified类型的对象

严重程度：中

C++标准对其 [dcl.type.cv]进行了如下描述：

> Except that any class member declared mutable can be modified, any attempt to modify a const object during its lifetime results in undefined behavior.

类似的还有：

> What constitutes an access to an object that has volatile-qualified type is implementation-defined. If an attempt is made to refer to an object defined with a volatile-qualified type through the use of a glvalue with a non-volatile-qualified type,the program behavior is undefined.

不要试图通过去除const限制符来修改对象，const限制符意味着API的设计者不想让被修饰的对象被修改尽管对象可以通过其他非常规手段修改。不要试图去除volatile限制符，该限制符意味着API的设计者想让编译器以未知的方式来访问该对象，其他任何访问volatile对象会导致未定义行为。

导致的结果是非正常程序终止或拒绝服务攻击(Denial of service attack)。

#### 代码样例对比

``` cpp
void g(const int &ci) {
  int &ir = const_cast<int &>(ci);
  ir = 42;
}

void f() {
  const int i = 4;
  g(i);
}
```

以上的代码很明显会导致未定义行为，把const属性去除了，再来修改。想要修改，就必须手动把const去除，如下:

``` cpp
void g(int &i) {
  i = 42;
}

void f() {
  int i = 4;
  g(i);
}
```

#### 代码样例对比

``` cpp
#include <iostream>

class S {
  int cachedValue;
  int compute_value() const; // expensive
public:
  S() : cachedValue(0) {}
  // ...
  int get_value() const {
    if (!cachedValue) {
      const_cast<S *>(this)->cachedValue = compute_value();
    }
    return cachedValue;
  }
};

void f() {
  const S s;
  std::cout << s.get_value() << std::endl;
}
```
以上这段代码，首先看函数f(),类似S的对象s是用const修饰的，不可修改其状态，但是其成员函数get_value却通过const_cast this指针把const属性给去掉来修改cachedValue，这样就会导致未定义行为。

如果还是希望修改const修饰的s对象，以下有办法对这种情况做出妥协，不会导致未定义行为，但是这样的做法少有人做：

``` cpp
#include <iostream>

class S {
  mutable int cachedValue;
  int compute_value() const; // expensive
public:
  S() : cachedValue(0) {}
  // ...
  int get_value() const {
    if (!cachedValue) {
      cachedValue = compute_value();
    }
    return cachedValue;
  }
};

void f() {
  const S s;
  std::cout << s.get_value() << std::endl;
}
```

就是仅仅用mutable修饰cachedValue。

#### 代码样例对比

``` cpp
#include <iostream>

struct S {
  int i;
  S(int i) : i(i) {}
};

void g(S &s) {
  std::cout << s.i << std::endl;
}

void f() {
  volatile S s(12);
  g(const_cast<S &>(s));
}
```

以上代码也是通过去除volatile属性来访问对象s，然后通过函数g()来试图读取s，这直接会导致未定义行为。要做以下修改：

``` cpp
#include <iostream>

struct S {
  int i;
  S(int i) : i(i) {}
};

void g(volatile S &s) {
  std::cout << s.i << std::endl;
}

void f() {
  volatile S s(12);
  g(s);
}
```

### EXP56-CPP. 不要调用从语言链接上不匹配的函数

严重程度：低，在C和C++之间不匹配的语言链接一般不会造成安全漏洞，但是C++和其他除C语言之外的不匹配语言链接会大概率导致非正常程序终止，包括安全漏洞。

C++允许使用语言链接规范来与其他语言互交通信。（也就是C++不通过网络方式可以调用其他语言的接口，或者其他语言可以调用C++的接口）。然而，这个语言规范定义了一套不同语言函数之间的调用方式和数据访问方式。

默认，所有的函数类型就是函数名和参数名，其外部链接都具有C++语言链接。C++标准对语言链接做出了以下描述:

>  Two function types with different language linkages are distinct types even if they are otherwise identical.

当跨语言调用一个不匹配的函数类型的时候，就会导致未定义行为。当语言链接不匹配时，就会破坏调用栈，就是因为调用约定或其他ABI不匹配造成了。

当然，许多编译器厂商没有实现不同语言链接的规范，尽管C++标准要求这么做。比如，在GCC 6.1.0 Clang 3.9和VS2015中都一致认为以下的代码是ill-formed的，因为f()的重定义优先级要比well-formed的f()的重载要高。

``` cpp
typedef void (*cpp_func)(void);
extern "C" typedef void (*c_func)(void);
void f(cpp_func fp) {}
void f(c_func fp) {}
```

虽然C++的编译器一般都遵守C++标准，但是在严格的遵守条件下，各样的C++编译器在实现C++标准上都会有分歧，也是因为实践取舍的考量。一些编译器仅仅支持了C和C++的语言链接。

请看下面的代码:

``` cpp
#include <cstdlib>

void f(int *int_list, size_t count) {
  std::qsort(int_list, count, sizeof(int),[](const void *lhs, const void *rhs) -> int {
          return reinterpret_cast<const int *>(lhs) < reinterpret_cast<const int *>(rhs);
      });
}
```

以上的这段代码在调用qsort的函数的时候没匹配语言链接，但是实际上在VS2015 for x86上是允许的，尽管lambda函数调用操作符被隐式转换成了一个带有C++语言链接的一个函数指针，但是qsort函数其实是期望的是一个C语言链接的函数指针。

#### 代码样例对比

``` cpp
extern "java" typedef void (*java_callback)(int);
extern void call_java_fn_ptr(java_callback callback);
void callback_func(int);

void f() {
  call_java_fn_ptr(callback_func);
}
```

分析以上代码，函数call_java_fn_ptr期望接收一个java语言链接的一个函数指针，因为该函数指针可能会被Java解释器使用，回调进C++的函数中。然而，在实际的调用中，却给出了一个带有C++链接的回调函数callback_func。这就导致了jAVA解释器在调用这个函数指针的时候，出现了未定义行为。可以改成以下代码：

``` cpp
extern "java" typedef void (*java_callback)(int);
extern void call_java_fn_ptr(java_callback callback);
extern "java" void callback_func(int);

void f() {
  call_java_fn_ptr(callback_func);
}
```

### EXP57-CPP. 不要删除或转换指向不完整类的指针

严重等级：中等。转换或引用不完整类会导致错误的内存地址，通过指针删除不完整类（如果该类拥有不正确的析构函数）会导致未定义行为，程序终止，资源泄漏都可能发生。

C++标准[expr.delete]有如下描述：

> If the object being deleted has incomplete class type at the point of deletion and the complete class has a non-trivial destructor or a deallocation function, the behavior is undefined.

#### 代码样例对比

``` cpp
class Handle {
  class Body *impl; // Declaration of a pointer to an incomplete class
public:
  ~Handle() { delete impl; } // Deletion of pointer to an incomplete class
  // ...
};
```

以上的代码用了pimpl的模式实现了Handle类，但是却在其析构函数里面企图删除一个指向不完整类的指针，如果Body类没有正确的析构函数，那么就会导致未定义行为。应该改成下面:

``` cpp
class Handle {
  class Body *impl;
  // Declaration of a pointer to an incomplete class
public:
  ~Handle();
  // ...
};

// Elsewhere
class Body { /* ... */ };

Handle::~Handle() {
  delete impl;
}
```

Body一定要有实现。

``` cpp
#include <memory>

class Handle {
  std::shared_ptr<class Body> impl;
public:
  Handle();
  ~Handle() {}
  // ...
};
```

或者可以用以上的shared_ptr智能指针解决。它有能力指向不完整的类（Body类没有实现），而且不会出问题。但是std::unique_ptr就不能。

#### 代码样例对比

``` cpp
// File1.h
class B {
protected:
  double d;
public:
  B() : d(1.0) {}
};

// File2.h
void g(class D *);
class B *get_d(); // Returns a pointer to a D object

// File1.cpp
#include "File1.h"
#include "File2.h"

void f() {
  B *v = get_d();
  g(reinterpret_cast<class D *>(v));
}

// File2.cpp
#include "File2.h"
#include "File1.h"
#include <iostream>

class Hah {
protected:
  short s;
public:
  Hah() : s(12) {}
};

class D : public Hah, public B {
  float f;
public:
  D() : Hah(), B(), f(1.2f) {}
  void do_something() {
    std::cout << "f: " << f << ", d: " << d << ", s: " << s << std::endl;
  }
};

void g(D *d) {
  d->do_something();
}

B *get_d() {
  return new D;
}
```

指针向下转换（把一个指向基类的指针转换为继承类的指针）可能需要调整一定量的指针指向的地址并且只有当类继承的结构的布局是已知的时候才能被决定。在以上代码中，函数f()通过get_d()得到了完整类型B的多态指针，然后这个指针在传入g()之前被转换成了不完整类型D的指针。这个转换有可能失败，然后对d->do_something()的调用就会导致未定义行为。

所以C++提供了dynamic_cast，就是运行时转换，这个转换只是尝试，一旦转换失败，那么返回的指针为nullptr。所以可以改成以下代码：

``` cpp
// File1.h -- contents identical.
// File2.h
void g(class B *); // Accepts a B object, expects a D object
class B *get_d(); // Returns a pointer to a D object
// File1.cpp
#include "File1.h"
#include "File2.h"

void f() {
  B *v = get_d();
  g(v);
}

// File2.cpp
// ... all contents are identical until ...
void g(B *d) {
  D *t = dynamic_cast<D *>(d);
  if (t) {
    t->do_something();
  } else {
    // Handle error
  }
}

B *get_d() {
  return new D;
}
```

### EXP59-CPP. offsetof函数使用在合法的类型和成员上

严重等级：中等。不合法的类型或成员传入offsetof中会导致未定义行为，可能会因为数据完整性的破坏而执行非法代码。

offsetof宏是由C标准定义的可移植的方法，来判断给定的成员相对于对象起始位置的偏移（字节）。C语言标准有如下描述：

> offsetof(type, member-designator) which expands to an integer constant expression that has type size_t, the value of which is the offset in bytes, to the structure member (designated by member-designator), from the beginning of its structure (designated by type). The type and member designator shall be such that given static type t; then the expression &(t.member-designator) evaluates to an address constant. (If the specified member is a bit-field, the behavior is undefined.)

C++标准[support.types]也有如下增添的描述：

> The macro offsetof(type, member-designator) accepts a restricted set of type arguments in this International Standard. If type is not a standard-layout class, the results are undefined. The expression offsetof(type, member-designator) is never type-dependent and it is value-dependent if and only if type is dependent. The result of applying the offsetof macro to a field that is a static data member or a function member is undefined. No operation invoked by the offsetof macro shall throw an exception and noexcept(offsetof(type, member-designator)) shall be true.

#### 代码样例对比

``` cpp
#include <cstddef>

struct D {
  virtual void f() {}
  int i;
};

void f() {
  size_t off = offsetof(D, i);
  // ...
}
```

在以上代码中，类型D不是标准布局的类，所以传入offsetof宏中，会导致未定义行为。以上代码在VS2015 for x86上用/Wall警告全开的方式编译，不会产生任何警告。off的运行结果在VS2015上是4，是因为类型D有虚函数表指针的存在。

``` cpp
#include <cstddef>

struct D {
  virtual void f() {}
  struct InnerStandardLayout {
    int i;
  } inner;
};

void f() {
  size_t off = offsetof(D::InnerStandardLayout, i);
  // ...
}
```
以上就改了下代码，因为D类型不是标准布局的类，所以类型D中成员的i是不可能正确求出它的偏移的。然而，却可以在D中重新制造一个标准布局的类。

#### 代码样例对比

``` cpp
#include <cstddef>

struct S {
  static int i;
  // ...
};

int S::i = 0;

extern void store_in_some_buffer(void *buffer, size_t offset, int val);
extern void *buffer;

void f() {
  size_t off = offsetof(S, i);
  store_in_some_buffer(buffer, off, 42);
}
```

以上代码也会导致未定义行为，因为i是类S的静态成员，因为C++标准中提到，静态数据成员不是类实例化对象的一部分。同样上面的代码在VS2015 x86下/Wall编译不会产生任何警告。所以是非常危险的。

``` cpp
#include <cstddef>

struct S {
  static int i;
  // ...
};

int S::i = 0;
extern void store_in_some_buffer(void *buffer, size_t offset, int val);

void f() {
  store_in_some_buffer(&S::i, 0, 42);
}
```
可以直接对i进行取地址，然后在上面存取数据。

### EXP60-CPP. 不要跨执行边界传递非标准布局类型的对象

严重等级：高。影响非常广，要么正确，要么良好运行，要不导致未定义行为。

标准布局的类型一般被用来和其他编程语言通信，该类型的布局被严格定义，C++标准中[class]定义了一个标准布局的类需要满足以下条件。

- 没有虚函数
- 对于所有的非静态数据成员要有相同的访问权限
- 与第一个非静态数据成员相同类型的类不能有基类
- 类中不能有非标准布局类型的非静态数据成员
- 在类的继承层级中只要有一个有非静态数据成员的类，那么它所有的子类包括它就是非标准布局类

执行边界的定义就是不同的的编译器（包括不同版本的编译器）编译出来的代码二进制模块的边界。比如，一个函数声明在一个头文件中，但是函数定义是在库中，运行时才被载入。执行边界就是调用方的可执行文件调用库的二进制文件的边界。这样的边界有时候也被成为ABI边界，因为这跟二进制模块的互交有关。

不要对非标准布局的类型的对象布局作出任何特定假设。比如相同类型对象被不同编译器所编译的布局不同但是它们之间互相引用，这就导致了程序的移植和正确性问题。

如果真需要跨二进制边界传递非标准布局的类型对象，需要严格依赖两个通信模块相同的ABI。如果是不同编译器编译的两个模块，也需要严格依赖二者的ABI是兼容的。

#### 代码样例对比

``` cpp
// library.h
struct S {
  virtual void f() { /* ... */ }
};

void func(S &s); // Implemented by the library, calls S::f()

//library.cpp
void func(S &s)
{
    //... any implementations
}

// finally the library.cpp complied to the library.lib or library.so or library.dll 

// application.cpp finally complied to the application.exe or application.elf
#include "library.h"

void g() {
  S s;
  func(s);
}

//application.exe calls library.dll
```
以上代码根据之前的定义描述，就知道是有问题的代码。如果是不同编译器编译的这两个模块很容易导致预测不到的行为，软件的正常运行严格依赖不同编译器之间的二进制兼容。为了避免不必要的麻烦，应该提早检测，所以改成如下代码：

``` cpp
// library.h
#include <type_traits>

struct S {
  void f() { /* ... */ } // No longer virtual
};

static_assert(std::is_standard_layout<S>::value, "S is required to be a standard layout type");

void func(S &s); // Implemented by the library, calls S::f()

// application.cpp
#include "library.h"

void g() {
  S s;
  func(s);
}
```

以上的代码就删除虚函数的标记，把类型S改为标准布局的C++类，并用静态断言再次保证类型S为标准布局的类。这样可以提早提醒开发人员，避免遗漏这个隐患。

#### 代码样例对比

``` cpp

//app.cpp
struct B {
  int i, j;
};

struct D : B {
  float f;
};

//library.h
extern "Fortran" void func(void *);

//library.cpp
void func(void* )
{

}

//app.cpp
void foo(D *d) {
  func(d);
}
```

以上的代码显示，一个指向非标准布局类的指针在foo函数中被传入了一个具有Fortran语言链接的func函数中，这样显然是不正确的，可能会导致非确定行为，有安全隐患。

``` cpp
struct B {
  int i, j;
};

struct D : B {
  float f;
};

extern "Fortran" void func(void *);

void foo(D *d) {
  struct {
    int i, j;
    float f;
  } temp;

  temp.i = d->i;
  temp.j = d->j;
  temp.f = d->f;
  func(&temp);
}
```
以上代码只改了一个地方，就是把非标准布局类型D序列化成局部标准布局类型，然后再传入func中。这样就完美了。

### EXP61-CPP. 一个lambda对象在生命周期必须在它所引用捕获的对象生命周期内

严重程度：高。实际上本质来说，就是捕获引用到了一个处于它自身声明周期外的对象。

这里不介绍lambda捕获列表的知识点，直接看代码样例分析。

#### 代码样例对比

``` cpp
auto g() {
  int i = 12;
  return [&] {
    i = 100;
    return i;
  };
}

void f() {
  int j = g()();
}
```

以上代码很简单了，g()函数返回了一个lambda函数，但是这个lambda函数以引用捕获了局部变量i，然后g()函数返回完成，i的生命周期已经完结。然而，返回的lambda函数还持有i的引用，并且被调用了，对i进行了i=100的访问。这直接导致未定义行为。

``` cpp
auto g() {
  int i = 12;
  return [=] () mutable {
    i = 100; 
    return i;
  };
}

void f() {
  int j = g()();
}
```

知道了原因很容易改正确，把lambda函数的引用捕获改成值传递捕获，这样i就是lambda的隐式的非静态数据成员，而且i的生命周期与lambda的生命周期就绑定了，简单来说，i的生命周期就是lambda的生命周期。

#### 代码样例对比

``` cpp
auto g(int val) {
  auto outer = [val] {
    int i = val;
    auto inner = [&] {
      i += 30;
      return i;
    };
    return inner;
  };
  return outer();
}

void f() {
  auto fn = g(12);
  int j = fn();
}
```
从上面代码可以看出。inner的lambda函数引用捕获了outer lambda函数的局部变量，但是inner函数对变量i的访问超出了outer函数和变量i的生命周期，，所以会导致未定义行为。这样就改成以下：

``` cpp
auto g(int val) {
  auto outer = [val] {
    int i = val;
    auto inner = [i] {
      return i + 30;
    };
    return inner;
  };
  return outer();
}

void f() {
  auto fn = g(12);
  int j = fn();
}
```

### EXP62-CPP. 不要访问不属于对象的value representation的bit位域

严重等级：高。依赖于Bit位的访问会导致implementation-defined的行为以及恶意代码执行的后果。

C++标准[basic.types]对于对象的value representation有如下说明：

> The object representation of an object of type T is the sequence of N unsigned char objects taken up by the object of type T, where N equals sizeof(T). The value representation of an object is the set of bits that hold the value of type T.

#### 代码样例对比

``` cpp
#include <cstring>

struct S {
  unsigned char buffType;
  int size;
};

void f(const S &s1, const S &s2) {
  if (!std::memcmp(&s1, &s2, sizeof(S))) {
    // ...
  }
}
```

以上代码片段很明显的可以看出，作者想用object presentation的对比两个对象是否相等，但是这是很可能出问题的，根据C++标准，类的内存布局可能会因为内存对齐而被编译器插入padding位。这就导致了这些padding位的具体内容是implementation-defined的。所以我们不能依赖实现。因为两个对象的padding位的具体内容可能不同，造成比较对象不正确。

改成以下代码手工对比：

``` cpp
struct S {
  unsigned char buffType;
  int size;
  friend bool operator==(const S &lhs, const S &rhs) {
    return lhs.buffType == rhs.buffType && lhs.size == rhs.size;
  }
};

void f(const S &s1, const S &s2) {
  if (s1 == s2) {
    // ...
  }
}
```

#### 代码样例对比

``` cpp
#include <cstring>

struct S {
  int i, j, k;
  // ...
  virtual void f();
};

void f() {
  S *s = new S;
  // ...
  std::memset(s, 0, sizeof(S));
  // ...
  s->f(); // undefined behavior
}
```

上面的代码意图通过memset把S类型的s对象置0，如果S类型没有虚函数还好，但是错误就在S类型有虚函数，所以s对象有个虚函数表的指针，这个指针的值也被置为0了。这显然是意料之外。一旦调用虚函数，那么该指针无效，引发未定义行为。如果对于这种类型，最好自己写一个清空函数，如下:

``` cpp
struct S {
  int i, j, k;
  // ...
  virtual void f();
  void clear() { i = j = k = 0; }
};

void f() {
  S *s = new S;
  // ...
  s->clear();
  // ...
  s->f(); // ok
}
```

以上代码看clear()函数的实现，把非静态数据成员全部置为0了，而且也没有覆盖虚函数表指针。

### EXP63-CPP. 不要依赖moved-from对象的值

严重程度： 中等。一般来说，moved-from对象的状态是合法的，但是状态是未指定的。依赖非指定的值可能会导致程序非正常终止也会导致数据完整性违例。

许多类型，包括用户自定义的类型，还有STL所提供的类型都支持move语义（移动语义）。

如果一个函数的参数是某类型对象的右值引用，那么当该函数调用的时候，函数参数对象就会隐式调用自身的移动赋值函数或者移动构造函数。所谓移动就是把一个对象的状态移动到另一个对象中，这就产生了，前一个对象状态被掏空，处于一个未指定值的状态，所以不能依赖前一个对象的值。

#### 代码样例对比

``` cpp
#include <iostream>
#include <string>

void g(std::string &&v) {
  std::cout << v << std::endl;
}

void f() {
  std::string s;
  for (unsigned i = 0; i < 10; ++i) {
    s.append(1, static_cast<char>('0' + i));
    g(std::move(s));
  }
}
```
以上这段代码为了提高性能，g()函数的参数用了std::string的右值引用。首先对象s和g()函数的操作都在for循环中，开始的第一步不会有问题，但是第一步最后std::move把对象s的状态给掏空了，处于未指定状态，但是第二步开始，又对这个对象s进行append操作，这就导致了问题了,非期待的输出结果。

知道了原因，就好办了，不应该依赖被移动过的对象的状态，应该新创建一个对象，代码如下:

``` cpp
#include <iostream>
#include <string>

void g(std::string &&v) {
  std::cout << v << std::endl;
}

void f() {
  for (unsigned i = 0; i < 10; ++i) {
    std::string s(1, static_cast<char>('0' + i));
    g(std::move(s));
  }
}
```

#### 代码样例对比

``` cpp
#include <algorithm>
#include <iostream>
#include <vector>

void f(std::vector<int> &c) {
  std::remove(c.begin(), c.end(), 42);
  for (auto v : c) {
    std::cout << "Container element: " << v << std::endl;
  }
}
```

以上的代码元素42被移除以后，那么vector c的迭代范围就是非法的范围了，强制进行range for会导致非指定行为(unspecified behavior)。

可以用以下两种方式来改正:

``` cpp
#include <algorithm>
#include <iostream>
#include <vector>

void f(std::vector<int> &c) {
  auto e = std::remove(c.begin(), c.end(), 42);
  for (auto i = c.begin(); i != c.end(); i++) {
    if (i < e) {
      std::cout << *i << std::endl;
    }
  }
}
```

换一种方式：

``` cpp
#include <algorithm>
#include <iostream>
#include <vector>

void f(std::vector<int> &c) {
  c.erase(std::remove(c.begin(), c.end(), 42), c.end());
  for (auto v : c) {
    std::cout << "Container element: " << v << std::endl;
  }
}
```