# C++自动类型推断

### 1、C++中的类型退化（decay）

- 裸数组退化成指针类型
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

- 调用了std::decay

  

