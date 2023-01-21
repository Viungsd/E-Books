# C++强制类型转换与C语言的区别

我们首先复习一下C语言的类型转换：

1、隐式转换（编译器自动转换，编译器认为不会发生危险）一般为占内存较小的类型转换成较大的类型，此时数据不会丢失。例如：bool->char、char->int、int->long etc

2、显式转换（强制类型转换，编译器认为可能会有危险即提出警告，但若程序员认为没有危险则可强转）一般为占内存大的类型转小的类型，可能发生数据丢失。其后果由程序员负责。例如：int->char 其高位数据将会丢失，可能导致未知后果

无论何种转换，都有两种后果：

- 变量的类型信息发生变化（Must Be）
- 变量的值发生变化(May be)

|                  | 隐式转换 | 显式转换                                                     |
| ---------------- | -------- | ------------------------------------------------------------ |
| 类型信息变化？   | YES      | YES                                                          |
| 变量值发生变化？ | NO       | 1、int a=1; char c = char(a); //没变<br />2、double a=8.1; int b=int(a);///变化，小数位丢失<br />3、int * pInt = &a; double  * pD = (double*)pInt; //没变 |

我们不难发现一个简单道理：**C语言中对指针类型的进行任何形式的类型转换其值都不会改变。**

为何我要强调这一点呢？由于C++中引入多继承、虚函数机制，导致这一点上C与C++会有根本性不同。**言则C++中指针进行类型转换时，指针所表示的地址值编译器会自动根据类型之间的继承关系以及内存布局情况进行修正。**

参考如下代码：

```
struct AAA {
	int a = 2;///对象的成员变量
	int b = 1;
};

struct BBB:AAA {///存在虚函数，导致每个对象起始处增加一个虚函数表指针vftable
	int c = 5;
	virtual void pt() {
	}
};
///由于出现了虚函数，同一地址，进行不同的类型转换，其值可能不一样,在C语言中则不会出现这种情况
int main() {
	BBB bb;
	
	BBB* pB = &bb;                     ///0x005afd2c,指向bb的首地址
	AAA* pA = pB;                      ///0x005afd30,隐式转换=pb+4(越过vftable)
	BBB* pBB = static_cast<BBB*>(pA);  ///0x005afd2c,强制转换=pA-4(保留vftable)
	AAA* pVA = (AAA*)(void*)pB;        ///0x005afd2c
	AAA* pLA = (AAA*)(long)pB;         ///0x005afd2c

	return 0;
}
```

通过上面的简单的例子可以发现，在C++中同样的一个指针用不同的方式进行转换时其值是可能发生变化的。我们再继续探讨多继承时的情况：

```
struct AAA {
	int a = 2;///对象的成员变量
	int b = 1;
};
struct BBB {
	int c = 5;
	int d = 3;
};
struct CC :AAA, BBB {
	int e = 6;
};

int main() {
	CC c;

	CC* pC = &c;                        ///0x012ffd38
	AAA* pA = pC;                       ///0x012ffd38
	BBB* pB = pC;                       ///0x012ffd40
	CC* pCA = static_cast<CC*>(pA);     ///0x012ffd38
	CC* pCB = static_cast<CC*>(pB);     ///0x012ffd38

	return 0;
}
```

如果对C++对象的内存布局模型有所了解的话，便能很容易理解其指针值的变化规律**—遵循C++对象的内存布局**

