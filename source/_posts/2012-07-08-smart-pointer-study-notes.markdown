---
layout: post
title: "std::tr1::shared_ptr学习笔记"
date: 2012-07-08 11:27
comments: true
categories: C/C++
---

C++有一个很头疼的事情，当程序比较复杂的时候，管理指针对象的生命周期，对程序员来说相当麻烦。《小小帝国》上一个版本的开发期间，经常出现由于对象没有被析构而产生的内存泄露，或者由于某个地方错误释放对象而导致的非法内存访问。考虑到以后不能一直这样耗功夫去关注这方面的事情，这个版本的《小小帝国》引入了智能指针。

C++98里面只有一种智能指针，就是`std::auto_ptr`，但是它具有唯一所有权的特征，其实不好用(比如说你没有办法在容器中使用它)。《effective c++》作者推荐用的`std::tr1::shard_ptr`就好用很多，它采用RAII技术，通过引用计数的方式来自动管理对象的生存周期，基本上就符合我们的需求了。

而且`std::tr1::shared_ptr`在最新的c++11中，已经被列入标准指针了，而c++98中的`std::auto_ptr`已经出局。

下面是使用`std::tr1::shared_ptr`过程中的一些学习笔记。

##需要引入的头文件
	#include<tr1/memory>
##声明并初始化一个std::tr1::shared_ptr
	std::tr1::shared_ptr<T> ptr(new T());  
##返回实际的对象指针
	std::tr1::shared_ptr<T> ptr(new T());
	T* = ptr.get();
##清空当前shared_ptr
	std::tr1::shared_ptr<T> ptr(new T());
	ptr.reset();
这里所做的事情是：清空当前的`shared_ptr`，并且基于该指针所创建的其它`shared_ptr`指针的引用计数减一。
##智能指针判空
	std::tr1::shared_ptr<T> ptr(new T());
	if (ptr) {
		// do something
	}
##基类向下转型为子类
如果B是A的子类，那么向下转型是这样操作的
	std::tr1::shared_ptr<A> ptr_a(new A());
	std::tr1::shared_ptr<B> ptr_b = std::tr1::dynamic_pointer_cast<B>(ptr_a);
##如何打破循环引用
引用计数有个麻烦，它无法解决所谓循环引用的问题。比如有对象A和B，他们各自有一个智能指针指向对方，这种情况下对象是没有办法被释放的。

这种情况下，我们需要在打破循环的地方使用`std::tr1::weak_ptr`。`weak_ptr`是稍微“弱”一点的智能指针，它不会增加引用计数，但是在指针生命结束以后，又会把引用计数减一，从而可以打破循环引用的死锁情况。

所以在使用`shared_ptr`的时候，如果有引起循环引用的情况，需要在程序的某处使用`weak_ptr`，打破死锁情况，使得对象能够被正常释放。

##如果我们手头上只有对象的原始指针，那怎么办？
当我们手头上只有对象的原始指针，我们如何生成需要的`shared_ptr`。比如在对象的函数内部，我们只有`this`指针，从裸指针中我们怎么获取`shared_ptr`？

下面的方式是不对的。
	std::tr1::shared_ptr<T> ptr_a(this);
这种情况下，ptr_a对this的引用计数只有1，它没有办法知道其它智能指针对this指针的引用情况，所以当ptr_a的生命周期结束之后，它的引用计数变成0， 会把对象释放掉。这就导致其它智能指针的非法内存访问。

tr1提供了`std::tr1::enable_shared_from_this`来解决这个问题，你可以使用它来作为对象的基类，然后就可以从this指针中引出具有统一引用计数的shared_ptr了。
	class T : public std::tr1::enable_shared_from_this<T> 
	{ 
	public: 
  		std::tr1::shared_ptr<T> function() { 
    		return shared_from_this(); 
  		} 
	}; 
这里需要注意的一个问题是：`shared_from_this()`是通过一个对自己的 `weak_ptr`的引用来返回this的`shared_ptr`的，所以在返回智能指针的时候，你要确保对象已经构造完成。所以你不能在对象的构造函数里面调用`shared_from_this()`。

##智能指针的比较
可以拿两个智能指针来做比较，从源码上来看，本质上它是直接拿原始指针来做比较的。

如果我们有两个智能指针ptr_a和ptr_b:
	std::tr1::shared_ptr<T> ptr_a(new T());
	std::tr1::shared_ptr<T> ptr_b(new T());
那么下面的指针比较
	ptr_a == ptr_b;
本质上其实是
	ptr_a.get() == ptr_b.get()
##std::tr1::shared_ptr到底是什么？ 
`shared_ptr`本质上持有两个东西：一个是对象的原始指针`T*`，所以你可以通过`get()`函数获取到。另外它还持有一个全局的计数器`aux*`，它通过对象的拷贝构造函数、析构函数来对引用计数进行加减。

下面是一个不完全的类定义，帮助理解
{% codeblock shared_ptr lang:c %}
	template<class T>
	class shared_ptr
	{
    	struct aux
    	{
        	unsigned count;

        	aux() :count(1) {}
        	virtual void destroy()=0;
        	virtual ~aux() {} //must be polymorphic
    	};

    	template<class U, class Deleter>
    	struct auximpl: public aux
    	{
        	U* p;
        	Deleter d;

        	anximpl(U* pu, Deleter x) :p(pu), d(x) {}
        	virtual void destroy() { d(p); } 
    	};
    
    	template<class U>
    	struct default_deleter
    	{
        	void operator()(U* p) const { delete p; }
    	};

    	aux* pa;
    	T* pt;

    	void inc() { if(pa) interloked_inc(pa->count); }

    	void dec() 
    	{ 
        	if(pa && !interlocked_dec(pa->count)) 
        	{  pa->destroy(); delete pa; }
    	}
    public:

    	shared_ptr() :pa(), pt() {}

    	template<class U, class Deleter>
    	shared_ptr(U* pu, Deleter d) :pa(new auximpl<U,Deleter>(pu,d)) {}

    	template<class U>
    	explicit shared_ptr(U* pu) :pa(new auximpl<U,default_deleter<U> >(pu,default_deleter<U>())) {}

    	shared_ptr(const shared_ptr& s) :pa(s.pa), pt(s.pt) { inc(); }

    	template<class U>
    	shared_ptr(const shared_ptr<U>& s) :pa(s.pa), pt(s.pt) { inc(); }

    	~shared_ptr() { dec(); }
		shared_ptr& operator=(const shared_ptr& s)
    	{
        	if(thsi!=&s)
        	{
            	dec();
            	pa = s.pa; pt=s.pt;
            	inc();
        	}        
        	return *this;
    	}

    	T* operator->() const { return p; }
    	T& operator*() const { return *p; }
	};
{% endcodeblock %}	
***
基本上这次引入`std::tr1::shared_ptr`所用到的知识点就是以上了，如有纰漏，敬请指正。