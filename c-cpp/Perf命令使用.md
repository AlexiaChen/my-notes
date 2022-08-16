#### Perf命令使用

[Tutorial - Perf Wiki (kernel.org)](https://perf.wiki.kernel.org/index.php/Tutorial)

[Perf examples - Perf Wiki (kernel.org)](https://perf.wiki.kernel.org/index.php/Perf_examples)

这个底层是有内核相关事件支持的，可以分析更细致的性能问题，kernel的耗时。CPU L2缓存的失效，分配的内存，CPU在内存IO上的短时暂停现象等等

```bash
sudo apt-get install linux-tools-commons
```

通过 `perf list`来查看perf支持的预定义事件，perf支持软件事件和硬件事件,大概有这些:

```txt
硬件事件：
cpu-cycles OR cycles
instructions
cache-references
cache-misses
branch-instructions OR branches
branch-misses
bus-cycles
stalled-cycles-frontend OR idle-cycles-frontend
stalled-cycles-backend OR idle-cycles-backend
ref-cycles
软件事件：
cpu-clock
task-clock
page-faults OR faults
context-switches OR cs
cpu-migrations OR migrations
minor-faults
major-faults
alignment-faults
emulation-faults
```

这里我们主要关心cpu-clock的事件，因为大部分软件性能分析都在这里，所以是一个好的开始，硬件分析就复杂多了

###### 花费时间和时间计数

首先需要先建立一个基线性能，这个性能是将来代码变更或者编译参数做变更以后为了方便对比性能而目的的。`cpu-clock`这个事件就是我们可以用来作为一个基线的标准。

```bash
perf stat -e cpu-clock <your_program>
```

然后会有类似的输出:

```txt
 Performance counter stats for './naive':

      16515.108000 cpu-clock

      16.752186472 seconds time elapsed
```

以上是事件的次数和花费的时间，将来就可以用作对比。你看到这些数据可能会懵，因为这跟我们平常的性能测试不太一样。

```bash
perf stat -e cpu-clock,faults <your_program>
```

```txt
 Performance counter stats for './naive':

      16508.392000 cpu-clock
               885 page-faults

      16.740473493 seconds time elapsed
```

###### 一些性能分析的实际问题

我们在实操的时候，一般会问以下三个问题:

- 程序的哪部分耗费了大部分时间？
- 软件或硬件事件的数量是否表明有实际的性能问题需要修复？
- 怎样修复性能问题？

为了回答这些问题，我们需要成为侦探。

第一个问题：花费最多执行时间的代码区域被称为热点（瓶颈）。热点是调整和优化的最佳位置，因为在使热点更快方面所做的一点点努力都会有很大的回报。在只执行一次或不需要太多时间的代码上花费工程努力是不值得的。所以cpu-clock很重要。

第二个问题:，诸如页面故障等事件的数量是否表明有实际的性能问题需要修复。在上面的第二个例子中，885个页面故障是否太多？为了回答这个问题，我们需要对程序中的算法和数据结构有一些了解或直觉。我们至少需要知道这个程序是CPU密集的（做大量的计算）还是内存密集的（需要在内存中读/写大量的数据）。我们还需要知道事件在程序中发生的位置。如果这些事件与一个已知的热点（瓶颈）相关，那么我们可能已经发现了一个真正的性能问题，并需要解决它。

最后一个问题是最难回答的：让我们假设我们已经发现了一个有大量页面故障的热点。我们需要研究该程序的算法和数据结构，并确定该程序是否有一个不必要的大的已分配的内存大小，并在短时间内触及许多页面。我们可能需要减少或重新组织数据，使数据驻留在更少的页面上。一旦代码修改完成，我们必须测量修改后的程序的性能，并将其执行时间与基线进行比较，看是否有任何加速。

性能调整是艰苦的实验性工作，如果修改后的程序比基线运行得慢，请不要感到惊讶。如果一个假设失败了，那就尝试另一个。继续尝试!

###### 怎样找到热点瓶颈

运行并收集程序的profile的数据，推荐用`-ggdb`的flag进行编译你的程序:

```bash
perf record -e cpu-clock,faults <your_program>
```

查看Profile，其中的comm是command也就是你的程序命令的意思，dso也就是动态库的意思，主要显示这两个模块

```bash
perf report --stdio --sort comm,dso
```

运行以上命令你会看到代码热点的排序，从用户空间到内核的，因为我们关注的是两个事件，CPU和内存页错误，所以会得到两个列表:

```txt
# ========
# captured on: Fri Jun 14 14:45:35 2013
# hostname : raspberrypi
# os release : 3.6.11+
# perf version : 3.6.0
# arch : armv6l
# nrcpus online : 1
# nrcpus avail : 1
# cpudesc : ARMv6-compatible processor rev 7 (v6l)
# total memory : 448776 kB
# cmdline : /usr/bin/perf_3.6 record -e cpu-clock,faults ./naive 
# event : name = cpu-clock, type = 1, config = 0x0, config1 = 0x0, config2 = 0x0
# event : name = page-faults, type = 1, config = 0x2, config1 = 0x0, config2 = 0
# HEADER_CPU_TOPOLOGY info available, use -I to display
# ========
#
# Samples: 74K of event 'cpu-clock'
# Event count (approx.): 74945
#
# Overhead  Command      Shared Object
# ........  .......  .................
#
    99.31%    naive  naive            
     0.51%    naive  libc-2.13.so     
     0.17%    naive  [kernel.kallsyms]
     0.01%    naive  ld-2.13.so       


# Samples: 75  of event 'page-faults'
# Event count (approx.): 887
#
# Overhead  Command      Shared Object
# ........  .......  .................
#
    83.54%    naive  naive            
     7.55%    naive  ld-2.13.so       
     7.44%    naive  libc-2.13.so     
     1.35%    naive  [kernel.kallsyms]
     0.11%    naive  perf_3.6
```

很显然，对于CPU来说，程序本身(naive) 花费了最多时间，其次是libc-2.13.so。内存上也是naive最多。

接下来我们通过命令查看更详细的report:

```bash
perf report --stdio --dsos=naive,libc-2.13.so
```

```txt
# ========
# captured on: Fri Jun 14 14:45:35 2013
# hostname : raspberrypi
# os release : 3.6.11+
# perf version : 3.6.0
# arch : armv6l
# nrcpus online : 1
# nrcpus avail : 1
# cpudesc : ARMv6-compatible processor rev 7 (v6l)
# total memory : 448776 kB
# cmdline : /usr/bin/perf_3.6 record -e cpu-clock,faults ./naive 
# event : name = cpu-clock, type = 1, config = 0x0, config1 = 0x0, config2 = 0x0
# event : name = page-faults, type = 1, config = 0x2, config1 = 0x0, config2 = 0
# HEADER_CPU_TOPOLOGY info available, use -I to display
# ========
#
# Samples: 74K of event 'cpu-clock'
# Event count (approx.): 74813
#
# Overhead  Command  Shared Object                   Symbol
# ........  .......  .............  .......................
#
    99.32%    naive  naive          [.] multiply_matrices  
     0.34%    naive  libc-2.13.so   [.] random             
     0.15%    naive  naive          [.] initialize_matrices
     0.13%    naive  libc-2.13.so   [.] random_r           
     0.04%    naive  libc-2.13.so   [.] rand               
     0.02%    naive  naive          [.] 0x000006d8         
     0.00%    naive  libc-2.13.so   [.] _dl_addr           
     0.00%    naive  libc-2.13.so   [.] __printf_fp        
     0.00%    naive  libc-2.13.so   [.] fprintf            
     0.00%    naive  libc-2.13.so   [.] memchr             


# Samples: 63  of event 'page-faults'
# Event count (approx.): 807
#
# Overhead  Command  Shared Object                   Symbol
# ........  .......  .............  .......................
#
    91.82%    naive  naive          [.] initialize_matrices
     6.44%    naive  libc-2.13.so   [.] 0x000ca168         
     1.61%    naive  libc-2.13.so   [.] get_nprocs         
     0.12%    naive  libc-2.13.so   [.] strchr
```

显然，CPU上是naive中的矩阵乘法拖慢的效率。内存中是初始化矩阵内存耗费了效率。当然从代码上，你就可以尝试优化矩阵乘法效率，如果还想深挖，就看反汇编代码，如下:

```bash
perf annotate --stdio --dsos=naive --symbol=multiply_matrices
```

```txt
 Percent |	Source code & Disassembly of naive
------------------------------------------------
         :
         :	Disassembly of section .text:
         :
         :	00008bd0 :
         :
         :	void multiply_matrices()
         :	{
    0.00 :	  8bd0:       push    {r4, r5, r6, r7, r8, r9, sl}
         :	  int i, j, k ;
         :
         :	  for (i = 0 ; i < MSIZE ; i++) {
    0.00 :	  8bd4:       mov     r6, #0
    0.00 :	  8bd8:       ldr     sl, [pc, #124]  ; 8c5c 
    0.00 :	  8bdc:       ldr     r4, [pc, #124]  ; 8c60 
    0.00 :	  8be0:       ldr     r8, [pc, #124]  ; 8c64 
         :
         :	void multiply_matrices()
    0.00 :	  8be4:       mov     r7, #2000       ; 0x7d0
    0.00 :	  8be8:       mul     r5, r7, r6
    0.00 :	  8bec:       mov     r0, #0
    0.00 :	  8bf0:       sub     r5, r5, #4
    0.00 :	  8bf4:       add     ip, r8, r5
    0.00 :	  8bf8:       add     r5, sl, r5
    0.01 :	  8bfc:       vldr    s15, [pc, #84]  ; 8c58 
    0.22 :	  8c00:       mov     r1, r5
    0.00 :	  8c04:       add     r2, r4, r0, lsl #2
    0.20 :	  8c08:       mov     r3, #0
         :
         :	  for (i = 0 ; i < MSIZE ; i++) {
         :	    for (j = 0 ; j < MSIZE ; j++) {
         :	      float sum = 0.0 ;
         :	      for (k = 0 ; k < MSIZE ; k++) {
         :	        sum = sum + (matrix_a[i][k] * matrix_b[k][j]) ;
    1.76 :	  8c0c:       add     r1, r1, #4
    0.58 :	  8c10:       vldr    s14, [r2]
    1.07 :	  8c14:       vldr    s13, [r1]
         :	  int i, j, k ;
         :
         :	  for (i = 0 ; i < MSIZE ; i++) {
         :	    for (j = 0 ; j < MSIZE ; j++) {
         :	      float sum = 0.0 ;
         :	      for (k = 0 ; k < MSIZE ; k++) {
    1.67 :	  8c18:       add     r3, r3, #1
   54.13 :	  8c1c:       cmp     r3, #500        ; 0x1f4
         :	        sum = sum + (matrix_a[i][k] * matrix_b[k][j]) ;
    2.39 :	  8c20:       mov     r9, r1
   36.60 :	  8c24:       vmla.f32        s15, s13, s14
         :	  int i, j, k ;
         :
         :	  for (i = 0 ; i < MSIZE ; i++) {
         :	    for (j = 0 ; j < MSIZE ; j++) {
         :	      float sum = 0.0 ;
         :	      for (k = 0 ; k < MSIZE ; k++) {
    1.34 :	  8c28:       add     r2, r2, #2000   ; 0x7d0
    0.00 :	  8c2c:       bne     8c0c 
         :	        sum = sum + (matrix_a[i][k] * matrix_b[k][j]) ;
         :	      }
         :	      matrix_r[i][j] = sum ;
    0.01 :	  8c30:       vmov    r3, s15
         :	void multiply_matrices()
         :	{
         :	  int i, j, k ;
         :
         :	  for (i = 0 ; i < MSIZE ; i++) {
         :	    for (j = 0 ; j < MSIZE ; j++) {
    0.00 :	  8c34:       add     r0, r0, #1
    0.00 :	  8c38:       cmp     r0, #500        ; 0x1f4
         :	      float sum = 0.0 ;
         :	      for (k = 0 ; k < MSIZE ; k++) {
         :	        sum = sum + (matrix_a[i][k] * matrix_b[k][j]) ;
         :	      }
         :	      matrix_r[i][j] = sum ;
    0.01 :	  8c3c:       str     r3, [ip, #4]!
         :	void multiply_matrices()
         :	{
         :	  int i, j, k ;
         :
         :	  for (i = 0 ; i < MSIZE ; i++) {
         :	    for (j = 0 ; j < MSIZE ; j++) {
    0.01 :	  8c40:       bne     8bfc 
         :
         :	void multiply_matrices()
         :	{
         :	  int i, j, k ;
         :
         :	  for (i = 0 ; i < MSIZE ; i++) {
    0.00 :	  8c44:       add     r6, r6, #1
    0.00 :	  8c48:       cmp     r6, #500        ; 0x1f4
    0.00 :	  8c4c:       bne     8be8 
         :	        sum = sum + (matrix_a[i][k] * matrix_b[k][j]) ;
         :	      }
         :	      matrix_r[i][j] = sum ;
         :	    }
         :	  }
         :	}
    0.00 :	  8c50:       pop     {r4, r5, r6, r7, r8, r9, sl}
    0.00 :	  8c54:       bx      lr
```
