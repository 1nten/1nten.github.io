---
title: "[基础]C/C++"
date: 2021-03-8T15:11:34+08:00
draft: false
---

#### 裸函数

```
void __declspec(naked) Function()  //编译器不执行格外操作
{
  __asm                            //在c中调用汇编格式__asm{}
  {
     ret 
  }                     
                                   //需要手动写ret才能正常运行
}
```

常见的几种调用约定：

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528155637.png)

__cdecl:默认调用约定。外平栈。

__stdcall:内平栈。

__fastcall:2个参数以内时，直接用寄存器存取数据，提高执行速度。内平栈。

#### 浮点数的存储

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528155641.png)

以12.5为例

**符号部分：正数为0，负数为1**

12=>> c =>> 1100

0.5 乘二取整 1.0 =>>1

12.5 =>> 1100.1 =>> 1.[1001](http://。/) * 10^3

**尾数部分：** [**1001** ](http://。/)0000000000000000000

指数部分：127 + 3 = 130 =>> 1000 0010 //127 + 指数计算即可

0 **1000 0010** [**1001** ](http://。/)**0000000000000000000**

#### 内存图

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528155645.png)

#### 全局变量&局部变量

```
mov [ arr (00427326)] ，eax  //全局变量

mov dword ptr [ebp-4] , 6Ah  //局部变量
mov dword ptr [ebp-8], 14h
```

全局变量

1、全局变量在程序编译完成后地址就已经确定下来了，只要程序启动，全局变量就已经存在了，启动后里面

2、全局变量的值可以被所有函数所修改，里面存储的是最后一次修改的值.

3、全局变量所占内存会一直存在，知道整个进程结束.

4、全局变量的反汇编识别：

MOV寄存器，byte/word/dword ptr ds: [0x12345678]
通过寄存器的宽度，或者byte/word/dword来判断全局变量的宽度

[全局变量就是所谓的基址](http:)

局部变量的特点：

1、局部变量在程序编译完成后并没有分配固定的地址.

2、在所属的方法没有被调用时，局部变量并不会分配内存地址，只有当所属的程序被调用了，才会在堆栈中分配内存.

3、当局部变量所属的方法执行完毕后，局部变量所占用的内存将变成垃圾数据.局部变量消失.

4、局部变量只能在方法内部使用，函数A无法使用函数B的局部变量.

5、局部变量的反汇编识别：

[ebp-4] [ebp-8] [ebp-0xC]

#### if else 逆向

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528155650.png)

```
#include <iostream>
int a;
void test1(int x, int y) {
	int z = a;
	if (x <= y) {
		a = x + y;
	}
	return;
}
int a2;
void test2(int d, int e) {
	int b = a2;
	int c=2;
	
	if (d >= e)
		c = c + 1;
	if (d < e)
		a = c;
	else
		a = b + c;
}
int main() {
	int x = 1, y = 2;
	//test1(x, y);
	test2(x, y);
	return 0;
}
```

####  **数组缓冲区溢出**

```
void Fun()			
{			
	int arr[5] = {1,2,3,4,5};		
			
	arr[6] = (int)HelloWord;		
			
}			
```

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528155653.png)call 指令 堆栈图

个人理解：

当数组作为局部变量时，在缓存区中是从上往下存的。如：定义int ARRAY[3]={0,1,2}.在堆栈图如图这样存储。

这时，给ARRAY[2],ARRAY[3]赋值就会覆盖EBP,EIP的值，造成缓存区溢出。



#### switch&where&for&do....where反汇编

switch

1、switch语句 是if语句的简写，case后面的值只能是整数，不允许变量

![img](image-1024x383.png)

2、添加case后面的值，一个一个增加，观察反汇编代码的变化(何时生成大表).

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528155656.png)

> 当case **后面的值** 连续的n个时，会生成大表（节省内存）

3、将3中的常量值的顺序打乱，观察反汇编代码(观察顺序是否会影响生成大表).

> 例如0，1，2，3，4，50，不生成大表

4、将case后面的值改成从100开始到109，观察汇编变化(观察值较大时是否生成大表).

> **生成大表** ，sub 100

5、将连续的10项中抹去1项或者2项，观察反汇编有无变化(观察大表空缺位置的处理).

> 抹去值较少时，只生成大表，空缺的地方会用default执行语句的地址


6、在10项中连续抹去，不要抹去最大值和最小值(观察何时生成小表).
7、将case后面常量表达式改成亳不连续的值，观察反汇编变化.

> **抹去值较多时，生成大表，同时生成小表，存default最后一位的地址**

总结：使用switch语句时，case尽量连续，编译器会生成大表提高效率。如果case不规律switch基本等同于if语句。区别在于switch反汇编会把比较条件放在一块，而if...else比较条件和执行内容相间。

### 指针

```
//实现数组值的互换
#include<stdio.h>
void Fun4()
{
    int arr[5] = { 1,2,3,4,5 };

    //..此处添加代码，使用指针，将数组的值倒置
    int* p =arr;
    for(int i = 0; i<(5/2); i++)
    {
        int t = *(p+i);
        *(p+i) = *(p+4-i);
        *(p+4-i) = t;
    }

    for (int k =0;k<5;++k)
    {
        printf("%d\n", *(p + k));
    }
}
int main(){
    Fun4();
    return 0 ;
}
```

这一堆数据中存储了角色的血值信息，假设血值的类型为int类型，值为100(10进制)请列出所有可能的值以及该值对应的地址.

方法一：

```
#include<stdio.h>

char data[100] = {
    0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x08,0x09,
    0x00,0x20,0x10,0x03,0x03,0x0c,0x00,0x00,0x33,0x00,
    0x00,0x33,0x00,0x47,0x0c,0x0e,0x00,0x0d,0x00,0x11,
    0x00,0x00,0x00,0x02,0x64,0x00,0x00,0x00,0xaa,0x00,
    0x00,0x00,0x64,0x10,0x00,0x00,0x00,0x00,0x00,0x00,
    0x00,0x01,0x02,0x00,0x74,0x0f,0x41,0x00,0x00,0x00,
    0x01,0x01,0x02,0x00,0x05,0x00,0x00,0x00,0x0a,0x00,
    0x00,0x02,0x74,0x0f,0x41,0x00,0x06,0x08,0x00,0x00,
    0x00,0x00,0x00,0x64,0x00,0x0f,0x00,0x00,0x0d,0x00,
    0x00,0x00,0x23,0x00,0x00,0x64,0x00,0x00,0x64,0x00
};

//假设该数据排列是由地位到高位
void FindBloodAddr(char* p[])
{
    int k = 0;
    //memset(p, 0, sizeof(char*) * 10);

    for (int i = 0; i < 100-3;)
    {
        if (data[i] == 0x64)
        {  //小端字节序所以是查,0x64,0x00,0x00,0x00,
            if (data[i + 1] == 0x00 && data[i + 2] == 0x00 && data[i + 3] == 0x00)//if (data[i -1] == 0x00 && data[i - 2] == 0x00 && data[i - 3] == 0x00) //
            {
                p[k++] = &data[i];
                printf("%d\n",i);
                i += 4;
                continue;
            }
            i++;
        }
        else
        {
            ++i;
        }
    }
}

int main()
{
    char* p[10];
    FindBloodAddr(p);

    return 0;
}
```

方法二：

```
#include<stdio.h>
void Function()
{
	char date[100] = {
		0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x07,0x09,
		0x00,0x20,0x10,0x03,0x03,0x0C,0x00,0x00,0x44,0x00,
		0x00,0x33,0x00,0x47,0x0C,0x0E,0x00,0x0D,0x00,0x11,
		0x00,0x00,0x00,0x02,0x64,0x00,0x00,0x00,0xAA,0x00,
		0x00,0x00,0x64,0x10,0x00,0x00,0x00,0x00,0x00,0x00,
		0x00,0x00,0x02,0x00,0x74,0x0F,0x41,0x00,0x00,0x00,
		0x01,0x00,0x00,0x00,0x05,0x00,0x00,0x00,0x0A,0x00,
		0x00,0x02,0x57,0x4F,0x57,0x00,0x06,0x08,0x00,0x00,
		0x00,0x00,0x00,0x64,0x00,0x0F,0x00,0x00,0x0D,0x00,
		0x00,0x00,0x23,0x00,0x00,0x64,0x00,0x00,0x64,0x00
		};

	char* p=date;
	int* x;
	for(int i=0; i<96; i++)
	{
		x=(int*)(p+i);
		if(*x == 0x64)
			printf("%x对应的地址是%x下标为%d",*x,x,i);

	}
}
int main()
{
    Function();

    return 0;
}
#include<stdio.h>
char gameData[] = {
    0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x07,0x09,
    0x00,0x20,0x10,0x03,0x03,0x0c,0x00,0x00,0x44,0x00,
    0x00,0x33,0x00,0x47,0x0c,0x0e,0x00,0x0d,0x00,0x11,
    0x00,0x00,0x00,0x02,0x64,0x00,0x00,0x00,0xaa,0x00,
    0x00,0x00,0x64,0x10,0x00,0x00,0x00,0x00,0x00,0x00,
    0x00,0x00,0x02,0x00,0x74,0x0f,0x41,0x00,0x00,0x00,
    0x01,0x00,0x00,0x00,0x05,0x00,0x00,0x00,0x0a,0x00,
    0x00,0x02,0x57,0x4f,0x57,0x00,0x06,0x08,0x00,0x00,
    0x00,0x00,0x00,0x64,0x00,0x0f,0x00,0x00,0x0d,0x00,
    0x00,0x00,0x23,0x00,0x00,0x64,0x00,0x00,0x64,0x00
};
char* FindRoleNameAddr(char* pData, char* pRoleName)
{
    char* ret = 0;
    int i = 100;
    while(i--)
    {
        while (*(ret=pData++) == *pRoleName)
        {
            char *s1 = ret, *s2 = pRoleName;
            while (*s1++ == *s2++)
            {
                if (!*s2)
                    return ret;
            }
        }
    }
    return 0;
}
void printRoleName(char* pData)
{
    int i = 100;
    while (i--)
    {
        while ((*pData>'A'&&*pData<'Z') || (*pData>'a'&&*pData<'z'))
        {
            printf("%c",*pData++);
        }
        pData++;
        printf("\n");
    }
}


int main()
{
    char* x=FindRoleNameAddr(gameData, "WOW");
    printf("%d\n",x);
    printRoleName(gameData);

    return 0;
}
1.创建一个int* arr[5]数组，并为数组赋值（使用&）
void Fun1()
{
    int* arr[5];
    int a1 = 10;
    int a2 = 20;
    int a3 = 30;
    int a4 = 40;
    int a5 = 50;

    arr[0] = &a1;
    arr[1] = &a2;
    arr[2] = &a3;
    arr[3] = &a4;
    arr[4] = &a5;
}

2.创建一个字符指针数组，存储所有的C的关键词，并全部打印出来
void Fun2()
{
    char* keyword[] = {
      "auto","double","int","struct","break","else", "long" ,"switch",
      "case" ,"enum" ,"register", "typedef" ,"char", "extern" ,"return" ,"union",
      "const", "float", "short", "unsigned", "continue", "for", "signed", "void",
      "default", "goto", "sizeof", "volatile", "do", "if", "while", "static" };

    for(int i = 0; i < 32 ; i++)
    {
        printf("%s, ", keyword[i]);   
    }
}

3.查找这些数据中，有几个id=1 level=8的结构体信息  
void Fun3()
{
    char data[] = {
        0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x07,0x09,
        0x00,0x20,0x10,0x03,0x03,0x0c,0x00,0x00,0x44,0x00,
        0x00,0x33,0x01,0x00,0x00,0x08,0x00,0x00,0x00,0x00,
        0x00,0x00,0x00,0x02,0x64,0x00,0x00,0x00,0xaa,0x00,
        0x00,0x00,0x64,0x01,0x00,0x00,0x00,0x08,0x00,0x00,
        0x00,0x00,0x02,0x00,0x74,0x0f,0x41,0x00,0x00,0x00,
        0x01,0x00,0x00,0x00,0x05,0x00,0x00,0x00,0x0a,0x00,
        0x00,0x02,0x57,0x4f,0x57,0x00,0x06,0x08,0x00,0x00,
        0x00,0x00,0x00,0x64,0x00,0x0f,0x00,0x00,0x00,0x00,
        0x00,0x00,0x23,0x00,0x00,0x64,0x00,0x00,0x64,0x00 };

    int count = 0;
    Player _target = { 1,8 };
    char *_data = data;
    char* ptarget = &_target;

    int i = 100;
    while (i--)
    {
        if (*_data++ == *ptarget)
        {
            char *s1 = _data, *s2 = ptarget + 1;
            int j = sizeof(Player) - 1;
            while (*s1++ == *s2++)
            {
                j--;
                if (!j)
                {
                    count++;
                    break;
                }
            }
        }
    }
}
```

#### 结构体指针

```
题目：查找这些数据中，有几个id=1 level=8的结构体信息。	
	
0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x07,0x09,	
0x00,0x20,0x10,0x03,0x03,0x0C,0x00,0x00,0x44,0x00,	
0x00,0x33,0x01,0x00,0x00,0x08,0x00,0x00,0x00,0x00,	
0x00,0x00,0x00,0x02,0x64,0x00,0x00,0x00,0xAA,0x00,	
0x00,0x00,0x64,0x01,0x00,0x00,0x00,0x08,0x00,0x00,	
0x00,0x00,0x02,0x00,0x74,0x0F,0x41,0x00,0x00,0x00,	
0x01,0x00,0x00,0x00,0x05,0x00,0x00,0x00,0x0A,0x00,	
0x00,0x02,0x57,0x4F,0x57,0x00,0x06,0x08,0x00,0x00,	
0x00,0x00,0x00,0x64,0x00,0x0F,0x00,0x00,0x0D,0x00,	
0x00,0x00,0x23,0x00,0x00,0x64,0x00,0x00,0x64,0x00	
	
结构体定义如下：	
	
typedef struct TagPlayer	
{	
	int id;
	int level;
}Player;	
#include "strxx.h"
#include <iostream>
#include <stdio.h>
typedef struct TagPlayer
{
    int id;
    int level;
}Player;

int main()
{
    unsigned char Mem[] = { 0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x07,0x09,
        0x00,0x20,0x10,0x03,0x03,0x0C,0x00,0x00,0x44,0x00,
        0x00,0x33,0x01,0x00,0x00,0x08,0x00,0x00,0x00,0x00,
        0x00,0x00,0x00,0x02,0x64,0x00,0x00,0x00,0xAA,0x00,
        0x00,0x00,0x64,0x01,0x00,0x00,0x00,0x08,0x00,0x00,
        0x00,0x00,0x02,0x00,0x74,0x0F,0x41,0x00,0x00,0x00,
        0x01,0x00,0x00,0x00,0x05,0x00,0x00,0x00,0x0A,0x00,
        0x00,0x02,0x57,0x4F,0x57,0x00,0x06,0x08,0x00,0x00,
        0x00,0x00,0x00,0x64,0x00,0x0F,0x00,0x00,0x0D,0x00,
        0x00,0x00,0x23,0x00,0x00,0x64,0x00,0x00,0x64,0x00 };
    Player *sPlayer;
    int nSum = 0;
    for (int i=0;i<sizeof(Mem)-sizeof(Player);i++)
    {
        sPlayer = (Player *)&Mem[i];
        if (1 == sPlayer->id && 8 == sPlayer->level)
            nSum++;
    }
    printf("结构体个数为%d\n", nSum);
    getchar();
    return 0;
}
```

#### 数组指针

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528155705.png)

# c++

### 引用

```
text(a)  // a 可能为int int&
```

继承是数据的复制

模板是代码的复制

### 运算符重载&友元

友元

什么情况下需要友元函数:
(1) 运算符重载的某些场合需要使用友元.
(2)两个类要共享数据的时候.
友元函数和类的成员函数的区别：
(1)成员函数有this指针，而友元函数没有this指针
(2)友元函数是不能被继承的，就像父亲的朋友未必是儿子的

1、运算符重载就是函数替换

2、. :: ?: sizeof # 不能重载

```
引用类型								
								
struct Base								
{								
	int x;							
	int y;							
	Base(int x,int y)							
	{							
		this->x = x;						
		this->y = y;						
	}							
};								
void PrintByPoint(Base* pb)								
{								
	printf("%d  %d\n",pb->x,pb->y);							
								
	pb = (Base*)0x123456;							
								
	//为所欲为...							
}								
void PrintByRef(Base& refb,Base* pb)								
{								
	printf("%d  %d\n",refb.x,refb.y);							
	Base b1(21,31);							
	//&refb = b1; //引用不能重新赋值							
	refb = b1;  //这个不是重新赋值，这个是把b1的值赋给refb代表的对象							
	printf("%d  %d\n",pb->x,pb->y);							
}								
								
为了避免出现这种情况，可以将refb声明为常量，不可修改：								
void PrintByRef(const Base& refb)								
{								
	printf("%d  %d\n",refb.x,refb.y);							
	Base b1(21,31);							
	//&refb = b1; //引用不能重新赋值							
	//refb = b1;  //不允许							
}								
								
								
int main(int argc, char* argv[])								
{								
	Base base(1,2);							
								
	PrintByRef(base,&base);							
								
	return 0;							
}								
								
								
总结：								
								
1、引用类型是C++里面的类型								
								
2、引用类型只能赋值一次，不能重新赋值								
								
3、引用只是变量的一个别名.								
								
4、引用可以理解成是编译器维护的一个指针，但并不占用空间(如何去理解这句话？).								
								
5、使用引用可以像指针那样去访问、修改对象的内容，但更加安全.								
```

### Vector动态数组实现

> [记一个坑](http://inten.xyz/index.php/2020/07/31/c/#)： **在myclass.cpp（实现函数具体功能）和对应的myclass.h（声明函数）**使用template <class T_ELE>时，vs2019会报错“无法解析的外部符号”。
>
> 原因：编译器编译时会给函数生成新的名字，两个文件的函数名不一样就会报错。（待求证）
>
> 解决方案：将函数的声明和具体实现写在同一个文件中。

```
#include<stdio.h>
#include<Windows.h>

#define SUCCESS           			 1 // 成功								
#define ERROR            			 -1 // 失败								
#define MALLOC_ERROR			 -2 // 申请内存失败								
#define INDEX_ERROR		 	 -3 // 错误的索引号								

template <class T_ELE>
class Vector
{
public:
	Vector();
	Vector(DWORD dwSize);
	~Vector();
public:
	DWORD	at(DWORD dwIndex, OUT T_ELE* pEle);					//根据给定的索引得到元素				
	DWORD    push_back(T_ELE Element);						//将元素存储到容器最后一个位置				
	VOID	pop_back();					//删除最后一个元素				
	DWORD	insert(DWORD dwIndex, T_ELE Element);					//向指定位置新增一个元素				
	DWORD	capacity();					//返回在不增容的情况下，还能存储多少元素				
	VOID	clear();					//清空所有元素				
	BOOL	empty();					//判断Vector是否为空 返回true时为空				
	DWORD	erase(DWORD dwIndex);					//删除指定元素				
	DWORD	size();					//返回Vector元素数量的大小				
private:
	BOOL	expand();
private:
	DWORD  m_dwIndex;						//下一个可用索引				
	DWORD  m_dwIncrement;						//每次增容的大小				
	DWORD  m_dwLen;						//当前容器的长度				
	DWORD  m_dwInitSize;						//默认初始化大小				
	T_ELE* m_pVector;						//容器指针				
};

template <class T_ELE>
Vector<T_ELE>::Vector()
	:m_dwInitSize(10), m_dwIncrement(5)
{
	//1.创建长度为m_dwInitSize个T_ELE对象										
	m_pVector = new T_ELE[m_dwInitSize];
	//2.将新创建的空间初始化										
	memset(m_pVector, 0, sizeof(T_ELE) * m_dwInitSize);
	//3.设置其他值									
	m_dwIndex = 0;
	m_dwLen = m_dwInitSize;
}

template <class T_ELE>
Vector<T_ELE>::Vector(DWORD dwSize)
	:m_dwIncrement(5)
{
	// 1.创建长度为m_dwInitSize个T_ELE对象
	m_pVector = new T_ELE[dwSize];
	//2.将新创建的空间初始化										
	memset(m_pVector, 0, sizeof(T_ELE) * dwSize);
	//3.设置其他值									
	m_dwIndex = 0;
	m_dwLen = dwSize;

}
template <class T_ELE>
Vector<T_ELE>::~Vector()
{
	//释放空间 delete[]										
	delete[] m_pVector;
	m_pVector = NULL;
}

template <class T_ELE>
BOOL Vector<T_ELE>::expand()
{
	DWORD dwLenTemp = 0;
	T_ELE* pVectorTemp = NULL;
	// 1. 计算增加后的长度										
	dwLenTemp = m_dwLen + m_dwIncrement;
	// 2. 申请空间										
	pVectorTemp = new T_ELE[dwLenTemp];
	memset(pVectorTemp, 0, sizeof(T_ELE) * dwLenTemp);
	// 3. 将数据复制到新的空间										
	memcpy(pVectorTemp, m_pVector, sizeof(T_ELE) * m_dwLen);
	// 4. 释放原来空间		
	delete[] m_pVector;
	m_pVector = pVectorTemp;
	pVectorTemp = NULL;
	// 5. 为各种属性赋值
	m_dwLen = dwLenTemp;

	return SUCCESS;
}

template <class T_ELE>
DWORD  Vector<T_ELE>::push_back(T_ELE Element)
{
	//1.判断是否需要增容，如果需要就调用增容的函数										
	if (m_dwIndex >= m_dwLen)
	{
		expand();
	}
	//2.将新的元素复制到容器的最后一个位置										
	memcpy(&m_pVector[m_dwIndex], &Element, sizeof(T_ELE));
	//3.修改属性值										
	m_dwIndex++;

	return SUCCESS;
}

template<class T_ELE>
VOID Vector<T_ELE>::pop_back()                  //删除最后一个元素		
{
	memset(&m_pVector[--m_dwIndex], 0, sizeof(T_ELE));
}

template <class T_ELE>
DWORD  Vector<T_ELE>::insert(DWORD dwIndex, T_ELE Element)
{
	//1.判断索引是否在合理区间										
	if (dwIndex<0 || dwIndex>m_dwIndex)
	{
		return INDEX_ERROR;
	}
	//2.判断是否需要增容，如果需要就调用增容的函数										
	if (m_dwIndex >= m_dwLen)
	{
		expand();
	}
	//3.将dwIndex只后的元素后移										
	for (DWORD i = m_dwIndex; i > dwIndex; i--)
	{
		memcpy(&m_pVector[i], &m_pVector[i - 1], sizeof(T_ELE));
	}
	//4.将Element元素复制到dwIndex位置										
	memcpy(&m_pVector[dwIndex], &Element, sizeof(T_ELE));
	//5.修改属性值	
	m_dwIndex++;

	return SUCCESS;
}

template<class T_ELE>
DWORD Vector<T_ELE>::capacity()
{
	DWORD capacity = 0;
	capacity = m_dwLen - m_dwIndex;

	return capacity;
}

template <class T_ELE>
DWORD Vector<T_ELE>::at(DWORD dwIndex, OUT T_ELE* pEle)
{
	//判断索引是否在合理区间										
	if (dwIndex<0 || dwIndex>m_dwIndex)
	{
		return INDEX_ERROR;
	}
	//将dwIndex的值复制到pEle指定的内存										
	memcpy(pEle, &m_pVector[dwIndex], sizeof(T_ELE));

	return SUCCESS;
}

template<class T_ELE>
VOID Vector<T_ELE>::clear()           //清空所有元素
{
	memset(m_pVector, 0, sizeof(T_ELE) * m_dwIndex);
	//delete[] m_pVector;
	m_dwIndex = 0;
	m_dwLen = 0;
	//m_pVector = NULL;
}

template<class T_ELE>
BOOL Vector<T_ELE>::empty()
{
	if (m_dwIndex != 0 || m_pVector != NULL)
	{
		return FALSE;
	}
	return TRUE;
}

template<class T_ELE>
DWORD Vector<T_ELE>::erase(DWORD dwIndex)
{
	if (dwIndex<0 || dwIndex>m_dwIndex)
	{
		return INDEX_ERROR;
	}
	for (DWORD i = dwIndex; i < m_dwIndex; i++)
	{
		memcpy(&m_pVector[i], &m_pVector[i + 1], sizeof(T_ELE));
	}
	m_dwIndex--;

	return SUCCESS;
}

template<class T_ELE>
DWORD Vector<T_ELE>::size()                 //返回Vector元素数量的大小
{
	return m_dwLen;
}


VOID TestVector()
{
	int at = 0;
	Vector<int>* pVector = new Vector<int>(5);

	//printf("capacity: %x\n", pVector->capacity());
	//printf("size: %x\n", pVector->size());

	pVector->push_back(1);
	pVector->insert(0, 2);
	//printf("capacity: %x\n", pVector->capacity());
	pVector->push_back(2);
	//printf("capacity: %x\n", pVector->capacity());
	pVector->push_back(3);
	//printf("capacity: %x\n", pVector->capacity());
	pVector->push_back(4);
	pVector->insert(4, 6);
	pVector->insert(1, 7);
	//printf("at: %x\n", pVector->at(7,&at));
	//printf("capacity: %x\n", pVector->capacity());
	pVector->push_back(5);
	pVector->insert(5, 8);
	//printf("capacity: %x\n", pVector->capacity());
	pVector->erase(1);
	pVector->erase(2);
	pVector->erase(3);

	pVector->push_back(6);
	//printf("size: %x\n", pVector->size());

	//printf("capacity: %x\n", pVector->capacity());
	pVector->push_back(7);
	pVector->push_back(8);
	pVector->push_back(9);
	pVector->push_back(10);
	pVector->insert(5, 11);
	pVector->push_back(11);
	printf("capacity: %x\n", pVector->capacity());
	printf("size: %x\n", pVector->size());
	pVector->pop_back();
	pVector->pop_back();
	pVector->pop_back();
	printf("size: %x\n", pVector->size());
	pVector->pop_back();
	pVector->pop_back();
	pVector->pop_back();
	printf("size: %x\n", pVector->size());
	pVector->clear();
	printf("size: %x\n", pVector->size());


	pVector->push_back(1);
	pVector->insert(0, 2);
	printf("size: %x\n", pVector->size());
	pVector->push_back(2);
	printf("size: %x\n", pVector->size());
	pVector->push_back(3);
	printf("size: %x\n", pVector->size());
	pVector->push_back(4);
	pVector->insert(4, 6);
	pVector->insert(1, 7);
	printf("size: %x\n", pVector->size());
	//printf("capacity: %x\n", pVector->capacity());
	pVector->push_back(5);
	pVector->insert(5, 8);
	//printf("capacity: %x\n", pVector->capacity());
	pVector->erase(1);
	pVector->erase(2);
	pVector->erase(3);

	pVector->push_back(6);
	//printf("size: %x\n", pVector->size());

	//printf("capacity: %x\n", pVector->capacity());
	pVector->push_back(7);
	pVector->push_back(8);
	pVector->push_back(9);
	pVector->push_back(10);
	pVector->insert(5, 11);
	pVector->push_back(11);
	printf("capacity: %x\n", pVector->capacity());
	printf("size: %x\n", pVector->size());
	pVector->pop_back();
	pVector->pop_back();
	pVector->pop_back();
	printf("size: %x\n", pVector->size());
	pVector->pop_back();
	pVector->pop_back();
	pVector->pop_back();
}

int main()
{
	TestVector();

	return 0;
}
```

### 链表

```
#include<stdio.h>
#include<Windows.h>

#define SUCCESS           1 // 执行成功											
#define ERROR            -1 // 执行失败											
#define INDEX_IS_ERROR   -2 // 错误的索引号											
#define BUFFER_IS_EMPTY  -3 // 缓冲区已空											

template<class T_ELE>
class NODE
{
public:
	T_ELE  Data;
	NODE<T_ELE>* pNext;
};

template <class T_ELE>
class LinkedList :public NODE<T_ELE>
{
public:
	LinkedList();                           //默认构造函数
	~LinkedList();
public:
	BOOL  IsEmpty();						//判断链表是否为空 空返回1 非空返回0				
	DWORD  Clear();						//清空链表				
	DWORD GetElement(IN DWORD dwIndex, OUT T_ELE& Element);						//根据索引获取元素				
	DWORD GetElementIndex(IN T_ELE& Element);						//根据元素获取链表中的索引				
	DWORD Insert(IN T_ELE Element);						//新增元素				
	DWORD Insert(IN DWORD dwIndex, IN T_ELE Element);						//根据索引新增元素				
	DWORD Delete(IN DWORD dwIndex);						//根据索引删除元素				
	DWORD GetSize();						//获取链表中元素的数量
	VOID Show();
private:
	NODE<T_ELE>* m_head;						//链表头指针，指向第一个节点				
	DWORD m_dwLength;						//元素的数量				
};

//无参构造函数 初始化成员									
template<class T_ELE> LinkedList<T_ELE>::LinkedList()
	:m_head(NULL), m_dwLength(0)
{

}

//析构函数 清空元素									
template<class T_ELE> LinkedList<T_ELE>::~LinkedList()
{
	Clear();
}

//判断链表是否为空									
template<class T_ELE> BOOL LinkedList<T_ELE>::IsEmpty()
{
	if (m_head == NULL || m_dwLength == 0)
	{
		return TRUE;
	}
	else
	{
		return FALSE;
	}
}

//清空链表									
template<class T_ELE> DWORD LinkedList<T_ELE>::Clear()
{
	// 1. 判断链表是否为空								
	if (m_head == NULL || m_dwLength == 0)
	{
		return BUFFER_IS_EMPTY;
	}
	// 2. 循环删除链表中的节点								
	NODE<T_ELE>* pTempNode = m_head;
	for (DWORD i = 0; i < m_dwLength; i++)
	{
		NODE<T_ELE>* pIterator = pTempNode;
		pTempNode = pTempNode->pNext;
		delete pIterator;
	}
	// 3. 删除最后一个节点并将链表长度置为0		
	m_head = NULL;
	m_dwLength = 0;
	return SUCCESS;
}

//根据索引获取元素									
template<class T_ELE> DWORD LinkedList<T_ELE>::GetElement(IN DWORD dwIndex, OUT T_ELE& Element)
{
	NODE<T_ELE>* pTempNode = NULL;
	// 1. 判断索引是否有效								
	if (dwIndex<0 || dwIndex>m_dwLength)
	{
		return INDEX_IS_ERROR;
	}
	// 2. 取得索引指向的节点								
	pTempNode = m_head;
	for (DWORD i = 0; i < dwIndex; i++)
	{
		pTempNode = pTempNode->pNext;
	}
	// 3. 将索引指向节点的值复制到OUT参数								
	memcpy(&Element, &pTempNode->Data, sizeof(T_ELE));
	return SUCCESS;
}

//根据元素内容获取索引									
template<class T_ELE> DWORD LinkedList<T_ELE>::GetElementIndex(IN T_ELE& Element)
{
	NODE<T_ELE>* pTempNode = NULL;
	// 1. 判断链表是否为空	
	if (m_head == NULL || m_dwLength == 0)
	{
		return INDEX_IS_ERROR;
	}
	// 2. 循环遍历链表，找到与Element相同的元素								
	pTempNode = m_head;
	for (DWORD i = 0; i < m_dwLength; i++)
	{
		if (!memcmp(&Element, &pTempNode->Data, sizeof(T_ELE)))
		{
			return i;
		}
		pTempNode = pTempNode->pNext;
	}
	return ERROR;
}

//在链表尾部新增节点									
template<class T_ELE> DWORD LinkedList<T_ELE>::Insert(IN T_ELE Element)
{
	NODE<T_ELE>* pNewNode = new NODE<T_ELE>;
	memset(pNewNode, 0, sizeof(NODE<T_ELE>));
	memcpy(&pNewNode->Data, &Element, sizeof(T_ELE));
	// 1. 判断链表是否为空								
	if (m_head == NULL || m_dwLength == 0)
	{
		m_head = pNewNode;
		m_dwLength++;
		return SUCCESS;
	}
	// 2. 如果链表中已经有元素	
	NODE<T_ELE>* pTempNode = m_head;
	for (DWORD i = 0; i < m_dwLength - 1; i++)
	{
		pTempNode = pTempNode->pNext;
	}
	pTempNode->pNext = pNewNode;
	m_dwLength++;
	return SUCCESS;
}

//将节点新增到指定索引的位置						0 1 2 3 4			
template<class T_ELE> DWORD LinkedList<T_ELE>::Insert(IN DWORD dwIndex, IN T_ELE Element)
{
	NODE<T_ELE>* pPreviousNode = NULL;
	NODE<T_ELE>* pCurrentNode = NULL;
	NODE<T_ELE>* pNextNode = NULL;
	NODE<T_ELE>* pNewNode = new NODE<T_ELE>;
	memset(pNewNode, 0, sizeof(NODE<T_ELE>));
	memcpy(&pNewNode->Data, &Element, sizeof(T_ELE));
	//  1. 判断链表是否为空		
	if (m_head == NULL || m_dwLength == 0)
	{
		if (dwIndex == 0)
		{
			m_head = pNewNode;
			m_dwLength++;
			return SUCCESS;
		}
		return BUFFER_IS_EMPTY;
	}
	//  2. 判断索引值是否有效								
	if (dwIndex<0 || dwIndex>m_dwLength)
	{
		return INDEX_IS_ERROR;
	}
	//  3. 如果索引为0			
	if (dwIndex == 0)
	{
		pNewNode->pNext = m_head;
		m_head = pNewNode;
		m_dwLength++;
		return SUCCESS;
	}
	//  4. 如果索引为链表尾		
	if (dwIndex == m_dwLength)
	{
		pCurrentNode = m_head;
		for (DWORD i = 0; i < dwIndex - 1; i++)
		{
			pCurrentNode = pCurrentNode->pNext;
		}
		pCurrentNode->pNext = pNewNode;
		m_dwLength++;
		return SUCCESS;
	}
	//  5. 如果索引为链表中		
	pCurrentNode = m_head;
	for (DWORD i = 0; i < dwIndex - 1; i++)
	{
		pCurrentNode = pCurrentNode->pNext;
	}
	pNextNode = pCurrentNode->pNext;
	pCurrentNode->pNext = pNewNode;
	pNewNode->pNext = pNextNode;
	m_dwLength++;
	return SUCCESS;
}

//根据索引删除节点									
template<class T_ELE> DWORD LinkedList<T_ELE>::Delete(IN DWORD dwIndex)
{
	NODE<T_ELE>* pPreviousNode = NULL;
	NODE<T_ELE>* pCurrentNode = NULL;
	NODE<T_ELE>* pNextNode = NULL;
	//  1. 判断链表是否为空								
	if (m_head == NULL || m_dwLength == 0)
	{
		return BUFFER_IS_EMPTY;
	}
	//  2. 判断索引值是否有效								
	if (dwIndex<0 || dwIndex>m_dwLength)
	{
		return INDEX_IS_ERROR;
	}
	//  3. 如果链表中只有头节点，且要删除头节点								
	if (m_dwLength == 1 && dwIndex == 0)
	{
		delete m_head;
		m_head = NULL;
		m_dwLength--;
		return SUCCESS;
	}
	//  4. 如果要删除头节点								
	if (dwIndex == 0)
	{
		pNextNode = m_head->pNext;
		delete m_head;
		m_head = pNextNode;
		m_dwLength--;
		return SUCCESS;
	}
	//  5. 如果是其他情况	
	pPreviousNode = m_head;
	for (DWORD i = 0; i < dwIndex - 1; i++)
	{
		pPreviousNode = pPreviousNode->pNext;
	}
	pCurrentNode = pPreviousNode->pNext;
	pNextNode = pCurrentNode->pNext;
	pPreviousNode->pNext = pNextNode;
	delete pCurrentNode;
	m_dwLength--;
	return SUCCESS;
}
//获取链表中节点的数量									
template<class T_ELE> DWORD LinkedList<T_ELE>::GetSize()
{
	if (m_head == NULL || m_dwLength == 0)
	{
		return BUFFER_IS_EMPTY;
	}
	else
	{
		return m_dwLength;
	}
}

template<class T_ELE> VOID LinkedList<T_ELE>::Show()
{
	if (m_head == NULL || m_dwLength == 0)
	{
		printf("Linklist is empty.\n");
		return;
	}
	NODE<T_ELE>* pTempNode = m_head;
	for (DWORD i = 0; i < m_dwLength; i++)
	{
		printf("pNode->data: %d\n", pTempNode->Data);
		pTempNode = pTempNode->pNext;
	}
}

VOID TestLink()
{
	LinkedList<int> Link;
	int index = 4;
	int value = 0;
	int element = 10;

	Link.Insert(0, 9);

	Link.Insert(1);
	printf("Link.GetElementIndex(%d): %d\n", element, Link.GetElementIndex(element));

	Link.Insert(1, 8);

	Link.Insert(2);
	printf("Link.GetSize(): %d\n", Link.GetSize());
	Link.Insert(3);
	Link.GetElement(index, value);
	printf("Link.GetElement(%d, %d)\n", index, value);
	Link.Delete(1);
	Link.Delete(0);

	Link.Delete(2);
	printf("Link.GetSize(): %d\n", Link.GetSize());
	Link.Insert(1);
	Link.Show();
	Link.Clear();

	Link.Show();
}

int main()
{
	TestLink();

	return 0;
}
```