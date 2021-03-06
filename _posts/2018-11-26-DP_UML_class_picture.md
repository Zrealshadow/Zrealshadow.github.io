---
layout: post
title: "设计模式——UML类图介绍"
subtitle: "泛化,实现,聚合,组合,关联,依赖"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - Design Pattern
---

# UML类图解读

>Author：Zreal
>参考https://design-patterns.readthedocs.io/zh_CN/latest/index.html 图说设计模式

主要是讲解类和类的关系，，，就是不同箭头的使用QAQ，真的很烦人

下面的的概念都已这张图为例子

![img](/img/assets/1543059259115.png)



### 类之间的关系

**泛化关系（generalization）**

继承关系是is-a的关系，如果两个对象之间可以用is-a来表事，就是继承关系。

eg：SUV是小汽车

用空心箭头实线表示：

![img](/img/assets/1543059457450.png)

泛化关系：表现为继承**非抽象类**



##### 实现关系（realize）

eg：”车“是一个抽象概念，在现实中并无法直接用来定义对象；只有指明具体的子类(汽车还是自行车)，才 可以用来定义对象

用空心箭头虚线表示：

![img](/img/assets/1543059718259.png)

实现关系：表现为继承**抽象类**



##### 聚合关系（aggregation）

聚合关系用于表示实体对象之间的关系，表示整体由部分构成的语义；例如一个部门由多个员工组成；

与组合关系不同的是，**整体和部分不是强依赖的**，即使整体不存在了，部分仍然存在；例如， 部门撤销了，人员不会消失，他们依然存在；

用空心菱形箭头表示

![img](/img/assets/1543059889514.png)



##### 组合关系（composition）

与聚合关系一样，组合关系同样表示整体由部分构成的语义；比如公司由多个部门组成；

但组合关系是**一种强依赖的特殊聚合关系**，如果整体不存在了，则部分也不存在了；例如， 公司不存在了，部门也将不存在了；

用实心菱形箭头表示：

![img](/img/assets/1543059995240.png)

##### 关联关系（association）

关联关系是用一条直线表示的；它描述不同类的对象之间的结构关系；它是一种静态关系， 通常与运行状态无关，一般由常识等因素决定的；它一般用来定义对象之间静态的、天然的结构； 所以，关联关系是一种“强关联”的关系；

比如，乘车人和车票之间就是一种关联关系；学生和学校就是一种关联关系；

关联关系默认不强调方向，表示对象间相互知道；如果特别强调方向，如下图，表示A知道B，但 B不知道A；

![img](/img/assets/1543060162043.png)

**在最终代码中，关联对象通常是以成员变量的形式实现的；**



##### 依赖关系（dependency）

与关联关系不同的是，它是一种临时性的关系，通常在运行期间产生，并且随着运行时的变化； 依赖关系也可能发生变化；

显然，依赖也有方向，双向依赖是一种非常糟糕的结构，我们总是应该保持单向依赖，杜绝双向依赖的产生；

依赖关系是用一套带箭头的虚线表示的；如下图表示A依赖于B；他描述一个对象在运行期间会用到另一个对象的关系

![img](/img/assets/1543060391585.png)



在最终代码中，依赖关系体现为**类构造方法及类方法的传入参数，箭头的指向为调用关系**；依赖关系除了临时知道对方外，还是“使用”对方的方法和属性；
