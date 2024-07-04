#cpp

## 前言

随着C++ 17标准今年出来以后，各大社区都有C++ 17的讨论，C++的爱好者们都希望这门语言变得更像现代编程语言一样，越来越好用，拥有越来越高级的抽象语义。以下是知乎上的几个讨论：

- [怎样看待C++17众多功能流产?](https://www.zhihu.com/question/42152208)
- [C++17 有哪些值得注意的新特性?](https://www.zhihu.com/question/32222337)
- [C++17 基本完成，对于新特性大家怎么看?](https://www.zhihu.com/question/56943731)
- [C++ 17的19个基本特性](https://www.oschina.net/news/85129/top-19-new-features-of-c17-you-need-to-know)

另外，各大编译器产商也对C++ 17的标准实现进行了跟进。本着垠神的文章[《如何掌握所有的程序语言》](http://www.yinwang.org/blog-cn/2017/07/06/master-pl)里面所说，不要关注语言的语法细节，而着重关注学习语言的特性，我打算探索下C++ 17的语言特性，当然，有些特性在其他语言也有对应的概念，比如any，variant，optional等在Java 8中也有了。


## Optional

C++17提供了optional这个特性，该特性以类的形式提供，管理一个可选的值。它的常用场景其实没什么高深的，类似于lambda表达式简化了代码，该特性可以用来表示函数的返回值，该返回值可以表现函数可能会调用失败的信息。（注：boost也实现了此特性）

比如你想实现一个把字符串转换为整数的函数，你采用最传统的方法来实现:

``` cpp
bool parse_int(const std::string& s, int& i)
{
   // 实现算法
}
```

上面这个函数的签名其实并不美观，简洁。第一个参数为需要处理的输入字符串，第二个参数是处理成功时候通过引用的输出参数。最后用了bool返回值来告诉caller是否处理失败。

或者为了减少参数，降低干扰信息，还可以用一个方法，但是还是比较丑陋:

``` cpp
int* parse_int(const std::string& s)
{
   // 实现算法
   // 如果失败，返回nullptr
}
```
上面的函数用空指针代替了失败的信息。函数签名还是不美观，而且这样的用法极少，需要动态内存分配。很少有开发人员会通过内部new一个int对象，等到caller使用完毕，caller需要自己手动delete:

``` cpp
int *result = parse_int("12334");
if(result)
{
    std::cout << *result << std::endl;
    delete result; // 如果使用完毕忘记释放，内存泄漏
}
else
{
    std::cout << "none" << std::endl;
}
```

如果使用optional就方便了:

``` cpp
std::optional<int> parse_int(const std::string& s)
{
   int result;
   //....
   if(isSuccess)
   {
       return result;
   }
   return {};
}

int main()
{
     std::cout << "result is: " << parse_int("123456").value_or("invalid paramter") << '\n';
     std::cout << "result is: " << parse_int("dwafawf").value_or("invalid paramter") << '\n';
    
    if(auto result = parse_int("5643"))
    {
        std::cout << "result is: " << *result << '\n';
    }
    
    return 0;
}
```
以上代码会输出:

```
result is: 123456
result is: invalid parameter
result is: 5643
```

还有一个让代码简洁的例子:

``` cpp
template<typename Key, typename Value>
class Lookup
{
    std::optional<Value> get(Key key);
};

Lookup<std::string, std::string> location_lookup;
std::string location = location_lookup.get("waldo").value_or("unknown");
```

## Variant

在C++ 17中以std::variant这个类提供，表示一个类型安全的Union类型。当然，boost有对应的[Boost.Variant](http://www.boost.org/doc/libs/1_57_0/doc/html/variant.html), Qt有对应的[QVariant](http://doc.qt.io/qt-5/qvariant.html)。 这个类类型的一个实例在给定任何一个时刻保留其中一个类型的值。

设计的动机：

在很多时候，在开发C++程序的过程中，你需要一个类型表示多种类型其中的任何一个类型的时候，你可能相当，union这个关键字来实现以下:

``` cpp
union { int i; double d; } u;
u.d = 3.14;
u.i = 3; // overwrites u.d (OK: u.d is a POD type)
```
变量u既可以保存int类型的值，又可以保存double类型的值，但是同一个时刻，只有其中一个类型的值保存在其中，但是很遗憾，union关键字一般只支持基本类型，比如int，char，double之类的POD类型，如果用C++使用面向对象的方式编程，以下非POD的类型就不支持:

``` cpp
union {
  int i;
  std::string s; // illegal: std::string is not a POD type!
} u;
```

于是std::variant就产生了:

``` cpp
 std::variant< int, std::string > u("hello world");
 std::cout << u; // output: hello world
 u = 13;
 std::cout << u; // output: 13

std::variant<int, double> v{ 12 };
std::get<int>(v); // == 12
std::get<0>(v); // == 12
v = 12.0;
std::get<double>(v); // == 12.0
std::get<1>(v); // == 12.0
```

当然，std::variant确实有点用，但是有点鸡肋的感觉。相比Qt的QVariant相差十万八千里。

## Any

在C++17中，这玩意儿以std::any的类提供，表示对于一个任意类型的类型安全的单值容器。当然了，Boost库也有对应的[Boost.Any](http://www.boost.org/doc/libs/develop/doc/html/any.html),当然了这玩意儿跟Variant很像，所以在boost的官方文档中，有了这两个类型的对比:

> As a discriminated union container, the Variant library shares many of the same features of the Any library. However, since neither library wholly encapsulates the features of the other, one library cannot be generally recommended for use over the other.

> That said, Boost.Variant has several advantages over Boost.Any, such as:
> - Boost.Variant guarantees the type of its content is one of a finite, user-specified set of types.
> - Boost.Variant provides compile-time checked visitation of its content. (By contrast, the current version of Boost.Any provides no visitation mechanism at all; but even if it did, it would need to be checked at run-time.)
> - Boost.Variant enables generic visitation of its content. (Even if Boost.Any did provide a visitation mechanism, it would enable visitation only of explicitly-specified types.)
> - Boost.Variant offers an efficient, stack-based storage scheme (avoiding the overhead of dynamic allocation).

> Of course, Boost.Any has several advantages over Boost.Variant, such as:

> - Boost.Any, as its name implies, allows virtually any type for its content, providing great flexibility.
> - Boost.Any provides the no-throw guarantee of exception safety for its swap operation.
> - Boost.Any makes little use of template metaprogramming techniques (avoiding potentially hard-to-read error messages and significant compile-time processor and memory demands).


以下是std::any的用法：

``` cpp
std::any x{ 5 };
x.has_value() // == true
std::any_cast<int>(x) // == 5
std::any_cast<int&>(x) = 10;
std::any_cast<int>(x) // == 10
```

还有更详细的用法:

``` cpp
#include <string>
#include <iostream>
#include <any>
 
int main()
{
    // simple example 
 
    auto a = std::any(12);
 
    std::cout << std::any_cast<int>(a) << '\n'; 
 
    try {
        std::cout << std::any_cast<std::string>(a) << '\n';
    }
    catch(const std::bad_any_cast& e) {
        std::cout << e.what() << '\n';
    }
 
    // advanced example
 
    a = std::string("hello");
 
    auto& ra = std::any_cast<std::string&>(a); //< reference
    ra[1] = 'o';
 
    std::cout << "a: " << std::any_cast<const std::string&>(a) << '\n'; //< const reference
 
    auto b = std::any_cast<std::string&&>(a); //< rvalue reference (no need for std::move)
 
    // Note, 'b' is a move-constructed std::string, 'a' is now empty
 
    std::cout << "a: " << *std::any_cast<std::string>(&a) //< pointer
        << "b: " << b << '\n';
}
```

从以上代码可以看出来，std::any的使用场景旨在提供类型安全的void* ，你会发现，在很多C/C++开发的系统软件的源码里面都用void* 来传递上下文（context），无论是线程上下文也好还是其他也罢，但是用void* 来传递上下文信息，原来的类型信息就丢失了，到了要使用上下文的时候，需要强制转换成原来的类型，但是万一转换的类型不对，出错怎么办？如果出错，程序会直接崩溃，不会有任何提示，如果使用了std::any在转换的过程中，如果出错，还会以抛异常的方式来提示用户。

[所以大多数情况下使用Any，既可以消除void\*的隐患问题，又一样的保证了之前void\*的低开发成本，一举两得。](https://www.zhihu.com/question/56317879)


## std::string_view

这个东西呢，是非真正意义地引用一个字符串，一般用来提供一个字符串之上的抽象，也就是这个view是std::string的一个抽象，可以简单理解为，view是std::string对象的展示层，它不存储实际的数据，只读，不可修改。std::string是Model层，与软件工程中的Model和View对应。

由于以上的一些特点，通常会用到在字符串操作上[性能比较苛刻的场景](https://www.zhihu.com/question/63164644)。

stackoverflow上也有一个[讨论](https://stackoverflow.com/questions/20803826/what-is-string-view)以说明这个库特性的设计动机。

以下是用法:

``` cpp
std::string str{ "   trim me" };
std::string_view v{ str };

v.remove_prefix(std::min(v.find_first_not_of(" "), v.size()));

str; //  == "   trim me"
v; // == "trim me"
```

``` cpp
#include <iostream>
#include <string_view>
int main()
{
    std::string str = "Exemplar";
    std::string_view v = str;
    std::cout << v[2] << '\n';
//  v[2] = 'y'; // Error: cannot modify through a string view
    str[2] = 'y';
    std::cout << v[2] << '\n';
}
```

在以上的第二个代码片段，可以看到把在Model层的str对象修改了，与它关联的view对象立即内容就随之改变了，反之就不能通过view层的v对象来修改Model层的str对象。如果是C++之前的引用概念的话，就能通过引用来修改原对象了。

以前端开发中的React和Vue的状态管理来说，就是说，View的状态变化与Model的状态变化一致，View的状态随着Model的状态改变而改变。但是View自身的状态改变却不能影响Model的状态改变。这个状态传递是单向的，不是双向的。

## std::invoke

这个没多少要说的，就是调用一个callable的对象，还可以传递参数。

Callable对象顾名思义就是，可以像普通函数那样调用的对象，比如std::function等等。

以下为用法:

``` cpp
#include <functional>
#include <iostream>
 
struct Foo {
    Foo(int num) : num_(num) {}
    void print_add(int i) const { std::cout << num_+i << '\n'; }
    int num_;
};
 
void print_num(int i)
{
    std::cout << i << '\n';
}
 
struct PrintNum {
    void operator()(int i) const
    {
        std::cout << i << '\n';
    }
};
 
int main()
{
    // invoke a free function
    std::invoke(print_num, -9);
 
    // invoke a lambda
    std::invoke([]() { print_num(42); });
 
    // invoke a member function
    const Foo foo(314159);
    std::invoke(&Foo::print_add, foo, 1);
 
    // invoke (access) a data member
    std::cout << "num_: " << std::invoke(&Foo::num_, foo) << '\n';
 
    // invoke a function object
    std::invoke(PrintNum(), 18);
}
```

以上代码输出:

```
-9
42
314160
num_: 314159
18
```

然而，还可以再复杂一点，创建一个代理调用函数的模版类:

``` cpp
template <typename Callable>
class Proxy {
    Callable c;
public:
    Proxy(Callable c): c(c) {}
    template <class... Args>
    decltype(auto) operator()(Args&&... args) {
        // ...
        return std::invoke(c, std::forward<Args>(args)...);
    }
};

auto add = [] (int x, int y) {
  return x + y;
};

Proxy<decltype(add)> p{ add };
p(1, 2); // == 3
```
该模板类可以接收任何函数并作为其代理。

## std::apply

这个函数是调用Callable对象并把元组（Tuple）化的参数序列传递给Callable对象，这样就方便了:

``` cpp
auto add = [] (int x, int y) {
  return x + y;
};
std::apply(add, std::make_tuple( 1, 2 )); // == 3
```

这个特性在类似的函数式语言都会有，比如[Scheme](https://www.zhihu.com/question/54627596)中。


## 类模版参数推导

自动模版参数推导类似已经完成的函数的参数推导，但是现在可以推导模版类构造函数了:

``` cpp
template <typename T = float>
struct MyContainer {
  T val;
  MyContainer() : val() {}
  MyContainer(T val) : val(val) {}
  // ...
};

MyContainer c1{ 1 }; // OK MyContainer<int>
MyContainer c2; // OK MyContainer<float>
```

## 用auto声明无类型的模版参数

``` cpp
emplate <auto ... seq>
struct my_integer_sequence {
  // Implementation here ...
};

// Explicitly pass type `int` as template argument.
auto seq = std::integer_sequence<int, 0, 1, 2>();
// Type is deduced to be `int`.
auto seq2 = my_integer_sequence<0, 1, 2>();
```


## Folding表达式

一个folding表达式执行一个封装了对模版参数二元操作的折叠。

- 一个类似(... op expr)或(expr op ...)这样形式的表达式，其中op为fold操作符，expr为未展开的参数包裹，这样的形式叫一元折叠(unary fold)。

- 一个类似(expr1 op ... op expr2)这样形式的表达式，其中op为fold操作符，叫二元折叠(binary fold)。expr1和expr2都是未展开的参数包裹，但也可能不都是未展开的。

二元折叠:

``` cpp
template<typename... Args>
bool logicalAnd(Args... args) {
    // Binary folding.
    return (true && ... && args);
}

bool b = true;
bool& b2 = b;
logicalAnd(b, b2, true); // == true
```

一元折叠:

``` cpp
template<typename... Args>
auto sum(Args... args) {
    // Unary folding.
    return (... + args);
}

sum(1.0, 2.0f, 3); // == 6.0
```

## 在花括号初始化列表中的auto推导的新规则


改变了当采用统一初始化语法auto的推导规则。原来，auto x{3}被推导为std::initializer_list\<int\>类型，现在变为直接推导为int类型。



``` cpp
auto x1{ 1, 2, 3 }; // error: not a single element
auto x2 = { 1, 2, 3 }; // decltype(x2) is std::initializer_list<int>
auto x3{ 3 }; // decltype(x3) is int
auto x4{ 3.0 }; // decltype(x4) is double
```



## constexpr的lambda表达式

使用constexpr构造编译时lambda表达式:

``` cpp
auto identity = [] (int n) constexpr { return n; };
static_assert(identity(123) == 123);

constexpr auto add = [] (int x, int y) {
  auto L = [=] { return x; };
  auto R = [=] { return y; };
  return [=] { return L() + R(); };
};

static_assert(add(1, 2)() == 3);

constexpr int addOne(int n) {
  return [n] { return n + 1; }();
}

static_assert(addOne(1) == 2);
```

## lambda以值方式捕获this指针

之前的C++标准只能以引用的方式捕获this指针，现在可以以值来捕获了。因为以前在使用callback函数的异步代码中，必须要求一个合法对象，万一对象超过其生命周期，那么程序就挂了。所以在C++17中*this是对当前的对象拷贝了一个副本，而this还是类似C++11的标准一样以引用捕获:

``` cpp
struct MyObj {
  int value{ 123 };
  auto getValueCopy() {
    return [*this] { return value; };
  }
  auto getValueRef() {
    return [this] { return value; };
  }
};

MyObj mo;
auto valueCopy = mo.getValueCopy();
auto valueRef = mo.getValueRef();
mo.value = 321;
valueCopy(); // 123
valueRef(); // 321
```

## 内联变量

在该标准中，inline关键字既能作用于函数也能作用于变量了，作用于变量和作用于函数的语义都是一样的。使用场景嘛，都是为了提高性能。

``` cpp
// Disassembly example using compiler explorer.
struct S { int x; };
inline S x1 = S{321}; // mov esi, dword ptr [x1]
                      // x1: .long 321

S x2 = S{123};        // mov eax, dword ptr [.L_ZZ4mainE2x2]
                      // mov dword ptr [rbp - 8], eax
                      // .L_ZZ4mainE2x2: .long 123
```

## 嵌套namespace

这个不必多说，主要是书写代码更简洁了:

``` cpp
// C++ 11
namespace A {
  namespace B {
    namespace C {
      int i;
    }
  }
}

// C++ 17
namespace A::B::C {
  int i;
}
```

## Structured bindings

其实这个特性Python里面也有类似的。这个标准提案的目的是解构初始化，它是这样使用的： auto [x,y,z] = expr。 expr作为一个表达式，它需要返回tuple-like的对象，这个对象的元素必须与x，y，z进行绑定。tuple-like的对象包括std::tuple, std::pair , std::array和聚合结构体。

``` cpp
using Coordinate = std::pair<int, int>;
Coordinate origin() {
  return Coordinate{0, 0};
}

const auto [ x, y ] = origin();
x; // == 0
y; // == 0
```

## constexpr if

这个特性还是相当有用的，让代码实例化依赖于编译时的条件。

``` cpp
template <typename T>
constexpr bool isIntegral() {
  if constexpr (std::is_integral<T>::value) {
    return true;
  } else {
    return false;
  }
}

static_assert(isIntegral<int>() == true);
static_assert(isIntegral<char>() == true);
static_assert(isIntegral<double>() == false);

struct S {};
static_assert(isIntegral<S>() == false);
```


## UTF-8字符字面量

UTF-8终于被纳入标准了:

``` cpp
char x = u8'x'; //被编码为UTF-8
```
