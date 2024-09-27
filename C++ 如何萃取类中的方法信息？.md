## C++ 如何萃取类中的方法信息？

问题：如何利用C++的编译时类型萃取功能，判断某个类中是否存在某个方法呢？接下来我们来探讨一下：

```
int fun( char a) {
    return a + 4;
}

class  TEST
{
    int i = 9;
public:
    int test(int a) {
        return i;
    }
    int operator()(char a) {
        return a + i;
    }

    int operator()(char a,int am) {
        return a + i;
    }
};

class  TEST00
{
    int i = 9;
public:
   static int test(int a) {
        return 0;
    }
    int operator()(char a, int am) {
        return a + i;
    }
};

template<typename T>
struct is_member_function {
    constexpr static auto value = false;
};

template<typename RET, typename CLS, typename ...ARG>
struct is_member_function<RET (CLS::*)(ARG...)> {
    using type = RET(CLS::*)(ARG...);
    using fun_type = RET (ARG...);
    
    constexpr static auto value = true;
};

int main()
{
  constexpr auto a00 = is_member_function<decltype(&TEST::test)>::value; //true
  constexpr auto a01 = is_member_function<decltype(&TEST00::test)>::value; //false

  constexpr auto a02 = is_member_function<decltype(fun)>::value; //false

  constexpr auto a03 = is_member_function<decltype(&TEST00::operator())>::value; //true

  

    return 0;
}

```

