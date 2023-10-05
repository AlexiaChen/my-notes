

熟悉Linux的开发人员都知道，在Linux下开发程序，如果程序崩溃了，可以通过配置Core Dump，来让程序崩溃的瞬间产生一个Dump文件，然后通过dump文件来调试程序为什么崩溃。但是windows下就比较麻烦。


windows下配置用户态程序的Dump非常[麻烦](https://msdn.microsoft.com/en-us/library/windows/desktop/bb787181\(v=vs.85\).aspx)。

总结后第一种的方式是用一个微软出品的[官方工具](https://support.microsoft.com/zh-cn/help/241215/how-to-use-the-userdump-exe-tool-to-create-a-dump-file)，是一个UserDump.exe的命令行工具。

用法是： 下载下来，安装，解压到特定目录，里面就可以看到userdump.exe这个命令行工具了，把这个命令行工具随着公司的产品软件打包，在产品软件中的main函数中编写如下代码:

```cpp

LONG __stdcall
MyUnhandledExceptionFilter(
    EXCEPTION_POINTERS *ExceptionInfo
)
{
    // Invoke userdump here.
    // The implementation is similar to MyFilterFunction,
    // above.

    (void)ExceptionInfo;

    QString dumperExe = QString("%1/%2/%3").arg(qApp->applicationDirPath()).arg("dumper").arg("userdump.exe");

    QString cmdLine = QString("%1 -k -w %2").arg(dumperExe).arg("TRMSMonitor.exe"); // product.exe

    QProcess* startDump = new QProcess;
    startDump->start(cmdLine);
    startDump->waitForFinished(-1);

    return EXCEPTION_EXECUTE_HANDLER;
}

int main(int argc, char* argv[])
{
    QApplication a(argc, argv);

    SetUnhandledExceptionFilter(MyUnhandledExceptionFilter);

    //主程序单例模式
    RunGuard guard("TRMS");

    if (!guard.tryToRun()) {
        exit(-1);
    }

    // any code here

    return a.exec();
}
```

以上代码的意思是，在程序崩溃的瞬间，未处理异常的时刻，调用userdump命令行来生成程序自身的Dump文件。

方法二，修改系统注册表的方式来让系统自动产生Dump，最推荐这样的方式，这种方式最准确。用法[戳这里](https://www.cnblogs.com/hushaojun/p/6388153.html)。

如果WinDbg下载了符号表，那么直接可以把产生crash的源码崩溃地点显示出来：

```txt

Microsoft (R) Windows Debugger Version 6.12.0002.633 X86 Copyright (c) Microsoft Corporation. All rights reserved. Loading Dump File [C:\crashdumps\mid.exe.2324.dmp] User Mini Dump File with Full Memory: Only application data is available WARNING: Minidump contains unknown stream type 0x15 WARNING: Minidump contains unknown stream type 0x16 Symbol search path is: srv*c:\symbols*http://msdl.microsoft.com/download/symbols Executable search path is: Windows 7 Version 16299 MP (4 procs) Free x64 Product: WinNt, suite: SingleUserTS Machine Name: Debug session time: Fri May 4 10:57:03.000 2018 (UTC + 8:00)
System Uptime: 0 days 23:20:34.747
Process Uptime: 0 days 0:00:03.000
........................................
This dump file has an exception of interest stored in it.
The stored exception information can be accessed via .ecxr.
(914.35c4): Access violation - code c0000005 (first/second chance not available)
ntdll!ZwWaitForMultipleObjects+0x14:
00007ffe`5c5b0e14 c3 ret
0:000> !analyze -v
*******************************************************************************
* *
* Exception Analysis *
* *
*******************************************************************************
 
Failed calling InternetOpenUrl, GLE=12029
 
FAULTING_IP: 
mid!CJASE2000App::InitInstance+3c [d:\furen-work\furen-projects\tuxedo-projects\zjj\mid\jase2000.cpp @ 223]
00007ff6`ced7aa9c 45892424 mov dword ptr [r12],r12d
 
EXCEPTION_RECORD: ffffffffffffffff -- (.exr 0xffffffffffffffff)
ExceptionAddress: 00007ff6ced7aa9c (mid!CJASE2000App::InitInstance+0x000000000000003c)
 ExceptionCode: c0000005 (Access violation)
ExceptionFlags: 00000000
NumberParameters: 2
 Parameter[0]: 0000000000000001
 Parameter[1]: 0000000000000000
Attempt to write to address 0000000000000000
 
DEFAULT_BUCKET_ID: NULL_POINTER_WRITE
 
PROCESS_NAME: mid.exe
 
ERROR_CODE: (NTSTATUS) 0xc0000005 - 0x%p
 
EXCEPTION_CODE: (NTSTATUS) 0xc0000005 - 0x%p
 
EXCEPTION_PARAMETER1: 0000000000000001
 
EXCEPTION_PARAMETER2: 0000000000000000
 
WRITE_ADDRESS: 0000000000000000
FOLLOWUP_IP: 
mid!CJASE2000App::InitInstance+3c [d:\furen-work\furen-projects\tuxedo-projects\zjj\mid\jase2000.cpp @ 223]
00007ff6`ced7aa9c 45892424 mov dword ptr [r12],r12d
 
MOD_LIST: <ANALYSIS/>  
NTGLOBALFLAG: 0
 
APPLICATION_VERIFIER_FLAGS: 0
 
FAULTING_THREAD: 00000000000035c4
 
PRIMARY_PROBLEM_CLASS: NULL_POINTER_WRITE
 
BUGCHECK_STR: APPLICATION_FAULT_NULL_POINTER_WRITE

LAST_CONTROL_TRANSFER: from 000000005e44ccae to 00007ff6ced7aa9c
 
STACK_TEXT: 
00000000`00cff3a0 00000000`5e44ccae : 00000000`00000001 00000000`00000001 00007ff6`ced30000 00000000`00000000 : mid!CJASE2000App::InitInstance+0x3c [d:\furen-work\furen-projects\tuxedo-projects\zjj\mid\jase2000.cpp @ 223]
00000000`00cff710 00007ff6`ceda05e3 : 00000000`02eb4383 00000000`00000000 00000000`00000000 00000000`00000000 : mfc100!AfxWinMain+0x76
00000000`00cff750 00007ffe`5a8d1fe4 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : mid!__tmainCRTStartup+0x15f [f:\dd\vctools\crt_bld\self_64_amd64\crt\src\crtexe.c @ 547] 00000000`00cff800 00007ffe`5c57f061 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0x14
00000000`00cff830 00000000`00000000 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x21
 
 
FAULTING_SOURCE_CODE: 
 219: // registerExceptionHandler(); 
220: 
 221: int *pSS = NULL;
 222: > 223: *pSS = 0;
 224: 
 225: m_hMutex = CreateMutex( 
226: NULL, // no security descriptor
 227: FALSE, // mutex not owned
 228: "KD20_Transaction"); // object name
 
SYMBOL_STACK_INDEX: 0
 
SYMBOL_NAME: mid!CJASE2000App::InitInstance+3c
 
FOLLOWUP_NAME: MachineOwner
 MODULE_NAME: mid
 
IMAGE_NAME: mid.exe
 
DEBUG_FLR_IMAGE_TIMESTAMP: 5aebcbd4
 
STACK_COMMAND: dt ntdll!LdrpLastDllInitializer BaseDllName ; dt ntdll!LdrpFailureData ; ~0s; .ecxr ; kb
 
FAILURE_BUCKET_ID: NULL_POINTER_WRITE_c0000005_mid.exe!CJASE2000App::InitInstance
 
BUCKET_ID: X64_APPLICATION_FAULT_NULL_POINTER_WRITE_mid!CJASE2000App::InitInstance+3c
 
WATSON_STAGEONE_URL: http://watson.microsoft.com/StageOne/mid_exe/2_0_0_0/5aebcbd4/mid_exe/2_0_0_0/5aebcbd4/c0000005/0004aa9c.htm?Retriage=1
 
Followup: MachineOwner
```

EOF