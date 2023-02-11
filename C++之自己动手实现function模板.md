# C++之自己动手实现std::function

C++的function模板可以包裹C++中一切可以被callable的对象，包括：

1. lamada表达式（匿名仿函数对象）
2. 普通函数指针（或者类的static函数指针）
3. 类的成员函数指针
4. 仿函数对象（实现operator( )函数的任意对象）

目的：上述4种情况，其类型在C++中都不一样，但通过function的包裹后，可以把它们都统一成一种类型。

```
///2.普通函数指针
int testFun(int a,int b) {
    cout << "testFun -> a:" << a << "  b:" << b << endl;
    return a + b;
}

///4.仿函数对象
struct A {
    int i = 10;
    int cc = 90;
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

- 4种callable对象在C++中类型本不同，需要使用类型萃取技术统一成相同类型int(int ,int)
- 函数指针、仿函数等调用方式本不同，如何统一调用方式 ？
- 4种callable对象size不同，（例如，函数指针大小为4个字节，而lamada对象大小取决于所捕获的变量），如何在function对象中同时支持这四种对象？
- 同一个function< int (int,int) >对象，如何同时支持上面四种对象？（可以同时用函数指针、仿函数对象、lamada表达式等进行赋值操作）

```
template <typename RET,typename ...ARC>
struct base_callable
{
   // virtual void copy(void* dest) = 0;
   // virtual void move(void* dest) = 0;
    virtual RET operator()(ARC...) = 0;
};

template <typename T, typename RET, typename ...ARC>
struct noalloc_callable :base_callable<RET,ARC...> {
    noalloc_callable(T _a):_m(_a) {
    }

    RET operator()(ARC... arc) override{
        return _m(arc...);
    }

    T _m;
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

///const 类函数指针
template <typename RET, typename CLS, typename ...ARC >
struct is_mem_func<RET(CLS::*)(ARC...) const> {
    using type_func = RET(ARC...);
};

///普通函数指针
template <typename RET, typename ...ARC >
struct is_mem_func<RET(ARC...)> {
    using type_func = RET(ARC...);
    using base_func = base_callable<RET, ARC...>;
    
    using base_type = func<type_func>;

    RET operator()(ARC... arc) {  
        auto p = static_cast<base_type*>(this)->get_ptr();   
        return (*p)(arc...);
    }

    template<typename C>
    is_mem_func(C arg) {
        using callable_type = noalloc_callable<C,RET,ARC...>;
        auto& data = static_cast<base_type*>(this)->data;
        if (sizeof(callable_type) <= sizeof(data.content)) {
            new(data.content) callable_type(arg);
            data._[base_type::capacity - 1] = (base_func*)&data.content;
        }
    }

};


template<typename T>
struct func :is_mem_func<T> {
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
    func(C arg):is_mem_func<T>(arg) {
    }
     
    union {
        base_func* content[(capacity - 1)];
        base_func* _[capacity];
    }data;
};


template<typename T>
func(T)->func<typename is_mem_func<decltype(&T::operator())>::type_func>;

template<typename RET, typename ...ARG>
func(RET(ARG...))->func<RET(ARG...)>;

```



