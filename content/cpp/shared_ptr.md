+++
categories = ["c++11"]
date = "2017-05-25T06:17:57+08:00"
title = "智能指针:指向基类的智能指针与指向子类的智能指针转换"

+++
以前没仔细想过这个问题，自己以前实现过的也不支持这个操作。昨天重读 <<effective c++>>，**条款45时 运用成员函数模板，接受所有兼容类型**。发现了这个问题的实现方法。
<!--more-->

# 声明基类和子类，A，B

	class A
	{
	public:
		A()
		{
			cout << "new A";
		}
		~A()
		{
			cout << "del A";
		}
	};

	class B :public A
	{
	public:
		B()
		{
			cout << "new B";
		}
		~B()
		{
			cout << "del B";
		}
	};
	
# 子类智能指针转基类智能指针
可以的，赋值操作也可以进行

	std::shared_ptr<B> bbb(new B);
	std::shared_ptr<A> aaa(bbb);
	std::shared_ptr<A> aaa2 = bbb;
	
拷贝构造时，会调用以下拷贝构造函数，is_convertible会检查是否可以转化（继承关系）。

	template<class _Ty2,
	class = typename enable_if<is_convertible<_Ty2 *, _Ty *>::value,
		void>::type>
	shared_ptr(const shared_ptr<_Ty2>& _Other) _NOEXCEPT
	{	// construct shared_ptr object that owns same resource as _Other
	this->_Reset(_Other);
	}
	
enable_if这个关键字我还不太会，不过看效果时编译期就可以检查出是否可以转换成功。基类转子类的智能指针时无法编译通过的：

	shared_ptr<A> aaa(new A);
	shared_ptr<B> bbb(aaa); // 无法转换
	
	 error C2664: 'std::shared_ptr<B>::shared_ptr(std::shared_ptr<B> &&) noexcept': cannot convert argument 1 from 'std::shared_ptr<A>' to 'std::nullptr_t'
	 
# 另一种转化方法：operator T*();

游戏项目中是自己实现的智能指针，没有特殊为这种转换写构造函数，而是写了的转换，在上述调用时参数自动转换成T*。
**不过这种实现方法要求引用计数保存在对象中，而不是智能指针**，因为无法对隐世转换的智能指针增加计数。




