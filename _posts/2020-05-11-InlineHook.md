---
layout:     post
title:      InlineHook
subtitle:   c++
date:       2020-05-11
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

## InlineHook -- jmp

1) TargetFun：要被Hook的目标函数

2）DetourFun：用于代替TargetFun的自定义函数

3）TrampolineFun：调用原函数的入口

步骤：

1）确定Hook方式及需要在Trampoline中执行的命令。

2）准备TrampolineFun函数：因为TrampolineFun函数的每条指令都需要自行控制，需要该函数为裸函数，这样就可以保证编译器不会为该函数添加额外的指令。

3）准备jmp指令并写入

计算跳转偏移:（因为都在同一块内存中）

	计算公式：int xxxx = MyFunAddr – SystemFunAddr - CodeLength;
			  跳转偏移 = 目标地址 - 指令所在- 指令长度

	一般为 相对JMP(0xE9) + 4字节地址，方便计算 
	        公式：源地址 + 相对偏移 = 目的地址
	        公式：目的地址 - 源地址 = 相对偏移

**对MssageBoxA的Hook：**

	#include "stdafx.h"
	#include <windows.h>
	#include <string>
	
	using namespace std;
	
	ULONG g_jmpBackAddr = 0; //保存MessageBoxA函数中，我们写入jmp指令，下一条指令的位置。
	
	//定义裸函数：TrampolineFun函数
	//关键字：_declspec(naked) 这样编译器不会为该函数添加额外的指令（例如典型的保存栈帧和开辟栈空间指令）
	//可以将函数原型定义与TargetFun函数完全一样，以便调用
	_declspec(naked) int WINAPI OriginalMessageBox(HWND hWnd, LPCTSTR lpText, LPCTSTR lpCaption, UINT uType)
	{
		_asm
		{
			//写入的jmp指令破坏了原来的前3条指令，因此在这里执行原函数的前3条指令
			mov edi, edi
			push ebp
			mov ebp, esp
			jmp g_jmpBackAddr  //跳回原函数被Hook指令之后的位置，绕过自己安装的Hook
		}
	}
	
	//自己的Hook函数
	int WINAPI My_MessageBoxA(HWND hWnd, LPCTSTR lPText, LPCTSTR lpCaption, UINT uType)
	{
		int ret;
		string newCaption = "good";
		string newText = "MessageBox Hook good" ;
		printf("有人调用MessageBox!\n");
		uType |= MB_ICONERROR;
		ret = OriginalMessageBox(hWnd, (LPCTSTR)newText.c_str(), (LPCTSTR)newCaption.c_str(), uType);
		return ret;
	}
	
	
	//Hook--jmp
	void HookMessageBoxA()
	{
		//获取跳回原函数被Hook指令之后的位置
		PBYTE AddrMessageBoxA = (PBYTE)GetProcAddress(GetModuleHandleA("user32.dll"), "MessageBoxA");
		g_jmpBackAddr = (ULONG)AddrMessageBoxA + 5; //函数地址加5字节偏移
	
	
		//向原函数添加跳转到DetourFun的jmp
		BYTE newEntry[5] = { 0 };
		newEntry[0] = 0xE9; //Long Jmp 的机器码
		//计算跳转偏移
		//计算公式：int xxxx = MyFunAddr – SystemFunAddr - CodeLength;
		*(ULONG*)(newEntry + 1) = (ULONG)My_MessageBoxA - (ULONG)AddrMessageBoxA - 5;
	
		//修改MessageBoxA函数开头，写入我们的jmp
		DWORD dwOldProtect;
		MEMORY_BASIC_INFORMATION mbi = { 0 };
		VirtualQuery(AddrMessageBoxA, &mbi, sizeof(MEMORY_BASIC_INFORMATION));
		VirtualProtect(mbi.BaseAddress, 5, PAGE_EXECUTE_READWRITE, &dwOldProtect);
		memcpy(AddrMessageBoxA, newEntry, 5);   //写入jmp 指令的机器码
		VirtualProtect(mbi.BaseAddress, 5, dwOldProtect, &dwOldProtect);
		
		//或者WriteProcessMemory
		if (0)
		{
			DWORD dwBytesReturned = 0;
			WriteProcessMemory((HANDLE)GetCurrentProcessId(), AddrMessageBoxA, newEntry, 5, &dwBytesReturned);
		}
	}
	
	
	int _tmain(int argc, _TCHAR* argv[])
	{
		MessageBoxA(NULL, "zhongchang", "Test", MB_OK);
	
		HookMessageBoxA();
	
		MessageBoxA(NULL, "zhongchang", "Test", MB_OK);
		
		string str = "zhongchang";
		string str1 = "Test";
		OriginalMessageBox(NULL, (LPCTSTR)str.c_str(), (LPCTSTR)str1.c_str(), MB_OK);
	
		return 0;
	}


## 其他Hook方式

1）push/ret方式

	0040E9D1 68 44332211    push 11223344
	
	0040E9D6 C3             retn
	
	memcpy(newEntry, "\x68\x44\x33\x22\x11\xC3", 6);
	*(ULONG*)(newEntry + 1) = (ULONG)pFnDetourFun;

2) jmp eax方式

	7c809B12 B8 44332211  mov eax, 11223344
	
	7c809B17 FFE0         jmp eax
	
	memcpy(newEntry, "\xB8\x44\x33\x22\x11\xFF\xE0", 7);
	*(ULONG*)(newEntry + 1) = (ULONG)pFnDetourFun;

3) HotPatch方式

先在TargetFun - 5d的位置写LongJmp指令，再在TargetFun开头写Short Jmp指令。(因为会对齐所以存在nop)

例：

	xxxxxxx0  E9 53 01 79 8B      jmp DetourFun   //函数入口上方因对齐残留的空间
	xxxxxxx1  EB F9               jmp xxxxxxx0    //函数之前的开始位置
	xxxxxxx2  55                  push ebp
	xxxxxxx3  8B EC               mov ebp, esp

实现：
	
	77D507E5     E9 66086B88  jmp InlineHo.00401050
	77D507EA     EB F9        jmp short USER32.77D507E5
	
	newEntry[0] = 0xEB; 
	newEntry[1] = 0xF9;
	HotPatchCode[0] = 0xE9;
	*(ULONG*)(HotPatchCode + 1) = (ULONG)pFnDetourFun - ((ULONGHookPoint - 5)) - 5;
	
## 跳转指令问题

在进行InlineHook时，必须要手动构造跳转指令，而跳转指令jmp（call指令也是）在计算转移目的地址时都是按相对偏移计算的。以直接jmp指令（机器码0xE9）为例，他的跳转范围以当前位置为分解先，前后各2GB空间，这在x86平台总内存空间为4GB的情况下是没有问题的，但在x64平台上就不够用了（Wow64进程被视为与x86模式等同）。因为在x64平台上，各模块的加载基址之间一般都超过4GB，不能像在x86平台上那样方便地从被Hook的模块跳转到我们的Detour函数所在的模块，所以，要使用的转移指令必须能够直接包含目标地址（64位）的指令。

常用方法：

1）mov rax, addr/jmp rax

此时寄存器扩充为64位，共占12个字节。

	mov rax, addr
	jmp rax

	机器码                   指令
	48B88877665544332211   mov rax, 1122334455667788
	ffe0                   jmp rax

2）push/retn

与在x86平台不同，x64平台上的push指令只能push一个32位值，但是由于对齐的关系，实际上占用了64位，所以可以先push地址的低位，再修改高位。这种方法不改变任何寄存器，占用14个字节。

	机器码                      指令
	6844332211                 push 55667788H
	c744240488776655           mov dword ptr [rsp + 4], 11223344H //高8位地址在后面
	c3                         ret


3) jmp [addr]

间接寻址jmp

						机器码                  指令
	00000000`7708cb70	ff2500000000           jmp qword ptr [00000000`7708cb76]
	00000000`7708cb76   887766                 mov byte ptr [rdi + 66h], dh
	00000000`7708cb79   55                     push rbp
	00000000`7708cb7a   443322                 xor r12d, dword ptr [rdx]
	00000000`7708cb7d   119090488d05           adc dword ptr [rax + 58D4890H], edx

	0:000> dq 00000000`7708cb76
	00000000`7708cb76 11223344`55667788  085b1905`8d489090

在x64平台上，因为0xff25方式的jmp也是按相对偏移计算的，所以上面这个跳转的实际寻址地址是：

	Adderss = (eip + 0x0(偏移量) + 0x6(当前指令长度)) = 0x000000007708cb76

而0x000000007708cb76处的值实际上是QWORD 0x1122334455667788，这才是跳转的最终地址。

这种跳转方式占用的空间为14个字节，并不改变任何寄存器。

## 龙胆花 -- 喜欢看忧伤时的你 



