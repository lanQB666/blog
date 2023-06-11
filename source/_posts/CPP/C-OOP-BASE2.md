---
title: C++ OOP BASE2
tags:
  - language
  - cpp
  - cpp11
categories:
  - language
  - cpp
abbrlink: 43100
date: 2023-06-05 18:26:15
---

## 类型转换

### 转换函数 conversion function

```cpp
class Fraction
{
public:
	Fraction(int num, int den=1)
	: m_numerator(num),m_denominator(den){}

	operator double() const
	{return (double)(m_numerator/m_denominator);}

private:
	int m_numerator;
	int m_denominator;
};

Fraction f(3,5);
double d = 4+f;//调用operator double 将f转化为 0.6
```

- **调用**：成员函数 `operator double()` 表示 `Fraction` 转化为 double 类型时需要自动调用的转换函数。
- **语法**：转化函数{% label 不需要返回类型和参数 orange%}。通常需要添加 `const` 。
- **编译器处理**：执行 `double d = 4+f;` 时，编译器会{% label 先找是否定义了全局的操作符重载函数 orange%} `+` ，本例没有定义此函数，所以编译器会寻找 `Fraction` 类的 double 转换函数。
- **类型限制**：转换函数可以是{% label 任何类型 orange%}，不一定是基本类型。

> 总而言之，转换函数可以将 class 转化为其他类型

### Non-**explicit**-one-argument ctor

注意 `Fraction` 的构造函数：

```cpp
class Fraction
{
public:
	Fraction(int num, int den=1)
	: m_numerator(num),m_denominator(den){}

	Fraction operator +(const Fraction& f)
	{return Fraction(...)}

private:
	int m_numerator;
	int m_denominator;
};

Fraction f(3,5);
Fraction d = f+4;//调用 non-explicit ctor 将4转为Fraction
```

- 是双参数 (two parameter) 单实参 (one argument)的函数。
- 当执行 `Fraction d=f+4` 时，编译器会调用重载运算符函数 `+` ，但 4 不是 `Fraction` 类。因此，编译会以 4 为实参调用 **non-explicit ctor** 创建一个 `Fraction` 类的对象再调用重载 `+` 。

> 单参数的构造函数可以看成将其他类型转化 class 的类型。

### Conversion function vs non-explicit-one-argument function

如果重载 `+` 和转换函数都存在：

```cpp
class Fraction
{
public:
	Fraction(int num, int den=1)
	: m_numerator(num),m_denominator(den){}

	operator double() const
	{return (double)(m_numerator/m_denominator)}

	Fraction operator +(const Fraction& f)
	{return Fraction(...)}
private:
	int m_numerator;
	int m_denominator;
};

Fraction f(3,5);
Fraction d = f+4;//error ambiguous
```

此时编译器有多种处理方式：

1. 将 4 构造为 `Fraction` 然后与 `f` 相加得到结果。
2. 将 `f` 转化为 double 然后加 4，再将其构造为 `Fraction`

出现{% label 歧义 orange%}导致编译器报错。

### Explicit-one-argument ctor

可以在构造函数前添加 `explicit` 关键字，表示这个构造函数只能在构造的时候使用，而{% label 不会在转换类型 orange%}的时候使用。

```cpp
class Fraction
{
public:
	explicit Fraction(int num, int den=1)
	: m_numerator(num),m_denominator(den){}

	operator double() const
	{return (double)(m_numerator/m_denominator)}

	Fraction operator +(const Fraction& f)
	{return Fraction(...)}
private:
	int m_numerator;
	int m_denominator;
};

Fraction f(3,5);
Fraction d = f+4;//error conversion from 'double' to 'Fraction'
```

但是此时执行 `Fraction d = f+4;` 编译器将 f 转化为 double 再加 4，此时{% label 结果无法再通过构造函数转化为 orange%} `Fraction` ，还是报错。

### 函数库中的转换函数

![Pasted image 20230604210446.png](https://s2.loli.net/2023/06/12/nQXNV6lO3gWCyw1.png)

`vector` 重载了 `[]` 用于返回一个 `bool` 类型的结果。但函数库内实际采用的是返回 `reference` ，其有类型转换函数 `bool()`

## 智能指针 pointer-like class

![Pasted image 20230604231459.png](https://s2.loli.net/2023/06/12/TV2Don4lYxR751F.png)

- **本质**：class 的对象像指针，完成比指针更多的工作。
- **要求**：class 内部一定包含一个真正的指针并且需要重载 `*` 和 `->` ，值得注意的是两者重载都是{% label 无参数 orange%}的常量函数。
- `->` 特别点在于会{% label 循环 orange%}使用，例如 `sp->method()` 中 `sp->` 返回了 `px` 默认又会再次调用 `px->` ，最后得到 `px->method()` 。

### 关于迭代器

![Pasted image 20230604232756.png](https://s2.loli.net/2023/06/12/hH2XZDdQWKGv4fF.png)

迭代器也是一种智能指针，可以看到除了重载了 `*` 和 `->` ，迭代器还重载了 `++` `--` 等更多的运算符。

{% note primary %}
链表与迭代器
![Pasted image 20230604233056.png](https://s2.loli.net/2023/06/12/D9Fh4RMgknxzuCA.png)

迭代器 `*` 获得迭代器当前所指向结点的元素，即为 `(*node).data`

迭代器 `->` 获得迭代器当前所指向结点的元素的地址，即为 `data` 的地址 `&(operator*())`
{% endnote %}

## 仿函数 function-like classes

```cpp
template <class T1,class T2>
struct Pair{
	T1 first;
	T2 second;
	pair():first(T1()),second(T2()) {}
	pair(const T1 &a, const T2 &b):first(a),second(b){}
...
};

template <class T>
struct identity{
	const T&
	operator()(const T &x) const{return x;}
};
template <class T>
struct select1st{
	const typename Pair::first_type&
	operator()(const Pair &x) const
	{return x.first;}
};
template <class T>
struct select2nd{
	const typename Pair::second_type&
	operator()(const Pair &x) const
	{return x.second;}
};
```

- 一个类对 `()` 进行重载，就变成仿函数
- `select1st` 和 `select2nd` 以一个 `Pair` 为传入参数，返回 `Pair` 的第 1、2 个元素。
- 上面列处的标准库仿函数并不完整，实际上在{% label 标准库中的仿函数还会继承 orange%}一些奇特的 base classes 如 `unary_function` `binary_function` ，这些主题在标准库内容继续讨论。

## 成员模板

```cpp
template <class T1,class T2>
struct pair
	typedef T1 first type;
	typedef T2 second type;

	T1 first;
	T2 second;

	pair()
		:first(T1()),second(T2()){}
	pair(const T1 &a,const T2 &b)
		:first(a),second(b){}

	template <class U1,class U2>
	pair(const pair<Ul,U2>& p)
		:first(p.first),second(p.second){}
};
```

![Pasted image 20230605161821.png](https://s2.loli.net/2023/06/12/C8PkW6l39tgA4BJ.png)

- 在类成员函数定义时再创建一个函数模板
- 观察成员模板，有 `first(p.first),second(p.second)` ，意思是初始化时可以传入任意类型 `U1,U2` ，但是必须{% label 满足要求 orange%}： `U1,U2` {% label 两种类型能够分别初始化类模板的两种类型 orange%} `T1,T2`
	- 可以用派生类初始化基类
	- 因此可以用 `pair<Derived1,Derived2>()` 初始化一个 `pair<Base1,Base2>` 类型，反之不行。

> 总而言之，类成员模板可能有要求，在定义时正确，在实际使用时可能会不符合要求而出错。

![Pasted image 20230605163124.png](https://s2.loli.net/2023/06/12/LNEXKFx9yU8e1BZ.png)

- 基类的指针可以指向派生类对象，即为派生类指针可以初始化基类指针。

> 可以用派生类初始化基类；派生类指针可以初始化基类指针，指向基类的指针可以指向一个派生类

## 模板特化 specialization

![Pasted image 20230605174102.png](https://s2.loli.net/2023/06/12/6gWxNEBupSPRCqw.png)

```cpp
template<>
struct class<typename>{
...
};
```

- 特化是泛化的反义词
- 泛化模板可以让任意类型按照同一段泛化模板的代码段运行
- 当泛化模板和特化模板均存在时，对应特化化模板指定的类型而言，{% label 优先执行特化模板 orange%}所属的代码。（如上图的 char, int, long）其他类型仍然执行泛化模板。

### 偏特化 (局部特化) partial specialization

#### 数量

```cpp
//泛化模板
template<typename T,typename Alloc=......>
class vector
{
...
};
//将第一个参数特化
template<typename Alloc=...>
class vector<bool,Alloc>
{
...
```

上图泛化模板有两个参数，最后一个参数有默认值。可以创建一个特化模板，将第一个参数特化为 bool。

#### 范围

```cpp
//泛化模板
template <typename T>
class C
{
	...
};
// 特化模板
template
<typename T>
class C<T*>
{
	...
};

C <string>
```

## 模板模板参数 

```cpp
template<typename T,
		template <typename T>
		class Container
		>
class XCls
{
private:
	Container<T>c;
public:
	...
};

```

```cpp
template<typename T>
using Lst list<T,allocator<T>>;
```

```cpp
XCls<string,list> mylst1;// 错误
XCls<string,Lst> mylst2;
```

- 在模板定义的语法中，因为历史遗留问题， `<>` 中 `typename` 和 `class` 是一个意思。
- `list` 是一个容器，限制需要设计一个 class 可以通过模板{% label 传入一个容器以及容器 orange%}的类型。
- `XCls<string,list>` 错误，因为模板的第二个类型应该是一个模板。因此应该用 `list` 设计一个新模板（这块的语法属于 C++11），再将其作为参数。

```cpp
template<typename T,
		template <typename T>
			class Smartpt
		>
class XCls
{
private:
	Smartptr<T> sp;
public:
	XCls() : sp(new T) { }
};
```

```cpp
XCls<string,shared_ptr>pl;
XCIs<double,unique_ptr>p2;//错误
XCls<int,weak_ptr>p3;//错误
XC1s<long,auto_ptr>p4;
```

## C++11 三大主题

### 不定数量参数的模板 variadic templates

![Pasted image 20230608200318.png](https://s2.loli.net/2023/06/12/gywKYxpUdbectio.png)

- **语法**：模板参数包 `typename...` ; 函数参数类型包 `const Types&...` ; 函数参数包 `args...` 。{% label 都是在末尾添加 orange%} `...`
- **含义**：模板参数包 `Types` 即定义了任意数量的任意类型，函数类型包定义了任意数量的输入参数 `args` ，在函数体内使用 `args...` 使用 `args` 。
- 本例本质上是将输入参数分为一个参数 `firstArg` 和一包参数 `args` ，每次将 `firstArg` 打印输出，然后将 `args` 作为参数调用 `print` 。当没有输入参数时，会调用 1 处的 `print()` 作为{% label 递归终止 orange%}什么也不打印。
- **参数数量**：可以使用 `sizeof(args)` 查看参数包的参数数量

### Auto

```cpp
// 不使用auto
list<string>c;
...
list<string>::iterator ite;
ite find(c.begin（）,c.end（）,target);
```

```cpp
// 使用auto
list<string>c;
...
auto ite find(c.begin（）,c.end（）,target);
```

由编译器自己自动匹配返回值类型

### Ranged-base for 

- 语法：从右边的容器的每个元素赋值给左边的变量

```cpp
for (decl : coll){
	statement
}
```

- 使用

```cpp
fox(int i:{2,3,5,7,9,13,17,19){
	cout<<i<<endl;
}
```

```cpp
vector<double>vec;
...
for (auto elem:vec){ //迭代值
	cout<<elem<<endl;
}
for (auto& elem:vec){ //迭代引用
	elem *=3;
}
```

## 引用 reference

![Pasted image 20230608220653.png](https://s2.loli.net/2023/06/12/RXchz1OxDjEvm9J.png)

- C++11 可以创建引用类型的变量
- **初始化**：引用类型变量创建时可以代表其他变量
- **赋值**：引用类型变量初始化后就{% label 不能再重新代表其他变量 orange%}，如代码中 `r=x2` 会导致 x 和 r 均赋值为 5。
- **大小和内存**：变量和引用的大小和内存均相同 (假象)

![Pasted image 20230608223558.png](https://s2.loli.net/2023/06/12/onci1zEbmvtOJyS.png)

- 引用一般不作为变量创建，而是用来传递参数，作为{% label 参数类型和返回类型 orange%}。
- 引用和指针相比优势在于，接口更统一，用法和传值相同。引用和传值相比速度更快。
- 相同类型、个数的引用传递和值传递函数不能同时存在。

## 对象模型：vptr 和 vtbl

![Pasted image 20230608232146.png](https://s2.loli.net/2023/06/12/8fyiKAxpYgqwXvG.png)

- **基本情况**：A 有两个虚函数 `vfunc1` `vfunc2` ，B 继承 A 并重写了 `vfunc1` ，C 继承 B 并再次重写 `vfunc1` 。三种类共有四个虚函数和四个函数。
- **vptr**：当类{% label 有虚函数时 orange%}，类的内存中存在{% label 一个 orange%}指针 vptr ，该指针指向已给 vtbl 
- **vtbl**：vrbl 是一个表格，每个表项存放{% label 虚函数 orange%}的地址。
- **调用虚函数**：当调用虚函数时，先通过 p （对象的地址）找到 vptr，再找到表格 vtbl，最后找到对应的虚函数并调用。

### This pointer

![Pasted image 20230611220814.png](https://s2.loli.net/2023/06/12/96QXhf8TvYqpiwe.png)

- **基本情况**：设计模式——Template
-  `OnFileOpen()` 在内部会调用 `Serialize()` ，而父类和子类都有一个该虚函数 `Serialize()` 。子类在调用 `OnFileOpen()` 时，内部执行的是{% label 子类自身的 orange%} `Serialize()` ，这本质上是因为子类是通过 `this` 指针调用该函数的，所以调用的是自己的 `Serialize()` 。

### 动态绑定

![Pasted image 20230611222851.png](https://s2.loli.net/2023/06/12/nzdDHgVx6wsYL8C.png)

![Pasted image 20230611222902.png](https://s2.loli.net/2023/06/12/93I2tAZ4mJxb6oE.png)

- `a.vfunc1()` 是**静态绑定**，调用的是类 A 的函数 `vfunc1` ，对应汇编语句是直接通过 `call` 指定调用类 A 的函数 `vfunc1` 的{% label 固定地址 orange%}。
- 后面两个 `pa->vfunc1()` ，虽然指针 `pa` 是 A 类型的，但是它指向的是类 B 的一个对象的地址，因此是**动态绑定**，调用的是类 B 的函数 `vfunc1` ，对应汇编语句是通过 `call` 执行调用一个{% label 动态地址 orange%}。

## 其他主题

### Const

![Pasted image 20230611225439.png](https://s2.loli.net/2023/06/12/3hkHpaf8vjFD61g.png)

- 在函数小括号后面用 const 修饰函数，表示该函数不会修改数据成员。
- Const 对象不能调用非 const 成员函数。{% label 如果类中成员函数 const 和 non-const 版本均存在 orange%}，则 const 对象只会调用 const 成员函数，非 const 对象只会调用非 const 成员函数。
- **COW**：标准库中的字符串在实现是考虑了写时复制，常字符串调用常成员函数，该情况不需要考率 cow；一般字符串调用成员函数需要考虑 cow，因为内容相同的多个字符串共享同一块内存区，如果对字符串进行修改（写），则会复制字符串再修改。

### New 和 delete 

![Pasted image 20230611233226.png](https://s2.loli.net/2023/06/12/QfVdHPb9O6Dk3RB.png)

可以重载**全局**的 `new,new[],delete,delete[]` ，参数为大小，由编译器传递。

![Pasted image 20230611233425.png](https://s2.loli.net/2023/06/12/Ghk1RQpLd6uOMgl.png)

![Pasted image 20230611233639.png](https://s2.loli.net/2023/06/12/lh2NHKPYynvISD6.png)

- 可以重载成员函数 `new,delete` `new[],delete[]`

```cpp
Foo *pf = new(300,'c')Foo;
```

- 重载 `new(),delete()` ，第一个参数是大小，后面的参数由自己设计