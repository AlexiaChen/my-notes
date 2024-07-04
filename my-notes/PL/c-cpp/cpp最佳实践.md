

> 该篇文章是我自己的经验总结，不可能100%适合读者，当然相关的C++工程实践书籍类似《Effective C++》，《More Effective C++》，《Modern Effective C++》，《Learning C++ Best Practices》 《Google C++ Coding Style》等等可能都有类似描述,我这篇文章可能也是从以上的书籍文章汲取了一些。

# 工欲善其事必先利其器

现在软件工程越来越发达，C++的标准也一直在改进更新，这门在大众来看的“古老”语言也在慢慢变得更加像现代编程语言一样了，现代软件工程持续交付，持续集成等工具概念层出不穷，这里我想推荐些工具给读者改进优化项目开发流程

- [Cmake](https://cmake.org/)，最好的C++跨平台构建工具，没有之一，automake，qmake在它面前黯然失色。

- [Travis CI](http://travis-ci.org/), 这个是持续集成工具，能在Github上很好的工作。

- [CppCheck](http://cppcheck.sourceforge.net/), C/C++ 静态分析工具，免费的，能查出很多类型缺陷，内存泄漏和资源泄漏。当然还有很多语言的静态分析工具，如果有兴趣，请看[这里](https://en.wikipedia.org/wiki/List_of_tools_for_static_code_analysis)

另外，在个人项目和公司项目中，C++编译器，无论在g++，MSVC，或者clang上，请把警告级别调整到最高。MSVC我是调整到W4级别，g++上，由于本人不熟悉g++的警告类型，那么请开-Wall -Wextra警告并严格观察，另外g++上还可以开-Weffc++选项，编译器会按照《Effective C++》的实践规范来检查代码的隐患。这些都是很重要，编译器的警告很[重要](https://www.zhihu.com/question/29155164)！！要好好利用静态类型语言带来的优点。最后把警告尽量消除到0 warnings为止！ 最后的最后，用C++的静态分析工具检查一遍所有的源码，选择性的消除工具报告出来的缺陷。这样你会发现，后期的软件的运行时的BUG会少很多，特别是不明不白的crash。

# 正文

#### 1.基本C++命名规范

- 类名用驼峰命名法: MyClass
- 类的成员函数和变量名开头单词用小写：myMethod
- 常量全用大写：const double PI = 3.1415926

另外C++标准库和Boost采用另一种规范，如果你的代码与标准库和Boost混合写契合度很高，推荐用以下的规范：

- 宏名称单词全用大写，单词之间用下划线隔开： INT_MAX
- 模版参数使用驼峰命名法： InputInterator
- 其他所有变量和函数名，类名全用小写单词加下划线隔开：make_shared,unordered_map,dynamic_cast

#### 2.区分私有成员变量

- 在私有成员变量前面加入m_前缀, m代表“member”： m_height

当然，个别一些习惯，是在私有成员变量后加下划线后缀： object_

#### 3.区分函数参数

- 在函数参数名加入t_前缀： t_height

当然，代码最重要的还是要与CodeBase一致，最终看公司的规范，这里只是一个样例，t可以认为是“the”的缩写。这只是区分函数参数与局部变量的一种策略。

#### 4.任何命名不能是下划线开头

如果你这么做，那么可能会与编译器的扩展关键字造成冲突。如果好奇，那么请看stackoverflow的这个[讨论](http://stackoverflow.com/questions/228783/what-are-the-rules-about-using-an-underscore-in-a-c-identifier)。

#### 5.一个良好的样例

``` cpp
class MyClass
{
public:
  MyClass(int t_data, int t_attr)
    : m_data(t_data),m_attr(t_attr)
  {
  }

  int getData() const
  {
    return m_data;
  }

  int attribute() const
  {
      return m_attr;
  }

private:
  int m_data;
  int m_attr;
};
```

#### 6.空指针的表示请用nullptr

C++ 11 中的空指针是一个特定的值，用以代替0或NULL。 如果好奇，请看知乎的[讨论](https://www.zhihu.com/question/55936870), 不然值会有二义性。另外，知乎上还有[讨论2](https://www.zhihu.com/question/22203461)。

#### 7.注释

优先使用//来注释代码块，不要使用/**/

#### 8.不要在头文件中使用using namespace

这会导致using的名字空间污染范围扩散，因为使用了这个头文件的源文件都隐式使用这个名字空间了，这将来容易造成名字空间的冲突，该错误查找困难。不利于后来开发人员维护。

#### 9.头文件守护

这个想必很多人已经习惯了，不过如果不这样做的危害还是要说一下，这样可以防止头文件被重复包含多次而造成的问题,也能解决意外包含其他工程头文件的冲突。

``` cpp
#ifndef MYPROJECT_MYCLASS_HPP
#define MYPROJECT_MYCLASS_HPP

namespace MyProject {
  class MyClass {
  };
}

#endif
```

#### 10.代码块一定要用{}

如果不这么做，可能会导致一些语义错误。

``` cpp
// Bad Idea
// 这么做虽然没错，能按照预想运行，但是会给后来人员造成迷惑
for (int i = 0; i < 15; ++i)
  std::cout << i << std::endl;

// Bad Idea
// 这就有错了，std::cout没在循环内，变量i也不是循环内的，与预想不一致
int sum = 0;
for (int i = 0; i < 15; ++i)
  ++sum;
  std::cout << i << std::endl;


// Good Idea
// 这个语义就完全正确了。
int sum = 0;
for (int i = 0; i < 15; ++i) {
  ++sum;
  std::cout << i << std::endl;
}
```

#### 11.限制代码列的字符数

一般推荐是80-100个字符之间，我自己是80。一般IDE和文本编辑器都可以强制限制。

``` cpp
// Bad Idea
// 难阅读
if (x && y && myFunctionThatReturnsBool() && caseNumber3 && (15 > 12 || 2 < 3)) {
}

// Good Idea
// 逻辑思路跟得上了，容易阅读
if (x && y && myFunctionThatReturnsBool()
    && caseNumber3
    && (15 > 12 || 2 < 3)) {
}
```

#### 12.使用""包含本地头文件

<> 是保留给标准库和系统库头文件的，自己写的本地头文件#include "MyHeader.h"

#### 13.初始化成员变量

最好用初始化成员列表来初始化。

``` cpp
// Bad Idea
class MyClass
{
public:
  MyClass(int t_value)
  {
    m_value = t_value; //这是赋值，而不是初始化
  }

private:
  int m_value;
};


// Good Idea
// C++ 初始化成员列表是C++语言特有的，这样写代码更加清晰干净，
// 而且还有潜在的性能提升，因为初始化和赋值不是一个概念。
// 《Effective C++》也提到过
class MyClass
{
public:
  MyClass(int t_value)
    : m_value(t_value)
  {
  }

private:
  int m_value;
};
```
好奇请戳知乎的[讨论](https://www.zhihu.com/question/37345224)。

当然，在C++ 11中，你可以考虑总是给成员变量一个默认值，

``` cpp
// ... //
private:
  int m_value = 0;
// ... //
```

使用大括号初始化,因为它在编译时不允许数据收窄。

``` cpp
// Best Idea

// ... //
private:
  int m_value{ 0 }; // allowed
  unsigned m_value_2 { -1 }; // compile-time error, narrowing from signed to unsigned.
// ... //
```

优先使用大括号初始化，除非有原因的特殊要求不那么做。

#### 14.总是使用名字空间

在C语言时代，很多库的开发者，为了防止函数符号链接时冲突，就在函数名加入库名称的前缀，比如OpenCV的函数都是cv_xxxx。当然，这是历史原因，如果是采用C++ 编译器，就应该使用namespace防止符号冲突，采用boost库的方式。

#### 15.使用标准库提供的正确的整型类型

在C++中，最好不要出现int类型，最好是intxxx_t , uintxxx_t。 表示大小请使用std::size_t。

可以看这里的[参考](http://www.cplusplus.com/reference/cstdint/)，这样提高可移植性，因为在不同类型的平台上，这些类型会typedef到特定类型上去。

注意： signed char 保证至少 8 位，int 保证至少 16 位，long 保证至少 32 位，long long 保证至少 64 位。

如果还不明白，去看stackoverflow的[讨论](http://stackoverflow.com/questions/13398630/why-are-c-int-and-long-types-both-4-bytes)

#### 16.Tab和空格不要混合使用

这个绝对禁止，应该从编辑器和IDE里面更改设置，比如让Tab等于4个空格。至于设置Tab等于多少个空格合理，这个是个人喜好问题，不然就是Emacs和Vim之争了。

#### 17.不要害怕模版

这个，我对于C++的模版元编程不熟悉，就不做过多讨论了。模版可以说是另外一种语言，另一种“函数式”语言。它是图灵完备(Turing-Complete)的。

``` cpp
// check specific class if it has foo() function
template <typename Ty>
class has_foo_function {

private:
	typedef char yes[1];
	typedef char no[2];
	
	template <typename Inner>
	static yes& test(Inner *I, decltype(I->foo()) * = nullptr);
	
	template <typename>
	static no& test(...);
	
public:
	static const bool value =
		sizeof(test<Ty>(nullptr)) == sizeof(yes);
};

class MyTest1
{
public:
	void foo(){};
	
};

class MyTest2
{
public:
	
};

class MyTest3
{
public:
	int foo(int s){ return s; }
};

int main()
{
  std::cout << has_foo_function<MyTest1>::value << std::endl; // 1
	std::cout << has_foo_function<MyTest2>::value << std::endl; // 0
	std::cout << has_foo_function<MyTest3>::value << std::endl; // 0
  return 0;
}
```

感兴趣可以在网络上看到各种玩法：
- [玩模板元编程走火入魔是一种怎样的体验](https://www.zhihu.com/question/46612915)
- [C++ 模板元编程的应用有哪些，意义是什么](https://www.zhihu.com/question/21656266)
- [模板元编程和泛函编程都是函数式编程吗](https://www.zhihu.com/question/39637015)
- [如何正确的学习C++的模板和模板元编程](https://www.zhihu.com/question/23463256)
- [怎么样才算是精通 C++](https://www.zhihu.com/question/19794858)

#### 18.慎用操作符重载

重载操作符是个很方便的特性，比如用C++ 实现一个BigInteger类，一个类的实体表示一个大整数，如果给类实现一个add函数表示大整数做加法，a.add(b),那么太不直观了，a + b更加方便直观。另一个官方例子就是std::string重载了+号操作符，以便字符串的拼接。当然，重载操作符本质上还是调用函数。

虽然方便，但是如果随意重载操作符，可能导致许多诡异的错误，详情见这里的[讨论](http://stackoverflow.com/questions/4421706/operator-overloading/4421708)。

尤其是，当需要编写重载操作符的时候，应该把以下几点时时刻刻记住在脑子里：

- 当有必须的理由的时候才重载 operator= 
- 对于剩下的全部操作符，也是有需要在一个场景下使用的时候才重载它们，这个场景比如，+ - 是一对语义上有关联的符号
- 时刻注意[操作符的优先级](http://en.cppreference.com/w/cpp/language/operator_precedence)，即使重载了，那操作符的优先级也不会改变，不能凭感觉猜测。
- 不要重载一些特殊奇怪的操作符，比如 ~ 和 % #之类的，除非有理由让你这么做
- [**永远**](http://stackoverflow.com/questions/5602112/when-to-overload-the-comma-operator)不要重载逗号(,)操作符

剩下给出一个重载操作符的[参考资料](http://courses.cms.caltech.edu/cs11/material/cpp/donnie/cpp-ops.html)

#### 19.消除隐式转换

- 对于单参构造函数,必须在构造函数前加入 explicit 关键字，要求显式调用
- 对于操作符转换也是一样的，需要加 explicit 关键字 

``` cpp
//bad idea
struct S {
  operator int() {
    return 2;
  }
};

//good idea
struct S {
  explicit operator int() {
    return 2;
  }
};
```

#### 20.考虑零原则（Rule of Zero）

该原则声称，不要提供任何编译器能自动生成的函数（拷贝构造，拷贝赋值构造，移动构造，移动赋值构造，析构函数），除非该类需要构造一些关于所有权（ownership）的概念。

该原则的目的是让当更多的成员变量被添加到该类时，让编译器提供最佳的版本，换句话说，这些东西最好让编译器来维护。

该原则的背景在[这里](http://blog.florianwolters.de/educational/2015/01/31/The_Rule_of_Zero/)，然后[这篇文章](http://www.nirfriedman.com/2015/06/27/cpp-rule-of-zero/)做了实现并解释了技术。


#### 21.尽可能多的用const修饰

const 告诉编译器方法或变量是不可变（immutable）的，这也能帮助编译器优化代码,同时也能帮助开发者知道函数是否有副作用（side-effect）。另外，常引用(const &)也能阻止编译器进行不必要的拷贝代价。感兴趣的可以看这篇文章底下[John Carmack的评论](http://kotaku.com/454293019)。

``` cpp
// If the parameter is input parameter, not output
// Bad Idea
class MyClass
{
public:
  void do_something(int i);
  void do_something(std::string str);
};


// Good Idea
class MyClass
{
public:
  void do_something(const int i);
  void do_something(const std::string &str);
};


``` 

#### 22.小心函数返回的类型

Getters:
- 以引用返回或者常引用返回的,在观察者模式下这会有一定的性能提升。
- 以值返回更加有助于线程安全（thread safety）,即使有拷贝的代价，由于[RVO](https://en.wikipedia.org/wiki/Return_value_optimization)的作用，性能也不会损失。

临时和局部变量：
- 总是以值返回

#### 23.不要用常引用(const &)来传递或返回一个基本类型变量

``` cpp
// Very Bad Idea
class MyClass
{
public:
  explicit MyClass(const int& t_int_value)
    : m_int_value(t_int_value)
  {
  }

  const int& get_int_value() const
  {
    return m_int_value;
  }

private:
  int m_int_value;
}

// Good Idea
class MyClass
{
public:
  explicit MyClass(const int t_int_value)
    : m_int_value(t_int_value)
  {
  }

  int get_int_value() const
  {
    return m_int_value;
  }

private:
  int m_int_value;
}

```
为什么呢？ 因为对于基本类型变量用引用传递或返回会导致指针操作，这会更加慢。如果通过值传递，是利用处理器寄存器直接传递的，会更加快。

#### 24.避免裸指针访问内存

直接内存分配和释放在C++中做到完全消除内存错误和内存泄漏是很难的，要善于利用RAII，C++ 11提供了智能指针这种方便的工具，基本可能杜绝内存泄漏。

``` cpp
// Bad Idea
MyClass *myobj = new MyClass;

// ...
delete myobj;


// Good Idea
auto myobj = std::make_unique<MyClass>(constructor_param1, constructor_param2); // C++14
auto myobj = std::unique_ptr<MyClass>(new MyClass(constructor_param1, constructor_param2)); // C++11
auto mybuffer = std::make_unique<char[]>(length); // C++14
auto mybuffer = std::unique_ptr<char[]>(new char[length]); // C++11

// or for reference counted objects
auto myobj = std::make_shared<MyClass>(); 

// ...
// myobj is automatically freed for you whenever it is no longer used.
```

#### 25.用std::array 或 std::vector 代替C风格的数组

这两个都使用内存连续分配的，可以完全代替C数组。如果想获得最原生C数组的性能，可以用std::array，它还可以结合STL的算法，非常方便。另外，避免使用std::shared_ptr来指向一个数组，用std::unique_ptr代替。

#### 26.使用C++风格的转换代替C风格的转换

使用C++风格的转换(static_cast<> dynamic_cast<>)代替C风格的转换,C++风格的转换编译器会加入更多的检查，类型更加安全。

``` cpp
// Bad Idea
double x = getX();
int i = (int) x;

// Not a Bad Idea
int i = static_cast<int>(x);
```

#### 27.不要定义可变参函数

类似printf,可变参的函数不是类型安全的，错误的输入可能导致异常终止，或者让程序产生未定义行为，未定义行为会导致软件的安全问题，如果你有支持C++ 11的编译器，请使用变参模版来代替。

附加内容： [如何防止下一个类似HeartBleed的漏洞](https://www.dwheeler.com/essays/heartbleed.html)

#### 28.避免使用宏

宏是预处理器(preprocessor)做的事，这个事件在编译器开始编译代码之前，这会让编译器少做了类型检查，因为宏本质是个文本替换，如果出现BUG，调试器很难找出问题源头所在。

``` cpp
// Bad Idea
#define PI 3.14159;

// Good Idea
// 类似于Java，Java全局常量也必须定义在类中
namespace my_project {
  class Constants {
  public:
    // 如果上面这段宏被展开了，那么实际会变成下面这行:
    //   static const double 3.14159 = 3.14159;
    //这会导致编译出错. 有些时候，这样的错误很难理解.
    static const double PI = 3.14159;
  };
}
```

#### 29.函数参数尽量避免bool类型

这对于阅读代码来说，不会增加更多有意义的信息，有了bool变量，就说明函数的输出依赖了这个条件，导致做了过多的事情，一个函数尽量保证只做一件事并做好一件事。如果遇到需要这样写的函数，就把bool的条件分离出一个单独的函数来处理,或者传入枚举类型，让参数更有意义。

[这篇文章](https://mortoray.com/2015/06/15/get-rid-of-those-boolean-function-parameters/)讲述了更多的细节，以及如何解决。

#### 30.尽量避免裸循环

使用C++ 新标准提供的更加高级的语义。

``` cpp

std::vector<DeviceCollect::st_port> tcp_ports;

// Very Bad Idea
for(int i = 0; i < tcp_ports.size(); ++i)
{
  //do_something
  tcp_ports[i];
}

// range for , foreach semantics
// Not a Bad Idea
for(const auto & prot : tcp_ports)
{
    //do_something
    port;
}


// Very Good Idea
tcp_ports.erase(
		std::remove_if(std::begin(tcp_ports), std::end(tcp_ports),
		[](const DeviceCollect::st_port & o) -> bool { return o.processPath.compare(std::string("空")) == 0; }),
		std::end(tcp_ports));
```

#### 31.正确使用关键字override和final

这些关键字能帮助开发者更清晰的理解虚函数是怎样被使用的，如果虚函数的函数签名有变化，override关键字也能捕获一些潜在的错误，导致编译出错。也能[提示编译器](http://stackoverflow.com/questions/7538820/how-does-the-compiler-benefit-from-cs-new-final-keyword)如何更好的优化虚函数。

#### 32.了解使用的类型

这个在条款15也提到过，这里会给出一个[更详细的参考](https://www.viva64.com/en/a/0010/)。

#### 33.避免全局数据

全局数据会导致函数间的无法意料的副作用，也会导致代码间难以并行化，或者不可能并行化。

#### 34.避免静态变量

除了全局数据，静态数据的构造和析构可能不总是能按照你预想的进行。如果你的程序是跨平台的，那么这个噩梦可能成真，请看这么个例子，一个[g++的BUG](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=66830)引出了共享静态数据析构顺序的一个问题。这个共享静态数据是从动态库中加载的。

#### 35.shared_ptr

该指针[几乎等同于全局变量](http://stackoverflow.com/questions/18709647/shared-pointer-to-an-immutable-type-has-value-semantics/)，因为它可以让不同的代码访问相同的数据。

#### 36.单例模式

单例模式一般都需要保证线程安全，因为单例一般实现是static变量和shared_ptr实现的。


#### 37.互斥量和可变量要一起使用

- 一个可变成员变量如果会被共享就需要被互斥量同步(mutex)，或者把可变量做成原子的（atomic）
- 如果一个成员变量本身是一个互斥量，它也是可变的，要使用它，必须在const 函数中使用

更多详情，请看[这里](https://herbsutter.com/2013/05/24/gotw-6a-const-correctness-part-1-3/)。

#### 38.需要的时候就前置声明

用这段代码：
``` cpp
// some header file
class MyClass;
class MyClassB

void doSomething(const MyClass &);
void fuck(MyClassB *);
```
代替以下代码：
``` cpp
// some header file
#include "MyClass.hpp"
#include "MyClassB.hpp"

void doSomething(const MyClass &);
void fuck(MyClassB *);
```

如果类是模版类，可以这么写：
``` cpp
template<typename T> class MyTemplatedType;
```

这样在编译器重新构建依赖的时候，可以有效降低编译时间。

#### 39.避免不必要的模版实例化

模版实例化不是没有代价的，过多的模版实例化，每种类型实例化为一份代码，那么会造成编译出的机器码膨胀，也会加大编译时间。

更多详情，请看[这篇文章](http://articles.emptycrate.com/2015/04/27/template_code_bloat_revisited_a_smaller_makeshared.html)。

#### 40.避免递归模版实例化

递归会让模版实例化瞬间变多，加重编译器负担,造成更加难以理解的代码。

问题解决请看[这篇文章](http://articles.emptycrate.com/2016/05/14/folds_in_cpp11_ish.html)。

#### 41.不要包含无用的头文件

降低编译依赖，减少编译时间。

#### 42.使用初始化列表

``` cpp
// This
std::vector<ModelObject> mos{mo1, mo2};

// -or-
auto mos = std::vector<ModelObject>{mo1, mo2};

// Don't do this
std::vector<ModelObject> mos;
mos.push_back(mo1);
mos.push_back(mo2);
```
初始化列表能带来更大性能的提升，减少对象的拷贝和容器的resize

#### 43.使用move语义

move语义是C++ 11的最大革新了。

相关文章:
- [关于C++右值及std::move()的疑问](https://www.zhihu.com/question/50652989)
- [C++ 中的「移动」在内存或者寄存器中的操作是什么，为什么就比拷贝赋值性能高呢](https://www.zhihu.com/question/55735384)
- [如何理解C++中的move语义](https://www.zhihu.com/question/43513150)

对于大多数代码，只要这样就可以了:
``` cpp
ModelObject(ModelObject &&) = default;
```
但是MSVC2013不支持，MSVC2015支持。

#### 44.减少shared_ptr的拷贝

该智能指针允许共享，类似实现Java中的引用计数来保证无资源泄漏，但是拷贝的代价很高，yi你用计数必须是原子的和线程安全的。

#### 45.尽可能降低拷贝或重新赋值

``` cpp
// Bad Idea
std::string somevalue;

if (caseA) {
  somevalue = "Value A";
} else {
  somevalue = "Value B";
}

// Better Idea
const std::string somevalue = caseA ? "Value A" : "Value B";
```

更加复杂的场景可以这样：
``` cpp
// Bad Idea
std::string somevalue;

if (caseA) {
  somevalue = "Value A";
} else if(caseB) {
  somevalue = "Value B";
} else {
  somevalue = "Value C";
}

// Better Idea
// 这类似与javascript中的匿名函数立即调用
// (function (){ return "hello" })()
const std::string somevalue = [&](){
    if (caseA) {
      return "Value A";
    } else if (caseB) {
      return "Value B";
    } else {
      return "Value C";
    }
  }();
```

IIFE in C++ 11 请看[这里](http://articles.emptycrate.com/2014/12/16/complex_object_initialization_optimization_with_iife_in_c11.html)。

#### 46.不要大量使用异常

这个《Google C++ Coding Style》上有说，这里就不多做解释了。

#### 47.避免使用new

``` cpp
std::shared_ptr<ModelObject_Impl>(new ModelObject_Impl());

// should become
std::make_shared<ModelObject_Impl>(); // (it's also more readable and concise)
```

#### 48. 优先使用unique_ptr

如果能用unique_ptr解决的场景，就不要用shared_ptr。

当前的最佳实践就是，从一个工厂函数中返回一个unique_ptr，如果有需要，再从unique转成shared。

``` cpp
std::unique_ptr<ModelObject_Impl> factory();

auto shared = std::shared_ptr<ModelObject_Impl>(factory());
```

#### 49.限制变量作用范围

变量应该尽可能的推迟声明，什么时候需要用到再声明，不能像C89那样，全部放在函数开头。这样也影响阅读性。降低变量作用范围同时降低内存的使用,帮助编译器生成更高效的代码。

``` cpp
// Good Idea
for (int i = 0; i < 15; ++i)
{
  MyObject obj(i);
  // do something with obj
}

// Bad Idea
MyObject obj; // 无意义的对象初始化
for (int i = 0; i < 15; ++i)
{
  obj = MyObject(i); // 无必要的赋值操作
  // do something with obj
}
// obj还继续占用内存，直到离开它自身的作用范围

```

#### 50.优先使用double，而不是float

当然了，这也得看场景和编译器的优化水平了，double可能比float更高效。
使用float，因为精度更低,所以在转换中可能更低效。当然，如果在向量化操作中，如果你牺牲精度而换取高性能，float可能会更加高效。

double是C++浮点数的默认类型
``` cpp
float a = 3.14f; // 3.14f默认double，会有一层隐式向低精度转换
double b = 3.14f;  // 由于默认是double 无需转换
```

#### 51.优先使用++i而不是i++ （可以废弃）

当然，现代编译器优化水平相当高了，在for循环中，两者优化出来的机器码都是一样的，但是你无法保证在个别的编译器平台上能这样。

#### 52. char是char，string是string

``` cpp
// Bad Idea
std::cout << someThing() << "\n";

// Good Idea
std::cout << someThing() << '\n';
```

前者会被编译器解析成const char*， 当写入流的时候，会检查末尾'\0'结束符。后者被解释成单个字符，省去了CPU很多操作。

如果在特别场景下大量使用前者，可能前者会慢慢变成性能瓶颈。

#### 53.不要使用std::bind

std::bind 在lambda表达式出现之前可以用，但是之后就没有多大用了。大多数情况下，可以用lambda表达式代替。

``` cpp
// Bad Idea
auto f = std::bind(&my_function, "hello", std::placeholders::_1);
f("world");

// Good Idea
auto f = [](const std::string &s) { return my_function("hello", s); };
f("world");
```

#### 54.善用脚本，拥抱其他语言和框架

静态类型语言与动态类型语言结合才更强劲，参考使用boost::python。
该用什么语言完成的工作就用那种语言完成。写Web就用Java拉，前端就javascript啦，不要折腾没用的。

**当你每次鄙视一种语言的时候，就失去了一次向它学习的机会。**
