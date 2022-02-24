---
layout:     post
title:      Default-Constructor
subtitle:   c++
date:       2022-2-24
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - c++
---

# Default Constructor构建

Default Constructor 只有在编译器需要的时候才会被合成出来，被合成出来的 constructor 只执行编译器所需的行动。也就是说即使有需要为一个 class 合成一个 default constructor，那个 constructor 也不会将我们的 members object 进行初始化，所以设计者必须提供一个明显的 default constructor,将 members object 进行初始化操作。

对于 class X，如果没有任何 user-declared constructor，那么会有一个 default constructor 被暗中声明出来，一个被暗中声明出来的 default constructor 将是一个 trivial（无用的） constructor。

一个 nontrivial default constructor 只有在编译器需要的情况下才会被合成出来。即四种情况：

	1）带有 Default Constructor 的 Member Class Object
	
	2）带有 Default Constructor 的 Base Class
	
	3）带有一个 Virual Function 的 Class
	
	4）带有一个 Vitual Base Class 的 Class

## 带有 Default Constructor 的 Member Class Object

如果一个 class 没有任何 construct，但它内含一个 member class object，并且这个 member class object 有 default constructor，那么这个 class 的 implicit（隐式）default constructor 就是一个 “nontrivial（有用的）”，编译器需要为此 class 合成出一个 default constructor。不过这个合成操作只有在 constructor 真正需要被调用时才会发生。

**如：**

Bar::foo是一个 Member Class Object，而class Foo拥有 default constructor。

	class Foo  { Public: Foo();};
	class Foo1 { Public: Foo1();};
	class Foo2 { Public: Foo2();};
	class Foo3 { Public: Foo3();};
	class Bar  { Public: Foo foo; Foo foo1; Foo foo2; Foo foo3; int iNu; };
	
	int main()
	{
		Bar bar; //定义
	}

被合成的 Bar default constructor 内含必要的代码，能够调用 Bar class类中所有的的 Member Class Object 的 Default Constructor 来处理对应的 member object bar::foo、 member object bar::foo1、member object bar::foo2、member object bar::foo3，但它并不产生任何代码来初始化 Bar::iNu。因为初始化前者是编译器的责任，将 Bar::iNu 初始化则是程序员的责任。

如果 class Bar 内含一个或着一个以上的 Member Class Object，那么 class Bar 的每一个 constructor 必须调用每一个 Member Class Object 的 Default Constructor。如果 class Bar 的 default constructor已经被明确定义出来了，那么编译器会扩张这个已经存在的 Constructor，在其中安插一些代码，使得 user code 在被执行之前，先调用必要的 Default Constructor。

**显示定义的default constructor：**

	Bar::Bar()
	{
		iNu = 0;	
			
	}

扩张后：

	Bar::Bar()
	{

		//调用 对应的 Constructor
		foo.Foo::Foo();
		foo1.Foo1::Foo1();
		foo2.Foo2::Foo2();
		foo3.Foo3::Foo3();
		
		//初始化自己的成员
		iNu = 0;				
	}
	
c++语言要求以 “Member Class Object” 在 class 中的声明顺序来调用各个 Constructor。这一点由编译器完成，它为每一个 constructor 安插程序代码，以 member 声明顺序调用每一个 member 所关联的 default constructor，这些代码被安插在 explicit user code 之前。

## 带有 Default Constructor 的 Base Class

一个没有任何 constructor 的 class 派生自一个 “带有 default constructor” 的 base class，那么这个 derived class 的 default constructor 会被视为 nontrivial，并因此需要被合成出来。它将调用上一层 base classes 的 default constructor（根据它们声明的次序）。对一个后继派生的 class 而言，这个合成的 constructor 和一个 “被明确提供的 default constructor” 没什么差异。

当存在多个 constructors ，但其中都没有 default constructor 时。编译器会扩张现有的每一个 constructor，将 “用以调用所有必要的 default constructor” 的程序代码加进去，它不会合成一个新的 default constructor，这是因为其它 “由 user 所提供的 constructor” 存在的缘故。如果同时也存在着 “带有 default constructors” 的 member class objects，那些 default constructor 也会被调用——在所有 base class constrouctor 都被调用之后。

## 带有一个virtual Function 的 Class

这两种情况，也需要合成出 default constructor:

	1.class 声明（或继承）一个 virtual function。
	
	2.class 派生自一个继承串链，其中有一个或更多的 virtual base classes。

**例：**

纯虚类

	class Widget
	{
		public:
			virtual void flip() = 0;
		
		//.....
		
	};
	
	void flip (const Widget& widget) { widget.flip(); }

假设 Bell 和 Whistle 都派生自 Widget

	void foo()
	{
		Bell b;
		Whistle w;
	
	
		flip( b );	
		flip( w );
	}

当上面代码只执行时，在编译期间会对代码进行扩张：

&nbsp; &nbsp;1）一个 virtual function table（vtbl）会被编译器产生出来，用来存放 class 的 virtual functions 的地址。

&nbsp; &nbsp;2）在每一个 class object中，一个额外的 Pointer Member（vptr）会被编译器合成出来，内含相关的 class vtbl 的地址。


同时，对 widget.flip() 的 虚拟引发操作会被重新改写：
	
	// widget.flip(); 扩展为 	

	( *widget.vptr[1] )( &widget );

&nbsp; &nbsp;1）其中 1 表示 flip() 在 virtual table 中的固定索引。

&nbsp; &nbsp;2）&widget 代表要交给 “被调用的某个 flip() 函数实体” 的 this 指针。

为了实现这个扩张操作，编译器必须为每一个 Widget（或其派生类）Object 的 Vptr 设顶初值，放置适当的 virtual table 地址。对于 class 所定义的每一个 constructor，编译器会安插一些代码来做这些事情。对于那些未声明任何 constructors 的 classes，编译器会为他们合成一个 default constructor，以便正确的初始化每一个 class object 的 vptr。

## 带有一个 Virtual Base Class 的 Class

Virtual Base Class 的实现法在不同的编译器之间有极大的差异。然而，每一种实现法的共同点在于必须使用 virtual base class 在其每一个 derived class object 中的位置，能够于执行期准备妥当。

**例：**

	class X { public: int i; };
	
	class A : public virtual X { public : int j; };
	
	class B : public virtual X { public : double d; };
	
	class C : public A, public B { public : int k; };
	
	//无法在编译时期确定出 pa->X::i 的位置
	void foo ( const A *pa ) { pa->i = 1024; }
	
	int main()
	{
		
		foo (new A);
	
		foo (new C);	
	
		// ...
	}

编译器无法固定住 foo() 之中“经由 pa 而存取的 X::i” 的实际偏移位置，因为 pa 的真正类型可以改变。编译器必须改变“执行存取操作”的那些代码，使 X::i 可以延迟至执行期才决定下来。

原先 cfront 的作法是靠 “在 derived class object 的每一个 virtual base classes 中安插一个指针” 完成。所有 “经由 reference 或 pointer 来存取一个 virtual base class” 的操作都可以通过相关指针完成。如foo()可以被改写：

	//可得的编译器转变操作，__vbcX 表示编译器所产生的指针，指向 virtual base class X
	//会在这个derived class object对象构建时一起被生成，派生类会调用基类的constructor
	
	void foo( const A *pa ) { pa->__vbcX->i = 1024; }

\_\_vbcX（或编译器所做出的某个什么东西）是在 class object 构建期间被完成的。对于 class 所定义的每一个 constructor，编译器会安插那些 “允许每一个 virtual base class 的执行器存取操作” 的代码。如果 class 没有声明任何 constructor，编译器必须为它合成一个 default constructor。

## vtbl(虚函数表) 与 vptr

c++对具有 Virtual Functions 用两个步骤来提供而外的支持：

&nbsp; &nbsp;1）每一个 class 产生出一堆指向 Virtual Functions 的指针，放在表格中，这个表被称为 virtual table（vtbl）。

&nbsp; &nbsp;2）每一个 class object 被添加了一个指针，指向相关的 virtual table。通常这个指针被称为vptr。vptr 的设定和重置都由每一个 class 的 constructor 、 destructor 和 copy assignment运算符自动完成。每一个 class 所关联的 type\_info object 也经由 virtual table 被指出来，通常放在表格的第一个 slot 处。



## 总结

这四种情况，会导致 “编译器必须为未声明 constructor 的 class 合成一个 default constructor”。C++ Stardand 把那些合成物称为 implicit nontrivial default constructor（隐式的有用的默认构造函数）。被合成出来的 constructor 只能满足编译器（而非程序）的需要。它之所以能够完成任务，是借着 “调用 member object 或 base class 的 default constructor” 或是 “为每一个 object 初始化其 virtual function 机制或 virtual base class 机制” 而完成。至于没有存在那四种情况而有没有声明任何 constructor 的 classes，我们说他们拥有的是 implicit trivial default constructors（隐式的无用的默认构造函数），他们实际上并不会被合成出来。

在合成的 default constructor 中，只有 base class subobjects 和 member class objects 会被初始化。所有其他的 nonstatic data member，如整数、整数指针、整数数组等等都不会被初始化。这些初始化操作对程序而言或许有需要，但对编译器则并非必要。如果程序需要一个 “把某指针设为0” 的 default constructor，那么提供他的人应该事程序员。

**即：**

&nbsp; &nbsp;1）任何 class 如果没有定义 default constructor，并不一定会被合成出一个来。

&nbsp; &nbsp;2）编译器合成出来的 default constructor 不会明确设定 “class 内每一个 data member 的默认值”。


## 参考 《深度探索C++对象模型》
