---
title: "[uCore]Lab1"
date: "2021-06-13"
tags: ["uCore"]
ShowToc: true
TocOpen: true
draft: false
---

## [练习1]
[练习](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_2_1_1_ex1.html)
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

## [练习2]

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

## [练习3]

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

## [练习4]

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

## [练习5]

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

## [练习6]

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

Linux 内核是在进入保护模式后才建立中断例程的，不过在保护模式下，中断向量表已经不存在了，取而代之的是中断描述符表（Interrupt Descriptor Table，IDT）。Linux 的系统调用和DOS中断调用类似，不过Linux是通过`int 0x80`指令进入一个中断程序后再根据eax寄存器的值来调用不同的子功能函数的

### 操作系统如何识别文件系统

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
    ![img](https://www.kro1lsec.com:442/images/2021/06/14/20210614153109.png)
  - 运算单元根据控制单元的信号，进行运算。
  - 存储单元CPU内部的L1、L2缓存及寄存器。这部分缓存采用SRAM存储器。SRAM不需要刷新电路即可保存内部存储的数据，但因为体积较大，集成度较低。

- CPU中的寄存器分为两大类：程序可见寄存器（例如通用寄存器、段寄存器）和程序不可见寄存器（例如中断描述符寄存器IDTR）。
  ![img](https://www.kro1lsec.com:442/images/2021/06/14/20210614153044.png)

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