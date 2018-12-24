---
layout: post
title: "可变分区储存管理"
author: "Zreal"
catalog: true
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - C++
  - OS
---
# 可变分区储存管理

## 要求

>1.可以随机输入初始可用内存空间容量，以及进程对内存空间的请求序列，支持首次适应算法、最佳适应算法和最坏适应算法，能够在每次内存分配和回收操作后，显示内存空间的使用情况。具体信息见测试用例格式说明。
>
>2.空闲分区通过空闲区链进行管理，在内存分配时，优先考虑低地址部分的空闲区。
>
>3.当申请空间大于可用空闲内存空间时，不满足此次申请，仍显示此次申请前内存空间的使用情况，后续也不再对此次申请进行任何处理。
>
>4.进程对内存空间的申请和释放可由用户自定义输入。例如，与测试用例示例输入对应的内存空间请求序列为：
>
>(1)   初始状态下可用内存空间为640KB；
>
>(2)   进程1申请130KB；
>
>(3)   进程2申请60KB；
>
>(4)   进程3申请100KB；
>
>(5)   进程2释放60KB；
>
>5.分别采用首次适应算法、最佳适应算法和最坏适应算法模拟内存空间的动态分配与回收，每次分配和回收后显示出内存空间的使用情况（参见测试用例示例输出）。

## 算法及其数据结构

首先整个程序的框架如图：

![img](/img/assets/image-20181217083048142-5006648.png)



对于整个内存空间，我用一个int的hashmap来代表这个空间中内存的每个bit的占有情况，如果为零则为空闲，如果有数字，则表示被特定进程占有。

对于所有的空闲内存，存储在一个vec里面，对于不同的算法要求，对这个vec按不同的要求进行排列。遍历这个vec分配符合要求的内存空间。释放内存后，对这个vec进行重新维护。空闲内存的结构体如下：

```c++
//三个属性分别代表空闲内存块在map中的起始位置，结束位置和长度。
struct Empty_Block {
    int begin;
    int end;
    int len;
    Empty_Block(int a, int b) {
        this->begin = a;
        this->end = b;
        this->len = b - a + 1;
    }
};
```



**分配内存**

*void Allocate_Memory(vector<Empty_Block\*> &vec_empty_block, vector\<int> &info_vec,int choose_meth)*

该函数是分配内存的函数

参数：

vector<Empty_Block*> &vec_empty_block 为对空闲块vec的引用，里面存储的该时刻内存下的空闲块

vector\<int>&info_vec 为对该时刻指令输入的引用。

Int choose_meth 为选择方法的引用



**分配内存流程图**：

![img](/img/assets/image-20181217090004098-5008404.png)



这里对于相应算法，对空闲内存块的排序方案如下

| 算法         | 排序方案                                                 |
| ------------ | -------------------------------------------------------- |
| 首次适应算法 | 选择内存大小合理的，起始地址最靠前的空闲块               |
| 最坏适应算法 | 选择内存大小最大的内存块（在内存大小相同时，选择靠前的） |
| 最佳适应算法 | 选择内存大小不小于可分配内存的最小空闲块                 |

```c++
//首次适应算法
bool First_Fit_cmp(Empty_Block * a, Empty_Block *b) {
    return a->begin<b->begin;
}
//最坏适应算法
bool Worst_Fit_cmp(Empty_Block * a, Empty_Block * b) {
    if (a->len > b->len) {
        return true;
    }
    else {
        if (a->len == b->len) {
            return a->begin < b->begin;
        }
        else {
            return false;
        }
    }
}
//最佳适应算法
bool  Best_Fit_cmp(Empty_Block * a, Empty_Block *b) {
    return a->len<b->len;
}

```

内存分配函数如下：

```c++
void Allocate_Memory(vector<Empty_Block*> &vec_empty_block, vector<int> &info_vec,int choose_meth) {
    //首先将它进行排序
    switch (choose_meth)
    {
        case 1:    sort(vec_empty_block.begin(), vec_empty_block.end(), First_Fit_cmp); break;
        case 2: sort(vec_empty_block.begin(), vec_empty_block.end(), Best_Fit_cmp); break;
        case 3: sort(vec_empty_block.begin(), vec_empty_block.end(), Worst_Fit_cmp); break;
    }

    //然后对其进行遍历找到大小符合的空闲块
    int block_size = info_vec[3];
    //设置一个flag,是否存在符合条件的空闲块,如果存在用一个块记录下来；
    bool isExist = false;
    Empty_Block *one_empty_block;
    int pos = 0;
    vector<Empty_Block *>::iterator it = vec_empty_block.begin();
    for (int i=0; it<vec_empty_block.end(); i++,it++) {
        if ((*it)->len >= block_size) {
            isExist = true;
            one_empty_block = (*it);
            pos = i;
            break;
        }
    }
    //如果存在这样一个空闲块
    if (isExist) {
        //因为这一个空闲块现在被分配所以从空闲块vec中del
        vec_empty_block.erase(vec_empty_block.begin() + pos);
        //如果存在这样一个空闲块,对选出的空闲块进行操作，重置map中的值
        int cnt = 0;
        int pnum = info_vec[1];
        for (int i = one_empty_block->begin; i <= one_empty_block->end; i++) {
            if (cnt < block_size) {
                map[i] = pnum;
                cnt++;
            }
            else {
                one_empty_block->begin = i;
                one_empty_block->len = one_empty_block->end - i + 1;
                vec_empty_block.push_back(one_empty_block);
                break;
            }
        }
    }
    //遍历map，实现状态输出
    string out = to_string(info_vec[0]);
    int f = map[0];
    int sf = 0;
    for (int i = 0; i < Max_SIZE; i++) {
        if (map[i] != f) {
            string s;
            if (f == 0) {
                s = ".0";
            }
            else {
                s = ".1." + to_string(f);
            }
            out += '/' + to_string(sf) + '-' + to_string(i - 1) + s;
            f = map[i];
            sf = i;
        }
    }
    //加入最后一个
    string s;
    if (f == 0) {
        s = ".0";
    }
    else {
        s = ".1." + to_string(f);
    }
    out += '/' + to_string(sf) + '-' + to_string(Max_SIZE - 1) + s;
    cout << out << endl;
}
```





**释放内存**

​	释放内存的思路就简单许多。一共两次对hashmap的遍历。首先按照指令的要求在hashmap上对对应bit置0，然后重新遍历一遍hashmap生成一个新的vec空闲块数组同时打印内存空间状态。

```c++
void My_Free_Memory(vector<Empty_Block*> &vec_empty_block, vector<int>&info_vec){
    //先将对应进程的内存在map里置零
    int pnum = info_vec[1];
    int block_size = info_vec[3];
    int cnt = 0;
    for (int i = 0; i < Max_SIZE; i++) {
        if (map[i] == pnum) {
            if (cnt < block_size) {
                map[i] = 0;
                cnt++;
            }
        }
    }
    //此时内存空间的空闲块数存在改变，清空记录空闲块的vector，重新进行维护

    vec_empty_block.clear();
    //遍历map，重新记录空闲块，同时也输出内存空间的状态
    string out = to_string(info_vec[0]);
    int f = map[0];
    int sf = 0;
    for (int i = 0; i < Max_SIZE; i++) {
        if (map[i] != f) {
            string s = f == 0 ? ".0" : (".1." + to_string(f));
            out += '/' + to_string(sf) + '-' + to_string(i - 1) + s;
            if (f == 0) {
                //如果这是一块空闲块，加入空闲块向量
                vec_empty_block.push_back(new Empty_Block(sf, i - 1));
            }
            f = map[i];
            sf = i;
        }
    }
    //加入最后一个
    string s = f == 0 ? ".0" : (".1." + to_string(f));
    out += '/' + to_string(sf) + '-' + to_string(Max_SIZE - 1) + s;
    if (f == 0) {
        vec_empty_block.push_back(new Empty_Block(sf, Max_SIZE - 1));
    }
    cout << out << endl;
}
```


