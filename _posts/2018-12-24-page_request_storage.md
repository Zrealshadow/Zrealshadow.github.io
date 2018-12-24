---
layout: post
title: "页式储存中页面置换算法"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - C++
  - OS
---
# 请求页式中的页面置换算法



## 要求

 1.可以随机输入分配给一个进程的内存块数，以及该进程的页面访问序列，具体信息见测试用例格式输入部分说明。

 2.分别采用最佳算法OPT（当有多个页面可置换时，按照先进先出原则进行置换）、先进先出算法FIFO和最近最久未用算法LRU进行页面置换，其中LRU算法采用栈式方法实现。

 3.显示页面变化时内存块装入页面列表的详细情况，并显示是否产生页面置换，并计算缺页次数及缺页率。具体信息见测试用例格式输出部分说明。

## 算法及数据结构

整个程序结构是比较简单的，输入命令解析后，按照不同的算法请求页面



**FIFO 先进先出算法**

​	这个算法相比较来说是最简单的算法，我们用vector<pair<int,int>>来模拟页面内存，其中first元素为页面号，second属性为页面进入的序号，来标记页面进入的顺序。

​	算法流程：

​	1.申请并初始化内存空间

​	2.对页面请求队列进行遍历，

​	3.对于每个个请求的页面，遍历一遍内存，出现以下三种情况：

​		a. 内存空间存在空页面，停止遍历内存，对于给定的空页面号使其被请求页面代替

​		b. 内存空间命中，停止遍历内存

​		c. 内存空间未命中，遍历内存空间后，选择second属性最小的页面进行替换

​	4.遍历一般内存空间，输出其状态

```c++
void fifo_page_replacement(vector<int> &vec_pages) {
	//先进先出 ，用vector来模拟队列
	vector<pair<int,int>> inner_memory(MAX_MEMORY);
	for (int i = 0; i < inner_memory.size(); i++) {
		inner_memory[i] = make_pair(-1, -1);
	}
	int missing_page_cnt = 0;
	vector<int>::iterator it = vec_pages.begin();
	int init_pos = 0;
	string ans;
	int f = 0;
	for (; it != vec_pages.end(); it++) {
		int v = (*it);
		bool is_hitting = false;
		int pos = 0;
		for (unsigned int i = 0; i < inner_memory.size(); i++) {
			if (inner_memory[i].first == -1) {
				//为空
				break;
			}
			else {
				if (inner_memory[i].first == v) {
					//命中
					pos = i;
					is_hitting = true;
					break;
				}
				else {
					//未命中,找先
					if (inner_memory[i].second < inner_memory[pos].second) {
						pos = i;
					}
				}
			}
		}

		if (!is_hitting) {
			if (init_pos < MAX_MEMORY) {//存在空的内存块
				inner_memory[init_pos] = make_pair(v, f++);
				init_pos++;
			}
			else {//无内存块，需要置换
				inner_memory[pos] = make_pair(v, f++);
			}
			missing_page_cnt++;
		}
		else {

		}

		//遍历一遍内存，输出其状态
		string sub_ans = print_inner_memory(inner_memory, is_hitting);
		if (it != vec_pages.end() - 1) {
			ans += sub_ans + "/";
		}
		else {
			ans += sub_ans;
		}
	}
	std::cout << ans << endl << missing_page_cnt << endl;
}
```



**OPT 最佳置换算法**

​	该算法对于页面置换的方式：当要对内存内的页面进行置换时，要置换在后来的请求队列中最后使用的页面。

​	内存空间依旧用一个opt_block的数据结构进行模拟，其中num属性记录页面号，next属性记录在请求队列中，该页面之后的离它最近的与它页面号相同的页面的序列，如果该页面之后不存在符合要求的页面，则second赋值一个极大值。arr属性代表该页面在在请求队列中的到达次序。

```c++
struct opt_block {
	int num;
	int arr;
	int next;
};
```

​	算法流程：

1.申请并初始化内存空间

2.对请求空间进行遍历

3.对于每个请求页面，遍历一遍内存，出现以下三种情况：

​	a.内存里存在空页面，内存遍历结束，将请求页面放入内存空间。

​	b.内存命中，对命中的页面进行维护，主要在现有基础上遍历请求队列，更行next属性

​	c.内存未命中，在对内存遍历过程中找到内存空间中next最大，在next相同情况下，arr最小的页面进行置换。置换后对新页面的next属性进行维护。

4.遍历内存空间输出内存空间占用情况。

```c++
void opt_page_replacement(vector<int> &vec_pages) {
	int missing_page_cnt = 0;
	vector<int>::iterator it = vec_pages.begin();
	vector<struct opt_block> inner_memory(MAX_MEMORY);

	for (unsigned int i = 0; i < inner_memory.size(); i++) {
		inner_memory[i].num = -1;
	}
	int f = 0;
	string ans;
	for (int j = 0; it != vec_pages.end(); j++, it++) {
		int v = (*it);
		bool is_hitting = false;
		int pos = 0;
		for (unsigned int i = 0; i < inner_memory.size(); i++) {
			if (inner_memory[i].num == -1) {
				pos = i;
				break;
			}
			else {
				if (inner_memory[i].num == v) {
					pos = i;
					is_hitting = true;
					break;
				}
				else {
					if (inner_memory[i].next > inner_memory[pos].next || (inner_memory[i].next == inner_memory[pos].next&&
						inner_memory[i].arr < inner_memory[pos].arr)) {
						pos = i;
					}
				}
			}
		}
		int next_v_pos = MAX_PAGES_NUM;
		for (unsigned int i = j + 1; i < vec_pages.size(); i++) {
			if (vec_pages[i] == v) {
				next_v_pos = i;
				break;
			}
		}
		inner_memory[pos].next = next_v_pos;


		if (!is_hitting) {
			inner_memory[pos].num = v;
			missing_page_cnt++;
			inner_memory[pos].arr = f++;
		}

		string substr;
		for (unsigned int i = 0; i < inner_memory.size(); i++) {
			string ch = inner_memory[i].num == -1 ? "-," : (Int_to_Str(inner_memory[i].num) + ",");
			substr += ch;
		}
		substr += is_hitting ? "1" : "0";
		if (it != vec_pages.end()-1) {
			substr += "/";
		}
		ans +=substr;
	}
	cout << ans << endl << missing_page_cnt << endl;
}
```



**LRU最久未用算法**

​	首先介绍这个算法，LRU是对于OPT的一种改进，也是实际过程中运用较多的一种算法。从理论上来讲，OPT是一种最佳的算法，但是实际过程中，我们并不知道整个页面请求队列的情况，也就是不知道在处理该页面时，后续页面的请求状况。因此我么令这个后验概率近似等于先验概率，换句话说，在最近请求的页面，我们认为在之后也会立即被请求，很久之前请求过的页面，我们认为在很久之后才会请求。总的来说就是，当在内存中进行置换页面时，我们优先置换最久未被使用页面（这一个特性可以用栈的数据机构完美展示）。

​	这个算法给的要求使用栈试方法进行实现，所以在程序的输出格式上相对于上面两种算法会较为特别。这个栈式方法比较像队列符合先进先出的性质，用队列自然会想到用c++里面的queue数据结构，但是queue并不支持对队列里面信息的遍历。因此我们用vector<pair<int,int>>的数据结构来对队列进行模拟，first属性用来记录页面号，second属性来记录页面进入以来经过的时间。在后续操作中发现其实second这个属性时没必要的，因为second属性是用来比较内存中页面进入的顺序，因为栈中是一个先进先出的机制，所以内存空间的排序就自带了每个页面进入内存的优先顺序。

​	算法还需注意的几点：

​	1.如果内存空间中，页面命中，不能将队首元素pop出，而是将命中元素pop出，再重新将一个相同的页面压入队列。

​	2.如果内存空间中为空，页面命中,也将对内存空间中命中的页面pop出，再重新将一个相同的额页面压入队列。

​	3.在输出格式上，因为是栈式结构，当内存为空时，应从靠后的空页面进行填充。

​	算法流程：

​	1.申请及初始化内存空间

​	2.对请求队列进行遍历

​	3.对于每一个请求页面，对内存空间进行遍历,会遇到以下三种情况：

​		a.如果未命中，且存在空页面，对空页面进行填充

​		b.如果未命中，不存在空页面，将队列头的页面pop出，队列尾进入新的页面

​		c.如果命中，将命中的页面pop出，队列尾进入相同的新的页面

​	4.对内存空间进行遍历，输出内存空间的状态

```c++
void lru_page_replacement(vector<int> &vec_pages) {
	//用vector模拟内存，如果命中，将其移到vec的末端，如果未命中，置换vec第一个元素，将其加入vec尾端
	vector<pair<int,int>>inner_memory(MAX_MEMORY,make_pair(-1,-1));
	vector<int>::iterator it = vec_pages.begin();
	int init_pos = 0;
	string ans;
	int missing_page_cnt = 0;
	for (; it != vec_pages.end(); it++) {
		int pos = 0;//置换或命中的位置
		int v = (*it);
		bool is_hitting = false;
		for (unsigned int i = 0; i < inner_memory.size(); i++) {
			if (inner_memory[i].first == -1) {//页面为空
				break;
			}
			else {
				if (inner_memory[i].first== v) {//命中
					pos = i;
					is_hitting = true;
					break;
				}
				else {

				}
			}
		}
		if (is_hitting) {//命中
			if (init_pos < MAX_MEMORY) {
				inner_memory.erase(inner_memory.begin() + pos);
				inner_memory.push_back(make_pair(-1, -1));
				inner_memory[init_pos - 1] = make_pair(v, 0);
			}
			else {
				inner_memory.erase(inner_memory.begin() + pos);
				inner_memory.push_back(make_pair(v, 0));
			}

		}
		else {
			if (init_pos < MAX_MEMORY) {
				inner_memory[init_pos] = make_pair(v, 0);
				init_pos++;
			}
			else {
				inner_memory.erase(inner_memory.begin());
				inner_memory.push_back(make_pair(v, 0));
			}
			missing_page_cnt++;
		}

		//遍历一遍内存，输出状态
		string sub_ans = print_inner_memory(inner_memory, is_hitting);
		if (it != vec_pages.end() - 1) {
			ans += sub_ans + "/";
		}
		else {
			ans += sub_ans;
		}

	}
	std::cout << ans << endl << missing_page_cnt << endl;
}

```

