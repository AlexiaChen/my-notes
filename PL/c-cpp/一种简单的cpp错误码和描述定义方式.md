#cpp

我们平时有这样的需求，可能是C语言用户的老习惯了，在底层的组件中更喜欢用返回错误码的形式来告知函数的调用结果，一般来说，简单用#define 一个宏来包装下返回值。
``` c
#define ERR_SYSTEM_INIT -23 // system initailized fail
```
以上定义了一个错误码返回-23，意味着系统初始化失败。但是宏包含的信息太少，有些时候，用户不能知道错误的详细原因。必须给这错误加以说明。所以就索性写了一个Error类可以定义错误码和相关错误描述信息，并通过类似于Win32 API **GetLastError**这样的函数让错误码返回更详细的说明：

``` c
#include <string>
#include <map>
#include <cassert>

class Error
{
  public:
   Error(int value, const std::string& str)
   {
     m_value = value;
     m_message =    str;
#ifdef _DEBUG
     ErrorMap::iterator found = GetErrorMap().find(value);
     if (found != GetErrorMap().end())
     assert(found->second == m_message);
#endif
     GetErrorMap()[m_value] = m_message;
   }

// auto-cast Error to integer error code
   operator int() { return m_value; }
  private:
    int m_value;
    std::string m_message;

    typedef std::map<int, std::string> ErrorMap;
    static ErrorMap& GetErrorMap()
    {
       static ErrorMap errMap;
       return errMap;
    }

 public:

    static std::string GetErrorString(int value)
    {
       ErrorMap::iterator found = GetErrorMap().find(value);
       if (found == GetErrorMap().end())
       {
          assert(false);
          return "";
       }
       else
       {
          return found->second;
       }
    }   
};
```

以下是该类的使用方法：

``` c
#include <isotream>
#include "ErrorHandle.h"

static Error SYSTEM_NOT_INIT(-23,"system initailized fail,because some reason");

int foo()
{
  return SYSTEM_NOT_INIT;
}

int main()
{
    int err_code = foo();
    // print error details
    std::cout << Error::GetErrorString(err_code) << std::endl;
    return 0;
}
```