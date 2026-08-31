
## 开篇：第 15 章的主线——"Python 的'多线程'为什么不是真正的多线程"

前几章我们揭示了 PVM 的运行机制：

- **函数/类/控制流**​ 是字节码层面的抽象 [第 10-12 章]
- **import**​ 是一套可扩展的查找-加载协议 [第 14 章]
- **PVM 启动**​ 时建立了"运行时-解释器-线程"三级状态 [第 13 章]

但有一个根本问题没回答：**当你 `import threading; Thread(target=...).start()` 时，CPython 是怎么让多个线程"看起来"并发执行的？为什么 CPU 密集型多线程程序在 Python 中跑不满多核？**​ 第 15 章要揭开的，是 GIL 这一"阿喀琉斯之踵"的全貌。

**本章与书里的最大时差**：

- **书里（Python 2.5）**：GIL 是基于"检查间隔"（check interval，默认 100 个字节码指令）触发切换；线程调度是 C 层硬逻辑
- **现代（3.14）**：
    - 3.2 起"检查间隔"被**绝对时长**取代，`sys.setswitchinterval()` 默认 **5 毫秒**
    - CPython 在 GIL 下**同一时刻只有一个线程能执行 Python 字节码**
    - 3.12 通过 **PEP 684**​ 引入每解释器独立 GIL——子解释器可以真正并行
    - 3.13 通过 **PEP 703**​ 引入实验性 free-threaded build（编译时 `--disable-gil`），3.14 转为正式支持

**本章双轨阅读法**：

- **轨道一（机制，跨版本稳定）**：Python 线程 = OS 线程 + GIL 协调；线程切换发生在字节码指令之间；I/O 操作时释放 GIL
- **轨道二（关键演化）**：
    - 线程切换从"字节码计数"改为"绝对时长"（3.2+）
    - `thread` 模块更名为 `_thread`（3.7+ 内部模块），`threading` 始终可用
    - 每解释器独立 GIL，子解释器真正并行（3.12+ PEP 684）
    - 编译时禁用 GIL 的 free-threaded build（3.13 实验性，3.14 正式支持，PEP 703）
    - 在 free-threaded build 中，内置类型（dict/list/set）内部加锁，但**不推荐**依赖其内部锁做同步

---

## 15.1 GIL 与线程调度

**书里（Python 2.5）的视角**：

- GIL 是保护 Python 内部状态的全局互斥锁
- 切换基于"检查间隔"——每执行 100 条字节码指令，当前线程释放 GIL 并触发切换
- 线程调度由操作系统负责，CPython 只是周期性地让出 GIL

**现代（3.14）的视角**：

**关键演化**：3.2 起，"检查间隔"的概念被彻底废弃。

> _"The notion of a 'check interval' to allow thread switches has been abandoned and replaced by an absolute duration expressed in seconds. This parameter is tunable through `sys.setswitchinterval()`. It currently defaults to 5 milliseconds."_

也就是说：

- **旧机制（2.x）**：每 N 条字节码指令切换一次（N 默认 100）
- **新机制（3.2+）**：每 N 秒切换一次（N 默认 0.005 秒 = 5 毫秒）

**GIL 的本质与线程调度的真相**（依据官方文档）：

> _"The system's profile function is called similarly to the system's trace function... Please note that the actual value can be higher, especially if long-running internal functions or methods are used. Also, **which thread becomes scheduled at the end of the interval is the operating system's decision. The interpreter doesn't have its own scheduler.**"_

**这句话包含三个核心事实**：

1. **CPython 解释器本身没有调度器**——线程切换的最终决策权在操作系统
2. **切换间隔只是"理想值"**——实际切换间隔可能更长（特别是长运行的 C 函数）
3. **GIL 是 Python 层面的互斥锁**，不是 OS 层面的调度器

**GIL 下的"原子性"保证**（依据官方 FAQ）：

> _"In general, Python offers to switch among threads only between bytecode instructions; how frequently it switches can be set via `sys.setswitchinterval()`. Each bytecode instruction and therefore all the C implementation code reached from each instruction is therefore atomic from the point of view of a Python program."_

这意味着：**单条字节码指令是原子的**。对于"看起来像原子操作"的内置数据类型操作（如 `L.append(x)`、`dict[k]=v`、`counter += 1` 的读取-修改-写入部分），实际上是原子的——但要注意：

- `counter += 1` 是**多条字节码指令**（`LOAD` → `BINARY_ADD` → `STORE`），**不是原子的**
- `L.append(x)` 是**单条字节码指令**（`CALL_METHOD`），**是原子的**

**第一性原理**：

- **GIL 不是为了"支持多线程"而存在，而是为了"让单线程下的内存管理不用加锁"**。引用计数的递增/递减必须是原子的，否则会出现竞态条件。GIL 是最简单的实现方案——整个解释器只有一个线程能执行字节码，引用计数自然就是线程安全的。
- **"线程切换发生在字节码之间"是 GIL 调度模型的精髓**。CPython 不会在一条字节码指令执行到一半时切换线程，这保证了单条字节码操作的原子性。但 Python 层面的复合操作（如 `+=`）由多条字节码组成，因此**不是原子的**。
- **`setswitchinterval` 的时长是"理想值"而非"保证值"**。CPython 在每个字节码指令执行后检查是否超过了切换间隔，如果超过则释放 GIL 并尝试切换。但**具体哪个线程被 OS 选中来获得 GIL，CPython 无法控制**。

**洞见——为什么 `sys.setswitchinterval()` 不能用来"修复"竞态条件？**

```
import sys, threading

counter = 0
def increment():
    global counter
    for _ in range(1000000):
        counter += 1

# 错误思路：把切换间隔调到极大，让一个线程"独占" GIL
sys.setswitchinterval(1000.0)  # 1000秒——看起来 counter 不会被打断

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()
print(counter)  # 仍然可能小于 2000000！
```

为什么？因为：

1. `counter += 1` 编译成 3 条字节码，**不是原子的**
2. 即使在极长的切换间隔下，OS 可能在任何时候抢占线程（如 I/O 发生、硬件中断）
3. 更重要的是，**依赖 GIL 的"伪原子性"是不安全的**——正确的做法是用 `threading.Lock`

**工业实际**：

```
import sys, threading

# 1. 查看和设置切换间隔
print(sys.getswitchinterval())  # 0.005（默认5毫秒）
sys.setswitchinterval(0.001)    # 切换到1毫秒——更频繁的线程切换
# 注意：过小的值会导致过多上下文切换开销
# 过大的值会导致线程饥饿（某个线程长时间占用 GIL）

# 2. 验证单字节码的原子性
import dis
def foo(L, x):
    L.append(x)
dis.dis(foo)
# 会看到 CALL_METHOD 是一条字节码指令——append 是原子的

def bar(c):
    c += 1
dis.dis(bar)
# 会看到 LOAD_FAST + BINARY_ADD + STORE_FAST 三条指令——不是原子的

# 3. 正确的并发计数器
counter = 0
lock = threading.Lock()
def safe_increment():
    global counter
    for _ in range(1000000):
        with lock:
            counter += 1
```

---

## 15.2 初见 Python Thread

**书里（Python 2.5）**：Python 通过 `thread` 模块（底层）和 `threading` 模块（高层）提供线程支持。在 2.x 中，`threading` 模块包含 camelCase 命名的方法（如 `startNewThread`）。

**现代（3.14）的视角**：

**关键演化**：

- `thread` 模块变为内部模块 `_thread`
- `threading` 模块从 3.7 起**不再是可选模块，始终可用**
- 2.x 风格的 camelCase 命名（如 `currentThread`、`activeCount`）在 3.10 中已弃用

**CPython 实现细节**（官方文档明确指出）：

> _"在 CPython 中，由于存在全局解释器锁，同一时刻只有一个线程可以执行 Python 代码（虽然某些性能导向的库可能会去除此限制）。如果你想让你的应用更好地利用多核心计算机的计算资源，推荐你使用 `multiprocessing` 或 `concurrent.futures.ProcessPoolExecutor`。但是，如果你想要同时运行多个 I/O 密集型任务，则多线程仍然是一个合适的模型。"_

这段话包含三个重要的工程判断：

1. **CPU 密集型任务**：多线程无法利用多核，应该用 `multiprocessing`
2. **I/O 密集型任务**：多线程仍然是合适的模型（因为 I/O 操作会释放 GIL）
3. **某些性能库会释放 GIL**：如 NumPy 的数值计算、OpenCV 的图像处理等——在这些库的 C 代码执行期间，GIL 被释放

**threading 模块的核心组件**：

- `threading.Thread`：线程类，通过 `target` 参数或继承 `run()` 方法创建
- `threading.Lock` / `RLock`：互斥锁 / 可重入锁
- `threading.Event`：事件同步原语
- `threading.Condition`：条件变量
- `threading.Semaphore` / `BoundedSemaphore`：信号量
- `threading.Timer`：定时器线程
- `threading.local`：线程局部数据
- `threading.excepthook`：处理线程中未捕获的异常（3.8+）

**第一性原理**：

- **Python 线程是 OS 线程的薄封装**。每个 `threading.Thread` 对象背后都有一个真实的 OS 线程（通过 `_thread` 模块创建）。Python 线程的"并发"是 OS 线程的并发，但受限于 GIL，同一时刻只有一个线程能执行 Python 字节码。
- **`threading` 模块的全部价值在于"同步原语"**。如果只有线程创建而没有锁、事件、条件变量等同步机制，多线程程序会因竞态条件而无法正确工作。`threading` 模块提供的高层接口让编写正确的并发程序成为可能。

**洞见——为什么 I/O 密集型任务适合多线程，CPU 密集型不适合？**

```
CPU 密集型（如数值计算）：
  Thread A: 获取 GIL → 执行字节码（不释放 GIL）→ 5ms 后释放 GIL
  Thread B: 等待 GIL → 获取 GIL → 执行字节码 → 5ms 后释放 GIL
  结果：两个线程交替执行，但同一时刻只有一个在运行——多核未被利用

I/O 密集型（如网络请求）：
  Thread A: 获取 GIL → 发起网络请求（释放 GIL）→ 等待 I/O 响应
  Thread B: 立即获取 GIL → 发起另一个网络请求（释放 GIL）→ 等待
  结果：两个线程真正并行等待 I/O——多核的 I/O 带宽被充分利用
```

这就是为什么：

- **Web 服务器**（I/O 密集）：多线程是合适的（如 Flask/Django 开发服务器）
- **数据处理**（CPU 密集）：应该用 `multiprocessing` 或多进程
- **现代趋势**：I/O 并发更倾向于用 `asyncio`（协程），避免线程开销

**工业实际**：

```
import threading, time

# 1. 创建线程的两种方式
# 方式A：使用 target 参数
def worker(name):
    print(f"Thread {name} starting")
    time.sleep(1)  # 模拟 I/O 操作（释放 GIL）
    print(f"Thread {name} done")

t = threading.Thread(target=worker, args=("A",))
t.start()
t.join()

# 方式B：继承 Thread 类
class MyThread(threading.Thread):
    def run(self):
        print(f"{self.name} running")
        time.sleep(1)
        print(f"{self.name} done")

t = MyThread(name="B")
t.start()
t.join()

# 2. CPU 密集型 vs I/O 密集型
import multiprocessing

def cpu_bound(n):
    """CPU 密集型：多线程无效，应用多进程"""
    return sum(i * i for i in range(n))

def io_bound(url):
    """I/O 密集型：多线程有效"""
    import urllib.request
    with urllib.request.urlopen(url) as response:
        return response.read()

# CPU 密集型：使用 ProcessPoolExecutor
with concurrent.futures.ProcessPoolExecutor() as executor:
    results = list(executor.map(cpu_bound, [1000000] * 4))

# I/O 密集型：使用 ThreadPoolExecutor
with concurrent.futures.ThreadPoolExecutor() as executor:
    results = list(executor.map(io_bound, urls))

# 3. 线程异常处理（3.8+ threading.excepthook）
def handle_thread_exception(args):
    print(f"Thread {args.thread.name} raised {args.exc_type.__name__}: {args.exc_value}")

threading.excepthook = handle_thread_exception
```

---

## 15.3 Python 线程的创建

**书里（Python 2.5）**：线程通过 `thread.start_new_thread(func, args)` 创建，或者通过 `threading.Thread` 高级封装。

**现代（3.14）的视角**：

**低层创建**：`_thread.start_new_thread(func, args, kwargs)`

```
import _thread

def worker(n):
    print(f"Worker {n} running in thread {_thread.get_ident()}")

# 创建线程（不推荐直接使用，缺乏同步机制）
_thread.start_new_thread(worker, (1,))
```

**高层创建**：`threading.Thread`

```
import threading

# 创建线程的标准方式
t = threading.Thread(
    target=func,        # 线程执行的函数
    args=(arg1, arg2), # 位置参数
    kwargs={'k': 'v'}, # 关键字参数
    name="MyThread",   # 线程名（可选）
    daemon=False       # 是否守护线程
)
t.start()  # 启动线程
t.join()   # 等待线程结束
```

**线程创建的内部流程**：

1. `Thread.start()` 调用 `_thread.start_new_thread()` 创建 OS 线程
2. OS 线程启动后，调用 `Thread._bootstrap()` 进行初始化
3. `_bootstrap()` 调用 `Thread.run()` 执行用户代码
4. `Thread.run()` 默认调用 `target(*args, **kwargs)`
5. 线程结束时，调用 `Thread._bootstrap()` 的清理逻辑

**守护线程 vs 非守护线程**：

- **非守护线程**：主线程必须等待所有非守护线程结束后才能退出
- **守护线程**：主线程退出时，守护线程被强制终止（不执行清理）
- **典型用途**：心跳检测、后台日志收集等"可有可无"的后台任务

**Python 3.13+ 的重要演化——Free-threaded build**：

> _"Starting with the 3.13 release, CPython has support for a build of Python called **free threading**​ where the global interpreter lock (GIL) is disabled. Free-threaded execution allows for full utilization of the available processing power by running threads in parallel on available CPU cores."_

在 free-threaded build 中：

- 线程创建的方式不变（`threading.Thread` 仍是标准接口）
- 但创建的线程可以**真正并行**执行 Python 字节码
- 单线程性能会有 10-15% 的下降（因为引用计数等操作需要线程安全机制）
- 许多 C 扩展尚未适配，导入时会自动重新启用 GIL

**第一性原理**：

- **线程创建 = OS 线程创建 + Python 层封装**。Python 线程不是"绿色线程"或"协程"，而是真实的 OS 线程。GIL 只是在 Python 字节码层面限制了并行度，但 OS 线程本身是真实并行的。
- **`Thread.start()` 是非阻塞的**——它立即返回，新线程异步执行。这就是为什么需要 `join()` 来等待线程完成。
- **守护线程的设计哲学**：守护线程是"为其他线程服务的"后台任务，不应阻止进程退出。主线程退出时，所有守护线程被立即终止——这意味着守护线程中的 `finally` 块可能不会执行，资源可能不会被正确释放。

**洞见——为什么在 CPU 密集型场景中，多线程可能比单线程还慢？**

```
import threading, time

def cpu_work():
    total = 0
    for i in range(10_000_000):
        total += i
    return total

# 单线程
t0 = time.time()
cpu_work()
cpu_work()
print(f"Sequential: {time.time() - t0:.2f}s")

# 多线程
t0 = time.time()
t1 = threading.Thread(target=cpu_work)
t2 = threading.Thread(target=cpu_work)
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Threaded: {time.time() - t0:.2f}s")
# 多线程版本可能更慢！因为 GIL 争用 + 上下文切换开销
```

原因：

1. **GIL 争用**：两个线程频繁争夺 GIL，导致大量上下文切换
2. **切换开销**：每次切换都需要保存/恢复线程状态
3. **CPU 缓存失效**：线程切换导致 CPU 缓存频繁失效

**正确的做法**：CPU 密集型用 `multiprocessing`：

```
import multiprocessing

if __name__ == '__main__':
    t0 = time.time()
    with multiprocessing.Pool(2) as pool:
        results = pool.map(lambda _: cpu_work(), [None, None])
    print(f"Multiprocess: {time.time() - t0:.2f}s")
    # 真正并行，充分利用多核
```

**工业实际**：

```
import threading
import concurrent.futures
import urllib.request

# 1. 线程池处理 I/O 密集型任务（推荐方式）
urls = [
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
]

def fetch_url(url):
    with urllib.request.urlopen(url) as response:
        return url, len(response.read())

# 使用 ThreadPoolExecutor（高层 API）
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(fetch_url, url) for url in urls]
    for future in concurrent.futures.as_completed(futures):
        url, size = future.result()
        print(f"{url}: {size} bytes")

# 2. 守护线程的典型用例：后台心跳
def heartbeat():
    while True:
        print("Heartbeat...")
        time.sleep(5)

hb = threading.Thread(target=heartbeat, daemon=True)
hb.start()  # 主线程退出时，心跳线程自动终止

# 3. 线程局部数据（避免共享状态）
thread_local = threading.local()

def worker():
    thread_local.count = 0
    for _ in range(1000):
        thread_local.count += 1
    print(f"Thread {threading.current_thread().name}: {thread_local.count}")

threads = [threading.Thread(target=worker) for _ in range(3)]
for t in threads: t.start()
for t in threads: t.join()
# 每个线程有独立的 count，互不干扰

# 4. 3.13+ Free-threaded build 的线程创建（同样的 API）
import sys
if sys._is_gil_enabled():
    print("GIL enabled - threads won't truly parallelize CPU work")
else:
    print("Free-threaded - threads CAN truly parallelize CPU work")
# 线程创建代码完全相同，但行为不同
```

---

## 15.4 Python 线程的调度

**书里（Python 2.5）**：基于"检查间隔"的协作式调度——每执行 100 条字节码指令，当前线程主动释放 GIL。

**现代（3.14）的视角**：

**调度模型的核心事实**：

> _"Please note that the actual value can be higher, especially if long-running internal functions or methods are used. Also, which thread becomes scheduled at the end of the interval is the operating system's decision. **The interpreter doesn't have its own scheduler.**"_

**调度的三个层次**：

**1. Python 层面的"理想切换间隔"**：

- `sys.setswitchinterval(interval)` 设置理想的切换间隔（默认 0.005 秒）
- CPython 在每个字节码指令执行后检查是否超时
- 如果超时，当前线程释放 GIL

**2. OS 层面的线程调度**：

- 哪个线程获得 GIL 由 OS 决定
- CPython 无法控制，也无法预测
- OS 调度策略（如 CFS）影响线程的实际执行顺序

**3. GIL 的竞争**：

- 释放 GIL 后，所有等待 GIL 的线程参与竞争
- 在 3.2+ 的实现中，刚释放 GIL 的线程处于劣势（让其他线程优先获取）
- 这避免了"线程饿死"问题

**I/O 操作时的 GIL 释放**：

```
# 在进行 I/O 操作时，Python 会自动释放 GIL
# 例如：
import time
time.sleep(1)  # 释放 GIL，其他线程可以运行
# 网络 I/O、文件 I/O、subprocess 等都会释放 GIL
```

**长时间运行的 C 扩展**：

- 如果 C 扩展在执行期间释放 GIL，其他线程可以并行运行
- NumPy、SciPy 等科学计算库在其核心计算中释放 GIL
- 这就是为什么 Python 在数据科学中表现良好——计算在 C 层并行，Python 层只是粘合剂

**第一性原理**：

- **Python 线程调度 = "Python 层切换间隔" + "OS 层线程调度" + "GIL 竞争"三层叠加**。Python 层只负责"何时释放 GIL"，OS 层负责"谁获得 GIL"，GIL 本身负责"保证单时刻单线程执行 Python 字节码"。
- **`setswitchinterval` 设置的是"理想值"**。实际切换间隔可能更长——特别是在执行长运行的 C 函数时。这意味着：
    - 设置过小的间隔：上下文切换开销大，整体性能下降
    - 设置过大的间隔：线程饥饿，响应性差
    - 默认值 5ms 是经验最优值

**洞见——为什么 `sys.setswitchinterval()` 调优通常是徒劳的？**

```
import sys, threading, time

def cpu_task():
    total = 0
    for i in range(5_000_000):
        total += i

# 测试不同切换间隔的影响
for interval in [0.001, 0.005, 0.01, 0.1]:
    sys.setswitchinterval(interval)
    
    t0 = time.time()
    threads = [threading.Thread(target=cpu_task) for _ in range(2)]
    for t in threads: t.start()
    for t in threads: t.join()
    elapsed = time.time() - t0
    
    print(f"Interval {interval:.3f}s: {elapsed:.2f}s")
    # 你会看到：间隔越小，开销越大；间隔越大，线程饥饿越严重
```

**更重要的洞见**：在 CPU 密集型场景下，无论怎么调 `setswitchinterval`，总执行时间都差不多——因为 GIL 确保了串行执行。调优 `setswitchinterval` 只对**特定类型的 I/O 密集型任务**有意义（如需要公平调度的多个网络请求）。

**工业实际**：

```
import sys, threading, time

# 1. 观测线程切换行为
def monitor_switch():
    current_thread = threading.current_thread()
    last_switch = time.time()
    count = 0
    while count < 10:
        now = time.time()
        if now - last_switch > 0.001:  # 检测到可能的切换
            print(f"[{now:.3f}] Thread {current_thread.name} running")
            last_switch = now
            count += 1
        # 执行一些计算
        sum(range(10000))

# 2. 调试死锁：sys._current_frames()
import sys
def dump_thread_stacks():
    """打印所有线程的当前栈帧——调试死锁利器"""
    for thread_id, frame in sys._current_frames().items():
        print(f"\nThread {thread_id}:")
        traceback.print_stack(frame)

# 3. I/O 密集型任务中的 GIL 行为
import urllib.request

def fetch_and_process(url):
    # 网络 I/O 时释放 GIL
    with urllib.request.urlopen(url) as response:
        data = response.read()
    # 数据处理时获取 GIL（CPU 工作）
    return process_data(data)

# 4. 使用 threading.settrace 跟踪所有线程
import sys
def global_trace(func):
    """为所有新创建的线程设置追踪函数"""
    threading.settrace(func)

def my_tracer(frame, event, arg):
    if event == 'call':
        print(f"Thread {threading.current_thread().name}: calling {frame.f_code.co_name}")
    return my_tracer

global_trace(my_tracer)
```

---

## 15.5 Python 子线程的销毁

**书里（Python 2.5）**：线程执行完 `run()` 方法后自然退出，或者通过 `thread.exit()` 强制退出。

**现代（3.14）的视角**：

**线程的自然销毁**：

1. `Thread.run()` 执行完毕（正常返回或抛出异常）
2. `_bootstrap()` 执行清理工作
3. 线程状态从 `sys.path` 中移除
4. OS 线程被回收

**关键事实——Python 线程不能被强制杀死**：

```
# Python 没有提供强制终止线程的 API！
# 这是故意的设计——强制杀线程会导致：
# - 资源泄漏（文件、锁、网络连接未释放）
# - 数据结构不一致（如 dict 在扩容时被打断）
# - 死锁（线程持有锁时被杀死）

# 正确的线程终止方式：使用"协作式取消"
class CancellableThread(threading.Thread):
    def __init__(self):
        super().__init__()
        self._stop_event = threading.Event()
    
    def stop(self):
        self._stop_event.set()
    
    def run(self):
        while not self._stop_event.is_set():
            # 定期检查停止信号
            do_work()
            self._stop_event.wait(timeout=0.1)  # 最多阻塞 100ms
```

**守护线程的特殊销毁**：

```
# 守护线程在主线程退出时被强制终止
daemon_thread = threading.Thread(target=long_running_task, daemon=True)
daemon_thread.start()
# 主线程退出时，daemon_thread 被立即终止
# 不执行 finally 块，不释放资源！
```

**线程异常的处理**：

```
# 3.8+ 可以通过 threading.excepthook 处理未捕获的线程异常
def handle_thread_exception(args):
    """
    args 包含：
    - exc_type: 异常类型
    - exc_value: 异常值
    - exc_traceback: 异常回溯
    - thread: 引发异常的线程
    """
    print(f"Thread {args.thread.name} crashed: {args.exc_type.__name__}: {args.exc_value}")

threading.excepthook = handle_thread_exception

def worker():
    raise RuntimeError("Something went wrong!")

t = threading.Thread(target=worker)
t.start()
t.join()
# 异常不会传播到主线程，但会通过 excepthook 打印
```

**子解释器中的线程销毁（3.12+ PEP 684）**：

> _"Using `Py_NewInterpreterFromConfig` you can create a sub-interpreter that is completely isolated from other interpreters, including having its own GIL... When the call returns, the current thread state is NULL. All thread states associated with this interpreter are destroyed."_

当销毁拥有独立 GIL 的子解释器时：

- 该解释器关联的所有线程状态被销毁
- 目标解释器的 GIL 必须被持有
- 函数返回时不持有任何 GIL

**第一性原理**：

- **Python 线程的销毁是"协作式"的，而非"抢占式"的**。Python 故意不提供 `Thread.kill()` API，因为强制终止线程会破坏 Python 内部状态的一致性。这是 GIL 设计哲学的延伸——既然 GIL 保证了内存管理的线程安全，那么线程销毁也必须"优雅"。
- **线程销毁的真正挑战是"资源清理"**。线程可能在持有锁、打开文件、占用网络连接时被"要求"退出。协作式取消（通过 Event/标志位）允许线程在安全的检查点清理资源后退出。

**洞见——为什么 `Thread.join(timeout)` 可能永远阻塞？**

```
import threading, time

def stuck_thread():
    while True:
        # 没有退出检查！
        time.sleep(1)

t = threading.Thread(target=stuck_thread, daemon=False)
t.start()
t.join(timeout=2)  # 2秒后返回，但线程仍在运行！
print(t.is_alive())  # True——线程还在运行

# 问题：t 永远不会真正结束，进程也不会退出（因为非守护线程）
```

**正确的模式**：

```
class StoppableThread(threading.Thread):
    def __init__(self):
        super().__init__()
        self._running = True
    
    def stop(self):
        self._running = False
    
    def run(self):
        while self._running:
            try:
                do_work()
            except Exception as e:
                print(f"Error: {e}")
                break
        print(f"Thread {self.name} cleaned up and exiting")

# 使用
t = StoppableThread()
t.start()
time.sleep(5)
t.stop()
t.join()  # 这次会真正退出
print("Thread stopped cleanly")
```

**工业实际**：

```
import threading
import queue
import time

# 1. 使用队列发送停止信号（生产者-消费者模式）
def consumer(task_queue, stop_event):
    while not stop_event.is_set():
        try:
            task = task_queue.get(timeout=0.5)
            process_task(task)
        except queue.Empty:
            continue  # 超时，检查 stop_event
        except Exception as e:
            print(f"Task failed: {e}")
        finally:
            task_queue.task_done()

stop_event = threading.Event()
task_queue = queue.Queue()
consumer_thread = threading.Thread(
    target=consumer, 
    args=(task_queue, stop_event)
)
consumer_thread.start()

# 主线程发送任务...
# 停止时：
stop_event.set()
consumer_thread.join()
print("Consumer stopped gracefully")

# 2. 线程池的优雅关闭
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    # 提交任务...
    pass  # 退出 with 块时，executor.shutdown(wait=True) 被自动调用
    # 所有线程完成任务后才退出

# 3. 处理线程中的未捕获异常（3.8+）
import threading

def custom_excepthook(args):
    """记录线程异常到日志"""
    import logging
    logging.error(
        f"Thread {args.thread.name} raised {args.exc_type.__name__}: {args.exc_value}",
        exc_info=(args.exc_type, args.exc_value, args.exc_traceback)
    )

threading.excepthook = custom_excepthook
```

---

## 15.6 Python 线程的用户级互斥与同步

**书里（Python 2.5）**：提供 `thread.allocate_lock()` 创建锁，以及基础的同步原语。

**现代（3.14）的视角**：

`threading` 模块提供的同步原语：

**1. Lock（互斥锁）**：

```
lock = threading.Lock()
# 基本用法
lock.acquire()  # 阻塞获取锁
try:
    # 临界区
    shared_resource += 1
finally:
    lock.release()

# 推荐用法（上下文管理器）
with lock:
    shared_resource += 1
```

**2. RLock（可重入锁）**：

```
rlock = threading.RLock()
# 同一线程可以多次获取
with rlock:
    with rlock:  # 不会死锁
        do_something()
```

**3. Event（事件）**：

```
event = threading.Event()
# 一个线程等待事件
def waiter():
    event.wait()  # 阻塞直到 event.set() 被调用
    print("Event received!")

# 另一个线程触发事件
def trigger():
    time.sleep(1)
    event.set()  # 唤醒所有等待的线程
```

**4. Condition（条件变量）**：

```
condition = threading.Condition()
# 生产者
with condition:
    produce_item()
    condition.notify()  # 唤醒一个等待者
    # condition.notify_all()  # 唤醒所有等待者

# 消费者
with condition:
    while not has_item():
        condition.wait()  # 释放锁并等待通知
    consume_item()
```

**5. Semaphore / BoundedSemaphore（信号量）**：

```
semaphore = threading.Semaphore(3)  # 最多3个线程同时访问
with semaphore:
    access_shared_resource()
```

**6. Barrier（屏障）**：

```
barrier = threading.Barrier(3)  # 3个线程同步点
def worker():
    prepare()
    barrier.wait()  # 等待所有线程到达
    proceed()
```

**Free-threaded build 中的重要警告**（3.13+）：

> _"The free-threaded build of CPython aims to provide similar thread-safety behavior at the Python level to the default GIL-enabled build. Built-in types like `dict`, `list`, and `set` use internal locks to protect against concurrent modifications... **However**, Python has not historically guaranteed specific behavior for concurrent modifications to these built-in types, so this should be treated as a description of the current implementation, not a guarantee... **It's recommended to use the `threading.Lock` or other synchronization primitives instead of relying on the internal locks of built-in types**, when possible."_

**关键警告**：**不要依赖内置类型的"内部锁"做同步**。即使在 free-threaded build 中，dict/list/set 的内部锁也只是为了"保护解释器内部状态不被破坏"，并不保证符合 Python 语义的并发修改安全。

**第一性原理**：

- **同步原语的本质是"把并行执行串行化"**。锁、事件、条件变量等都是为了让多个线程在访问共享资源时协调顺序。GIL 保证了字节码级别的原子性，但不保证 Python 层面的复合操作原子性——这就是为什么需要用户级同步。
- **`with lock:` 不只是语法糖，它是"异常安全的临界区"**。即使在临界区中抛出异常，`finally` 块确保锁被释放。手写 `acquire()`/`release()` 容易忘记释放锁（特别是在异常路径上），导致死锁。

**洞见——为什么 `Lock` 和 `RLock` 的选择很重要？**

```
import threading

# 错误示例：使用 Lock 导致死锁
lock = threading.Lock()

def outer():
    with lock:
        inner()  # 死锁！同一线程试图再次获取 Lock

def inner():
    with lock:  # 阻塞——Lock 不可重入
        do_something()

# 正确示例：使用 RLock
rlock = threading.RLock()

def outer():
    with rlock:
        inner()  # OK——RLock 可重入

def inner():
    with rlock:
        do_something()
```

**何时使用 RLock**：

- 递归函数需要获取锁
- 多个方法需要获取同一个锁（方法间相互调用）
- 不确定调用栈深度时（保守选择）

**何时使用 Lock**：

- 简单的临界区保护
- 性能敏感场景（Lock 比 RLock 轻量）
- 明确不会有重入需求时

**工业实际**：

```
import threading
import time
from typing import Any
from collections.abc import Callable

# 1. 线程安全的单例模式
class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:  # 双重检查锁定
                    cls._instance = super().__new__(cls)
        return cls._instance

# 2. 读写锁的实现（Python 标准库没有，需要自己实现）
class ReadWriteLock:
    def __init__(self):
        self._read_lock = threading.Lock()
        self._write_lock = threading.Lock()
        self._readers = 0
    
    def acquire_read(self):
        with self._read_lock:
            self._readers += 1
            if self._readers == 1:
                self._write_lock.acquire()
    
    def release_read(self):
        with self._read_lock:
            self._readers -= 1
            if self._readers == 0:
                self._write_lock.release()
    
    def acquire_write(self):
        self._write_lock.acquire()
    
    def release_write(self):
        self._write_lock.release()

# 3. 使用 Condition 实现线程安全的队列
class ThreadSafeQueue:
    def __init__(self, maxsize=0):
        self._queue = []
        self._maxsize = maxsize
        self._not_empty = threading.Condition()
        self._not_full = threading.Condition()
        self._size = 0
    
    def put(self, item, block=True, timeout=None):
        with self._not_full:
            if self._maxsize > 0:
                self._not_full.wait_for(lambda: self._size < self._maxsize, timeout)
            self._queue.append(item)
            self._size += 1
            with self._not_empty:
                self._not_empty.notify()
    
    def get(self, block=True, timeout=None):
        with self._not_empty:
            self._not_empty.wait_for(lambda: self._size > 0, timeout)
            item = self._queue.pop(0)
            self._size -= 1
            with self._not_full:
                self._not_full.notify()
            return item

# 4. 死锁检测和避免
def transfer(from_account, to_account, amount):
    """银行转账——需要按固定顺序获取锁以避免死锁"""
    # 按账号 ID 排序获取锁（避免死锁的经典技术）
    first, second = sorted([from_account, to_account], key=lambda x: x.id)
    
    with first.lock:
        with second.lock:
            from_account.balance -= amount
            to_account.balance += amount

# 5. 3.13+ free-threaded build 的同步建议
import sys
if not sys._is_gil_enabled():
    print("Running in free-threaded mode")
    print("Recommendations:")
    print("- Always use explicit locks for shared mutable state")
    print("- Don't rely on built-in types' internal locks")
    print("- Consider thread-local storage for per-thread state")
```

---

## 15.7 高级线程库——threading

**书里（Python 2.5）**：`threading` 模块提供 Thread 类、Lock、RLock、Semaphore、Event、Condition 等基本同步原语。

**现代（3.14）的视角**：

**threading 模块的完整 API**（依据官方文档）：

**线程管理函数**：

- `threading.active_count()`：当前存活的 Thread 对象数
- `threading.current_thread()`：返回当前线程的 Thread 对象
- `threading.get_ident()`：返回当前线程的标识符
- `threading.get_native_id()`：返回 OS 分配的线程 ID（3.8+）
- `threading.enumerate()`：返回所有存活 Thread 对象的列表
- `threading.main_thread()`：返回主线程对象（3.4+）
- `threading.settrace(func)`：为所有线程设置追踪函数（3.10+）
- `threading.setprofile(func)`：为所有线程设置性能分析函数（3.10+）

**线程类（Thread）**：

```
threading.Thread(
    target=None,      # 线程执行的函数
    name=None,        # 线程名
    args=(),         # 位置参数
    kwargs={},       # 关键字参数
    daemon=None      # 是否守护线程
)
```

**方法**：

- `start()`：启动线程
- `run()`：线程执行的入口（可重写）
- `join(timeout=None)`：等待线程结束
- `is_alive()`：检查线程是否存活
- `getName()` / `setName()`：获取/设置线程名

**属性**：

- `ident`：线程标识符
- `native_id`：OS 线程 ID
- `daemon`：是否为守护线程
- `name`：线程名

**高级同步原语**：

- `Lock` / `RLock`：互斥锁 / 可重入锁
- `Event`：事件
- `Condition`：条件变量
- `Semaphore` / `BoundedSemaphore`：信号量
- `Barrier`：屏障
- `Timer`：定时器
- `local`：线程局部数据

**3.8+ 新增的重要特性**：

- `threading.excepthook`：处理线程未捕获异常
- `threading.gettrace()` / `threading.getprofile()`：获取全局追踪/分析函数

**与 `concurrent.futures` 的关系**：

```
# threading 模块是低层 API
# concurrent.futures 是高层的线程池/进程池抽象
from concurrent.futures import ThreadPoolExecutor

# 高层 API——自动管理线程生命周期
with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(task, i) for i in range(10)]
    for future in concurrent.futures.as_completed(futures):
        result = future.result()
```

**工业实践中的 threading 使用模式**：

**1. 线程池模式**（推荐）：

```
from concurrent.futures import ThreadPoolExecutor
import urllib.request

def fetch_url(url):
    with urllib.request.urlopen(url) as response:
        return response.read()

urls = ["http://example.com"] * 10
with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch_url, urls))
```

## 15.7 高级线程库——threading（续）

接上文，我们补全"生产者-消费者模式"的完整代码，并继续展开这一节剩余的内容。

### 补全：生产者-消费者模式（工业标准实现）

```
# 2. 生产者-消费者模式（完整工业实现）
import queue
import threading
import time
import random

task_queue = queue.Queue(maxsize=5)  # 有界队列，防止内存爆炸
stop_event = threading.Event()       # 协作式停止信号

def producer(name):
    """生产者：模拟生成任务"""
    for i in range(5):
        if stop_event.is_set():
            break
        task = f"{name}-task-{i}"
        # put() 在队列满时自动阻塞，无需手动加锁
        task_queue.put(task)
        print(f"[{time.strftime('%H:%M:%S')}] {name} produced: {task}")
        time.sleep(random.uniform(0.1, 0.3))  # 模拟生产耗时
    print(f"{name} finished producing")

def consumer(name):
    """消费者：处理任务直到收到停止信号"""
    while not stop_event.is_set():
        try:
            # get(timeout=1) 避免永久阻塞，定期检查 stop_event
            task = task_queue.get(timeout=1)
            print(f"[{time.strftime('%H:%M:%S')}] {name} consumed: {task}")
            time.sleep(random.uniform(0.2, 0.5))  # 模拟处理耗时
            task_queue.task_done()  # 通知队列任务已完成
        except queue.Empty:
            continue  # 超时，重新检查 stop_event
    print(f"{name} shutting down")

# 启动 1 个生产者、2 个消费者
p = threading.Thread(target=producer, args=("Producer-A",))
c1 = threading.Thread(target=consumer, args=("Consumer-1",))
c2 = threading.Thread(target=consumer, args=("Consumer-2",))

p.start(); c1.start(); c2.start()

# 运行 3 秒后发送停止信号
time.sleep(3)
stop_event.set()

# 等待所有线程优雅退出
p.join(); c1.join(); c2.join()
print("All threads stopped gracefully")
```

**为什么 `queue.Queue` 是线程通信的首选？**

- 内部已封装 `Condition` + `Lock`，无需手动同步
- `put()` / `get()` 自动阻塞，避免忙等（busy-waiting）
- `task_done()` + `join()` 提供任务完成追踪
- 有界队列（`maxsize`）天然提供背压（backpressure），防止生产者压垮消费者

### 其他高级模式

**3. Timer（延迟执行）**：

```
def delayed_task():
    print(f"[{time.strftime('%H:%M:%S')}] Delayed task executed!")

timer = threading.Timer(2.0, delayed_task)  # 2秒后执行
timer.start()
# timer.cancel() 可以取消（在启动后、执行前）
```

**4. threading.local（线程局部存储）**：

```
thread_local = threading.local()

def worker():
    thread_local.value = threading.current_thread().ident
    print(f"Thread {threading.current_thread().name}: value={thread_local.value}")

threads = [threading.Thread(target=worker) for _ in range(3)]
for t in threads: t.start()
for t in threads: t.join()
# 每个线程有独立的 value，互不干扰——避免了共享状态
```

**5. Barrier（多线程同步点）**：

```
barrier = threading.Barrier(3)  # 3个线程必须都到达才能继续

def worker(name):
    print(f"{name} preparing...")
    time.sleep(random.uniform(0.5, 2))
    print(f"{name} ready, waiting for others...")
    barrier.wait()  # 阻塞直到3个线程都到达
    print(f"{name} all ready, proceeding!")

threads = [threading.Thread(target=worker, args=(f"T{i}",)) for i in range(3)]
for t in threads: t.start()
for t in threads: t.join()
```

### 15.7 的第一性原理

- **`threading` 模块 = `_thread` 的面向对象封装 + 高层同步原语**。底层 `_thread` 只提供"创建线程"和"原始锁"，而 `threading` 提供了 `Thread` 对象、生命周期管理（`start`/`join`/`is_alive`）、以及丰富的同步工具（`Lock`/`Event`/`Condition`/`Queue` 等）。
- **`queue.Queue` 是"最安全的线程通信方式"**。它把"锁 + 条件变量 + 缓冲区"封装成一个类，用户只需要 `put()` 和 `get()`，不需要关心内部同步细节。这是"不要自己实现同步"原则的最佳实践。
- **`threading.local` 通过字典映射线程 ID 到数据**。每个线程访问 `local.x` 时，实际访问的是 `local.__dict__[thread_id]['x']`。这使得线程可以拥有"全局可见但线程私有"的变量，是 Web 框架（如 Flask 的 `request` 上下文）的核心机制。

### 15.7 的洞见——为什么 `queue.Queue` 比手动 `list + Lock` 更好？

```
# 错误示例：手动用 list + Lock
shared_list = []
lock = threading.Lock()

def bad_producer():
    with lock:
        shared_list.append("task")

def bad_consumer():
    while True:
        with lock:
            if shared_list:
                task = shared_list.pop(0)  # list.pop(0) 是 O(n) 操作！
                process(task)
        time.sleep(0.01)  # 忙等！浪费 CPU
```

问题：

1. **忙等**：消费者不断轮询，浪费 CPU
2. **性能**：`list.pop(0)` 是 O(n)，高并发下成为瓶颈
3. **复杂性**：手动管理锁容易出错（忘记释放、死锁）

`queue.Queue` 用 `collections.deque`（O(1) 的 `append`/`popleft`）+ `Condition`（等待-通知）解决了所有问题。

### 15.7 工业实际

```
# 1. Web 服务器的线程池（简化版 Flask 开发服务器模型）
from concurrent.futures import ThreadPoolExecutor
import socket

def handle_request(client_socket):
    request = client_socket.recv(1024)
    # 处理请求...
    client_socket.send(b"HTTP/1.1 200 OK\r\n\r\nHello!")
    client_socket.close()

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('localhost', 8080))
server.listen(5)
print("Server listening on port 8080")

with ThreadPoolExecutor(max_workers=10) as executor:
    while True:
        client_sock, addr = server.accept()
        executor.submit(handle_request, client_sock)
        # 每个请求在独立线程中处理，I/O 时释放 GIL

# 2. 后台心跳 + 主工作线程（守护线程的典型用法）
def heartbeat():
    while True:
        print(f"[{time.strftime('%H:%M:%S')}] Heartbeat OK")
        time.sleep(5)

hb_thread = threading.Thread(target=heartbeat, daemon=True)
hb_thread.start()

# 主工作
do_main_work()
# 主工作结束，进程退出，守护线程自动终止

# 3. 3.13+ free-threaded build 的 threading 注意事项
import sys
if hasattr(sys, '_is_gil_enabled') and not sys._is_gil_enabled():
    print("Free-threaded mode: threading.Lock is essential for shared state")
    print("Built-in types' internal locks are NOT sufficient for complex logic")
```

---

## 第 15 章合体总结：Python 多线程的全景图

|阶段/主题|书里（Python 2.5）|现代（Python 3.14）|
|---|---|---|
|**GIL 切换机制**​|每 100 条字节码指令切换（检查间隔）|每 5ms 切换（`sys.setswitchinterval`，3.2+）|
|**GIL 调度**​|CPython 有简单调度逻辑|CPython 无调度器，OS 决定谁获得 GIL|
|**线程模块**​|`thread` 公开模块|`_thread` 内部模块，`threading` 始终可用|
|**线程切换**​|基于字节码计数|基于绝对时长，实际值可能更高|
|**子解释器**​|共享 GIL|3.12+ PEP 684 允许每解释器独立 GIL|
|**Free-threaded**​|不存在|3.13 实验性，3.14 正式支持（PEP 703）|
|**线程终止**​|无安全终止机制|协作式取消（Event/标志位），无强制 kill|
|**同步原语**​|Lock/RLock/Event/Condition/Semaphore|新增 Barrier、Timer、threading.local 等|
|**线程池**​|无标准库支持|`concurrent.futures.ThreadPoolExecutor`|
|**内置类型线程安全**​|依赖 GIL|free-threaded build 有内部锁但不推荐依赖|

### 三个必须带走的洞见

**洞见一：GIL 不是线程调度器，它只是一个互斥锁。**

- CPython 本身没有线程调度器——`sys.setswitchinterval()` 只是设置"理想切换间隔"，实际哪个线程获得 GIL 由 OS 决定。
- 单条字节码指令是原子的（因为不会在指令中间切换），但 Python 层面的复合操作（如 `+=`）由多条字节码组成，**不是原子的**。
- 调优 `setswitchinterval` 对 CPU 密集型任务无效——因为 GIL 确保串行执行。它只对需要公平调度的 I/O 密集型任务有意义。

**洞见二：Python 多线程的"有效场景"是 I/O 密集型，不是 CPU 密集型。**

- I/O 操作时 GIL 被释放，多个线程可以真正并行等待 I/O
- CPU 密集型任务应该用 `multiprocessing` 或多进程，因为 GIL 阻止了并行执行 Python 字节码
- 某些 C 扩展（NumPy 等）在执行核心计算时释放 GIL，所以数据科学中 Python 可以"看起来"并行——实际上是 C 层在并行

**洞见三：协作式取消是唯一安全的线程终止方式。**

- Python 故意不提供 `Thread.kill()`——强制杀线程会破坏内部状态（锁未释放、文件未关闭、数据结构不一致）
- 正确的模式：用 `threading.Event` 作为停止信号，线程定期检查并优雅退出
- 守护线程在主线程退出时被强制终止，不执行 `finally` 块——只适合"可有可无"的后台任务

### 立刻可做的实验

1. **观测 GIL 切换行为**：
    
    ```
    import sys, threading, time
    
    def cpu_task():
        total = 0
        for i in range(5_000_000):
            total += i
    
    print(f"Current switch interval: {sys.getswitchinterval()}s")
    
    t0 = time.time()
    t1 = threading.Thread(target=cpu_task)
    t2 = threading.Thread(target=cpu_task)
    t1.start(); t2.start()
    t1.join(); t2.join()
    print(f"Two threads: {time.time() - t0:.2f}s")
    
    # 对比单线程
    t0 = time.time()
    cpu_task(); cpu_task()
    print(f"Sequential: {time.time() - t0:.2f}s")
    ```
    
2. **验证 I/O 密集型 vs CPU 密集型**：
    
    ```
    import threading, time, urllib.request
    
    def io_task():
        urllib.request.urlopen('http://httpbin.org/delay/1').read()
    
    def cpu_task():
        sum(range(10_000_000))
    
    # I/O 多线程（应该接近单线程时间，因为并行等待）
    t0 = time.time()
    threads = [threading.Thread(target=io_task) for _ in range(4)]
    for t in threads: t.start()
    for t in threads: t.join()
    print(f"I/O 4 threads: {time.time() - t0:.2f}s")  # ~1s（并行）
    
    # CPU 多线程（应该接近单线程的2倍，因为串行）
    t0 = time.time()
    threads = [threading.Thread(target=cpu_task) for _ in range(2)]
    for t in threads: t.start()
    for t in threads: t.join()
    print(f"CPU 2 threads: {time.time() - t0:.2f}s")  # ~2x 单线程时间
    ```
    
3. **死锁演示与避免**：
    
    ```
    import threading
    
    lock_a = threading.Lock()
    lock_b = threading.Lock()
    
    def thread_1():
        with lock_a:
            time.sleep(0.1)
            with lock_b:  # 可能死锁
                print("Thread 1 got both locks")
    
    def thread_2():
        with lock_b:
            time.sleep(0.1)
            with lock_a:  # 可能死锁
                print("Thread 2 got both locks")
    
    # 避免死锁：按固定顺序获取锁
    def safe_thread_1():
        first, second = sorted([lock_a, lock_b], key=id)
        with first:
            with second:
                print("Safe thread got both locks")
    ```
    
4. **观测线程栈（调试利器）**：
    
    ```
    import sys, threading, traceback
    
    def dump_all_stacks():
        for tid, frame in sys._current_frames().items():
            print(f"\nThread {tid}:")
            traceback.print_stack(frame)
    
    # 在死锁或卡住时调用 dump_all_stacks()
    ```
    

### 版本差异速查

|书里（Python 2.5）|现代去哪儿找|
|---|---|
|`thread` 公开模块|`_thread` 内部模块（3.7+）|
|`thread.start_new_thread()`|`_thread.start_new_thread()` 或 `threading.Thread`|
|检查间隔 100 条字节码|`sys.setswitchinterval()` 默认 5ms（3.2+）|
|`threading` 可选模块|`threading` 始终可用（3.7+）|
|`threading.currentThread()` 等 camelCase|3.10 弃用，用 `threading.current_thread()`|
|无 `threading.excepthook`|3.8+ 提供，处理未捕获线程异常|
|无 `threading.get_native_id()`|3.8+ 提供，返回 OS 线程 ID|
|无 `concurrent.futures`|3.2+ 提供高层线程池/进程池|
|无 free-threaded build|3.13 实验性，3.14 正式支持（PEP 703）|
|子解释器共享 GIL|3.12+ PEP 684 允许独立 GIL|

