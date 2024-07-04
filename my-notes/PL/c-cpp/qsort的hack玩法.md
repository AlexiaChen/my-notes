#cpp

无聊，写着耍耍。有本书叫短码之美，感受下短码的魅力。

```c
#include <stdio.h>
#include <stdlib.h>

char cmp_shellcode[] = "\x55"
"\x89\xe5"
"\x8b\x4d\x08"
"\x8b\x45\x0c"
"\x8b\x10"
"\x8b\x01"
"\x29\xd0"
"\x5d"
"\xc3";

int main(n)
{
  int a = 2, b = 5;
  int data1[10] = {2,3,4,67,32,25,63,23,64, 88};
  
  int data2[10] = {43,15,42,13,44,24,54,33,1,10};

  qsort(data1, 10, sizeof(int),(int (*)(int *, int *))&cmp_shellcode);
  qsort(data2, 10, sizeof(int),"YXZQQQ\x8b\x00+\x02\xc3");

  for(n = 0; n < 10; printf("%d ", data1[n]), n++);
  
  puts("");
  
  for(n = 0; n < 10; printf("%d ", data2[n]), n++);

  return 0;
}
```