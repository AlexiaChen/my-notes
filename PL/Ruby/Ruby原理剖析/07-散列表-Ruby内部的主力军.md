
# 第7章 读书笔记：散列表——Ruby 内部的主力军

> 💡 这一章回答的核心问题是：**为什么 Ruby 内部几乎"一切皆散列表"？它到底怎么做到既快又省内存的？**
> 
> 从第一性原理看，散列表解决的是一个根本性的计算机科学问题：**如何在"无序的数据"和"O(1) 的访问"之间架起桥梁**。Ruby 选择散列表作为内部主力数据结构，不是偶然——方法表、常量表、实例变量、全局变量、Hash 对象本身，全都是散列表。理解散列表，就是理解 Ruby 性能的命门。

📌 **时代注记（非常重要）**：原书基于 Ruby 2.0，描述的是**桶+链表**的传统实现：11 个桶、密度超 5 扩容、rehash 时重建链表。但 **Vladimir Makarov 在 2015 年（Ruby 2.3+）彻底重写了 `st.c`，改为开放寻址（open addressing）+ entries 数组 + bins 数组的新实现**。作者 Pat Shaughnessy 在 2025 年新版本的_Chapter 7 重写_中已采用新实现。

**本章的读书笔记将采取"双视角"**：先用原书 Ruby 2.0 的视角讲清第一性原理，再指出现代 Ruby 的演进——这样你既能读懂原书，又能理解今天的 Ruby 源码。

---

## 7 散列表：Ruby 内部的主力军

原书作者开篇点明：**"Every time you write a method, Ruby creates an entry in a hash table"**。这不是修辞，是 C 层面的事实：

- **方法表**（`m_tbl`）：每个 RClass 里都有一张散列表，存 `方法名 → 方法条目`
    
- **常量表**（`const_tbl`）：每个 RClass 里都有一张散列表，存 `常量名 → 常量值`
    
- **实例变量**：对象的 `ivptr` 数组 + 类的 `iv_index_tbl` 散列表（映射变量名 → 数组索引）
    
- **全局变量**：`global_tbl` 散列表
    
- **`generic_iv_tbl`**：给 Symbol/nil 等立即值存实例变量的全局散列表
    
- **Hash 对象本身**：`RHash` 结构体内嵌 `st_table`
    

🎯 **我的洞见**：Ruby 选择散列表作为"万能内部数据结构"，背后是第一性原理的权衡——

- **数组**访问 O(1) 但查找 O(n)
    
- **链表**插入 O(1) 但查找 O(n)
    
- **树**查找 O(log n) 但实现复杂、缓存不友好
    
- **散列表**平均 O(1) 查找、O(1) 插入，且**实现相对简单**
    

Ruby 是高度动态的语言——方法可以在运行时添加、类可以在运行时修改、实例变量可以动态出现。这种"键集合在运行时不断变化"的场景，散列表是唯一合适的选择。**散列表让 Ruby 的动态性在性能上变得可行**。

---

## 7.1 Ruby 中的散列表

### 7.1.1 在散列表中保存值

原书 Ruby 2.0 的实现：

当执行 `my_hash[:key] = "value"` 时：

1. Ruby 调用内部哈希函数：`some_value = internal_hash_function(:key)`
    
2. 计算桶索引：`some_value % 11 = 2`（Ruby 1.8/1.9 初始 11 个桶）
    
3. 创建一个 `st_table_entry` 结构，存入 key 和 value
    
4. 把这个 entry **头插法**挂到桶 2 的链表上
    

**现代 Ruby（2.3+）的实现**：

Ruby 3.x 对每个 Hash 对象维护两个数组：

```
RHash
└── st_table
    ├── entries 数组：[{hash, key, value}, ...]  ← 按插入顺序
    └── bins 数组：[entry_index, ...]            ← 桶，存 entry 在 entries 数组中的索引
```

保存流程：

1. 计算键的哈希值 `h`
    
2. 用 `h` 的低几位作为 bins 数组的索引（因为 bins 大小是 2 的幂，`h & (bins.size-1)` 即可，比取模快得多）
    
3. 把 key/value 顺序追加到 entries 数组的末尾
    
4. 在 bins 数组的对应位置存入"entries 数组中的索引"
    

🎯 **我的洞见（本节最核心的认知）**：

> **现代 Ruby 的散列表用"entries 数组 + bins 数组"分离了"数据存储"和"索引查找"**。
> 
> - **entries 数组**：紧凑存储所有键值对，**按插入顺序排列**——这让 `Hash#each`、`Hash#keys` 等操作变成简单的数组遍历，无需追逐链表指针
>     
> - **bins 数组**：扮演"索引"角色，存 entries 数组中的索引，实现 O(1) 查找
>     
> 
> 这种设计的第一性原理是**缓存局部性（cache locality）**——现代 CPU 访问连续内存比访问散布的指针快得多。原书的链表实现，每个 entry 都是独立 malloc 的，指针跳跃严重；新实现把 entry 紧凑排列在数组中，**遍历速度提升约 40%**（在 Intel Haswell CPU 上测得）。

### 7.1.2 从散列表中检索值

原书 Ruby 2.0 的检索：

```
puts my_hash[:key]
```

1. 重新计算 `:key` 的哈希值：`some_value = internal_hash_function(:key)`
    
2. 计算桶索引：`some_value % 11 = 2`
    
3. 到桶 2 的链表中遍历，用 `eql?` 比较每个 entry 的 key
    
4. 匹配成功则返回 value
    

**现代 Ruby 的检索**：

1. 计算 `h = key.hash`
    
2. 计算 bin 索引：`bin_idx = h & (bins.size - 1)`
    
3. 从 bins[bin_idx] 取出 entry 索引 `e_idx`
    
4. 如果 `e_idx` 是空标记，返回 nil
    
5. 否则检查 `entries[e_idx]`：
    
    - 如果 `entries[e_idx].hash == h` 且 `key.eql?(entries[e_idx].key)`，返回 value
        
    - 否则发生**冲突**，用二次哈希值计算下一个 bin 位置（线性同余生成器，满足 Hull-Dobell 定理，保证能遍历所有 bin）
        
    
6. 重复直到找到匹配或遇到空 bin
    

💡 **工业应用**：理解检索过程对性能优化有直接意义——

```
# 慢：字符串作为键（每次都要计算 hash + 字符串比较）
hash["user_count"] = 42
hash["user_count"]  # 每次都要重新 hash 字符串

# 快：Symbol 作为键（Symbol 的 hash 值被缓存）
hash[:user_count] = 42
hash[:user_count]  # hash 值稳定，且 Symbol 比较是 O(1) 的指针比较
```

这就是为什么 Rails 等框架**强烈推荐使用 Symbol 作为 Hash 键**——Symbol 的哈希值在创建时计算一次并缓存，且 `eql?` 比较是指针比较，远比字符串比较快。

---

## 7.2 散列表如何扩展以容纳更多的值

### 7.2.1 散列冲突

**原书 Ruby 2.0 的冲突处理**：

- 多个键映射到同一个桶 → 冲突
    
- Ruby 1.8/1.9 用**链表**处理：每个 `st_table_entry` 有一个 `next` 指针，指向同桶的下一个 entry
    
- 随着元素增多，链表变长，查找退化到 O(n)
    

Ruby 监控**桶密度（density）**——平均每桶 entry 数。一旦密度超过 **5**（C 源码中的常量），就触发扩容。

**现代 Ruby 的冲突处理**：

- 用**开放寻址（open addressing）**而非链表
    
- 冲突时不链到另一个 entry，而是**探测下一个 bin**
    
- 探测序列由二次哈希值决定：`next_bin = f(collision_bin_index, original_hash)`
    
- 这个函数保证满足 Hull-Dobell 定理，即经过若干次迭代后能遍历所有 bin，最终找到目标或空 bin
    
- 删除的 entry 用特殊标记（DELETED）填充，而非真正清空，以避免破坏探测链
    

🎯 **我的洞见**：从链表到开放寻址的演进，本质是一场**"指针追逐 vs 缓存友好"的权衡**。

- **链表法**：每个 entry 独立分配，指针跳跃，CPU 缓存命中率低
    
- **开放寻址**：entry 紧凑排列在数组中，冲突时探测相邻 bin，CPU 预取器能有效工作
    

Makarov 的重写让 Ruby 散列表在 Intel Haswell CPU 上**平均提速 40%**——这个数字背后，是计算机体系结构（缓存层级）对算法选择的深刻影响。

### 7.2.2 重新散列条目（Rehashing）

**原书 Ruby 2.0 的 rehash**：

当密度 > 5 时：

1. 分配一个更大的 bins 数组（原大小 × 2 左右，Ruby 1.8 用质数序列：11 → 19 → 37 → 67...）
    
2. 遍历所有 `st_table_entry`
    
3. 对每个 entry 重新计算 `hash % new_num_bins`
    
4. 把 entry 插入到新 bins 数组对应的链表头部
    
5. 释放旧的 bins 数组
    

C 源码片段（Ruby 1.8.7）：

```
static void rehash(table) register st_table *table; {
  // ... 分配 new_bins 数组
  for(i = 0; i < old_num_bins; i++) {
    ptr = table->bins[i];
    while (ptr != 0) {
      next = ptr->next;
      hash_val = ptr->hash % new_num_bins;
      ptr->next = new_bins[hash_val];
      new_bins[hash_val] = ptr;
      ptr = next;
    }
  }
  // ... 释放旧 bins
}
```

**现代 Ruby 的扩容**：

作者在 Ruby 3.4.1 上重跑了 Experiment 7-2，观察到明显的扩容尖峰：

```
插入第 1 个元素：~0.6 ms
插入第 2-8 个元素：~0.6 ms
插入第 9 个元素：~2.0 ms  ← 扩容尖峰！
插入第 10-16 个元素：~0.6 ms
插入第 17 个元素：~3.1 ms  ← 又一次扩容尖峰！
```

扩容触发条件：**bins 数组长度至少是 entries 数组长度的 2 倍**。这保持了负载因子（load factor）约 0.5 的健康水平。

扩容过程：

1. 创建新的 entries 数组和 bins 数组（大小翻倍）
    
2. 把旧 entries 数组中的有效 entry 复制到新数组
    
3. 重新计算每个 entry 的 bin 索引
    
4. 在某些情况下尝试"compacting"——移除已删除的 entry，复用数组
    

⚠️ **工业现实与性能警示**：

```
# 已知大小的 Hash，预分配可减少扩容次数
large_hash = {}
(1..10000).each { |i| large_hash[i] = i * i }
# 上面的代码会触发约 log2(10000) ≈ 14 次扩容
# 每次扩容都要复制所有现有 entry，累积开销显著

# 优化：如果事先知道大小，可以考虑分批初始化
# 或使用 Hash.new 配合默认值块
```

💡 **现代 Ruby 的 packed hash 优化**：6 个元素以内，Ruby 2.0+ 完全不用散列表——直接存数组，线性查找。这就是为什么插入第 7 个元素会有尖峰——此时要从 packed array 转换为真正的散列表。

---

## 7.3 Ruby 如何实现散列函数

散列函数是散列表的灵魂。一个优秀的散列函数必须满足：

1. **确定性**：同一个对象每次调用 `hash` 返回相同值
    
2. **均匀分布**：不同对象应尽量映射到不同的哈希值
    
3. **雪崩效应**：输入微小变化应导致输出巨大变化
    

**Symbol 的哈希函数**：

```
:user.object_id  # Symbol 的哈希值基于 object_id，稳定且分布均匀
```

**String 的哈希函数**：

Ruby 使用**带随机种子的 FNV-1a 变种**算法。种子在 Ruby 进程启动时随机生成（通过 `hashseed`），防止哈希洪水攻击（Hash Flooding DoS）。

**Integer 的哈希函数**：

```
42.hash  # 对 Fixnum，hash 值就是其值本身（或经过简单变换）
```

🎯 **我的洞见（本节最震撼的实验）**：

原书 Experiment 7-3 展示了哈希函数的极端重要性：

```
class KeyObject
  def hash
    1  # 故意返回常数！所有对象都映射到同一个 bin
  end
  
  def eql?(other)
    super
  end
end
```

实验结果：

```
哈希大小 1-10：      ~1.5 ms
哈希大小 100：       显著增加
哈希大小 1000：      ~1600 ms（1.6 秒！）
哈希大小 10000：     数十秒甚至分钟级
```

**所有元素都挤在同一个 bin 里，查找退化成 O(n) 线性扫描**。这个实验完美证明了——**散列表的性能完全依赖于哈希函数的质量**。

🔑 **工业应用**：

```
# 自定义对象作为 Hash 键时，必须正确实现 hash 和 eql?
class User
  attr_reader :id, :email
  
  def hash
    # 必须结合所有参与 eql? 比较的字段
    [id, email].hash
  end
  
  def eql?(other)
    other.is_a?(User) && id == other.id && email == other.email
  end
  
  # 可选：实现 == 委托给 eql?
  def ==(other)
    eql?(other)
  end
end

users = {}
users[User.new(1, "a@b.com")] = "Alice"
users[User.new(1, "a@b.com")]  # => "Alice"  ✅ 正确查找
```

⚠️ **致命陷阱**：

```
# 如果键对象在放入 Hash 后被修改，且不调用 rehash，会导致查找失败
a = ["a", "b"]
c = ["c", "d"]
h = { a => 100, c => 300 }
h[a]  # => 100

a[0] = "z"  # 修改了键！
h[a]  # => nil  ❌ 查找失败！

h.rehash    # 重建哈希表
h[a]  # => 100 ✅
```

`Hash#rehash` 的 C 实现会遍历所有 entry，用当前的 hash 值重新计算 bin 索引。**这就是为什么键对象应该是不可变的（immutable）**——Rails 的 `ActiveModel` 等框架在把对象作为键使用时，都会先 dup + freeze。

### 7.3.1 Ruby 2.0 中的散列优化

这是本章最实用的工程优化：

**Packed Hash（紧凑散列）**：

- 对于 **6 个或更少元素**的 Hash，Ruby 2.0+ **完全不使用 `st_table`**
    
- 直接在 RHash 结构体内嵌一个数组，按插入顺序存储 key/value 对
    
- **查找时线性扫描数组**，调用 `eql?` 比较
    
- **根本不调用 `hash` 函数**
    

为什么这样做更快？

```
传统散列表查找：
1. 调用 key.hash（计算哈希值）
2. 计算 bin 索引
3. 访问 bins 数组
4. 访问 entries 数组
5. 调用 eql? 比较
→ 5 步，且涉及多次间接内存访问

Packed hash 查找（≤6 元素）：
1. 线性扫描数组（最多 6 次）
2. 每次调用 eql? 比较
→ 对于小 Hash，直接比较比"计算 hash + 间接访问"更快
```

**实测数据**：

- Rails 应用中，**高达 40% 的 Hash 永远不会增长到超过 1 个元素**
    
- 典型 Rails 请求会分配大量小 Hash（路由参数、视图局部变量、SQL 结果等）
    
- Packed hash 优化在典型 Rails 应用上带来 **2.5%+ 的整体性能提升**
    

🎯 **我的洞见**：这个优化揭示了软件工程中一个普适的第一性原理——

> **为常见情况优化，比为最坏情况优化更重要。**
> 
> 散列表的理论复杂度是 O(1)，但对于小尺寸 Hash，O(1) 的常数因子（hash 计算 + 多次间接内存访问）远大于 O(n) 线性扫描的常数因子（当 n ≤ 6 时）。Ruby 核心团队通过**实测数据**（Rails 应用中的 Hash 大小分布）发现了这个优化机会，而不是靠理论推导。

⚠️ **现代演进**：Ruby 3.x 进一步优化了 packed hash：

- 8 个元素以内使用 packed hash（从 6 扩大到 8）
    
- 使用 8/16/32/64 位索引，根据 Hash 大小动态调整，节省内存
    
- 极小的 Hash（1-2 个元素）甚至**完全不分配 bins 数组**，纯线性查找
    

💡 **工业应用**：

```
# 知道 Hash 会很小时，放心用——Ruby 已经为你优化了
config = { timeout: 30, retries: 3 }
metadata = { version: "1.0", author: "Alice" }

# 不要过早优化为数组——Hash 在小尺寸下已经极快
# 反模式：为了"性能"而用 Struct/数组替代小 Hash
# 实际上 packed hash 的性能与数组相当，且更灵活
```

---

## 7.4 总结

本章我们从第一性原理梳理了 Ruby 散列表的实现与演进：

1. **Ruby 内部大量使用散列表**：方法表、常量表、实例变量索引、全局变量、Hash 对象本身，全都是散列表 。这是 Ruby 动态性的性能基石。
    
2. **原书 Ruby 2.0 的实现**：
    
    - 桶+链表结构，初始 11 个桶
        
    - 密度 > 5 时扩容并 rehash
        
    - 6 个元素以内的 Hash 使用 packed array，线性查找，不调用 `hash` 函数
        
    
3. **现代 Ruby（2.3+）的实现**：
    
    - **entries 数组 + bins 数组**分离数据存储与索引
        
    - **开放寻址**而非链表，利用缓存局部性
        
    - bins 大小为 2 的幂，用位运算代替取模
        
    - 负载因子约 0.5（bins 数组 ≥ 2 × entries 数组）
        
    - 删除标记而非真删除，维持探测链完整性
        
    - Intel Haswell 上**平均提速 40%**
        
    
4. **散列函数的质量决定性能**：糟糕的 `hash` 方法（如总返回常数）会让查找退化到 O(n)。Symbol 的 hash 值稳定且分布均匀，是 Hash 键的最佳选择。
    
5. **Packed hash 优化**：≤6 元素的 Hash 不使用 `st_table`，直接数组存储+线性查找。Rails 应用中 40% 的 Hash 不超过 1 个元素，此优化带来 2.5%+ 整体性能提升。
    
6. **扩容的代价**：插入第 7、17 个元素时会出现明显的性能尖峰（packed → 散列表转换、散列表扩容）。
    

🔑 **贯穿全章的核心洞见**：

> **Ruby 散列表的演进史，是一部"缓存局部性 vs 算法复杂度"的博弈史。**
> 
> - **算法层面**：开放寻址 + 2 的幂 bins + 位运算映射，把 O(1) 的常数因子压到最低
>     
> - **数据布局层面**：entries 数组紧凑排列，让遍历从"指针追逐"变为"顺序扫描"
>     
> - **自适应层面**：小 Hash 用 packed array 完全绕过散列计算，根据实测数据分布做优化
>     
> - **安全层面**：随机 hash seed 防止哈希洪水攻击
>     
> 
> 这四个维度的优化，让 Ruby 的散列表既是计算机科学经典理论的实践，又是现代 CPU 体系结构的深度适配。

⚠️ **给后续章节的铺垫**：

- **第8章"垃圾收集"**：散列表的 entries 数组、bins 数组、st_table_entry 结构，全都是 GC 管理的对象。扩容时的内存分配、packed → 散列表转换时的旧数组释放，都与 GC 密切相关
    
- **第9章及以后**：Ruby 3.x 的 **Object Shapes**​ 技术进一步优化了实例变量的存储——它本质上用"形状"概念替代了部分散列表查找，这与本章的散列表优化思路一脉相承
    
- **性能工程的普适原则**：Ruby 散列表的优化策略（为常见情况特化、利用缓存局部性、自适应数据结构）是所有高性能系统的通用原则
    

📌 **工业实践建议**：

1. **用 Symbol 作为 Hash 键**——hash 值稳定、比较 O(1)
    
2. **小 Hash（≤6 元素）放心用**——Ruby 已经通过 packed hash 优化到极致
    
3. **自定义对象作为键时，正确实现 `hash` 和 `eql?`**——结合所有参与相等的字段
    
4. **键对象应该是不可变的**——放入 Hash 后修改键会导致查找失败，需要调用 `rehash`
    
5. **警惕哈希洪水攻击**——处理用户输入作为 Hash 键时（如 JSON 解析），Ruby 的随机 hash seed 已经防护，但极端情况下仍需留意
    
6. **预分配大 Hash**——如果知道 Hash 会很大，考虑分批初始化以减少扩容次数
    
7. **理解 `rehash` 的代价**——它遍历所有 entry 重建索引，是 O(n) 操作
    
