---
title: "[CSAPPLAB]Data lab"
date: 2021-04-11T15:07:34+08:00
tags: ["CSAPPLAB"]
ShowToc: true
TocOpen: true
draft: false
---

## 前置知识

csapp第二章

**位运算符**

| 运算符 | 描述                                                         | 实例                                                         |
| :----- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| &      | 按位与操作，按二进制位进行"与"运算。运算规则：`0&0=0;    0&1=0;     1&0=0;      1&1=1;` | (A & B) 将得到 12，即为 0000 1100                            |
| \|     | 按位或运算符，按二进制位进行"或"运算。运算规则：`0|0=0;    0|1=1;    1|0=1;     1|1=1;` | (A \| B) 将得到 61，即为 0011 1101                           |
| ^      | 异或运算符，按二进制位进行"异或"运算。运算规则：`0^0=0;    0^1=1;    1^0=1;   1^1=0;` | (A ^ B) 将得到 49，即为 0011 0001                            |
| ~      | 取反运算符，按二进制位进行"取反"运算。运算规则：`~1=-2;    ~0=-1;` | (~A ) 将得到 -61，即为 1100 0011，一个有符号二进制数的补码形式。 |
| <<     | 二进制左移运算符。将一个运算对象的各二进制位全部左移若干位（左边的二进制位丢弃，右边补0）。 | A << 2 将得到 240，即为 1111 0000                            |
| >>     | 二进制右移运算符。将一个数的各二进制位全部右移若干位，正数左补0，负数左补1，右边丢弃。 | A >> 2 将得到 15，即为 0000 1111                             |

**运算符的优先级**

| 类别       | 运算符                            | 结合性   |
| :--------- | :-------------------------------- | :------- |
| 后缀       | () [] -> . ++ - -                 | 从左到右 |
| 一元       | + - ! ~ ++ - - (type)* & sizeof   | 从右到左 |
| 乘除       | * / %                             | 从左到右 |
| 加减       | + -                               | 从左到右 |
| 移位       | << >>                             | 从左到右 |
| 关系       | < <= > >=                         | 从左到右 |
| 相等       | == !=                             | 从左到右 |
| 位与 AND   | &                                 | 从左到右 |
| 位异或 XOR | ^                                 | 从左到右 |
| 位或 OR    | \|                                | 从左到右 |
| 逻辑与 AND | &&                                | 从左到右 |
| 逻辑或 OR  | \|\|                              | 从左到右 |
| 条件       | ?:                                | 从右到左 |
| 赋值       | = += -= *= /= %=>>= <<= &= ^= \|= | 从右到左 |
| 逗号       | ,                                 | 从左到右 |

## bitXor

利用狄摩根定律即可

```c
int bitXor(int x, int y) {
  return ~(~x&~y)&~(x&y);
}
```

## tmin

返回最小补码,最高位为1，其他位为0

```cpp
int tmin(void) {
//1左移31位右边补0
  return 1<<31;
}
```

## tmax

判断是否是补码最大值

**思路**

四位的最大值`x=0111` 然后x+1=`1000`，`1000` 取非= `0111` =x。

x^x=0，用`!((~(x+1)^x))` 判断这个是否为1即可判断是否为最大值。

例外：当x=-1时，由于-1=1111，故加一个判断条件`!!(x+1)` 

```cpp
int isTmax(int x) {

  return !((~(x+1)^x))&!!(x+1);
}
```

## allOddBits

判断一个数是否所有的奇数位都为1。(位的序号从0到32)

**思路**

`A=1010` A是一个典型的偶数位都是1的数，那只要一个四位的二进制数`X & A = A` 就说明这个二进制符合条件。判断`x & 0xAAAAAAAA == 0xAAAAAAAA` 即可。

```cpp
int allOddBits(int x) {
    int a=0xAA<<8; //0xAA00
    int c=a|0xAA;  //0xAAAA
    int d=c<<16|c; //0xAAAAAAAA
  return !((x&d)^(d)); //a == b 等价于 !(a^b)
}
```

## negate

求一个数的相反数。

```cpp
int negate(int x)

  return ~x+1 ;
}
```

## isAsciiDight

判断一个数字是否在(0x30和0x39)之间

**思路**

找规律00110000和00111001

```c
int isAsciiDigit(int x) {
	int t = x & 0xFFFFFFC0; //判断前26位是否为0
	x = x & 0x3F;
	return !t&!((x+6)&0x40)&(x>>4)&(x>>5)&1;
}
```

## conditional

实现三目运算符x ？y：z

```c
int conditional(int x, int y, int z) {
  // y与z之间肯定用或运算
  // 而在x与y, x与z之间肯定用与运算
  // 如何运算使得
  //    1.x == 0时返回0; 否则返回0xffffffff
  //    2.x == 0时返回0xffffffff; 否则返回0
  // 我们可以利用 0 - 1 = 0xffffffff, 1 - 1 = 0这条特性来返回正确的值 
  int cond1 = (!x + (~1 + 1)) & y;
  int cond2 = (!!x + (~1 + 1)) & z;

  return cond1 | cond2;
}
```

## isLessOrEqual

判断x是否小于等于y

**思路**

分别取出x和y的符号，进行判断。若y大于0，x小于0，则显然为真。若y小于0，x大于0，则显然为假。剩下的异号的情况则用y + (-x) 判断即可。

```c
CODE
int isLessOrEqual(int x, int y) {
	int sgnx = (x>>31)&1; //取出x的符号
	int sgny = (y>>31)&1; //取出y的符号
	return !sgny & sgnx | !(sgnx^sgny) & !((y + (~x+1))&(1<<31));
}
```

## logicalNeg

使用其他的逻辑运算符和位运算符实现 ！运算符

```cpp
int logicalNeg(int x) {
   return ((x | (~x +1)) >> 31) + 1;
}
```

## howManyBits

给一个数字x,求出要表示出x需要的最少的位数。

**思路**

```cpp
int howManyBits(int x) {
  /*
    判断传入的数需要多少个二进制来记录
        正数则查找从左边第一个1开始，一直到最右边那一位，的位数，再加上一个符号位
        负数则查找从左边第一个0开始，一直到最右边那一位，的位数，再加上一个符号位
  */
  int bit16, bit8, bit4, bit2, bit1;
  // 注意,如果x为负，则x >> 31为0xffffffff
  int sign = x >> 31;
  // 因为正数记录从左到右第一个1,负数记录第一个0
  // 所以可以对操作数取反，将负数转为正数，正数不变，便于更好的计算
  x ^= sign;
  // 二分查找，先判断高16位有无存在1,并将范围缩小到高16位或低16位
  // 如果高16位存在0,则bit16 == 16,否则等于0
  bit16 = (!!(x >> 16)) << 4;
  x >>= bit16;
  // 查找剩余16位中的高8位是否存在1
  bit8 = (!!(x >> 8)) << 3;
  x >>= bit8;
  // 查找剩余8位中的高4位是否存在1
  bit4 = (!!(x >> 4)) << 2;
  x >>= bit4;
  // 查找剩余4位中的高2位是否存在1
  bit2 = (!!(x >> 2)) << 1;
  x >>= bit2;
  // 查找剩余2位中的高1位是否存在1
  bit1 = (!!(x >> 1)) << 0;
  x >>= bit1;
    // 注意return中要再加上一个符号位
  return bit16 + bit8 + bit4 + bit2 + bit1 + x + 1;
}
```

## floatScale2

本题给一个无符号数，让你把它看成一个浮点数(都是32位)，让你输出x * 2 的值

思路：

按照浮点数1,8,23的分布将符号，指数，尾数分别取出，并分类讨论即可。

![image-20210531200330668](https://www.kro1lsec.com:442/images/2021/06/07/20210607162052.png)

```c
unsigned floatScale2(unsigned uf) {
	unsigned frac = uf&0x007fffff;
	uf>>=23;
	unsigned exp = uf&0xff;
	uf>>=8;
	if (!exp) {
		frac <<= 1;
		int t = frac>>23;
		if (t) 	{
			frac = frac & 0x007fffff;
			exp++;
		}
	} else if (exp != 0xff)  exp++;
	return (uf<<31) + (exp<<23) + frac;
}
//方法二
unsigned floatScale2(unsigned uf) {
  // 求2乘以传入的参数uf
  // 首先获取其exp部分的值
  int exp = (uf >> 23) & 0xff;
  int sign = uf & (1 << 31);
  // 如果传入的是非规格化的值
  if(exp == 0)
    return sign | uf << 1;
  // 如果传入的是无穷大或NaN
  else if(exp == 0xff)
    // 没办法继续乘了，只能返回参数
    return uf;
  exp++;
  // 如果乘2后的结果超出范围（即溢出），则返回无穷大
  if(exp == 0xff)
    return sign | 0x7f800000; // expr全为0, frac全为1
  return sign | (exp << 23) | (uf & 0x7fffff);
}
```

## floatFloat2Int

给你一个无符号数，并将其看成浮点数(32位)，要求输出(int)x的值

```C
int floatFloat2Int(unsigned uf) {
  int exp = (uf >> 23) & 0xff;
  int sign = (uf >> 31) & 1;
  int frac = uf & 0x7fffff;
  int shiftBits = 0;
  // 0比较特殊，先判断0(正负0都算作0)
  if(!(uf & 0x7fffffff))
    return 0;
  // 判断是否为NaN还是无穷大
  if(exp == 0xff)
    return 0x80000000u;
  // 指数减去偏移量，获取到真正的指数
  exp -= 127;
  // 需要注意的是，原来的frac一旦向左移位，其值就一定会小于1，所以返回0
  if(exp < 0)
    return 0;
  // 获取M，注意exp等于-127和不等于-127的情况是不一样的。当exp != -127时还有一个隐藏的1
  if(exp != -127)
    frac |= (1 << 23);
  // 要移位的位数。注意float的小数点是点在第23位与第22位之间
  shiftBits = 23 - exp;
  // 需要注意一点，如果指数过大，则也返回0x80000000u
  if(shiftBits < 31 - 23)
    return 0x80000000u;
  // 获取真正的结果
  frac >>= shiftBits;
  // 判断符号
  if(sign == 1)
    return ~frac + 1;
  else
    return frac;
}
```

## floatPower2

给一个整数x，要求输出2.0x的值。

```C
unsigned floatPower2(int x) {
    // 判断指数是否上溢或者下溢
    int exp = x + 127;
    if(exp > 0xfe)
        return 0x7f800000;
    else if(exp < 0)
        return 0;
    return exp << 23;
}
```

