## C/C++ 链接器

[Beginner's Guide to Linkers (lurklurk.org)](https://www.lurklurk.org/linkers/linkers.html)

## GDB

[Beej's Quick Guide to GDB](https://beej.us/guide/bggdb/)

## 不使用调试器

 Rob Pike had to say about what he learnt from Ken Thompson:

> A year or two after I'd joined the Labs, I was pair programming with Ken Thompson on an on-the-fly compiler for a little interactive graphics language designed by Gerard Holzmann. I was the faster typist, so I was at the keyboard and Ken was standing behind me as we programmed. We were working fast, and things broke, often visibly—it was a graphics language, after all. When something went wrong, I'd reflexively start to dig in to the problem, examining stack traces, sticking in print statements, invoking a debugger, and so on. But Ken would just stand and think, ignoring me and the code we'd just written. After a while I noticed a pattern: Ken would often understand the problem before I would, and would suddenly announce, "I know what's wrong." He was usually correct. I realized that Ken was building a mental model of the code and when something broke it was an error in the model. By thinking about _how_ that problem could happen, he'd intuit where the model was wrong or where our code must not be satisfying the model.Ken taught me that thinking before debugging is extremely important. If you dive into the bug, you tend to fix the local issue in the code, but if you think about the bug first, how the bug came to be, you often find and correct a higher-level problem in the code that will improve the design and prevent further bugs.I recognize this is largely a matter of style. Some people insist on line-by-line tool-driven debugging for everything. But I now believe that thinking—without looking at the code—is the best debugging tool of all, because it leads to better software.

Linus email said [a/lt-debugger (lwn.net)](https://lwn.net/2000/0914/a/lt-debugger.php3)

> I don't like debuggers. Never have, probably never will. I use gdb all the
time, but I tend to use it not as a debugger, but as a disassembler on steroids that you can program.


[I do not use a debugger – Daniel Lemire's blog](https://lemire.me/blog/2016/06/21/i-do-not-use-a-debugger/)

#### 有用的GDB调试命令

1.  _backtrace full_: 局部变量的完整backtrace
2.  _up_, _down_, _frame_: 在栈帧间穿梭
3.  _watch_: 当达到特定条件，挂起当前进程
4.  _set print pretty on_: 打印出更好看的源码
5.  _set logging on_: Log debugging session to show to others for support
6.  _set print array on_: 打印更好看的数组
7.  _finish_: 执行到函数尾
8.  _enable_ and _disable_: 打开关闭某断点
9.  _tbreak_: Break once, and then remove the breakpoint
10.  _where_: 执行到哪行代码了
11.  _info locals_: 查看当前函数的所有局部变量
12.  _info args_: 查看当前函数的所有参数args
13.  _list_: 查看源码
14.  _rbreak_: break on function matching regular expression

可以直接用TUI的界面运行gdb

```bash
gdb -tui
```

gdb支持反向运行:

```txt
* reverse-continue ('rc') -- Continue program being debugged but run it in reverse
* reverse-finish -- Execute backward until just before the selected stack frame is called
* reverse-next ('rn') -- Step program backward, proceeding through subroutine calls.
* reverse-nexti ('rni') -- Step backward one instruction, but proceed through called subroutines.
* reverse-step ('rs') -- Step program backward until it reaches the beginning of a previous source line
* reverse-stepi -- Step backward exactly one instruction
* set exec-direction (forward/reverse) -- Set direction of execution.
```

gdb文字版cheat sheet

```txt
GDB commands by function - simple guide
---------------------------------------
More important commands have a (*) by them.

Startup 
% gdb -help         	print startup help, show switches
*% gdb object      	normal debug 
*% gdb object core 	core debug (must specify core file)
%% gdb object pid  	attach to running process
% gdb        		use file command to load object 

Help
*(gdb) help        	list command classes
(gdb) help running      list commands in one command class
(gdb) help run        	bottom-level help for a command "run" 
(gdb) help info         list info commands (running program state)
(gdb) help info line    help for a particular info command
(gdb) help show         list show commands (gdb state)
(gdb) help show commands        specific help for a show command

Breakpoints
*(gdb) break main       set a breakpoint on a function
*(gdb) break 101        set a breakpoint on a line number
*(gdb) break basic.c:101        set breakpoint at file and line (or function)
*(gdb) info breakpoints        show breakpoints
*(gdb) delete 1         delete a breakpoint by number
(gdb) delete        	delete all breakpoints (prompted)
(gdb) clear             delete breakpoints at current line
(gdb) clear function    delete breakpoints at function
(gdb) clear line        delete breakpoints at line
(gdb) disable 2         turn a breakpoint off, but don't remove it
(gdb) enable 2          turn disabled breakpoint back on
(gdb) tbreak function|line        set a temporary breakpoint
(gdb) commands break-no ... end   set gdb commands with breakpoint
(gdb) ignore break-no count       ignore bpt N-1 times before activation
(gdb) condition break-no expression         break only if condition is true
(gdb) condition 2 i == 20         example: break on breakpoint 2 if i equals 20
(gdb) watch expression        set software watchpoint on variable
(gdb) info watchpoints        show current watchpoints

Running the program
*(gdb) run        	run the program with current arguments
*(gdb) run args redirection        run with args and redirection
(gdb) set args args...        set arguments for run 
(gdb) show args        show current arguments to run
*(gdb) cont            continue the program
*(gdb) step            single step the program; step into functions
(gdb) step count       singlestep \fIcount\fR times
*(gdb) next            step but step over functions 
(gdb) next count       next \fIcount\fR times
*(gdb) CTRL-C          actually SIGINT, stop execution of current program 
*(gdb) attach process-id        attach to running program
*(gdb) detach        detach from running program
*(gdb) finish        finish current function's execution
(gdb) kill           kill current executing program 

Stack backtrace
*(gdb) bt        	print stack backtrace
(gdb) frame        	show current execution position
(gdb) up        	move up stack trace  (towards main)
(gdb) down        	move down stack trace (away from main)
*(gdb) info locals      print automatic variables in frame
(gdb) info args         print function parameters 

Browsing source
*(gdb) list 101        	list 10 lines around line 101
*(gdb) list 1,10        list lines 1 to 10
*(gdb) list main  	list lines around function 
*(gdb) list basic.c:main        list from another file basic.c
*(gdb) list -        	list previous 10 lines
(gdb) list *0x22e4      list source at address
(gdb) cd dir        	change current directory to \fIdir\fR
(gdb) pwd          	print working directory
(gdb) search regexpr    forward current for regular expression
(gdb) reverse-search regexpr        backward search for regular expression
(gdb) dir dirname       add directory to source path
(gdb) dir        	reset source path to nothing
(gdb) show directories        show source path

Browsing Data
*(gdb) print expression        print expression, added to value history
*(gdb) print/x expressionR        print in hex
(gdb) print array[i]@count        artificial array - print array range
(gdb) print $        	print last value
(gdb) print *$->next    print thru list
(gdb) print $1        	print value 1 from value history
(gdb) print ::gx        force scope to be global
(gdb) print 'basic.c'::gx        global scope in named file (>=4.6)
(gdb) print/x &main     print address of function
(gdb) x/countFormatSize address        low-level examine command
(gdb) x/x &gx        	print gx in hex
(gdb) x/4wx &main       print 4 longs at start of \fImain\fR in hex
(gdb) x/gf &gd1         print double
(gdb) help x        	show formats for x
*(gdb) info locals      print local automatics only
(gdb) info functions regexp         print function names
(gdb) info variables  regexp        print global variable names
*(gdb) ptype name        print type definition
(gdb) whatis expression       print type of expression
*(gdb) set variable = expression        assign value
(gdb) display expression        display expression result at stop
(gdb) undisplay        delete displays
(gdb) info display     show displays
(gdb) show values      print value history (>= gdb 4.0)
(gdb) info history     print value history (gdb 3.5)

Object File manipulation
(gdb) file object      		load new file for debug (sym+exec)
(gdb) file             		discard sym+exec file info
(gdb) symbol-file object        load only symbol table
(gdb) exec-file object 		specify object to run (not sym-file)
(gdb) core-file core   		post-mortem debugging

Signal Control
(gdb) info signals        	print signal setup
(gdb) handle signo actions      set debugger actions for signal
(gdb) handle INT print          print message when signal occurs
(gdb) handle INT noprint        don't print message
(gdb) handle INT stop        	stop program when signal occurs
(gdb) handle INT nostop         don't stop program
(gdb) handle INT pass        	allow program to receive signal
(gdb) handle INT nopass         debugger catches signal; program doesn't
(gdb) signal signo        	continue and send signal to program
(gdb) signal 0        		continue and send no signal to program

Machine-level Debug
(gdb) info registers        	print registers sans floats
(gdb) info all-registers        print all registers
(gdb) print/x $pc        	print one register
(gdb) stepi        		single step at machine level
(gdb) si        		single step at machine level
(gdb) nexti        		single step (over functions) at machine level
(gdb) ni        		single step (over functions) at machine level
(gdb) display/i $pc        	print current instruction in display
(gdb) x/x &gx        		print variable gx in hex
(gdb) info line 22        	print addresses for object code for line 22
(gdb) info line *0x2c4e         print line number of object code at address
(gdb) x/10i main        	disassemble first 10 instructions in \fImain\fR
(gdb) disassemble addr          dissassemble code for function around addr

History Display
(gdb) show commands        	print command history (>= gdb 4.0)
(gdb) info editing       	print command history (gdb 3.5)
(gdb) ESC-CTRL-J        	switch to vi edit mode from emacs edit mode
(gdb) set history expansion on       turn on c-shell like history
(gdb) break class::member       set breakpoint on class member. may get menu
(gdb) list class::member        list member in class
(gdb) ptype class               print class members
(gdb) print *this        	print contents of this pointer
(gdb) rbreak regexpr     	useful for breakpoint on overloaded member name

Miscellaneous
(gdb) define command ... end        define user command
*(gdb) RETURN        		repeat last command
*(gdb) shell command args       execute shell command 
*(gdb) source file        	load gdb commands from file
*(gdb) quit        		quit gdb
```

gdb 图片版 cheat sheet

![[f2ff8aa28c7b041d9b0115ff8ea8715.png]]

![[b14009f2edc53ae819e558a565bafd6.jpg]]

#### Strace命令调试程序

strace可以跟踪一个程序的Linux系统调用

- strace \<cmd_line\>, 输出中每一行后面的等号后面是函数的返回值
- sudo strace -p \<pid\> 追踪特定进程
- sudo strace -c -p \<pid\> 获取某个进程的详细报告，包括运行时间，函数调用时间
- sudo strace -i \<cmd_line\> 显示程序的指令指针（当前运行到哪一个函数）
- sudo strace -t \<\cmd_line> 显示每一个系统函数调用的时间戳
- sudo strace -T \<cmd_line\> 显示每一个系统函数调用完成所花费的时间
- sudo strace -e trace=write \<cmd_line\> 只查看write系统调用，相当于一个过滤器
- sudo strace -q -e trace=\<process | file | memory | network | signal\> \<cmd_line\> 追踪特定的系统相关函数，进程，文件，内存，网络相关的系统调用
- strace -d \<cmd_line\> 显示一些strace 的Debug的相关信息

