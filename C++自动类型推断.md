# C++自动类型推断

### 1、C++中的类型退化（decay）

- 裸数组（或者数组的引用）退化成指针类型（只降1维，2维->1维）
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

|        | 类型     | 发生类型退化 | 必须初始化？ |
| ------ | -------- | ------------ | ------------ |
| auto   | 非引用   | YES          | YES          |
| auto&  | 左值引用 | NO           | YES          |
| auto&& | 通用引用 | NO           | YES          |

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

### 5、decltype(auto)
