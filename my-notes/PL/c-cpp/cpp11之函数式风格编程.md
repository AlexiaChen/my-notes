> *C++是多范式编程语言，程序员可以选择多重范式的结合来编程，比如面向对象，面向过程，泛型编程，函数式特性。尤其是C++ 11标准中完善了函数式的一些特性，比如，lambda表达式，变参模版，新的STL函数bind。下面就讲讲怎样用C++ 11来写函数式风格的代码。*

# 函数式风格编程 

- 自动类型推导，<font color="red">auto</font>和<font color="red">decltype</font>

- lambda表达式的支持, 闭包，函数即数据

- 偏函数应用（partial funtion application），<font color="red">std::function</font>和<font color="red">std::bind</font>,lambda表达式和auto

- STL与高阶函数的组合

- 用变参模版(variadic templates)来进行列表生成(List manipulation)

- 用模版全特化(full  template 
specialisation)和偏特化(partial template 
specialisation)来进行模式匹配(pattern matching)

- 用<font color="red">std::async</font>来进行惰性求值

# 为什么要以函数式风格来编程

### STL结合lambda表达式更加简洁高效

``` cpp

std::accumulate(std::begin(vec),std::end(vec),                            
         [](int a,int b){return a+b;});
```

### 模式匹配之模版元编程

``` cpp
// 编译时计算阶乘
template <int N>
struct Fac{ static int const val= N * Fac<N-1>::val; };
template <>
struct Fac<0>{ static int const val= 1; };
```

### 还有更好更简洁的编程风格

``` cpp
// 使用range for
for (auto v: vec) std::cout << v << std::endl;
```

# 什么是函数式编程

- 函数式编程就是以数学中的函数概念进行编程

- 数学中的函数在给定同样参数的情况下，每次调用函数的返回值都相同，不会有副作用(side-effect)

- 正因为函数调用没有副作用，那么函数从语义上相当于结果（Result）,是等价的，可以互相替换

- 因为无副作用的特性，编译器的优化器可以从更高层次来优化，增加优化潜力，重新组合函数的调用顺序，或者把不同的函数调用放到不同的线程中并行计算，提高并发可能性

- 程序的控制流就是以数据依赖驱动的了，而不是指令顺序

# 函数式编程的特点

- 惰性求值(Lazy evaluation)
- 一等函数（first-class function）
- 高阶函数(higher-order function)
- 纯函数（pure function）  (这个C++不是纯函数，只有Haskell是纯函数)
- 递归
- 列表生成（manipulation of list）

# 一等函数

- 一等函数属于一等公民，与数据地位相同，函数即是数据
- 函数可以当作参数来传递
- 函数可以当作值一样被其他函数返回
- 函数能被存储在变量或者数据结构中

其实C语言也有类似的特性，函数指针，这样就有更强的表现能力，也明面上支持了回调。C++ 11直接支持lambda表达式了。就不需要那么复杂和不安全的函数指针了。

# 一等函数表达能力之分发表（Dispatch Table）

``` cpp
std::map<const char,std::function<double(double,double)>>  tab;

tab.insert(std::make_pair('+',[](double a,double b){return a + b;}));
tab.insert(std::make_pair('-',[](double a,double b){return a - b;}));
tab.insert(std::make_pair('*',[](double a,double b){return a * b;}));
tab.insert(std::make_pair('/',[](double a,double b){return a / b;}));

std::cout << "3.5+4.5= " << tab['+'](3.5,4.5) << std::endl;  //8
std::cout << "3.5*4.5= " << tab['*'](3.5,4.5) << std::endl;  // 15.75

tab.insert(std::make_pair('^',                                        
                  [](double a,double b){return pow(a,b);}));

std::cout << "3.5^4.5= " << tab['^'](3.5,4.5) << std::endl;  // 280.741
```

# 高阶函数

- 高阶函数是一种接受其他函数作为参数或者是返回其他函数的一种函数

说到这里，可能有人懵了，一等函数和高阶函数有啥不一样？请看以下代码块就知道了：

``` javascript
function make_add(){
    var f = function (a, b){ return a + b;}; 
    return f;
}
```

以上代码，函数make_add就是高阶函数，匿名函数function(a,b)可以被bind到变量f上，或者本身也可以直接返回，所以这匿名函数就是一等函数。

所以，高阶函数用朴素的观点来理解的话，可以这么说：<font color="red">高阶函数是一等函数的应用</font>

很多函数式编程语言，有三种经典的高阶函数应用直接体现：

- map , 应用一个函数到列表的每一个元素（element）上

- filter, 从一个列表中移除一些元素

- fold， 通过二分操作分解一个列表，直到一个列表变成单值(a single value)

很多编程语言已经有以上三种语义的支持了：

![](http://wx4.sinaimg.cn/large/a1ac93f3gy1fd9gqk9pduj20jx02yweu.jpg)


这三种语义有无数经典的应用场景：


 Map + Reduce = [MapReduce](https://en.wikipedia.org/wiki/MapReduce)

  
Haskell
``` haskell
vec = [1 . . 9]
str = ["Programming","in","a","functional","style."]

map(\a → a^2) vec    -- [1,4,9,16,25,36,49,64,81]
map(\a -> length a) str -- [11,2,1,10,6]

filter(\x-> x<3 || x>8) vec  -- [1,2,9] 
filter(\x → isUpper(head x)) str -- [“Programming”]

foldl (\a b → a * b) 1 vec   -- 362800
foldl (\a b → a ++ ":" ++ b ) "" str -- “:Programming:in:a:functional:style.”
```

  Python
``` python 
vec = range(1,10)
str = ["Programming","in","a","functional","style."]

map(lambda x :  x*x , vec) # [1,4,9,16,25,36,49,64,81]
map(lambda x : len(x), str) # [11,2,1,10,6]

filter(lambda x: x<3 or x>8 , vec) # [1,2,9]
filter(lambda x: x[0].isupper(),str) # [“Programming”]

reduce(lambda a , b: a * b, vec, 1) # 362800
reduce(lambda a, b: a + b, str,"") # “:Programming:in:a:functional:style.”
```

  C++ 11
``` cpp
std::vector<int> vec{1,2,3,4,5,6,7,8,9}
std::vector<std::string> str{"Programming","in","a","functional",       
                "style."}

std::transform(vec.begin(),vec.end(),vec.begin(),                 
        [](int i){ return i*i; }); // [1,4,9,16,25,36,49,64,81]
std::transform(str.begin(),str.end(),back_inserter(vec2),         
        [](std::string s){ return s.length(); }); // [11,2,1,10,6]

// [1,2,9]
vec.erase(std::remove_if(vec.begin(),vec.end(),
[](int i){ return !((i < 3) or (i > 8)) }),vec.end());
// [“Programming”]
str.erase(std::remove_if(str.begin(),str.end(),                   
        [](string s){ return !(isupper(s[0])); }),str.end());

std::accumulate(vec.begin(),vec.end(),1,                          
         [](int a, int b){ return a*b; }); // 362800

//“:Programming:in:a:functional:style.” 
std::accumulate(str.begin(),str.end(),string(""),                 
         [](string a,string b){ return a+":"+b; });      
```

# 纯函数 

- 纯函数VS不纯的函数（下图来自于[《Real World Haskell》](https://book.douban.com/subject/3134515/)）:

![](http://wx3.sinaimg.cn/large/a1ac93f3gy1fd9in6lmasj20j6048q3u.jpg)

- 纯函数是独立的，这样让程序验证更加容易，也更加利于重构,测试，维护

- 非常有利于优化，比如，保存函数的调用结果，并行化

- 单子（Monad），是Haskell处理“不纯世界”的一种解决方案,它封装了不纯的世界，也是Haskell中的一个很重要的子系统，它也是表示计算的一种结构，甚至可以定义一组计算的组成结构

# 递归

C++ 11
``` cpp
// 编译时计算阶乘
template <int N>
struct Fac{ static int const val= N * Fac<N-1>::val; };
template <>
struct Fac<0>{ static int const val= 1; };

// 计算过程
Fac<3>::value = 3 * Fac<2>::value
              = 3 * 2 * Fac<1>::value
              = 3 * 2 * 1 * Fac<0>::value
              = 3 * 2 * 1 * 1
              = 6
```

- 递归是一种控制结构
- 递归和列表处理结合在函数式语言中是一种强力的模式

同样的计算阶乘方式，我们用Haskell的模式匹配来实现以下：
``` haskell
fac 0 = 1
fac n = n * fac (n-1)
```

# 列表处理 

- 处理列表头head(x)
- 递归处理到列表尾tail(xs)

比如：[1,2,3,4,5]  head(x) = [1] tail(xs) = [2,3,4,5]

那么用模式匹配+递归写出如下Haskell代码:

``` haskell
-- 求和
mySum []     = 0
mySum (x:xs) = x + mySum xs
mySum [1,2,3,4,5]    -- 15

-- 计算过程
mySum [1,2,3,4,5] = 1 + mySum [2,3,4,5]
                  = 1 + 2 mySum [3,4,5]
                  = 1 + 2 + 3 mySum [4, 5]
                  = 1 + 2 + 3 + 4 mySum [5]
                  = 1 + 2 + 3 + 4 + 5 + mySum []
                  = 1 + 2 + 3 + 4 + 5 + 0
                  = 15

-- 实现map语义
myMap f [] = []
myMap f (x:xs)= f x: myMap f xs  
myMap (\x → x*x)[1,2,3] -- [1,4,9]

-- 计算过程
myMap (\x → x*x)[1,2,3] 
=> (\x -> x*x) 1 : myMap (\x -> x*x) [2,3]
=> 1 : (\x -> x*x) 2 : myMap (\x -> x*x) [3]
=> 1 : 4 : (\x -> x*x) 3 : myMap (\x -> x*x) []
=> 1 : 4 : 9 : []
=> [1,4,9]

  
```

下面用C++ 11 的变参模版来做以上的求和：

``` cpp
template<int ...> struct mySum;
template<>struct
mySum<>{
static const int value = 0;
};

template<int head, int ... tail> struct
mySum<head,tail...>{
static const int value= head + mySum<tail...>::value;
};

int sum = mySum<1,2,3,4,5>::value;  // 15

//计算过程
mySum<1,2,3,4,5>::value
=> 1 + mySum<2,3,4,5>::value
=> 1 + 2 + mySum<3,4,5>::value
=> 1 + 2 + 3 + mySum<4,5>::value
=> 1 + 2 + 3 + 4 + mySum<5>::value
=> 1 + 2 + 3 + 4 + 5 + mySum<>::value
=> 1 + 2 + 3 + 4 + 5 + 0
=> 15
```

其实呢，归而结网，列表处理的核心思想就是模式匹配。

再举个例子，用Haskell的模式匹配写一个乘法函数：

``` haskell
mult n 0 = 0
mult n m = (mult n (m – 1)) + n

-- 6
mult 3 2

--计算过程
mult 3 2
=> (mult 3 (2 - 1)) + 3
=> (mult 3 1) + 3
=> (mult 3 (1 - 1)) + 3 + 3
=> (mult 3 0) + 3 + 3
=> 0 + 3 + 3
=> 6
```

用C++ 11的模版元编程来写乘法：

``` cpp
template < int N1, int N2 > 
class Mult { 
  static const int value =  Mult<N1, N2 - 1>::value + N1; 
};

template < int N1 > 
class Mult <N1,0> { static const int value = 0; };

// 6
int result = Mult<3,2>::value;

//计算过程
Mult<3,2>::value
=> Mult<3,1>::value + 3
=> Mult<3,0>::value + 3 + 3
=> 0 + 3 + 3
=> 6
```

# 惰性求值 

- 简单说，就是到需要的时候再计算，不需要的时候不必计算
- 优势是，节省计算时间，节省内存使用。还有可以和无限数据结构结合(infinite data structures)

 EOF
