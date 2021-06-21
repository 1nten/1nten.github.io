---
title: "[uCore]Lab1"
date: "2021-06-20"
tags: ["uCore", "天问之路"]
ShowToc: true
TocOpen: true
draft: false
---

## 练习1
[练习指南](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_2_1_1_ex1.html)

**操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)**

- 设置了项目名称，EMPTY，SPACE，SLASH等，变量V设置是为了进行打印.
- 设置`GCCPRETIX`和`QEMU`，检查是否支持32位的elf文件。
- 配置 USELLVM
- include tools/function.mk,包含一个已经已经封装好函数的文件，下文大量调用。
- 为了生成ucore.img，首先需要生成kernel、bootblock
- 为了生成kernel，首先需要 kernel.ld init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o trapentry.o vectors.o pmm.o  printfmt.o string.o
- 为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign
- sign的作用是先检测bootblock的大小要小于500，然后在最后增加`0x55, 0xaa`的标识位， 然后复制到对应目录。
- ucore.img使用dd指令将对应的文件合并
- 最后配置target

all: 对应直接使用make的时候，就是编译出`ucore.img`，

lab1-mon使用qemu的monitor模式保存log记录，并运行一个gdb进行调试

debug-mon: 使用qemu的monitor和gdb配合调试

qemu: 使用qemu混合模式

qemu-mon:使用qemu monitor模式

qemu-nox:使用qemu 命令行模式

**一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?**

从/tool/sign.c的代码来看，一个磁盘主引导扇区只有512字节。且
第510个（倒数第二个）字节是0x55，
第511个（倒数第一个）字节是0xAA。

## [练习2](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_2_1_2_ex2.html)

使用qemu执行并调试lab1中的软件。

[单步调试和查看BIOS代码](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_5_appendix.html)

修改`tools/gdbinit`为

```sh
file obj/bootblock.o
set architecture i8086
target remote :1234
b* 0x7c00

#强制反汇编当前的指令
define hook-stop
x/i $eip
end
continue
```

- gdb脚本中的`define hook-stop ... end`，可以在每次gdb断下时自动执行内部的指令。上面gdb脚本中的`define .. end`告诉gdb在每次断下时输出下一条指令，方便调试。

  如果想单步调试BIOS，则可以删除`continue`。这样gdb就会在`0xfff0`处断下，然后我们就可以自由的单步跟踪。

- 之后执行`make debug`命令，即可自动打开qemu与已经连接完成的gdb。

- 坑！（排查了好久）：远程连接qemu时，**不要使用pwndbg插件**。因为使用该插件会导致连接到qemu后无法操作gdb。

  > `peda`插件可以正常使用。[谢kiprey师傅!](https://kiprey.github.io/)

[切换GitHub插件](https://blog.csdn.net/weixin_43092232/article/details/105648769)  vim ~/.gdbinit

```sh
source ~/peda/peda.py                                 #注释掉pwndbg即可                    
#source ~/pwndbg/gdbinit.py
#source ~/Pwngdb/pwngdb.py
#source ~/Pwngdb/angelheap/gdbinit.py

define hook-run
python
import angelheap
angelheap.init_angelheap()
end
end

x/5i (($cs << 4)+  $eip)
```

注意：BIOS的前几条指令在GDB中都需要手动加上段寄存器的值，否则会显示错误，因为`cs`寄存器**初始时非零**；同时gdb默认只输出`$ip` 所指向地址的指针，而不是`cs:ip`。

```sh
#错误
(gdb) si
=> 0xe05b:      add    %al,(%eax)
0x0000e05b in ?? ()

#正确
(gdb)  x/5i (($cs << 4)+  $eip)
   0xfe05b:     cmpw   $0xffc8,%cs:(%esi)
   0xfe060:     bound  %eax,(%eax)
   0xfe062:     jne    0xd241d0b2
   0xfe068:     mov    %edx,%ss
   0xfe06a:     mov    $0x7000,%sp

```

## [练习3](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_2_1_3_ex3.html)

分析bootloader进入保护模式的过程.

- 为何开启A20，以及如何开启A20?[关于A20 Gate](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_appendix_a20.html)

  为何：为了向下兼容
  
  如何：打开A20 Gate的步骤如下（参考lab1/boot/bootasm.S）：	

1. 等待8042 Input buffer为空；
2. 发送Write 8042 Output Port （P2）命令到8042 Input buffer；
3. 等待8042 Input buffer为空；
4. 将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer；

```assembly
    # 启用A20。通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，可以访问4G的内存空间。
    # 为了向后兼容最早的PC，物理地址线20被绑在低位，所以默认情况下，高于1MB的地址会环绕到0。这段代码取消了这一点。
seta20.1:
    inb $0x64, %al                   # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al					 # 读取到2则表明缓冲区中没有数据
    jnz seta20.1

    movb $0xd1, %al                  # 0xd1 -> port(端口) 0x64
    outb %al, $0x64                  # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                   # Wait for not busy(8042 input buffer(缓冲区) empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al     # 打开A20
    outb %al, $0x60     # 
```

- 如何初始化GDT表

  - 设置GDT中的第一项描述符为空。
  - 设置GDT中的第二项描述符为代码段使用，其属性为可读写可执行。
  - 设置GDT中的第三项描述符为数据段使用，其属性为可读写。

  ```
  #初始化GDT表：一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可:lgdt gdtdesc
  .p2align 2                                          # force 4 byte alignment
  gdt:
      SEG_NULLASM                                     # null seg
      SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
      SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel
  
  gdtdesc:
      .word 0x17                                      # sizeof(gdt) - 1
      .long gdt                                       # address gdt
  ```

  进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式

  ```c
  	    movl %cr0, %eax
  	    orl $CR0_PE_ON, %eax
  	    movl %eax, %cr0
  ```

  通过长跳转更新cs的基地址

  ```c
  	 ljmp $PROT_MODE_CSEG, $protcseg
  	.code32
  	protcseg:
  ```

  设置段寄存器，并建立堆栈

  ```c
  	    movw $PROT_MODE_DSEG, %ax
  	    movw %ax, %ds
  	    movw %ax, %es
  	    movw %ax, %fs
  	    movw %ax, %gs
  	    movw %ax, %ss
  	    movl $0x0, %ebp
  	    movl $start, %esp
  ```

  转到保护模式完成，进入boot主方法

  ```
  	    call bootmain
  ```

## [练习4](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_2_1_4_ex4.html)

[硬盘访问概述](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_3_2_3_dist_accessing.html)

[ELF执行文件格式概述](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_3_2_4_elf.html)

**bootloader加载ELF格式的OS的过程：**
`readsect`从设备的第secno扇区读取数据到dst位置，实现磁盘访问，然后包装为`readseg`函数，实现内存读取。在bootmain函数中首先读取elf头进来，并获取到程序头表，然后吧elf中的每个段依次读取进来。

**bootloader如何读取硬盘扇区的？**

```c
/* waitdisk - wait for disk ready */
static void waitdisk(void) {
  while ((inb(0x1F7) & 0xC0) != 0x40)
    /* do nothing */;
}

static void readsect(void *dst, uint32_t secno) {
	    waitdisk();
	
	    outb(0x1F2, 1);                         // 设置读取扇区的数目为1
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	        // 上面四条指令联合制定了扇区号
	        // 在这4个字节线联合构成的32位参数中
	        //   29-31位强制设为1
	        //   28位(=0)表示访问"Disk 0"
	        //   0-27位是28位的偏移量
	    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
	
	    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
	                                            // 幻数4因为这里以DW为单位
	}
```

readseg简单包装了readsect，可以从设备读取任意长度的内容。

```c
	static void
	readseg(uintptr_t va, uint32_t count, uint32_t offset) {
	    uintptr_t end_va = va + count;
	
	    va -= offset % SECTSIZE;
	
	    uint32_t secno = (offset / SECTSIZE) + 1; 
	    // 加1因为0扇区被引导占用
	    // ELF文件从1扇区开始
	
	    for (; va < end_va; va += SECTSIZE, secno ++) {
	        readsect((void *)va, secno);
	    }
	}
```

**bootloader是如何加载ELF格式的OS？**

```c
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw( 0x8A00, 0x8E00);
	    while (1);
	}
```

## [练习5]() 

```c
/* LAB1 YOUR CODE : STEP 1 
* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
* (2) call read_eip() to get the value of eip. the type is (uint32_t);
* (3) from 0 .. STACKFRAME_DEPTH
*    (3.1) printf value of ebp, eip
*    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
*    (3.3) cprintf("\n");
*    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
*    (3.5) popup a calling stackframe
*           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
*                   the calling funciton's ebp = ss:[ebp]

+|  栈底方向     | 高位地址
 |    ...       |
 |    ...       |
 |  参数3       |
 |  参数2       |
 |  参数1       |
 |  返回地址     |
 |  上一层[ebp]  | <-------- [ebp]
 |  局部变量     |  低位地址
*/
void print_stackframe(void){
    //读取当前栈帧的ebp和eip
    uint32_t ebp = read_eip();
    uint32_t eip = read_eip();
    //(3) from 0 .. STACKFRAME_DEPTH
    for(uint32_t i = 0 ; ebp != 0 && i <STACKFRAME_DEPTH;i++)
    {
        //读取
        cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
        //args[0]为参数1
        uint32_t* args = (uint32_t*)ebp +2;
        //the calling funciton's return addr eip  = ss:[ebp+4]
        for(uint32_t j = 0; j<4;j++)
            cprintf("0x%0x8x ", args[j]);
        cprintf("\n");
        //eip指向异常指令的下一条指令，所以要减一
        print_debuginfo(eip-1);
        //将ebp 和eip设置为上一个栈帧的ebp和eip
        // 注意要先设置eip后设置ebp，否则当ebp被修改后，eip就无法找到正确的位置
        eip = *((uint32_t*)ebp+ 1);
        ebp = *(uint32_t*)ebp;
            }
}
```

## [练习6]()

- **中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？**其中哪几位代表中断处理代码的入口？

  一个表项的结构如下

  中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移， 
  两者联合便是中断处理程序的入口地址。

  ```c
  /* Gate descriptors for interrupts and traps */
  struct gatedesc {
      unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
      unsigned gd_ss : 16;            // segment selector
      unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
      unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
      unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
      unsigned gd_s : 1;                // must be 0 (system)
      unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
      unsigned gd_p : 1;                // Present
      unsigned gd_off_31_16 : 16;        // high bits of offset in segment
  };
  ```

该表项的大小为`16+16+5+3+4+1+2+1+16 == 8*8`bit，即**8字节**。

根据IDT表项的结构，我们可以得知，IDT表项的第二个成员`gd_ss`为段选择子，第一个成员`gd_off_15_0`和最后一个成员`gd_off_31_16`共同组成一个段内偏移地址。根据段选择子和段内偏移地址就可以得出中断处理程序的地址。
编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init.

- 具体实现如下，详细信息以注释的形式写入代码中。

  ```c
  void idt_init(void) {
    // __vectors定义于vector.s中
    extern uintptr_t __vectors[];
    int i;
    for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++)
        // 目标idt项为idt[i]
        // 该idt项为内核代码，所以使用GD_KTEXT段选择子
        // 中断处理程序的入口地址存放于__vectors[i]
        // 特权级为DPL_KERNEL
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    // 设置从用户态转为内核态的中断的特权级为DPL_USER
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
    // 加载该IDT
    lidt(&idt_pd);
  }
  ```

编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

```c
trap_dispatch(struct trapframe *tf) {
    char c;

    switch (tf->tf_trapno) {
    case IRQ_OFFSET + IRQ_TIMER:
        /* LAB1 YOUR CODE : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
        ticks ++;// 全局变量ticks定义于kern/driver/clock.c
        if (ticks % TICK_NUM == 0) {
            print_ticks();
        }
        break;
```

## 附录

### 环境配置

- 执行`sudo apt-get install qemu-system`，安装qemu程序，为执行uCore做准备
- 下载该[github](https://github.com/chyyuu/os_kernel_lab)上的**master**分支（注意默认分支不是master分支）的uCore代码，解压使用。

### BIOS中断、DOS中断、Linux中断的区别

(操作系统真相还原0.20)

BIOS 和 DOS 都是存在于实模式下的程序，由它们建立的中断调用都是建立在中断向量表（ Interrupt Vector Table, IVT ）中的。它们都是通过软中断指令 int 中断号来调用的。

BIOS 中断调用的主要功能是提供了硬件访问的方法，该方法使对硬件的操作变得简单易行。不通过 BIOS 调用也是可以访问硬件。操作硬件是通过 in/out 指令来读写外设的端口， BIOS中断程序处理是用来操作硬件的，故该处理程序中一定到处都是 in/out 指令。

DOS 其中断向量号和BIOS的不能冲突。

**Linux 内核是在进入保护模式后才建立中断例程的，在保护模式下，中断向量表已经不存在了，取而代之的是中断描述符表**（Interrupt Descriptor Table，IDT）。Linux 的系统调用和DOS中断调用类似，不过Linux是通过`int 0x80`指令进入一个中断程序后再根据eax寄存器的值来调用不同的子功能函数的

### 操作系统如何识别文件系统

**依靠分区中超级块记录的魔数**

各分区都有超级块，一般位于本分区的第2个扇区。超级块里面记录了此分区的信息，其中就有文件系统的魔数，一种文件系统对应一个魔数，通过比较即可得知文件系统类型。

### 单步调试和查看BIOS代码

如果你是想看BIOS的汇编，可试试如下方法： 练习2可以单步跟踪，方法如下：

1 修改 lab1/tools/gdbinit,

```
set architecture i8086
target remote :1234
```

2 在 lab1目录下，执行

```
make debug
```

这时gdb停在BIOS的第一条指令处：

```
0xffff0: ljmp $0xf000,$0xe05b
```

3 在看到gdb的调试界面(gdb)后，执行如下命令，就可以看到BIOS在执行了

```
si
si
...
```

4 此时的`CS=0xf000, EIP=0xfff0`，如果想看BIOS的代码

```
x /2i 0xffff0
```

应该可以看到

```
0xffff0: ljmp $0xf000,$0xe05b
0xffff5: xor %dh,0x322f
```

进一步可以执行

```
x /10i 0xfe05b
```

可以看到后续的BIOS代码。

### CPU的实模式

- CPU大体上可分为控制单元、运算单元、存储单元

  - 控制单元是CPU的控制中心，大致由指令寄存器(IR, Instruction Register)、指令译码器(ID, Instruction Decoder)和操作控制器(OC, Operation Controller)组成。以下是一般的指令格式
    ![img](https://www.kro1lsec.com:442/images/2021/06/21/20210621151233.png)
  - 运算单元根据控制单元的信号，进行运算。
  - 存储单元CPU内部的L1、L2缓存及寄存器。这部分缓存采用SRAM存储器。SRAM不需要刷新电路即可保存内部存储的数据，但因为体积较大，集成度较低。

- CPU中的寄存器分为两大类：程序可见寄存器（例如通用寄存器、段寄存器）和程序不可见寄存器（例如中断描述符寄存器IDTR）。
  ![img](https://www.kro1lsec.com:442/images/2021/06/21/20210621151413.png)

- 实模式的主要特性是：**程序用到的地址都是真实的物理地址**。同时，实模式下的地址寻址空间只有1MB(20bit)

  > 从intel 80386开始的CPU，只要进入实模式，地址寻址空间就限制在1MB。

- 实模式下的地址计算方式为**16\*段寄存器值+段内偏移地址**，其CPU寻址方式为

  - 寄存器寻址
  - 立即数寻址
  - 内存寻址
    - 直接寻址。例如`mov ax, [0x1234]`
    - 基址寻址
    - 变址寻址
    - 基址变址寻址

#### a. CPU实模式下的1MB内存

- CPU初始状态为16位实模式，在实模式下只能访问1MB(20bits)内存。而硬件工程师将1MB的内存空间分成多个部分。
  ![img](https://www.kro1lsec.com:442/images/2021/06/21/20210621151424.png)

- 其中地址`0-0x9ffff`的640KB内存是DRAM，即插在主板上的内存条。
  顶部`0xf0000-0xfffff`的64KB内存是ROM，存放BIOS代码。

  > BIOS检测并初始化硬件，同时建立中断向量表，并保证能运行一些基本硬件的IO操作

- CPU中，插在主板上的物理内存并不是眼中“全部的内存”。地址总线宽度决定可以访问的内存空间大小。
  并不是只有插在主板上的内存条需要通过地址总线访问，还有一些外设同样是需要通过地址总线来访问。
  故地址总线上会提前预留出来一些地址空间给这些外设，其余的可用地址再指向DRAM。

### CPU的分段机制（重要）

#### a. 内存访问为什么要分段

- 以前程序都是直接访问物理内存，编译出的两个程序如果内存冲突，则无法同时运行。
- CPU采用“段基址+段内偏移地址”的方式来访问任意内存。好处是程序可以重定位，可以执行多个程序。
- 段基址不需要得是65536的倍数。
- 加载用户程序时，只要将整个段的内容复制到新的位置，再将段基址寄存器中的地址改成该地址，程序便可准确无误地运行，因为程序中用的是段内偏移地址。
- 改变段基址，通过在内存中一个段来回挪位置的方式可以访问到任意内存位置。程序分段可以将大内存分成可以访问的小段，访问到所有内存。
- 通过分段，在早期CPU实模式16位寄存器的情况下，计算**段基址 << 4 + 段内偏移地址**，即可访问到20位地址空间。

> 代码中的分段与CPU的分段不同。编译器负责挑选出数据具备的属性，从而根据属性将程序片段分类，比如划分出了只读属性的代码段和可写属性的数据段。编译器并没有让段具备某种属性，对于代码段，编译器只是将代码归类到一起，并没有为代码段添加额外的信息。

- **在实模式下，段基址直接写在段寄存器中；而在保护模式下，段寄存器中的不再是段基址，而是段选择子。**
- 分段机涉及4个关键内容：逻辑地址、段描述符（描述段的属性）、段描述符表（包含多个段描述符的“数组”）、段选择子（段寄存器，用于定位段描述符表中表项的索引）。只有在**保护模式**下才能使用分段存储管理机制。

#### b. 将逻辑地址（虚拟地址）转换为物理地址的两步操作

- 分段地址转换：CPU把逻辑地址（由段选择子selector和段偏移offset组成）中的段选择子的内容作为段描述符表的索引，找到表中对应的段描述符，然后把段描述符中保存的段基址加上段偏移值，形成线性地址（Linear Address）。
- 分页地址转换，这一步中把线性地址转换为物理地址。
  ![img](https://www.kro1lsec.com:442/images/2021/06/21/20210621151428.png)

#### c. 段描述符

- 在分段存储管理机制的保护模式下，每个段由如下三个参数进行定义：段基地址(Base Address)、段界限(Limit)和段属性(Attributes)

  - 段基地址：规定线性地址空间中段的起始地址。任何一个段都可以从32位线性地址空间中的任何一个字节开始，不用像实模式下规定边界必须被16整除。
  - 段界限：规定段的大小。可以以字节为单位或以4K字节为单位。
  - 段属性：确定段的各种性质。
    - 段属性中的粒度位（Granularity），用符号G标记。G=0表示段界限以字节位位单位，20位的界限可表示的范围是1字节至1M字节，增量为1字节；G=1表示段界限以4K字节为单位，于是20位的界限可表示的范围是4K字节至4G字节，增量为4K字节。
    - 类型（TYPE）：用于区别不同类型的描述符。可表示所描述的段是代码段还是数据段，所描述的段是否可读/写/执行，段的扩展方向等。其4bit从左到右分别是
      - 执行位：置1时表示可执行，置0时表示不可执行；
      - 一致位：置1时表示一致码段，置0时表示非一致码段；
      - 读写位：置1时表示可读可写，置0时表示只读；
      - 访问位：置1时表示已访问，置0时表示未访问。
    - 描述符特权级（Descriptor Privilege Level）（DPL）：用来实现保护机制。
    - 段存在位（Segment-Present bit）：如果这一位为0，则此描述符为非法的，不能被用来实现地址转换。如果一个非法描述符被加载进一个段寄存器，处理器会立即产生异常。操作系统可以任意的使用被标识为可用（AVAILABLE）的位。
    - 已访问位（Accessed bit）：当处理器访问该段（当一个指向该段描述符的选择子被加载进一个段寄存器）时，将自动设置访问位。操作系统可清除该位。

- 段描述符的格式
  ![img](https://www.kro1lsec.com:442/images/2021/06/21/20210621151431.png)

- 段描述符的结构

  ```c
  /* segment descriptors */
  struct segdesc {
      unsigned sd_lim_15_0 : 16;        // low bits of segment limit
      unsigned sd_base_15_0 : 16;        // low bits of segment base address
      unsigned sd_base_23_16 : 8;        // middle bits of segment base address
      unsigned sd_type : 4;            // segment type (see STS_ constants)
      unsigned sd_s : 1;                // 0 = system, 1 = application
      unsigned sd_dpl : 2;            // descriptor Privilege Level
      unsigned sd_p : 1;                // present
      unsigned sd_lim_19_16 : 4;        // high bits of segment limit
      unsigned sd_avl : 1;            // unused (available for software use)
      unsigned sd_rsv1 : 1;            // reserved
      unsigned sd_db : 1;                // 0 = 16-bit segment, 1 = 32-bit segment
      unsigned sd_g : 1;                // granularity: limit scaled by 4K when set
      unsigned sd_base_31_24 : 8;        // high bits of segment base address
  };
  ```

#### d. 全局描述符表

- 全局描述符表（GDT）是一个保存多个段描述符的“数组”，其起始地址保存在全局描述符表寄存器GDTR中。GDTR长48位，其中高32位为基地址，低16位为段界限。
- 全局描述符表的一个demo

```c
#define SEG(type, base, lim, dpl)                        \
    (struct segdesc){                                    \
        ((lim) >> 12) & 0xffff, (base) & 0xffff,        \
        ((base) >> 16) & 0xff, type, 1, dpl, 1,            \
        (unsigned)(lim) >> 28, 0, 0, 1, 1,                \
        (unsigned) (base) >> 24                            \
    }
/* *
 * 全局描述符表。
 *
 * 内核和用户段是相同的（除了DPL）。要加载
 * %ss寄存器，CPL必须等于DPL。因此，我们必须复制
 * 用户和内核的段。定义如下。
 * - 0x0 : 未使用(总是故障 -- 用于捕获NULL远指针)
 * - 0x8 : 内核代码段
 * - 0x10: 内核数据段
 * - 0x18: 用户代码段
 * - 0x20: 用户数据段
 * - 0x28: 为tss定义，在gdt_init中初始化
 * */
static struct segdesc gdt[] = {
    SEG_NULL,
    [SEG_KTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_KDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_UTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_UDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_TSS]    = SEG_NULL,
};
```

#### e. 选择子

- 线性地址部分的**选择子是用来选择哪个描述符表和在该表中索引哪个描述符的**。选择子可以做为指针变量的一部分，从而对应用程序员是可见的，但是一般是由连接加载器来设置的。
- 段选择子结构
  - 索引（Index）：在描述符表中从8192个描述符中选择一个描述符。处理器自动将这个索引值乘以8（描述符的长度），再加上描述符表的基址来索引描述符表，从而选出一个合适的描述符。
  - 表指示位（Table Indicator，TI）：选择应该访问哪一个描述符表。0代表应该访问全局描述符表（GDT），1代表应该访问局部描述符表（LDT）。
  - 请求特权级（Requested Privilege Level，RPL）：保护机制。
    ![img](https://www.kro1lsec.com:442/images/2021/06/21/20210621151436.png)
- 全局描述符表的第一个描述符无法被CPU使用，所以当一个段选择子的索引（Index）部分和表指示位（Table Indicator）都为0的时（即段选择子指向全局描述符表的第一项时），可以当做一个空的选择子。当一个段寄存器被加载一个空选择子时，处理器并不会产生一个异常。但是，当用一个空选择子去访问内存时，则会产生异常。

### BIOS是如何苏醒的（重要）

- BIOS代码被写进ROM中，该ROM被映射到低端1M内存的顶部，即地址`0xF0000~0xFFFFF`。BIOS的入口地址为`0xFFFF0`。
  开机接电的一瞬间，CPU的CS:IP寄存器被强制初始化为`0xF000:0xFFF0`，即`0xFFFF0`。
  由于实模式下最高寻址1MB，故`0xFFFF0`处是一条跳转指令`jmp far f000:e05b`，跳转至BIOS真正的代码。之后便开始检测并初始化外设、与`0x000-0x3ff`建立数据结构，中断向量表IVT并填写中断例程。

- BIOS最后校验启动盘中位于0盘0道1扇区(MBR)的内容。如果此扇区末尾两个字节分别是魔数0x55和0xaa，则BIOS认为此扇区中存在可执行的程序，并加载该512字节数据到0x7c00，随后跳转至此继续执行。使用的跳转指令为jmp 0:0x7c00，该指令是jmp指令的直接绝对远转移用法。

  > 磁盘与磁道的编号从0开始，扇区编号从1开始。
  > 选择`0x7c00`是避免覆盖已有的数据以及被其他数据覆盖。

### MBR/Bootloader

- bootloader的作用

  - 切换保护模式 & 段机制
  - 从硬盘上读取kernel in ELF格式的ucore kernel（跟在MBR后面的扇区），并放到内存中固定。
  - 跳转到ucoreOS的入口点执行，将控制权移交给ucore OS。

- MBR是主引导记录（Master Boot Record），也被称为主引导扇区，是计算机开机以后访问硬盘时所必须要读取的第一个扇区。其内部前446字节存储了bootloader代码，其后是4个16字节的“磁盘分区表”。

  > MBR是整个硬盘最重要的区域，一旦MBR物理实体损坏时，则该硬盘基本报废。

- bootloader的入口点为`0x7c00`。以下是一个简单的类MBR程序，该程序只会将`1 MBR`字符串打印到屏幕上并挂起。通过该程序我们可以对MBR结构有了更深的了解。

  ```assembly
  ;主引导程序
  ;------------------------------------------------------------
  SECTION MBR vstart=0x7c00 ; 起始地址编译为0x7c00
    mov ax,cs   ; 此时的cs为0，用0来初始化所有的段寄存器
    mov ds,ax
    mov es,ax
    mov ss,ax
    mov fs,ax
    mov sp,0x7c00 ; 0x7c00 以下空间暂时安全，故可用做栈。
  
  ; 清屏 利用0x06号功能，上卷全部行，则可清屏。
  ; -----------------------------------------------------------
  ;INT 0x10   功能号:0x06   功能描述:上卷窗口
  ;------------------------------------------------------
  ;输入：
  ;AH 功能号= 0x06
  ;AL = 上卷的行数(如果为0,表示全部)
  ;BH = 上卷行属性
  ;(CL,CH) = 窗口左上角的(X,Y)位置
  ;(DL,DH) = 窗口右下角的(X,Y)位置
  ;无返回值：
    mov     ax, 0x600
    mov     bx, 0x700
    mov     cx, 0          ; 左上角: (0, 0)
    mov     dx, 0x184f     ; 右下角: (80,25),
          ; VGA文本模式中,一行只能容纳80个字符,共25行。
          ; 下标从0开始,所以0x18=24,0x4f=79
    int     0x10            ; int 0x10
  
  ;;;;;;;;;    下面这三行代码是获取光标位置    ;;;;;;;;;
  ;.get_cursor获取当前光标位置,在光标位置处打印字符.
    mov ah, 3   ; 输入: 3 号子功能是获取光标位置,需要存入ah寄存器
    mov bh, 0   ; bh寄存器存储的是待获取光标的页号
  
    int 0x10    ; 输出: ch=光标开始行,cl=光标结束行
        ; dh=光标所在行号,dl=光标所在列号
  
  ;;;;;;;;;    获取光标位置结束    ;;;;;;;;;;;;;;;;
  
  ;;;;;;;;;     打印字符串    ;;;;;;;;;;;
    ;还是用10h中断,不过这次是调用13号子功能打印字符串
    mov ax, message
    mov bp, ax    ; es:bp 为串首地址, es此时同cs一致，
        ; 开头时已经为sreg初始化
  
    ; 光标位置要用到dx寄存器中内容,cx中的光标位置可忽略
    mov cx, 5   ; cx 为串长度,不包括结束符0的字符个数
    mov ax, 0x1301  ; 子功能号13是显示字符及属性,要存入ah寄存器,
        ; al设置写字符方式 ah=01: 显示字符串,光标跟随移动
    mov bx, 0x2 ; bh存储要显示的页号,此处是第0页,
        ; bl中是字符属性, 属性黑底绿字(bl = 02h)
    int 0x10    ; 执行BIOS 0x10 号中断
  ;;;;;;;;;      打字字符串结束 ;;;;;;;;;;;;;;;
  
    jmp $   ; 始终跳转到这条代码，为死循环，使程序悬停在此
  
    message db "1 MBR"
    ; 用\0 将剩余空间填满
    times 510-($-$$) db 0 ; $指代当前指令的地址，$$指代当前section的首地址
    ; 最后两位一定是0x55, 0xaa
    db 0x55,0xaa
  ```

- 程序在section处使用了`vstart`伪指令。该指令只要求编译器将后面的所有数据与变量的地址以0x7c00开始编址，并不负责加载。而加载是由MBR加载器将该程序加载到0x7c00处。

- 执行以下代码，即可看到程序输出

  ```shell
  # 编译汇编代码
  nasm mbr.asm -o mbr.bin
  # 制作img镜像。注意dd指令的复制操作与cp不一样，它是针对磁盘来进行的复制
  #   将编译出的mbr.bin写进mbr.img中的第0块
  dd if=mbr.bin of=mbr.img bs=512 count=1 conv=notrunc
  # 使用i386架构启动mbr.img
  qemu-system-i386 mbr.img
  ```

  

### 硬件访问

- 硬件提供了软件方面的接口，操作系统通过软件（计算机指令）就能控制硬件。软件的逻辑需要作用在硬件上才能体现出来。

- 硬件在输出上大体分为串行和并行，相应的接口是串行接口和并行接口。

- 访问外部硬件的两种方式

  - 将某个外设的内存映射到一定范围内的地址空间。例如显卡。显卡是显示器的适配器，CPU 不直接和显示器交互，它只和显卡通信。其中的显存被映射到主机物理内存上的低端1MB的`0xB8000~0xBFFFF`。CPU往显存上写字节便是往屏幕上打印内容。显存地址分布如下
    ![img](https://www.kro1lsec.com:442/images/2021/06/21/20210621151509.png)

  - 通过IO接口。CPU只访问IO接口，不关心另一边的外设。IO接口上也存在一些寄存器。

    - CPU使用IO接口与外设通信。IO接口是连接CPU与外部设备的逻辑控制部件，可分为硬件软件两部分。

    - 计算机与IO接口的通信是通过计算机指令来实现的。通过软件指令选择IO接口上的功能、工作模式的做法，称为“IO接口控制编程”，通常是用端口读写指令in/out实现。端口是IO接口开发给CPU的接口，一般的IO接口都有一组端口，每个端口都有自己的用途。`in/out`指令使用方式如下。

      ```assembly
      in al, dx  # al/ax 用于存放从端口读入的数据，dx指端口号
      in ax, dx
      
      out dx, al
      out dx, ax
      out 立即数, al
      out 立即数, ax
      ```

- 例子：直接向显卡中写入数据

  ```assembly
  ;主引导程序
  ;------------------------------------------------------------
  SECTION MBR vstart=0x7c00 ; 起始地址编译为0x7c00
    mov ax,cs   ; 此时的cs为0，用0来初始化所有的段寄存器
    mov ds,ax
    mov es,ax
    mov ss,ax
    mov fs,ax
    mov sp,0x7c00 ; 0x7c00 以下空间暂时安全，故可用做栈。
    mov ax,0xb800 ; 0xb800-0xbffff 用于文本模式显示适配器
    mov gs,ax
  
  ; 清屏 利用0x06号功能，上卷全部行，则可清屏。
  ; -----------------------------------------------------------
  ;INT 0x10   功能号:0x06   功能描述:上卷窗口
  ;------------------------------------------------------
  ;输入：
  ;AH 功能号= 0x06
  ;AL = 上卷的行数(如果为0,表示全部)
  ;BH = 上卷行属性
  ;(CL,CH) = 窗口左上角的(X,Y)位置
  ;(DL,DH) = 窗口右下角的(X,Y)位置
  ;无返回值：
    mov     ax, 0x600
    mov     bx, 0x700
    mov     cx, 0          ; 左上角: (0, 0)
    mov     dx, 0x184f     ; 右下角: (80,25),
          ; VGA文本模式中,一行只能容纳80个字符,共25行。
          ; 下标从0开始,所以0x18=24,0x4f=79
    int     0x10            ; int 0x10
  
    ; 输出背景色绿色，前景色红色，并且跳动的字符串“1 MBR”
    mov byte [gs:0x00], '1'
    mov byte [gs:0x01], 0xa4   ; A表示绿色背景闪烁，4 表示前景色为红色
    mov byte [gs:0x02], ' '
    mov byte [gs:0x03], 0xa4
    mov byte [gs:0x04], 'M'
    mov byte [gs:0x05], 0xa4
    mov byte [gs:0x06], 'B'
    mov byte [gs:0x07], 0xa4
    mov byte [gs:0x08], 'R'
    mov byte [gs:0x09], 0xa4
    jmp $   ; 始终跳转到这条代码，为死循环，使程序悬停在此
  
    ; 用\0 将剩余空间填满
    times 510-($-$$) db 0
    ; 最后两位一定是0x55, 0xaa
    db 0x55,0xaa
  ```

### 中断与异常（重要）

- 在操作系统中，有三种特殊的中断事件：
  - 异步中断(asynchronous interrupt)。这是由CPU外部设备引起的外部事件中断，例如I/O中断、时钟中断、控制台中断等。
  - 同步中断(synchronous interrupt)。这是CPU执行指令期间检测到不正常的或非法的条件(如除零错、地址访问越界)所引起的内部事件。
  - 陷入中断(trap interrupt)。这是在程序中使用请求系统服务的系统调用而引发的事件。
- 当CPU收到中断或者异常的事件时，它会暂停执行当前的程序或任务，通过一定的机制跳转到负责处理这个信号的相关处理例程中，在完成对这个事件的处理后再跳回到刚才被打断的程序或任务中。
- 其中，中断向量和中断服务例程的对应关系主要是由IDT（中断描述符表）负责。操作系统在IDT中设置好各种中断向量对应的中断描述符，留待CPU在产生中断后查询对应中断服务例程的起始地址。而IDT本身的起始地址保存在`idtr`寄存器中。
- 当CPU进入中断处理例程时，`eflags`寄存器上的`IF`标志位将会自动被CPU置为0，待中断处理例程结束后才恢复`IF`标志。

#### a. 中断描述符表

- 中断描述符表（Interrupt Descriptor Table, IDT）把每个中断或异常编号和一个指向中断服务例程的描述符联系起来。同GDT一样，IDT是一个8字节的描述符数组，但IDT的第一项可以包含一个描述符。
- IDT可以位于内存的任意位置，CPU通过IDT寄存器（IDTR）的内容来寻址IDT的起始地址。

#### b. IDT gate descriptors

- 中断/异常应该使用`Interrupt Gate`或`Trap Gate`。唯一区别就是：当调用`Interrupt Gate`时，Interrupt会被CPU自动禁止；而调用`Trap Gate`时，CPU则不会去禁止或打开中断，而是保留原样。

  > 这其中的原理是当CPU跳转至`Interrupt Gate`时，其eflag上的IF位会被清除。而`Trap Gate`则不改变。

- IDT中包含了3种类型的Descriptor

  - Task-gate descriptor
  - Interrupt-gate descriptor （中断方式用到）
  - Trap-gate descriptor（系统调用用到）

#### c. 中断处理过程

##### 1) 起始阶段

- CPU执行完每条指令后，判断中断控制器中是否产生中断。如果存在中断，则取出对应的中断变量。
- CPU根据中断变量，到IDT中找到对应的中断描述符。
- 通过获取到的中断描述符中的段选择子，从GDT中取出对应的段描述符。此时便获取到了中断服务例程的段基址与属性信息，跳转至该地址。
- CPU会根据CPL和中断服务例程的段描述符的DPL信息确认是否发生了特权级的转换。若发生了特权级的转换，这时CPU会从当前程序的TSS信息（该信息在内存中的起始地址存在TR寄存器中）里取得该程序的内核栈地址，即包括内核态的ss和esp的值，并立即将系统当前使用的栈切换成新的内核栈。这个栈就是即将运行的中断服务程序要使用的栈。紧接着就将当前程序使用的用户态的ss和esp压到新的内核栈中保存起来；
- CPU需要**开始保存当前被打断的程序的现场**（即一些寄存器的值），以便于将来恢复被打断的程序继续执行。这需要利用内核栈来保存相关现场信息，即依次压入当前被打断程序使用的eflags，cs，eip，errorCode（如果是有错误码的异常）信息；
- CPU利用中断服务例程的段描述符将其第一条指令的地址加载到cs和eip寄存器中，**开始执行中断服务例程**。这意味着先前的程序被暂停执行，中断服务程序正式开始工作。

##### 2) 终止阶段

- 每个中断服务例程在有中断处理工作完成后需要通过
  iret （或iretd ）指令恢复被打断的程序的执行。CPU执行IRET指令的具体过程如下：
  - 程序执行这条iret指令时，首先会从内核栈里弹出先前保存的被打断的程序的现场信息，即eflags，cs，eip重新开始执行；
  - 如果存在特权级转换（从内核态转换到用户态），则还需要从内核栈中弹出用户态栈的ss和esp，即栈也被切换回原先使用的用户栈。
- 如果此次处理的是带有错误码（errorCode）的异常，CPU在恢复先前程序的现场时，并不会弹出errorCode，需要要求相关的中断服务例程在调用iret返回之前添加出栈代码主动弹出errorCode。
