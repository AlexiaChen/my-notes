

最开始摸不到头脑，之前还能好好调试的啊，现在VS启动调试器变得巨卡。后来在VS的调试菜单的符号选项里面发现了系统环境变量_NT_SYMBOL_PATH 的值为：srv*c:\symbols*http://msdl.microsoft.com/download/symbols

才想起来以前用WinDbg分析过Dump文件，需要到微软的http连接里面下载windows符号文件，所以就配置了该系统环境变量。

VS也能自动识别这个环境变量，就去下载符号文件去了，所以变得很慢并卡死了，晕。

参考：

 - https://blog.csdn.net/caichengji1/article/details/77524961
