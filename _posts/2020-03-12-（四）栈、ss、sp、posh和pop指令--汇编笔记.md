---
layout:     post
title:      （四）栈、ss、sp、posh 和 pop 指令--汇编笔记
subtitle:   汇编语言
date:       2020-03-12
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - 汇编语言
---

## 栈

**栈是一种具有特殊的访问方式的存储空间。它的特殊性就在于，最后进入这个空间的数据，最先出去。**

## CPU提供的栈机制

	PUSH（入栈） push ax 将寄存器ax中的数据送入栈中
	
	pop（出栈）  pop ax 从栈顶取出数据送入ax

**8086CPU的入栈和出栈是以字为单位进行的**。（即2个连续地址的内存单元）高地址单元存放高8位，低地址单元存放低8位。（一般下面为大）

段寄存器**SS**（段地址） 寄存器**SP**（偏移地址）

**在任意时刻，SS:SP指向栈顶元素。**

**ESP：栈指针寄存器(extended stack pointer)，其内存放着一个指针，该指针永远指向系统栈最上面一个栈帧的栈顶。**

**EBP：基址指针寄存器(extended base pointer)**


**push &emsp;ax** 执行：

&emsp;&emsp;1) SP = SP - 2, **SS:SP**指向当前栈顶前面的单元，以当前栈顶前面的单元为新的栈顶。（SS的位置是地址起始端，而SP的地址位置一般是从栈底开始的也就是地址位置大的那一边，所以入栈就是从底下往上放，所以SP的地址会越来越小，SP - 2 是因为入栈和出栈是以字为单位，即2个连续地址的内存单元）

&emsp;&emsp;2）将ax中的内容送入SS:SP指向的内存单元处，**SS：SP此时指向新栈顶**。

在任意时刻，SS：SP指向栈顶元素，当栈为空的时候，栈中没有元素，也就不存在栈顶元素，所以SS：SP只能指向栈的最底部单元下面的单元，该单元的偏移地址为栈最底部的字单元的偏移地址+2（可以看作就是最后一个元素出栈后，sp+2的行为）。（当最后一个元素出栈后此时SP的地址为栈最底部的地址-2，所以栈最底部的字单元的偏移地址）

**栈空，ss：sp指向栈空间最高地址单元的下一个单元。**

**pop&emsp;ax**的执行过程和**push&emsp;ax**刚好相反：

1)将SS：SP指向的内存单元处的数据送入ax中。

2）SP = SP + 2，SS：SP指向当前栈顶下面的单元，以当前栈顶下面的单元为新的栈顶。

出栈后，SS：SP指向新的栈顶，**pop操作前的栈顶元素依然存在**，但是，它并不在栈中。当再次执行push等入栈指令后，SS：SP移至之前的位置，并在里面写入新的数据，他将被覆盖。

## 栈顶超界的问题

当栈满的时候再使用push，或空栈的时候再使用pop，都会发生栈顶超界的问题。

## push、pop指令

1）通用寄存器

	push 寄存器  将一个寄存器中的数据入栈
	
	pop 寄存器   出栈，用一个寄存器接受出栈的数据

2）段寄存器

	push 段寄存器  将一个段寄存器中的数据入栈
	
	pop 段寄存器   出栈，用一个段寄存器接受出栈的数据

3）内存单元

	push [内存单元]  将一个内存单元中的数据入栈 (栈操作都是以字为单位，即2个连续地址的内存单元)
	
	pop [内存单元]   出栈，用一个内存单元接受出栈的数据

**例：**

	mov ax, 1000H
	
	mov ds, ax
	
	push [0]
	
	pop [2]

**例2：**10000H ~ 1000FH 为栈

	mov ax, 1000H
	
	mov ss, ax
	
	mov sp, 0010H     -> 1000FH + 1 //栈空，ss：sp指向栈空间最高地址单元的下一个单元。
	
	push ax
	
	push bx
	
	push ds

**例3：**

	mov ax, 1000H
	
	mov ss, ax
	
	mov sp, 0010H
	
	mov ax, 001AH
	
	mov bx, 001B
	
	push ax
	
	push bx
	
	mov ax, 0000H (sub ax, ax 这个2个字节，前面3个字节)
	
	mov bx, 0000H (sub bx, bx 这个2个字节，前面3个字节)
	
	pop bx
	
	pop ax

push、pop实质上就是一种内存传送指令，可以在寄存器和内存之间传送数据，与mov指令不同的是，**push和pop指令访问的内存单元的地址不是在指令中给出的，而是由SS:SP指出的。同时，push和pop指令还要改变SP中的内容。**

push和pop指令同mov指令不同，CPU执行mov指令只需要一步操作，就是传送，而执行push、pop指令却需要两步操作。**执行push时，CPU的两步操作是：先改变SP，后向SS：SP处传送。执行pop时，CPU的两步操作是：先读取SS：SP处的数据，后改变SP。**

push、pop等栈操作指令，修改的只是SP。也就是说，栈顶的变化范围最大为：0~FFFFH。

SS、SP指示栈顶；改变SP后写入内存的入栈指令；读内存后改变SP的出栈指令。

8086CPU只记录栈顶，栈空间的大小我们要自己管理。

## 栈段

在编程时，可以根据需要，将一组内存单元定义为一个段。将一段内存当作栈段，仅仅是我们在编程中的一种安排，CPU并不会由于这种安排，就在执行push、pop等栈操作指令时自动地将我们定义地栈段当作栈空间来访问。

**例：**

10000H~1FFFFH空间当作栈，初始状态为空，此时SS=1000H，SP = ？

&emsp;&emsp;当栈只有一个元素时，SS = 1000H SP = FFFEH

&emsp;&emsp;栈为空相当于唯一地元素出栈，即 SP + 2

&emsp;&emsp;当栈为空时，SS = 1000H SP = 0000H  //不会向上进位

**例：**

一个栈最大可以设为多少？

&emsp;&emsp;push、pop等指令在执行地时候只修改SP，所以栈顶地变化范围时0~FFFFH，从栈空时候地 SP = 0，一直压栈，直到栈满时，SP = 0;如果再次压栈，栈顶将环绕，覆盖了原来栈中地内容。所以一个栈段地容量最大为64KB。（FFFF + 1 = 0000H，只能存放8位所以不能获取进位。）

## 松虫草 -- 凄凉的等待
