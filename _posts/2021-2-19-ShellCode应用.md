---
layout:     post
title:      ShellCode应用
subtitle:   c++
date:       2021-2-19
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

# shellCold

1.ShellCode编写原则

&nbsp;  1) 不能有全局变量

&nbsp;  2) 不能使用常量字符串

	char szBuffer[] = "ShellCode"; //会使用常量区，所以不可用
	
	//写成
	char szBuffer[] = {'S', 'h', 'e', 'l', '\0'}; //这个使用堆栈

	char szBuffer[] = {'K', '0', 'e', '0', '\0'}; //UNICODE格式，高字节为零，占2个字符，一般函数名为UinCode格式


&nbsp;  3) 不能使用系统调用
	
	//也会依赖导入表，不可用
	HMODULE hMoudle = LoadLibrary("user32.dll");
	GetProcAddress(hMoudle, "MessageBoxA");

可以获取	LoadLibrary 和 GetProcAddress的地址来使用。


&nbsp;  4) 不能嵌套调用其他函数

原因：不依赖环境，放到任何地方都可以执行的机器码。

将写好的ShellCode的机器码拿出他是一段放到哪里都可以执行的代码，不依赖任何环境。

## 例

	#include <iostream>
	#include <windows.h>
	
	typedef struct _UNICODE_STRING
	{
	    USHORT Length;
	    USHORT MaximumLength;
	    PWSTR  Buffer;
	}UNICODE_STRING, *PUNICODE_STRING;
	
	typedef struct _PEB_LDR_DATA
	{
		ULONG Length; // +0x00
		BOOLEAN Initialized; // +0x04
		PVOID SsHandle; // +0x08
		LIST_ENTRY InLoadOrderModuleList; // +0x0c
		LIST_ENTRY InMemoryOrderModuleList; // +0x14
		LIST_ENTRY InInitializationOrderModuleList;// +0x1c
	    VOID* EntryInProgress;
	} PEB_LDR_DATA, *PPEB_LDR_DATA; // +0x24
	
	typedef struct _LDR_DATA_TABLE_ENTRY
	{
		LIST_ENTRY InLoadOrderLinks;
		LIST_ENTRY InMemoryOrderLinks;
		LIST_ENTRY InInitializationOrderLinks;
		PVOID DllBase;
		PVOID EntryPoint;
		ULONG SizeOfImage;
		UNICODE_STRING FullDllName;
		UNICODE_STRING BaseDllName;
		ULONG Flags;
		WORD LoadCount;
		WORD TlsIndex;
		union
		{
			LIST_ENTRY HashLinks;
			struct
			{
				PVOID SectionPointer;
				ULONG CheckSum;
			};
		};
		union
		{
			ULONG TimeDateStamp;
			PVOID LoadedImports;
		};
		_ACTIVATION_CONTEXT* EntryPointActivationContext;
		PVOID PatchInformation;
		LIST_ENTRY ForwarderLinks;
		LIST_ENTRY ServiceTagLinks;
		LIST_ENTRY StaticLinks;
	} LDR_DATA_TABLE_ENTRY, * PLDR_DATA_TABLE_ENTRY;
	
	typedef DWORD (WINAPI* PGetProcAddress) (_In_ HMODULE hModule, _In_ LPCSTR lpProcName);
	typedef HMODULE (WINAPI* PLoadLibraryA) (_In_ LPCSTR lpLibFileName);
	typedef int (WINAPI* PMessageBoxA) (_In_opt_ HWND hWnd, _In_opt_ LPCSTR lpText ,_In_opt_ LPCSTR lpCaption, _In_ UINT uType);
	
	PMessageBoxA MessageBoxFun = NULL;
	PGetProcAddress GetProAddrNameFun = NULL;
	PLoadLibraryA LoadLibraryAFun = NULL;
	
	//获取 LoadLibrary 和 GetProcAddress的函数地址 
	void GetMoudleAddress()
	{
		//LDR链
		PLDR_DATA_TABLE_ENTRY pLdr = NULL;
		PLDR_DATA_TABLE_ENTRY pBegLdr = NULL;
		DWORD dwKernel32Addr = 0;
	
	    //不能使用常量字符串, UniCode格式
	    //Kernel32.dll
	    char szKernel32Name[] = {'K', 0, 'E', 0, 'R', 0, 'N', 0, 'E', 0,  'L', 0,  '3', 0, '2', 0, '.', 0, 'D', 0, 'L', 0, 'L', 0, 0, 0};
		//wchar_t szKernel32Name[] = { 'K', 'E', 'R',  'N', 'E', 'L', '3', '2', '.', 'D', 'L', 'L', 0};
	
	    //GetProcAddress
	    char szGetProAddrName[] = {'G', 'e', 't', 'P', 'r', 'o', 'c', 'A', 'd', 'd', 'r', 'e', 's', 's', 0};
	
	    //LoadLibrary
	    char szLoadLibraryName[] = {'L', 'o', 'a', 'd', 'L', 'i', 'b', 'r', 'a', 'r', 'y', 'A', 0};
	
	    //获取链表 TEB -> PEB -> _PEB_LDR_DATA -> _LDR_DATA_TABLE_ENTRY
		_asm
		{
			mov eax, fs:[0x30]      //PEB
			mov eax, [eax + 0x0c]   //PEB -> Ldr
			add eax, 0x0c           //LDR_DATA_TABLE_ENTRY -> InLoadOrderModuleList
			mov pBegLdr, eax        //获取 InLoadOrderModuleList 的地址
			mov eax, [eax]          
			mov pLdr, eax           //获取 InLoadOrderModuleList 的成员指针
		}
	
		//遍历处Kernerl32.dll的地址
		while (pLdr != pBegLdr)
		{
			WORD* pLast = (WORD*)pLdr->BaseDllName.Buffer;
			WORD* pFirst = (WORD*)szKernel32Name;
	
			//wprintf(L"%s \n", pLdr->BaseDllName.Buffer);
	
			//比较字符串
			while (*pFirst != 0 && *pFirst == *pLast)
			{
				pFirst++;
				pLast++;
			}
	
			if (*pFirst == *pLast)
			{
				dwKernel32Addr = (DWORD)pLdr->DllBase;
				break;
			}
			
			pLdr = (PLDR_DATA_TABLE_ENTRY)pLdr->InLoadOrderLinks.Flink;
		}
	
		//遍历PE导出表获取GetProcAddress和LoadLibrary的地址
		PIMAGE_DOS_HEADER pDos = NULL;
		PIMAGE_NT_HEADERS pNt = NULL;
		PIMAGE_FILE_HEADER pPe = NULL;
		PIMAGE_OPTIONAL_HEADER32 pOptionalPe = NULL;
	
		//获取DOS头
		pDos = (PIMAGE_DOS_HEADER)dwKernel32Addr;
		pNt = (PIMAGE_NT_HEADERS)((DWORD)dwKernel32Addr + pDos->e_lfanew);
		pPe = (PIMAGE_FILE_HEADER)((DWORD)pNt + 4);     //标准PE头
		pOptionalPe = (PIMAGE_OPTIONAL_HEADER32)((DWORD)pPe + IMAGE_SIZEOF_FILE_HEADER); //可选pe头
	
		//导出表
		PIMAGE_DATA_DIRECTORY pExportFunctionData = (PIMAGE_DATA_DIRECTORY)pOptionalPe->DataDirectory;
		PIMAGE_EXPORT_DIRECTORY pExportFunctionInfo = (PIMAGE_EXPORT_DIRECTORY)((DWORD)dwKernel32Addr + pExportFunctionData->VirtualAddress);
		
		DWORD* pAddofNames = (DWORD*)((DWORD)dwKernel32Addr + pExportFunctionInfo->AddressOfNames);
		WORD* pAddofOrd = (WORD*)((DWORD)dwKernel32Addr + pExportFunctionInfo->AddressOfNameOrdinals);
		DWORD* pAddofFun = (DWORD*)((DWORD)dwKernel32Addr + pExportFunctionInfo->AddressOfFunctions);
	
		DWORD dwCnt = 0;
		char* pFinded = NULL;
		char* pSrc = szGetProAddrName;
	
		for (; dwCnt < pExportFunctionInfo->NumberOfNames; dwCnt++)
		{
			pFinded = (char*)((DWORD)dwKernel32Addr + pAddofNames[dwCnt]);
	
			//printf("%s \n", pFinded);
	
			//判断名字
			while (*pSrc && *pSrc == *pFinded)
			{
				pSrc++;
				pFinded++;
			}
	
			if (*pSrc == *pFinded)
			{
				GetProAddrNameFun = (PGetProcAddress)((DWORD)dwKernel32Addr + pAddofFun[pAddofOrd[dwCnt]]);
				break;
			}
	
			pSrc = szGetProAddrName;
		}
	
		LoadLibraryAFun = (PLoadLibraryA)GetProAddrNameFun((HMODULE)dwKernel32Addr, szLoadLibraryName);
	
		return;
	}
	
	int main()
	{
		GetMoudleAddress();
		
		//User32.dll
		char szUser32[] = { 'U', 's', 'e', 'r', '3', '2', '.', 'd', 'l', 'l', 0};
		//MessageBoxA
		char szMessageBoxA[] = {'M', 'e', 's', 's', 'a', 'g', 'e', 'B', 'o', 'x', 'A', 0};
	
		HMODULE hMessageBoxA = LoadLibraryAFun(szUser32);
		MessageBoxFun = (PMessageBoxA)GetProAddrNameFun(hMessageBoxA, szMessageBoxA);
	
		char szText[] = { 'T', 'e', 'x', 't', 0};
		char szTextt[] = { 'T', 'e', 'x', 't', 0 };
	
		MessageBoxFun(NULL, szText, szTextt, MB_OK);
	
		return 0;
	}

## 蓝色妖姬 -- 相守
