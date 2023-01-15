# C++ Metaprogram

### 1、如何在编译期判断类中是否有指定名称的成员变量？

定义如下几个类，如何判断类中是否存在名称为“c”的成员变量呢？

```
struct AAA {
	int c = 0;///对象的成员变量
};

struct BBB {
	static float c;///类静态成员变量，也就是全局变量
};

struct CCC {
	char m;
	static double c0;///类静态成员变量，也就是全局变量
};
```

可以看到，AAA、BBB中都包含成员变量c，但是c的类型不同，且一个是静态成员变量，一个不是。代码如下：

```
///非类型模板参数 C++17后支持 auto或decltype(auto) 推断
template<typename T,auto = &T::c>///这里的auto最关键，使得可以同时支持类的静态成员变量和对象成员变量
std::true_type has_member_c(T);

std::false_type has_member_c(...);

template<typename T>
bool has_member_c_v = decltype(has_member_c(T{}))::value;

int main() {
	auto a = has_member_c_v<AAA>;//True，AAA包含名称为c的对象成员变量,auto推断为成员对象指针int AAA::*
	auto b = has_member_c_v<BBB>; //True，BBB包含名称为c的静态成员变量,auto推断为float*
	auto c = has_member_c_v<CCC>; ///False，CCC中不包含名称为c的成员变量
	auto d = has_member_c_v<int>; ///False，int中不包含名称为c的成员变量
	
	return 0;
}
```

分析：上述代码实现中只要类中包含名称为c变量的，无论其类型，无论其是否静态成员变量，都可。那问题来了，如果我们要区分静态成员变量和对象成员变量，那该如何呢？？

### 2、区分静态成员变量和对象成员变量

1、识别对象成员变量

```
template<typename T, auto T::* p = &T::c>//c必须是对象成员变量，静态成员变量此次会推断失败
std::true_type has_instance_member_c(T);

std::false_type has_instance_member_c(...);

template<typename T>
bool has_instance_member_c_v = decltype(has_instance_member_c(T{}))::value;

int main() {
	struct DDD {
		char c = 0;///对象的成员变量
	};

	auto a = has_instance_member_c_v<AAA>; ///True，AAA包含名称为c的对象成员变量,auto推断为int AAA::*
	auto b = has_instance_member_c_v<BBB>; //False，BBB包含的是名称为c的静态成员变量,所以第一个模板推断失败
	auto c = has_instance_member_c_v<CCC>; ///False，CCC中不包含名称为c的成员变量
	auto d = has_instance_member_c_v<int>; ///False，int中不包含名称为c的成员变量
	auto e = has_instance_member_c_v<DDD>; ///True，DDD中包含名称为c的成员变量,auto推断为char DDD::*
	return 0;
}
```

2、静态成员变量，这个就简单了，利用上面的结果直接取反即可。

```
template<typename T>
bool has_static_member_c_v = has_member_c_v<T> && (!has_instance_member_c_v<T>);

struct EEE {
	static char c;///静态成员变量
};

int main() {
	auto a = has_static_member_c_v<AAA>; ///False，AAA不包含静态成员变量c
	auto b = has_static_member_c_v<BBB>; //True，BBB包含静态成员变量c
	auto c = has_static_member_c_v<CCC>; ///False，CCC中不包含静态成员变量c
	auto d = has_static_member_c_v<int>; ///False，int中不包含静态成员变量c
	auto e = has_static_member_c_v<EEE>; ///True，EEE中包含静态成员变量c
	
	return 0;
}
```

### 3、在上面基础上加上对成员变量的类型判断

上述代码，只识别了成员变量的名称为"c"，但是对其类型没有任何限制，如果我们需要判断类中包含一个名称为c的成员变量，且其类型必须为指定类型，那该如何呢？

