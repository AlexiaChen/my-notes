
## 面向对象编程

### OOP50-CPP. 不要在构造函数或析构函数中调用虚函数

严重程度: 低。

虚函数允许成员函数根据对象的动态类型在运行时静态分派调用。一般这样的规范在面向对象编程的实践规范需要类的继承和函数覆盖。

然而在对象的构造和析构期间，对象的虚函数分派被限制了。C++标准[class.cdtor]有如下声明:

> Member functions, including virtual functions, can be called during construction or destruction. When a virtual function is called directly or indirectly from a constructor or from a destructor, including during the construction or destruction of the class’s nonstatic data members, and the object to which the call applies is the object (call it x) under construction or destruction, the function called is the final overrider in the constructor’s or destructor’s class and not one overriding it in a more-derived class. If the virtual function call uses an explicit class member access and the object expression refers to the complete object of x or one of that object’s base class subobjects but not x or one of its base class subobjects, the behavior is undefined.

在构造函数或析构函数中都不能直接调用或者间接调用虚函数。因为对象的构造顺序是先构造基类，再一级一级的往下推，试想一下，虚函数是为了实现面向对象的多态行为，也就是继承类覆盖了虚函数，对虚函数做实现，如果一个继承类的对象在构造函数或析构函数期间调用了自己的虚函数，那么该对象都没有构造完成，对象资源没有初始化完成就调用会直接造成崩溃或者其他，同样问题也存在析构函数，可能资源已经被释放了一部分了，这时候去调用很容易出现问题。

#### 代码样例对比

``` cpp
struct B {
  B() { seize(); }
  virtual ~B() { release(); }
protected:
  virtual void seize();
  virtual void release();
};

struct D : B {
  virtual ~D() = default;
protected:
  void seize() override {
    B::seize();
    // Get derived resources...
  }

  void release() override {
    // Release derived resources...
    B::release();
  }
};
```
以上代码基类中在析构函数和构造函数中就调用了虚函数，运行这段代码的结果就是，对于任何类型D的对象的构造和析构阶段都不会正确调用到seize()和release()。应该改成以下:

``` cpp
class B {
  void seize_mine();
  void release_mine();
public:
  B() { seize_mine(); }
  virtual ~B() { release_mine(); }
protected:
  virtual void seize() { seize_mine(); }
  virtual void release() { release_mine(); }
};

class D : public B {
  void seize_mine();
  void release_mine();
public:
  D() { seize_mine(); }
  virtual ~D() { release_mine(); }
protected:
  void seize() override {
    B::seize();
    seize_mine();
  }
  
  void release() override {
    release_mine();
    B::release();
  }
};
```
以上代码都没有调用虚函数了，而是自定义了一个seize_mine()和release_mine()函数。这样都能保证无论是基类对象和继承类对象都能正确释放自己的资源。

### OOP51-CPP. 不要切分继承类的对象

严重程度: 低。 导致信息被暴露，这造成程序不正常终止或拒绝服务攻击。

一个继承类的对象一般会包含一些对于它自身的额外成员变量来扩展基类。当通过赋值，值拷贝一个继承类的对象到一个它基类的对象，那么继承类对象多余的这部分信息就是丢失，因为基类对象没有足够的存储空间来容纳这部分数据。这种行为就是切分对象。

所以不要用继承类对象来初始化基类对象，除非用引用，指针或者类指针的抽象结构(智能指针等)。

#### 代码样例对比

``` cpp
#include <iostream>
#include <string>

class Employee {
  std::string name;
protected:
  virtual void print(std::ostream &os) const {
    os << "Employee: " << get_name() << std::endl;
  }
public:
  Employee(const std::string &name) : name(name) {}
  const std::string &get_name() const { return name; }
  friend std::ostream &operator<<(std::ostream &os, const Employee &e) {
    e.print(os);
    return os;
  }
};

class Manager : public Employee {
  Employee assistant;
protected:
  void print(std::ostream &os) const override {
    os << "Manager: " << get_name() << std::endl;
    os << "Assistant: " << std::endl << "\t"
       << get_assistant() << std::endl;
  }
public:
  Manager(const std::string &name, const Employee &assistant) :Employee(name), assistant(assistant) {}
  const Employee &get_assistant() const { return assistant; }
};

void f(Employee e) {
  std::cout << e;
}

int main() {
  Employee coder("Joe Smith");
  Employee typist("Bill Jones");
  Manager designer("Jane Doe", typist);
  f(coder);
  f(typist);
  f(designer);
}
```
以上代码在调用到f(designer)的时候出问题了，因为f()的参数是以值拷贝传递的，而Employee又是Manager的父类，所以，发生了对象截断，最后输出的时候，Manager的信息丢失了。

当然可以用指针:

``` cpp
// Remainder of code unchanged...
void f(const Employee *e) {
  if (e) {
    std::cout << *e;
  }
}

int main() {
  Employee coder("Joe Smith");
  Employee typist("Bill Jones");
  Manager designer("Jane Doe", typist);
  f(&coder);
  f(&typist);
  f(&designer);
}
```
以上代码还对指针做了判断，遵循了不要解引用空指针的规范。当然，还可以用引用:

``` cpp
// Remainder of code unchanged...
void f(const Employee &e) {
  std::cout << e; 
}

int main() {
  Employee coder("Joe Smith");
  Employee typist("Bill Jones");
  Manager designer("Jane Doe", typist);
  f(coder);
  f(typist);
  f(designer);
}
```

#### 代码样例对比

``` cpp
#include <iostream>
#include <string>
#include <vector>

void f(const std::vector<Employee> &v) {
  for (const auto &e : v) {
    std::cout << e;
  }
}

int main() {
  Employee typist("Joe Smith");
  std::vector<Employee> v {
                           typist,
                           Employee("Bill Jones"),
                           Manager("Jane Doe", typist)};
  f(v);
}
```
以上代码建立了一个Employee的向量类型，初始化的试图保存Manager的对象，所以会造成对象被截断。应该改成:

``` cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

void f(const std::vector<std::unique_ptr<Employee>> &v) {
  for (const auto &e : v) {
    std::cout << *e;
  }
}

int main() {
  std::vector<std::unique_ptr<Employee>> v;
  
  v.emplace_back(new Employee("Joe Smith"));
  v.emplace_back(new Employee("Bill Jones"));
  v.emplace_back(new Manager("Jane Doe", *v.front()));

  f(v);
}
```
以上代码改成了用std::unique_ptr来保存new出来的对象，这样防止了对象被截断。

### OOP52-CPP. 不要删除没有虚析构函数的多态对象

严重程度:低。直接导致未定义行为，实践上看，造成的结果是，程序不正常终止和内存资源泄漏。

C++标准[expr.delete]有如下声明:

> In the first alternative (delete object), if the static type of the object to be deleted is different from its dynamic type, the static type shall be a base class of the dynamic type of the object to be deleted and the static type shall have a virtual destructor or the behavior is undefined. In the second alternative (delete array) if the dynamic type of the object to be deleted differs from its static type, the behavior is undefined.

所以不要通过指针删除一个继承类的对象，该继承类的基类没有虚析构函数。正确的做法是，基类函数应该定义虚析构函数，不然会导致未定义行为。

#### 代码样例对比

``` cpp
struct Base {
  virtual void f();
};

struct Derived : Base {};

void f() {
  Base *b = new Derived();
  // ...
  delete b;
}
```
以上代码显然基类Base没有虚析构函数，违反了该原则。变量b是多态类型的指针，什么是多态，就是多种类型的表现形式。b指针具有静态类型Base *的类型信息，也有动态类型Derived *的类型的信息。所以，当delete b调用时，因为Base类没有虚析构函数，这将导致未定义行为。C++标准[class.dtor]有如下声明:

> If a class has no user-declared destructor, a destructor is implicitly declared as defaulted. An implicitly declared destructor is an inline public member of its class.

所以Base即使默认生成有析构函数，但是不是virtual的。没有虚析构函数的基类统统都是错的:

``` cpp
#include <memory>

struct Base {
  virtual void f();
};

struct Derived : Base {};

void f() {
  std::unique_ptr<Base> b = std::make_unique<Derived()>();
}
```
以上虽然用了智能指针自动释放资源，但是本质与用裸指针会导致同样的问题，基类的虚析构函数没有定义。所以必须改成以下:

``` cpp
struct Base {
  virtual ~Base() = default;
  virtual void f();
};

struct Derived : Base {};

void f() {
  Base *b = new Derived();
  // ...
  delete b;
}
```
以上代码就是well-defined behavior的了，显式声明基类的析构函数是虚函数，并且设置为默认的虚析构函数。

### OOP53-CPP. 用成员变量的声明顺序在构造函数的成员初始化列表中初始化成员变量

严重程度: 中等。

对于这个，C++标准[class.base.init]有如下声明：

> In a non-delegating constructor, initialization proceeds in the following order:

> • First, and only for the constructor of the most derived class, virtual base classes are initialized in the order they appear on a depth-first left-to-right traversal of the directed acyclic graph of base classes, where “left-to-right” is the order of appearance of the base classes in the derived class base-specifier-list.

> • Then, direct base classes are initialized in declaration order as they appear in the base-specifier-list (regardless of the order of the mem-initializers).

> • Then, non-static data members are initialized in the order they were declared in the class definition (again regardless of the order of the mem-initializers).

> • Finally, the compound-statement of the constructor body is executed.
[Note: The declaration order is mandated to ensure that base and member subobjects are destroyed in the reverse order of initialization. —end note]

如果不按照正确的顺序初始化列表，那么将直接导致未定义行为，比如读到了为初始化的内存。

#### 代码样例对比

``` cpp
class C {
  int dependsOnSomeVal;
  int someVal;
public:
  C(int val) : someVal(val), dependsOnSomeVal(someVal + 1) {}
};
```
以上的代码显然用初始化列表的形式没有按照成员变量的声明顺序初始化，这样可能会有无法预料的值出现，所以可以根据初始化列表的初始化顺序改变下声明顺序：

``` cpp
class C {
  int someVal;
  int dependsOnSomeVal;
public:
  C(int val) : someVal(val), dependsOnSomeVal(someVal + 1) {}
};
```

#### 代码样例对比

``` cpp
class B1 {
  int val;
public:
  B1(int val) : val(val) {}
};

class B2 {
  int otherVal;
public:
  B2(int otherVal) : otherVal(otherVal) {}
  int get_other_val() const { return otherVal; }
};

class D : B1, B2 {
public:
  D(int a) : B2(a), B1(get_other_val()) {}
};
```
以上代码用初始化列表调用父类的构造函数的时候顺序也不正确，因为在继承声明的时候，B1比B2类先初始化，这样会导致未定行为，可以改成以下代码：

``` cpp
class B1 {
  int val;
public:
  B1(int val) : val(val) {}
};

class B2 {
  int otherVal;
public:
  B2(int otherVal) : otherVal(otherVal) {}
};

class D : B1, B2 {
public:
  D(int a) : B1(a), B2(a) {}
};
```

### OOP54-CPP. 小心得处理自拷贝赋值

严重程度：低。 如果允许拷贝赋值操作符损坏对象，那么会导致未定义行为。虽然严重程度低，但是修复补救的代价高昂。

自拷贝赋值发生在各种各样的场景下，但是本质来说，所有的自拷贝赋值都是以下代码的衍生:

``` cpp
#include <utility>
struct S { /* ... */ }
void f() {
  S s;
  s = s; // Self-copy assignment
}
```
用户提供的拷贝操作符必须恰当的处理自拷贝赋值的情况。

#### 代码样例对比

``` cpp
#include <new>

struct S { /* ... */ }; // Has nonthrowing copy constructor

class T {
  int n;
  S *s1;
public:
  T(const T &rhs) : n(rhs.n), s1(rhs.s1 ? new S(*rhs.s1) : nullptr){}
  ~T() { delete s1; }
  // ...
  T& operator=(const T &rhs) {
    n = rhs.n;
    delete s1;
    s1 = new S(*rhs.s1);
    return *this;
  }
};
```
以上代码没有检测自拷贝的场景，如果自拷贝发生了，那么delete s1就把所指向的资源释放了，然后rhs.s1指向的是非法内存，对其解引用就是错误。应该改成下面：

``` cpp
#include <new>

struct S { /* ... */ }; // Has nonthrowing copy constructor

class T {
  int n;
  S *s1;
public:
  T(const T &rhs) : n(rhs.n), s1(rhs.s1 ? new S(*rhs.s1) : nullptr){}
  ~T() { delete s1; }
  // ...
  T& operator=(const T &rhs) {
   if (this != &rhs) {
       n = rhs.n;
       delete s1;
       try {
         s1 = new S(*rhs.s1);
       } catch (std::bad_alloc &) {
         s1 = nullptr; // For basic exception guarantees
         throw;
       }
   }
    return *this;
 }
};
```
当然也可以通过std::swap实现:

``` cpp
#include <new>

struct S { /* ... */ }; // Has nonthrowing copy constructor

class T {
  int n;
  S *s1;
public:
  T(const T &rhs) : n(rhs.n), s1(rhs.s1 ? new S(*rhs.s1) : nullptr){}
  ~T() { delete s1; }
  
  void swap(T &rhs) noexcept {
    std::swap(n, rhs.n);
    std::swap(s1, rhs.s1);
  }
  // ...
  T& operator=(const T &rhs) {
    T(rhs).swap(*this);
    return *this;
 }
};
```

###  OOP55-CPP. 不要使用pointer-to-member操作符访问不存在的成员

严重程度: 高。修复代价也很高昂。

pointer-to-member操作符就是.* 和 ->* 。一般是用来获取对象或者函数，通过函数方法获取对象封装的成员的。类似于下面,下面是的函数调用1和2等价的:

``` cpp
struct S {
  void f() {}
};

void func() {
  S o;
  void (S::*pm)() = &S::f;
  o.f(); // 1
  (o.*pm)(); // 2
}
```

C++标准[expr.mptr.oper]有如下声明:

> Abbreviating pm-expression.*cast-expression as E1.*E2, E1 is called the object
expression. If the dynamic type of E1 does not contain the member to which E2 refers, the behavior is undefined.

#### 代码样例对比

``` cpp
struct B {
  virtual ~B() = default;
};

struct D : B {
  virtual ~D() = default;
  virtual void g() { /* ... */ }
};

void f() {
  B *b = new B;
  // ...
  void (B::*gptr)() = static_cast<void(B::*)()>(&D::g);
  (b->*gptr)();
  delete b;
}
```
以上代码用pointer-to-member来获取D::g()，但是之后有转换成了B::* 。所以当调用这个动态类型为D的对象，pointer-to-member的调用是well-defined的。然而，动态类型的底层对象是B，会导致未定义行为。应该改成以下：

``` cpp
struct B {
  virtual ~B() = default;
};

struct D : B {
  virtual ~D() = default;
  virtual void g() { /* ... */ }
};

void f() {
  B *b = new D; // Corrected the dynamic object type.
  // ...
  void (D::*gptr)() = &D::g; // Moved static_cast to the next line.
  (static_cast<D *>(b)->*gpter)();
  delete b;
}
```

#### 代码样例对比

``` cpp
struct B {
  virtual ~B() = default;
};

struct D : B {
  virtual ~D() = default;
  virtual void g() { /* ... */ }
};

static void (D::*gptr)(); // Not explicitly initialized, defaults to nullptr.

void call_memptr(D *ptr) {
  (ptr->*gptr)();
}

void f() {
  D *d = new D;
  call_memptr(d);
  delete d;
}
```
以上代码由于没有显式初始化pointer-to-member对象，所以默认是nullptr，对其解引用直接导致未定义行为。所以要初始化:

``` cpp
struct B {
  virtual ~B() = default;
};

struct D : B {
  virtual ~D() = default;
  virtual void g() { /* ... */ }
};

static void (D::*gptr)() = &D::g; //显式初始化

void call_memptr(D *ptr) {
  (ptr->*gptr)();
}

void f() {
  D *d = new D;
  call_memptr(d);
  delete d;
}
```

### OOP56-CPP. 遵守handler的替换规范

严重程度: 低。如果违反替换规范会导致未定义行为。

C++标准[handler.fuunctions]指出new_handler, terminate_handler和unexpected_handler可以用自定义的实现全局替换它们。

比如一个程序可以通过调用std::set_terminate()函数来设置一个自定义的terminate handler，这个handler中可以在程序终止前打印点必要的调试信息等。

然而，C++标准[res.on.function]第一段做出了如下声明:

>  However, the C++ Standard, [res.on.functions], paragraph 1, states the following: In certain cases (replacement functions, handler functions, operations on types used to instantiate standard library template components), the C++ standard library depends on components supplied by a C++ program. If these components do not meet their requirements, the Standard places no requirements on the implementation.
 
第二段作出了更详细的声明:

> In particular, the effects are undefined in the following cases:

> • for handler functions, if the installed handler function does not implement the semantics of the applicable Required behavior: paragraph

也就是说，替换任何handler函数必须要遵循相关的语义规范原则。

以下是三个handler函数的对应的替换规范原则:

- New Handler

在C++标准[new.handler]是这样指定的:

> Required behavior: A new_handler shall perform one of the following:

> • make more storage available for allocation and then return;

> • throw an exception of type bad_alloc or a class derived from bad_alloc;
> • terminate execution of the program without returning to the caller; 

- Terminate Handler

在C++标准[terminate.handler]是这样指定的:

> Required behavior: A terminate_handler shall terminate execution of the program
without returning to the caller.

- Unexpected Handler

在C++标准[new.handler]是这样指定的:

> Required behavior: An unexpected_handler shall not return.

当然，就现在来说，unexpected_handler已经是C++弃用的功能了，不推荐使用。

#### 代码样例对比

``` cpp
#include <new>

void custom_new_handler() {
  // Returns number of bytes freed.
  extern std::size_t reclaim_resources();
  reclaim_resources();
}

int main() {
  std::set_new_handler(custom_new_handler);
  // ...
}
```
以上代码显然没有遵守New Handler规范的任何一项，没有考虑内存不足时候的失败情况，抛出bad_alloc或者直接终止程序的代码，应该改成以下:

``` cpp
#include <new>

void custom_new_handler() {
  // Returns number of bytes freed.
  extern std::size_t reclaim_resources();
  if (0 == reclaim_resources()) {
    throw std::bad_alloc();
  }
}

int main() {
  std::set_new_handler(custom_new_handler);
  // ...
}
```

### OOP57-CPP. 优先选择特定的成员函数来代替C标准库提供的重载操作符的函数

严重程度: 高。修复的代价也是极高。对于大多数违反此条款的行为会导致程序非正常行为.然而，覆盖了对象的内存布局会导致恶意代码的运行。

一些C标准库函数会在对象上执行Byte级别宽度的操作。比如，std::memcmp就是以字节来比较两个对象，std::memcpy就是以字节级别把对象的内存字节表示拷贝到目标缓冲区。然而，对于一些内存空间非平坦的对象类型就会导致程序非正常行为。

C++标准[class]有如下声明:

> A trivially copyable class is a class that:

> • has no non-trivial copy constructors,

> • has no non-trivial move constructors,

> • has no non-trivial copy assignment operators,

> • has no non-trivial move assignment operators, and

> • has a trivial destructor.

> A trivial class is a class that has a default constructor, has no non-trivial default constructors, and is trivially copyable. [Note: In particular, a trivially copyable or trivial class does not have virtual functions or virtual base classes. — end note]

另外，C++标准[class]还有如下声明:

> A standard-layout class is a class that:

> • has no non-static data members of type non-standard-layout class (or array of such types) or reference,

> • has no virtual functions and no virtual base classes,

> • has the same access control for all non-static data members,

> • has no non-standard-layout base classes,

> • either has no non-static data members in the most derived class and at most one base class with non-static data members, or has no base classes with non-static data members, and

> • has no base classes of the same type as the first non-static data member.

不要使用std::memset, std::memcpy来操作non-trivial的类对象,不要使用std::memcmp来操作非标准布局的类对象。对于这些情况，C++提供了对应的解决方案:

| C标准库函数                           | C++对应的等价功能 | 
| :-------------:                      |:-------------:|  
| std::memset                          | 类构造函数     |
| std::memcpy std::memmove std::strcpy | 类拷贝构造函数或operator =() 重载 |  
| std::memcmp std::strcmp              | operator<(), operator>(), operator==(), 或 operator!=() 重载| 

#### 代码样例对比

``` cpp
#include <cstring>
#include <iostream>

class C {
  int scalingFactor;
  int otherData;
public:
  C() : scalingFactor(1) {}
  void set_other_data(int i);
  int f(int i) {
    return i / scalingFactor;
  }
  // ...
};

void f() {
  C c;
  // ... Code that mutates c ...
  // Reinitialize c to its default state
  std::memset(&c, 0, sizeof(C));
  std::cout << c.f(100) << std::endl;
}
```
以上代码试图通过memset设置一段内存清0的方法初始化对象c,而对象c是一个non-trivial的类对象，本来默认的构造函数初始化成员scalingFactor为1的，后来又被memset清空为0了，所以最终调用f(100)也是非预料的结果。程序出现非正常行为。当然，以上代码也违反了EXP62-CPP规范。

所以应该改成以下:

``` cpp
#include <cstring>
#include <iostream>

class C {
  int scalingFactor;
  int otherData;
public:
  C() : scalingFactor(1) {}
  void set_other_data(int i);
  int f(int i) {
    return i / scalingFactor;
  }
  // ...
};

template <typename T>
T& clear(T &o) {
  using std::swap;
  T empty;
  swap(o, empty);
  return o;
}

void f() {
  C c;
  // ... Code that mutates c ...
  // Reinitialize c to its default state
  clear(c);
  std::cout << c.f(100) << std::endl;
}
```
以上代码通过原对象与一个新的合法初始状态的空对象交换来达到不破坏原对象内存字节布局重新初始化对象状态的目的。安全也高效。

#### 代码样例对比

``` cpp
#include <cstring>

class C {
  int *i;
public:
  C() : i(nullptr) {}
  ~C() { delete i; }
  
  void set(int val) {
    if (i) { delete i; }
    i = new int{val};
  } 
  // ...
};

void f(C &c1) {
  C c2;
  std::memcpy(&c2, &c1, sizeof(C));
}
```
以上代码通过std::memcpy把c1对象内部的状态拷贝的对象c2中。但是，很显然，这个C类是non-trivial的,内部包含的指针i指向同一个对象，所以当c1和c2对象都销毁时，会造成指针i指向的资源释放两次甚至多次。应该自定义一个赋值运算符来改写代码:

``` cpp
class C {
  int *i;
public:
  C() : i(nullptr) {}
  ~C() { delete i; }
  
  void set(int val) {
    if (i) { delete i; }
    i = new int{val};
  }
  
  C &operator=(const C &rhs) noexcept(false) {
    if (this != &rhs) {
      int *o = nullptr;
      if (rhs.i) {
        o = new int;
        *o = *rhs.i;
      }
      // Does not modify this unless allocation succeeds.
      delete i;
      i = o;
    }
    return *this;
  }
  // ...
};

void f(C &c1) {
  C c2 = c1;
}
```

#### 代码样例对比

``` cpp
#include <cstring>
class C {
  int i;
public:
  virtual void f();
  // ...
};

void f(C &c1, C &c2) {
  if (!std::memcmp(&c1, &c2, sizeof(C))) {
    // ...
  }
}
```
以上代码类C，有虚函数，显然是非标准布局的类，然后还用std::memcmp比较，这是错的, 比较在某些情况下会不成功，特别是类还有继承关系的时候，应该改成下面:

``` cpp
class C {
  int i;
public:
  virtual void f();
  bool operator==(const C &rhs) const {
    return rhs.i == i;
  }
  // ...
};

void f(C &c1, C &c2) {
  if (c1 == c2) {
    // ...
  }
}
```

### OOP58-CPP. 对象的拷贝操作必须不能修改源对象（source object）

严重程度: 低。 

拷贝操作（包括拷贝构造函数和拷贝赋值运算符）一般是期望拷贝源对象的显著属性到目标对象中,换句话说的结果就是，目标对象是原始对象的一个副本。

理想情况下，拷贝操作需要用惯用的符号标记,比如对于拷贝构造函数的声明，必须是这样T(const T &)，意为源对象不可修改。对于拷贝赋值运算符的声明必须这样T& operator=(const T&)。所以拷贝操作必须遵循惯用法,不然就违反了C++标准中CopyConstructible或CopyAssignable的概念了。

在C++ 11之前，如果一个拷贝操作修改了源对象，那么仅仅是为了实现类似move语义的唯一方式，然而那时候语言并没有提供一种强制的方式来实现这样的方案，所以就导致了脆弱的API，其中一个例子就是std::auto_ptr。但是在C++ 11和之后的标准中，这个场景有更加合适的代替者了，直接用move操作替换拷贝操作。

在C++ 11中，std::unique_ptr在实现所有权语义方面是比std::auto_ptr更好的代替方案。std::unique_ptr显式删除了拷贝构造函数和拷贝赋值运算，与之代替的是移动构造函数和移动赋值运算符。最终std::auto_ptr在C++ 11标准中被废弃了。

#### 代码样例对比

``` cpp
#include <algorithm>
#include <vector>

class A {
  mutable int m;
public:
  A() : m(0) {}
  
  explicit A(int m) : m(m) {}
  
  A(const A &other) : m(other.m) {
    other.m = 0;
  }
  
  A& operator=(const A &other) {
    if (&other != this) {
      m = other.m;
      other.m = 0;
    }
    return *this;
  }
  
  int get_m() const { return m; }
};

void f() {
  std::vector<A> v{10};
  A obj(12);
  std::fill(v.begin(), v.end(), obj);
}
```
以上代码中，在拷贝构造函数里面修改了成员变量m，因为其声明是mutable的，所以f()函数中的结果不是所预料中的，原本是想把std::vector中的10个A类型的对象全部替换成obj对象的状态，结果不是，vector中的A类对象除了第一个对象其余全部都还是为0，因为调用了其拷贝构造函数，把源对象状态m设置为0了，其余9次重复调用源对象的拷贝构造函数，全部成为0了。要改成以下:

``` cpp
#include <algorithm>
#include <vector>

class A {
  int m;
public:
  A() : m(0) {}
  
  explicit A(int m) : m(m) {}
  
  A(const A &other) : m(other.m) {}
  
  A(A &&other) : m(other.m) { other.m = 0; }
  
  A& operator=(const A &other) {
    if (&other != this) {
      m = other.m;
    }
    return *this;
  }
  
  A& operator=(A &&other) {
    m = other.m;
    other.m = 0;
    return *this;
  }
  int get_m() const { return m; }
};

void f() {
  std::vector<A> v{10};
  A obj(12);
  std::fill(v.begin(), v.end(), obj);
}
```