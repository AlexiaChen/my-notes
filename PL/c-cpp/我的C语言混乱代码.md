#cpp

> *其实这段代码是我半年前写的了，现在才把它放到自己的博客上来，就当是记录下吧 :)*

[IOCCC](http://www.ioccc.org/)是国际C语言混乱代码大赛(The International Obfuscated C Code Contest )，大二的时候知道的，当时我也写了好些，不知道放哪里了，刚又想到一个新玩法，虽然还远远达不到IOCCC的级别，不过也将就着看吧，输出我的微信ID。 = w =

``` c
#include<stdio.h>
double _[] = { 19910948789782318, 193, 8, 1};
char* __ = (char*)_;
int main(int ___)
{
    _['!' - 30] && _['$' - 35] && (_[0] *= 10, main(--_['$' - '#']));
    *(char*)_ += 5, _['#' - ' '] = 0;
    _['%' - '#'] && (putchar(*__++), main(--_['%' - '#']));  
}
```

最后附上一些参考吧，CoolShell的博主也写过类似的文章给大家启发：
- [《6个变态的C语言Hello World程序》](http://coolshell.cn/articles/914.html)
- [《如何加密/混乱C源代码》](http://coolshell.cn/articles/933.html)
- [《如何写出无法维护的代码》](http://coolshell.cn/articles/4758.html)