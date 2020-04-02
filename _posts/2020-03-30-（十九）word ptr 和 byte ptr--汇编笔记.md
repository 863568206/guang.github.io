---
layout:     post
title:      （十九）word ptr 和 byte ptr--汇编笔记
subtitle:   汇编语言
date:       2020-03-30
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - 汇编语言
---

## 指令要处理的数据有多长

8086CPU的指令，可以处理两种尺寸的数据，**byte**和**word**。所以在机器指令中要指明，指令进行的是字操作还是字节操作。

**1）通过寄存器名指明要处理的数据的尺寸。**

字操作：

	mov ax, 1
	
	mov bx, ds:[0]
	
	mov ds, ax
	
	mov ds:[0], ax
	
	inc ax
	
	add ax, 1000

字节操作：

	mov al, 1
	
	mov al, bl
	
	mov al, ds:[0]
	
	mov ds:[0], al
	
	inc al
	
	add al, 100

**2）在没有寄存器名的情况下，用操作符“X ptr”指明内存单元的长度，X在汇编指令中可以为word或byte。**

用**word ptr**指明了指令访问的内存单元是一个字单元。

	mov word ptr ds:[0], 1
	
	inc word ptr [bx]
	
	inc word ptr ds:[0]
	
	add word ptr [bx], 2

用**byte ptr**指明了指令访问的内存单元是一个字节单元。

	mov byte ptr ds:[0], 1
	
	inc byte ptr [bx]
	
	inc byte ptr ds:[0]
	
	add byte ptr [bx], 2

在没有寄存器参与的内存单元访问指令中，用**word ptr**或**byte ptr**显性地指明所要访问的内存单元的长度是很有必要的。否则，CPU无法得知所要访问的单元是字单元，还是字节单元。

例：

内存：

	2000：1000 FF
	2000: 1001 FF
	2000: 1002 FF

指令：

	mov ax, 2000H
	mov ds, ax
	mov byte ptr ds:[1000H], 1

则地址变为：

	2000：1000 01
	2000: 1001 FF
	2000: 1002 FF	

“**mov byte ptr ds:[1000H], 1**”访问的地址为**ds:1000H**的字节单元，修改的是**ds:1000H**单元的内容。

指令：

	mov ax, 2000H
	mov ds, ax
	mov word ptr ds:[1000H], 1

则地址变为：

	2000：1000 01
	2000: 1001 00
	2000: 1002 FF

“**mov word ptr ds:[1000H], 1**”访问的地址为**ds:1000H**的字单元，修改的是**ds:1000H**和**ds:1001H**两个单元的内容。（此时**ds:1000H**是低位，**ds:1001H**是高位）

3）其他方法

有些指令默认了访问的是字单元还是字节单元，比如，**push[1000H]**就不用指明访问的是字单元还是字节单元，因为**push**指令只进行 **字操作**。