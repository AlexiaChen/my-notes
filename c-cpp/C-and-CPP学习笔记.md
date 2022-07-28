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

