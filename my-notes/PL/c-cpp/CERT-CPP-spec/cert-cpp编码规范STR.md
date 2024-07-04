

## 字符串类

### STR50-CPP. 保证字符串类有足够的存储空间预留给字符数据和null结束符

严重程度： 高。造成缓冲区溢出，攻击者运行任意代码。

C风格的字符串数组要求字符串结尾隐式包含一个null结束符('\0')。但是C++的std::basic_string模版类就不需要null结束符。

#### 代码样例对比

``` cpp
//bad, 输入无边界，容器溢出
#include <iostream>

void f() {
  char buf[12];
  std::cin >> buf;
}

////////////////////////////////
// 还是不正确，bufOne不会溢出，但实际BufTwo会溢出
#include <iostream>

void f() {
  char bufOne[12];
  char bufTwo[12];
  std::cin.width(12);
  std::cin >> bufOne;
  std::cin >> bufTwo;
}

/////////////////////////////////////////
// 以下就正确了
#include <iostream>
#include <string>

void f() {
  std::string input;
  std::string stringOne, stringTwo;
  std::cin >> stringOne >> stringTwo;
}
```

#### 代码样例对比

``` cpp
#include <fstream>
#include <string>

void f(std::istream &in) {
  char buffer[32];
  
  try {
    in.read(buffer, sizeof(buffer));
  } catch (std::ios_base::failure &e) {
    // Handle error
  }
  
  std::string str(buffer);
// ...
}
```
以上代码看似没有问题，read函数也传入sizeof了，std::basic_istream\<T\>::read()用来把文件的数据读入32个字节的buffer中，但是read()函数并不保证字符串会以null结束符结尾，所以接下来的调用std::string的构造函数，如果buffer不以null结尾那么会直接导致未定义行为。那么按照以下就好改正了:

``` cpp
#include <fstream>
#include <string>

void f(std::istream &in) {
  char buffer[32];
  
  try {
    in.read(buffer, sizeof(buffer));
  } catch (std::ios_base::failure &e) {
    // Handle error
  }

  std::string str(buffer, in.gcount());
// ...
}
```

相反插入null结束符，以上代码保证了明确的写入32个字符在str中，不用让std::string的构造函数自己遍历字符串判断，因为std::string之前的单参构造函数在没有遇到null结束符之前不会结束，会一直复制数据运行下去。

###  STR51-CPP. 构造std::string对象时，不要传入null指针

严重程度: 高。  解引用空指针是未定义行为，典型的一个现象就是程序非正常终止。

std::basic_string使用了traits的设计模式来处理不同的字符串类型。尤其std::basic_string是配合std::char_traits来创建的std::string, std::wstring, std::u16string和
std::u32string等字符串类，其中std::char_traits::length()函数一般是用来决定在null结尾的字符串中的字符个数的。以下std::basic_string成员函数的调用都会间接导致std::char_traits::length()函数的调用：

``` cpp
basic_string::basic_string(const charT *, const Allocator &)
basic_string &basic_string::append(const charT *)
basic_string &basic_string::assign(const charT *)
basic_string &basic_string::insert(size_type, const charT *)
basic_string &basic_string::replace(size_type, size_type,
const charT *)
basic_string &basic_string::replace(const_iterator,
const_iterator, const charT *)
size_type basic_string::find(const charT *, size_type)
size_type basic_string::rfind(const charT *, size_type)
size_type basic_string::find_first_of(const charT *,
size_type)
size_type basic_string::find_last_of(const charT *, size_type)
size_type basic_string::find_first_not_of(const charT *,
size_type)
size_type basic_string::find_last_not_of(const charT *,
size_type)
int basic_string::compare(const charT *)
int basic_string::compare(size_type, size_type, const charT *)
basic_string &basic_string::operator=(const charT *)
basic_string &basic_string::operator+=(const charT *)
```

以下std::basic_string的非成员函数都会间接导致std::char_traits::length()函数的调用：

``` cpp
basic_string operator+(const charT *, const basic_string&)
basic_string operator+(const charT *, basic_string &&)
basic_string operator+(const basic_string &, const charT *)
basic_string operator+(basic_string &&, const charT *)
bool operator==(const charT *, const basic_string &)
bool operator==(const basic_string &, const charT *)
bool operator!=(const charT *, const basic_string &)
bool operator!=(const basic_string &, const charT *)
bool operator<(const charT *, const basic_string &)
bool operator<(const basic_string &, const charT *)
bool operator>(const charT *, const basic_string &)
bool operator>(const basic_string &, const charT *)
bool operator<=(const charT *, const basic_string &)
bool operator<=(const basic_string &, const charT *)
bool operator>=(const charT *, const basic_string &)
bool operator>=(const basic_string &, const charT *)
```

所以在调用以上提及的函数时不能在const charT* 传入null指针，统统会导致对null指针的解引用。

下面提一下各种标准库的实现细节：

> 如果是stdlibc++ 调用以上函数传入了null指针，会主动抛std::logic_error的异常，但是直接调用std::char_traits::length()函数时就不会。然而，对于这样的情况抛std::loginc_error C++标准并没有作出要求。另一些标准库厂商，libc++和Visual Studio STL的实现上来看，就没有实现上述跑异常的行为，所以你不能对此严重的错误作出任何指望。

#### 代码样例对比

``` cpp
#include <cstdlib>
#include <string>

void f() {
  std::string tmp(std::getenv("TMP"));
  if (!tmp.empty()) {
    // ...
  }
}
```

以上代码试图通过std::getenv得到环境变量的值，但是如果没有此环境变量该函数会返回null指针以表示失败，所以这就导致了std::string的构造函数发生了未定义行为。应该改成以下:

``` cpp
#include <cstdlib>
#include <string>

void f() {
   const char *tmpPtrVal = std::getenv("TMP");
   std::string tmp(tmpPtrVal ? tmpPtrVal : "");
   if (!tmp.empty()) {
     // ...
   }
}
```

### STR52-CPP. 使用合法的引用，指针和迭代器来指向std::basic_string的元素

严重程度：高。

因为std::basic_string本质上就是是字符容器，你可以简单看成用std::vector\<char\>实现了std::basic_string。所以该条规则是CTR51-CPP规则的一条特例。因此不作过多讨论。

### STR53-CPP. 访问字符串元素时检查下标范围

严重程度：高

此条款是CTR50-CPP规则的一条特例，所以不作过多讨论。