
# 第11章 读书笔记：Rubinius——用 Ruby 实现的 Ruby

> 💡 这一章回答的核心问题是：**如果让 Ruby 自己实现 Ruby，会是什么样？**
> 
> 从第一性原理看，Rubinius 解决的是一个极具野心的工程问题：**能不能用 Ruby 这门动态语言，来写 Ruby 虚拟机本身的核心类库？**​ 答案是——把虚拟机拆成两层：底层用 C++ 写"原语"（primitive），上层用 Ruby 写"绝大部分核心类"。这样既获得了 C++ 的性能和控制力，又让 Ruby 开发者能直接阅读、修改 Ruby 自身的实现。Rubinius 不仅是 Ruby 的一种实现，更是对 Smalltalk-80 " Everything is an object " 架构哲学的当代复兴 。

📌 **时代注记**：Rubinius 由 Evan Phoenix 领导开发，1.0 版本发布于 2011 年左右，底层 C++ 虚拟机使用 LLVM 做 JIT，并包含分代垃圾回收器 。原书基于这个时期的 Rubinius。虽然今天 Rubinius 的活跃度不如 MRI/JRuby/TruffleRuby，但它提出的"Ruby 实现 Ruby"理念和 bootstrap/common/delta 分层架构，深刻影响了后续 Ruby 实现的发展。本章的知识对理解所有"元实现"（语言实现语言自身）都有价值。

---

## 11 Rubinius：用 Ruby 实现的 Ruby

承接前几章——我们看了 MRI（C 写的 Ruby）和 JRuby（Java 写的 Ruby）。现在看第三条路径：**Rubinius 用 Ruby 自身实现了 Ruby 的大部分地区**​ 。

Rubinius 的两大部分 ：

1. **Rubinius 内核（Kernel）**：用 Ruby 编写，实现了许多语言特性和内置核心类（如 String、Array 等）
    
2. **Rubinius 虚拟机（VM）**：用 C++ 编写，执行字节码，负责垃圾回收、线程、JIT 等底层任务
    

🎯 **我的洞见**：Rubinius 的本质是一场**"自举（bootstrap）"实验**——

> **用 Ruby 写 Ruby 的核心类，用 C++ 写 Ruby 的虚拟机原语。**
> 
> 这不是简单的代码组织偏好，而是对 Smalltalk-80 架构哲学的彻底复兴 ：
> 
> - Smalltalk-80 的核心理念是"一切皆对象"，且虚拟机本身由紧凑的原语集构成
>     
> - Rubinius 借鉴了这一理念：**用 Ruby 实现语言的绝大部分语义，只为性能关键的极少数操作下沉到 C++**
>     
> - 这种方法让"VM 开发民主化"——Ruby 开发者无需精通 C/C++，就能贡献 VM 核心类的实现
>     

Rubinius 项目的核心开发者 Evan Phoenix 曾明确表示：Rubinius 采用 Smalltalk 的"一切皆对象"方法，同时通过原语（primitive）构建紧凑内核，从而获得卓越的灵活性和扩展性 。

---

## 11.1 Rubinius 内核和虚拟机

### 11.1.1 词法分析和解析

Rubinius 处理 Ruby 程序的方式与 MRI 类似——先词法分析，再语法解析，生成 AST，然后编译成字节码 。

但有一个关键差异：**Rubinius 的编译器本身就是用 Ruby 写的**（位于 `lib/compiler/` 目录下）。这意味着：

- 解析器、AST 构建、字节码生成——全部是 Ruby 代码
    
- 这些 Ruby 代码被 Rubinius 自己的 VM 执行
    
- 形成"鸡生蛋、蛋生鸡"的自举问题，通过预编译的 CodeDB 缓存解决
    

🎯 **我的洞见**：Rubinius 的解析编译流水线体现了"自举"的精髓——

```
Ruby 源码 → Ruby 写的词法分析器 → Ruby 写的语法解析器 → AST
         → Ruby 写的字节码生成器 → Rubinius 字节码 → C++ VM 执行
```

**整条流水线都是 Ruby 代码**。这带来的直接好处是：Ruby 开发者可以读、可以改、可以 debug 自己的编译器。这是 MRI（C 写的编译器）和 JRuby（Java 写的编译器）都无法提供的开发体验。

### 11.1.2 使用 Ruby 编译 Ruby

Rubinius 的编译器将 Ruby 源码编译为字节码指令，存储在 `CompiledCode` 对象中。这个 `CompiledCode` 对象包含 ：

- `iseq`：指令序列（opcode 的 Tuple）
    
- `literals`：字面量元组
    
- `local_count`：局部变量槽位数
    
- `stack_size`：最大栈深度
    
- `registers`：虚拟寄存器数
    
- `total_args`/`required_args`：参数信息
    

🎯 **我的洞见**：注意 `CompiledCode` 同时包含**字节码（iseq）和丰富的元数据**。这与 MRI 的 `rb_iseq_t` 结构异曲同工，但 Rubinius 的设计更"面向对象"——`CompiledCode` 本身就是一个 Ruby 对象，可以在运行时被检查、修改、甚至重新编译。

这种"编译产物也是对象"的设计，是 Ruby 元编程哲学在 VM 层的延伸——**既然"一切皆对象"，那么编译后的代码也应该是对象**。

### 11.1.3 Rubinius 字节码指令

Rubinius 的指令集设计哲学非常独特——**它不是传统的"正交精简指令集"，而是"语义丰富的中间表示"**​ ：

> "The instruction set is designed to provide a rich expression of a particular language in the instruction set, which is an intermediate representation..."
> 
> （指令集旨在为特定语言提供丰富的表达，它是一种中间表示...）

指令集包含 8 大类 ：

1. **通用栈指令**：`push_local`、`pop`、`dup`、`swap` 等
    
2. **寄存器指令**：对机器整数和双精度浮点数执行典型算术运算
    
3. **分支指令**：`goto`、`goto_if_true`、`goto_if_equal` 等
    
4. **并发指令**：线程、锁、同步原语
    
5. **POSIX 系统调用指令**：直接映射系统调用
    
6. **托管对象生命周期操作**：分配、GC 标记等
    
7. **断言指令**：不修改状态但可暂停
    
8. **插桩指令**：不修改状态但可发射信息
    

此外还有 **PEG（解析表达式语法）指令**和**方法/函数解析/缓存/调用指令**​ 。

🎯 **我的洞见（本节最核心的认知）**：

> **Rubinius 的指令集是"语义丰富"的，而非"正交精简"的。**
> 
> 传统 VM（如 JVM）追求指令集的正交性和细粒度，把高级语义丢弃，留给 JIT 去重新合成。Rubinius 反其道而行——**把 Ruby 的语义直接编码进指令集**，让指令本身就能表达"方法调用"、"块 yield"、"super 调用"等高级概念。
> 
> 这种设计的第一性原理是：**指令集是"特定语言的中间表示"**，而非"通用机器的指令集"。它让 Rubinius 的字节码更接近 Ruby 语义，简化了编译器的前端，同时让 JIT 后端能利用这些语义信息做更深度的优化。

**Direct-threaded dispatch（直接线程化分发）**：

Rubinius 的解释器执行模型非常精巧 。`Interpreter::prepare` 方法做两件事 ：

1. **第一遍**：统计需要 GC 跟踪的引用槽位（rcount）
    
2. **第二遍**：把符号化的 opcode ID **替换为对应的 C++ 函数指针**，直接存入 opcode 数组
    

执行时，`Interpreter::execute` 直接调用函数指针——**没有中央 switch 语句**，每条指令的执行就是一次 C++ 函数调用。

```
// 概念示意：
opcode[0] = &Instructions::push_local;  // 不再是 case PUSH_LOCAL:
opcode[1] = &Instructions::send_stack;  // 不再是 case SEND_STACK:
// 执行时：
for (int ip = 0; ; ) {
    ip = opcode[ip](state, call_frame);  // 直接调用 C++ 函数
}
```

🎯 **我的洞见**：direct-threaded dispatch 是现代解释器的标配优化（MRI 的 YARV 也用类似技术 ）。它的精髓是——**用 C++ 函数指针数组替代 switch 分发**，让 CPU 的分支预测器能更好地工作，同时让每条指令的执行在 stack trace 中清晰可见（这正是 Rubinius 调试器强大的原因 ）。

### 11.1.4 Ruby 和 C++ 一起工作

这是 Rubinius 最精妙的设计——**Ruby 代码和 C++ 代码如何协作**？

答案是 **`Rubinius.primitive` 指令**​ 。当一个 Ruby 方法以 `Rubinius.primitive :symbol` 开头时，它告诉 Rubinius 编译器：这个方法实际由 C++ 实现，symbol 对应 C++ 原语表中的函数。

```
# core/bootstrap/string.rb 中的示例
class String
  def [](index, length=nil)
    Rubinius.primitive :string_aref
    # 如果 C++ 原语返回 cNotYet，则执行下面的 Ruby 代码作为 fallback
    # ...
  end
end
```

对应的 C++ 实现 ：

```
// machine/builtin/string.cpp
Object* String::aref(STATE, Fixnum* index, Object* length) {
    // ... C++ 实现 String#[] 的逻辑
}
```

🎯 **我的洞见**：`Rubinius.primitive` 机制实现了**"Ruby 接口 + C++ 实现"的无缝对接**：

- Ruby 层定义方法签名、参数处理、fallback 逻辑
    
- C++ 层实现性能关键的核心操作
    
- 如果 C++ 原语返回 `cNotYet` 哨兵值，解释器继续执行 Ruby 方法体的剩余部分
    

这种模式让 Rubinius 的核心类既有 Ruby 的可读性，又有 C++ 的性能。更重要的是——**它让 Ruby 开发者能读懂 95% 的核心类实现**（因为那是 Ruby 代码），只在少数热点处看到 C++ 原语。

### 11.1.5 使用 C++ 对象实现 Ruby 对象

Rubinius 用 C++ 对象表示 Ruby 对象，正如 MRI 用 C 结构体（RObject、RClass）表示 。

```
定义类时：Rubinius 创建 Class C++ 类的实例
创建对象时：Rubinius 创建 Object C++ 类的实例
```

Rubinius VM 的 README 明确说明 ：

> "Classes defined in builtin/*.hpp are C++ classes mapped directly to ruby objects."
> 
> （builtin/*.hpp 中定义的类是直接映射到 Ruby 对象的 C++ 类。）

关键约束 ：

- **不允许虚函数**：C++ 会在每个对象中插入 vptr 指针，但 Rubinius 必须完全控制对象的内存布局
    
- **只允许单继承**：保持数据成员顺序的一致性
    

🎯 **我的洞见**：这个约束揭示了 Rubinius 对象模型的第一性原理——

> **Ruby 对象在内存中的布局，必须完全由 Rubinius 自己控制，不能让 C++ 编译器"偷偷"插入任何东西。**
> 
> 这就是为什么：
> 
> - 禁用虚函数（避免 vptr 插入）
>     
> - 只用单继承（保持内存布局可预测）
>     
> - C++ 类只是"内存布局的声明"，真正的多态通过 Rubinius 自己的 klass_ 指针实现
>     

每个 Rubinius 的 C++ 对象（如 `Object` 类）包含 ：

- **klass_ 指针**：指向对象的类（对应 MRI 的 RBasic.klass）
    
- **ivars_ 指针**：指向实例变量存储
    

这跟 MRI 的 `RObject` 结构（`RBasic` + `ivptr`）在概念上完全对应——**只是用 C++ 类替代了 C 结构体，用 C++ 的封装替代了 C 的裸指针操作**。

### 11.1.6 Rubinius 中的（栈）回溯

Rubinius 使用**影子栈（shadow stack）**技术 ——每个调用帧都是适当的数据结构，可以轻松检查，而无需深入 CPU 寄存器。

这就是为什么 Rubinius 的回溯（backtrace）如此强大：

```
# 在 Rubinius 中，你可以这样探索回溯
def method_a
  method_b
end

def method_b
  puts caller  # 清晰的 Ruby 级回溯
  # 甚至可以访问 MethodContext 等内部数据结构
end
```

🎯 **我的洞见**：Rubinius 的影子栈设计，体现了对"可调试性"的极致追求——

- **CPU 原生栈**：方法调用信息散布在寄存器和栈内存中，难以 introspect
    
- **影子栈**：每个调用帧都是 Rubinius 对象，可以像普通对象一样被检查、传递、序列化
    

这让 Rubinius 能够提供：

- **全速调试器**：可以在不暂停 VM 的情况下 inspect 调用栈
    
- **MethodContext 访问**：Ruby 代码可以直接访问内部的调用帧对象
    
- **图像持久化**：未来计划实现 Smalltalk 风格的"镜像持久化"——把整个应用状态保存到磁盘
    

⚠ **工业现实**：这种"可调试性优先"的设计，让 Rubinius 在开发体验上独树一帜。但代价是——影子栈的维护开销，让 Rubinius 在原始性能上未必能超越 MRI。这是工程上经典的"可调试性 vs 性能"权衡。

---

## 11.2 Rubinius 和 MRI 中的数组

数组是 Ruby 中最常用的数据结构之一。对比 MRI 和 Rubinius 的数组实现，能深刻理解两种设计哲学的差异。

### 11.2.1 MRI 中的数组

MRI 的 `RArray` 结构使用经典的"溢出"策略 ：

```
RArray
├── RBasic (flags + klass)
├── len: 数组长度
├── capa: 容量（已分配槽位数）
└── ptr: 指向元素数组的指针
```

当数组元素数 ≤ 3 时，ptr 直接内嵌在 RArray 结构体中（embedded array）；超过 3 个时，ptr 指向堆上分配的数组。

**Array#shift 在 MRI 中的实现**：需要把所有剩余元素向左移动一位，O(n) 时间复杂度。频繁调用 shift 会导致大量内存拷贝。

### 11.2.2 Rubinius 中的数组

Rubinius 的 Array 采用更精巧的设计。C++ 层的 `Array` 类有三个核心字段 ：

```
class Array : public Object {
    Fixnum* total_;      // 数组长度
    Tuple* tuple_;       // 指向底层 Tuple 对象（存储实际数据）
    Fixnum* start_;      // 在 tuple 中的起始偏移
};
```

**关键创新：start_ 偏移量**。

`Tuple` 是 Rubinius 的底层存储类——一个固定大小的、由 GC 直接管理的 `Object*` 引用连续序列 。它不暴露为公开的 Ruby 类，纯粹作为 Array 等内置类型的存储基础设施。

🎯 **我的洞见（本节最核心的认知）**：

> **Rubinius 的 Array 用 `start_` 偏移量实现了 O(1) 的 shift/unshift。**
> 
> 当调用 `Array#shift` 时，Rubinius 不需要移动任何元素——只需要把 `start_` 加 1，`total_` 减 1。被"移除"的元素在 tuple 中依然存在（直到 tuple 被 GC 回收或数组被压缩），但对用户来说，数组的第一个元素已经是原来第二个元素了。
> 
> 这是用"空间换时间"的经典优化：
> 
> - **读操作**：O(1)——直接通过 `tuple_[start_ + index]` 访问
>     
> - **shift/unshift**：O(1)——只修改 `start_` 和 `total_`
>     
> - **代价**：tuple 中可能包含已被逻辑上"移除"的元素，造成轻微的空间浪费
>     

相比之下，MRI 的 Array#shift 需要 O(n) 时间移动元素。Rubinius 的设计在长期需要频繁 shift/unshift 的场景（如队列、滑动窗口）中具备显著的性能优势。

### 11.2.3 阅读 Array#shift 源码

Rubinius 中 `Array#shift` 的 Ruby 实现 ：

```
# kernel/common/array.rb (第 848 行附近)
def shift(n=undefined)
  Rubinius.check_frozen
  
  if n.equal? undefined
    return nil if @total == 0
    obj = @tuple.at @start
    @tuple.put @start, nil
    @start += 1
    @total -= 1
    obj
  else
    n = Rubinius::Type.coerce_to(n, Fixnum, :to_int)
    raise ArgumentError, "negative array size" if n < 0
    slice!(0, n)
  end
end
```

🎯 **我的洞见**：这段 Ruby 代码的美妙之处在于——**它如此清晰，以至于不需要任何 C++ 知识就能完全理解 Array#shift 的语义**：

- `@total == 0` 检查数组是否为空
    
- `@tuple.at @start` 取出当前第一个元素
    
- `@tuple.put @start, nil` 把该位置设为 nil（断开引用，便于 GC）
    
- `@start += 1` 和 `@total -= 1` 调整偏移和长度
    
- 返回取出的元素
    

**这正是 Rubinius "Ruby 实现 Ruby" 哲学的胜利**——核心类的方法体是纯 Ruby 代码，可读、可改、可 debug。而在 MRI 中，Array#shift 是 C 函数，Ruby 开发者无法直接阅读和修改。

### 11.2.4 修改 Array#shift 方法

Rubinius 允许你在运行时直接修改核心类的方法——这是它最令人兴奋的能力 ：

```
# 重新定义 Array#shift，添加日志
class Array
  alias_method :original_shift, :shift
  
  def shift(n=undefined)
    puts "shift called on #{self.inspect}"
    original_shift(n)
  end
end

[1, 2, 3].shift  # 输出: shift called on [1, 2, 3]
                # => 1
```

更激进的实验——直接修改 `kernel/common/array.rb` 源文件，重新编译 Rubinius，你就有了一个"你自己定制的 Ruby 实现"。

🎯 **我的洞见**：这种"可修改的核心类"能力，是 Rubinius 对 Ruby 元编程哲学的极致延伸——

- **MRI**：核心类用 C 写，普通 Ruby 开发者无法修改
    
- **JRuby**：核心类用 Java 写，普通 Ruby 开发者无法修改
    
- **Rubinius**：核心类用 Ruby 写，任何人都可以读、可以改、可以实验
    

这降低了 VM 开发的门槛，让 Ruby 社区能直接参与 Ruby 自身的演进。Rubinius 甚至采用了"首次提交补丁者即获提交权限"的极度开放的社区模式 。

⚠ **工业现实**：这种开放性是一把双刃剑。它极大地促进了实验和创新（Rubinius 贡献了大量 Ruby 实现的技术遗产，包括 RubySpec 规范测试套件、现代化的 FFI 接口等），但也导致项目方向分散、稳定性挑战。这也是为什么今天 Rubinius 不再是主流 Ruby 实现，但它的技术遗产（RubySpec、FFI、分代 GC 思路）被 MRI 和其他实现广泛吸收。

---

## 11.3 总结

本章我们从第一性原理梳理了 Rubinius 的架构设计与核心创新：

1. **两层架构**：Rubinius 由 **C++ 虚拟机**（执行字节码、GC、JIT、线程）和 **Ruby 内核**（核心类、编译器、库）组成 。这是"Ruby 实现 Ruby"的物理基础。
    
2. **Smalltalk-80 哲学复兴**：Rubinius 借鉴 Smalltalk 的"一切皆对象"+ 紧凑原语内核的设计 ，让 Ruby 开发者能直接参与 VM 开发。
    
3. **Ruby 与 C++ 协作**：通过 `Rubinius.primitive` 指令，Ruby 方法可以调用 C++ 原语 。C++ 实现性能关键操作，Ruby 实现接口和 fallback 逻辑。如果 C++ 原语返回 `cNotYet`，解释器继续执行 Ruby 方法体 。
    
4. **语义丰富的指令集**：8 大类指令（栈、寄存器、分支、并发、系统调用、对象生命周期、断言、插桩），外加 PEG 和方法调用指令 。指令集是"特定语言的中间表示"，而非"正交精简指令集"。
    
5. **Direct-threaded dispatch**：`Interpreter::prepare` 把符号 opcode 替换为 C++ 函数指针，`Interpreter::execute` 直接调用函数指针，无中央 switch 。这让 CPU 分支预测更高效，也让 stack trace 清晰可读。
    
6. **C++ 对象映射 Ruby 对象**：`builtin/*.hpp` 中的 C++ 类直接映射到 Ruby 对象 。约束：禁用虚函数（避免 vptr 插入）、只用单继承（保持内存布局可控）。
    
7. **影子栈**：每个调用帧都是 Rubinius 对象，可轻松 introspect 。这让 Rubinius 拥有强大的调试能力和未来"镜像持久化"的潜力。
    
8. **Array 的创新设计**：C++ 层 `Array` 类用 `total_` + `tuple_` + `start_` 三个字段 。`start_` 偏移量让 `shift`/`unshift` 达到 O(1) 时间复杂度，而 MRI 需要 O(n) 移动元素。
    
9. **Array#shift 的 Ruby 实现**：核心方法体是纯 Ruby 代码 ，可读、可改、可实验。这是"Ruby 实现 Ruby"哲学的直接体现。
    
10. **可修改的核心类**：Rubinius 允许运行时修改甚至重新编译核心类 ，极大降低了 VM 开发门槛。
    

🔑 **贯穿全章的核心洞见**：

> **Rubinius 的本质，是"自举（bootstrap）"理念在 Ruby 世界的彻底实践——用 Ruby 写 Ruby 自身。**
> 
> 这场实验的第一性原理可以浓缩为三句话：
> 
> 1. **语义用 Ruby 写**：核心类的方法体是 Ruby 代码，可读、可改、可 debug
>     
> 2. **性能用 C++ 写**：原语操作下沉到 C++，通过 `Rubinius.primitive` 机制对接
>     
> 3. **VM 用 C++ 写**：解释器、JIT、GC、线程调度——这些是 Ruby 表达不了的底层机制，必须用 C++
>     
> 
> 这种"分层"不是权宜之计，而是对 Smalltalk-80 架构哲学的忠实复兴 。它让 Rubinius 获得了无与伦比的**可调试性**和**可扩展性**，代价是原始性能和项目稳定性。
> 
> Rubinius 最终没有成为主流 Ruby 实现，但它的技术遗产深远：
> 
> - **RubySpec**：为 Ruby 语言提供了精确的规范测试套件，被 MRI、JRuby、TruffleRuby 广泛采用
>     
> - **现代化 FFI**：Rubinius 创造了第一个现代化的 Ruby FFI，让 Ruby 代码能干净地调用 C 函数
>     
> - **分代 GC 思路**：Rubinius 的精确分代垃圾回收器，影响了后续 Ruby GC 的发展
>     
> - **"Ruby 实现 Ruby"理念**：启发了后来的 TruffleRuby（用 Java/Sulong 实现 Ruby 自举）
>     

⚠ **给全书的总结与展望**：

通过前 11 章，我们走完了 Ruby 内部实现的完整地图：

|实现|语言|虚拟机|对象模型|现状|
|---|---|---|---|---|
|**MRI/CRuby**​|C|YARV（栈式解释器）|C 结构体（RObject/RClass）|官方主流实现|
|**JRuby**​|Java|JVM + 自研 IR|Java 类映射 Ruby 对象|企业级、JVM 生态|
|**Rubinius**​|Ruby + C++|C++ VM + LLVM JIT|C++ 类映射 Ruby 对象|技术实验、遗产丰富|
|**TruffleRuby**​|Java + 自举|GraalVM Truffle|自举的 Ruby 实现|性能先锋|

**四条路径，一个目标——让 Ruby 更快、更灵活、更强大。**

Rubinius 虽然不再是主流，但它证明了一件事：**Ruby 足够强大，可以用自身来实现自身**。这种"自举"能力，是 Ruby 语言成熟度的终极证明，也是 Matz 设计哲学的伟大胜利——"Ruby 优化了程序员的快乐"，而这种快乐，甚至延伸到了"写 Ruby 虚拟机"这件事上。

📌 **工业实践启示**：

1. **理解 Rubinius 的设计，能帮助你理解 Ruby 的"一切皆对象"哲学**——它不只是语法糖，而是深入到 VM 层的设计原则
    
2. **Rubinius 的 bootstrap/common/delta 分层架构**​ ，预示了现代软件架构中"分层加载、渐进增强"的思想
    
3. **`Rubinius.primitive` 机制**展示了"高级语言 + 原语下沉"的混合编程范式，这种范式在 WebAssembly、CUDA 等现代系统中广泛存在
    
4. **Rubinius 的 Array 设计**（start_ 偏移量）是数据结构优化的经典案例——用空间换时间，用 O(1) 的偏移调整替代 O(n) 的元素移动
    
5. **Rubinius 的技术遗产**（RubySpec、FFI、分代 GC）至今仍在造福 Ruby 社区——理解它，就是理解现代 Ruby 的历史根基
    

