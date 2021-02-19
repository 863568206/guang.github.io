---
layout:     post
title:      ShellCold简介
subtitle:   c++
date:       2020-05-06
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

## ShellCode


借鉴：[ShellCode](https://blog.csdn.net/u013761036/article/details/52853319)

调用函数时会call，在call的之前会push函数的参数，call时回将函数要执行的下一条地址入栈，而ret会通过pop出这条地址进行返回，即可以通过修改这条地址从而达到调用其他函数的目的。

	#include "stdafx.h"
	#include <string>
	#include <windows.h>
	
	using namespace std;
	
	DWORD dwGetHomeAddress; //保存跳回main函数的地址
	typedef VOID (*TYPEFUN)();
	
	TYPEFUN m_pF0; //0号函数的地址
	TYPEFUN m_pF1; //1号函数的地址
	TYPEFUN m_pF2; //2号函数的地址
	
	void Func0()
	{
		MessageBox(NULL, L"F0", L"tit", MB_OK);
		DWORD dwNextCmdAddress;
		_asm
		{
	
			mov esp, ebp
			
			//pop dwNextCmdAddress //ret 会将其出栈，这步其实没必要
			push dwGetHomeAddress
	
			ret
		}
	}
	
	
	void Func1()
	{
		MessageBox(NULL, L"F1", L"tit", MB_OK);
		DWORD dwNextCmdAddress;
		_asm
		{
	
			mov esp, ebp
			//在调用前并没有将ebp压入栈，只是通过ret调用过来
			//pop dwNextCmdAddress  //ret 会将其出栈，这步其实没必要
			push m_pF0
	
			ret
		}
	
	}
	
	void Func2()
	{
		MessageBox(NULL, L"F2", L"tit", MB_OK);
	
		DWORD dwNowEbp, dwNowEsp;
		DWORD dwStackEbp, dwStackEsp;
		
		_asm
		{
			
			//保存寄存器的值
			mov dwNowEbp, ebp
			mov dwNowEsp, esp 
		
			mov esp, ebp //刚进站时esp的值给了ebp，所以此时esp是栈顶的位置
			pop dwStackEbp //对应出站时的 pop ebp，即这是进站时候 push ebp的值
			pop dwGetHomeAddress //保存main函数的地址，再call 会压入栈下一条指令的地址，所以这是main函数下一条指令的地址
			push m_pF1  //将下一条指令的地址换位F1函数的，即ret出栈时返回的地址
			push dwStackEbp //还原栈的位置
	
			
			//恢复寄存器的值
			mov ebp, dwNowEbp
			mov esp, dwNowEsp
			
		}
	}
	
	int _tmain(int argc, _TCHAR* argv[])
	{
		m_pF0 = Func0;
		m_pF1 = Func1;
		m_pF2 = Func2;
		Func2();
		MessageBox(NULL, L"main", L"tit", MB_OK);
		return 0;
	}


# 密蒙花 -- 幸福来的如此神秘
