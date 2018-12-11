---
layout: post
title: "单片机进程调度"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - C++
  - OS
---
# 单处理机进程调度

## 要求

>1. 可以随机输入若干进程，支持先来先服务、短作业优先、最短剩余时间优先、时间片轮转、动态优先级调度算法，能够输出进程的调度过程。具体信息见测试用例格式说明。
>2. 每个进程由一个进程控制块表示。
>3. 实现先来先服务调度算法：进程到达时间可由进程创建时间表示。进程到达时间相同时，优先处理进程号小的进程。
>4. 实现短作业优先调度算法：可指定进程要求的运行时间。进程运行时间相同时，按照先来先服务原则进行处理。
>5. 实现最短剩余时间优先调度算法：可指定进程要求的运行时间。进程运行时间相同时，按照先来先服务原则进行处理。
>6. 实现时间片轮转调度算法：可指定生成时间片大小。进程到达时间相同时，优先处理进程号小的进程；进程执行完一个时间片进入就绪队列时，其优先级低于首次进入就绪队列的进程。
>7. 实现动态优先级调度算法：可指定进程的初始优先级（优先级与优先数成反比，优先级最高为0），优先级改变遵循下列原则：进程在就绪队列中每停留一个时间片（停留时间>0），优先级加1，进程每运行一个时间片，优先级减3。进程到达时间相同时，优先处理进程号小的进程，且仅在时间片完或进程运行结束时发生进程调度。

## 数据结构及算法分析

设置一个结构体用来储存进程的属性,该结构体代表进程控制块。

包括进程号，进程到达的时间，进程结束的时间，进程持续时间，进程优先级，进程时间片

```c++
struct process {
	int num;
	int arr_time;
	int end_time;
	int ltime;
	int lv;
	int dt;
};
```



​	对于每一个调度算法，我们应该明确的是，每一种算法都是在操纵一个进程的**等待队列**，这个队列不仅仅是先进先出的那种数据结构的队列，对于每一个新的进程到来后，都要依据某种规则对队列进行维护。这种规则就是每个调度算法所不同的地方。大体解答的思路如下：

​	首先用一个vector vec1（**进入队列**）来记录所有的即将到来的进程，按进程的到达时间进行排序，然后以时刻t为循环变量，进行循环。

​	首先遍历vec1，看是否到达时间<=t，如果满足，将其加入等待队列，如果不满足遍历终止。

​	对于时刻t来说，已经将到达的进程加入了等待队列，按照规则维护一遍等待队列，在选出队列头的元素，运行该进程。同时时刻也从t变为t+dt（dt为进程的运行时长或时间片长度）。

​	循环上述步骤，直至等待队列和vec1都为空时，终止循环。

#### 先来先服务算法（FIFO）

​	因为先来先服务算法比较简单，所以这里用了一点小的trick。将**进入队列和等待队列进行了合并**，因为按照先来先服务调度算法的规则，进入队列和等待队列的都按照进程的到达时间进行了排序。然后按照该队列直接进行输出即可。这里我也没有将时刻t设置成循环变量，而是直接对排好序的队列进行遍历，输出。

```c++
/*先来先服务算法 排序函数*/
bool cmp_FCFS(struct process a,struct process b) {
	if (a.arr_time < b.arr_time) {
		return true;
	}
	else if (a.arr_time == b.arr_time) {
		return a.num < b.num;
	}
	else {
		return false;
	}
	
}

/*先来先服务算法*/
void FCFS(vector<struct process> map) {
	sort(map.begin(), map.end(), cmp_FCFS);
	vector<struct process>::iterator it=map.begin();
	int begin_time=0, end_time=0;
	for (int i = 1;it != map.end(); it++, i++) {
		struct process onep = *it;
		begin_time = end_time > onep.arr_time ? end_time : onep.arr_time;
		end_time = begin_time + onep.ltime;
		printf("%d/%d/%d/%d/%d\n", i, onep.num, begin_time, end_time,onep.lv);
	}
}
```



#### 短作业优先算法（SJF)

​	对于SJF，我也没有完全按照整体思路进行操作。

​	首先，同样是对进入队列vec1按照进程的到达时间进行排序，以时刻t作为循环因子，当时刻t时，vec1的前n个进程，这里我定义了两个迭代器，一个it指向当前我要工作的进程，f_it指向在当前时刻下已经进入等待队列里的最后一个进程。这里我将进入队列和等待队列在放在了同一个vec里面，f_it之前的为等待队列，f_it之后的为还未进入等待队列的进程。而it到f_it之间为按作业长度排序的等待队列，f_it之后为按照进程到达时间排序的进入队列。

​	 每次更新了t时刻后，首先移动f_it更新等待队列，然后对其进行排序，维护等待队列。选取队列中的第一个进程进行操作。

代码如下：

```c++
/*短作业优先算法，排序函数*/
bool cmp_SJF_pure(struct process a, struct process b) {
	return a.ltime < b.ltime;
}
/*等待队列的维护*/
bool cmp_SJF_init(struct process a, struct process b) {
	if (a.arr_time < b.arr_time)
		return true;
	else if (a.arr_time == b.arr_time) {
		return a.ltime < b.ltime;
	}
	else
		return false;
}

/*短作业优先算法*/
void SJF(vector<struct process> map) {
	sort(map.begin(), map.end(), cmp_SJF_init);
	vector<struct process>::iterator it = map.begin();
	int cnt = 0;
	for (int t = 0; it != map.end();) {
		vector<struct process>::iterator f_it = it;
		if ((*f_it).arr_time <= t) {
			while (f_it != map.end()&&(*f_it).arr_time <= t) {
				f_it++;
			}
			stable_sort(it, f_it, cmp_SJF_pure);
			struct process onep = *it;
			cnt++;
			printf("%d/%d/%d/%d/%d\n", cnt, onep.num, t, t + onep.ltime, onep.lv);
			t += onep.ltime;
			it++;
		}
		else {
			t++;
		}
	}
}
```



#### 最短剩余时间调度算法（SRTF）

​	首先要注意的一点是：**SRTF算法是可剥夺的**

​	因此作为循环变量的时刻t，每一个时刻我们都要对其进行判断和维护，看在这个时刻内进程是否被剥夺。

思路：

​	在这个算法中我依旧是将进入队列和等待队列放在一个vec1里面，利用不同的cmp函数进行维护。

​	首先判断t时刻下，有哪些进程进入了等待队列，对等待队列vec_flag_end_vec_flag进行按时间长度进行排序。然后找出第一个进程，为该长度为1的时间段内操作的进程。我们将该进程时间长度-1，然后加入队尾。同时用一个新的vec2来记录在每个长度为1的时间段内操作的进程的信息。最后在对vec1，按照达到顺序进行重新排序。

​	t++，对下一个时刻重复上述步骤，直到vec1和等待队列空，跳出循环

​	最后对vec2的结果进行整理，例如连续操作同样进程被分成了两次输出，

>​	刚开始是为了省一个队列内存空间，最后整个实现下来，感觉思路复杂化了，应该老实将进入队列和等待队列分开，这样可以少一次对于整体的sort操作。我这种做法，性能上并没有提高，反而思路不清晰，很容易出现错误。

代码如下：

```c++
bool cmp_SRTF_pure(struct process a, struct process b) {
	return a.ltime < b.ltime;
}
bool cmp_SRTF_init(struct process a, struct process b) {
	return cmp_SJF_init(a, b);
}
/*最短剩余时间优先调度算法*/
void SRTF(vector<struct process> map) {
	sort(map.begin(), map.end(), cmp_SRTF_init);
	int size = map.size();
	vector<struct process> ff(0);
	int f = -1;
	int vec_flag = 0;
	for (int t = 0; vec_flag!=size;t++) {
		int end_vec_flag = vec_flag;
		if (map[end_vec_flag].arr_time <= t) {
			while (end_vec_flag < size&&map[end_vec_flag].arr_time<=t) {
				end_vec_flag++;
			}

			stable_sort(map.begin()+vec_flag, map.begin()+end_vec_flag, cmp_SRTF_pure);
			struct process onep = map[vec_flag];
			struct process tp;

			tp.arr_time = t;
			tp.ltime = 1;
			tp.end_time = t + 1;
			tp.num = onep.num;
			tp.lv = onep.lv;
			if (f == -1) {
				ff.push_back(tp);
				f++;
			}
			else {
				if (ff[f].num == tp.num) {
					ff[f].ltime++;
					ff[f].end_time++;
				}
				else {
					ff.push_back(tp);
					f++;
				}
			}
			
			vec_flag++;
			onep.ltime--;
			if (onep.ltime != 0) {
				map.push_back(onep);
				size++;
				sort(map.begin() + vec_flag, map.end(), cmp_SRTF_init);
			}
		}
	}
	vector<struct process>::iterator itf = ff.begin();
	for (int i = 1; itf != ff.end(); itf++, i++) {
		struct process onep = *itf;
		printf("%d/%d/%d/%d/%d\n", i, onep.num, onep.arr_time, onep.arr_time + onep.ltime, onep.lv);
	}
}
```



#### 时间片轮转算法（RR）

​	在这个调度算法中，我将两个进入队列和等待队列分开，对t进行循环叠加，将t时刻到达的进程加入等待队列，取等待队列的第一个进程运行他的一个时间片，判断如果时间片大于等于作业时长，及在一个时间片内作业完结。否则，运行一个时间片长度的时间，对进程的作业时长处理后等待队列队尾。t时刻跳转到t+dt。**在这个步骤之前还有一步，将在t-t+dt时间段内到达的进程加入队列。之后再将未处理完的进程加入等待队列队尾。**

​	时间片轮转算法在将两个队列分开后，思路和代码相对来说比较清晰

```c++
bool cmp_RR_init(struct process a, struct process b) {
	if (a.arr_time < b.arr_time)
		return true;
	else if (a.arr_time == b.arr_time)
		return a.num < b.num;
	else return false;
}
void RR(vector<struct process>map) {
	sort(map.begin(), map.end(),cmp_RR_init);
	vector<struct process> queue(0);
	int vec_flag = 0;
	int q_head = 0, q_tail = 0;
	int cnt = 0;
	for (int t = 0;;){
		//t时刻时，某些进程到达
			while (vec_flag!= map.size() && map[vec_flag].arr_time == t) {
				queue.push_back(map[vec_flag]);
				q_tail++;
				vec_flag++;
			}
		if ( q_head == q_tail) {
			t++;
		}
		else {
			struct process onep = queue[q_head++];
			if (onep.ltime > onep.dt) {
				printf("%d/%d/%d/%d/%d\n", ++cnt, onep.num, t, t + onep.dt, onep.lv);
				onep.ltime -= onep.dt;
				t += onep.dt;
				//在运行这个时间片之间，到达的进程加入队列
				while (vec_flag < map.size() && map[vec_flag].arr_time <= t) {
					queue.push_back(map[vec_flag]);
					q_tail++;
					vec_flag++;
				}
				//在将变化后的t时刻下的 剩余进程插入队列
				queue.push_back(onep);
				q_tail++;
			}
			else {
				printf("%d/%d/%d/%d/%d\n", ++cnt, onep.num, t, t + onep.ltime, onep.lv);
				t += onep.ltime;
				//在运行的这段时间内，到达的进程加入队列
				while (vec_flag < map.size() && map[vec_flag].arr_time <= t) {
					queue.push_back(map[vec_flag]);
					q_tail++;
					vec_flag++;
				}
			}
		}
		//没有进程到来，就绪队列也没有进程 及处理完毕
		if (vec_flag == map.size() && q_head == q_tail)
			break;
	}

}
```



#### 动态优先级调度算法（PRI）

​	动态优先级调度算是这几个算法中最难实现的。

​	首先也是两个队列，进入队列和等待队列，也是按照时间进行循环。

​	1.t时刻时，将t时刻达到的进程加入等待队列，对t时刻时等待队列里的进程按照优先级进行排序。选取第一个进程进行操作，操作一个时间片或者其长度。

​	2.t跳转到t+dt，在**这个时间段之间（划重点：不包含t+dt时刻到达的进程）**到达的进程加入等待队列。然后对整个等待队列进行维护，调整优先级。

​	3.如果进程作业后还剩余时间，则将其剩余的进程优先级初始化后加入等待队列。

​	4.从1开始重复上述步骤，如果等待队列为空且进入队列也为空则跳出循环。

```c++
bool cmp_PRI_init(struct process a, struct process b) {
	if (a.arr_time < b.arr_time) {
		return true;
	}
	else if (a.arr_time == b.arr_time)
		return a.num < b.num;
	else
		return false;
}
bool cmp_PRI_pure(struct process a, struct process b) {
	if (a.lv < b.lv) {
		return true;
	}
	else if (a.lv == b.lv)
		if (a.arr_time < b.arr_time)
			return true;
		else if (a.arr_time == b.arr_time)
			return a.num < b.num;
		else
			return false;
	else
		return false;
}
void PRI(vector<struct process> map) {
	sort(map.begin(), map.end(), cmp_PRI_init);
	vector<struct process>queue(0);
	int vec_flag = 0;
	int q_tail = 0, q_head = 0;
	int t = 0;
	int cnt = 0;
	do {
		//t时刻的到达队列
		while (vec_flag < map.size() && map[vec_flag].arr_time == t) {
			queue.push_back(map[vec_flag]);
			vec_flag++;
			q_tail++;
		}
		//排序，将优先级最高的排到最前
		stable_sort(queue.begin() + q_head, queue.begin() + q_tail, cmp_PRI_pure);
		//就绪队列为空
		if ( q_tail == q_head) {
			t++;
		}
		else {
			//选出要处理的进程
			struct process onep = queue[q_head++];
			if (onep.ltime > onep.dt) {
				onep.lv += 3;
				printf("%d/%d/%d/%d/%d\n", ++cnt, onep.num, t, t + onep.dt, onep.lv);
				onep.ltime -= onep.dt;
				t += onep.dt;
				
				//在t之前到达队列的进程，加入就绪队列 很关键这里没有等于号
				while (vec_flag != map.size() && map[vec_flag].arr_time <t) {
					queue.push_back(map[vec_flag]);
					vec_flag++;
					q_tail++;
				}
				//由于 在t这个时间段内，就绪队列里的进程等待了,因此要改变其优先级
				for (int i = q_head; i < q_tail; i++) {
					queue[i].lv = queue[i].lv == 0 ? 0 : queue[i].lv - 1;
				}

				//再将t时刻的完成的部分进程插入队列
				queue.push_back(onep);
				q_tail++;
			}
			else {
				printf("%d/%d/%d/%d/%d\n", ++cnt, onep.num, t, t + onep.ltime, onep.lv + 3);
				t += onep.ltime;
				//同样在onep.ltime这个时间段内（不包括右侧时间点)加入队列的进程
				while (vec_flag != map.size() && map[vec_flag].arr_time < t) {
					queue.push_back(map[vec_flag]);
					vec_flag++;
					q_tail++;
				}
				//由于 在t这个时间段内，就绪队列里的进程等待了,因此要改变其优先级
				for (int i = q_head; i < q_tail; i++) {
					queue[i].lv = queue[i].lv == 0 ? 0 : queue[i].lv - 1;
				}
			}

		}
	} while (vec_flag != map.size() || q_tail != q_head);
} 
```



最后为主函数及一些辅助函数的代码

```c++
 #include "stdlib.h"
#include "string.h"
#include "vector"
#include "algorithm"
#include "iostream"
#include "sstream"
using namespace std;
bool cmp_FCFS(struct process a, struct process b);
bool cmp_SJF_pure(struct process a, struct process b);
bool cmp_SJF_init(struct process a, struct process b);
bool cmp_SRTF_init(struct process a, struct process b);
bool cmp_SRTF_pure(struct process a, struct process b);
bool cmp_RR_init(struct process a, struct process b);
bool cmp_PRI_init(struct process a, struct process b);
bool cmp_PRI_pure(struct process a, struct process b);
void PRI(vector<struct process>map);
void RR(vector<struct process>map);
void FCFS(vector<struct process> map);
void SJF(vector<struct process> map);
void SRTF(vector<struct process> map);

vector<struct process>q(0);
//进程进入队列
/*分词*/
vector<string> split(string s, char h) {
	vector<string> elem(0);
	int j = 0;
	for (int i = 0; i < s.length(); i++) {
		if (s[i] == h) {
			string sub_s = s.substr(j, i - j);
			j = i + 1;
			elem.push_back(sub_s);
		}
	}
	string sub_s = s.substr(j, s.length() - 1);
	elem.push_back(sub_s);
	return elem;
}
int main()
{
	int c;
	scanf("%d", &c);

	string str;
	char ch=getchar();
	while (getline(cin,str)){
		char sep = '/';
		vector<string> elem = split(str,sep);
		vector<string>::iterator it = elem.begin();
		struct process p;
		vector<int> v(0);
		for (; it != elem.end(); it++) {
			string s = *it;
			int si;
			stringstream ss;
			ss << s;
			ss >> si;
			v.push_back(si);
		}
		p.num = v[0];
		p.arr_time = v[1];
		p.ltime = v[2];
		p.end_time = v[1] + v[2];
		p.lv = v[3];
		p.dt = v[4];
		q.push_back(p);
	}
	switch (c)
	{
	case 1:FCFS(q); break;
	case 2:SJF(q);  break;
	case 3:SRTF(q); break;
	case 4:RR(q); break;
	case 5:PRI(q); break;
	default:
		break;
	}
}
```





















zui
