#cpp

这篇文章，我们来一步一步通过C++的来构造闭包。当然，在没有GC的语言中，闭包的特性属于半残，不是很完美。不过虽然不完美，但是还是能构造一些高阶的表达方式的。这篇文章不会讲解闭包的概念。

首先我们构造一个可调用lambda函数的一个函数,注意这时候是不支持返回值的：

```cpp
void call(const std::function<void()>& fn)
{
    try
    {
        fn();
    }
    catch(const std::exception& e)
    {
        // handle exception here
    }
}

void print_line(const std::string& text)
{
    std::cout << text << '\n';
}

call([](){ print_line("called.") });

```

当然，我们还可以改造成模板的形式，本质是一样的：

```cpp

template <typename Callable>
void call(const Callable& fn)
{
    fn();
}

```

但是，以上两种方式都没有返回值，如果要支持返回值必须每个返回值类型都指定，比如：

```cpp
double call(const std::function<double()>& fn)
{
    return fn();
}

double sum(double a, double b)
{
    return a + b;
}

double result = call([](){ return sum(3.14, 2.71); });

```

这样太不灵活通用了，非常麻烦。其实我们可以通过闭包来巧妙地来达到我们需要的目的：

```cpp

void callImpl(const std::function<void()>& fn)
{
    try
    {
        fn();
    }
    catch(const std::exception& e)
    {
        // handle exception here
    }
}

template <typename Fn>
auto call(const Fn& fn) -> decltype(fn())
{
    decltype(fn()) result{};
    auto wrapperFn = [&]() -> void {
        result = fn();
    };

    callImpl(wrapperFn);
    return result;
}

double sum(double a, double b)
{
    return a + b;
}


double result = call([](){ return sum(3.14, 2.71); });

```

这样lambda函数的返回值类型就不用显式指定了。

但是大家注意到没，到现在虽然call模板函数不用显式指定了，但是还是有返回值的，毕竟最后有一个return result的语句，也就是说，对于没返回值的lambda函数，该模板call函数不支持，还有不完善的地方。导致编译错误。

```cpp
call([](){ print_line("called."); }); // compile error
```

当然，通过C++的模板元编程，我还有可以有方法来解决这个不完善的地方：

```cpp

void callImpl(const std::function<void()>& fn)
{
    try
    {
        fn();
    }
    catch(const std::exception& e)
    {
        // handle exception here
    }
}

template <typename Fn>
auto call(const Fn& fn) -> decltype(fn())
{
   return _call(fn, std::is_same<decltype(fn()), void>());
}

template <typename Fn>
void _call(const Fn& fn, std::true_type)
{
    callImpl([&](){ fn(); });
}

template <typename Fn>
auto _call(const Fn& fn, std::false_type) -> decltype(fn())
{
    decltype(fn()) result{};
    callImpl([&]() { result = fn(); });
    return result;
}

auto result = call([]() { return sum(1,2); }); // return int
auto result = call([]() { return sum(1.2,2.3); }); // return double
call([]() { print_line("called"); }); // return void

```

以上代码就很完美了，几乎无懈可击。

EOF
