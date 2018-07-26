---
layout: post
title: 记一次C++静态反射的尝试
categories: moderncpp
---

## 1. OBJECTIVE
实现目标，能够获取一个用户自定义类型的成员变量和成员方法，包含静态和非静态的。实现方式有两种，侵入式和非侵入式。侵入式的方案，可以比较好的处理保护类型和私有类型的成员，但是面对无法修改源码的类库却束手无策。非侵入式的实现不需要修改用户自定义类型，可以很好的处理外部的类库，但是又无法处理对非公有成员。两种方案各有优劣，笔者个人是倾向于非侵入式的方案。毕竟，即使反射出了非公有的成员，我们也无法访问封装好的非公有成员。

量化具体的目标，详细如下：
1. 访问类名；
2. 访问公有的非静态成员变量；
3. 访问公有的非静态成员函数；
4. 访问公有的静态成员变量；
5. 访问公有的静态成员函数；
6. 访问基类；

用代码接口来更详细的表达具体目标：
```cpp
class foo
{
public:
    // constructor
    foo() : var_a_(0) {};
    foo(int a) : var_a_(a) {}

    // non-static method
    int get() const { return var_a_; }
    void set(int a) { var_a_ = a; }

public:

    // static data member
    static double static_var;

    // static method
    static void set_svar(double b) { static_var = b; }

public:
    // non-static data member
    int var_a_;
};

double foo::static_var = 0;
```
* 类名：命名空间::类名，这里就是foo或者::foo;
* 非静态成员变量：使用pointer to memeber data(PMD)的元组,这里是

```cpp 
std::tuple<int foo::*>;
```
* 非静态成员函数：使用pointer to member function(PMF)的元组，这里是：
```cpp
std::tuple<int(foo::*)(void) const, void(foo::*)(int)>;
```
* 静态成员变量：使用指针类型的元组即可，使用时同样要注意一处定义原则(ODR)：
```cpp
std::tuple<double*>;
```
* 静态成员函数：使用函数指针的元组即可：
```cpp
std::tuple<void(double)>;
```
* 基类：导出所有基类的类型，单继承导出一个类型，多继承导出基类的元组。

## 2. MANUAL VERSION
为了降低实现的难度，笔者在尝试的时候是先写出手动实现的版本。最终，这些手写的代码会使用宏来替代。最终宏需要生成的版本，大概是这样的：
```cpp
namespace iguana
{
    template <typename T>
    struct meta_info;
}
```
上面的代码是反射库提供的基础外观，针对需要反射的类型，就要生成对应的特化代码。
```cpp
// in global namespace
template <>
struct ::iguana::meta_info<foo>
{
    // base class
    using base_class = void;
    // class name
    static constexpr std::string_view name() noexcept
    {
        using namespace std::string_literals;
        return "foo"sv;
    }
    // pointer to member data
    static constexpr auto mdata_names() noexcept
        -> std::array<std::string_view, 1>
    {
        using namespace std::string_literals;
        return {"var_a_"sv};
    }
    static constexpr auto mdata() noexcept
    {
        return std::make_tuple(&foo::var_a_);
    }
    // pointer to static data
    static constexpr auto sdata_names() noexcept
        -> std::array<std::string_view, 1>
    {
        using namespace std::string_literals;
        return {"static_var"sv};
    }
    static constexpr auto sdata() noexcept
    {
        return std::make_tuple(&foo::static_var);
    }
    // pointer to member function
    static constexpr auto mfunc_names() noexcept
        -> std::array<std::string_view, 2>
    {
        using namespace std::string_literals;
        return {"get"sv, "set"sv};
    }
    static constexpr auto mfunc() noexcept
    {
        return std::make_tuple(&foo::get, &foo::set);
    }
    // pointer to static function
    static constexpr auto sfunc_names() noexcept
        -> std::array<std::string_view, 1>
    {
        using namespace std::string_literals;
        return {"set_svar"sv};
    }
    static constexpr auto mfunc() noexcept
    {
        return std::make_tuple(&foo::set_svar);
    }
};
```
## 3. SEMI-AUTOMAIC
手写这些代码很啰嗦也很不高效，但是宏可以缩减这些代码。我自己用宏实现的半自动版本，虽然看起来也很啰嗦，如下：
```cpp
IGUANA_REFLECT(
    foo,
    IGUANA_CTOR((), (int))
    IGUANA_MDATA(var_a_)
    IGUANA_MFUNC(set, get)
    IGUANA_SDATA(static_var)
    IGUANA_SFUNC(set_svar)
);
```
在实践的过程中，我碰到的最大的挑战就是构造函数和函数重载。构造对象是一种语义，所以构造函数是无法获取地址的，也可以认为构造函数不是函数，详细可以参考**Inside C++ object model**中的The Semantics of Constructors一章。其次就是函数重载，因为对重载函数取地址，编译期会抱怨不知道取的是哪一个？

构造函数的解决方案，我用了一个元组保存构造函数的参数列表，然后再用一个元组记录所有的构造构造函数。
```cpp
template <>
struct ::iguana::meta_info<foo>
{
    // ...
    using ctor_list = std::tuple<
        std::tuple<void>,
        std::tuple<int>
    >;
    // ...
}
```
有一个细节需要注意，对于非默认构造函数，需要检查参数的类型，不能有void. 如果有需要用static_assert报编译期错误。接下来就是函数重载了，如果给foo增加一个成员函数get的重载版本：
```cpp
class foo
{
public:
    // ...
    int get(int to_add) const
    {
        return var_a_ + to_add;
    }
    // ...
};
```
就需要显示的把每个get的signature都写出来：
```cpp
IGUANA_REFLECT(
    foo,
    // ...
    IGUANA_MFUNC(
        set, 
        (get, int(void) const),
        (get, int(int) const)
    )
    // ...
);
```
## 4. IMPLEMETATION DETAILS
实现的时候，我使用了boost.pp库。宏元编程是一个很烧脑的事情，不过有了boost.pp情况会好很多，但依然很烧脑。介绍思路以前，先简介一下boost.pp的抽象的三种基于宏token的数据结构：sequence, tuple和list.

```cpp
(a)(b)(c)                       // This is sequence
(a, b, c)                       // This is tuple
(a, (b, (c, BOOST_PP_NIL)))     // This is list
``` 
这里需要注意的是，PP使用逗号","作为tokens的划分，所以一定要处理好C++ templates. 

我在实现中使用了宏元的tuple和sequence. IGUANA_REFLECT(...)
展开后，实际上是如下形式：
```cpp
IGUANA_REFLECT(
    foo,
    ((IGUANA_CTOR_PROCESS, ((), (int)))
    ((IGUANA_MDATA_PROCESS, (var_a_))
    ((IGUANA_MFUNC_PROCESS, (set, get)))
    ((IGUANA_SDATA_PROCESS, (static_var)))
    ((IGUANA_SFUNC_PROCESS, (set_svar)))
)
```
简单解析一下，IGUANA_REFLECT宏的输入只有两个tokens。第一个token就是foo类，除了生成类名，稍后会用作sequence遍历算法的上下文。而第二个token是一个sequence，每个sequence内部是一个tuple.
```cpp
((tuple1))((tuple2))((tuple3))((tuple4))((tuple5))
```
每个tuple都是一个二元的元组，展开的结果如下：
```cpp
(IGUANA_CTOR_PROCESS, ((), (int)) )
(IGUANA_MDATA_PROCESS, (var_a_) )
(IGUANA_MFUNC_PROCESS, (set, get) )
(IGUANA_SDATA_PROCESS, (static_var) )
(IGUANA_SFUNC_PROCESS, (set_svar) )
```
tuple的第二个token依旧是一个tuple，例如((), (int))和(set, get). 最外层的IGUANA_REFLEECT实际上使用了sequence的for_each算法，对sequence的每个元素进行展开：
```cpp
#define IGUANA_REFLECT(CLASS, seq)
template<> struct iguana::reflect_info<CLASS> {\
// ...\
BOOST_PP_SEQ_FOR_EACH(IGUANA_APPLY_PROCESS, CLASS, seq)\
};
```
展开之后的效果大概是这样的：
```cpp
template<> struct iguana::reflect_info<foo> {
    // ...
    IGUANA_CTOR_PROCESS(foo, (), (int))
    IGUANA_MDATA_PROCESS(foo, var_a_)
    IGUANA_MFUNC_PROCESS(foo, set, get)
    IGUANA_SDATA_PROCESS(foo, static_var)
    IGUANA_SFUNC_PROCESS(foo, set_svar)
};
```
每个IGUANA_XXX_PROCESS宏，负责实际生成获取成员名和获取成员指针的代码。这一部分的代码是基于iguana之前的实现改进的，这里就不赘述了，详细的代码可以参照[此链接](https://github.com/Madokakaroto/Kathedrale-Lazengann/blob/master/include/kath/reflection.hpp)。我暂时还没有上传示例代码，后面会添加上，更新在文章中。
## 5. TRAVERSE
反射代码是可以半自动生成了，但是使用起来不是很方便。所以我试着提供了一个visitor的机制，可以比较方便的访问：
```cpp
struct foo_visitor
{
    template <typename Ptr, typename Arr>
    void visit_sdata(std::string_view const& name, 
        Ptr ptr, size_t index, Arr const& names) const
    {
        std::cout << name << index << std::endl;
    }

    template <typename PMD, typename Arr>
    void visit_mdata(std::string_view const& name, 
        PMD pmd, size_t index, Arr const& names) const
    {
        std::cout << name << index << std::endl;
    }

    template <typename FPtr, typename Arr>
    void visit_sfunc(std::string_view const& name, 
        FPtr ptr, size_t index, Arr const& names) const
    {
        std::cout << name << index << std::endl;
    }

    template <typename PMF, typename Arr>
    void visit_mfunc(std::string_view const& name, 
        PMF pmf, size_t index, Arr const& names) const
    {
        std::cout << name << index << std::endl;
    }
};

// visit the meta info
iguana::visit<foo>(foo_visitor{});
```
虽然我使用了一些模板元，visitor可以选择性的遍历各种成员，还是挺啰嗦了。如果大家有什么好的idea，欢迎交流。

## 6. LIMITATION & SUMMARY
此次尝试死了不少脑细胞，结果还是有一些限制：
1. 不能反射私有成员；
2. 不能反射函数模板；
3. 还是很啰嗦；
这种用宏的半自动方案，其实可以利用libclang，在pre-compile期做一次分析，然后生成这一份半自动代码，从而改进到全自动的目的。

最后，C++反射依旧需要标准的完善才能有一个比较完备的结果。反射标准就目前的进度来看，最早得C++20才能进TS，等到所有的编译器完整支持，可能到C++23才能够使用。所以，撸一个符合自己项目需求的反射方案，至少还能够服役五年。

以上是我自己实践C++编译期反射的一些思路，肯定有不少槽点，欢迎来purecpp社区吐槽和交流。QQ群号：296561497.