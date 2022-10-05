#cpp 
#win32 


在main函数程序初始化的时候加入以下代码即可：

```cpp
SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOGPFAULTERRORBOX); _set_abort_behavior(0,_WRITE_ABORT_MSG);
```

这样程序就悄无声息的崩溃了，不然守护进程都不起作用。如果不这样做，弹出错误对话框程序如果不点击关闭或发送错误报告就僵死在那里了，守护进程一直发现进程没挂，就不重启。

参考： </br>
[1] https://stackoverflow.com/questions/9718695/how-can-i-supress-all-error-dialogs-when-a-process-crashes-i-only-want-it-to-cr

[2]
https://stackoverflow.com/questions/1861506/prevent-modal-dialog-on-win32-process-crash