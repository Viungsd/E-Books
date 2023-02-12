# C++左值、右值、引用，一次搞懂

什么是左值，右值，在此不再赘述，本文主要帮你搞懂引用类型作为定义变量、函数形参、函数返回类型时分别都发生了些什么？假设定义了下述类AAA：

```
struct AAA {
    int i=10;
    AAA() {
        cout << "AAA()" << hex << this << endl;
    }
    AAA(const AAA&a) {
        cout << " AAA(const AAA & )" <<"from:" << hex << &a <<" to this:" <<hex<<this << endl;
    }
    ~AAA() {
        cout << " ~AAA()" << hex << this << endl;
    }
};
```

### 1、左值引用、右值引用用于函数内部定义变量

如下表格详细介绍了，这三种类型定义变量时的详情：

|                  | 非引用类型                                                   | 左值引用类型     | 右值引用类型     |
| ---------------- | ------------------------------------------------------------ | ---------------- | ---------------- |
| 是否必须初始化   | NO                                                           | YES              | YES              |
| 变量所占内存大小 | 变量类型的size                                               | ③指针大小        | 指针大小         |
| 如果没有初始化   | 自动调用默认构造函数                                         | Error            | Error            |
| 使用左值初始化   | 调用拷贝构造函数                                             | OK【只引用地址】 | Error            |
| 使用右值初始化   | ①优先调用移动构造，<br />如果没有则调用拷贝构造函数<br />【编译器可能优化】 | ②Error           | OK【只引用地址】 |

Note:

①、一般编译器会进行优化，如果用纯右值进行初始化，编译器连Move都省了

②、const 左值引用可以接受使用右值初始化。

③、无论左值、右值引用，其本质实现都是指针，所占大小都是指针大小，但是你通过sizeof(laaaa) 发现其大小是类型大小而非指针大小，这是因为任何对引用类型变量的使用，编译器都会当场对其原对象的操作，但如果你在一个类中定义一个引用类型【左、右都可】，打印其size就会是指针大小。

无论左值、右值引用、初始化时，只是引用原来变量地址，不会触发任何构造函数。

```
int main()
{
    ///非引用变量定义
    AAA aaa = AAA();///AAA()调用默认构造，生成临时变量纯右值，用纯右值初始化aaa编译器会自动优化，连移动构造都省了
    AAA bbb = aaa;///用左值初始化，调用Copy构造
    AAA ccc = std::move(aaa);///std::move强制把aaa转为右值（将亡值），优先触发移动构造，没有则copy构造

    ///左值引用变量定义
    AAA& laaaa   = aaa;///不会触发任何构造函数、只是引用aaa
   /// AAA& laaaa00 = AAA(); Error ///必须是左值初始化，不能是右值

   ///右值引用变量定义
    AAA&& raaa = AAA();///AAA()调用默认构造函数生成临时变量【右值】，右值赋值给右值引用本身不触发任何调用
   // AAA&& raaa0 = aaa;///右值引用，只接受右值初始化、不接受左值
    AAA&& raa1 = std::move(aaa);///std::move强制把aaa转换为右值，右值赋值给右值引用本身不触发任何调用，整个赋值过程不触发任何构造函数【std::move函数的调用也会被编译器优化，其实就是强制类型转换为(int&&)类型】
}

上述代码运行结果如下：
AAA()000000191CBEF914  ///aaa的初始化，默认构造
 AAA(const AAA & )from:000000191CBEF914 to this:000000191CBEF934 //bbb的拷贝构造
 AAA(const AAA & )from:000000191CBEF914 to this:000000191CBEF954 ///ccc的拷贝构造【本应优先调用移动构造，但没有实现】
AAA()000000191CBEF9B4 ///raaa 初始化所使用的临时变量的默认构造函数
 ~AAA()000000191CBEF9B4
 ~AAA()000000191CBEF954
 ~AAA()000000191CBEF934
 ~AAA()000000191CBEF914
```

通过上述分析，可算是把整个过程整明白了。

### 2、左值引用、右值引用用于函数形参类型

作为函数形式参数定义时，其规则与上面的表格是一样的，参考如下代码：

```
///形参定义为 非引用类型
auto test1(AAA a) {///参数接受左值、右值
}

///形参定义为 左值引用类型
auto test2(AAA& a) {///参数只接受左值
}

///形参定义为 const 左值引用类型
auto test3(const AAA& a) {///参数接受左值、右值
}

///形参定义为 右值引用类型
auto test4(AAA&& a) {///参数只接受右值
}

int main()
{
    AAA aaa = AAA();
    cout << "start--" << endl;

    test1(aaa);///左值作为实参，调用拷贝构造函数
    test2(aaa);///不触发任何构造函数
    test3(aaa);///不触发任何构造函数
  ///  test4(aaa);///Error 不接受左值
    cout << "======" << endl;

    test1(AAA());///AAA()调用默认构造函数生成临时变量，纯右值赋值给非引用类型，编译器优化后免去任何构造
 ///   test2(AAA());///Error 只接受左值，不接受右值
    test3(AAA()); ///AAA()调用默认构造函数生成临时变量，右值赋值给const左值引用，这个过程不触发构造
    test4(AAA());///AAA()调用默认构造函数生成临时变量，右值赋值给右值引用，这个过程不触发构造

    cout << "end" << endl;
}

打印结果如下：

AAA()00000023BD6FF5F4 ///aaa 变量的默认构造函数
start--
 AAA(const AAA & )from:00000023BD6FF5F4 to this:00000023BD6FF6D4 ///test1(aaa);拷贝到test1的形参a
 ~AAA()00000023BD6FF6D4 ///test1函数形参a被析构
======
AAA()00000023BD6FF714/// test1(AAA());代码内部 AAA() 的调用
 ~AAA()00000023BD6FF714 ///test1函数形参a被析构
AAA()00000023BD6FF754/// test3(AAA());代码内部 AAA() 的调用
 ~AAA()00000023BD6FF754 ///test3函数形参a被析构
AAA()00000023BD6FF774 /// test4(AAA());代码内部 AAA() 的调用
 ~AAA()00000023BD6FF774 ///test4函数形参a被析构
end
 ~AAA()00000023BD6FF5F4 ///aaa 变量被析构
```

可以看到，作为函数形参定义时，其规则与上述表格是一致的。



### 3、左值引用、右值引用用于函数返回类型

如下表格详细介绍了，这三种类型作为函数返回类型时的情况：

|                              | 非引用类型  | 左值引用         | 右值引用         |
| :--------------------------- | ----------- | ---------------- | ---------------- |
| 返回局部变量？               | OK【拷贝】  | NO               | NO               |
| 函数调用结果作为？           | 右值        | 左值             | 右值             |
| 函数内部返回对象，会触发构？ | 拷贝构造    | 不会【只是引用】 | 不会【只是引用】 |
| 形参作返回值？               | YES【拷贝】 | 看情况           | 看情况           |
|                              |             |                  |                  |

#### 1、函数返回局部变量？

```
AAA test1() {///总是安全的，会拷贝一个副本返回
    AAA a;
    a.i = 5;
    return a;///优先调用拷贝构造函数，如果拷贝构造函数被禁，会调用移动构造函数（局部变量，即将失效）。
}

AAA& test2() {///编译不报错，运行会出现BUG,函数返回的引用对象【局部变量】已经被析构了。
    AAA b;
    b.i = 5;
    return b;
}

AAA&& test3() {///编译不报错，运行会出现BUG,函数返回的引用对象【局部变量】已经被析构了。
    AAA c;
    c.i = 5;
    return std::move(c);
}

int main()
{
    cout << "start--111" << endl;

    AAA aaa = test1();
//  AAA& aaa0 = test1();///Error
    AAA&& aaa1 = test1();

    cout << "start--222" << endl;

    AAA bbb = test2();///左值作为初始值，调用copy构造，但是函数返回的左值引用的是函数内部已经被析构的局部变量，因而此处会生成错误数据 BUG
    AAA& bb1b = test2();///用左值初始化左值引用，不会触发构造函数，但是引用的对象早已经被析构，BUG
//  AAA&& bb2b = test2(); //Error，右值引用不能接受左值初始化

    cout << "start--333" << endl;

    AAA ccc = test3();///用右值初始化，调用移动构造（没有则调用拷贝），但是该右值引用的对象早已经被析构,BUG
 // AAA& ccc = test3();//Error
 // AAA&& ccc = test3();///VS2022 error : “ccc”:“AAA && ”与“AAA”的间接寻址级别不同
   
    cout << "======" << endl;
}


运行结果如下：

start--111
AAA()000000F8528FFAE4
 AAA(const AAA & )from:000000F8528FFAE4 to this:000000F8528FFC24
 ~AAA()000000F8528FFAE4
AAA()000000F8528FFAE4
 AAA(const AAA & )from:000000F8528FFAE4 to this:000000F8528FFC64
 ~AAA()000000F8528FFAE4
start--222
AAA()000000F8528FFAE4 ///test2 内部局部变量b 构造
 ~AAA()000000F8528FFAE4 ///test2 内部局部变量b 被析构
 AAA(const AAA & )from:000000F8528FFAE4 to this:000000F8528FFC84 ///拷贝了已经被释放的变量，BUG
AAA()000000F8528FFAE4 ///test2 内部局部变量b 构造
 ~AAA()000000F8528FFAE4 ///test2 内部局部变量b 被析构
start--333
AAA()000000F8528FFAE4 ///test3 内部局部变量c 构造
 ~AAA()000000F8528FFAE4 ///test3 内部局部变量c 被析构
 AAA(const AAA & )from:000000F8528FFAE4 to this:000000F8528FFCC4///优先调用移动构造，没有则调用拷贝构造，但是拷贝的是test3 内部已被析构局部变量c ，BUG
======
 ~AAA()000000F8528FFCC4
 ~AAA()000000F8528FFC84
 ~AAA()000000F8528FFC64
 ~AAA()000000F8528FFC24
```

#### 2、形参作为返回值

定义如下代码，其函数返回值都是左值引用，且都是将函数形参作为返回值。

```
namespace B {
    AAA& test1(AAA a) {///接受左值、右值
        a.i = 10;
        return a;///引用了函数形参a，a在函数返回后会被析构, BUG
    }

    const AAA& test2(const AAA& a) {///接受左值、右值
        return (a);
    }

    AAA& test3(AAA& a) {///只接受左值，函数返回后,a所引用的对象依然有效，所以返回a，完全正常。
        return a;
    }

    AAA& test4(AAA&& a) {///只接受右值，临时值，返回一个临时值的引用
        return a;
    }
};
```

##### 1、test1总是会出现BUG。

```
///test1永远会出现BUG。
int main()
{
   AAA& laaa =  B::test1(AAA());///引用了函数内部的形参，函数返回后形参被析构，BUG
   
   ///调用顺序：test1内部局部变量被析构->函数内部形参a被析构->函数返回值作为左值触发aaa的拷贝构造函数
   AAA  aaa  = B::test1(AAA());///BUG，由于函数内部形参a 析构 后 才会调用aaa的拷贝构造，拷贝了一个已被析构的对象
}


运行结果：

AAA()000000992C10F754 ///test1 形参a的构造
 ~AAA()000000992C10F754 ///test1 形参a的析构，laaa引用了这个被析构的对象a，后续如果使用会BUG
AAA()000000992C10F794 ///test1 形参a的构造
 ~AAA()000000992C10F794 ///test1 形参a的析构
 AAA(const AAA & )from:000000992C10F794 to this:000000992C10F674 ///aaa拷贝了上面已经被析构的形参a
======
 ~AAA()000000992C10F674 ///aaa被析构
```

##### 2、test2某些情况会出现BUG：

因为test3不可能接受右值，所以不可能出现BUG。

```
///test2可能导致BUG（当参数为右值且函数结果被 延长了有效期）
int main()
{
    AAA tm;
    const AAA& la4aa = B::test2(tm); ///OK，参数为左值，相当于直接引用了tm，这里不产生任何构造函数调用

    cout << "start--1111" << endl;

    AAA xxx = B::test2(tm);///OK，参数为左值，函数结果为左值，函数返回后tm依然有效，触发copy构造

    cout << "start--2222" << endl;

    const AAA& laaa =  B::test2(AAA());///Error,参数为右值，函数结果为左值，函数返回后形参被析构，BUG, 引用了一个已经被析构的对象
   
    cout << "start--3333" << endl;

   AAA x1xx = B::test2(AAA());///OK，参数为右值，函数结果为左值，函数返回后形参被析构，但是这里x1xx的拷贝构造函数 是在 函数形参被析构之前调用的，所以OK
   
    cout << "======" << endl;
}


运行结果：

AAA()000000F166AFF784 ///tm构造函数
start--1111
 AAA(const AAA & )from:000000F166AFF784 to this:000000F166AFF7C4///xxx以左值初始化，调用copy构造，OK
start--2222
AAA()000000F166AFF8E4 ///test2 参数a构造 
 ~AAA()000000F166AFF8E4 ///test2 参数a析构，laaa引用了这个已经被析构的对象，BUG
start--3333
AAA()000000F166AFF904 ///test2 参数a构造 
 AAA(const AAA & )from:000000F166AFF904 to this:000000F166AFF804 ///x1xx拷贝构造函数调用，OK，从test2的参数a中copy过来
 ~AAA()000000F166AFF904//////test2 参数a析构
======
 ~AAA()000000F166AFF804
 ~AAA()000000F166AFF7C4
 ~AAA()000000F166AFF784
```



##### 3、test4大部分情况会出现BUG：

test4将一个形参【右值】返回给了调用者作为左值，但是函数返回后，引用的形参会被析构，导致BUG

```
///大部分情况会BUG
int main()
{
    const AAA& laaa =  B::test4(AAA());///Erro，
   
    B::test4(AAA()).i = 33; ///Error

    AAA x1xx = B::test4(AAA());///OK，恰好由于x1xx的拷贝构造函数调用 先于 test4形参的析构，所以OK
}

运行结果如下：

AAA()000000F55DAFFA94
 ~AAA()000000F55DAFFA94
AAA()000000F55DAFFAB4
 ~AAA()000000F55DAFFAB4
AAA()000000F55DAFFAD4
 AAA(const AAA & )from:000000F55DAFFAD4 to this:000000F55DAFF9B4
 ~AAA()000000F55DAFFAD4
======
 ~AAA()000000F55DAFF9B4
```

综上所述，函数类型返回左值引用时，需要相当小心，否则会出现难以预测的BUG。
