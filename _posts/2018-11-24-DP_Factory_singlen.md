---
layout: post
title: "设计模式——模式介绍 Part one"
subtitle: "简单工厂模式，工厂模式，抽象工厂模式，单例模式"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - Design Pattern
---
# 设计模式---简单工厂，工厂，抽象工厂，单例

个人感觉设计模式虽然是概念，但是研究代码的组成可能跟容易理解

> Author：Zreal
>
> 参考https://design-patterns.readthedocs.io/zh_CN/latest/index.html 图说设计模式



## 简单工厂模式（Simple Factory Pattern）

##### 动机

考虑一个简单的软件应用场景，一个软件系统可以提供多个外观不同的按钮（如圆形按钮、矩形按钮、菱形按钮等），这些按钮都源自同一个基类，不过在继承基类后不同的子类修改了部分属性从而使得它们可以呈现不同的外观，如果我们希望在使用这些按钮时，不需要知道这些具体按钮类的名字，只需要知道表示该按钮类的一个参数，并提供一个调用方便的方法，把该参数传入方法即可返回一个相应的按钮对象，此时，就可以使用简单工厂模式

##### 定义

简单工厂模式，又称为静态工厂方法(Static Factory Method)模式，它属于类创建型模式。在简单工厂模式中，可以根据参数的不同返回不同类的实例。简单工厂模式专门定义一个类来负责创建其他类的实例，被创建的实例通常都具有共同的父类。

时序图：

![img](/img/assets/1543048421237.png)

类图：

![img](/img/assets/1543048101059.png)

Factory：工厂负责实现创建所有实例的内部逻辑

Product：抽象产品是所创建的所有对象的父类，负责描述所有实例所共有的公共接口

ConcreteProduct：具体产品是创建目标



代码很简单，该模式很好理解

**优点**

- 工厂类含有必要的判断逻辑，可以决定在什么时候创建哪一个产品类的实例，客户端可以免除直接创建产品对象的责任，而仅仅“消费”产品；简单工厂模式通过这种做法实现了对责任的分割，它提供了专门的工厂类用于创建对象。
- 客户端无须知道所创建的具体产品类的类名，只需要知道具体产品类所对应的参数即可，对于一些复杂的类名，通过简单工厂模式可以减少使用者的记忆量。
- ***通过引入配置文件，可以在不修改任何客户端代码的情况下更换和增加新的具体产品类，在一定程度上提高了系统的灵活性。***

**缺点**

- 由于工厂类集中了所有产品创建逻辑，一旦不能正常工作，整个系统都要受到影响。
- 使用简单工厂模式将会增加系统中类的个数，在一定程序上增加了系统的复杂度和理解难度。
- ***系统扩展困难，一旦添加新产品就不得不修改工厂逻辑，在产品类型较多时，有可能造成工厂逻辑过于复杂，不利于系统的扩展和维护。***
- 简单工厂模式由于使用了静态工厂方法，造成工厂角色无法形成基于继承的等级结构。**(工厂模式对其进行了改进)**





## 工厂模式（Factory Method Pattern）

和简单工厂的区别就是

对于每一个concreteProduct都有一个concreteFactory，每次声明定义一个工厂时，我们选择对应的实例工厂，这样可以符合开闭原则，每次当有新的Product加入工厂时，新建一个对应新物品的concreteFactory类，而不是修改简单工厂里的逻辑代码。

（虽然个人觉得这个方法有点扯，这个直接new一个新的concreteProduct类好像没什么区别）

类图：

![img](/img/assets/1543050012872.png)



 下面这段代码可以明显看到简单工厂模式和工厂模式的区别：

```c++
int main(int argc, char *argv[])
{
	Factory * fc = new ConcreteFactory();
	Product * prod = fc->factoryMethod();
	prod->use();
	
	delete fc;
	delete prod;
	
	return 0;
}
```

优点：和简单工厂差不多

缺点：

- 在添加新产品时，需要编写新的具体产品类，而且还要提供与之对应的具体工厂类，系统中类的个数将成对增加，在一定程度上增加了系统的复杂度，有更多的类需要编译和运行，会给系统带来一些额外的开销。
- 由于考虑到系统的可扩展性，需要引入抽象层，在客户端代码中均使用抽象层进行定义，增加了系统的抽象性和理解难度，且在实现时可能需要用到DOM、反射等技术，增加了系统的实现难度。





## 抽象工厂（Abstract Factory）

首先弄清楚**产品等级结构**和**产品族**概念：

- **产品等级结构** ：产品等级结构即产品的继承结构，如一个抽象类是电视机，其子类有海尔电视机、海信电视机、TCL电视机，则抽象电视机与具体品牌的电视机之间构成了一个产品等级结构，抽象电视机是父类，而具体品牌的电视机是其子类。
- **产品族** ：在抽象工厂模式中，产品族是指由同一个工厂生产的，位于不同产品等级结构中的一组产品，如海尔电器工厂生产的海尔电视机、海尔电冰箱，海尔电视机位于电视机产品等级结构中，海尔电冰箱位于电冰箱产品等级结构中。

这样就很清楚为什么要引入抽象工厂的概念了，很多时候，需要生产的不是一组产品等级结构，而是产品族

在抽象工厂中，一个工厂等级结构可以负责**多个**不同产品等级结构中的产品对象的创建。

**动机**

在工厂方法模式中具体工厂负责生产具体的产品，每一个具体工厂对应一种具体产品，工厂方法也具有唯一性，一般情况下，一个具体工厂中只有一个工厂方法或者一组重载的工厂方法。但是有时候我们需要一个工厂可以提供多个产品对象，而不是单一的产品对象。

类图：

![img](/img/assets/1543050463659.png)



AbstractFactory：抽象工厂类

ConcreteFactory：具体工厂

AbsrtactProduct：抽象产品

Product：具体产品

感觉上抽象工厂类优点类似，简单工厂和工厂类的结合。共产类里面通过增加返回Product对象的方法来达到同一个工厂调用无语不同产品等级结构上的产品。



**优点**

- 抽象工厂模式隔离了具体类的生成，使得客户并不需要知道什么被创建。由于这种隔离，更换一个具体工厂就变得相对容易。所有的具体工厂都实现了抽象工厂中定义的那些公共接口，因此只需改变具体工厂的实例，就可以在某种程度上改变整个软件系统的行为。另外，应用抽象工厂模式可以实现高内聚低耦合的设计目的，因此抽象工厂模式得到了广泛的应用。
- 当一个产品族中的多个对象被设计成一起工作时，它能够保证客户端始终只使用同一个产品族中的对象。这对一些需要根据当前环境来决定其行为的软件系统来说，是一种非常实用的设计模式。
- 增加新的具体工厂和产品族很方便，无须修改已有系统，符合“开闭原则”。

**确定**

- 在添加新的产品对象时，难以扩展抽象工厂来生产新种类的产品，这是因为在抽象工厂角色中规定了所有可能被创建的产品集合，要支持新种类的产品就意味着要对该接口进行扩展，而这将涉及到对抽象工厂角色及其所有子类的修改，显然会带来较大的不便。
- 开闭原则的倾斜性（增加新的工厂和产品族容易，增加新的产品等级结构麻烦）



## 单例模式（Singleton）

**动机**

对于某些类来说，只有一个实例很重要，确保该类只有一个实例

单例模式的要点有三个：

- 一是某个类只能有一个实例；
- 二是它必须自行创建这个实例；
- 三是它必须自行向整个系统提供这个实例。

单例模式是一种对象创建型模式。单例模式又名单件模式或单态模式。

贴代码感受吧，单例模式是一个容易理解的模式

```c++
//instance 指向这个类的单例，这个类不允许创建实例，对于其本身创建的对象进行了封装，并提供给全部系统访问这个对象的方法，
Singleton * Singleton::instance = NULL;
Singleton::Singleton(){

}

Singleton::~Singleton(){
	delete instance;
}

Singleton* Singleton::getInstance(){
	if (instance == NULL)
	{
		instance = new Singleton();
	}
	
	return  instance;
}


void Singleton::singletonOperation(){
	cout << "singletonOperation" << endl;
}
```

使用：

```c++
int main(int argc, char *argv[])
{
	Singleton * sg = Singleton::getInstance();
	sg->singletonOperation();
	return 0;
}
```
