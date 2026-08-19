
# 第10章 读书笔记：JRuby——基于 JVM 的 Ruby

> 💡 这一章回答的核心问题是：**如果把 Ruby 从 C 语言世界搬到 Java 世界，它还是 Ruby 吗？**
> 
> 从第一性原理看，JRuby 解决的是一个根本性的移植问题：**如何让一门为 C 运行时设计的动态语言，在 JVM 上既保持语义兼容，又获得 JVM 的 JIT、GC、多线程和 Java 生态的优势？**​ 答案是——把 Ruby 源码编译成 JVM 字节码，用 Java 类表示 Ruby 对象，用 JVM 的 JIT 替代 YARV 的解释执行。这不是"模拟 Ruby"，而是"让 Ruby 成为 JVM 的一等公民"。

📌 **时代注记**：原书基于较早版本的 JRuby，描述的是"JRuby 把 Ruby 编译成 Java 字节码"的核心架构。今天的 JRuby 9.x/10.x 依然遵循这一架构，但增加了 IR（中间表示）层、invokedynamic 支持，并目标兼容 Ruby 3.1-3.4 语法 。本章的知识对理解现代 JRuby 依然是基础。

---

## 10 JRuby: 基于 JVM 的 Ruby

承接前 9 章——我们一直在 MRI（CRuby）的世界里探索：C 写的解释器、YARV 字节码、手工管理的对象结构、手工实现的散列表和字符串。但 Ruby 不止 MRI 一种实现。本章进入 JRuby 的世界：**用 Java 写成的 Ruby 实现，运行在 JVM 上**​ 。

🎯 **我的洞见**：JRuby 的存在本身就是 Ruby 语言设计哲学的证明——**"Ruby 是一门语言规范，而非一个 C 程序"**。MRI 只是 Ruby 的一个实现，JRuby 是另一个实现。两者共享相同的语法、相同的对象模型、相同的语义，但底层机制截然不同。这种"规范与实现分离"的设计，让 Ruby 社区能探索不同的性能路径（JVM 上的 JRuby、C++/Ruby 混合的 Rubinius、GraalVM 上的 TruffleRuby）。

---

## 10.1 使用 MRI 和 JRuby 运行程序

原书通过对比两个命令的启动流程，揭示了根本差异 ：

### MRI 的启动

```
$ ruby script.rb
```

`ruby` 命令直接启动一个**C 语言编写的二进制可执行文件**。这个二进制是 Ruby 源码在构建时编译出来的，它内部包含：

- 词法分析器（手写 C 代码）
    
- 语法解析器（Bison 从 `parse.y` 生成的 C 代码）
    
- YARV 虚拟机（解释执行字节码）
    
- 所有核心类的 C 实现
    

### JRuby 的启动

```
$ jruby script.rb
```

`jruby` 命令**不是一个二进制可执行文件**，而是一个 shell 脚本，它最终调用：

```
java -Xbootclasspath:jruby.jar org.jruby.Main script.rb
```

也就是说：

- `java` 命令启动 JVM（JVM 本身是 C 写的二进制）
    
- JVM 加载 `jruby.jar`（包含 JRuby 的 Java 实现）
    
- JRuby 在 JVM 内部解析和执行 `script.rb`
    

🎯 **我的洞见（本节最核心的认知）**：

> **MRI 是"裸金属"路径——C 二进制直接执行 Ruby；JRuby 是"托管"路径——JVM 执行 Java 代码，Java 代码再执行 Ruby。**
> 
> 这两条路径的本质差异：
> 
> - MRI：Ruby 源码 → C 解析器 → YARV 字节码 → YARV 解释器执行
>     
> - JRuby：Ruby 源码 → Java 解析器 → AST → JVM 字节码 → JVM 执行
>     
> 
> JRuby 不生成 YARV 字节码，而是**直接生成 JVM 字节码**。这意味着 Ruby 方法在 JVM 看来就是一个 Java 方法，可以享受 JVM 的全部优化 。

---

## 10.1.1 JRuby 如何解析和编译代码

JRuby 的解析编译流水线 ：

```
Ruby 源码 → RubyLexer（词法分析）→ RubyParser（语法解析）→ AST → IR → JVM 字节码
```

**关键组件**：

1. **RubyLexer**：手写的词法分析器，输出 token 流
    
2. **RubyParser**：使用 **Jay**（Java 版的 Bison/Yacc）生成的语法解析器，从 token 流构建 AST
    
    - 原书特别指出：MRI 用 Bison，JRuby 用 Jay——两者算法同源
        
    
3. **AST → IR**：AST 被转换为 **IR（Intermediate Representation）**——一种线性寄存器传输指令的中间表示
    
4. **IR 优化**：在 IR 上做基于 Ruby 语义的高层优化
    
5. **IR → JVM 字节码**：通过 **ASM 库**在内存中生成 JVM 字节码类
    

🎯 **我的洞见**：JRuby 引入 IR 层是工程上的关键决策。为什么不直接 AST → JVM 字节码？

- **IR 是"与语言无关"的中间表示**：它在 AST（语言相关）和 JVM 字节码（平台相关）之间提供了一个缓冲层
    
- **IR 支持高层优化**：利用 Ruby 语义知识做优化（如方法内联、逃逸分析）
    
- **IR 同时支持解释执行和 JIT 编译**：IR 可以先被解释执行（快速启动），达到调用阈值后再 JIT 成 JVM 字节码
    
- **IR 是 CFG 结构**：基本块的线性寄存器传输指令，便于优化
    

这种设计让 JRuby 成为一个**混合模式引擎（mixed-mode engine）**——先解释 IR，热点方法 JIT 成 JVM 字节码，JVM 字节码再被 HotSpot 的 C1/C2 编译器 JIT 成机器码 。

⚠ **与原书的差异**：原书那个时代 JRuby 的编译器相对简单——从 AST 直接编译成 JVM 字节码。现代 JRuby 引入了 IR 层，**这是一个重要的架构演进**，让 JRuby 的优化能力与 HotSpot 更好地配合 。

### 工业实际：JRuby 编译器设计要点

JRuby 编译器有一个著名的设计目标：**给定一个 `.rb` 文件，生成一个 `.class` 文件**。

输出的 `.class` 文件至少包含：

- `main()`：命令行入口，创建 JRuby 运行时并启动
    
- `load()`：脚本的顶层加载方法（包含前置和后置设置）
    
- `run()`：脚本主体的执行方法（JIT 使用）
    
- `file()`：脚本主体的实际入口
    

然后根据文件内容，还会生成：

- 普通方法定义 → Java 方法
    
- 类和模块体 → Java 方法
    
- 闭包体 → Java 方法
    
- rescue/ensure 体 → 合成方法
    

**关键设计**：只有类体、rescue/ensure 体和链式顶层脚本方法会被直接调用；其他方法绑定到 MOP（Meta-Object Protocol）中，运行时通过反射调用 。

---

## 10.1.2 JRuby 如何执行代码

执行流程的核心 ：

1. **JRuby 运行时初始化**：JVM 加载 `jruby.jar`，创建 `Ruby` 对象（代表 JRuby 运行时实例）
    
2. **解析 Ruby 代码**：`RubyParser` 生成 AST
    
3. **AST → IR**：`IRBuilder` 将 AST 转换为 IR 作用域（如 `IRMethod`、`IRClosure`）
    
4. **解释执行 IR**：初期直接解释 IR，快速启动
    
5. **JIT 编译**：当方法调用次数达到阈值（约 50 次），IR 被 JIT 编译为 JVM 字节码
    
6. **HotSpot 进一步优化**：生成的 JVM 字节码被 HotSpot 的 C1 编译器编译（简单优化、快速产出），热点再被 C2 编译（深度优化）
    

🎯 **我的洞见**：这是 JRuby 性能模型的核心——**两层 JIT**：

```
Ruby 方法 → (JRuby JIT) → JVM 字节码 → (HotSpot JIT) → 机器码
```

- **第一层 JIT**（JRuby）：Ruby 语义 → JVM 字节码。利用 Ruby 语义知识做优化（如 `invokedynamic` 支持多态方法调用）
    
- **第二层 JIT**（HotSpot）：JVM 字节码 → 机器码。利用 JVM 运行时的 profiling 信息做优化（如内联、逃逸分析、锁消除）
    

这种"双重 JIT"让 JRuby 在长期运行的服务端应用中表现出色——**预热（warm-up）完成后，性能往往超过 MRI**​ 。

💡 **工业应用**：

- **长时间运行的服务端应用**：JRuby on Rails 应用部署在 JVM 上，预热后吞吐量高、延迟稳定
    
- **Java 互操作**：直接在 Ruby 中调用 Java 类库（如 Apache POI 处理 Excel、Lucene 做全文检索）
    
- **现有 Java 企业基础设施**：JRuby 应用可以无缝接入 Java 生态的监控、APM、连接池等基础设施
    

---

## 10.1.3 用 Java 类实现 Ruby 类

这是 JRuby 最优雅的设计——**Ruby 的每个核心类型，都对应一个 Java 类**​ ：

|Ruby 类型|JRuby 的 Java 实现|
|---|---|
|所有 Ruby 对象|`IRubyObject` 接口|
|`Object`|`RubyBasicObject`|
|`String`|`RubyString`|
|`Array`|`RubyArray`|
|`Hash`|`RubyHash`|
|`Module`/`Class`|`RubyModule`/`RubyClass`|

每个 Ruby 对象在 JVM 堆上都是一个 Java 对象，实现 `IRubyObject` 接口。Ruby 的类层级通过 Java 的类继承来映射 。

🎯 **我的洞见（本节最核心的认知）**：

> **JRuby 用 Java 的类系统"模拟"了 Ruby 的对象模型。**
> 
> 这不是简单的"包装"——而是深度的双向映射：
> 
> - Ruby 对象 ↔ Java 对象（`RubyString` 实例）
>     
> - Ruby 类 ↔ Java 类（`RubyString` 类）
>     
> - Ruby 方法调用 ↔ Java 方法调用（通过 `invokedynamic` 或反射）
>     
> - Ruby 的 `self` ↔ Java 的 `this`
>     
> 
> 这种映射让 JRuby 能**直接利用 JVM 的 JIT 优化**——当 HotSpot 看到 `RubyString` 对象的 `toString()` 方法被频繁调用，它可以内联、去虚化、甚至做逃逸分析把对象分配在栈上。

但代价是——**C 扩展无法直接使用**。MRI 的 C 扩展假设 Ruby 对象是 C 结构体（`RObject`、`RString` 等），而 JRuby 的 Ruby 对象是 Java 对象。这是 JRuby 长期以来的最大短板 。

⚠ **工业现实**：

```
# MRI 上能用的 C 扩展
require 'nokogiri'  # 底层是 C 库
require 'pg'        # PostgreSQL C 客户端
require 'rmagick'   # ImageMagick C 绑定

# 在 JRuby 上：
# - 要么使用纯 Java 替代品（如 jruby-pg 使用 JDBC）
# - 要么通过 FFI 调用原生库（性能损失）
# - 要么用 JRuby 的 Java 互操作直接调用 Java 库
```

这就是为什么 JRuby 社区发展出了"Java 化"的替代生态：

- `nokogiri` → `Nokogiri::HTML` 在 JRuby 上用 Xerces 实现
    
- `pg` → `jruby-pg` 使用 JDBC
    
- `rmagick` → `img_scalr` 使用 Java 的 BufferedImage
    

---

## 10.1.4 使用 `-J-XX:+PrintCompilation` 选项

原书 Experiment 10-1 引导我们用 JVM 的工具观察 JRuby 的 JIT 过程 ：

```
jruby -J-XX:+PrintCompilation script.rb
```

`-J` 前缀表示把后面的参数传递给 JVM（而非 JRuby）。`-XX:+PrintCompilation` 让 HotSpot 在 JIT 编译每个方法时打印一行日志 ：

```
1  java.lang.String::hashCode (64 bytes)
 2  org.jruby.RubyString::toString (28 bytes)
 3  org.jruby.RubyArray::get (38 bytes)
 ...
```

每行表示一个 Java 方法被 JIT 编译成机器码。当你运行 Ruby 程序时，会看到大量的 `org.jruby.*` 方法被编译——这些都是 JRuby 运行时本身的方法。而你的 Ruby 方法，会被编译成类似 `script_file$block_n_method_name` 这样的 JVM 方法 。

🎯 **我的洞见**：这个实验揭示了 JRuby 性能模型的真相——

> **你写的每一行 Ruby 代码，最终都变成了 JVM 方法。**
> 
> 当你定义 `def add(a, b); a + b; end`，JRuby 会生成一个 Java 方法（在合成的 `.class` 文件中），HotSpot 会把它 JIT 成机器码。从 JVM 的角度看，Ruby 方法调用就是 Java 方法调用。

这就是为什么 JRuby 在长期运行后能超越 MRI——**HotSpot 的 C2 编译器做了 MRI 的 YARV 解释器做不到的深度优化**：

- **方法内联**：把 `add` 的调用内联到调用方
    
- **逃逸分析**：如果 `add` 返回的字符串没有逃逸出方法，直接在栈上分配
    
- **去虚化**：如果某个方法调用点只有一种实际类型，去掉虚方法分派开销
    
- **锁消除**：消除不必要的同步
    

---

## 10.1.5 JIT 是否提升了 JRuby 程序的性能

原书的实验结论：**是的，JIT 显著提升了 JRuby 的性能，但需要预热**​ 。

典型表现：

```
第 1 次调用：慢（解释执行 IR）
第 2-10 次调用：逐渐加快（JRuby JIT 成 JVM 字节码）
第 11-50 次调用：更快（HotSpot C1 编译）
第 50+ 次调用：达到峰值性能（HotSpot C2 编译）
```

🎯 **我的洞见**：JRuby 的性能特征是"**慢启动、快长跑**"：

- **冷启动慢**：因为要先解释 IR，再 JIT 成字节码，再 JIT 成机器码——多层编译的累积开销
    
- **稳态快**：一旦达到峰值，JRuby 往往**快于 MRI**，因为 HotSpot C2 的优化能力远超 YARV 解释器
    

这就是为什么 JRuby 适合：

- ✅ 长时间运行的服务端应用（Rails 服务器、后台任务）
    
- ✅ 计算密集型任务（数值计算、数据处理）
    
- ❌ 短生命周期的脚本（CLI 工具、rake 任务）——预热开销大于收益
    

💡 **工业数据**：根据 Chris Seaton 的基准测试 ，JRuby 1.7.20 在启用 invokedynamic 后，相对 MRI 2.2.2 有 **3.6 倍**的加速。而基于 GraalVM 的 TruffleRuby（JRuby 的一个分支演进）在某些基准上达到 **950 倍**的加速 。

⚠ **现代 JRuby 的 invokedynamic**：

`invokedynamic` 是 JVM 为动态语言添加的指令，JRuby 是其首批重要用户 。但**在今天的 JRuby 中，invokedynamic 默认仍未启用**——因为它增加了预热时间 。要在 JRuby 9.x 中启用：

```
jruby --indy script.rb
```

---

## 10.2 JRuby 和 MRI 中的字符串

字符串是 Ruby 程序中最常用的对象类型之一。了解 JRuby 和 MRI 如何存储字符串，对性能优化至关重要。

### 10.2.1 JRuby 和 MRI 如何保存字符串数据

**MRI 的字符串存储**​ ：

MRI 的 `RString` 结构包含：

- 一个字节数组（实际字符数据）
    
- 长度字段
    
- 编码（encoding）
    
- 代码范围（code range，作为缓存优化）
    
- 共享指针（用于写时复制）
    

**JRuby 的字符串存储**​ ：

JRuby 的 `RubyString` 类使用 `ByteList` 类来跟踪实际的字符串数据：

- `ByteList` 包含一个 `byte[]` 数组
    
- `realSize` 实例变量存储字符串长度
    
- 编码和代码范围作为 `RubyString` 的字段
    

🎯 **我的洞见**：两者的内存布局惊人地相似——

|字段|MRI (RString)|JRuby (RubyString + ByteList)|
|---|---|---|
|字节数据|`char *ptr`|`byte[] bytes`|
|长度|`long len`|`int realSize`|
|编码|`rb_encoding *encoding`|`Encoding encoding`|
|代码范围|`int code_range`|`int codeRange`|
|共享|`RString *shared`|`ByteList` 共享引用|

**JRuby 没有复用 Java 的 `String` 类型**，原因是 ：

1. Ruby 字符串是可变的，Java `String` 是不可变的
    
2. Ruby 字符串有多种编码，Java `String` 内部固定为 UTF-16
    
3. 每次转换编码会带来昂贵的开销
    

⚠ **关键限制**：由于 JRuby 使用 Java 的 `byte[]`，**字符串长度上限为 2GB**（Java 数组的最大长度）。

### 10.2.2 写时复制（Copy-on-Write）

这是本章最重要的性能优化。

**核心思想**：当两个字符串的值相同时，让它们**共享同一个底层字节数组**，而不是各复制一份。

MRI 的写时复制 ：

```
str = "Geometry is knowledge of the eternally existent."
# 创建一个 RString 结构，指向字符数组

str2 = str.dup
# 创建第二个 RString 结构，但 shared 指针指向第一个 RString
# 两者共享同一个字符数组
```

JRuby 的写时复制 ：

```
str = "Geometry is knowledge of the eternally existent."
# 创建 RubyString + ByteList，ByteList 指向 byte[]

str2 = str.dup
# 创建新的 RubyString 和 ByteList
# 但新的 ByteList 的 bytes 指向同一个 byte[]
# 不复制实际字符数据
```

🎯 **我的洞见**：写时复制的第一性原理是——**"不可变共享，可变复制"**。

- 字符串字面量、子串、dup 的结果——这些在修改前都不会变
    
- 共享底层数组节省内存和时间
    
- 只有当其中一个字符串被修改时，才触发真正的复制
    

MRI 的 `RString` 通过 `shared` 指针实现共享；JRuby 的 `ByteList` 通过共享 `byte[]` 引用实现共享 。

### 10.2.3 创建唯一且非共享的字符串

原书 Experiment 10-2 探讨了一个微妙的问题：如何创建一个**真正独立、不与任何字符串共享**的字符串？

```
str = "Geometry is knowledge of the eternally existent."
# 此时 str 实际上是共享的——Ruby 内部立即共享了这个字符串
```

🎯 **我的洞见**：即使是看似简单的字符串字面量，MRI 和 JRuby 也会立即应用写时复制——这意味着**几乎所有字符串在创建时都是共享的**。

要创建非共享字符串，需要强制触发复制：

```
str = "Geometry is knowledge of the eternally existent.".dup
# 虽然 dup 通常共享，但在某些情况下会创建独立副本

# 更可靠的方式：通过字符串操作强制复制
str = "prefix" + "Geometry is knowledge" + " of the eternally existent."
# 拼接操作创建了全新的、非共享的字符串
```

### 10.2.4 可视化写时复制

原书通过图示展示了写时复制的内存布局 ：

```
创建 str:
str (RString) → ["Geometry is knowledge of the eternally existent."]

dup 后:
str (RString) ─┐
               ├─→ 共享字节数组: "Geometry is knowledge..."
str2 (RString)─┘

修改 str2 后:
str (RString) → ["GEOMETRY IS KNOWLEDGE OF THE ETERNALLY EXISTENT."]
str2 (RString) → ["Geometry is knowledge of the eternally existent."]
# 修改触发了复制，两个字符串现在有各自的字节数组
```

### 10.2.5 修改共享字符串更慢

这是写时复制的代价 ：

```
# 测量写时复制性能
str = "Geometry is knowledge of the eternally existent."
str2 = str.dup

# 第一次修改 str2——触发复制
str2.upcase!  # 较慢：需要分配新字节数组并复制数据

# 后续修改——不再共享，直接修改
str2.reverse!  # 较快：直接在原字节数组上操作
```

🎯 **我的洞见（本节最核心的认知）**：

> **写时复制是"时间换空间"的经典权衡——**
> 
> - **读操作**：快（共享，零复制）
>     
> - **首次写操作**：慢（触发复制：分配新数组 + 拷贝数据）
>     
> - **后续写操作**：快（已经独立，直接修改）
>     
> 
> 这意味着：**如果你的字符串主要是只读的（Ruby 程序中的绝大多数字符串都是如此），写时复制带来巨大的性能和内存优势。只有频繁修改的字符串才会承受复制开销。**

💡 **工业应用**：

```
# 好的模式：利用写时复制
template = "Hello, #{name}!"  # 字面量，共享
1000.times do |i|
  greeting = template.dup    # 共享底层数组，O(1)
  process(greeting)
end

# 坏的模式：频繁修改大字符串
html = "<html>"
10000.times do |i|
  html << "<div>#{i}</div>"  # 每次都修改 html，触发复制
end
# 应该用数组收集后 join：
parts = ["<html>"]
10000.times do |i|
  parts << "<div>#{i}</div>"
end
html = parts.join  # 只分配一次最终字符串
```

---

## 10.3 总结

本章我们从第一性原理梳理了 JRuby 的架构设计与核心优化：

1. **启动路径的根本差异**：MRI 是 C 二进制直接执行 Ruby；JRuby 是 shell 脚本调用 `java` 命令，在 JVM 上通过 `jruby.jar` 执行 Ruby 。
    
2. **解析编译流水线**：Ruby 源码 → `RubyLexer`（词法）→ `RubyParser`（Jay 生成，语法解析）→ AST → **IR（中间表示）**​ → JVM 字节码（通过 ASM 库在内存中生成）→ 机器码（HotSpot JIT） 。
    
3. **混合模式执行**：JRuby 先解释 IR（快速启动），热点方法 JIT 成 JVM 字节码，再由 HotSpot 的 C1/C2 编译器 JIT 成机器码——**双层 JIT**​ 。
    
4. **Java 类映射 Ruby 类型**：每个 Ruby 核心类型对应一个 Java 类（`RubyString`、`RubyArray`、`RubyHash` 等），所有 Ruby 对象实现 `IRubyObject` 接口 。这种设计让 JRuby 能直接利用 HotSpot 的优化。
    
5. **JIT 性能特征**：JRuby **慢启动、快长跑**。预热完成后，因 HotSpot C2 的深度优化，性能往往超过 MRI 。启用 `invokedynamic` 后可获得额外加速，但默认未启用（预热开销） 。
    
6. **字符串的写时复制**：
    
    - MRI 用 `RString` + `shared` 指针实现共享
        
    - JRuby 用 `RubyString` + `ByteList` 封装 `byte[]` 实现共享
        
    - **读操作快、首次写操作慢（触发复制）、后续写操作快**​
        
    - JRuby 字符串长度上限 2GB（Java `byte[]` 限制）
        
    
7. **JRuby 的代价**：不支持 C 扩展（这是最大短板） 。但可以通过 Java 互操作调用 Java 生态的库来弥补。
    

🔑 **贯穿全章的核心洞见**：

> **JRuby 的本质，是"让 Ruby 成为 JVM 的一等公民"。**
> 
> - **语义层**：Ruby 的语法、对象模型、元编程能力，被完整保留
>     
> - **实现层**：Ruby 对象 → Java 对象，Ruby 方法 → Java 方法，Ruby 字节码 → JVM 字节码
>     
> - **优化层**：利用 JVM 的 JIT、GC、多线程、invokedynamic
>     
> - **生态层**：无缝接入 Java 世界的数千个库
>     
> 
> 这种"语义保留、实现替换"的策略，让 Ruby 程序员获得了 JVM 的全部工程优势——成熟的 GC、真正的并行多线程（无 GIL）、企业级监控和运维工具——而无需改变编程模型。
> 
> 代价是放弃了 C 扩展生态，并接受"预热慢"的性能曲线。这是工程上经典的权衡：**用生态兼容性换取运行时性能和企业基础设施**。

⚠ **给后续章节的铺垫**：

- **第11章 Rubinius**：另一种 Ruby 实现——用 C++ VM + Ruby 实现核心库。与 JRuby 形成对比：JRuby 借用 JVM 的基础设施，Rubinius 自己构建 VM
    
- **第12章 MRI 的 GC**：JRuby 把 GC 责任交给了 JVM，而 MRI 自己实现 GC。对比两者能深入理解 GC 设计的权衡
    
- **现代 Ruby 的 JIT**：MRI 的 MJIT/RJIT 与 JRuby 的 JIT 思路相通——都是把 Ruby 字节码 JIT 成更低级的形式。但 MJIT 生成 C 代码交给 GCC/Clang 编译，RJIT 用 Rust 生成机器码，而 JRuby 生成 JVM 字节码交给 HotSpot——**三条路径，同一个目标**
    

📌 **工业实践建议**：

1. **长时间运行的服务端应用**：考虑 JRuby——预热后的吞吐量和稳定性往往优于 MRI
    
2. **需要 Java 库集成的场景**：JRuby 是天然选择（如调用 Apache POI、Lucene、Hadoop）
    
3. **依赖 C 扩展的项目**：坚持用 MRI——JRuby 不支持 C 扩展
    
4. **字符串处理优化**：利用写时复制——避免频繁修改大字符串，使用数组收集后 join
    
5. **监控 JIT**：用 `-J-XX:+PrintCompilation` 观察 JRuby 的 JIT 过程，识别未优化的热点
    
6. **预热策略**：JRuby 应用启动后，通过"热身请求"触发 JIT，再接入生产流量
    
7. **线程并行**：JRuby 无 GIL，可以真正并行——充分利用多核 CPU
    
