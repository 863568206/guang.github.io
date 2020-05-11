---
layout:     post
title:      AddressHook -- IATHook
subtitle:   c++
date:       2020-05-07
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

## IAT--Hook

注释已经写的很清楚了，**InstallModuleIATHook**主要是对输入表进行修改。

1) 定义一个Detour函数

2）查表、遍历匹配替换原地址，关闭写保护，以及写入Detour函数的地址。



	#include "stdafx.h"
	#include <windows.h>
	#include <dbghelp.h>
	#include <string>
	#pragma comment(lib, "Dbghelp.lib") 
	
	using namespace std;
	
	//定义一个与被Hook函数原型一致的函数指针
	typedef int (WINAPI *PFN_MessageBoxA) (HWND hWnd, LPCTSTR lPText, LPCTSTR lpCaption, UINT uType);
	
	//保存原始MessageBoxA的地址
	PFN_MessageBoxA OldMessageBox = NULL;
	
	//自己的Hook函数
	int WINAPI My_MessageBoxA(HWND hWnd, LPCTSTR lPText, LPCTSTR lpCaption, UINT uType)
	{
		int ret;
		string newCaption = "good";
		string newText = "MessageBox Hook good" ;
		printf("有人调用MessageBox!\n");
		uType |= MB_ICONERROR;
		ret = OldMessageBox(hWnd, (LPCTSTR)newText.c_str(), (LPCTSTR)newCaption.c_str(), uType);
		return ret;
	}
	
	
	BOOL InstallModuleIATHook(
							  HMODULE hModToHook,  //待Hook的模块基址(可以是当前进程的句柄)
							  char * szModuleName, //目标函数所在模块的名字
							  char * szFuncName,    //目标函数的名字
							  PVOID DetourFunc,    //detour函数的地址
							  PULONG * pThunkPointer, //用于接受指向修改的位置的指针
							  ULONG * pOriginalFuncAddr //用于接受原始函数的地址
							  )
	{
		PIMAGE_IMPORT_DESCRIPTOR pImportDescriptor;  //输入表
		PIMAGE_THUNK_DATA pThunkData; //IAT  INT
		ULONG ulSize;
		HMODULE hModule = 0;
		ULONG TargetFunAddr;
		PULONG lpAddr;
		char * szModName;
		BOOL result = FALSE;
		BOOL bRetn = FALSE;
		//457 //555
		hModule = LoadLibraryA(szModuleName);
		TargetFunAddr = (ULONG)GetProcAddress(hModule, szFuncName);  //获取目标函数的函数地址
	
		//获取待Hook模块输入表的起始地址
		pImportDescriptor = (PIMAGE_IMPORT_DESCRIPTOR) ImageDirectoryEntryToData(hModToHook, TRUE, IMAGE_DIRECTORY_ENTRY_IMPORT, &ulSize);
		
		//循环IID
		while (pImportDescriptor->FirstThunk) //PE加载后，仅FirstThunk
		{
			szModName = (char *) ((PBYTE)hModToHook + pImportDescriptor->Name); //??? 基址 + 偏移
			printf("[*]Cut Module Name: %s\n", szModName);
	
			if (_stricmp(szModName, szModuleName) != 0) //判断是不是待hook的函数所在的模块
			{
				printf("[*]Module Name does not match, search next...\n");
				pImportDescriptor++;    //不是则匹配下一个
				continue;
			}
	
			//模块名匹配，那么获取FirstThunk指向的地址表
			pThunkData = (PIMAGE_THUNK_DATA) ((BYTE *) hModToHook + pImportDescriptor->FirstThunk);
			
			//循环IAT
			while (pThunkData->u1.Function)  //IMAGE_THUNK_DATA   Function被输入函数的内存地址
			{
				lpAddr = (ULONG*) pThunkData;
	
				if ((*lpAddr) == TargetFunAddr)
				{
					printf("[*]Find target address!\n");
					
					//通常情况下，输入表所在的内存页都是只读的，因此要先修改内存页的属性为可写
					DWORD dwOldProtect;
					MEMORY_BASIC_INFORMATION mbi; //用来存放虚拟地址空间或虚拟页的属性或状态的结构
					VirtualQuery(lpAddr, &mbi, sizeof(mbi)); //查询地址空间中内存地址的信息
					bRetn = VirtualProtect(mbi.BaseAddress, mbi.RegionSize, PAGE_EXECUTE_READWRITE, &dwOldProtect); //修改页的属性为可写
	
					if (bRetn)
					{
						if (pThunkPointer != NULL)
						{
							*pThunkPointer = (PULONG)lpAddr;  //保存指针的位置
						}
						if (pOriginalFuncAddr != NULL)
						{
							*pOriginalFuncAddr = *lpAddr; //保存原始函数地址
						}
	
						//修改IAT中API的地址为Detour函数的地址
						*lpAddr = (ULONG)DetourFunc;	
						//恢复内存页的属性
						VirtualProtect(mbi.BaseAddress, mbi.RegionSize, dwOldProtect, 0);
						printf("[*]Hook ok.\n");
	
						result = TRUE;
					}
					
					break;
				}
	
				pThunkData++;
			}
	
			if (result)
			{
				break;
			}
			pImportDescriptor++;
		}
	
		FreeLibrary(hModule);
		return result;
	}
	
	int _tmain(int argc, _TCHAR* argv[])
	{
	
		HANDLE hmod = GetModuleHandle(NULL);//获取当前进程的句柄
		PULONG * pThunkPointer = NULL; //用于接受指向修改的位置的指针
		ULONG * pOriginalFuncAddr = NULL;//用于接受原始函数的地址
		
		////获得原函数的值
		HMODULE hModule = LoadLibraryA("User32.dll");
		OldMessageBox = (PFN_MessageBoxA)GetProcAddress(hModule, "MessageBoxA");  //获取目标函数的函数地址
	
		MessageBoxA(NULL, "zhongcheng", "Test", MB_OK);
		//Hook函数
		InstallModuleIATHook((HMODULE)hmod, "User32.dll", "MessageBoxA", My_MessageBoxA, pThunkPointer, pOriginalFuncAddr);
		
		MessageBoxA(NULL, "2", "Test", MB_OK);
	
		return 0;
	}


## 野蔷薇 -- 浪漫的爱情
