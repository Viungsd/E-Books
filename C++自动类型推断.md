# C++自动类型推断

### 1、C++中的类型退化（decay）

- 裸数组（或者数组的引用）退化成指针类型（也说不上退化，其实跟C语言中是一样的）
- 去除CV属性（const、volatile）等
- 左值引用，右值引用 退化成非引用类型

### 2、C++中什么时候会发生类型退化呢？

- auto  类型变量

  ```
  int main()
  {
      int a = 90;
      const int axx = a;
      int&& raaa = 90;
      int& laaa = a;
      const int&& ccraaa = 90;
      const int& cclaaa = a;
      int aray[5] = {};
      
      ///VS2022 鼠标指向auto变量名，编译器就会提示推断出来的是什么类型！
      auto _a = a;    ///int _a
      auto _axx = axx;  ///int _axx
      auto _raaa = raaa; ///int _raaa
      auto _laaa = laaa; ///int _laaa
      auto _ccraaa = ccraaa; ///int  _ccraaa
      auto _cclaaa = cclaaa; ///int _cclaaa
      auto pA = aray; ///int *pA
  }
  ```

- 函数模板参数为值传递

  ```
  template<typename T>
  void Test(T a) {///函数参数为按值传递
      ///cout << "T:" << boost::typeindex::type_id_with_cvr<T>().pretty_name() << endl;
  }
  
  int main()
  {
      int a = 90;
      const int axx = a;
      int&& raaa = 90;
      int& laaa = a;
      const int&& ccraaa = 90;
      const int& cclaaa = a;
      int aray[5][8] = {};
  
      Test(a);      /// deduced T = int
      Test(axx);    /// deduced T = int
      Test(raaa);   /// deduced T = int
      Test(laaa);   /// deduced T = int
      Test(ccraaa); /// deduced T = int
      Test(cclaaa); /// deduced T = int
      Test(aray);   /// T:int (*)[8] 直接使用数组名字，实际上类型退化了
      Test(&aray);  /// T:int (*)[5][8]
      Test(aray[0]); /// T:int *
      Test(&aray[0]); /// T:int (*)[8]
      Test(aray[0][0]); /// T:int
      Test(&aray[0][0]); /// T:int*
  }  
  ```

- 调用了std::decay

  ```
  #include <type_traits>
   
  template <typename T, typename U>
  constexpr bool is_decay_equ = std::is_same_v<std::decay_t<T>, U>;
   
  int main()
  {
      static_assert(
          is_decay_equ<int, int> &&             ///YES
          ! is_decay_equ<int, float> &&         ///NO
          is_decay_equ<int&, int> &&            ///YES
          is_decay_equ<int&&, int> &&           ///YES
          is_decay_equ<const int&, int> &&      ///YES
          is_decay_equ<int[2], int*> &&         ///YES
          ! is_decay_equ<int[4][2], int*> &&    ///NO
          ! is_decay_equ<int[4][2], int**> &&   ///NO
          is_decay_equ<int[4][2], int(*)[2]> && ///YES
          is_decay_equ<int(int), int(*)(int)>   ///YES
      );
  }
  ```
  

### 3、auto  、auto& 、auto&& 区别

|        | 类型     | 发生类型退化 | 初始值为左值 | 初始值为右值 |
| ------ | -------- | ------------ | ------------ | ------------ |
| auto   | 非引用   | YES          | 非引用       | 非引用       |
| auto&  | 左值引用 | NO           | 左值引用     | Error        |
| auto&& | 通用引用 | NO           | 左值引用     | 右值引用     |

auto上面已经介绍过了，在此不再赘叙，接下来主要介绍后面两种：

##### 1、auto&的使用情况：

根据引用折叠的规则，auto&将始终表示左值引用，且CV（const、volatile）属性不会被丢弃，且数组名也不会被退化成指针，代码参考如下：

```
int main()
{
    int a = 90;
    const int axx = a;
    int&& raaa = 90;
    int& laaa = a;
    const int&& ccraaa = 90;
    const int& cclaaa = a;
    int aray[5][8] = {};

    ///VS2022 鼠标指向auto变量名，编译器就会提示推断出来的是什么类型！
    auto& _a = a;    ///int &
    auto& _axx = axx;  ///const int & 
    auto& _raaa = raaa; ///int& && = int & 折叠引用 
    auto& _laaa = laaa; ///int& &  = int & 折叠引用 
    auto& _ccraaa = ccraaa; ///const int & 折叠引用
    auto& _cclaaa = cclaaa; ///const int & 折叠引用
    auto& pA = aray;   ///int (&) [5][8]
    auto& pA1 = aray[0];/// int (&)[8]
    auto& pA2 = aray[0][0];/// int &
   // auto& p00A = &aray[0]; ///Error C2440: “初始化”: 无法从“int (*)[8]”转换为“int (*&)[8]”
   // auto& pA = &aray;/// Error auto &”与“int (&)[5][8]”的间接寻址级别不同
}
```

##### 2、auto&& 通用引用的使用情况：

所谓“通用引用”，传递的初始化值是左值则推断为左值引用，初始化是右值则推断为右值引用，且CV（const、volatile）属性不会被丢弃，且数组名也不会被退化成指针。参考代码如下：

```
int main()
{
    int a = 90;
    const int axx = a;
    int&& raaa = 90;
    int& laaa = a;
    const int&& ccraaa = 90;
    const int& cclaaa = a;
    int aray[5][8] = {};

    ///初始值都是左值，因此类型推断都是左值引用
{
    auto&& _a = a;    /// int & 
    auto&& _axx = axx;  ///const int & 
    auto&& _raaa = raaa; ///int& && = int & 折叠引用 
    auto&& _laaa = laaa; ///int& &  = int & 折叠引用 
    auto&& _ccraaa = ccraaa; ///const int & 折叠引用
    auto&& _cclaaa = cclaaa; ///const int & 折叠引用
    auto&& pA = aray;   ///int (&) [5][8]
    auto&& pA1 = aray[0];/// int (&)[8]
    auto&& pA2 = aray[0][0];/// int &
}
   
    ///初始值都是右值，因此类型推断都是右值引用
    auto&& _a = std::move(a);    /// int && 
    auto&& _axx = std::move(axx);  ///const int && 
    auto&& _raaa = std::move(raaa); ///int&& && = int && 折叠引用 
    auto&& _laaa = std::move(laaa); ///int&& &&  = int && 折叠引用 
    auto&& _ccraaa = std::move(ccraaa); ///const int && 折叠引用
    auto&& _cclaaa = std::move(cclaaa); ///const int && 折叠引用
    auto&& pA = std::move(aray);   ///int (&&) [5][8]
    auto&& pA1 = std::move(aray[0]);/// int (&&)[8]
    auto&& pA2 = std::move(aray[0][0]);/// int &&
}
```

### 4、decltype类型推断

decltype关键字可以编译时推断表达式的类型，不会导致上述所说的类型退化！decltype推导规则如下：

1. 如果e是一个没有带括号的标记符表达式或者类成员访问表达式，那么的decltype（e）就是e所命名的实体的类型。此外，如果e是一个被重载的函数，则会导致编译错误。
2. 否则 ，假设e的类型是T，如果e是一个将亡值，那么decltype（e）为T&&
3. 否则，假设e的类型是T，如果e是一个左值，那么decltype（e）为T&。
4. 否则，假设e的类型是T，则decltype（e）为T。

```
int main()
{
    int a = 90;
    const int axx = a;
    int&& raaa = 90;
    int& laaa = a;
    const int&& ccraaa = 90;
    const int& cclaaa = a;
    int aray[5][8] = {};

   ///VS2022 鼠标指向变量名，编译器就会提示推断出来的是什么类型！因此很好验证

   ///表达式为标记符表达式，且没有自带括号，与原类型完全一致，不会出现类型退化
    decltype(a) _a = 40;    /// int
    decltype(axx) _axx = 50;  ///const int 
    decltype(raaa) _raaa = std::move(raaa); ///int&&
    decltype(laaa) _laaa = laaa; ///int&
    decltype(ccraaa) _ccraaa =  std::move(ccraaa); ///const int && 
    decltype(cclaaa) _cclaaa = std::move(cclaaa); ///const int &
    decltype(aray) pA;  /// int pA[5][8]
   
    ///表达式不是标记符表达式，且没有自带括号,且表达式返回左值，因而推断为引用
    decltype(aray[0]) pA1 = aray[2];/// int (&pA1)[8] :[]操作返回左值，所以推断为引用
    decltype(aray[0][0]) pA2 = aray[0][0];/// int &pA2 : [][]操作返回左值，所以推断为引用

     ///表达式不是标记符表达式，且没有自带括号,表达式结果非左值，
    decltype(aray[0][0]+5) pA3 = aray[0][0];/// int pA3 

    ///表达式带括号，且表达式为左值，因而返回引用
    decltype((a)) _000a = a;/// int &_000a
    
}
```



### 5、decltype(auto)

```
const int ab = 90;
decltype(auto) a = ab; ///等同于：decltype(ab) a = ab;
```

