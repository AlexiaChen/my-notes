
# 第3章 读书笔记：Ruby 如何执行代码

> 💡 这一章回答的核心问题是：**字节码已经生成了，YARV 到底怎么把它"跑起来"？**
> 
> 从第一性原理看，执行的本质是**用一个有限状态的机器，沿着指令序列移动，配合栈来完成"取操作数 → 计算 → 存结果"的循环**。YARV 的特殊之处在于——它不是一台栈机，而是**两台栈机叠加**：一台管自己的指令执行，一台管 Ruby 程序的调用关系。

📌 **时代注记**：本章原书基于 Ruby 2.0 时代的 YARV。作者 Pat Shaughnessy 在 2025 年的新版预告中明确表示："Chapter 3 关于 YARV 虚拟机的内容自 2014 年以来**没有太大变化**"，只是帧结构里新增了一些值，常见指令也有重命名，并把原第 4 章部分内容挪到了第 3 章 。这意味着——**你在本章学到的执行模型，对现代 Ruby 依然成立**。

---

## 3 Ruby 如何执行代码

YARV 执行代码的核心矛盾是：**它既要执行自己内部的低级指令（如 `putobject`、`opt_plus`），又要追踪 Ruby 程序层面的方法调用、块调用、lambda 调用**。这两件事的栈结构完全不同。

为此，YARV 设计了**双栈架构**​ ：

1. **内部栈（Internal Stack）**：执行 YARV 指令，跟踪中间值、参数、返回值
    
2. **Ruby 调用栈（Call Stack）**：用一串 `rb_control_frame_t` 结构记录"哪个方法调用了哪个方法/块/lambda"
    

这让它不仅是栈机，而是**双栈机（double-stack machine）**​ 。

🔑 **核心洞见**：理解 YARV 的关键是理解**两个指针、一个帧**——

- **PC（Program Counter）**：指向下一条要执行的指令
    
- **SP（Stack Pointer）**：指向内部栈的栈顶
    
- **`rb_control_frame_t`**：每个 Ruby 调用栈帧都是一个 C 结构体，里面**同时保存了 PC、SP、self、以及代码类型（[METHOD]/[BLOCK]）**​
    

当 YARV 调用一个 Ruby 方法时，它做两件事：在内部栈上做参数传递（SP 移动），同时在调用栈上**压入一个新的 `rb_control_frame_t`**（CFP 指向新帧）。方法返回时，弹出该帧，回到上一帧的 PC 继续执行 。

---

## 3.1 YARV 内部栈和 Ruby 调用栈

YARV 内部栈和 Ruby 调用栈是**两套独立的栈**，但密切配合。

### 内部栈的特征

- 栈式虚拟机（stack-oriented），**不用寄存器**存放操作数，所有中间值都压在栈上
    
- `putobject 2` 把 2 压栈；`opt_plus` 弹两个、相加、把结果压回栈
    
- SP 寄存器标记栈顶位置
    
- 每条指令执行后，PC 自动递增到下一条（除非是跳转指令）
    

### Ruby 调用栈的特征

- 用一串 `rb_control_frame_t` 结构表示
    
- CFP（Current Frame Pointer）指向当前帧
    
- 每个帧里保存了该层的 self、PC、SP 的副本
    
- 帧类型用 `[METHOD]`、`[BLOCK]` 等标记
    
- 这就是 `caller`、`caller_locations` 看到的调用栈——也是异常回溯（backtrace）的数据来源
    

🎯 **我的洞见**：双栈设计的根本原因是**关注点分离**——

- **内部栈**只关心"当前这条指令的操作数在哪里"，它是**无状态的、机械的**
    
- **调用栈**只关心"我现在执行的是哪个 Ruby 作用域"，它是**有语义的、逻辑的**
    

如果把两者合并，YARV 就无法区分"方法 A 调用方法 B"和"`putobject` 接着 `opt_plus`"——前者是 Ruby 语义层面的调用，后者只是指令层面的顺序执行。**双栈让 YARV 能在两个抽象层次上同时工作**。

### 3.1.1 逐句查看 Ruby 如何执行简单脚本

以 `puts 2+2` 为例（这是原书贯穿第 1-3 章的主线例子）。编译出的 YARV 指令大致是：

```
0000 putself
0001 putobject 2
0003 putobject 2
0005 opt_plus
0006 opt_send_without_block :puts, 1
0009 leave
```

逐步执行过程 ：

|步骤|指令|内部栈变化|PC 移动|
|---|---|---|---|
|1|`putself`|压入 `main`|→ 0001|
|2|`putobject 2`|压入 `2`|→ 0003|
|3|`putobject 2`|压入 `2`|→ 0005|
|4|`opt_plus`|弹出两个 `2`，压入 `4`|→ 0006|
|5|`opt_send_without_block :puts, 1`|弹出 `4` 作为实参，调用 `puts`，压入返回值 `nil`|→ 0009|
|6|`leave`|弹出 `nil` 作为脚本返回值|结束|

注意：这条脚本**没有形成 Ruby 调用栈**（因为它在顶层作用域直接执行），所以这一节我们只观察内部栈的演变 。

🎯 **我的洞见**：观察这个过程，你会发现 YARV 执行指令的节奏非常机械——**一条一条来，每条指令只跟栈打交道，不关心全局**。这种"机械性"正是虚拟机执行的高效之源：

- PC 自动递增（除了跳转）
    
- SP 随压栈/弹栈自动移动
    
- 每条指令的"栈效应（stack effect）"是静态已知的——比如 `opt_plus` 总是弹 2 压 1
    

这种确定性让 YARV 的解释器循环可以做得极简。现代 Ruby 的 MJIT/RJIT 正是利用"栈效应已知"这一特性，把热点指令序列直接 JIT 成机器码 。

💡 **工业实战**：你可以自己用 `RubyVM::InstructionSequence` 观察任何代码的字节码 ：

```
puts RubyVM::InstructionSequence.compile("x = 1 + 2").disasm
# == disasm: ...
# 0000 putobject 1
# 0002 putobject 2
# 0004 opt_plus
# 0005 setlocal_x, 0
# 0008 leave
```

这在性能调优、理解 `String#+` vs 字符串插值差异、分析方法调用开销时，是**终极武器**——它让你"看见" Ruby 代码下面的真实执行。

### 3.1.2 执行块调用

块调用是 Ruby 最具特色的控制结构，它的执行涉及**三栈帧的协作**。以原书经典例子 ：

```
str = "The quick brown fox..."
10.times do
  str2 = "jumps over the lazy dog."
  puts "#{str} #{str2}"
end
```

执行过程（结合原书 Figure 3-9 到 3-13 的图解）：

**第 1 步**：执行 `str = "..."`，YARV 在顶层帧的内部栈上分配 `str` 的空间，并通过 **EP（Environment Pointer，环境指针）**​ 记录其位置 。

**第 2 步**：遇到 `10.times do ... end` 时，YARV **创建一个 `rb_block_t` 结构**来表示这个块。最关键的动作是——**把当前帧的 DFP（Dynamic Frame Pointer）复制到 `rb_block_t` 中**​ 。这意味着块"记住"了创建它时的栈帧位置。

**第 3 步**：调用 `Fixnum#times` 方法，压入一个新的方法帧。此时调用栈上有两层帧：底层是顶层帧（含 `str`），上层是 `times` 的方法帧。

**第 4 步**：`times` 的内部 C 代码开始迭代，每次迭代调用 `yield`。`yield` 内部通过 **`invokeblock` 指令**激活块 ：

```
invokeblock  → 弹出块参数，执行块，把结果压栈
```

**第 5 步**：`invokeblock` 执行时，**压入第三个帧——块帧**。关键动作是：把 `rb_block_t` 里保存的 DFP 复制到这个新块帧中 。

这样，块帧就同时拥有：

- 自己的局部变量（如 `str2`），通过自身的 EP 访问
    
- 外层作用域的变量（如 `str`），通过继承来的 DFP 做**动态访问**​
    

🎯 **我的洞见（本节最关键的认知）**：

> **块的本质 = 一段指令（ISEQ）+ 一个捕获的栈帧指针（DFP）**。
> 
> 这就是为什么块能"关闭"住外层变量——不是复制，而是**持有指向外层栈帧的指针**。`rb_block_t` 这个结构就是 Ruby 闭包的实体 。

这个设计的精妙之处在于：

1. **零拷贝**：外层变量不被复制，块直接通过指针访问
    
2. **实时性**：如果外层作用域修改变量，块内立即可见（反之亦然）
    
3. **生命周期管理**：块的 DFP 形成了帧之间的引用链，GC 通过这个链判断哪些栈帧还不能回收
    

⚠️ **工业现实**：

- 这种"指针捕获"设计意味着**长生命周期的块可能阻止整个外层栈帧被 GC**——这是 Ruby 内存泄漏的常见隐蔽来源
    
- 现代 Ruby 的 `Proc` 对象、`lambda`、以及 Rails 等框架中大量使用的块，底层都是这套机制
    
- `invokeblock` 指令在现代 Ruby 的 MJIT/RJIT 中是**重点优化对象**——通过块内联（inlining）消除帧切换开销
    

---

## 3.2 访问 Ruby 变量的两种方式

Ruby 变量的访问方式取决于**变量定义在哪个作用域**：

- **局部访问（Local Access）**：变量就在当前帧里 → 用 `getlocal`/`setlocal`
    
- **动态访问（Dynamic Access）**：变量在外层帧里（典型场景：块访问外层变量）→ 用 `getdynamic`/`setdynamic`，通过 DFP 逐级上爬
    

### 3.2.1 本地变量访问

本地变量的访问通过 `getlocal` 和 `setlocal` 指令完成。在 Ruby 源码的 `insns.def` 中，它们的定义是 ：

```
DEFINE_INSN getlocal (lindex_t idx, rb_num_t level) () (VALUE val){
  val = *(vm_get_ep(GET_EP(), level) - idx);
}

DEFINE_INSN setlocal (lindex_t idx, rb_num_t level) (VALUE val) () {
  vm_env_write(vm_get_ep(GET_EP(), level), -(int)idx, val);
}
```

其中：

- **`idx`**：变量在当前帧本地表中的索引
    
- **`level`**：嵌套深度（0 表示当前帧，1 表示上一层块帧，以此类推）
    

对于普通的局部变量访问，`level` 总是 0，所以 Ruby 2.0+ 做了**操作数优化（operand optimization）**——把 `level=0` 直接编码进指令名，变成 `getlocal_OP__WC__0` 和 `setlocal_OP__WC__0`，省去了运行期传递 `level` 参数的开销 。

🎯 **我的洞见**：

> **`getlocal idx, 0` 的本质是一次 O(1) 的数组随机访问**——`vm_get_ep(GET_EP(), 0)` 拿到当前环境的基址，减去 `idx` 就是要访问的栈槽位。这就是为什么局部变量访问极快。

对比一下不同方案：

- **局部变量**：`*(base - idx)`，O(1)，无函数调用
    
- **实例变量**：`@foo` 需要哈希查找（虽然 Ruby 有优化）
    
- **全局变量**：`$foo` 需要哈希查找 + 线程同步
    
- **动态变量访问**：需要逐级爬 DFP 链
    

**这就是为什么在热点循环中，局部变量永远是最快的**——它直接映射到栈内存偏移。

### 3.2.2 方法参数被看成本地变量

这是 Ruby 设计中非常优雅的统一：

> **方法参数在 YARV 层面就是局部变量**。

唯一区别是：调用方在方法调用前，把参数值**按约定顺序压入栈**，被调用方法的帧直接把这些栈槽当作自己的局部变量来用 。

看原书的例子 ：

```
def add_two(a, b)
  sum = a + b
end
```

编译出的本地表：

```
Local Table
[ 2] sum
[ 3] b
[ 4] a
```

对应的指令：

```
0000 getlocal 4      # 取 a
0002 getlocal 3      # 取 b
0004 opt_plus        # a + b
0005 dup
0006 setlocal 2      # sum = ...
```

🎯 **我的洞见**：这个设计的第一性原理是**统一性**。

- 方法参数和方法内定义的局部变量，**在 YARV 层面没有任何区别**
    
- 参数只是在"方法调用前由调用方预先压栈"的局部变量
    
- 这意味着：访问参数的指令（`getlocal`）和访问局部变量的指令**完全相同**
    

这带来了几个工程优势：

1. **简化实现**：YARV 不需要为参数单独设计一套访问机制
    
2. **性能一致**：参数访问和局部变量访问速度一样快
    
3. **统一优化**：MJIT/RJIT 可以对两者做相同的优化
    

这也解释了为什么 Ruby 的方法调用约定是这样的：调用方负责把 `self`、位置参数、块参数等按顺序压栈，被调用方"醒来"时发现栈上已经是整齐的布局，直接用 `getlocal` 取用即可 。

### 3.2.3 动态变量访问

动态变量访问是 Ruby 闭包能力的基石。当一个块需要访问外层作用域的变量时，就用 `getdynamic`/`setdynamic` 指令，通过 DFP 链逐级上爬 。

在较老版本的 `insns.def` 中可以看到 ：

```
DEFINE_INSN getdynamic (dindex_t idx, rb_num_t level) () (VALUE val) {
  rb_num_t i;
  VALUE *dfp2 = GET_DFP();
  for (i = 0; i < level; i++) {
    dfp2 = (VALUE *)*dfp2;
  }
  val = *(dfp2 - idx);
}
```

算法非常直白：

1. 从当前 DFP 出发
    
2. 沿着 DFP 链向上爬 `level` 层（每层 DFP 的第一个槽位存着上一层 DFP 的指针）
    
3. 到达目标帧后，用 `idx` 索引定位变量
    

还是用前面的例子：

```
str = "The quick brown fox..."
10.times do
  str2 = "jumps over the lazy dog."
  puts "#{str} #{str2}"
end
```

块内访问 `str`（外层变量）时，编译出的指令是：

```
getdynamic str, 1    # level=1，表示要向上爬 1 层
```

🎯 **我的洞见（本节最核心的认知）**：

> **DFP 链是一个单向链表，把嵌套块的作用域串起来**。
> 
> 每一层块帧的 DFP 指向**创建它时的外层帧**。如果块嵌套块，就形成 DFP → DFP → DFP 的链。
> 
> `getdynamic idx, level` 就是在这条链上走 `level` 步，然后取 `idx` 位置的变量。

这个设计的含义：

1. **`level` 越大，访问越慢**：`getdynamic 0, 3` 比 `getdynamic 0, 1` 慢，因为要多爬两层
    
2. **深度嵌套块有隐性性能成本**：三层嵌套块访问最外层变量，要走 3 步 DFP 链
    
3. **`level=0` 的动态访问退化为局部访问**：这就是为什么 Ruby 2.0+ 可以把 `getlocal_OP__WC__0` 优化掉 `level` 参数
    

💡 **工业应用：Binding 对象**

动态变量访问在工程上的最直接体现是 **`Binding` 对象**​ 。Binding 封装了"某个代码位置的执行上下文"——包括变量、方法、`self`、可能的块。

```
def demo
  a = 1
  binding
end

b = demo
b.local_variable_get(:a)              # => 1
b.local_variable_set(:b, 3)           # 动态创建变量 b
b.local_variable_get(:b)              # => 3
b.eval("a + 1")                       # => 2
```

`Binding#local_variable_get` 的内部实现，本质上就是**拿着 Binding 里保存的环境指针，去做动态变量访问**​ ——和块访问外层变量的机制完全相同。

⚠️ **实际工程意义**：

- **DSL 设计**：Rails 的 `render`、RSpec 的 `describe/it`、Capybara 的测试 DSL，都大量依赖 Binding 和动态变量访问
    
- **元编程**：`eval` 在 Binding 上下文中执行，是 Ruby 元编程的利器
    
- **调试器**：Byebug、debug.gem 等调试器通过 Binding 让开发者在断点处检查和修改变量
    
- **性能警示**：深度嵌套块 + 频繁访问外层变量 + 长生命周期的 Proc，是 Ruby 应用的"性能三角"——三者同时出现时，既有 DFP 爬链开销，又有栈帧无法释放的内存压力
    

---

## 3.3 总结

本章我们从第一性原理梳理了 YARV 的执行模型：

1. **双栈架构**：YARV 是双栈机——内部栈执行指令（跟踪中间值/参数/返回值），Ruby 调用栈用 `rb_control_frame_t` 结构记录方法/块/lambda 的调用关系。PC、SP、self 保存在每个帧里 。
    
2. **简单脚本的执行**：YARV 机械地、一条一条地执行指令，PC 自动递增，SP 随压栈/弹栈移动。每条指令的"栈效应"静态已知，这是解释器高效和 JIT 可行的基础 。
    
3. **块调用的执行**：块 = `rb_block_t` 结构（一段 ISEQ + 一个捕获的 DFP）。`invokeblock` 指令激活块时，把 DFP 复制到新的块帧，使块既能访问自己的局部变量（通过 EP），又能访问外层变量（通过继承的 DFP）。**这就是 Ruby 闭包的实体**​ 。
    
4. **变量访问的两种方式**：
    
    - **局部访问**：`getlocal`/`setlocal`，O(1) 栈槽访问，`level` 通常为 0
        
    - **动态访问**：`getdynamic`/`setdynamic`，通过 DFP 链逐级上爬 `level` 层
        
    
5. **方法参数 = 局部变量**：唯一的区别是参数由调用方预先压栈。这带来了实现的统一性、性能的一致性、优化的通用性 。
    
6. **Binding 对象**：是动态变量访问在工程上的直接应用，让我们可以在任意时刻捕获和恢复执行上下文 。
    

🔑 **贯穿全章的核心洞见**：

> **YARV 的执行模型，本质上是用"栈 + 指针"模拟了 Ruby 的作用域链。**
> 
> - 内部栈模拟了"表达式求值"——无语义、纯机械
>     
> - 调用栈模拟了"方法调用链"——有语义、记录路径
>     
> - DFP 链模拟了"块嵌套链"——跨作用域、支持闭包
>     
> 
> 这三套栈/链的组合，恰好对应了 Ruby 语义的三个维度：**表达式、调用、作用域**。YARV 用最简的栈式架构，完整表达了 Ruby 这门动态语言的全部语义。

⚠️ **给后续章节的铺垫**：

- 下一章将深入**控制结构的执行**——`branchif`/`branchunless`/`jump` 如何实现 `if`/`while`/`until`，以及 Ruby 如何用**抛出异常**的方式实现 `break`
    
- 本章讲的 DFP 链、环境指针，将在后续章节的**闭包和块的生命周期管理**中再次出现
    
- **​ catch table（捕获表）**——本章未深入但原书提及——是 YARV 实现异常和 `ensure` 的关键机制，它记录了"在哪个指令范围抛出什么异常应该跳到哪里"，是连接"正常执行流"和"异常执行流"的桥梁
    
- 现代 Ruby 的 **MJIT/RJIT**​ 正是在 YARV 指令序列的基础上工作：识别热点指令序列（特别是 `send`/`invokeblock`），将其 JIT 成机器码。理解本章的双栈模型，是理解 JIT 如何"在两条世界线之间切换"的基础
    

📌 **工业实践建议**：

当你需要诊断 Ruby 性能问题时，按以下顺序排查：

1. **用 `RubyVM::InstructionSequence.disasm` 看字节码**——确认没有意外的对象分配或方法调用
    
2. **检查热点循环中的变量访问**——局部变量 > 实例变量 > 全局变量 > 动态变量
    
3. **警惕长生命周期的 Proc/块**——它们通过 DFP 链阻止外层栈帧释放
    
4. **用 `ObjectSpace` + `GC` 分析**——确认没有因为闭包捕获导致的内存泄漏
    
5. **考虑 MJIT/RJIT**——对计算密集型热点，JIT 可以带来 2-3 倍提升
    

