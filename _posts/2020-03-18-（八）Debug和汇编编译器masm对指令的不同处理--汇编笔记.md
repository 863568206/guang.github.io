---
layout:     post
title:      （八）Debug和汇编编译器masm对指令的不同处理--汇编笔记
subtitle:   汇编语言
date:       2020-03-18
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - 汇编语言
---

## Debug和汇编编译器masm对指令的不同处理

**idata表示一个常量。**

**debug中编译的是程序（汇编指令）而不是源程序（汇编指令 + 伪指令）。**

**Debug**和编译器 **masm** 对形如 **mov ax, [idata]** ，解释是不同的，**debug** 将它解释为“**[idata]**是一个内存单元，idata为内存单元的偏移地址”，而编译器将其解释为“**idata**，单纯的一个常量数，不代表地址。”

**debug：**

	mov ax, [1]

**汇编：**

	mov ax, 1

**解决办法：**

1）用[bx]（用寄存器保存一下）

	mov ax, 2000H
	mov ds, ax
	mov bx, 0
	mov al, [bx]

2) 显示的给出段地址所在的段寄存器

	mov ax, 2000H
	mov ds, ax
	mov al, ds:[0]

比较（汇编源程序，汇编编译器）：

	mov al, [0] 含义: al = 0，同 mov al, 0 将常量0送入al中
	
	mov al, ds:[0] 含义：al = ds * 16 + 0 将内存单元中的数据送入al中

	mov al, [bx] 含义：al = ds * 16 + bx 将内存单元中的数据送入al中
	
	mov al，ds：[bx] 含义：同mov al, [bx]

**所以：**

&emsp;1) 在汇编源程序中，如果用指令访问一个内存单元，则在指令中必须用“[...]”表示内存单元，如果在"[]"里用一个常量直接给出内存单元的偏移地址，就要在“[]”的前面显示地给出段地址所在的段寄存器：

	mov al，ds：[0]

如果没有在[]的前面显示地给出段地址所在的段寄存器，比如：

	mov al, [0]

那么，编译器masm将把指令中的“[idata]”解释为“idata”。

&emsp;2）如果在“[]”里用寄存器，比如bx，间接给出内存单元的偏移地址，则段地址默认在ds中。当然，也可以显示地给出段地址所在地段寄存器。