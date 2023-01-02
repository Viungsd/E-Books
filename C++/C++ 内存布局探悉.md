# C++ 类的内存布局探悉

本文将深度探悉C++的类内存布局，借助VS2022开发工具可以让我们看到类的内存布局情况，操作步骤如下：

1. 打开VS2022开发人员命令提示符(Developer Command Prompt for VS 2022)，注意普通命令提示符不行
2. 运行命令：cl /d1reportSingleClassLayoutObj main.cpp（Obj是所要打印的类的名字）

### 1、最简单的类，无继承，无虚函数

```
struct Obj {
	int m_obj1;
	int m_obj2;
};
```

命令输出内容如下：

```
class Obj       size(8): ///总大小=8
        +---
 0      | m_obj1 ///第一个是m_obj1，4个字节大小
 4      | m_obj2 ///第二个是m_obj2,4个字节大小
        +---
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：**类里面的元素按照声明的顺序依次排列（考虑内存对齐，下面不再提示内存对齐问题）**

### 2、无继承，但有虚函数的类

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void print() {
	}
	virtual ~Obj() {
	}
};
```

命令输出内容如下：

```
class Obj       size(12):///总大小=12，起始处多了一个vfptr指针指向该类的虚函数表Obj::$vftable@
        +---
 0      | {vfptr} ///虚函数表指针，4个字节大小，指向了虚函数表，以供运行时调用虚函数
 4      | m_obj1
 8      | m_obj2
        +---

Obj::$vftable@: ///虚函数表
        | &Obj_meta  ///类的相关信息，用于RTTI,运行时类型识别
        |  0         ///暂时不知为何多一个0？
 0      | &Obj::print   ///Obj::print
 1      | &Obj::{dtor}  ///Obj::~Obj 析构函数

Obj::print this adjustor: 0
Obj::{dtor} this adjustor: 0
Obj::__delDtor this adjustor: 0
Obj::__vecDelDtor this adjustor: 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

总结：

1. 类中一旦有虚函数，会导致类的所有对象的首地址处增加一个指向虚函数表的指针vfptr
2. 虚函数表中首部保存了该类类型信息地址：&Obj_meta，用以支持运行时类型识别
3. 虚函数表中依次保存了该类的所有虚函数地址

### 3、单继承的类

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Animal:Obj {
	int m_anmal1;
	int m_anmal2;
	
	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(20):///总大小=20
        +---
 0      | +--- (base class Obj)
 0      | | {vfptr}  ///指向Animal::$vftable@ 地址
 4      | | m_obj1
 8      | | m_obj2
        | +---
12      | m_anmal1
16      | m_anmal2
        +---

Animal::$vftable@:
        | &Animal_meta ///类的相关信息，用于RTTI,运行时类型识别
        |  0
 0      | &Animal::print
 1      | &Animal::{dtor} //Animal没有重写析构函数，由此可见此时编译器自动生成了析构函数
 2      | &Animal::speak  //新增的虚函数，在末尾处增加

Animal::print this adjustor: 0
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 0
Animal::__delDtor this adjustor: 0
Animal::__vecDelDtor this adjustor: 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：

1. 子类的虚函数表中，子类重写的虚函数会在原位置覆盖父类的虚函数
2. 子类新增的虚函数会增加在虚函数表后面
3. 父类的成员变量在前面，子类的成员变量在父类的后面
4. 父子类共用了起始处虚函数指针，指向了子类的虚函数表。由于该表中，子类的重写的虚函数地址已覆盖了父类的，所以无论在子类、父类中调用该函数，最终都会调用子类的虚函数，以实现所谓“动态联编”

### 4、单个类的虚继承

与上述唯一不同的点在于，把普通继承方式修改为虚继承:

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Animal :virtual Obj { ///虚继承,与上述的唯一不同
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(28)://总共增加了3个指针，2个虚函数表指针，一个虚基类表指针
        +---
 0      | {vfptr}//virtual function pointer，指向Animal::$vftable@Animal@，只包含新增的虚函数
 4      | {vbptr}//virtual base pointer,指向Animal::$vbtable，表中包含所有虚基类的数据的偏移量
 8      | m_anmal1
12      | m_anmal2
        +---
        +--- (virtual base Obj)
16      | {vfptr}//指向Animal::$vftable@Obj，只包含所有Obj的虚函数表（子类覆盖父类），不包含新增的
20      | m_obj1
24      | m_obj2
        +---

Animal::$vftable@Animal@://只包含类型信息、新增的虚函数表
        | &Animal_meta   //类型信息，用以支持RTTI
        |  0             //
 0      | &Animal::speak //新增的虚函数

//由于虚继承导致所有虚基类的成员信息将依次放在对象的尾部，如果子类中访问虚基类的数据成员信息，需要知道偏移量
Animal::$vbtable@:///虚基类数据偏移量表，保存类所有的虚基类的数据成员的偏移量
 0      | -4    ///可能表示vbptr的偏移量？
 1      | 12 (Animald(Animal+4)Obj) ///虚基类Obj的数据成员偏移量

Animal::$vftable@Obj@://只包含所有Obj的虚函数表（子类覆盖父类），不包含新增的
        | -16  //表示虚基类Obj的数据成员在整个对象中的偏移量
 0      | &Animal::print ///Obj有，但是被Animal覆盖了
 1      | &Animal::{dtor}///Obj有，但是被Animal覆盖了

Animal::print this adjustor: 16
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 16
Animal::__delDtor this adjustor: 16
Animal::__vecDelDtor this adjustor: 16
vbi:       class  offset o.vbptr  o.vbte fVtorDisp
             Obj      16       4       4 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：由此可知，当继承方式修改为virtual虚继承时，对象的内存布局方式会发生较大改变，主要表现为：

1. 所有虚基类的数据成员信息将不会在头部，而是依次放到尾部，其各自偏移量会保存在一个vbtable表中(virtual Base table)
2. 每个虚基类的数据成员前面都会有该类自己的虚函数表指针，表内只保存该类的所有虚函数（当然子类重写的方法会覆盖父类的）。
3. 整个对象起始处依然有一个虚函数表指针，只不过该表内只包含类型信息、新增的虚函数信息（不含父类的）
4. 整个对象起始处的虚函数表指针后面，会增加一个虚基类表指针，表内记录了所有虚基类数据地址偏移量
5. 所有基类的虚函数表，表头不再包含类型信息，只增加了本类的数据信息在整个对象中的偏移量

### 5、普通多继承

所谓普通继承，指的是非虚继承。

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void ObjTest() {
	}
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Live {
	int m_live1;
	int m_live2;
	virtual void print() {
	}
	virtual void liveTest() {
	}
	virtual ~Live() {
	}
};

struct Animal :Obj ,Live{
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(32)://大小=32，多增加了2个虚函数表指针
        +---
 0      | +--- (base class Obj)
 0      | | {vfptr}///虚函数表指针->Animal::$vftable@Obj，覆盖父类Obj的虚函数表的基础上，增加新增的
 4      | | m_obj1
 8      | | m_obj2
        | +---
12      | +--- (base class Live)
12      | | {vfptr} ///虚函数表指针->Animal::$vftable@Live,覆盖父类Live的虚函数表的基础上，不含新增的
16      | | m_live1
20      | | m_live2
        | +---
24      | m_anmal1
28      | m_anmal2
        +---

Animal::$vftable@Obj@://首个基类的虚函数表，在父类Obj的虚函数表（子类重写会覆盖）基础上，增加新增的虚函数
        | &Animal_meta ///类型信息，支持RTTI
        |  0
 0      | &Obj::ObjTest    //Obj::ObjTest，子类没有重写，所有保留父类的
 1      | &Animal::print   //Obj::print，子类重写了
 2      | &Animal::{dtor}  //Obj::~Obj，析构函数，子类重写了（编译器自动生成的）
 3      | &Animal::speak   //子类新增的虚函数

Animal::$vftable@Live@://Live基类的虚函数表信息，只包含Live自己的虚函数（子类重写会覆盖），不含新增的
        | -12  ///Live类的数据信息地址偏移量
 0      | &thunk: this-=12; goto Animal::print //Live::print，子类重写了，所有调用子类的
 1      | &Live::liveTest                      //Live::LiveTest，子类没有重写，所有保留父类的  
 2      | &thunk: this-=12; goto Animal::{dtor}//Live::~Live，子类重写了，所有调用子类的

Animal::print this adjustor: 0
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 0
Animal::__delDtor this adjustor: 0
Animal::__vecDelDtor this adjustor: 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：

1. 父类数据按照声明顺序依次放在首部位置，最后才是子类的数据成员
2. 子类与第一个声明的类共享虚函数表指针（头部位置的指针），子类重写的虚函数会覆盖所有父类的虚函数表中，但是子类新增的虚函数只增加到第一个声明的类的虚函数表后面
3. 第一个虚函数表和后面的虚函数表格式不同，第一个的表头包含类型信息，而后的虚函数不包含（只包含该类数据地址偏移量）。