#dll 
#win32


我相信写WIN32程序的人，做过DLL，都会很清楚__declspec(dllexport)的作用，它就是为了省掉在DEF文件中手工定义导出哪些函数的一个方法。当然，如果你的DLL里全是C++的类的话，你无法在DEF里指定导出的函数，只能用__declspec(dllexport)导出类。但是，MSDN文档里面，对于__declspec(dllimport)的说明让人感觉有点奇怪，先来看看MSDN里面是怎么说的：

> *不使用 __declspec(dllimport) 也能正确编译代码，但使用 __declspec(dllimport) 使编译器可以生成更好的代码。编译器之所以能够生成更好的代码，是因为它可以确定函数是否存在于 DLL 中，这使得编译器可以生成跳过间接寻址级别的代码，而这些代码通常会出现在跨 DLL 边界的函数调用中。但是，必须使用 __declspec(dllimport) 才能导入 DLL 中使用的变量。*

初看起来，这段话前面的意思是，不用它也可以正常使用DLL的导出库，但最后一句话又说，必须使用 __declspec(dllimport) 才能导入 DLL 中使用的变量这个是什么意思？

那我就来试验一下，假定，你在DLL里只导出一个简单的类，注意，我假定你已经在项目属性中定义了 SIMPLEDLL_EXPORT:

SimpleDLLClass.h文件:

```cpp
#ifdef SIMPLEDLL_EXPORT
#define DLL_EXPORT __declspec(dllexport)
#else
#define DLL_EXPORT
#endif

class DLL_EXPORT SimpleDLLClass
{
public:
  SimpleDLLClass();
  virtual ~SimpleDLLClass();

  virtual getValue() { return m_nValue;};
private:
  int m_nValue;
};
```

SimpleDLLClass.cpp文件:
```cpp
#include "SimpleDLLClass.h"

SimpleDLLClass::SimpleDLLClass():m_nValue(0)
{
}

SimpleDLLClass::~SimpleDLLClass()
{
}
```
然后你再使用这个DLL类，在你的APP中include SimpleDLLClass.h时，你的APP的项目不用定义 SIMPLEDLL_EXPORT 所以，DLL_EXPORT 就不会存在了，这个时候，你在APP中，不会遇到问题。这正好对应MSDN上说的__declspec(dllimport)定义与否都可以正常使用。但我们也没有遇到变量不能正常使用呀。那好，我们改一下SimpleDLLClass,把它的m_nValue改成static,然后在cpp文件中加一行

int SimpleDLLClass::m_nValue=0;
如果你不知道为什么要加这一行，那就回去看看C++的基础。 改完之后，再去LINK一下，你的APP，看结果如何，结果是LINK告诉你找不到这个m_nValue。明明已经定义了，为什么又没有了？？肯定是因为我把m_nValue定义为static的原因。但如果我一定要使用Singleton的Design Pattern的话，那这个类肯定是要有一个静态成员，每次LINK都没有，那不是完了？ 如果你有Platform SDK，用里面的Depend程序看一下，DLL中又的确是有这个m_nValue导出的呀。
再回去看看我引用MSDN的那段话的最后一句。 那我们再改一下SimpleDLLClass.h，把那段改成下面的样子:

```cpp
#ifdef SIMPLEDLL_EXPORT
#define DLL_EXPORT __declspec(dllexport)
#else
#define DLL_EXPORT __declspec(dllimport)
#endif
```
再LINK，一切正常。原来dllimport是为了更好的处理类中的静态成员变量的，如果没有静态成员变量，那么这个__declspec(dllimport)无所谓。

_declspec(dllexport)与_declspec(dllimport)都是DLL内的关键字，即导出与导入。他们是将DLL内部的类与函数以及数据导出与导入时使用的。主要区别在于，dllexport是在这些类、函数以 及数据的申明的时候使用。用过表明这些东西可以被外部函数使用，即（dllexport）是把DLL中的相关代码（类，函数，数据）暴露出来为其他应用程 序使用。使用了（dllexport）关键字，相当于声明了紧接在（dllexport）关键字后面的相关内容是可以为其他程序使用的。而 dllimport关键字是在外部程序需要使用DLL内相关内容时使用的关键字。当一个外部程序要使用DLL内部代码（类，函数，全局变量）时，只需要在 程序内部使用（dllimport）关键字声明需要使用的代码就可以了，即（dllimport）关键字是在外部程序需要使用DLL内部相关内容的时候才 使用。（dllimport）作用是把DLL中的相关代码插入到应用程序中。

　　 _declspec(dllexport)与_declspec(dllimport)是相互呼应，只有在DLL内部用dllexport作了声明，才能 在外部函数中用dllimport导入相关代码。实际上，在应用程序访问DLL时，实际上就是应用程序中的导入函数与DLL文件中的导出函数进行链接。而 且链接的方式有两种：隐式迎接和显式链接。

　　隐式链接是指通过编译器提供给应用程序关于DLL的名称和DLL函数的链接地址，面在应用程序中不需要显式地将DLL加载到内存，即在应用程序中使用dllimport即表明使用隐式链接。不过不是所有的隐式链接都使用dllimport。

显式链接刚同应用程序用语句显式地加载DLL，编译器不需要知道任何关DLL的信息

以下是一个DLL头文件的正规编写方式：

```cpp
#ifdef DIALOG_MAINMENU_EXPORTS
#define DIALOG_MAINMENU_API __declspec(dllexport) 
#else
#define DIALOG_MAINMENU_API __declspec(dllimport) 
#endif

class Dialog_MainMenu {
public:
    static DIALOG_MAINMENU_API enum GAME_STATES {
        MAINMENU, GAME, OPTIONS, CREDITS, QUIT
    };
    static DIALOG_MAINMENU_API GAME_STATES CurrentGameState;
    DIALOG_MAINMENU_API GAME_STATES GetState();
};
```
一下截取一段老外的解释：

> *OK - when you compile the dll - you are exporting the types. So, you need to define the static member in .cpp file of the dll. You also need to make sure that you have enabled the definition of DIALOG_MAINMENU_EXPORTS in compiler settings. This will make sure types are exported.*

> *Now, when you link the console application with the dll - you will #include dll's header and dont enable any definition of DIALOG_MAINMENU_EXPORTS in compiler settings (just leave the settings default). This will make the compiler understand that now you are importing the types from your dll into exe application.*

参考：

- http://stackoverflow.com/questions/2481138/unresolved-external-symbol
- http://stackoverflow.com/questions/17901973/unresolved-external-symbol-declspecdllimport