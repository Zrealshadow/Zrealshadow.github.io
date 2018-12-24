---
layout: post
title: "windows下多线程实践"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - C++
  - OS
---
# 生产者消费者多线程实现



## 实验要求

 1. 可以随机设置生产者进程、消费者进程，以及缓冲区的数量；其中生产者进程、消费者进程，以及缓冲区的数量均不超过20；并且生产者进程、消费者进程、缓冲区均从0开始编号；程序中可以任意指定一种方式向缓冲区投放或取出产品，比如生产者向缓冲区以单元号循环方式投放产品。

​    2.随机输入生产产品的数量，能够输出正确的生产与消费过程，时间限制1秒（输入输出信息见测试用例格式说明）。



## 思路

​	分析题意，虽然说实验要求是设置不同的进程，但是分析可发现，缓冲区每次只能让同一个进程进入，因此每个进程之间是并发执行的，而且对于每一个生产者消费者，内存消耗并不大，所以在该实验中，决定不用fork（）创造新的进程，二用thread创造线程来模拟多进程的效果。

## 数据结构

```c++
#include "stdafx.h"
#include "windows.h"
#include <stdio.h>
#include <iostream>
#include <time.h>
#include <stdlib.h>
HANDLE Empty, full, mutex,P_NUM,S_NUM;
//Empty和full为进程之间的共享变量，mutex为进程间的互斥变量，P_NUM和S_NUM都是对于消费总数和生产总数的互斥对象。
#define N 20
int buffer[N+5];
int f_P = 0,f_C=0;
//生产者们已经生产的产品总数和消费者们已经消费的产品总数
int Pnum, Cnum, buffer_size, Dnum;
//生产者的序号，消费者的序号，缓冲区的大小，最大生产总数

```

​	首先要求要创建新的线程。因此要调用操作系统的API接口即windows.h这个库。



**main（）函数**

​	main函数中

1.首先从argv里面提取出需要的参数生产者个数，消费者个数，缓冲区大小，和最大生产数。

2.然后对Empty，full，mutex，调用CreateSemaphore API创建共享和互斥变量。

3.声明DWORD的动态数组和一个handle的动态数组储存之后要创建线程的id和线程的句柄。生产者个数+消费者个数=创建新线程的个数。

4.然后利用循环调用CreateThread API依次创建新的线程。等到所有进程结束销毁线程，程序结束。

```c++
int main(int argc,char *argv[])
{
	Pnum = trans(argv[1]);
	Cnum = trans(argv[2]);
	buffer_size = trans(argv[3]);
	Dnum = trans(argv[4]);
    S_NUM=CreateSemaphore(NULL,1,1,NULL);
    P_NUM=CreateSemaphore(NULL,1,1,NULL);
	Empty = CreateSemaphore(NULL, buffer_size, buffer_size, NULL);
	full = CreateSemaphore(NULL, 0, buffer_size, NULL);
	mutex = CreateSemaphore(NULL, 1, 1, NULL);
	DWORD *ThreadId = (DWORD *)malloc((Pnum + Cnum + 5) * sizeof(DWORD*));
	HANDLE*ThreadHandle =(HANDLE*)malloc((Pnum + Cnum + 5) * sizeof(HANDLE*));

	for (int i = 0; i < Pnum; i++) {
		ThreadHandle[i] = CreateThread(NULL, 0, produce, NULL, 0, &ThreadId[i]);
	}
	for (int i = 0; i < Cnum; i++) {
		ThreadHandle[i + Pnum] = CreateThread(NULL, 0, consumer, NULL, 0, &ThreadId[i+Pnum]);
	}

	WaitForMultipleObjects(Pnum+Cnum, ThreadHandle, TRUE, INFINITE);
    return 0;
}

```



**DWORD WINAPI produce(LPVOID param)**

**DWORD WINAPI consumer(LPVOID param)**

​	对于一个生产者和消费者来说，其思路还是比较简单

​	1.申请F_NUM或P_NUM的访问权限，是否总生产数符合要求，符合继续生产，否则结束线程

​	2.释放对于F_NUM或P_NUM的访问权限

​	3.首先申请共享变量empty的访问权限

​	4.再申请互斥变量的访问权限

​	5.申请完全部的访问权限后，开始对缓冲区进行操作

​	6.操作完成后一次释放已经申请的权限

​	7.重复步骤1

```c++
DWORD WINAPI produce(LPVOID param) {
	srand((unsigned)time(NULL));
	while (1) {
	    WaitForSingleObject(P_NUM,INFINITE);
		if (f_P >= Dnum)
			break;
		ReleaseSemphore(P_NUM,1,NULL);
		WaitForSingleObject(Empty, INFINITE);
		WaitForSingleObject(mutex, INFINITE);
		int D = rand() % 10;
		buffer[f_P % buffer_size] = D;
		cout << "P," << GetCurrentThreadId() << ",B," << f_P%buffer_size << ",D" << D << endl;
        WaitForSingleObject(P_NUM,INFINITE);
		f_P++;
        ReleaseSemphore(P_NUM,1,NULL);
		ReleaseSemaphore(mutex, 1, NULL);
		ReleaseSemaphore(full, 1, NULL);
		cout << "P,end" << endl;
		Sleep(1);
	}
	return 0;
}

DWORD WINAPI consumer(LPVOID param) {
	while (1) {
        WaitForSingleObject(F_NUM,INFINITE);
        if (f_C >= Dnum)
            break;
        ReleaseSemphore(F_NUM,1,NULL);
		WaitForSingleObject(full, INFINITE);
		WaitForSingleObject(mutex, INFINITE);
		int D = buffer[f_C % buffer_size];
		buffer[f_C % buffer_size] = -1;
		cout << "C," << GetCurrentThreadId() << ",B," << f_C%buffer_size << ",D" << D << endl;
        WaitForSingleObject(F_NUM,INFINITE);
        f_C++;
        ReleaseSemphore(F_NUM,1,NULL);
		ReleaseSemaphore(mutex, 1, NULL);
		ReleaseSemaphore(Empty, 1, NULL);
		cout << "C,end" << endl;
		Sleep(1);
	}
	return 0;
}
```



## 完整源码及注释

```c++
// 生产者消费者问题.cpp : 定义控制台应用程序的入口点。
//

#include "stdafx.h"
#include "windows.h"
#include <stdio.h>
#include <iostream>
#include <time.h>
#include <stdlib.h>
using namespace std;
HANDLE Empty, full, mutex,P_NUM,S_NUM;
//Empty和full为进程之间的共享变量，mutex为进程间的互斥变量，P_NUM和S_NUM都是对于消费总数和生产总数的互斥对象。
#define N 20
int buffer[N+5];
int f_P = 0,f_C=0;
//生产者们已经生产的产品总数和消费者们已经消费的产品总数
int Pnum, Cnum, buffer_size, Dnum;
//生产者的序号，消费者的序号，缓冲区的大小，最大生产总数
DWORD WINAPI produce(LPVOID param) {
	srand((unsigned)time(NULL));
	while (1) {
	    WaitForSingleObject(P_NUM,INFINITE);
		if (f_P >= Dnum)
			break;
		ReleaseSemphore(P_NUM,1,NULL);
		WaitForSingleObject(Empty, INFINITE);
		WaitForSingleObject(mutex, INFINITE);
		int D = rand() % 10;
		buffer[f_P % buffer_size] = D;
		cout << "P," << GetCurrentThreadId() << ",B," << f_P%buffer_size << ",D" << D << endl;
        WaitForSingleObject(P_NUM,INFINITE);
		f_P++;
        ReleaseSemphore(P_NUM,1,NULL);
		ReleaseSemaphore(mutex, 1, NULL);
		ReleaseSemaphore(full, 1, NULL);
		cout << "P,end" << endl;
		Sleep(1);
	}
	return 0;
}

DWORD WINAPI consumer(LPVOID param) {
	while (1) {
        WaitForSingleObject(F_NUM,INFINITE);
        if (f_C >= Dnum)
            break;
        ReleaseSemphore(F_NUM,1,NULL);
		WaitForSingleObject(full, INFINITE);
		WaitForSingleObject(mutex, INFINITE);
		int D = buffer[f_C % buffer_size];
		buffer[f_C % buffer_size] = -1;
		cout << "C," << GetCurrentThreadId() << ",B," << f_C%buffer_size << ",D" << D << endl;
        WaitForSingleObject(F_NUM,INFINITE);
        f_C++;
        ReleaseSemphore(F_NUM,1,NULL);
		ReleaseSemaphore(mutex, 1, NULL);
		ReleaseSemaphore(Empty, 1, NULL);
		cout << "C,end" << endl;
		Sleep(1);
	}
	return 0;
}

int trans(char *c) {
	int l = strlen(c);
	int sum = 0;
	for (int i = 0; i < l; i++) {
		sum = sum * 10 + c[i] - '0';
	}
	return sum;
}

int main(int argc,char *argv[])
{
	Pnum = trans(argv[1]);
	Cnum = trans(argv[2]);
	buffer_size = trans(argv[3]);
	Dnum = trans(argv[4]);
    S_NUM=CreateSemaphore(NULL,1,1,NULL);
    P_NUM=CreateSemaphore(NULL,1,1,NULL);
	Empty = CreateSemaphore(NULL, buffer_size, buffer_size, NULL);
	full = CreateSemaphore(NULL, 0, buffer_size, NULL);
	mutex = CreateSemaphore(NULL, 1, 1, NULL);
	DWORD *ThreadId = (DWORD *)malloc((Pnum + Cnum + 5) * sizeof(DWORD*));
	HANDLE*ThreadHandle =(HANDLE*)malloc((Pnum + Cnum + 5) * sizeof(HANDLE*));

	for (int i = 0; i < Pnum; i++) {
		ThreadHandle[i] = CreateThread(NULL, 0, produce, NULL, 0, &ThreadId[i]);
	}
	for (int i = 0; i < Cnum; i++) {
		ThreadHandle[i + Pnum] = CreateThread(NULL, 0, consumer, NULL, 0, &ThreadId[i+Pnum]);
	}

	WaitForMultipleObjects(Pnum+Cnum, ThreadHandle, TRUE, INFINITE);
    return 0;
}


```

