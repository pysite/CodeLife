# 第三周记录

C++进阶——模板(泛型)、STL

## 模板

### 函数模板

- C++另一种编程思想称为泛型编程，主要利用的技术就是模板
- C++提供两种模板机制：函数模板和类模板

#### 函数模板语法

函数模板作用：

建立一个通用函数，其函数返回值类型和形参类型可以不具体制定，用一个虚拟的类型来代表；

**语法**：

```c++
template<typename T>
函数声明或定义
```

**解释**：

template：声明创建模板

typename：表明其后面的符号是一种数据类型，可以用class代替

T：通用的数据类型，名称可以代替，通常为大写字母

**例子**：

```c++
template<typename T>	//typename关键字可以替换成class
void mySwap(T &a, T &b){
    T temp = a;
    a = b;
    b = temp;
}

int a = 20;
int b = 10;
//1.自动类型推导
myswap(a,b);
//2.显示指定类型
mySwap<int>(a,b);
```

使用函数模板有两种方式：自动类型推导、显示指定类型

#### 函数模板注意事项

- 自动类型推导，必须推导出一致的数据类型T，才可以使用
- 模板必须要确定出T的数据类型，才可以使用

```c++
template<class T>
void func(){
	...
}
func();			//报错，这个时候就必须指定出T的类型，否则模板确定不了
func<int>();	//不报错
```





#### 函数模板案例——数组排序

```c++
template<class T>
void mySwap(T &a, T &b){
    T temp;
    temp = b;
    b = a;
    a = temp;
}

template<class T>
void mySort(T arr[], int len){
    for(int i = 0; i < len; i++){
        int max = i;
        for(int j = i + 1; j < len; j++){
            if(arr[max] < arr[j]){
                max = j;
            }
        }
        if(max != i){
            mySwap(arr[max], arr[i]);
        }
    }
}
//实现通用对数组进行排序的函数
void test01(){
    char charArr[] = "badcfe";
    int num = sizeof(charArr)/sizeof(char);	//num是7
    mySort(charArr, num);
}
```

#### 普通函数与函数模板的区别

- 普通函数调用时可以发生自动类型转换（隐式类型转换）

  就是说例如参数类型是int，可以传入char或double类型的参数；

- 函数模板调用时，如果利用自动类型推导，不会发生隐式类型转换

- 如果利用显式指定类型的方式，可以发送隐式类型转换

#### 普通函数与函数模板的调用规则

当普通函数和函数模板同名时，调用规则如下：

1. 如果函数模板和普通函数都可以调用，优先调用普通函数
2. 可以通过空模板参数列表来强制调用函数模板
3. 函数模板也可以发生重载
4. 如果函数模板可以产生更好的匹配，优先调用函数模板

```c++
void func(int a){...}

template<class T>
void func(T a){...}

int a = 10;
func(a);	//满足1规则，调用的是普通函数
			//如果此时func普通函数只有声明而没有定义，依然调用的是普通函数，这时会报错
func<>(a);	//满足2规则，空模板参数列表使调用的是模板函数

//满足3规则，模板函数也可重载
template<class T>
void func(T a, T b){...}

char x = 'x';
func(x);	//调用的是模板函数，比起将char隐式类型转换为int，编译器更喜欢模板类型推导
```

在实际工作中，如果提供了模板函数，就不要再写普通函数。

#### 模板的局限性

```
template<class T>
void f(T a, T b){
	a = b;
}
```

问题1：如果a和b是一个数组，就无法实现；（**不太懂**，略）



```
template<class T>
void f(T &a, T &b){
	if(a > b){...}
}
```

问题2：如果T传入的是自定义数据类型例如Person，也无法正常工作。

**解决方法**：（问题2的解决方法）

- 运算符重载，但是太麻烦了，必须给出现过的每个运算符都得重载一遍

- 提供一个重载函数（具体化的模板），例如：

  ```c++
  template<> void f(Person &a, Person &b){	//一定要记得加template<>
  	...										//表明这是模板函数的重载而非普通函数
  }
  ```

  这样当传入的是Person类型，就会调用上面的模板函数的重载函数；



### 类模板

#### 类模板基本语法

```c++
template<class NameType, class AgeType>
class Person{
public:
    Person(NameType name, AgeType age){
        m_Name = name;
        m_Age = age;
    }
    NameType m_Name;
    AgeType m_Age;
}
Person<string, int> p1("小明",18);
```

#### 类模板和函数模板的区别

1. 类模板没有自动类型推导的使用方式；（就是必须要带上<某种类型>)

2. 类模板在模板参数列表中可以有默认参数；例如：

   ``template<class NameType, class AgeType = int>``

   这样在使用时，尖括号里面可以只带一个类型；

#### 类模板中成员函数创建时机

类模板中成员函数和普通类中成员函数创建时机是有区别的：

- 普通类中的成员函数一开始就可以创建；
- 类模板中的成员函数在调用时才创建；

```c++
template<class T>
class MyClass{
	T obj;
	void func(){
		obj.subFunc();
	}
}
//上面的代码可以编译通过就是因为类模板中成员函数只有在调用时才创建
//并且即使T类型没有subFunc成员函数也是可以创建相应MyClass对象的，只要不调用func()函数就可
```

总之就是类模板成员函数在调用时才去检查；

#### 类模板对象做函数参数

将类模板实例化出来的对象向函数传参的方式。

共有三种传入方式：

1. 指定传入的类型 --- 直接显示对象的数据类型
2. 参数模板化 --- 将对象中的参数变为模板进行传递
3. 整个类模板化 --- 将这个对象类型模板化进行传递

```c++
template<class T1, class T2>
class Person{
public:
	Person(T1 name, T2 age){...}
	T1 m_Name;
	T2 m_Age;
}
Person<string, int>p("小明", 18);

//1.指定传入类型
void print1(Person<string, int> &p){...}
//2.参数模板化
template<class T1, class T2>	//就是告诉T1和T2是两个类型
void print2(Person<T1, T2> &p){...}
//3.整个类模板化
template<class T>
void print2(T &p){...}
```

#### 类模板与继承

- 当子类继承的父类是一个类模板时，子类在继承时必须要指出父类中T的类型，即继承的是哪个父亲；
- 如果不指定，编译器无法给子类分配内存；
- 如果想灵活指定出父类的T类型，则子类也是个类模板；

```c++
template<class T>
class Base{
	T m;
}
//1.指定出父类的T类型
class Son:public Base<int>{
    ...
}
//2.不指出父类的T类型
template<class T1, class T2>
class Son:public Base<T2>{
    T1 obj;
}
```

#### 类模板成员函数类外实现

```c++
template<class T>
class Person{
	T m;
	void func(T obj);
}
//类外实现
template<class T>
void Person<T>::func(T obj){...}
```

#### 类模板分文件编写

问题：

- 类模板中成员函数创建时机是在调用阶段，导致分文件编写时链接不到；

解决：

- 解决方式1：直接包含.cpp源文件；
- 解决方式2：将声明和实现写到同一个文件中，并更改后缀名为.hpp，hpp是约定的名称，并不是强制；

问题就是说我们平常写一个类通常把声明写道.h文件，把定义写道.cpp文件，用这个类的时候只需要包含.h文件即可。但是模板类的函数都是在调用时才创建的，因此对于模板类不能只包含.h文件，要么改为包含.cpp文件（cpp文件中已经包含了.h文件），要么改为包含同时含有声明和定义的.hpp文件。

新学到我们可以在.h文件中添加#pragma once参数来告诉编译器该头文件只被包含一次。

#### 类模板与友元

1. 全局函数类内实现 - 直接在类内声明友元即可；

2. 全局函数类外实现 - 需要提前让编译器知道全局函数的存在；

注意：全局函数可以在类内实现，只要加上friend就不算类的成员函数了。

```c++
//类声明在最前
template<class T1, class T2>
class Person;
//全局函数声明在其后
template<class T1, class T2>
void printPerson2(Person<T1, T2> p);

template<class T1, class T2>
class Person{
    //全局函数类内实现
	friend void printPerson(Person<T1, T2> p){
		...
	}
    //全局函数类外实现
    //加空模板参数列表<>, 必须要让编译器提前知道这个全局函数，即让该函数的声明或定义在类之前，
    //而这个函数又用到了Person，因此还需将Person的声明放在最前面
    friend void printPerson2<>(Person<T1, T2> p);
    ...
}
//类外实现
template<class T1, class T2>
void printPerson2(Person<T1, T2> p){
    ...
}
Person<string, int> p("Jack",21);
printPerson(p);
```



## STL

回顾：面向对象三大特性：封装、继承、多态

### STL初识

#### STL基本概念

- STL从广义上分为：容器、算法、迭代器
- 容器和算法之间通过迭代器进行无缝链接
- STL几乎所有的代码都采用了模板类或模板函数

#### STL六大组件

1. 容器：各种数据结构，如vector、list；
2. 算法：各种常用的算法，如sort、find、copy；
3. 迭代器：扮演了容器与算法之间的胶合剂；
4. 仿函数：行为类似函数，可作为算法的某种策略；之间template中的<>就是仿函数；
5. 适配器：一种用来修饰容器或者仿函数或迭代器接口的东西；
6. 空间配置器：负责空间的配置与管理；

#### STL中容器、算法、迭代器

容器分为序列式容器和关联式容器：

- 序列式容器：强调值得排序，序列式容器中的每个元素均有固定的位置；
- 关联式容器：二叉树结构，各元素之间没有严格的物理上的顺序关系；

算法分为质变算法和非质变算法

- 质变算法：指运算过程中会更改区间内的元素的内容，例如拷贝、替换、删除；
- 非质变算法：不会更改区间内的元素，例如查找、计数、遍历、寻找机值；

算法使用迭代器访问容器的元素，每个容器都有自己专属的迭代器。

| 种类           | 功能                                         | 支持运算                                |
| -------------- | -------------------------------------------- | --------------------------------------- |
| 输入迭代器     | 对数据的只读访问                             | 只读，支持++、==、!=                    |
| 输出迭代器     | 对数据的只写访问                             | 只写，支持++                            |
| 前向迭代器     | 读写操作，并能向前推进迭代器                 | 读写，支持++、==、!=                    |
| 双向迭代器     | 读写操作，并能向前和向后操作                 | 读写，支持++、--                        |
| 随机访问迭代器 | 读写操作，可以以跳跃的方式访问任意数据，最强 | 读写，支持++、--、[n]、-n、<、>、<=、>= |

前三种容器很少使用，一般容器的迭代器都是后两者。

### 迭代器初识

#### vector存放内置数据类型

```c++
#include<vector>
vector<int> v;
//向容器插入数据
v.push_back(10);
//通过迭代器访问数据
vector<int>::iterator itBegin = v.begin();	//起始迭代器
vector<int>::iterator itEnd = v.end();	//容器中最后一个元素的下一个，不是最后一个

//1.第一种遍历方式
while(itBegin != itEnd){
    cout << *itBegin;
    itBegin++;
}
//2.第二种遍历方式
//直接auto it = v.begin()也行
for(vector<int>::iterator it = v.begin(); it != v.end(); it++){...}
//3.第三种遍历方式
#include<algorithm>
void myPrint(int val){...}
for_each(v.begin(), v.end(), myPrint);	//这叫回调myPrint
```



### string容器

> 代码示例只涉及这些函数的部分调用方式，这些函数还有很多其它的重载形式

#### 构造函数：

- ``string();``

- ``string(const char *s);``

- ``string(const string & str);``

- ``string(int n, char c);``

#### 赋值操作：

- 用 = 号
- 用string.assign()函数

#### 拼接操作：

- 用 += 号
- 用string.append()函数

#### 查找操作：

- 用find函数；
- 用rfind函数是从右往左找，即找最后一次出现的位置；

```c++
string str1 = "abcdefg";
//0参数是代表从第0个位置开始找, 这个参数可以省略
str1.find("de",0);	//返回3，代表从下标3开始
str1.find("df");	//返回-1，代表没有

str1.rfind("de");	//返回3
```

#### 替换操作

- 用replace函数；

```c++
string str1 = "abcdefg";
//从1号位置起，3个字符变为"1111"
str1.replace(1,3,"1111");	//str1变为“a1111efg"
```

#### 比较操作

- 用compare函数

```c++
string str1 = "abcd";
string str2 = "abcd";
str1.compare(str2);	//相当于是str1-str2，如果两者相同则是0
```

#### 字符存取

- 用[]类似数组方式来读写单个字符
- 用at函数

```c++
string str = "abcd";
//读
str[0];
str.at(0);
//写
str[1] = 'x';
str.at(0) = 'x';	//注意at方式也可以写
```

#### 插入和删除

- 用insert函数来插入
- 用erase函数来删除

```c++
string str = "abcd";
str.insert(1,"111");	//str变为a111bcd
str.earse(1,3);			//str变回abcd
```

#### 截取字串

- 用substr函数

  该函数只有一种形式``substr(int pos, int n);``

```
string base = "abcdefg";
string sub = base.substr(0,3);	//sub是abc
```



### vector容器

#### vector基本概念

**功能**：

- vector数据结构和数组非常相似，也称为单端数组

**vector与普通数组区别**：

- 不同之处在于数组是静态空间，而vector可以动态扩展

**动态扩展**：

- 并不是在原空间续接新空间，而是找更大的内存空间，然后将原数据拷贝到新空间，释放原空间；

**迭代器**：

- vector的迭代器是支持随机访问的迭代器（最强的那种）



#### vector构造函数

- ``vector<T> v;``
- ``vector(v.begin(), v.end());``
- ``vector(n, elem);``
- ``vector(const vector & vec);``



#### vector赋值操作

- 用 = 号来拷贝构造
- ``assign(v.begin(), v.end())``
- ``assign(n, elem)``



#### vector容量和大小

> 对vector容器的容量和大小操作

- ``empty()``	    //判断是否为空
- ``capacity()``  //容器的容量
- ``size()``          //容器中元素的个数
- ``resize(int num)``               //注意若容器变长，则以默认值填充新位置
- ``resize(int num, elem)``  //注意若容器变长，则以elem填充新位置



#### vector插入和删除

- ``push_back(ele)``
- ``pop_back()``
- ``insert(const_iterator pos, ele)``  //注意pos是迭代器，不是int类型
- ``insert(const_iterator pos, int count, ele)``
- ``erase(const_iterator pos)``
- ``erase(const_iterator start, const_iterator end)``
- ``clear()``  //删除容器中所有元素

#### vector数据存取

- ``at(int idx)``
- ``operator[]``
- ``front()``  //返回容器中第一个数据元素
- ``back()``    //返回容器中最后一个数据元素

#### vector互换容器

- ``swap(vec)``  //将vec与本身的元素互换，不是复制

```c++
//swap实际用途：巧用swap可以收缩内存空间
//resize只会改变size，不会缩小分配的内存capacity
//想缩小capacity，可以新建一个vector再swap，之后再delete原来的vector
vector<int> v1 = vector(10000,1);
vector<int>(v1).swap(v1);	
//新拷贝构造一个匿名vector，之后再调用swap和原来的vector互换就可缩小原来vec的capacity，底层原理类似于指针互换那样
```

#### vector预留空间

- ``reserve(int len)``  //容器预留个len个元素长度，预留位置不可初始化，元素不可访问

此函数用来减少重新扩展内存的操作的次数；



### deque容器

#### deque基本概念

**deque与vector的区别**：

- vector对于头部的插入删除效率低，数据量越大，效率越低
- deque相对而言，对头部的插入删除速度会比vector快
- vector访问元素时的速度会比deque快，这和两者内部实现有关（感觉就是vector是数组，deque是~~链表实现~~？是中控器实现）

deque迭代器依然是支持随机访问的迭代器



**deque内部工作原理**：

deque内部有个**中控器**，维护每段缓冲区中的内容，缓冲区中存放真实数据；

中控器维护的是每个缓冲区的地址，使得使用deque时像一片连续的内存空间；

即deque是由**一段一段数组**组成的；

> 中控器: ... 0x01 0x02 ...
>
> 实际缓冲区:
>
> ​	0x01: 0		0		0	   elem elem
>
> ​	0x02: elem elem elem elem elem



#### deque构造函数

- ``deque<T> d``
- ``deque(d.begin(), d.end())``
- ``deque(n, elem)``
- ``deque(const deque &deq);``

#### deque赋值操作

- ``deque& operator=(const deque &deq)``
- ``assign(d.begin(), d.end())``
- ``assign(n, elem)``

#### deque大小操作

> 和vector相同，但是没有capacity，因为deque是由中控器控制，没有容量的概念（可以无限扩）

- ``deque.empty()``
- ``deque.size()``
- ``deque.resize(num)``
- ``deque.resize(num, elem)``

#### deque插入和删除

两端插入操作：

- ``push_back(elem)``
- ``push_front(elem)``
- ``pop_back()``
- ``pop_front()``

指定位置操作：

> 这里的pos是iterator，而非index

- ``insert(pos, elem)``	
- ``insert(pos, n, elem)``
- ``insert(pos, beg, end)``
- ``clear()``
- ``erase(begin, end)``
- ``erase(pos)``

#### deque数据存取

- ``at(int idx)``
- ``operator[]``
- ``front()``
- ``back()``

#### deque排序

- ``sort(iterator begin, iterator end)``

```c++
#include<deque>
#include<algorithm>
deque<int> d;
...
sort(d.begin(), d.end());	//默认按照升序
```

对于支持随机访问的迭代器的容器都可以利用sort算法直接对其进行排序；

