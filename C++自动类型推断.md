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
      int aray[5] = {};
  
      Test(a);      /// deduced T = int
      Test(axx);    /// deduced T = int
      Test(raaa);   /// deduced T = int
      Test(laaa);   /// deduced T = int
      Test(ccraaa); /// deduced T = int
      Test(cclaaa); /// deduced T = int
      Test(aray);   /// deduced T = int*
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
  
  

