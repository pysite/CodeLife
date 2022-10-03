# 第二周



## 运算符重载

运算符重载就是定义名称为类似operator+的成员函数；

### 加法运算符重载

#### 通过成员函数重载

```c++
//假设类Person有int成员m_A和m_B，同时有个成员函数operator+
Person operator+(Person &p){
    Person temp;
    temp.m_A = this->m_A + p.m_A;
    temp.m_B = this->m_B + p.m_B;
    return temp;
}
//这样就可以
Person p3 = p1 + p2;
```

#### 通过全局函数重载

```c++
Person operator+(Person &p1, Person &p2){
	Person temp;
    temp.m_A = p1.m_A + p2.m_A;
    temp.m_A = p1.m_B + p2.m_B;
    return temp;
}
//这样也可以
Person p3 = p1 + p2;
//通过这种Person operator+(Person &p1, int num)还可以实现Person p3 = p1 + 10;
```

内置的数据类型(int, double...)肯定是不能运算符重载的。

### 左移运算符重载

```c++
//只能利用全局函数重载左移运算符
ostream& operator<<(ostream &cout, Person &p){
    cout<<"m_A="<<p.m_A<<"m_A="<<p.m_B<<endl;
    return cout;	//必须把cout return回去
}
```

### 递增运算符重载

```c++
//重载前置++运算符，返回自身的引用是为了实现++(++A)之后A被加2次，如果不是引用则A的值只会被加1次
MyInteger& MyInteger::operator++(){	//MyInteger是自定义的一个类
    m_Num++;
    return *this;
}

//重载后置++运算符，只需在参数中加一个int占位参数即可
MyInteger MyInteger::operator++(int){
    MyInteger temp = *this;
    m_Num++;
    return temp;	//这里只能返回值了不能引用，不能返回局部变量的引用
}
```

### 赋值运算符重载

回顾：C++编译器至少给一个类添加4个函数

1.默认构造函数（无参，函数体为空）

2.默认析构函数（无参，函数体为空）

3.默认拷贝构造函数，对属性进行值拷贝

4.赋值运算符operator=，对属性进行值拷贝

如果类中有属性指向堆区，做赋值操作时也会出现深浅拷贝问题；

```c++
Person& Person::operator=(Person &p){
    if(m_Age != NULL){	//这里的m_Age是个int的指针
        delete m_Age;
        m_Age = NULL;
    }
    m_Age = new int(*p.m_Age);
    return *this;	//这样才可以实现p3=p2=p1这种操作
}
```

### 关系运算符重载

关系运算符就是<,>,==等

```c++
//==可以换为!=,<=,>=等各种关系符
bool Person::operator==(Person &p){
    if(this->m_Age == p.Age)
        return true;
    return false;
} 
```

### 函数调用运算符重载

- 函数调用运算符()可以重载
- 由于重载后的使用方式非常像函数的调用，因此称为仿函数
- 仿函数没有固定写法，非常灵活

```c++
class MyPrint{
public:
    //重载函数调用运算符
    void operator()(string test){
        cout<<test<<endl;
    }
}
MyPrint myPrint;
myPrint("hello world");

//仿函数非常灵活
class MyAdd{
public:
    int operator()(int num1, int num2){
        return num1+num2;
    }
}
MyAdd myAdd;
int result = myAdd(100, 100);
int result2 = MyAdd()(100, 100);	//匿名函数对象，先创建对象再调用重载的函数
```



## 继承

```c++
//B是父类（基类）
//public是继承方式，公有继承
class A:public B{	
	...
}
```

### 继承方式

公共继承，保护继承，私有继承

```c++
class A{
public:
    int a;
protected:
    int b;
private:
    int c;
}
```

此时创建A的子类B，

- 如果是公共继承，则a,b,c三种变量的权限不变；

- 如果是保护继承，则a,b是protected，而c依然是private；
- 如果是私有继承，则a,b,c都是private；

相当于继承方式**决定了父类中的成员的权限的下界**；

### 继承中的对象模型

私有成员依然是可以被继承下去的，依然是占子类大小的。

### 继承中构造和析构顺序

就是类似从内到外的那样，先构造父类，再构造子类；

析构顺序和构造顺序相反；

### 继承同名成员处理方式

当子类与父类出现同名的成员，如何通过子类对象，访问到子类或父类中同名的数据：

- 访问子类同名成员，直接访问即可
- 访问父类同名成员，需要加作用域

```c++
class A{
public:
	int x;
    void func();
    void func(int);
}
class B: public A{
public:
    int x;
    void func();
}
B b;
b.x;	//访问自身的x
b.A::x;	//访问父类的x
b.func();	//访问自身的func
b.A::func();//访问父类的func
//如果子类中出现和父类同名的成员函数，子类的同名成员会隐藏掉父类中所有同名成员
b.func(100);//非法，必须加上作用域
```

### 继承同名静态成员处理方式

静态成员和非静态成员处理方式一样。不一样的是静态成员还可通过类名访问。

```
class A{
	static int x;
}
class B:public A{
	static int x;
}
B::x;
B::A::x;
```

### 多继承

C++允许一个类继承多个类

语法：``class 子类：继承方式 父类1，继承方式 父类2...``

多继承可能引发父类中有同名成员出现，需要加作用域区分

**C++实际开发中不建议用多继承**

### 菱形继承

菱形继承是指，有类A，A的子类有B和C，而类D又同时继承了B和C，子类D继承了两份相同的数据。

使用数据就会导致不明确的问题，**需要加作用域区分**。

也可以利用**虚继承**解决菱形继承问题，此时类A就叫虚基类。虚继承使得在D中来自类A的数据只有一个。

```c++
class A{
	int x;
}
//1.加作用域解决菱形继承问题
class B:public A{}
class C:public A{}
class D:public B, public C{}
D d;
d.B::x;	//必须加上作用域来区分
d.C::x;

//2.使用虚继承解决
class B:virtual public A{}
class C:virtual public A{}
class D：public B, public C{}	//这里就不需要加virtual了
D d;
//以下这三种访问的都是同一变量
d.B::x;
d.C::x;
d.x;
```

相当于此时D从B和C中继承的只是一个指针，指向同一个x的地址。（这个指针实际上叫做vbptr，指向的是虚基类，可以通过vbptr找到同一个x）

总结：

1. 简而言之，虚基类可以使得从多个类（它们继承自一个类）中派生出的对象只继承一个对象。

2. 继承关系可能是非常繁复的。一个类可能多重继承自别的类，而它的父类也可能继承自别的类。当该类从不同的途径继承了两个或者更多的同名函数时，如果没有对类名限定为virtual，将导致二义性。当然，如果使用了虚基类，则不一定会导致二义性。编译器将**选择继承路径上“最短”的父类成员函数加以调用**。该规则与成员函数的访问控制权限并不矛盾。也就是说，不能因为具有更高调用优先级的成员函数的访问控制权限是"private"，而转而去调用public型的较低优先级的同名成员函数。（就是说一定会调用继承路径最短的那个函数）
3. 虚基类表存储的是，虚基类相对直接继承类的偏移（D并非是虚基类的直接继承类，B,C才是）
4. 虚基类表是每个类一个，类对象之间共享；

[不懂可以看这篇博客](https://blog.csdn.net/longlovefilm/article/details/80558879)

## 多态

### 多态基本概念

多态是C++面向对象三大特性之一

多态分为两类

- **静态多态**：函数重载和运算符重载属于静态多态，复用函数名
- **动态多态**：派生类和虚函数实现运行时多态

>  在本节主要学习的就是动态多态

静态多态和动态多态区别：

- 静态多态的函数地址早绑定 - 编译阶段确定函数地址
- 动态多态的函数地址晚绑定 - 运行阶段确定函数地址

父类引用指向子类对象

```c++
class Animal{
public:
    void speak(){...}
}
class Cat:public Animal{
public:
    void speak(){...}
}
void doSpeak(Animal &animal){
    animal.speak();
}
//地址早绑定，在编译阶段确定函数地址
Cat cat;
doSpeak(cat);//这里调用的是Animal的speak

//如果要实现Cat的speak，必须将Animal类中的speak函数改为virtual函数
class Animal{
public:
    virtual void speak(){...}
}
class Cat:public Animal{
public:
    void speak(){...}
}
```

**总结**：

1. 只需将基类中的成员函数声明为虚函数即可，派生类中重写的virtual函数自动成为虚函数;

2. 基类中的析构函数必须为虚函数，否则会出现对象释放错误。以上例说明，如果不将基类的析构函数声明为virtual，那么在调用delete animal语句时将调用基类的析构函数，而不是应当调用的派生类的析构函数，从而出现对象释放错误的问题。

3. 虚函数的使用将导致类对象占用更大的内存空间。对这一点的解释涉及到虚函数调用的原理：编译器给每一个包括虚函数的对象添加了一个隐藏成员：指向虚函数表的指针。虚函数表（virtual function table）包含了虚函数的地址，由所有虚函数对象共享。当派生类重新定义虚函数时，则将该函数的地址添加到虚函数表中。无论一个类对象中定义了多少个虚函数，虚函数指针只有一个。相应地，每个对象在内存中的大小要比没有虚函数时大4个字节（32位主机，不包括虚析构函数）。如下：

   ```c++
   cout<<sizeof(base)<<endl;           //输出12
   cout<<sizeof(inheriter)<<endl;		//输出12
   ```

   base类中包括了两个整型的成员变量，各占4个字节大小，再加上一个虚函数指针，共计占12个字节；inheriter类继承了base类的两个成员变量以及虚函数表指针，因此大小与基类一致。如果inheriter多重继承自另外一个也包括了虚函数的基类，那么隐藏成员就包括了两个虚函数表指针。

4. 虚函数表是每个类一个，类对象之间共享；



**相当于继承的时候会把vbptr和vfptr都继承下来**。

动态多态要满足的条件：

1. 有继承关系
2. 子类重写父类的虚函数

动态多态的使用：

- 父类的指针或者引用指向子类对象

### 多态的原理剖析

```c++
class Animal{
public:
	void speak();
}
sizeof(Animal);	//结果为1，空类的大小为1
```

```c++
class Animal{
public:
	virtual void speak(){}
}
sizeof(Animal);	//结果为4，虚函数表指针

class Cat:public Animal{
public:
    //继承了原父类中的vfptr，但是虚函数表中原来Animal的speak函数已经改为了Cat的Speak函数
    virtual void speak(){}
}
```

### 纯虚函数

在多态中，通常父类中虚函数的实现是毫无意义的，主要都是调用子类重写，因此可将虚函数进一步改为纯虚函数；

纯虚函数语法：``virtual 返回值类型 函数名 (参数列表) = 0;``

含有纯虚函数的类叫**抽象类**；

**抽象类特点**：

- 无法实例化对象
- 子类必须重写抽象类中的纯虚函数，否则也属于抽象类

### 虚析构和纯虚析构

多态使用时，如果子类中有属性开辟到堆区，那么父类指针在释放时无法调用到子类的析构代码

如果子类中没有堆区数据，可以不写为虚析构或纯虚析构；



解决方式：将父类中的析构函数改为**虚析构**或者**纯虚析构**

虚析构和纯虚析构共性：

- 可以解决父类指针释放子类对象
- 都需要有具体的函数实现

虚析构和纯虚析构区别：

- 如果是纯虚析构，该类属于抽象类，无法实例化对象

虚析构语法：

``virtual ~类名(){}``

纯虚析构语法：

``virtual ~类名() = 0;``不同的是，纯虚析构函数必须要有实现，但是它又能使类变为抽象类。

```c++
class A{
    virtual ~A()=0;
}
A::~A(){}	//纯虚析构函数必须要有实现，同时A又是个抽象类，不可实例化
```



## 文件操作

C++中对文件操作需要包含头文件``<fstream>``

**文件类型分为两种**：

1. 文本文件     - 文件以文本的ASCII码形式存储在计算机中
2. 二进制文件 - 文件以文本的二进制形式存储在计算机中，用户一般不能直接读懂它们

操作文件的三大类：

1. ofstream：写操作
2. ifstream：读操作
3. fstream：读写操作



### 文本文件

#### 写文件

写文件步骤如下：

1. 包含头文件

   ``#include <fstream>``

2. 创建流对象

   ``ofstream ofs;``

3. 打开文件

   ``ofs.open("文件路径",打开方式);``

4. 写数据

   ``ofs << "写入的数据";``

5. 关闭文件

   ``ofs.close();``

| 打开方式    | 解释                       |
| ----------- | -------------------------- |
| ios::in     | 读                         |
| ios::out    | 写                         |
| ios::ate    | 初始位置：文件尾           |
| ios::app    | 追加方式写文件             |
| ios::trunc  | 如果文件存在先删除，再创建 |
| ios::binary | 以二进制方式操作           |

注意：文件打开方式可以配合使用，利用|操作符

例如：用二进制方式写文件 ``ios::binary|ios::out``

#### 读文件

读文件与写文件步骤类似，但是读取方式相对于比较多

读文件步骤如下：

1. 包含头文件

   ``#include <fstream>``

2. 创建流对象

   ``ifstream ifs;``

3. 打开文件并判断文件是否打开成功

   ``ifs.open("文件路径",打开方式);``

   ``if(ifs.is_open()){}``

4. 读数据

   四种方式读取

   1. 第一种

      ```c++
      char buf[1024] = {};
      while(ifs >> buf){...}
      ```

   2. 第二种

      ```c++
      char buf[1024] = {};
      while(ifs.getline(buf, sizeof(buf))){...}
      ```

   3. 第三种

      ```c++
      string buf;
      while(getline(ifs, buf)){...}
      ```

   4. 第四种（不推荐

      ```c++
      char c;
      while((c = ifs.get()) != EOF){...}
      ```

5. 关闭文件

   ``ifs.close()``

### 二进制文件

#### 写文件

二进制方式写文件主要利用流对象调用成员函数write

函数原型：``ostream& write(const char * buffer, int len);``

参数解释：字符指针buffer指向内存中一段存储空间。len是读写的字节数。

```c++
class Person{
public:
    char m_Name[64];
    int m_Age;
}

ofstream ofs("test.txt", ios::out | ios:binary);	//把open那一步可以合并
//将一个类Person的对象写入文件
Person p = {"张三", 18};
ofs.write((const char*)&p, sizeof(Person));
ofs.close();
```

#### 读文件

二进制方式读文件主要利用流对象调用成员函数read

函数原型：``istream& read(char* buffer, int len);``

参数解释：字符指针buffer指向内存中一段存储空间，len是读写的字节数

```c++
ifstream if("test.txt", ios::in | ios::binary);
Person p;
ifs.read((char *)&p, sizeof(Person));
ifs.close();
```



