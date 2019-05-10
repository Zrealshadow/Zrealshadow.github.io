---
layout: post
title: "利用Google Test进行测试"
author: "Zreal"
catalog: true
header-img: "img/roman-post.jpg"
tags:
  - C++
  - Testing
---



# Sofeware Testing Experient

>Author: Zreal	曾令泽
>
>Student ID：1120162062
>
>Platform：MAC OS  / Clion
>
>Testing Framework：Google Test




## Requirement

要求编写一个复数类，要求有4条。一是有构造函数，能对复数初始化。二是对复数c1，c2，c3.....能实现连加运算，令c=c1+c2+c3+.....此处可以重载加法操作符。三是有函数实现两个复数相加，并按照a+ib的形式输出。四是能实现对一个复数c=a+ib，定义double x=c有效，使x的值为实部和虚部之和。

**summary:**

- 测试1：构造函数初始化功能
- 测试2：连加功能，及重载假发操作符是否成功
- 测试3：复数相加后输出形式是否正确
- 测试4：对实部和虚部相加功能



## Source code

>测试源码进行了修改，因为测试规模较小，所以没有将gtest作为一个子模块嵌入到项目里。而是将该complex 作为一个类，集成到gTest的项目里面。

complex.h

```c++
//
// Created by e1ixir on 2019/5/9.
//

#ifndef TEST2_COMPLEX_H
#define TEST2_COMPLEX_H

#include "string"
class complex {
private:
    int a,b;
public:
    complex(){};
    complex(int aa,int bb):a(aa),b(bb){};
    complex operator+ (complex c);
    void add_c(complex x1);
    double add();
    int geta(){return a;};
    int getb(){return b;};

    std::string add_c_s(complex x1);
};


#endif //TEST2_COMPLEX_H

```



complex.cpp

```c++
//
// Created by e1ixir on 2019/5/9.
//
#include "iostream"
#include "complex.h"
complex complex::operator+(complex c) {
    c.a=c.a+a;
    c.b=c.b+b;
    return c;
}

void complex::add_c(complex x1){
    std::cout<<x1.geta()+a<<"+"<<x1.getb()+b<<"i"<<std::endl;
}

std::string complex::add_c_s(complex x1) {
    std::string ans=std::to_string(x1.geta()+a)+"+"+std::to_string(x1.getb()+b)+"i";
    return ans;
}
// 为了测试R3，人为添加了 add_c_s方法，返回宇哥string。判断string，来确定R3。

double complex::add(){
    double c=a+b;
    return c;
}
```





## EXTRA INFO: how to use gtest in Clion

参考：

https://hacpai.com/article/1515035925270

https://blog.csdn.net/cckooo/article/details/79624258



**Step one：** **Download**

下载gtest文件，将其复制到project目录下

**Step two: rewrite CMakeList.txt**

添加下列语句

```makefile
add_subdirectory(./googletest)
include_directories(googletest/googletest/include googletest/googletest)
link_directories(./googletest)
set(LIBRARIES
        gtest
        pthread)
target_link_libraries(test2 ${LIBRARIES})
```

配置好后可以看到clion里有执行按钮

![img](/img/assets/image-20190510120318450.png)

可以点击执行进行单元测试



**Step three: model of testing**

```c++
#include "gtest/gtest.h"

int add(int a, int b){
    return a+b;
}

TEST(test1, c1){
EXPECT_EQ(3, add(1,2));
}

GTEST_API_ int main(int argc, char** argv){
    testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

具体API 可参考：

https://github.com/google/googletest/blob/master/googletest/docs/primer.md

常用

```c++
TEST(TestSuitName,TestName){
}
//一个用例单元

EXPECT_EQ(EXPECT_RESULT, ACTUAL_RESULT)
//判断输出和预计输出是否相等

GTEST_API_ int main(int argc, char** argv){
    testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
//Main 函数执行 unit test

```



## Testing Process

**Requiremet 1**

测试代码：

```c++
void test_init(int a,int b){
    complex t= complex(a,b);
    EXPECT_EQ(a,t.geta())<<"test a";
    EXPECT_EQ(b,t.getb())<<"test b"<<std::endl<<"*********************";
}


TEST(test1,init_the_class){
    test_init(1,2);
    test_init(12,12);

    test_init(2,0);
    test_init(10,0);

    test_init(0,12);
    test_init(0,3);

    test_init(0,0);

    test_init(-1,-1);
    test_init(-1,0);
    test_init(0,-1);
    test_init(-1,12);
    test_init(12,-1);
}
```

思路：

测试构造函数，及看在构造函数传入的参数是否赋值给了属性，用geta()及getb() 查看内部属性是否和构造时传入的参数相同。

测试用例选择：

正数，0，负数，和 实部和虚部 排列组合

测试结果如下：

![img](/img/assets/image-20190510105425310.png)



**Requirement 2**

测试代码：

```c++
void test_add(vector<complex> a,complex ans){
    vector<complex>::iterator it=a.begin();
    complex ans_c=complex(0,0);
    for(;it!=a.end();it++){
        ans_c=ans_c+(*it);
    }
    EXPECT_EQ(ans.geta(),ans_c.geta());
    EXPECT_EQ(ans.getb(),ans_c.getb());
}

TEST(test1,add_the_class){
    vector<complex> a{complex(0,1),complex(1,2),complex(3,4),complex(8,0)};
    test_add(a,complex(12,7));

    vector<complex> b{complex(0,12),complex(-3,-10),complex(-2,-1),complex(1,-12)};
    test_add(b,complex(-4,-11));

    vector<complex> c{complex(0,0)};
    test_add(c,complex(0,0));
}
```

思路：

传入 vector\<complex\> 和 预期输出的complex。在test_add函数里，连加vector里的complex，比较complex 属性 a，b；

用例构造：

正数/负数/0

测试结果：

![img](/img/assets/image-20190510110444273.png)



**Requirement 3**

因为该要求的方法没有输出，因此在complex 源码里面增加了add_c_s()方法，返回和打印方法相同的string，判断string 进行测试

add_c_s

```c++
std::string complex::add_c_s(complex x1) {
    std::string ans=std::to_string(x1.geta()+a)+"+"+std::to_string(x1.getb()+b)+"i";
    return ans;
}
```

测试：

```c++
TEST(test1,add_display_the_class){
    complex a=complex(0,0);
    EXPECT_EQ("1+2i",a.add_c_s(complex(1,2)))<<a.add_c_s(complex(1,2));
    EXPECT_EQ("1-2i",a.add_c_s(complex(1,-2)));
    EXPECT_EQ("-10-20i",a.add_c_s(complex(-10,-20)));
    EXPECT_EQ("0",a.add_c_s(complex(0,0)));
    EXPECT_EQ("-1+2i",a.add_c_s(complex(-1,2)));
}
```

测试样例：

正数，0，负数，和 实部和虚部 排列组合

测试结果：

![img](/img/assets/image-20190510112023617.png)



可以看到出现错误，当虚部出现负数或0时，打印的结果出现错误。

分析源码

```c++
void complex::add_c(complex x1){
    std::cout<<x1.geta()+a<<"+"<<x1.getb()+b<<"i"<<std::endl;
}
```

没有考虑到虚部是负数或0的情况，直接打印了"+"；



**Error：**

在Gtest文档中，判断字符串char[]类型的会用`EXPECT_STREQ`，但是如果用该API判断std::string会出现如下Error：

![img](/img/assets/image-20190510113812373.png)

解决方法参考：https://stackoverflow.com/questions/43763615/windows-gtest-expect-streq-error-no-matching-function-for-call-to-cmphelperst

**EXPECT_STREQ 只用来比较 raw c strings(char *) ,std::string 应用EXPECT_EQ进行比较**



**Requirement 4**

测试代码

```c++
TEST(test1,add){
    complex a=complex(1,2);
    EXPECT_DOUBLE_EQ(3.0,a.add());
    complex b=complex(10,-10);
    EXPECT_DOUBLE_EQ(0,b.add());
    complex c=complex(-10,-10);
    EXPECT_DOUBLE_EQ(-20.0,c.add());
}
```

测试用例设计：

正负0

测试结果：

![img](/img/assets/image-20190510113134009.png)

通过测试
