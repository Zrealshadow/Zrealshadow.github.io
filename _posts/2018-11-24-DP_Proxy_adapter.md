---
layout: post
title: "设计模式——模式介绍 Part two"
subtitle: "代理模式 适配器模式"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - Design Pattern
---
# 设计模式---代理，适配器

> Author：Zreal

> 参考https://design-patterns.readthedocs.io/zh_CN/latest/index.html 图说设计模式



## 代理模式(Proxy Pattern)

**定义**

给某一个对象提供一个代理，并由代理对象控制对原对象的引用。代理模式的英 文叫做Proxy或Surrogate，它是一种对象结构型模式



类图：

![img](/img/assets/1543055539219.png)

时序图：

![img](/img/assets/1543055662537.png)

Subject:抽象主题角色

Proxy：代理主题角色

RealSubject：真实主题角色

**代码分析**

```c++
class Proxy : public Subject
{

public:
	Proxy();
	virtual ~Proxy();
	//一个共有的请求功能的函数
	void request();

private:
	void afterRequest();
	void preRequest();
    //私有的连接的真正的对象
	RealSubject *m_pRealSubject;

};
```

```c++
//Proxy类型的实现
Proxy::Proxy(){
	//有人觉得 RealSubject对象的创建应该是在main中实现；我认为RealSubject应该
	//对用户是透明的，用户所面对的接口都是通过代理的；这样才是真正的代理； 
	m_pRealSubject = new RealSubject();
}

Proxy::~Proxy(){
	delete m_pRealSubject;
}

void Proxy::afterRequest(){
	cout << "Proxy::afterRequest" << endl;
}


void Proxy::preRequest(){
	cout << "Proxy::preRequest" << endl;
}


void Proxy::request(){
	preRequest();
	m_pRealSubject->request();
	afterRequest();
}
```



**优点**

- 代理模式能够协调调用者和被调用者，在一定程度上降低了系 统的耦合度。
- 远程代理使得客户端可以访问在远程机器上的对象，远程机器 可能具有更好的计算性能与处理速度，可以快速响应并处理客户端请求。
- 虚拟代理通过使用一个小对象来代表一个大对象，可以减少系 统资源的消耗，对系统进行优化并提高运行速度。
- 保护代理可以控制对真实对象的使用权限。



**缺点**

- 由于在客户端和真实主题之间增加了代理对象，因此 有些类型的代理模式可能会造成请求的处理速度变慢。
- 实现代理模式需要额外的工作，有些代理模式的实现 非常复杂。



**试用环境**

- 远程(Remote)代理：为一个位于不同的地址空间的对象提供一个本地 的代理对象，这个不同的地址空间可以是在同一台主机中，也可是在 另一台主机中，远程代理又叫做大使(Ambassador)。
- ***虚拟(Virtual)代理：如果需要创建一个资源消耗较大的对象，先创建一个消耗相对较小的对象来表示，真实对象只在需要时才会被真正创建。***（很多网页上的缩略图就是这种模式）
- Copy-on-Write代理：它是虚拟代理的一种，把复制（克隆）操作延迟 到只有在客户端真正需要时才执行。一般来说，对象的深克隆是一个 开销较大的操作，Copy-on-Write代理可以让这个操作延迟，只有对象被用到的时候才被克隆。
- 保护(Protect or Access)代理：控制对一个对象的访问，可以给不同的用户提供不同级别的使用权限。
- 缓冲(Cache)代理：为某一个目标操作的结果提供临时的存储空间，以便多个客户端可以共享这些结果。
- 防火墙(Firewall)代理：保护目标不让恶意用户接近。
- 同步化(Synchronization)代理：使几个用户能够同时使用一个对象而没有冲突。
- 智能引用(Smart Reference)代理：当一个对象被引用时，提供一些额外的操作，如将此对象被调用的次数记录下来等。



## 适配器模式（Adapter Pattern)

个人感受：该模式就是在这样的一个场景下，例如 client 现在要调用一个request 的API ，要实现specialrequest（）的功能，现在就需要对能实现specialrequest的类进行一个包装，这个包装之后的类就是适配器，他的作用就是使客户端再调用request时能达到specialrequest的效果。

**定义**

适配器模式(Adapter Pattern) ：将一个接口转换成客户希望的另一个接口，适配器模式使接口不兼容的那些类可以一起工作，其别名为包装器(Wrapper)。适配器模式既可以作为类结构型模式，也可以作为对象结构型模式。

类图：

对象适配器

![img](/img/assets/1543057243940.png)

时序图：

![img](/img/assets/1543057291499.png)

```c++
class Adapter : public Target
{

public:
	Adapter(Adaptee *adaptee);
	virtual ~Adapter();

	virtual void request();

private:
	//指向真正功能的对象
	Adaptee* m_pAdaptee;
	

};
```

```c++
#include "Adapter.h"

Adapter::Adapter(Adaptee * adaptee){
	m_pAdaptee =  adaptee;
}

Adapter::~Adapter(){

}
//可以看到，request接口其实就是实现specificRequest的功能
void Adapter::request(){
	m_pAdaptee->specificRequest();
}
```

```c++
class Adaptee
{

public:
	Adaptee();
	virtual ~Adaptee();

	void specificRequest();

};
```

程序执行顺序

```c++
int main(int argc, char *argv[])
{
    //先声明具有specificRequest功能的对象 adaptee
	Adaptee * adaptee  = new Adaptee();
    //创建一个关联adaptee对象的Adapter对象
	Target * tar = new Adapter(adaptee);
    //调用tar 的request 及调用 adaptee的specificRequest功能
	tar->request();
	
	return 0;
}
```
