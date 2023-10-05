

## 文件IO

### FIO50-CPP. 在使用文件流输入输出的时候不要不使用中介定位调用

严重程度:低。如果在使用文件流做输入输出的时候没有调用中介flush和中介定位那么就是未定义行为。

C++标准[filebuf]有如下声明:

> The restrictions on reading and writing a sequence controlled by an object of class
basic_filebuf<charT, traits> are the same as for reading and writing with the
Standard C library FILEs.

当然C++是兼容C的，C标准也有类似以下声明:

> When a file is opened with update mode . . ., both input and output may be performed on
the associated stream. However, output shall not be directly followed by input without an
intervening call to the fflush function or to a file positioning function (fseek,
fsetpos, or rewind), and input shall not be directly followed by output without an
intervening call to a file positioning function, unless the input operation encounters endof-file.

所以，结果以下场景就会导致未定义行为:

- Receiving input from a stream directly following an output to that stream without an intervening call to std::basic_filebuf\<T\>::seekoff() if the file is not at end-offile

- Outputting to a stream after receiving input from that stream without a call to std::basic_filebuf\<T\>::seekoff() if the file is not at end-of-file

#### 代码样例对比

``` cpp
#include <fstream>
#include <string>

void f(const std::string &fileName) {
  std::fstream file(fileName);
  if (!file.is_open()) {
    // Handle error
    return;
  }

  file << "Output some data";
  std::string str;
  file >> str;
}
```
以上代码通过文件流试图把"Output some data"写入文件尾，然后并把该字符串又重新读出到str对象中，然而，这输入输出的中间并没有调用中介定位函数，所以行为是未定义的。所以改成以下:

``` cpp
#include <fstream>
#include <string>

void f(const std::string &fileName) {
  std::fstream file(fileName);
  if (!file.is_open()) {
    // Handle error
    return;
  }

  file << "Output some data";

  std::string str;
  file.seekg(0, std::ios::beg);
  file >> str;
}
```
以上代码通过std::basic_istream\<T\>::seekg()把文件流指针调整到了文件的开头，这样才能读取到写入的字符串。

### FIO51-CPP. 当不需要操作文件的时候一定要关闭文件

严重程度: 中等。容易导致系统资源浪费，造成程序不正常终止。

对std::basic_filebuf\<T\>::open()的调用一定要有std::basic_filebuf\<T\>::close()来关闭文件，至少都要在程序结束之前关闭。 

注意，std::basic_ifstream\<T\>, std::basic_ofstream\<T\>和std::basic_fstream\<T\>这三个类底层都依赖std::basic_filebuf\<T\>对象提供的open和close来打开关闭文件。

#### 代码样例对比

``` cpp
#include <exception>
#include <fstream>
#include <string>

void f(const std::string &fileName) {
  std::fstream file(fileName);
  if (!file.is_open()) {
    // Handle error
    return;
  }
  // ...
  std::terminate();
}
```
以上代码显然有问题，因为用文件流对象打开了一个文件，但是file对象没有正确关闭，因为代码用std::terminate()强制终止了程序，导致file对象的析构函数没有调用，自然就没有正确关闭文件。std::terminate()本质上是调用了std::abort()才导致的问题。

当然，如果非要在f()函数中强制终止程序，那么需要手工保证文件关闭，不能依赖RAII来关闭资源:

``` cpp
#include <exception>
#include <fstream>
#include <string>

void f(const std::string &fileName) {
  std::fstream file(fileName);
  if (!file.is_open()) {
    // Handle error
    return;
  }

  // ...
  file.close(); //在调用terminate之前手工调用close关闭文件
  if (file.fail()) {
    // Handle error
  }
  std::terminate();
}
```

当然，还可以通过RAII机制来关闭文件，这样的方式是C++主推的一种手段，方便安全:

``` cpp
#include <exception>
#include <fstream>
#include <string>

void f(const std::string &fileName) {
  {
    std::fstream file(fileName);
    if (!file.is_open()) {
      // Handle error
      return;
    }
  } // 文件正确关闭，因为文件流对象file到这里退出作用域，自动调用了file对象的析构函数
  std::terminate();
}
```

