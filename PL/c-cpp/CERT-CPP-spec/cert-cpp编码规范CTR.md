---
title: CERT C++编码规范翻译（CTR）
date: 2017-07-19 14:30:46
tags:
    - C/C++
---

## 容器类

### CTR50-CPP. 保证容器的索引和迭代器在合法的范围内

严重程度： 高。 会导致任意内存地址的数据被覆盖，从而导致程序不正常终止。

这个主要内容没什么可以说明的。越界一定是开发人员的错。

#### 代码样例对比

``` cpp
#include <cstddef>

void insert_in_table(int *table, std::size_t tableSize, int pos, int value) {
  if (pos >= tableSize) {
    // Handle error
    return;
  }
  table[pos] = value;
}
```

以上的代码实现是需要在一个table的特定下标插入一个值。函数还有个下标检测的越界判断，看似合理。但是pos参数被声明了为int，int类型是默认有符号数，而tableSize的类型是无符号的std::size_t，这两个类型比较在某些极端情况下会失效。一旦pos被不小心赋值为负数，那么if判断失效，导致table访问越界。所以改成以下为好:

``` cpp
#include <cstddef>
void insert_in_table(int *table, std::size_t tableSize, std::size_t
pos, int value) {
if (pos >= tableSize) {
// Handle error
return;
}
table[pos] = value;
}
```
pos最好也声明为std:size_t，这样就可以防止负数被传入进函数了。

还可以用以下办法:

``` cpp
#include <cstddef>
#include <new>

void insert_in_table(int *table, std::size_t tableSize, std::size_t pos, int value) {
  // #1
  if (pos >= tableSize) {
    // Handle error
    return;
  }
  table[pos] = value;
}

template <std::size_t N>
void insert_in_table(int (&table)[N], std::size_t pos, int value) {
  // #2
  insert_in_table(table, N, pos, value);
}

void f() {
  // Exposition only
  int table1[100];
  int *table2 = new int[100];
  insert_in_table(table1, 0, 0); // Calls #2
  insert_in_table(table2, 0, 0); // Error, no matching func. call
  insert_in_table(table1, 100, 0, 0); // Calls #1
  insert_in_table(table2, 100, 0, 0); // Calls #1
  delete [] table2;
}
```

以上代码是使用无类型模版的手段把数组越界检测提前到编译时。

#### 代码样列对比(std::vector)

``` cpp
#include <vector>

void insert_in_table(std::vector<int> &table, long long pos, int value) {
  if (pos >= table.size()) {
    // Handle error
    return;
  }
  table[pos] = value;
}
```
以上的代码与之前一个的代码样例中所反映的问题是一样的。long long类型的pos是有符号类型，比较可能失效，导致越界。应该改成:

``` cpp
#include <vector>

void insert_in_table(std::vector<int> &table, std::size_t pos, int value) {
  if (pos >= table.size()) {
    // Handle error
    return;
  }
  table[pos] = value;
}
```

其实还可以巧妙利用vector的at成员函数来访问特定下标，这个成员函数提供越界检测，越界会抛出std::out_of_range的异常：

``` cpp
#include <vector>

void insert_in_table(std::vector<int> &table, std::size_t pos, int value) noexcept(false) {
  table.at(pos) = value;
}
```

至于insert_in_table 之后声明有noexcept(false)，就是表明该函数可能抛出异常，遵循C++的Honor Exception Spec。 当然不写也可以，但是最好是写上。

#### 代码样例对比(iterators)

``` cpp
#include <iterator>

template <typename ForwardIterator>
void f_imp(ForwardIterator b, ForwardIterator e, int val,std::forward_iterator_tag) {
  do {
    *b++ = val;
  } while (b != e);
}

template <typename ForwardIterator>
void f(ForwardIterator b, ForwardIterator e, int val) {
  typename std::iterator_traits<ForwardIterator>::iterator_category cat;
  f_imp(b, e, val, cat);
}
```

以上对于f_imp函数通过迭代器访问一个容器，参数e是end迭代器，假设e永远是传入正确的，但是也会造成迭代器解引用错误，因为一旦容器是空的，参数b就等于e。但是do while循环是先计算，后比较。这就引入问题了，所以得改成以下:

``` cpp
#include <iterator>

template <typename ForwardIterator>
void f_imp(ForwardIterator b, ForwardIterator e, int val,std::forward_iterator_tag) {
  while (b != e) {
    *b++ = val;
  }
}

template <typename ForwardIterator>
void f(ForwardIterator b, ForwardIterator e, int val) {
  typename std::iterator_traits<ForwardIterator>::iterator_category cat;
  f_imp(b, e, val, cat);
}
```
先比较迭代器的合法性，再进行解引用b。

### CTR51-CPP. 使用合法的引用，指针，迭代器来引用容器的元素

严重等级：高。一旦持有一个不合法的引用并进行访问容器，那么就直接导致未定义行为。

迭代器就是指针的泛化。它允许通过统一的方式来访问不同的数据结构容器。

C++标准[container.requirements.general]有如下声明：

> Unless otherwise specified (either explicitly or by defining a function in terms of other functions), invoking a container member function or passing a container as an argument to a library function shall not invalidate iterators to, or change the values of, objects within that container.

也就是说，C++标准是允许引用和指针无效化的，当你通过容器类提供的操作函数。举个例子，当你从容器中得到一个指向某元素的指针，然后erase了那个元素，然后又在删除元素的位置insert一个新的元素，就会导致现存的指针虽然合法，但是指向了不同的对象。所以任何操作都可能会使指针或引用无效化，要慎重对待。

以下列出容器的哪些操作会使引用，指针，迭代器无效化。

- std::queue  insert() emplace_front() emplace_back() emplace() push_front() push_back() erase() pop_back() resize() clear()

- std::forward_list erase_after() pop_front() resize() remove() unique() clear()

- std::list earse() pop_front() pop_back() clear() remove() remove_if() unique() 

- std::vector reserve() insert() emplace_back() emplace() push_back() erase() pop_back() resize() clear()

- std::set,std::multiset std::map std::multimap earse() clear()

- std::unordered_set std::unordered_multiset std::unordered_map std::unordered_multimap erase() clear() insert() emplace() rehash() reserve() 

- std::valarray resize()

#### 代码样例对比

``` cpp
#include <deque>

void f(const double *items, std::size_t count) {
  std::deque<double> d;
  auto pos = d.begin();
  for (std::size_t i = 0; i < count; ++i, ++pos) {
    d.insert(pos, items[i] + 41.0);
  }
}
```
以上代码在第一次调用insert的时候pos迭代器失效了，所以就导致后续的循环导致未定义行为。可以通过插入后更新失效迭代器杜绝未定义行为:

``` cpp
#include <deque>

void f(const double *items, std::size_t count) {
  std::deque<double> d;
  auto pos = d.begin();
  for (std::size_t i = 0; i < count; ++i, ++pos) {
    pos = d.insert(pos, items[i] + 41.0);
  }
}
```

还有一种通过泛型算法的方案来解决迭代器失效的问题:

``` cpp
#include <algorithm>
#include <deque>
#include <iterator>

void f(const double *items, std::size_t count) {
  std::deque<double> d;
  std::transform(items, items + count, std::inserter(d, d.begin()),
    [](double d) { return d + 41.0; });
}
```

### CTR52-CPP. 保证库函数不溢出

严重程度: 高。buffer overflow会导致攻击者经过一定的构造条件执行任意的恶意代码

把数据拷贝到不足够放下数据的容器会导致buffer overflow，覆写不合法的内存区块。避免这样的问题，需要限制目标容器的size，最好与数据的size一样大。或者更大一点也可以。

在C语言时代，std::memcpy std::memmove std::memset都可能导致内存区块被覆盖，而且它们不检测内存区块的合法性。所以建议使用C++ STL中提供的std::copy std::fill std::transform等函数来操作,尽管使用C++ 的STL也可以同样导致buffer overflow。

#### 代码样例对比

``` cpp
#include <algorithm>
#include <vector>

void f(const std::vector<int> &src) {
  std::vector<int> dest;
  std::copy(src.begin(), src.end(), dest.begin());
  // ...
}
```

虽然dest是动态数组会随着push append增加存储空间，但是上面代码使用了std::copy，该函数不会扩展dest的空间，所以在拷贝src中第一个元素的时候，dest就buffer overflow了。

所以就有了以下方案解决，初始化dest的时候就把它的存储空间扩大到与src一样：

``` cpp
#include <algorithm>
#include <vector>

void f(const std::vector<int> &src) {
  // Initialize dest with src.size() default-inserted elements
  std::vector<int> dest(src.size());
  std::copy(src.begin(), src.end(), dest.begin());
  // ...
}
```

当然，还有一个方案，就是用std::back_insetr_iterator作为目标参数。这个迭代器它会根据std::copy这个算法一个一个拷贝元素的时候，自动扩展复制目标容器，这就保证目标容器与源容器一样大。

``` cpp
#include <algorithm>
#include <iterator>
#include <vector>

void f(const std::vector<int> &src) {
  std::vector<int> dest;
  std::copy(src.begin(), src.end(), std::back_inserter(dest));
  // ...
}
```

最简单的是用vector提供的拷贝构造函数:

``` cpp
#include <vector>

void f(const std::vector<int> &src) {
  std::vector<int> dest(src);
  // ...
}
```

#### 代码样例对比

``` cpp
#include <algorithm>
#include <vector>

void f() {
  std::vector<int> v;
  std::fill_n(v.begin(), 10, 0x42);
}
```

以上这个代码意图让填充10个0x42在vector中，但是vector默认没分配空间，这样直接填充就造成buffer overflow。改成以下方式：

``` cpp
#include <algorithm>
#include <vector>

void f1() {
  std::vector<int> v(10);
  std::fill_n(v.begin(), 10, 0x42);
}

/////////////////

void f2() {
  std::vector<int> v(10, 0x42);
}
```

### CTR53-CPP. 使用合法的迭代器范围

严重程度： 高。造成buffer overflow，导致运行任意代码

当迭代器遍历一个容器的时候，迭代器必须处于一个合法范围，一个迭代器的范围是一对迭代器，其中一个指向第一个元素，另一个指向最后一个元素的后一位。这两个迭代器形成的范围，就是合法的迭代器范围。

一个合法的迭代器范围需要包含以下所有特征：

- 所有的迭代器必须指向同一个容器
- 迭代器指向容器的开始和结尾

一个空的迭代器范围也是合法的（开始迭代器和尾部迭代器相等）

使用两个迭代器分别指向不同的容器，会导致未定义行为。

#### 代码样例对比

``` cpp
#include <algorithm>
#include <iostream>
#include <vector>

void f(const std::vector<int> &c) {
  std::for_each(c.end(), c.begin(), [](int i) { std::cout << i; });
}
```

以上代码看似正确了，但是for_each的第一参数需要是容器的begin迭代器，而它设成了end。整个迭代器反了。而且end迭代器指向的是最后一个元素的后一位，一旦解引用，立即造成未定义行为。应该改成以下正确的迭代顺序:

``` cpp
#include <algorithm>
#include <iostream>
#include <vector>

void f(const std::vector<int> &c) {
  std::for_each(c.begin(), c.end(), [](int i) { std::cout << i; });
}
```

如果你非得需要反向迭代，那么可以这样：

``` cpp
#include <algorithm>
#include <iostream>
#include <vector>

void f(const std::vector<int> &c) {
  std::for_each(c.rbegin(), c.rend(), [](int i) { std::cout << i; });
}
```

#### 代码样例对比

``` cpp
#include <algorithm>
#include <iostream>
#include <vector>

void f(const std::vector<int> &c) {
  std::vector<int>::const_iterator e;
  std::for_each(c.begin(), e, [](int i) { std::cout << i; });
}
```

以上代码明显这两个迭代器指向了不同的容器，但是由于STL的实现原因，编译器检测不出来这样的语义错误，没有任何的标准表明迭代器e会初始化为容器c的end()迭代器。这个知道原因后，就知道怎么改了，改成以上的代码样例对比就可以了，不作示范。

### CTR54-CPP. 没有指向相同容器的迭代器之间不要做减法

严重程度：中等。

这个与指针本质一样。如果指针指向不同的对象数组，它们之间也不能做减法。类似std:distance就是两个迭代器做减法，需要迭代器指向同一个容器。如果不这么做，那么会直接导致未定义行为。

#### 代码样例对比

``` cpp
#include <cstddef>
#include <iostream>

template <typename Ty>
bool in_range(const Ty *test, const Ty *r, size_t n) {
  return 0 < (test - r) && (test - r) < (std::ptrdiff_t)n;
}

void f() {
  double foo[10];
  double *x = &foo[0];
  double bar;
  std::cout << std::boolalpha << in_range(&bar, x, 10);
}
```
以上代码意图测试指针test，是否在[r,r+n]这个迭代器范围内，然而test并没有指向这个合法的范围容器中，所以test与r相减导致未定义行为。

``` cpp
#include <iostream>

template <typename Ty>
bool in_range(const Ty *test, const Ty *r, size_t n) {
  return test >= r && test < (r + n);
}

void f() {
  double foo[10];
  double *x = &foo[0];
  double bar;
  std::cout << std::boolalpha << in_range(&bar, x, 10);
}
```

以上代码试图整改，用比较运算符让test和r不必做减法，但是还是有问题的。因为C++标准有以下描述:

> If two operands p and q compare equal, p<=q and p>=q both yield true and p\<q and p>q both yield false. Otherwise, if a pointer p compares greater than a pointer q, p>=q, p>q, q\<=p, and q<p all yield true and p<=q, p<q, q>=p, and q>p all yield false. Otherwise, the result of each of the operators is unspecified.

所以比较两个不指向同一个容器的指针会导致未指定行为(unspecified hebavior)。尽管与之前的代码有所改善,但是还是不会产生可移植性的代码，可能在其他的硬件平台上就会失败(x86以外的平台)。然后我们再来改进以上代码。

``` cpp
#include <iostream>
#include <iterator>
#include <vector>

template <typename RandIter>
bool in_range_impl(RandIter test, RandIter r_begin, RandIter r_end,
  std::random_access_iterator_tag) {
  return test >= r_begin && test < r_end;
}

template <typename Iter>
bool in_range(Iter test, Iter r_begin, Iter r_end) {
  typename std::iterator_traits<Iter>::iterator_category cat;
  return in_range_impl(test, r_begin, r_end, cat);
}

void f() {
  std::vector<double> foo(10);
  std::vector<double> bar(1);
  std::cout << std::boolalpha << in_range(bar.begin(), foo.begin(), foo.end());
}
```

以上代码几乎等价于之前的代码片段，仅仅只是用迭代器代替了裸指针，当两个迭代器没有引用相同的容器并且互相进行比较运算的时候，还是会出现未指定行为。下面我们还是再来改进以上代码：

``` cpp
#include <functional>
#include <iostream>

template <typename Ty>
bool in_range(const Ty *test, const Ty *r, size_t n) {
  std::less<const Ty *> less;
  return !less(test, r) && less(test, r + n);
}

void f() {
  double foo[10];
  double *x = &foo[0];
  double bar;
  std::cout << std::boolalpha << in_range(&bar, x, 10);
}
```

以上代码用STL中的std::less<>来代替比较操作运算符。但是C++标准[comparisons]有如下声明:

>  For templates greater, less, greater_equal, and less_equal, the specializations for any pointer type yield a total order, even if the built-in operators <, >, <=, >= do not.

也就是这种方法是依赖实现的，实现不确定。结果是，以上代码还是没有移植性，std::less<>中如果还是有比较运算，那么同样会导致未定义行为。下面我们用另一种不会出现任何问题的方法来达到我们的目的，无非就是损失点性能。

``` cpp
#include <iostream>

template <typename Ty>
bool in_range(const Ty *test, const Ty *r, size_t n) {
  auto *cur = reinterpret_cast<const unsigned char *>(r);
  auto *end = reinterpret_cast<const unsigned char *>(r + n);
  auto *testPtr = reinterpret_cast<const unsigned char *>(test);
  for (; cur != end; ++cur) {
    if (cur == testPtr) {
      return true;
    }
  }
  return false;
}

void f() {
  double foo[10];
  double *x = &foo[0];
  double bar;
  std::cout << std::boolalpha << in_range(&bar, x, 10);
}
```

注意以上代码就完美可移植了，绝对不会出现任何问题。它用了遍历容器指针的值与要测试的指针的值一一做对比，并且转换成了unsigned char指针类型，用的是\==比较操作符，没有用大于小于比较。这样就不会出问题了。

### CTR55-CPP. 如果会导致溢出就不要在一个迭代器上做加法操作

严重程度：高。后果导致攻击者可以通过溢出制造运行任何代码的攻击。

C++标准[expr.add]有如下描述：

> If both the pointer operand and the result point to elements of the same array object, or one past the last element of the array object, the evaluation shall not produce an overflow; otherwise, the behavior is undefined.

因为迭代器是指针的泛化。所以通过在迭代器上做加法也有与指针一样的限制，C++标准[iterator.requirement.general]有如下声明：

> Just as a regular pointer to an array guarantees that there is a pointer value pointing past the last element of the array, so for any iterator type there is an iterator value that points past the last element of a corresponding sequence. These values are called pastthe-end values. Values of an iterator i for which the expression *i is defined are called dereferenceable. The library never assumes that past-the-end values are dereferenceable.

#### 代码样例对比

``` cpp
#include <iostream>
#include <vector>

void f(const std::vector<int> &c) {
  for (auto i = c.begin(), e = i + 20; i != e; ++i) {
    std::cout << *i << std::endl;
  }
}
```
以上代码因为不知道vector的size，就在begin位置偏移了20个单位，天晓得vector运行到f函数的时候是多大，所以应该改成以下：

``` cpp
#include <algorithm>
#include <vector>

void f(const std::vector<int> &c) {
  const std::vector<int>::size_type maxSize = 20;
  for (auto i = c.begin(), e = i + std::min(maxSize, c.size()); i != e; ++i) {
    // ...
  }
}
```

以上代码就比较了vector和意图遍历的大小20，以防止溢出。

### CTR56-CPP. 不要用指针在多态对象上进行算数运算

严重程度：高。导致内存冲突，并且这个也会造成攻击者制造运行任何代码的漏洞。

C++标准中[expt.add]有关于指针算数运算的定义：

> For addition or subtraction, if the expressions P or Q have type “pointer to cv T”, where T
is different from the cv-unqualified array element type, the behavior is undefined. [Note:
In particular, a pointer to a base class cannot be used for pointer arithmetic when the
array contains objects of a derived class type. —end note]

指针算数没有办法知道多态对象的size，试图在多态对象上执行指针算数运算的会导致未定义行为。

C++标准[sxpr.sub]中还说了，数据加小标其实本质就是指针算数运算：

> The expression E1[E2] is identical (by definition) to *((E1)+(E2)).

所以在多态对象上不要用指针算数也不要以数组下标的形式访问。

#### 代码样例对比

``` cpp
#include <iostream>
// / ... definitions for S, T, globI, globD ...
int globI;
double globD;

struct S {
  int i;
  S() : i(globI++) {}
};

struct T : S {
  double d;
  T() : S(), d(globD++) {}
};

void f(const S *someSes, std::size_t count) {
  for (const S *end = someSes + count; someSes != end; ++someSes) {
    std::cout << someSes->i << std::endl;
  }
}

int main() {
  T test[5];
  f(test, 5);
}
```

以上的代码someSes + count 明显违反了在多态对象上使用指针算数运算了，所以会造成未定义行为。

``` cpp
#include <iostream>
// ... definitions for S, T, globI, globD ...
int globI;
double globD;

struct S {
  int i;
  S() : i(globI++) {}
};

struct T : S {
  double d;
  T() : S(), d(globD++) {}
};

void f(const S *someSes, std::size_t count) {
  for (std::size_t i = 0; i < count; ++i) {
    std::cout << someSes[i].i << std::endl;
  }
}

int main() {
  T test[5];
  f(test, 5);
}
```

以上的代码用数组下标的方式运用了在多态对象上，因为指针做加法，不知道多态对象的大小，没有正确取得偏移量，所以也会导致未定义行为。需要改成以下：

``` cpp
#include <iostream>
// ... definitions for S, T, globI, globD ...
int globI;
double globD;

struct S {
  int i;
  S() : i(globI++) {}
};

struct T : S {
  double d;
  T() : S(), d(globD++) {}
};

void f(const S * const *someSes, std::size_t count) {
  for (const S * const *end = someSes + count; someSes != end; ++someSes) {
    std::cout << (*someSes)->i << std::endl;
  }
}

int main() {
  S *test[] = {new T, new T, new T, new T, new T};
  f(test, 5);
  for (auto v : test) {
    delete v;
  }
}
```

以上代码无懈可击了，使用多态对象的指针数组来代替多态对象数组，因为指针的size都是一样的，可以在指针上做指针算数运算了，不会出现未定义行为。或者可以用迭代器的方式来解决本质是一样的：

``` cpp
#include <iostream>
#include <vector>
// ... definitions for S, T, globI, globD ...
template <typename Iter>
void f(Iter i, Iter e) {
  for (; i != e; ++i) {
    std::cout << (*i)->i << std::endl;
  }
}

int main() {
  std::vector<S *> test{new T, new T, new T, new T, new T};
  f(test.cbegin(), test.cend());
  for (auto v : test) {
    delete v;
  }
}
```

本质上还是指向多态对象的指针的vector，一样的。

### CTR57-CPP. 提供合法的顺序谓词(predicate)

严重程度：低。可能导致错误行为或者无限循环.


#### 代码样例对比

``` cpp
#include <functional>
#include <iostream>
#include <set>

void f() {
  std::set<int, std::less_equal<int>> s{5, 10, 20};
  for (auto r = s.equal_range(10); r.first != r.second; ++r.first)
  {
    std::cout << *r.first << std::endl;
  }
}
```

以上代码试图通过一个比较谓词来创建一个set，但是呢，这个比较谓词并没有采用严格弱顺序的要求，所以，在对于值相等的时候返回false会失败。结果就时当遍历开始的时候，std::set::equal_range就会导致未指定行为。

``` cpp
#include <iostream>
#include <set>

void f() {
  std::set<int> s{5, 10, 20};
  for (auto r = s.equal_range(10); r.first != r.second; ++r.first)
  {
    std::cout << *r.first << std::endl;
  }
}
```

以上代码就完全没问题了，采用了set默认提供的比较谓词。

#### 代码样例对比

``` cpp
#include <iostream>
#include <set>

class S {
 int i, j;
 
public:
  S(int i, int j) : i(i), j(j) {}
  friend bool operator<(const S &lhs, const S &rhs) 
  {
    return lhs.i < rhs.i && lhs.j < rhs.j;
  }

  friend std::ostream &operator<<(std::ostream &os, const S& o) 
  {
     os << "i: " << o.i << ", j: " << o.j;
     return os;
  }
};

void f() {
  std::set<S> t{S(1, 1), S(1, 2), S(2, 1)};
  for (auto v : t) {
    std::cout << v << std::endl;
  }
}
```

以上代码试图把S类型的对象存储进set中，并且提供了一个重载的operator < 实现，运行对象间的std::less比较。然而，比较操作没有提供一个严格的弱顺序。需要改成以下形式：

``` cpp
#include <iostream>
#include <set>
#include <tuple>

class S {
  int i, j;
public:
  S(int i, int j) : i(i), j(j) {}
  
  friend bool operator<(const S &lhs, const S &rhs) {
    return std::tie(lhs.i, lhs.j) < std::tie(rhs.i, rhs.j);
  }
  
  friend std::ostream &operator<<(std::ostream &os, const S& o) {
    os << "i: " << o.i << ", j: " << o.j;
    return os;
  }
};

void f() {
  std::set<S> t{S(1, 1), S(1, 2), S(2, 1)};
  for (auto v : t) {
    std::cout << v << std::endl;
  }
}
```
以上代码用std::tie来恰当的实现了严格弱顺序的operator < 谓词。

### CTR58-CPP. 谓词函数对象应该不能被修改

严重程度：低。谓词函数对象的状态可能会产生无法预料的值。

C++标准库中大量的STL算法可能都允许接收一个谓词函数对象，在C++标准[algorithms.general]有如下描述：

> [Note: Unless otherwise specified, algorithms that take function objects as arguments are permitted to copy those function objects freely. Programmers for whom object identity is important should consider using a wrapper class that points to a noncopied implementation object such as reference_wrapper\<T\>, or some equivalent solution. — end note]

因为这些STL算法是实现定义的，没办法保证算法是否拷贝了谓词函数对象。

#### 代码样例对比

``` cpp
#include <algorithm>
#include <functional>
#include <iostream>
#include <iterator>
#include <vector>

class MutablePredicate : public std::unary_function<int, bool> {
  size_t timesCalled;
public:
  MutablePredicate() : timesCalled(0) {}
  bool operator()(const int &) {
    return ++timesCalled == 3;
  }
};

template <typename Iter>
void print_container(Iter b, Iter e) {
  std::cout << "Contains: ";
  std::copy(b, e, std::ostream_iterator<decltype(*b)>
                                    (std::cout, " "));
  std::cout << std::endl;
}

void f() {
  std::vector<int> v{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
  print_container(v.begin(), v.end());
  v.erase(
   std::remove_if(v.begin(), v.end(), MutablePredicate()),
   v.end());
   print_container(v.begin(), v.end());
}
```

以上代码在std::remove_if的时候传入了自定义的谓词函数对象，企图删除当谓词对象被调用第三次的时候的vector的元素。但是，意料之外，std::remove_if允许在谓词对象构造的时候是使用谓词对象的拷贝，也就是，可能是多个对象拷贝，这样就会导致结果可能不是你所预想。

在GCC 4.8.1 标准库是libstdc++的时候，以上代码会有以下输出：

```
Contains: 0 1 2 3 4 5 6 7 8 9
Contains: 0 1 3 4 6 7 8 9
```
看以上输出，把元素2和5删除了，不如我们所想。原因是std::remove_if生成了2个谓词函数的副本在调用谓词函数对象之前。这就导致了副本1号删除了对于它的第三次调用的元素，这个元素是2，意料之中。副本2号删除了对于它的第三次调用的元素，这个元素是5，也是意料之中。 所以居然删除了2个元素。

当然，如果你不想用自定义的谓词函数对象，那么也可以用lambda表达式实现，其实lambda表达式也是对象，与自定义谓词对象是一样的：

``` cpp
#include <algorithm>
#include <iostream>
#include <iterator>
#include <vector>

template <typename Iter>
void print_container(Iter b, Iter e) {
  std::cout << "Contains: ";
  std::copy(b, e,
  std::ostream_iterator<decltype(*b)>(std::cout, " "));
  std::cout << std::endl;
}

void f() {
  std::vector<int> v{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
  print_container(v.begin(), v.end());
  int timesCalled = 0;
  
  v.erase(std::remove_if(
          v.begin(),
          v.end(),
          [timesCalled](const int &) mutable {
              return ++timesCalled == 3;
          }),
          v.end());
  
  print_container(v.begin(), v.end());
}
```

其实本质和之前的代码也是一样的，只不过用lambda表达式简化了而已，还是会出现之前的问题，因为std::remove_if函数实现你没办法保证它构造了几个lambda函数对象的副本。如果是2个，那么与之前的代码输出是一样的。所以要构造一个谓词函数对象，并且强制让remove_if采用这一个对象，需要改成以下：

``` cpp
#include <algorithm>
#include <functional>
#include <iostream>
#include <iterator>
#include <vector>

class MutablePredicate : public std::unary_function<int, bool> {
  size_t timesCalled;
public:
  MutablePredicate() : timesCalled(0) {}
  bool operator()(const int &) {
    return ++timesCalled == 3;
  }
};

template <typename Iter>
void print_container(Iter b, Iter e) {
  std::cout << "Contains: ";
  std::copy(b, e,
  std::ostream_iterator<decltype(*b)>(std::cout, " "));
  std::cout << std::endl;
}

void f() {
  std::vector<int> v{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
  print_container(v.begin(), v.end());
  MutablePredicate mp;
  
  v.erase(
         std::remove_if(v.begin(), v.end(), std::ref(mp)),
         v.end());

  print_container(v.begin(), v.end());
}
```

就是手动构造一个谓词对象，然后用std::ref处理后传递进去，这个就保证调用前只有一个对象副本，程序输出就是正确结果了,恰好是第三次调用时候的元素2被删除：

```
Contains: 0 1 2 3 4 5 6 7 8 9
Contains: 0 1 3 4 5 6 7 8 9
```