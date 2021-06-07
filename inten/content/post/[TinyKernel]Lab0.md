---
title: "[TinyKernel]Lab0"
date: 2021-06-06T15:07:34+08:00
draft: false
---


#### gcc

```
#安装
sudo apt-get install build-essential
```

```
#编译
$ gcc -Wall hello.c -o hello
```

该命令将文件‘hello.c’中的代码编译为机器码并存储在可执行文件 ‘hello’中。机器码的文件名是通过 -o 选项指定的。该选项通常作为命令行中的最后一个参数。如果被省略，输出文件默认为 ‘a.out’。

选项 -Wall 开启编译器几乎所有常用的警告，默认情况下GCC 不会产生任何警告信息。

##### AT&T汇编基本语法

Ucore中用到的是AT&T格式的汇编，与Intel格式的汇编有一些不同。二者语法上主要有以下几个不同：

```assembly
    * 寄存器命名原则
        AT&T: %eax                      Intel: eax
    * 源/目的操作数顺序 
        AT&T: movl %eax, %ebx           Intel: mov ebx, eax
    * 常数/立即数的格式　
        AT&T: movl $_value, %ebx        Intel: mov eax, _value
      把value的地址放入eax寄存器
        AT&T: movl $0xd00d, %ebx        Intel: mov ebx, 0xd00d
    * 操作数长度标识 
        AT&T: movw %ax, %bx             Intel: mov bx, ax
    * 寻址方式 
        AT&T:   immed32(basepointer, indexpointer, indexscale)
        Intel:  [basepointer + indexpointer × indexscale + imm32)
```

如果操作系统工作于保护模式下，用的是32位线性地址，所以在计算地址时不用考虑segment:offset的问题。上式中的地址应为：

```
    imm32 + basepointer + indexpointer × indexscale
```

下面是一些例子：

```assembly
* 直接寻址 
AT&T:  foo                         Intel: [foo]
boo是一个全局变量。注意加上$是表示地址引用，不加是表示值引用。对于局部变量，可以通过堆栈指针引用。

* 寄存器间接寻址 
AT&T: (%eax)                        Intel: [eax]

* 变址寻址 
AT&T: _variable(%eax)               Intel: [eax + _variable]
AT&T: _array( ,%eax, 4)             Intel: [eax × 4 + _array]
AT&T: _array(%ebx, %eax,8)          Intel: [ebx + eax × 8 + _array]
```

##### GCC基本内联汇编

```sh
#例子
asm("statements");
asm("nop"); asm("cli");
#"asm" 和"__asm__"的含义是完全一样。如果有多行汇编，则每一行都要加上 "\n\t"，即换行符和tab 符
asm( "pushl %eax\n\t"
"movl $0,%eax\n\t"
"popl %eax"
);
```

##### GCC扩展内联汇编

```assembly
#基本格式
asm [volatile] ( Assembler Template
   : Output Operands
   [ : Input Operands
   [ : Clobbers ] ])
#例子
#define read_cr0() ({ \
unsigned int __dummy; \
__asm__( \
    "movl %%cr0,%0\n\t" \
#“＝r”表示相应的目标操作数（指令部分的%0）可以使用任何一个通用寄存器，并且变量__dummy 存放在这个寄存器中，
    :"=r" (__dummy)); \
__dummy; \
})
```


“＝m”就表示相应的目标操作数是存放在内存单元__dummy中。表示约束条件的字母很多，下表给出几个主要的约束字母及其含义：

| 字母       | 含义                                             |
| ---------- | ------------------------------------------------ |
| m, v, o    | 内存单元                                         |
| R          | 任何通用寄存器                                   |
| Q          | 寄存器eax, ebx, ecx,edx之一                      |
| I, h       | 直接操作数                                       |
| E, F       | 浮点数                                           |
| G          | 任意                                             |
| a, b, c, d | 寄存器eax/ax/al, ebx/bx/bl, ecx/cx/cl或edx/dx/dl |
| S, D       | 寄存器esi或edi                                   |
| I          | 常数（0～31）                                    |



```assembly
#例2
int count=1;
int value=1;
int buf[10];
void main()
{
    asm(
        "cld \n\t"
        "rep \n\t"
        "stosl"
    :
    : "c" (count), "a" (value) , "D" (buf)
    );
}
#例2得到的主要汇编代码
movl count,%ecx
movl value,%eax
movl buf,%edi
#APP
cld
rep
stosl
#NO_APP
```
几点说明：

- [1] 使用q指示编译器从eax, ebx, ecx, edx分配寄存器。 使用r指示编译器从eax, ebx, ecx, edx, esi, edi分配寄存器。
- [2] 不必把编译器分配的寄存器放入改变的寄存器列表，因为寄存器已经记住了它们。
- [3] "="是标示输出寄存器，必须这样用。
- [4] 数字%n的用法：数字表示的寄存器是按照出现和从左到右的顺序映射到用"r"或"q"请求的寄存器．如果要重用"r"或"q"请求的寄存器的话，就可以使用它们。
- [5] 如果强制使用固定的寄存器的话，如不用%1，而用ebx，则：

```c
#让gcc自己选择合适的寄存器
asm("leal (%1,%1,4),%0"
    : "=r" (x)
    : "0" (x)
);
asm("leal (%%ebx,%%ebx,4),%0"
    : "=r" (x)
    : "0" (x) 
);
```

> 注意要使用两个%,因为一个%的语法已经被%n用掉了。

#### make和Makefile

GNU make(简称make)是一种代码维护工具，在大中型项目中，它将根据程序各个模块的更新情况，自动的维护和生成目标代码。

make命令执行时，需要一个 makefile （或Makefile）文件，以告诉make命令需要怎么样的去编译和链接程序。

#####  makefile的规则

```
target ... : prerequisites ...
    command
    ...
    ...
```

target：目标文件，可以是object file，也可以是执行文件。还可以是一个标签（label）。

prerequisites：要生成那个target所需要的文件或是目标。

command：make需要执行的命令（任意的shell命令）。

这是一个文件的依赖关系，target这一个或多个的目标文件依赖于prerequisites中的文件，其生成规则定义在 command中。如果prerequisites中有一个以上的文件比target文件要新，那么command所定义的命令就会被执行。这就是makefile的规则。

##### GDB

https://inten.kro1lsec.com/post/%E5%B7%A5%E5%85%B7gdb/

#### Intel 80386

##### 运行模式

实模式、保护模式、SMM模式和虚拟8086模式

##### 内存架构

地址是访问内存空间的索引。内存地址有两个：一个是CPU通过总线访问物理内存用到的物理地址，一个是我们编写的应用程序所用到的逻辑地址（也有人称为虚拟地址）。

三种地址的关系如下：

- 分段机制启动、分页机制未启动：逻辑地址--->***段机制处理\***--->线性地址=物理地址
- 分段机制和分页机制都启动：逻辑地址--->***段机制处理\***--->线性地址--->***页机制处理\***--->物理地址

##### 寄存器

通用寄存器： eax, ebx, ecx, edx, esi, edi, esp, ebp,

段寄存器：cs(code segment), ds(data segment), es(extra segment), ss(stack segment), fs, gs,

指令寄存器, eip，保存的是下一条要执行指令的内存地址，在分段地址转换中表示指令的段内偏移地址，

标识寄存器， efalgs,

另外是cr0等不能被访问的控制寄存器，