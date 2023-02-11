# C++之自己动手实现std::function

C++的function模板可以包裹C++中一切可以被callable的对象，包括：

1. lamada表达式（匿名仿函数对象）
2. 普通函数指针（或者类的static函数指针）
3. 类的成员函数指针
4. 仿函数对象（实现operator( )函数的任意对象）

目的：上述4种情况，其类型在C++中都不一样，但通过function的包裹后，可以把它们都统一成一种类型。

```
///2.普通函数指针
int testFun(int a, int b) {
    std::cout << "testFun -> a:" << a << "  b:" << b << std::endl;
    return a + b;
}

///4.仿函数对象
struct A {
    int i = 10;
    int cc = 90;
    int print(int a) {
        return i + cc + a;
    }
    int  operator()(int a, char c) {///实现仿函数
        return a + c + i + cc;
    }
};

int main() {
///2.普通函数指针
    std::function pF = testFun;///std::function<int (int, int)> pF
    auto ret0 = pF(10,5);
    
///4.仿函数对象
    pF = A{};///std::function<int (int, char)> 
    auto ret1 = pF(10, 5);
    
///1.lamada表达式（匿名仿函数对象）
    pF = [](int a, int b) {///std::function<int (int, int)>
        return a + b;
    };
    auto ret2 = pF(10, 5);

    return 0;
}
```

如何自己实现function的功能呢？本人苦思许久，加上对STL模板的研读，总结后决定自己动手试试，难点如下：

- 4种callable对象在C++中类型本不同，需要使用类型萃取技术统一成相同类型（上面的例子是：int(int ,int)）
- 类成员函数指针、函数指针等调用方式不同，如何统一调用方式 ？
- 4种callable对象size不同，（例如，函数指针大小为4个字节，而lamada对象大小取决于所捕获的变量），如何在一个function对象中同时支持这四种对象？
- 同一个function< int (int,int) >对象，如何同时支持上面四种对象？（可以同时用函数指针、仿函数对象、lamada表达式等进行赋值操作）

```
#include <iostream>
#include <functional>
#include<type_traits>

template <typename RET, typename ...ARC>
struct base_callable///虚基类
{
    virtual void copy(void* dest) = 0;
    virtual RET operator()(ARC...) = 0;
};

template <typename T, typename RET, typename ...ARC>
struct noalloc_callable :base_callable<RET, ARC...> {
    noalloc_callable(T _a) :_m(_a) {
    }

    ///将自己拷贝到dest指定的地方
    void copy(void* dest) override {///调用自己的构造函数，初始化dest指定的内存区域，拷贝自己到dest去
        new(dest) noalloc_callable<T, RET, ARC...>(_m);
    }

    RET operator()(ARC... arc) override {
        if constexpr (std::is_member_function_pointer_v<T>) {///类成员函数指针
            return opt<RET>(_m, arc...);//主要因为类成员函数指针调用方式不同，需要拆解出arc的第一个参数
        }
        else {///函数指针、仿函数、lamada
            return _m(arc...);
        }
    }

    T _m;
private:
    ///必须将arc参数列表的第一个参数拆解出来,作为类成员函数指针调用
    template<typename R, typename TP, typename ARC1, typename ...ARCN>
    R opt(TP pointer, ARC1 a_1, ARCN... a_n) {
        return (a_1.*pointer)(a_n...);
    }
};


template<typename T>
struct func;

///只声明，不实现
template <typename >
struct is_mem_func;

///non const 类函数指针
template <typename RET, typename CLS, typename ...ARC >
struct is_mem_func<RET(CLS::*)(ARC...)> {
    using type_func = RET(ARC...);
};

///const 类函数指针，lamada表达式所用
template <typename RET, typename CLS, typename ...ARC >
struct is_mem_func<RET(CLS::*)(ARC...) const> {
    using type_func = RET(ARC...);
};

///普通函数指针
template <typename RET, typename ...ARC >
struct is_mem_func<RET(ARC...)> {
    using type_func = RET(ARC...);
    using base_func = base_callable<RET, ARC...>;
    using sub_class_type = func<type_func>;///子类类型

    RET operator()(ARC... arc) {
        auto p = static_cast<sub_class_type*>(this)->get_ptr();///强制转换成子类对象
        return (*p)(arc...);///调用虚函数对象实现的仿函数
    }

    template<typename C>
    is_mem_func(C arg) {
        using callable_type = noalloc_callable<C, RET, ARC...>;
        auto& data = static_cast<sub_class_type*>(this)->data;
        if (sizeof(callable_type) <= sizeof(data.content)) {///小对象直接放预留的栈空间
            new(data.content) callable_type(arg);
            data._[sub_class_type::capacity - 1] = (base_func*)&data.content;
        }
        else {///栈空间放不下，需要动态申请内存
        ////待实现....依葫芦画瓢即可
        }
    }
};

///T已经被推断指引转换成普通的函数指针了
///继承is_mem_func的目的是为了在父类中把T中的函数返回类型及参数类型拆解出来以实现仿函数operator()
template<typename T>
struct func final :is_mem_func<T> {
    using super_type = is_mem_func<T>;
    using base_func = typename super_type::base_func;

    static constexpr  auto  capacity = 10;
    func() {
        data._[capacity - 1] = nullptr;
    }

    base_func* get_ptr() {
        return data._[capacity - 1];
    }

    template<typename C>
    func(C arg) :is_mem_func<T>(arg) {
    }

    func& operator=(func c) {
        c.get_ptr()->copy(data.content);
        return *this;
    }
///用来保存较小对象预留的空间（在栈上）,如果下面预留空间放不下则就动态申请内存
///最后一个元素如果保存的是data的起始地址则说明对象是直接放在data里的，否则就指向动态申请的内存
    union {
        base_func* content[(capacity - 1)];
        base_func* _[capacity];
    }data;
};

///推断指引，不管初始化的是普通函数指针、类成员函数指针、仿函数对象、lamda表达式，都会推断成普通函数指针
template<typename T>////仿函数、lamda表达式推断指引
func(T)->func<typename is_mem_func<decltype(&T::operator())>::type_func>;

template<typename RET, typename ...ARG>///普通函数指针推断指引
func(RET(ARG...))->func<RET(ARG...)>;

template<typename RET, typename CLS, typename ...ARG>///类成员函数指针推断指引
func(RET(CLS::*)(ARG...))->func<RET(const CLS&, ARG...)>;

///Test
int main()
{
    A a;
    func ppp = &A::print;  ///func<int (const A &, int)>
    int ret = ppp(a, 6);///106

    ppp = [](const A&, int a) {
        return a * 10;
    };
    int ret666 = ppp(a, 6);///60

    func fc = testFun; /// func<int(int, int)> fc
    int ret0 = fc(4, 5);///9

    fc = A{};
    int ret1 = fc(6, 7);///113

    fc = [=](int a, int b) {
        return a + b + ret0;
    };
    int ret2 = fc(10, 20);///39

    return 0;
}
```



