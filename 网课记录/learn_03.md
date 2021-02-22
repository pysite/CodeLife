# 第四周记录



## stack容器

### stack基本概念

- 栈不允许有遍历行为；

- 栈可以返回目前栈中元素个数；

### stack常用接口

构造函数：

- stack<T> stk;
- stack(const stack &stk);

赋值操作：

- stack& operator=(const stack &stk);

数据存取：

- push(elem);
- pop();
- top()

大小操作：

- empty();
- size();



## queue容器

### queue基本概念

只有队头和队尾能被外界访问，因此不许有遍历的行为。

### queue常用接口

构造函数：

- queue<T> que;
- queue(const queue *que);

赋值操作：

- queue& operator=(const queue &que);

数据存取：

- push(elem);
- pop();
- back();
- front();

大小操作：

- empty();
- size();



## list容器

### list基本概念

list就是链表，物理存储单元上非连续。

STL中的链表是一个**双向循环链表**。

由于链表的存储方式并不是连续的内存空间，因此链表list中的迭代器只支持前移和后移，属于双向迭代器。

### list构造函数

- list<T> lst;
- list(beg, end);
- list(n, elem);
- list(const list &lst);

### list赋值和交换

- assign(beg, end);
- assign(n, elem);
- list& operator=(const list &lst);
- swap(lst);

### list大小操作

- size()				//返回容器中元素的个数
- empty()			//判断容器是否为空
- resize(num);    //重新指定容器的长度为num，若容器变长，则以默认值填充
- resize(num, elem);

### list插入和删除

- push_back(elem);
- pop_back()
- push_front(elem);
- pop_front();
- insert(pos, elem);  //在pos位置插入elem元素的拷贝，返回新数据的位置
- insert(pos, n, elem);  //在pos位置插入n个elem，无返回值
- clear();
- erase(beg, end);  //删除[beg,end)的数据，返回下一个数据的位置
- erase(pos);  //删除pos位置的数据，返回下一个数据的位置
- remove(elem);  //删除容器中所有**与elem值匹配**的数据

### list数据存取

没有``at()``和``operator[]``，因为链表不是连续线性，不支持随机访问。

==迭代器it不支持+1，只支持++和--==；

- front();
- back()

### list反转和排序

- reverse();  //反转链表
- sort();  //链表排序

==所有不支持随机访问迭代器的容器，不可以用标准算法==。

不支持随机访问迭代器的容器，内部会提供对应的一些算法。

```c++
list<int> lst;
lst.push_back(10);
lst.push_back(5);

sort(lst.begin(), lst.end());//会报错
lst.sort();	//不会报错，默认升序

bool myCompare(int v1, int v2){
    //这里v1,v2就是顺序的两个数
    //降序 
    return v1 > v2;
}
lst.sort(myCompare);	//降序排序
```



## set/multiset容器

### set基本概念

- 所有元素在插入时被自动排序。

- set/multiset属于**关联式容器**，底层结构用**二叉树**实现。

**set和multiset的区别**：

- set不允许容器中有重复的元素
- multiset允许容器中有重复的元素



### set构造和赋值

构造：

- set<T> st;
- set(const set &st);

赋值：

- set& operator=(const set &st);



### set大小和交换

set不允许有重复的值，因此不存在resize的说法（否则默认值可能会多个）

- size();
- empty();
- swap(st);



### set插入和删除

- insert(elem);
- clear();
- erase(pos);
- erase(beg, end);
- erase(elem);  //删除值为elem的元素



### set查找和统计

- find(key);  //查找key是否存在，若存在，返回**该键元素的迭代器**；若不存在，返回set.end()
- count(key);  //统计key的元素个数



### set和multiset区别

- set不可以插入重复数据，而multiset可以；

- set插入数据的同时会返回插入结果，表示插入是否成功；

  ==**插入返回一个<iterator, bool>的pair类型**==，其中bool表示此次插入操作是否成功；

  **如果set中有个数据已存在，就会插入失败**。

- multiset不会检测数据，因此可以插入重复数据；

```c++
set<int> s;
pair<set<int>::iterator, bool> ret = s.insert(10);
if(ret.second){
    ...	//插入成功
}
```



### 拓展：pair对组创建

**创建方式**：

- pair<type, type> p(value1, value2);
- pair<type, type> p = make_pair(value1, value2);

用.first或.second来获取其中的属性值。



### set容器排序

set容器**默认排序规则为从小到大**，掌握如何改变排序规则；

**主要技术点**：

- 利用仿函数，改变排序规则；

```c++
class MyCompare{
public:
    bool operator()(int v1, int v2){
        return v1 > v2;
    }
}
//指定排序规则为从大到小
set<int, MyCompare> s;
```

注意**自定义类型的仿函数的operator()函数的两个参数必须是const的**，且该函数形参后面也要添加个const。



## map容器

### map基本概念

简介：

- map中所有元素都是pair；
- pair中第一个元素为key（键值），起到索引作用，第二个元素为value（实值）
- 所有元素都会根据元素的键值自动排序

本质：

- map/multimap属于**关联式容器**，底层是二叉树实现。

优点：

- 可以根据key值快速找到value值，map是高校的，貌似是哈希查找的，能在几十亿的数据中快速查找

map和multimap的区别：

- map不允许容器中有重复key值的元素
- multimap允许容器中有重复key值元素；



### map构造和赋值

>  要注意的就是map插入和赋值都需要pair。

构造：

- map==<T1, T2>== mp;
- map(const map &mp);

赋值：

- map& operator=(const map &mp);

```c++
map<int, int> m;
m.insert(pair<int, int>(1,10));	//插入匿名对象
```



### map大小和交换

- size();
- empty();
- swap(st);



### map插入和删除

- inset(elem);

  ```c++
  map<int, int> m;
  m.insert(pair<int, int>(1, 10));
  m.insert(make_pair(2, 10));
  m.insert(map<int, int>::value_type(3, 10));
  m[4] = 40;	//不建议这种
  cout << m[5];	//不建议的原因：比如这种，会给你创建一个key为5且value为0的项。
  ```

- clear();

- erase(pos);  //pos是迭代器

- erase(beg, end);

- erase(key);  //删除容器中值为key的元素



### map查找和删除

- find(key);  //找得到就返回元素的迭代器，否则返回map.end()
- count(key);  //统计key的元素个数



### map排序

map容器**默认排序规则为按照key值从小到大排序**，掌握如何改变排序规则；

**主要技术点**：

- 利用仿函数，改变排序规则；

```c++
class MyCompare{
public:
    bool operator()(int v1, int v2){
        return v1 > v2;	//按照降序
    }
}
map<int, int, MyCompare> m;
```

自定义数据类型，map必须指定排序规则，同set。



# STL-函数对象

本章主要介绍STL中的常用算法对象，例如遍历算法、排序算法、查找算法等；

## 函数对象

### 函数对象概念

**概念**：

- 函数对象实际上就是仿函数，即重载了operator()的类；

**本质**：

- 函数对象（仿函数）是一个类，而不是一个类；



### 函数对象使用

**特点**：

- 函数对象在使用时，可以像普通函数那样调用，可以有参数，可以有返回值；
- 函数对象超出普通函数的概念，函数对象可以**有自己的状态**；
- 函数对象可以**作为参数传递**；



## 谓词

### 谓词概念

**概念**：

- 返回bool类型的仿函数称为谓词
- 如果operator()接受一个参数，那么叫做一元谓词
- 如果operator()接受两个参数，那么叫做二元谓词



### 一元谓词

在参数列表中如果看到Pred则一般就是想要一个谓词；

```c++
//本例假设我们想要找到大于5的数字
class GreaterFive{
public:
    bool operator()(int v){
        return v > 5;
    }
}
vector<int> v;
...
#include<algorithm>	//为了导入find_if
//找得到就返回迭代器，找不到就返回.end()
vector<int>::iterator it = find_if(v.begin(), v.end(), GreaterFive());
```



### 二元谓词

> 和一元谓词差不多，略



## 内建函数对象

### 内建函数对象意义

**概念**：

- 内建函数对象就是STL内部已有的一些函数对象；

分类：

- 算术仿函数 --- 加减乘
- 关系仿函数 ---大于小于等于
- 逻辑仿函数 --- 与或非

用法：

- 这些仿函数所产生的对象，用法和一般函数完全相同；
- 使用内建函数对象，需要引入头文件``#include<functional>``



### 算术仿函数

功能描述：

- 实现四则运算
- 其中**negate**是一元运算，其它都是二元运算

仿函数原型：（前一个<class T>是声明T是类的类型，后一个<T>是在给类传递类参数）

- template<class T> T plus<T>
- template<class T> T minus<T>
- template<class T> T multiplies<T>
- template<class T> T divides<T>
- template<class T> T modules<T>
- template<class T> T negate<T>  //取负仿函数

```c++
#include<funtional>
negate<int> n;
n(50);	//结果是-50

plus<int> p;
p(1,1);	//结果是2
```



### 关系仿函数

- template<class T> bool equal_to<T>
- template<class T> bool not_equal_to<T>
- template<class T> bool greater<T>
- template<class T> bool greater_equal<T>  //大于等于
- template<class T> bool less<T>
- template<class T> bool less_equal<T>

```c++
vector<int> v;
...
sort(v.begin(), v.end(), greater<int>());	//使用内建函数同样实现降序排序
```



### 逻辑仿函数

- template<class T> bool logical_and<T>
- template<class T> bool logical_or<T>
- template<class T> bool logical_not<T>

```c++
vector<bool> v1;
...
vector<bool> v2;
v2.resize(v1.size());	//不这样则下述transform函数会执行失败，必须先设置好空间。
//将v1容器中的元素搬运到v2中并将每个被搬运的元素进行取反的操作
transform(v1.begin(), v1.end(), v2.begin(), logical_not<bool>());
```



# STL-常用算法

概述：

- 算法主要是由头文件<algorithm><functional><numeric>组成
- <algorithm>是所有STL头文件中最大的一个，范围涉及到比较、交换、查找、遍历操作、复制、修改等；
- <numeric>体积很小，只包括几个在序列上面进行简单教学运算的模板函数；
- <functional>定义了一些模板类，用以声明函数对象；



## 常用遍历算法

- for_each;  //遍历容器
- transform;  //搬运容器到另一个容器中

### for_each

- for_each(iterator beg, iterator end, _func)

  _func是函数或函数对象

### transform

- transform(iterator beg1, iterator end1, iterator beg2, _func)

注意：func是函数或函数对象；==目标容器需要提前开辟空间==；



## 常用查找算法

- find  //查找元素
- find_if  //按条件查找元素
- adjacent_find  //查找相邻重复元素
- binary_search  //二分查找法
- count  //统计元素个数
- count_if  //按条件统计元素个数



### find

- 查找指定元素，找到返回指定元素的迭代器，找不到返回结束迭代器end()

函数原型：

- find(iterator beg, iterator end, value);

说明：这个函数底层就是*iter == value，传入的类型必须可以进行"等于符号"判断。因此对于自定义类型必须重载operator==()函数

```c++
class Person{
    bool operator==(const Person& p){	//注意是const Person &
        ...
    }
}
```



### find_if

- find_if(iterator beg, iterator end, _Pred)

注意：_Pred是函数或谓词（返回bool类型的仿函数）

在一元谓词一节有find_if的实例。



### adjacent_find

功能：

- 查找·**相邻**·**重复**·元素

原型：

- adjacent_find(iterator beg, iterator end);

  //查找相邻重复元素，返回相邻元素的第一个位置的迭代器

```c++
vector<int> v;	//假设有元素1,2,3,4,5,5
vector<int>::iterator it = adjacent_find(v.begin(), v.end());	//返回的it将指向第一个5
```



### binary_search

功能：

- 判定指定元素是否存在

原型：

- ``bool binary_search(iterator beg, iterator end, value)``

注意：在无序序列中不可以用



### count

功能描述：

- 统计元素个数

函数原型：

- count(iterator beg, iterator end, value);

注意：和之前的一样，自定义类型要定义operator==()函数



### count_if

函数原型：

- count_if(iterator beg, iterator end, _Pred)

  //就是统计满足条件的元素个数



## 常用排序算法

常用的排序算法：

- sort
- random_shuffle  //洗牌，对指定范围内的元素随机调整次序
- merge  //容器元素合并，并存储到另一容器中
- reverse  //反转指定范围内的元素



### sort

函数原型：

- sort(iterator beg, iterator end, _Pred)



### random_shuffle

函数原型：

- random_shuffle(iterator beg, iterator end);

```c++
#include<ctime>
srand((unsigned int)time(NULL));
random_shuffle(...);
```



### merge

功能：

- 两个容器元素合并，并存储到另一个容器中

函数原型：

- merge(iterator beg1, iterator end1, iterator beg2, iterator end2, iterator dest);

注意：

- 两个容器必须是**有序序列**；
- **必须提前给目标容器分配空间**；



### reverse

功能：

- 将容器内的元素反转，首尾对调；

函数原型：

- reverse(iterator beg, iterator end);



## 常用拷贝和替换算法

常用的算法：

- copy  //容器内指定范围元素拷贝到另一容器
- replace  //将容器内指定范围的元素修改为新元素
- replace_if //容器内指定范围满足条件的元素替换为新元素
- swap  //互换两个容器的元素

### copy

函数原型：

- copy(iterator beg, iterator end, iterator dest);

注意：

- 同样地，要为目标容器提前分配好空间；

### replace

函数原型：

- replace(iterator beg, iterator end, oldvalue, newvalue);

注意：

- 只要是oldvalue相等就替换，自定义类型应该也要重载operator==()

### replace_if

函数原型：

- replace_if(iterator beg, iterator end, _Pred, newvalue);

### swap

函数原型：

- swap(container c1, container c2);



## 常用算术生成算法

注意：

- 算术生成算法属于小型算法，使用时包含的头文件为``#include<numeric>``

算法简介：

- accumulate	//计算容器元素累计总和
- fill  //向容器中添加元素

### accumulate

函数原型：

- accumulate(iterator beg, iterator end, value);

注意：

- value是起始值，即sum的初始值

### fill

函数原型：

- fill(iterator beg, iterator end, value);

注意：

- 填充前也是要保证目标容器已分配空间；



## 常用集合算法

算法简介：

- set_intersection  //求交集
- set_union  //求并集
- set_difference  //求差集

### set_intersection

函数原型：

- set_intersection(iterator beg1, iterator end1, iterator beg2, iterator end2, iterator dest);

注意：

- 两个集合必须是**有序序列**；不是说容器是有序的，而是例如vector这种其中元素必须按照顺序排列；

- 目标容器依然是要提前分配空间

- 目标容器开辟空间需要**从两个容器中取小值**；
- set_intersection返回值即是交集中最后一元素的迭代器；

### set_union和set_difference

这两个函数和set_intersection的使用以及注意事项完全相同，注意的就是差集就是并集减去交集。



2021-02-22 开学前一晚的赎罪