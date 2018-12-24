---
layout: post
title: "磁盘调度算法"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - C++
  - OS
---
# 简单的磁盘调度算法



## 要求

​	可以随机输入磁道请求序列，当前磁头位置和磁头移动方向，支持先来先服务、最短寻道时间优先、扫描、循环扫描调度算法，能够输出磁头移动经过的磁道序列。具体信息见测试用例格式说明。



## 数据结构及算法



**vector\<int\>  init_input(str_tracks)**

​	该函数对输入的string类型的磁道请求序列进行处理，对字符串根据‘，’进行分割后，将每个数存储到一个vector<int\>中。便于处理

```c++
vector<int> init_input(string str_tracks) {
	vector<int> vec_tracks(0);
	int pos = 0;
	for (int i = 0; i < str_tracks.length(); i++) {
		if (str_tracks[i] == ',') {
			string sub = str_tracks.substr(pos, i);
			int v = Str_to_Int(sub);
			vec_tracks.push_back(v);
			pos = i + 1;
		}
	}
	string sub = str_tracks.substr(pos, str_tracks.length());
	int v = Str_to_Int(sub);
	vec_tracks.push_back(v);
	return vec_tracks;
}
```



**void FIFO_seek_track（vector<int\> &vec_tracks,const int &init_pos）**

​	该函数表示的是先来先服务的寻道算法，参数是储存磁道请求序列的vec和起始磁道位置。

​	该算法较为简单，按照磁道请求序列的顺序，依次对磁道进行处理。及遍历一遍vec_tracks即可完成整个磁道的寻道工作

```c++
void FIFO_seek_track(vector<int> &vec_tracks,const int &init_pos) {
	//到达第一个需要的路程
	string ans = Int_to_Str(init_pos) + "," + Int_to_Str(vec_tracks[0]);
	int sum = abs(vec_tracks[0] - init_pos);
	for (int i = 1; i < vec_tracks.size(); i++) {
		sum += abs(vec_tracks[i] - vec_tracks[i - 1]);
		ans += "," + Int_to_Str(vec_tracks[i]);
	}
	cout << ans << endl << sum << endl;
}
```



**void SSTF_seek_track(vector<int\> & vec_tracks,const int & init_pos,const int &dir)**

​	最短时间寻道算法，参数依次是储存磁道请求序列的vec ，起始磁道位置和磁道初始方向。

​	算法流程：

1.将init_pos插入vec，对vec进行按照磁道序号排序，如果dir为1按升序排列，如果dir为0按降序排列。然后得到init_pos在vec中的索引index。此时index为当前的磁道号的索引。

2.比较当前磁道号左右两个元素的离当前磁道号的距离，距离短的为优先寻道目标。如果距离相同，选择右边的元素作为优先寻道目标。

3.然后将当前磁道号从vec中移除，index更迭为优先寻道目标的在新vec中的索引（注意这个新vec中的索引，删除了当前磁道号后，其后的磁道号的在vec中的索引会改变。）

4.如果vec中元素个数大于1，重复步骤2，否则结束寻道

```c++
void SSTF_seek_track(vector<int> &vec_tracks,const int & init_pos,const int & dir) {
	string ans;
	vec_tracks.push_back(init_pos);
	sort(vec_tracks.begin(), vec_tracks.end(), cmp_up);
	vector<int> ::iterator ielement = find(vec_tracks.begin(), vec_tracks.end(), init_pos);
	int index = distance(vec_tracks.begin(), ielement);
	int sum = 0;
	while (1) {
		if (vec_tracks.size() == 1) {
			break;
		}
		int right = index + 1 >= vec_tracks.size() ? MAX_DISTANCE : abs(vec_tracks[index + 1] - vec_tracks[index]);
		int left = index - 1 < 0 ? MAX_DISTANCE : abs(vec_tracks[index - 1] - vec_tracks[index]);
		int last_index = index;
		if (left < right) {
			sum += abs(vec_tracks[index - 1] - vec_tracks[index]);
			index = index - 1;
		}
		else if (left == right) {
			sum += dir == 1 ? abs(vec_tracks[index + 1] - vec_tracks[index]) : abs(vec_tracks[index - 1] - vec_tracks[index]);
			index = dir == 1 ? index : index - 1;
		}
		else {
			sum += abs(vec_tracks[index + 1] - vec_tracks[index]);
		}
		if (vec_tracks[last_index] != init_pos) {
			ans += ",";
		}
		ans +=Int_to_Str(vec_tracks[last_index]);
		vec_tracks.erase(vec_tracks.begin() + last_index);
	}
	ans += "," + Int_to_Str(vec_tracks[0]);
	cout << ans << endl << sum << endl;
}
```





**void SCAN_seek_track(vector<int\> &vec_tracks,const int &init_pos, const int &dir)**

​	该函数表示的是扫描法，参数依次是储存磁道请求序列的vec ，起始磁道位置和磁道初始方向。

​	算法流程：

​	1.将init_pos插入vec，对vec进行按照磁道序号排序，如果dir为1按升序排列，如果dir为0按降序排列。然后得到init_pos在vec中的索引index。此时index为当前的磁道号的索引。

​	2.申请一个bool型map来代表磁道请求序列中被每个请求是否被处理（起始index的磁道请求已经被处理了）。申明定义一个last代表上一个被请求的磁道序号

​	3.从index开始遍历请求序列vec，遇到未被处理的请求进行处理。再从vvec_tracks.size()-1开始方向遍历队列，遇到未被处理的请求进行处理。

​	4.检查map是否全为1（及所有请求是否已被处理）如果已经全部处理，结束寻道，否则令index=0 重复步骤3.

```c++
bool check_finish(int *map,int max_index) {
	bool f = true;
	for (int i = 0; i < max_index; i++) {
		if (map[i] == 0) {
			f = false;
			break;
		}
	}
	return f;
}
void SCAN_seek_track(vector<int> &vec_tracks,const int &init_pos, const int &dir) {
	vec_tracks.push_back(init_pos);
	string ans = Int_to_Str(init_pos);
	if (dir == 1) {
		sort(vec_tracks.begin(), vec_tracks.end(), cmp_up);
	}
	else {
		sort(vec_tracks.begin(), vec_tracks.end(), cmp_down);
	}
	vector<int>::iterator ielement = find(vec_tracks.begin(),vec_tracks.end(),init_pos);
	int index = distance(vec_tracks.begin(), ielement);
	int sum = 0;
	int *map = new int[vec_tracks.size() + 5]();
	map[index] = 1;
	int last_pos = vec_tracks[index];
	while (1) {
		if (check_finish(map, vec_tracks.size())) {
			break;
		}
		for (int i = index; i < vec_tracks.size(); i++) {
			if (map[i] != 1) {
				sum += abs(vec_tracks[i] - last_pos);
				map[i] = 1;
				last_pos = vec_tracks[i];
				ans += "," + Int_to_Str(vec_tracks[i]);
			}
		}
		index = 0;
		for (int i = vec_tracks.size()-1; i >= 0; i--) {
			if (map[i] != 1) {
				sum += abs(vec_tracks[i] - last_pos);
				map[i] = 1;
				last_pos = vec_tracks[i];
				ans += "," + Int_to_Str(vec_tracks[i]);
			}
		}
	}
	cout << ans << endl << sum << endl;
}
```



**void C_SCAN_seek_track(vector<int\> &vec_tracks, const int &init_pos, const int &dir)**

​	循环扫描法，参数依次是储存磁道请求序列的vec ，起始磁道位置和磁道初始方向。

​	该算法在扫描发上进行了小小的改动，如果对磁道的访问请求是均匀分布的，当磁头到达磁盘的一端，并反向运动时落在磁头之后的访问请求相对较少。这是由于这些磁道刚被处理，而磁盘另一端的请求密度相当高，且这些访问请求等待的时间较长，为了解决这种情况，循环扫描算法规定磁头单向移动。

​	算法流程：

​	1.将init_pos插入vec，对vec进行按照磁道序号排序，如果dir为1按升序排列，如果dir为0按降序排列。然后得到init_pos在vec中的索引index。此时index为当前的磁道号的索引。

​	2.申请一个bool型map来代表磁道请求序列中被每个请求是否被处理（起始index的磁道请求已经被处理了）。申明定义一个last代表上一个被请求的磁道序号

​	3.从index开始遍历请求序列vec，遇到未被处理的请求进行处理。

​	4.检查map是否全为1（及所有请求是否已被处理）如果已经全部处理，结束寻道，否则令index=0 重复步骤3.

```c++
void C_SCAN_seek_track(vector<int> &vec_tracks, const int &init_pos, const int &dir) {
	vec_tracks.push_back(init_pos);
	string ans = Int_to_Str(init_pos);
	if (dir == 1) {
		sort(vec_tracks.begin(), vec_tracks.end(), cmp_up);
	}
	else {
		sort(vec_tracks.begin(), vec_tracks.end(), cmp_down);
	}
	vector<int>::iterator ielement = find(vec_tracks.begin(), vec_tracks.end(), init_pos);
	int index = distance(vec_tracks.begin(), ielement);
	int sum = 0;
	int *map = new int[vec_tracks.size() + 5]();
	map[index] = 1;
	int last_pos = vec_tracks[index];
	while (1) {
		if (check_finish(map, vec_tracks.size())) {
			break;
		}
		for (int i = index; i < vec_tracks.size(); i++) {
			if (map[i] != 1) {
				sum += abs(vec_tracks[i] - last_pos);
				map[i] = 1;
				last_pos = vec_tracks[i];
				ans += "," + Int_to_Str(vec_tracks[i]);
			}
		}
		index = 0;
	}
	cout << ans << endl << sum << endl;
}
```


