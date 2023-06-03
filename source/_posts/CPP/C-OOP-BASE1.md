---
title: C++ OOP BASE1
tags:
  - cpp
  - oop
categories:
  - language
  - cpp
abbrlink: 43292
date: 2023-05-31 15:16:35
---

本课程涵盖两条主线：**Object-Oriented** (面向对象) 和**泛型编程** (Generic
Programming)。

Part I 主要讨论 OO (面向对象) 的基础概念和编程手法。课程首先讨论 **Class without pointer members** 和 **Class with pointer members** 两种类型，而后上升至 OOP/OOD，包括 classes 之间最重要的三种关系：**继承** (Inheritance)、**复合** (Composition)、**委托** (Delegation)。

Part II 继续探讨相关主题，並加上低阶对象模型 (Object Model)，以及高阶的 Templates (模板) 编程。


## 类的分类

1. 不带有指针的类
2. 带有指针的类

![Pasted image 20230530223140.png](https://s2.loli.net/2023/06/03/sjaGSDdiEe9vFu6.png)

{% note warning  [style] %}
两种分类分别代表了两个设计方式：
1. **Object Based**: 面向单一 class 的设计
2. **Object Oriented**: 面向多重 classes 的设计，classes 和 classes 的关系
{% endnote %}

## 内联函数

![Pasted image 20230531002251.png](https://s2.loli.net/2023/06/03/UHDNF9KZQfTgx4k.png)

- **性能**： `inline` 关键字用来定义一个类的内联函数，引入它的主要原因是用它替代 C 中表达式形式的**宏定义**, 解决一些频繁调用的小函数大量消耗{% label 栈空间 orange%}（栈内存）的问题。
- **限制**： `inline` 的使用是有所限制的， `inline` 只适合函数体内代码{% label 简单 orange%}的函数使用，{% label 不能包含复杂的结构控制语句 orange%}例如 `while` 、 `switch` ，并且不能内联函数本身{% label 不能是直接递归函数 orange%}（即，自己内部还调用自己的函数）。
- **对编译器**： `inline` 函数仅仅是一个对编译器的**建议**，所以最后能否真正内联，看编译器的选择，并不是说声明了内联就会内联，声明内联{% label 只是一个建议 orange%}而已。
- **类的成员函数**：{% label 定义在类中 orange%}的成员函数{% label 缺省都是内联 orange%}的，如果在类定义时就在类内给出函数定义，那当然最好。如果在类中未给出成员函数定义，而又想内联该函数的话，那在类外要加上 `inline` ，否则就认为不是内联的。

## Access level 访问级别

![Pasted image 20230531003211.png](https://s2.loli.net/2023/06/03/mqhKMQClw6bzPk8.png)

- `public` 内的成员可以直接使用，而 `private` 中的成员不能直接使用，需要借助 `public` 的成员函数间接使用
- 应该将所有数据放在 `private` 中，可被外界调用的函数放在 `public` 中

## Constructor 构造函数

```cpp
class complex
{
public:
	complex(double r=0,double i=0)
	: re(r),im(i)
	{}
	...
}
```

等效于

```cpp
class complex
{
public:
	complex(double r=0,double i=0)
	{ re = r;im = i;}
}
```

- 成员变量再构造函数中初始化中使用": "函数方式**速度**比直接在构造函数中赋值要**快**。

> 编写构造函数时，尽量使用 ": " 方式编写。

### 构造函数重载 overloading

![Pasted image 20230531004344.png](https://s2.loli.net/2023/06/03/xwi1PUJE58fylWg.png)

- **编译器视角**：编译器会根据{% label 函数名和参数列表 orange%}重新命名
- 重载 `real` 是可行的，因为新的 `real` 与旧 `real` 参数不同
- 高亮部分的代码会出现问题，因为第一个构造函数{% label 已经有参数默认初始化列表 orange%}了，定义该类对象时可以不加入参数，这就产生了冲突。

### 构造函数放在 `private`

如果{% label 不希望类被构造 orange%}，就可以将构造函数放在 `private` 区内

![Pasted image 20230531005527.png](https://s2.loli.net/2023/06/03/8HIfQEBlraYW4XC.png)

- `Singleton` 设计模式要求外界只能存在一个对象
- 静态函数 `getInstance()` 会创建一个静态对象

## 常量成员函数 

![Pasted image 20230531010226.png](https://s2.loli.net/2023/06/03/bdLhUnDIVaF1uwr.png)

如图中 "?!" 标识的代码，使用者用 `const` 标识了对象，而编写者在编写 `real()` 和 `imag()` 时没有添加 `const`  ，则使用者在调用这两个函数时，编译器判断 `real()` 和 `imag()` 可能会更改对象而报错。

> 简而言之，如果设计成员函数时，如果它不需要修改成员，就一定要加上 `const`

## 值传递和引用传递

### 参数传递：pass by value vs. Pass by reference (to const)

![Pasted image 20230531014648.png](https://s2.loli.net/2023/06/03/YrFaLOGbnpdulgB.png)

- **pass by value**：直接传递参数
- **pass by reference**：传引用，在传递参数末尾添加 `&`
- **pass by reference to const**： `const` 修饰 pass by reference

> 函数参数传递，如果不需要改变参数值，建议使用==常引用传递== `const reference` 减小开销。类似于 C 语言传递指针再加 `const` 。

### 返回值传递：return by value vs. Return by reference (to const)

![Pasted image 20230531014702.png](https://s2.loli.net/2023/06/03/zwTAiSte1qfWV5b.png)

==返回值建议使用引用==，但如果返回引用的是该函数的一个指向堆局部变量指针，那么不能使用引用。

> 如果合法的话，返回值尽量使用引用传递

## 友元 friend

- **声明**：如果类想将一个函数作为友元，只需要在类中添加一条 `friend` 修饰的函数声明。友元函数==不是类的成员==。
- **位置**：友元函数在类中出现的位置语法上不受限制，但最好放在开始或结束的位置==集中==声明。
- **作用**：友元函数可以直接访问对应 class 的对象的非公有成员
- ==相同 class 的不同对象互为友元==

![Pasted image 20230531152907.png](https://s2.loli.net/2023/06/03/LZHwpqetGnFOE8B.png)

{% note info %}
title: 使用友元会打破封装
在封装的情况下，如果数据放在 `private` 区只要类的接口不变，那么类作者就可以比较自由的修改数据
{% endnote %}

## 操作符重载

### 成员函数操作符重载 （this 指针）

![Pasted image 20230531155722.png](https://s2.loli.net/2023/06/03/nsDphQzuayo8lMT.png)

- **运算符**：编译器碰到运算符会调用左边类型重载的运算符，在运算符重载过程中隐藏了 this 指针，该指针编译器会处理。
- **this**：所有成员函数都隐含 this 指针，使用者{% label 不能显式声明 orange%}，但是可以直接使用。

### 回到 return by reference

{% note  primary%}
![Pasted image 20230531163213.png](https://s2.loli.net/2023/06/03/J8amUr6XPpfznR3.png)
相较于传指针而言，传引用发送者不需要关心接收者是传值还是传引用。
{% endnote %}

## 非成员函数操作符重载

![Pasted image 20230531170157.png](https://s2.loli.net/2023/06/03/fVQoHv8thKsx7P6.png)

这种情况不能返回引用，因为返回值一定是局部变量。

特殊情况： 输入输出运算符 `<<` 和 `>>` 只能写成**非成员函数方式**，因为它们左侧必须是 `istream` 或 `ostream` 的成员。

## 带指针的类

设计一个 string 类，应实现的功能：

```cpp
int main()
{
	String s1(),
	String s2("hello");

	String s3(s1); #copy ctor
	cout << s3 << endl; 
	s3 = s2; #copy op

	cout << s3 << endl;
}
```

## 三大函数

{% note warning %}
带指针的类必须编写三大函数：

![Pasted image 20230531232727.png](https://s2.loli.net/2023/06/03/VW8CxrZb2FtMPSQ.png)

默认情况下，类的拷贝按照==比特复制==的方式实现，不带指针的类默认的拷贝实现就能胜任此工作，但带指针的类就会导致复制后的类中的指针指向同一片区域（如图）
{% endnote %}

### 析构函数

```cpp
inline
String::String(const char* cstr = 0)
{
	if (cstr) {
		m_data = new char[strlen(cstr)+1];
		strcpy(m_data, cstr);
	}
	else { // 未指定初值
		m_data = new char[1];
		*m_data = '\0';
	}
}
inline
String::~String()
{
	delete[] m_data;
}
```

- 带指针的类需要{% label 用析构函数清理动态分配的内存 orange%}

### 拷贝构造函数 copy constructor

```cpp
inline
String::String(const String& str)
{
	m_data = new char[ strlen(str.m_data) + 1 ];
	strcpy(m_data, str.m_data);
}
```

- 注意 `str.m_data` ，由于同 class 的不同对象互为友元，所以可以直接访问双方的 `private` 数据。

### 拷贝赋值函数 copy assignment operator

```cpp
inline
String& String::operator=(const String& str)
{
	if (this == &str) #检查自我赋值
		return *this;

	delete[] m_data;
	m_data = new char[ strlen(str.m_data) + 1 ];
	strcpy(m_data, str.m_data);
	return *this;
}
```

- 步骤：先将原有空间清空，再重新分配空间并拷贝。
- 自我赋值：检查 `=` 两边的是否是同一对象（地址是否相同）

## 栈、堆和内存管理

### 栈、堆和生存期

- Stack object 在作用域结束后消失
- Static object 在程序结束后结束
- Global object 作用域是整个程序
- heap object 在 `delete` 后结束，不 `delete` 会内存泄漏

### New

![Pasted image 20230601154646.png](https://s2.loli.net/2023/06/03/XO7kKJqpdQVARj1.png)

编译器处理过程：

1. 分配空间，内部调用  `malloc()`
2. 类型转化 `static_cast`
3. 调用类的构造函数

### Delete

![Pasted image 20230601155021.png](https://s2.loli.net/2023/06/03/fcLdexiCA4arpIB.png)

编译器处理过程：

1. 调用类的析构函数
2. 释放内存，内部调用 `free()`

### 动态分配内存块

#### 对象

有不带指针和带指针的两个类：

```cpp
class complex
{
	public:
		complex (double r = 0, double i = 0)
		: re (r), im (i)
		{ }
		complex& operator += (const complex&);
		double real () const { return re; }
		double imag () const { return im; }
	private:
		double re, im;
		friend complex& __doapl (complex*,
		const complex&);
};


class String
{
	public:
		String(const char* cstr = 0);
		String(const String& str);
		String& operator=(const String& str);
		~String();
		char* get_c_str() const { return m_data; }
	private:
		char* m_data;
};
```

在 VC 分配的空间如下：

![Pasted image 20230601155426.png](https://s2.loli.net/2023/06/03/jWl1NwMt6EDXSIc.png)

调试模式：

- 头尾部分（红色部分） $2\times 4$ 字节 
- 调试信息部分（灰色部分）$4\times 8+4$ 字节
- Complex 实际占用（绿色部分）$2\times 4$ 字节
- 填充部分（森绿色），VC 按照 16 字节{% label 对齐 orange%}，用{% label 全零 orange%}填充

Release 模式：

- 头尾部分（红色部分） $2\times 4$ 字节 
- Complex 实际占用（绿色部分）$2\times 4$ 字节

#### 数组

![Pasted image 20230601160849.png](https://s2.loli.net/2023/06/03/3FQSUhtwMsZed1z.png)

- 三个 Complex 对象连续排列，并且添加了一个 3 表示数组长度，其他的部分与上文类似

Array new 一定要搭配 array delete：

![Pasted image 20230601160943.png](https://s2.loli.net/2023/06/03/KS4LPIwBlO3Qu8n.png)

- 如果不使用 array delete（不加 `[]` ）就{% label 只会调用一次析构函数 orange%}，如果对象属于带指针的类，就会内存泄漏

## Static

![Pasted image 20230601184459.png](https://s2.loli.net/2023/06/03/qnegmXtdp9GOchx.png)

- 静态成员{% label 函数/数据 orange%}前有 `static` 关键字，静态数据成员需要在类外初始化 ( `double Account::m_rate=8.0` )
- 静态成员数据只有一份；静态成员函数与一般成员函数的区别在于不能使用 `this` 指针
- 调用成员函数 `c1.real()` 其实就相当于 `complex::real(&c1)` 。意思是不同对象调用的成员函数{% label 本质 orange%}是调用 class 的函数（静态成员函数）+对象的地址（ `this` 指针）。

![Pasted image 20230601165357.png](https://s2.loli.net/2023/06/03/4bEF3CtwrGpLvVk.png)

静态函数可以通过 class 和 object 调用

## 模板简介

### 类模板 Class Template

```cpp
template<typename T>
class complex
{
public:
	complex (T r = 0, T i = 0)
	: re (r), im (i)
	{ }
	complex& operator += (const complex&);
	T real () const { return re; }
	T imag () const { return im; }
private:
	T re, im;
	friend complex& __doapl (complex*, const complex&);
};
```

- 用 `template` 定义模板参数以便后面定义类的时候使用。
- **意义**：通过模板替换可以获得许多形式相同的新类型，减少代码量。
- **实例化**：类在实例化创建对象的时候需要指定模板对应的类型。

```cpp
{
	complex<double> c1(2.5,1.5);
	complex<int> c2(2,6);
	...
}
```

### 函数模板

```cpp
template <class T>
inline
const T& min(const T& a, const T& b)
{
return b < a ? b : a;
}
```

- 模板参数列表中的类型可以出现在返回值、函数参数和函数体中。
- **使用**：使用时，编译器会对函数模板进行**参数推导**，通过函数参数的类型推导，因此不需要指定类型。

```cpp
stone r1(2,3), r2(3,3), r3;
r3 = min(r1, r2);
```

## Namespace

```cpp
namespace std
{

}
```

- 建立一个命名空间，以放自定义函数名等与其他命名空间冲突

### 使用

```cpp
using namespace std;
```

- 可以直接使用命名空间中的名称 `cin,cout`

```cpp
using std::cout;
```

- 可以直接使用指定命名空间中的特定名称  `cour` ；命名空间的其他名称不能直接使用 `cin`

```cpp
std::cin
std::cout
```

- 使用特定命名空间的名称

## 组合和继承

三种关系

- 继承
- 复合
- 委托

### 复合 Composition

![Pasted image 20230601220849.png](https://s2.loli.net/2023/06/03/IKxjv9mG8Ju1Ngd.png)

![Pasted image 20230601221807.png](https://s2.loli.net/2023/06/03/3MnebWPxESdK6HO.png)

-  `queue` 有一个 `deque` 对象，即 `queue` 复合 `deque` 类型
- 从内存角度看，上图的复合表示 `queue` 对象有一个 `deque` 对象的内存
- `queue` 的函数实现是通过调用 `c` 的成员函数实现的

#### 复合的构造和析构

![Pasted image 20230601225015.png](https://s2.loli.net/2023/06/03/W8gP7DvkNa2iEcf.png)

- **构造**：由内至外，先调用 component 的默认构造函数，最后执行自己的构造函数
- **析构**：由外至内，先执行自己的析构函数，再执行 component 的构造函数

{% note warning %}
构造中，调用的是 component **默认**构造函数，如果不满足需求（如 component 是一个带指针的类），就需要自己在类的构造函数中实现

对**编译器**而言，会自动添加 `class()`
```cpp
Container::Container(…): { … };
==>
Container::Container(…): Component() { … };
```
{% endnote %}

### 委托 Delegation / Composition by reference

![Pasted image 20230601225715.png](https://s2.loli.net/2023/06/03/KZbJiSDgtGuyF6U.png)

-  `String` 有一个 `StringRep` 的**指针**，即 `String` 委托 `StringRep` 。 
- 将任务{% label 委托给指针指向的类实现 orange%}

### 继承

![Pasted image 20230601232323.png](https://s2.loli.net/2023/06/03/WIvj4rZTbFpNoeO.png)

- 方式：继承包括三种方式 public, private, protected
- 语法：高亮部分即为继承的语法
- 逻辑：继承从逻辑上表示为父类的“{% label 一种 orange%}”类别
- 继承：
	- 父类的数据会{% label 完整 orange%}的保存下来。同时子类可能有自己的数据。
	- 子类继承父类的函数，即有父类函数的{% label 调用权 orange%}

![Pasted image 20230601232849.png](https://s2.loli.net/2023/06/03/h4xq7RfHV9vKaPS.png)

可以看到，子类包含着父类的成分。
构造和析构过程和复合类似，分别由内至外或由外至内。

### 带虚函数的继承 Inheritance with virtual functions

- 非虚函数 non-virtual ：{% label 不希望子类重新定义 orange%}的函数。
- 虚函数 virtual：用 `virtual` 标识的成员函数，{% label 希望 orange%}子类重新定义的函数，但有默认定义
- 纯虚函数 pure virtual： 用 `virtual` 标识且末尾有 `=0` 的成员函数，子类{% label 一定 orange%}要重新定义

```cpp
class Shape {
public:
	virtual void draw( ) const = 0; // pure virtual
	virtual void error(const std::string& msg);//virtual
	int objectID( ) const; //non-virtual
	...
};
class Rectangle: public Shape { ... };
class Ellipse: public Shape { ... };
```

{% note primary%}
上述代码中，`objectID()` 用于为每个对象产生ID，因此不希望子类覆盖此函数，将其设计为**非虚函数**；`error()` 用于产生错误信息，由于不同形状产生的错误可能不同，子类可能会覆盖，因此将其设计为**虚函数**；`draw()` 用于绘制图形，不同形状的绘制不同，要求子类一定要重写此函数，因此设计为**纯虚函数**。
{% endnote %}

{% note primary %}
设计模式-Template Method：
![Pasted image 20230603002723.png](https://s2.loli.net/2023/06/03/KmY34jkSwXyOZCs.png)

- 情况：`CMyDoc` 继承 `CDocument`， `CDocument` 中有两个函数 `OnFileOpen` 和 `Serialize`， `OnFileOpen` 的实现需要使用 `Serialize`。
- 其中，`Serialize` 与{% label 具体文件的类型有关 orange%}，所以应当将其设置为**虚函数**，要求子类 `CMyDoc` 重写
- 调用基类成员函数：`myDoc. OnFileOpen` 意为调用基类的成员函数，在编译器将其处理成 `CDocument:: OnFileOpen (&myDoc)`(`this` 指针为 `&myDoc`)
- 调用过程：先执行 `OnFileOpen` 的函数体，直到执行 `Serialize` 时，根据 `this` 指针找到 `Serialize` 的函数体(基类覆盖的 `Serialize`)
{% endnote %}

### 继承+复合情况下的构造和析构

![Pasted image 20230603004739.png](https://s2.loli.net/2023/06/03/iMHXIDJTYvor57K.png)

- 第二种情况很简单，子类、基类和复合类层层包裹，构造和析构的顺序如下：

```
Component constructed!
Base constructed!
Derived constructed!
Derived destructed!
Base destructed!
Component destructed!
```

- 第一种情况

![Pasted image 20230603010806.png](https://s2.loli.net/2023/06/03/rJHN9ISmz5koptA.png)

```
Base constructed!
Component constructed!
Derived constructed!
Derived destructed!
Component destructed!
Base destructed!
```

## 委托+继承

### 多窗口打开文件的例子

![Pasted image 20230603011231.png](https://s2.loli.net/2023/06/03/aLGJKUi7NmR4I1p.png)

- 委托：一个文件 ( `Subject` ) 可以用多个窗口 ( `Observer` ) 打开，因此 `Subject` 委托给多个 `Observer`
- 继承：同一个文件的不同窗口可以有{% label 不同的呈现方式 orange%}，因此会有很多不同的类继承 `Observer` 以不同的方式展示文件
- 更改文件：当文件更改后，调用 `notify` 以更新所有窗口 ( `Observer` )

### 设计模式-Composite

![Pasted image 20230603014453.png](https://s2.loli.net/2023/06/03/6mfAzWuaIxl9BNn.png)

- `Primitive` 是对象个体； `Composite` 是窗口容器，其存放的可以是对象个体，也可以是窗口容器。因此，可以为两者写一个基类 `Component` ，而 `Composite` 就委托给 `Component` ，这样就实现了上述要求。
- `Component` 中的 `add` 函数不能是纯虚函数，应为虚函数。窗口容器可以 `add` 容纳多个组件，而 `Primitive` 不能有此行为，因此 `Primitive` 不能重写 `add`

### 设计模式-Prototype

设计一个原型以应对{% label 未来才会出现的子类 orange%}

![Pasted image 20230603110659.png](https://s2.loli.net/2023/06/03/juNocVFCzKGkOBm.png)

- **基类访问子类**：在子类 `LandSatImage` 中创建一个{% label 自身类型 orange%}的静态对象 `LSAT` ，其在创建时调用内部 `private` {% label  区内的构造函数 orange%}，然后构造函数内会调用 `addPrototype` 将对象放到原型类中{% label 以便原型类访问 orange%}。
- **基类创建子类**：原型类定义纯虚函数 `clone` 要求子类定义重写 `clone` 函数，原型类就可以通过子类的静态对象调用 `clone` 创建新对象。进一步，原型类的 `findAndClone` 可以找到给定子类并调用 `clone` 。

## 总结

课程从类的分类角度切入，分为不带指针的类 (如 Complex) 和带指针的类 (如 String)。介绍了{% label 内联函数、类的访问级别、构造函数、常量成员函数、值传递与引用传递、友元和操作符重载 orange%}。由于带指针的类存在指针指向另外一块内存区域，所以要特别注意{% label 三大函数 orange%} (析构函数、拷贝构造函数、拷贝赋值函数)需要自己编写。进一步，介绍了{% label 静态成员函数、模板简介以及命名空间 orange%}。最后有类之间的{% label 三大关系 orange%} (组合、委托和继承)，以及由此衍生的常见设计模式。
