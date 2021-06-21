---
title: "[CSAPPLAB]Bomb Lab"
date: 2021-04-15T15:07:34+08:00
tags: ["CSAPPLAB", "天问之路"]
ShowToc: true
TocOpen: true
draft: false
---

![image-20210321154547297](https://www.kro1lsec.com:442/images/2021/05/26/20210526141822.png)

总体感觉考查对汇编的理解和熟练度

#### phase_1

```assembly
400e37:   48 89 c7                mov    %rax,%rdi    //第一个参数  
400e3a:   e8 a1 00 00 00          callq  400ee0 <phase_1>
...
0x0000000000400ee0 <+0>:    sub    $0x8,%rsp 
0x0000000000400ee4 <+4>:    mov    $0x402400,%esi   //第二个参数
0x0000000000400ee9 <+9>:    callq  0x401338 <strings_not_equal>
```

让参数一，二相等即可

Border relations with Canada have never been better.

#### phase_2

利用callq 0x40145c <read_six_numbers>确定字符串格式，

利用0000000000400F1A   add     eax, eax让后边的数翻倍

字符串为“1 2 4 8 16 32”

#### phase_3

尝试分析时，发现jcc指令经常弄混，又回去复习了一遍

```assembly
cmp oprd1,oprd2 + JNBE/JA 无符号
cmp oprd1,oprd2 + JNLE/JG 有符号
oprd1>oprd2时跳转
JZ/JE 相等则跳转
```

代码中`0x402470+8*第一参数`打印内存信息，可以看到相应的跳转地址，已经对应的eax值应该是多少，也就是第二数字的应该的取值。打印内存信息的命令行为jmp QWORD PTR [rax*8+0x402470]

    0 207  ===》x/x (int *)0x402470 打印得到地址400f7c
    1 311  ===》x/x (int *)0x402478 打印得到地址400fb9
    2 707  ===》x/x (int *)0x402480 打印得到地址400f83
    3 256  ===》x/x (int *)0x402488 打印得到地址400f8a
    4 389  ===》x/x (int *)0x402490 打印得到地址400f91
    5 206  ===》x/x (int *)0x402498 打印得到地址400f98
    6 682  ===》x/x (int *)0x4024a0 打印得到地址400f9f
    7 327  ===》x/x (int *)0x4024a8 打印得到地址400fa6

#### phase_4

```assembly
; __unwind {
sub     rsp, 18h
lea     rcx, [rsp+18h+var_C]
lea     rdx, [rsp+18h+var_10]
mov     esi, offset aDD ; "%d %d"
mov     eax, 0
call    ___isoc99_sscanf
cmp     eax, 2                 //需要输入2个数字
jnz     short loc_401035       
//JNZ/JNE   不为0跳转,不相等跳转 if (i != j);if (i != 0);
```

```assembly
cmp     [rsp+18h+var_10], 0Eh
jbe     short loc_40103A
//JBE/JNA if (i <= j);可知第一个数字<=0xe
。。。
cmp     [rsp+18h+var_C], 0  //第一个参数为0
jz      short loc_40105D
```

分析`<fun4>`函数可知第二个参数的为0

密码为0 0

#### phase_5

两种不同的汇编看得还不习惯，贴了ida中的写法

```assembly
401089: eb 47                   jmp    4010d2 <phase_5+0x70>
000000000040108B                 movzx   ecx, byte ptr [rbx+rax]
40108b: 0f b6 0c 03             movzbl (%rbx,%rax,1),%ecx //将输入字符串的第一个字符给ecx
40108f: 88 0c 24                mov    %cl,(%rsp)
401092: 48 8b 14 24             mov    (%rsp),%rdx
401096: 83 e2 0f                and    $0xf,%edx     //取字符的最低4位到edx，大小从0~15
0000000000401099                 movzx   edx, ds:array_3449[rdx]
401099: 0f b6 92 b0 24 40 00    movzbl 0x4024b0(%rdx),%edx   //根据rdx中的数字，从0x4024b0(%rdx)中取字符
//maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?

4010a0: 88 54 04 10             mov    %dl,0x10(%rsp,%rax,1) //存储取出的字符
4010a4: 48 83 c0 01             add    $0x1,%rax  //处理次数+1
4010a8: 48 83 f8 06             cmp    $0x6,%rax  //判断处理次数是否到6次
4010ac: 75 dd                   jne    40108b <phase_5+0x29>
4010ae: c6 44 24 16 00          movb   $0x0,0x16(%rsp)   //为新创建的字符串添加结尾符“/0”
4010b3: be 5e 24 40 00          mov    $0x40245e,%esi  // flyers
4010b8: 48 8d 7c 24 10          lea    0x10(%rsp),%rdi //比较新创建的字符串与flyers是否相等
4010bd: e8 76 02 00 00          callq  401338 <strings_not_equal>
4010c2: 85 c0                   test   %eax,%eax
4010c4: 74 13                   je     4010d9 <phase_5+0x77>
4010c6: e8 6f 03 00 00          callq  40143a <explode_bomb>
00000000004010CB                 align 10h
4010cb: 0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
4010d0: eb 07                   jmp    4010d9 <phase_5+0x77>
4010d2: b8 00 00 00 00          mov    $0x0,%eax
4010d7: eb b2                   jmp    40108b <phase_5+0x29>
```

```sh
pwndbg> p/s (char*)0x4024b0
$1 = 0x4024b0 <array> "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```

f（9），l（15），y（14），e（5），r（6），s（7）

答案为：

```
ionefg（0x49，0x4f,0x4E,0x45,0x46,0x47）
```

#### phase_6

callq  40145c <read_six_numbers>

我们需要输入的6位数字，且都小于6

while循环

```assembly
 401128:    41 83 c4 01             add    $0x1,%r12d     //r12d计数
 40112c:    41 83 fc 06             cmp    $0x6,%r12d     //计数小于6
401130: 74 21                   je     401153 <phase_6+0x5f>

401132: 44 89 e3                mov    %r12d,%ebx
401135: 48 63 c3                movslq %ebx,%rax

401138: 8b 04 84                mov    (%rsp,%rax,4),%eax
40113b: 39 45 00                cmp    %eax,0x0(%rbp) //==>判断每个数字都不等于 0
40113e: 75 05                   jne    401145 <phase_6+0x51>
401140: e8 f5 02 00 00          callq  40143a <explode_bomb>
401145: 83 c3 01                add    $0x1,%ebx
401148: 83 fb 05                cmp    $0x5,%ebx
40114b: 7e e8                   jle     401135 <phase_6+0x41>
```

![image-20210321150502162](https://www.kro1lsec.com:442/images/2021/05/26/20210526141826.png)

要用7减完的序列做索引从链表取数，比较大小。从大到小排序。

3 4 5 6 1 2

用7减完

4 3 2 1 6 5

#### secret_phase

递归调用人逆傻了，参考丛丛的脚本爆破

大致思路是定位到if语句中参数的地址6030F0，再照着原函数的逻辑爆破。

```python
data = '''0x6030f0,0x0000000000000024,0x0000000000603110
0x603100,0x0000000000603130,0x0000000000000000
0x603110,0x0000000000000008,0x0000000000603190
0x603120,0x0000000000603150,0x0000000000000000
0x603130,0x0000000000000032,0x0000000000603170
0x603140,0x00000000006031b0,0x0000000000000000
0x603150,0x0000000000000016,0x0000000000603270
0x603160,0x0000000000603230,0x0000000000000000
0x603170,0x000000000000002d,0x00000000006031d0
0x603180,0x0000000000603290,0x0000000000000000
0x603190,0x0000000000000006,0x00000000006031f0
0x6031a0,0x0000000000603250,0x0000000000000000
0x6031b0,0x000000000000006b,0x0000000000603210
0x6031c0,0x00000000006032b0,0x0000000000000000
0x6031d0,0x0000000000000028,0x0000000000000000
0x6031e0,0x0000000000000000,0x0000000000000000
0x6031f0,0x0000000000000001,0x0000000000000000
0x603200,0x0000000000000000,0x0000000000000000
0x603210,0x0000000000000063,0x0000000000000000
0x603220,0x0000000000000000,0x0000000000000000
0x603230,0x0000000000000023,0x0000000000000000
0x603240,0x0000000000000000,0x0000000000000000
0x603250,0x0000000000000007,0x0000000000000000
0x603260,0x0000000000000000,0x0000000000000000
0x603270,0x0000000000000014,0x0000000000000000
0x603280,0x0000000000000000,0x0000000000000000
0x603290,0x000000000000002f,0x0000000000000000
0x6032a0,0x0000000000000000,0x0000000000000000
0x6032b0,0x00000000000003e9,0x0000000000000000
0x6032c0,0x0000000000000000,0x0000000000000000'''

data_dict = {}
fin_dict = {}

def data_collect():
	data_list = data.split("\n")
	last_try = 0
	for i in data_list:
		#print i
		x = i.split(",")
		x = [int(i,16) for i in x]
		if x[1]<0x600000 and x[1]>0:
			data_dict[x[0]] = [x[1],x[2],0]
			last_try = x[0]
		else:
			data_dict[last_try][2] = x[1]
	for k in data_dict.values():
		if k[1] == 0:
			if k[2] == 0:
				fin_dict[k[0]] = [0,0]
			else:
				fin_dict[k[0]] = [0,data_dict[k[2]][0]]
		else:
			if k[2] == 0:
				fin_dict[k[0]] = [data_dict[k[1]][0],0]
			else:
				fin_dict[k[0]] = [data_dict[k[1]][0],data_dict[k[2]][0]]
		
data_collect()


def func(num,i):
	if num == 0:
		return -1
	if i == num:
		result = 0
		return result
	if i>num:
		next_num = fin_dict[num][1]
		result = func(next_num,i)
		result = result+result+1
		return result
	if i<num:
		next_num = fin_dict[num][0]
		result = func(next_num,i)
		result *= 2
		return result

for i in range(1000):
	if func(0x24,i) == 2:
		print i
```
