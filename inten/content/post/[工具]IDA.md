---
title: "[工具]IDA"
date: 2021-03-10T15:07:34+08:00
draft: false
---

### 简介

加密与解密 p122

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528171820.png)

![img](https://www.kro1lsec.com:442/images/2021/05/28/20210528171822.png)

### 再探 IDA

#### 常用操作

##### 窗口介绍

Structures 结构体窗口

##### 显示硬编码

ACDU

a 字符串

##### 跳转指令

G

![image-20201116160213180](https://www.kro1lsec.com:442/images/2021/05/28/20210528171824.png)

##### 搜索（汇编）指令

Alt + t

##### 重命名

**n**

**Alt + q** 引用结构体窗口中对应的结构体

局部变量指定结构体 **T**

**y** 改函数的声明

d 改类型

##### 注释

; 会在任何引用该位置的地方留下注释的副本

：引用的地方没有副本

针对函数的注释 选中函数按;

会显示在被注释函数的上方

##### 交叉引用

Ctrl+x

所有调用该函数 / 数据的地方

在 F5 的界面里，选定一个函数按 **X** 交叉引用

##### IDA PATCH

修改错误无法撤销

保存文件

![image-20201116160557370](https://www.kro1lsec.com:442/images/2021/05/28/20210528171828.png)

##### 导入头文件

![image-20201125211537286](https://www.kro1lsec.com:442/images/2021/05/28/20210528171830.png)

注意：即使导入头文件，也无法识别标准库函数

##### 新建结构体

在结构体窗口按 insert

d 添加成员

#### 实战技巧

##### IDA 类型识别错误

```c++
        *(_BYTE *)(i + a2) = byte_442198[j] ^ (*(_BYTE *)(i + a1) + 13);
```

当出现 *(_BYTE* ) 这种类型，多半是识别错误

IDA 是动态识别的

有时候函数可能会少参数，先进入函数再退出即可正确识别

**字符串识别错误**

![image-20201124223325519](https://www.kro1lsec.com:442/images/2021/05/28/20210528171832.png)

再按一次 f5 实现刷新

##### [KEYPATCH](https://github.com/keystone-engine/keypatch)

**修改汇编指令**

patch 指令和正常汇编不太一样

IDA 不支持 jmp 0x12345678 的写法，应该改成 jmp 12345678h

![image-20201029161518962](https://www.kro1lsec.com:442/images/2021/05/28/20210528171834.png)

ida 对数据作用有严格区分 需要先转化成 Code 再 assembly

**修改硬编码**

![image-20201029161954447](https://www.kro1lsec.com:442/images/2021/05/28/20210528171836.png)

强制分析

u ：取消分析

d：作为数据分析

c ： 作为汇编指令分析（选中指令起始的那一行）

例：

![image-20201029160924609](https://www.kro1lsec.com:442/images/2021/05/28/20210528171839.png) 改着改着 ida 就识别不出来这是汇编指令了，强制分析也不顶用 。

![image-20201029161023964](https://www.kro1lsec.com:442/images/2021/05/28/20210528171842.png)

要只选中 E9 一行那一行，ida 分析的才是正确的。多选几行或者选中 f9 那行都不能正确识别 。

同样的硬编码翻译成汇编还可以不一样

“你把那个补码转成原码对比下就知道为啥了”

##### 内存 dump 脚本

调试程序时偶尔会需要 dump 内存，但 IDA Pro 没有直接提供此功能，可以通过脚本来实现。

```
import idaapi
data = idaapi.dbg_read_memory(start_address, data_length)fp = open('path/to/dump', 'wb')fp.write(data)fp.close()
```

##### 堆栈不平衡

某些函数在使用 f5 进行反编译时，会提示错误 "sp-analysis failed"，导致无法正确反编译。原因可能是在代码执行中的 pop、push 操作不匹配，导致解析的时候 esp 发生错误。

解决办法步骤如下：

1. 用 Option->General->Disassembly, 将选项 Stack pointer 打钩
2. 仔细观察每条 call sub_xxxxxx 前后的堆栈指针是否平衡
3. 有时还要看被调用的 sub_xxxxxx 内部的堆栈情况，主要是看入栈的参数与 ret xx 是否匹配
4. 注意观察 jmp 指令前后的堆栈是否有变化
5. 有时用 Edit->Functions->Edit function...,然后点击 OK 刷一下函数定义

# 常用插件

- [IDA FLIRT Signature Database](https://github.com/push0ebp/sig-database) -- 用于识别静态编译的可执行文件中的库函数
- [Find Crypt](https://github.com/polymorf/findcrypt-yara) -- 寻找常用加密算法中的常数（需要安装 [yara-python](https://github.com/VirusTotal/yara-python)）
- [IDA signsrch](https://github.com/nihilus/IDA_Signsrch) -- 寻找二进制文件所使用的加密、压缩算法
- [Ponce](https://github.com/illera88/Ponce) -- 污点分析和符号化执行工具
- [snowman decompiler](https://github.com/yegord/snowman/tree/v0.1.0) -- C/C++反汇编插件（F3 进行反汇编）
- [CodeXplorer](https://github.com/REhints/HexRaysCodeXplorer) -- 自动类型重建以及对象浏览（C++）（jump to disasm)
- [IDA Ref](https://github.com/nologic/idaref) -- 汇编指令注释（支持arm，x86，mips）
- [auto re](https://github.com/a1ext/auto_re) -- 函数自动重命名
- [nao](https://github.com/tkmru/nao) -- dead code 清除
- [HexRaysPyTools](https://github.com/igogo-x86/HexRaysPyTools) -- 类/结构体创建和虚函数表检测
- [DIE](https://github.com/ynvb/DIE) -- 动态调试增强工具，保存函数调用上下文信息
- [sk3wldbg](https://github.com/cseagle/sk3wldbg) -- IDA 动态调试器，支持多平台
- [idaemu](https://github.com/36hours/idaemu) -- 模拟代码执行（支持X86、ARM平台）
- [Diaphora](https://github.com/joxeankoret/diaphora) -- 程序差异比较
- [Keypatch](https://github.com/keystone-engine/keypatch) -- 基于 Keystone 的 Patch 二进制文件插件
- [FRIEND](https://github.com/alexhude/FRIEND) -- 哪里不会点哪里，提升汇编格式的可读性、提供指令、寄存器的文档等
- [SimplifyGraph](https://github.com/fireeye/SimplifyGraph) -- 简化复杂的函数流程图
- [bincat](https://github.com/airbus-seclab/bincat) -- 静态二进制代码分析工具包，2017 Hex-Rays 插件第一名
- [golang_loader_assist](https://github.com/strazzere/golang_loader_assist) -- Golang编译的二进制文件分析助手
- [BinDiff](https://www.zynamics.com/bindiff.html)
