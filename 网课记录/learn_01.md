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

相当于此时D从B和C中继承的只是一个指针，指向同一个x的地址。（这个指针实际上叫做vbptr，可以通过vbptr找到同一个x）