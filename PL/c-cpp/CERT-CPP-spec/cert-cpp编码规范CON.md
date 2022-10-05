## 并发编程

### CON50-CPP. 当mutex被上锁的时候不要销毁mutex

严重程度：中等。修复代价也非常高昂。导致不合法的控制流和数据冲撞。

Mutex对象一般是用来在并发访问的时候保护共享数据的。如果一个Mutex对象在线程阻塞等待上锁的时候销毁了，那么临界区和共享数据就再也不受保护了。

C++标准[thread.mutex.class]中有如下声明:

> The behavior of a program is undefined if it destroys a mutex object owned by any thread or a thread terminates while owning a mutex object

同样，条款说的Mutex对象在C++中包括std::mutex std::recursive_mutex, std::timed_mutex,
std::recursive_timed_mutex, and std::shared_timed_mutex。

#### 代码样例对比

``` cpp
#include <mutex>
#include <thread>

const size_t maxThreads = 10;

void do_work(size_t i, std::mutex *pm) {
  std::lock_guard<std::mutex> lk(*pm);
  // Access data protected by the lock.
}

void start_threads() {
  std::thread threads[maxThreads];
  std::mutex m;
  for (size_t i = 0; i < maxThreads; ++i) {
    threads[i] = std::thread(do_work, i, &m);
  }
}
```
以上代码看上去貌似没有问题，但是仔细看，mutex对象m是在start_threads()函数作用域的，可能线程还没有开始，或进行到一半，对象m就随着退出作用域自动销毁了，这时候各个线程再来操作对象m，显然是非法的，直接造成未定义行为。所以可以把对象m存放在全局作用域:

``` cpp
#include <mutex>
#include <thread>

const size_t maxThreads = 10;
std::mutex m;

void do_work(size_t i, std::mutex *pm) {
  std::lock_guard<std::mutex> lk(*pm);
  // Access data protected by the lock.
}

void start_threads() {
  std::thread threads[maxThreads];
  for (size_t i = 0; i < maxThreads; ++i) {
    threads[i] = std::thread(do_work, i, &m);
  }
}
```
以上代码在线程运行完毕也不会销毁对象m，所以可以放心运行，如果还是想把对象m存放到start_threads的作用域，那么可以让所有线程都join到主线程来等待线程运行完毕再销毁对象m：

``` cpp
#include <mutex>
#include <thread>

const size_t maxThreads = 10;

void do_work(size_t i, std::mutex *pm) {
  std::lock_guard<std::mutex> lk(*pm);
  // Access data protected by the lock.
}

void start_threads() {
  std::thread threads[maxThreads];
  std::mutex m;

  for (size_t i = 0; i < maxThreads; ++i) {
    threads[i] = std::thread(do_work, i, &m);
  }

  for (size_t i = 0; i < maxThreads; ++i) {
    threads[i].join();
  }
}
```

### CON51-CPP. 确保持有锁的线程在遇到异常的时候及时释放锁

严重程度: 低。一般会导致死锁。

Mutex对象一般通过调用其成员函数lock()上锁和unlock()解锁来保护共享数据，但是，如果一个异常在lock和unlock之间的代码段发生了，因为异常会直接改变中断控制流，这样就导致unlock没有调用，mutex就一直保持locked的状态，当其他线程想进入临界区或保护区段的时候，会一直阻塞等待锁的释放，但是显然已经不可能了，导致死锁。

异常的throw永远不允许一个mutex永远保持上锁的状态。

C++提供了一些lock_guard, unique_lock和shared_lock的RAII类来保证锁跳出作用域的时候及时得到释放。

#### 代码样例对比

``` cpp
#include <mutex>

void manipulate_shared_data(std::mutex &pm) {
  pm.lock();
  
  // Perform work on shared data.
  // if exception happens throw and did not call unlock()
  pm.unlock();
}
```
以上代码显然有问题，如果异常发生可能直接造成死锁，应该保证捕获异常，并且释放锁:

``` cpp
#include <mutex>

void manipulate_shared_data(std::mutex &pm) {
  pm.lock();
  try {
    // Perform work on shared data.
  } catch (...) {
    pm.unlock();
    throw;
  }
  pm.unlock(); // in case no exceptions occur
}
```
C++ 11推荐用RAII的机制来完成这样的方案，最简洁高效，降低了开发者的心智负担:

``` cpp
#include <mutex>

void manipulate_shared_data(std::mutex &pm) {
  std::lock_guard<std::mutex> lk(pm);
  // Perform work on shared data.
  // if exception happens and throw it , resulting manipulate_shared_data() exit its scope and lk object desctroied and release pm  
}
```

### CON52-CPP. 在多线程访问bit-fields的时候防止数据竞争

严重程度: 中等。尽管竞争的区域窗口(粒度)比较小，但是赋值和表达式的计算可能会由于数据的错误解释导致程序运行在一个未知的状态。

当访问一个bit-field，线程会无心的分开访问邻近内存的bit-field。这是由于编译器总是要求临近的不同bit-field存储在一个存储单元中（字节）,结果就导致了相邻bit-field还是会出现数据竞争的问题，还是必须用锁来保护。这样的错误很难发现。

#### 代码样例对比

``` cpp
struct MultiThreadedFlags {
  unsigned int flag1 : 2;
  unsigned int flag2 : 2;
};

MultiThreadedFlags flags;

void thread1() {
  flags.flag1 = 1;
}

void thread2() {
  flags.flag2 = 2;
}
```
以上代码显然有问题，flag1和flag2因为是bit-filed的表达，所以这两个存储区域可能是放在同一个内存地址的存储单元中，如果多线程访问不加锁保护，那么会导致未定义行为。所以需要加锁:

``` cpp
#include <mutex>

struct MultiThreadedFlags {
  unsigned int flag1 : 2;
  unsigned int flag2 : 2;
};

struct MtfMutex {
  MultiThreadedFlags s;
  std::mutex mutex;
};

MtfMutex flags;

void thread1() {
  std::lock_guard<std::mutex> lk(flags.mutex);
  flags.s.flag1 = 1;
}

void thread2() {
  std::lock_guard<std::mutex> lk(flags.mutex);
  flags.s.flag2 = 2;
}
```
以上代码用mutex对象来保护每一个MultiThreadFlags类型的对象。注意不能把mutex放到全局空间，这样就是保护范围就扩大了，没有意义，因为不同MultiThreadFlags类型的对象是没有任何关联的。当然如果不用bit-field来存储信息，以下代码就不用加锁了，因为不同的存储变量肯定是存储在不同的内存单元中（C++ 11）：

``` cpp
struct MultiThreadedFlags {
  unsigned char flag1;
  unsigned char flag2;
};

MultiThreadedFlags flags;

void thread1() {
  flags.flag1 = 1;
}

void thread2() {
  flags.flag2 = 2;
}
```
以上代码必须用支持C++ 11的编译器才有效果，因为C++标准[intro.memory]显式定义了内存地址：

> [Note: Thus a bit-field and an adjacent non-bit-field are in separate memory locations, and therefore can be concurrently updated by two threads of execution without interference. The same applies to two bit-fields, if one is declared inside a nested struct declaration and the other is not, or if the two are separated by a zero-length bit-field declaration, or if they are separated by a non-bit-field declaration. It is not safe to concurrently update two bit-fields in the same struct if all fields between them are also bit-fields of non-zero width. – end note ]

### CON53-CPP. 通过预先定义的顺序来加锁以避免出现死锁

严重程度： 低。 死锁一般会造成程序僵死，然后被攻击者构造死锁条件导致服务拒绝攻击。

这段没什么可说的，死锁必须得避免是必须的。

#### 代码样例对比

``` cpp
#include <mutex>
#include <thread>

class BankAccount {
  int balance;
public:
  std::mutex balanceMutex;
  BankAccount() = delete;
  explicit BankAccount(int initialAmount) : balance(initialAmount){}
  int get_balance() const { return balance; }
  void set_balance(int amount) { balance = amount; }
};

int deposit(BankAccount *from, BankAccount *to, int amount) {
  std::lock_guard<std::mutex> from_lock(from->balanceMutex);
  // Not enough balance to transfer.
  if (from->get_balance() < amount) {
    return -1; // Indicate error
  }
  std::lock_guard<std::mutex> to_lock(to->balanceMutex);
  from->set_balance(from->get_balance() – amount);
  to->set_balance(to->get_balance() + amount);
  return 0;
}

void f(BankAccount *ba1, BankAccount *ba2) {
  // Perform the deposits.
  std::thread thr1(deposit, ba1, ba2, 100);
  std::thread thr2(deposit, ba2, ba1, 100);
  thr1.join();
  thr2.join();
}
```
以上代码由于银行账户互相转账，但是加锁的顺序不一致，导致可能有死锁隐患。可以手动调整加锁顺序来消除:

``` cpp
#include <atomic>
#include <mutex>
#include <thread>

class BankAccount {
  static std::atomic<unsigned int> globalId;
  const unsigned int id;
  int balance;
public:
  std::mutex balanceMutex;
  BankAccount() = delete;
  explicit BankAccount(int initialAmount) : id(globalId++), balance(initialAmount) {}
  unsigned int get_id() const { return id; }
  int get_balance() const { return balance; }
  void set_balance(int amount) { balance = amount; }
};

std::atomic<unsigned int> BankAccount::globalId(1);

int deposit(BankAccount *from, BankAccount *to, int amount) {
  std::mutex *first;
  std::mutex *second;
  if (from->get_id() == to->get_id()) {
    return -1; // Indicate error
  }
  // Ensure proper ordering for locking.
  if (from->get_id() < to->get_id()) {
    first = &from->balanceMutex;
    second = &to->balanceMutex;
  } else {
    first = &to->balanceMutex;
    second = &from->balanceMutex;
  }

  std::lock_guard<std::mutex> firstLock(*first);
  std::lock_guard<std::mutex> secondLock(*second);
  // Check for enough balance to transfer.
  if (from->get_balance() >= amount) {
    from->set_balance(from->get_balance() – amount);
    to->set_balance(to->get_balance() + amount);
    return 0;
  }
  return -1;
}

void f(BankAccount *ba1, BankAccount *ba2) {
  // Perform the deposits.
  std::thread thr1(deposit, ba1, ba2, 100);
  std::thread thr2(deposit, ba2, ba1, 100);
  thr1.join();
  thr2.join();
}
```
或者用STL提供了std::lock同时锁住多个锁对象来避免死锁:

``` cpp
#include <mutex>
#include <thread>

class BankAccount {
  int balance;
public:
  std::mutex balanceMutex;
  BankAccount() = delete;
  explicit BankAccount(int initialAmount) :
  balance(initialAmount) {}
  int get_balance() const { return balance; }
  void set_balance(int amount) { balance = amount; }
};

int deposit(BankAccount *from, BankAccount *to, int amount) {
  // Create lock objects but defer locking them until later.
  std::unique_lock<std::mutex> lk1(from->balanceMutex, std::defer_lock);
  std::unique_lock<std::mutex> lk2(to->balanceMutex, std::defer_lock);
  
  // Lock both of the lock objects simultaneously.
  std::lock(lk1, lk2);
  if (from->get_balance() >= amount) {
    from->set_balance(from->get_balance() – amount);
    to->set_balance(to->get_balance() + amount);
    return 0;
  }
  return -1;
}

void f(BankAccount *ba1, BankAccount *ba2) {
  // Perform the deposits.
  std::thread thr1(deposit, ba1, ba2, 100);
  std::thread thr2(deposit, ba2, ba1, 100);
  thr1.join();
  thr2.join();
}
```

### CON54-CPP. 包装在循环中能伪唤醒的函数

严重程度： 低。如果在循环中对wait(), wait_for(), 或 wait_until()的调用包装不合理就会导致无限阻塞和服务拒绝攻击。

std::condition_variable类成员函数 wait(), wait_for()和wait_until()会临时放弃mutex对象的拥有权。然后其他线程就可以继续处理这个mutex对象了。这些函数必须总是被mutex保护区域的代码调用，只有等到被其他线程调用notify_one()或notify_all() 成员函数，线程才会继续执行。

比如像下面使用:

``` cpp
#include <condition_variable>
#include <mutex>

extern bool until_finish(void);
extern std::mutex m;
extern std::condition_variable condition;

void func(void) {
  std::unique_lock<std::mutex> lk(m);
  while (until_finish()) { // Predicate does not hold.
    condition.wait(lk);
  }
  // Resume when condition holds.
}
```

#### 代码样例对比

``` cpp
#include <condition_variable>
#include <mutex>

struct Node {
  void *node;
  struct Node *next;
};

static Node list;
static std::mutex m;
static std::condition_variable condition;

void consume_list_element(std::condition_variable &condition) {
  std::unique_lock<std::mutex> lk(m);
  if (list.next == nullptr) {
    condition.wait(lk);
  }
  // Proceed when condition holds.
}
```
以上代码是等到链表非空的时候再消费链表，用法用错了，必须把wait()的调用包装在循环中:

``` cpp
#include <condition_variable>
#include <mutex>

struct Node {
  void *node;
  struct Node *next;
};

static Node list;
static std::mutex m;
static std::condition_variable condition;

void consume_list_element() {
  std::unique_lock<std::mutex> lk(m);
  while (list.next == nullptr) {
    condition.wait(lk);
  }
  // Proceed when condition holds.
}
```
当然以下还有使用lambda表达式的隐含循环来使代码更简洁:

``` cpp
#include <condition_variable>
#include <mutex>

struct Node {
  void *node;
  struct Node *next;
};

static Node list;
static std::mutex m;
static std::condition_variable condition;

void consume_list_element() {
  std::unique_lock<std::mutex> lk(m);
  condition.wait(lk, []{ return list.next; });
  // Proceed when condition holds.
}
```
以上代码wait()实现使用了 while (!wait_condition()) wait(lock);的方式来进行等到，本质上也要用到循环。

### CON55-CPP. 在使用condition variable的时候保持线程安全和线程活动(存活)

严重程度： 低。不遵守条款一般是导致无限阻塞或者服务拒绝攻击。

线程存活一般来说就是保证不死锁。

condition variable一定要在while循环中使用(遵守CON54-CPP规范)。为了保证存活，程序需要在调用wait()成员函数之前用while循环的条件来测试等待条件。直到notify_all或者notify_one被其他线程调用，wait()被唤醒，再次检测条件谓词，决定是否继续等待。

notify_one()成员函数一般会解除阻塞与特定条件变量的其中一个线程。如果多个线程都等待同一个条件变量，那么调度器就选择其中任意一个线程将其唤醒（假设所有等待中的线程都处于相同的优先级）。

notify_all()成员函数一般会解除阻塞于特定条件变量的所有线程，当然，解除后，等待中的线程的开始执行顺序是未指定的。

对于以上所描述的原因，所以线程必须在wait()函数后就检查条件谓词。while循环就是在wait()函数调用前后检查条件谓词的最好选择。

在多线程中，如果每一个线程都使用属于自己的一个唯一的条件变量，那么notify_one()的使用是安全的。如果多线程共享一个条件变量，那么只有满足以下两个条款，notify_one()的使用才是安全的：

- 所有的线程在唤醒之后必须执行相同的操作集合，因为对于一次notify_one()的调用只能唤醒其中一个等待在条件变量上的线程

- 只有一个线程才能接收到notify_one()的信号被唤醒

notify_all()函数一般是对于notify_one()的使用不安全的时候用的，它可以唤醒所有阻塞在特定条件变量上的线程。

#### 代码样例对比

``` cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

std::mutex mutex;
std::condition_variable cond;

void run_step(size_t myStep) {
  static size_t currentStep = 0;
  std::unique_lock<std::mutex> lk(mutex);
  std::cout << "Thread " << myStep << " has the lock" << std::endl;
  
  while (currentStep != myStep) {
    std::cout << "Thread " << myStep << " is sleeping..." << std::endl;
    cond.wait(lk);
    std::cout << "Thread " << myStep << " woke up" << std::endl;
  }

  // Do processing...
  std::cout << "Thread " << myStep << " is processing..." << std::endl;
  currentStep++;

  // Signal awaiting task.
  cond.notify_one();
  std::cout << "Thread " << myStep << " is exiting..." << std::endl;
}

int main() {
  constexpr size_t numThreads = 5;
  std::thread threads[numThreads];
  // Create threads.
  for (size_t i = 0; i < numThreads; ++i) {
    threads[i] = std::thread(run_step, i);
  }

  // Wait for all threads to complete.
  for (size_t i = numThreads; i != 0; --i) {
    threads[i - 1].join();
  }
}
```
以上代码显然是创建多个线程然后通过一个共享局部静态变量currentStep串行化处理，每一个线程处理完成自己的工作就对currentStep加1。而且代码中的wait()也用while循环包装了，遵循CON54-CPP规范。那么错在哪里了呢？错就错在代码是用notify_one()，试想以下，最开始的时候，所有线程依次开始执行，必然有其中一个线程，也就是myStep为0的线程不会阻塞，其他线程都阻塞在条件变量上了，因为没有到自己执行。然而，myStep为0的线程调用了notify_one()，仅仅只唤醒了其中一个线程，如果是恰好唤醒了myStep为1的线程那还好，线程可以如愿被唤醒，如果是唤醒了myStep > 1 的线程那么那个线程还是因为条件不满足而继续等待，又会被阻塞，那么这时候完蛋了，所有线程都被阻塞住了，没有活动线程了，线程都不会被唤醒，那么整个程序就死锁了，无限阻塞。所以需要把notify_one替换成notify_all：

``` cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

std::mutex mutex;
std::condition_variable cond;

void run_step(size_t myStep) {
  static size_t currentStep = 0;
  std::unique_lock<std::mutex> lk(mutex);
  std::cout << "Thread " << myStep << " has the lock" << std::endl;
  
  while (currentStep != myStep) {
    std::cout << "Thread " << myStep << " is sleeping..." << std::endl;
    cond.wait(lk);
    std::cout << "Thread " << myStep << " woke up" << std::endl;
  }

  // Do processing...
  std::cout << "Thread " << myStep << " is processing..." << std::endl;
  currentStep++;

  // Signal awaiting task.
  cond.notify_all();
  std::cout << "Thread " << myStep << " is exiting..." << std::endl;
}

int main() {
  constexpr size_t numThreads = 5;
  std::thread threads[numThreads];
  // Create threads.
  for (size_t i = 0; i < numThreads; ++i) {
    threads[i] = std::thread(run_step, i);
  }

  // Wait for all threads to complete.
  for (size_t i = numThreads; i != 0; --i) {
    threads[i - 1].join();
  }
}
```
以上代码唤醒了所有等待在条件变量的线程，那么必然有其中一个线程满足条件，并唤醒剩下等待在条件变量下的线程，保证线程能依次执行。

如果开发者还是想使用notify_one()，那么有一个解决方案就是每个线程只用一个属于自己的唯一的条件变量:

``` cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

constexpr size_t numThreads = 5;
std::mutex mutex;
std::condition_variable cond[numThreads];

void run_step(size_t myStep) {
  static size_t currentStep = 0;
  std::unique_lock<std::mutex> lk(mutex);
  std::cout << "Thread " << myStep << " has the lock" << std::endl;
  
  while (currentStep != myStep) {
    std::cout << "Thread " << myStep << " is sleeping..." << std::endl;
    cond[myStep].wait(lk);
    std::cout << "Thread " << myStep << " woke up" << std::endl;
  }

  // Do processing ...
  std::cout << "Thread " << myStep << " is processing..." << std::endl;
  currentStep++;

  // Signal next step thread.
  if ((myStep + 1) < numThreads) {
    cond[myStep + 1].notify_one();
  }
  
  std::cout << "Thread " << myStep << " is exiting..." << std::endl;
}

int main() {
  constexpr size_t numThreads = 5;
  std::thread threads[numThreads];
  // Create threads.
  for (size_t i = 0; i < numThreads; ++i) {
    threads[i] = std::thread(run_step, i);
  }

  // Wait for all threads to complete.
  for (size_t i = numThreads; i != 0; --i) {
    threads[i - 1].join();
  }
}
```
以上代码就用notify_one()按顺序依次唤醒下一个处理线程。这方案总体来说会比notify_all()的方案更加高效。因为唤醒的目标线程是准确的。

### CON56-CPP. 在一个已获得锁的调用线程中不要“推测性”地对一个不支持递归的mutex再次上锁

严重程度： 低。 修复代价很高。用递归的方式上锁不支持递归的mutex是未定义行为，一般现象是直接造成死锁

C++ 标准库提供两种mutex来保护临界区（非递归和递归的mutex）。递归的mutex(std::recursive_mutex和std::recursive_timed_mutex)与非递归的mutex(std::mutex, std::timed_mutex, 和 std::shared_timed_mutex)的区别就在于递归的mutex可以被同一个线程递归的上锁多次。

所有的mutex类都提供以下“推测性”的lock函数：

try_lock()
try_lock_for()
try_lock_until()
try_lock_shared_for ()
try_lock_shared_until()

以上的函数一般是用来试图获取调用线程的mutex所有权，但是在试图获取的同时不会阻塞，调用之后会立即返回bool来判断是否lock成功。

C++标准[thread.mutex.requirements.mutex]中有如下所述:

> The expression m.try_lock() shall be well-formed and have the following semantics: Requires: If m is of type std::mutex, std::timed_mutex, or std::shared_timed_mutex, the calling thread does not own the mutex

另外在[thread.timedmutex.class]有如下声明:

> The behavior of a program is undefined if:

> • a thread that owns a timed_mutex object calls lock(), try_lock(), try_lock_for(), or try_lock_until() on that object

最后，[thread.sharedtimedmutex.class]中有如下声明:

> The behavior of a program is undefined if:

> • a thread attempts to recursively gain any ownership of a shared_timed_mutex.

因此对于已经获取锁的调用线程再次试图对一个非递归的mutex上锁时，直接会造成未定义行为。

#### 代码样例对比

``` cpp
#include <mutex>
#include <thread>

std::mutex m;
void do_thread_safe_work();

void do_work() {
  while (!m.try_lock()) {
    // The lock is not owned yet, do other work while waiting.
    do_thread_safe_work();
  }

  try {
    // The mutex is now locked; perform work on shared resources.
    // ...
    // Release the mutex.
  catch (...) {
    m.unlock();
    throw;
  }
  m.unlock();
}

void start_func() {
  std::lock_guard<std::mutex> lock(m);
  do_work();
}

int main() {
  std::thread t(start_func);
  do_work();
  t.join();
}
```
以上代码显然，线程t中，试图对非递归的mutex进行多次上锁，会造成未定义行为。当然，对于大多数编译器产商提供了一个同样的实现，程序会直接表现为死锁。所以可以改成以下代码,不要在线程入口函数中锁住mutex:

``` cpp
#include <mutex>
#include <thread>

std::mutex m;
void do_thread_safe_work();

void do_work() {
  while (!m.try_lock()) {
    // The lock is not owned yet, do other work while waiting.
    do_thread_safe_work();
  }

  try {
    // The mutex is now locked; perform work on shared resources.
    // ...
    // Release the mutex.
  catch (...) {
    m.unlock();
    throw;
  }
  m.unlock();
}

void start_func() {
  do_work();
}

int main() {
  std::thread t(start_func);
  do_work();
  t.join();
}
```