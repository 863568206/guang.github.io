---
layout:     post
title:      SSTD HOOK
subtitle:   c++
date:       2021-5-17
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - Driver
---

# SSDT

SSDT就是一个导出的全局变量，这个全局变量里面包含了系统服务表。windows内核导出了一个全局变量**KeServiceDescriptorTable** ，**KeServiceDescriptorTable**只提供了Ntoskrl.exe的系统服务表，没有提供 Win32K.sys的。同时在64位操作系统中，为了保护SSDT操作系统不再导出SSDT了。

## 动态获取SSDT、函数编号等

[动态获取SSDT、函数编号](http://yimitumi.com/2021/05/13/%E8%8E%B7%E5%8F%96SSTD%E5%87%BD%E6%95%B0%E5%90%8D-%E8%B0%83%E7%94%A8%E7%BC%96%E5%8F%B7-%E5%87%BD%E6%95%B0%E5%9C%B0%E5%9D%80/)

## 计算函数偏移

在64位SSDT中存放的是函数的偏移地址，而在32位中存放的直接是函数的地址。所以在64位中要计算一下函数的偏移：

	#ifdef _WIN64
		ULONGLONG pFunctionOffset = NULL; //函数偏移
	
		//计算函数偏移，在X64中，SSDT保存的是函数的偏移 ：真实函数地址 = ssdt(基址) + ssdt[nIndex]>>4 , 32位则不用
	
		ULONGLONG pMiTerminateProcess = g_pMiTerminateProcess << 4;
		pFunctionOffset = pMiTerminateProcess - (ULONG)g_SsdtAdd;
	
	#else
		ULONG pFunctionOffset = NULL; //函数偏移
		
		//计算函数偏移，在X64中，SSDT保存的是函数的偏移 ：真实函数地址 = ssdt(基址) + ssdt[nIndex]>>4 , 32位则不用
		pFunctionOffset = g_pMiTerminateProcess;
	#endif

## 替换函数地址

Hook SSDT就是修改系统服务表中存放函数的地址，将函数地址改为我们自己的函数。因为保护模式的存在系统服务表的物理页为只读的，所以我们要绕过页机制，可以通过CR0将页修改为只读，也可以通过[MDL](https://blog.csdn.net/Simon798/article/details/106946179/)绕过页保护。

CR0:

	//关闭页面保护  
	KIRQL WPOFFx64()
	{
		//例程将硬件优先级提高到IRQL = DISPATCH_LEVEL，从而屏蔽当前处理器上等效或更低IRQL的中断，并保存调用时IRQL
		KIRQL irql = KeRaiseIrqlToDpcLevel();
	
		//读取CR0寄存器并返回其值。
		UINT64 cr0 = __readcr0();
	
		//PG: 当设置该位时即开启了分页机制,PG在31位
		cr0 &= 0xfffffffffffeffff;
	
		__writecr0(cr0);
		_disable();
		return irql;
	}
	
	//开启页面保护  
	void WPONx64(KIRQL irql)
	{
		UINT64 cr0 = __readcr0();
		cr0 |= 0x10000;
		_enable();
		__writecr0(cr0);
		KeLowerIrql(irql); //还原IRQL
	}


MDL:

	BOOLEAN MDLModAddress(PVOID pModAddress, ULONGLONG pFunctionOffset, SIZE_T isize)
	{
		PVOID pNewAddress = NULL;
		PMDL pMdl = NULL;
	
		// 创建 MDL
		pMdl = MmCreateMdl(NULL, pModAddress, isize);  //32和64位数 不一样
		if (NULL == pMdl)
		{
			DbgPrint("MmCreateMdl Error!\n");
			return FALSE;
		}
	
		// 更新 MDL 对物理内存的描述
		MmBuildMdlForNonPagedPool(pMdl);
	
		pNewAddress = MmMapLockedPages(pMdl, KernelMode);
		if (NULL == pNewAddress)
		{
			IoFreeMdl(pMdl);
			DbgPrint("MmMapLockedPages Error!\n");
			return FALSE;
		}
	
		// 写入原函数地址
		RtlCopyMemory(pNewAddress, &pFunctionOffset, isize);
	
		// 释放
		MmUnmapLockedPages(pNewAddress, pMdl);
		IoFreeMdl(pMdl);
	
		return TRUE;
	}

## Hook NtTerminateProcess实现禁止关闭进程

	#include <ntddk.h>
	#include <WinDef.h>
	#include <aux_klib.h>
	
	#pragma comment(lib,"aux_klib.lib")  
	
	typedef struct _KSYSTEM_SERVICE_TABLE
	{
		PULONG ServiceTableBase;         //函数地址表基地址
		PULONG ServicrCounterTableBase;  //SSDT函数被调用的次数
		ULONG  NumberOfService;          //函数的个数
		PULONG ParamTableBase;           //函数参数表基地址
	
	} KSYSTEM_SERVICE_TABLE, * PKSYSTEM_SERVICE_TABLE;
	
	
	typedef struct _KSERVICE_TABLE_DESCRIPTOR
	{
	
		KSYSTEM_SERVICE_TABLE ntoskrnl;   //Ntoskrl.exe 的函数
		KSYSTEM_SERVICE_TABLE win32k;     //Win32K.sys 的函数
		KSYSTEM_SERVICE_TABLE notUsed1;
		KSYSTEM_SERVICE_TABLE notUsed2;
	
	} KSERVICE_TABLE_DESCRIPTOR, * PKSERVICE_TABLE_DESCRIPTOR;
	
	PULONG g_SsdtAdd = NULL;
	
	#ifdef _WIN64
	
	#else
		extern PKSERVICE_TABLE_DESCRIPTOR KeServiceDescriptorTable;    //只有在32位才能导出
	#endif
	
	
	//MiTerminateProcess的函数定义
	typedef NTSTATUS(__stdcall* PMiTerminateProcess)(HANDLE ProcessHandle, NTSTATUS ExitStatus);
	
	ULONGLONG g_pMiTerminateProcess = 0;
	
	ULONGLONG g_OldNtTerminateProcessAdd = 0;
	
	ULONGLONG g_ulSSDTFunctionIndex = 0;
	
	//动态定位KeServiceDescriptorTable的方法  
	ULONGLONG GetKeServiceDescriptorTable64()
	{
		UNICODE_STRING strStrnicmp;
		RtlInitUnicodeString(&strStrnicmp, L"_strnicmp");
		ULONGLONG ulongStrnicmp = (ULONGLONG)MmGetSystemRoutineAddress(&strStrnicmp);
		//KdPrint(("_strnicmp %x\n", ulongStrnicmp));
	
		char KiSystemServiceStart_pattern[] = "\x8B\xF8\xC1\xEF\x07\x83\xE7\x20\x25\xFF\x0F\x00\x00";   //特征码  
		ULONGLONG CodeScanStart = ulongStrnicmp;
		ULONGLONG CodeScanEnd = (ULONGLONG)&KdDebuggerNotPresent;
	
		ULONGLONG i, tbl_address, b;
		for (i = 0; i < CodeScanEnd - CodeScanStart; i++)
		{
			if (!memcmp((char*)(ULONGLONG)CodeScanStart + i, (char*)KiSystemServiceStart_pattern, 13))
			{
				for (b = 0; b < 50; b++)
				{
					tbl_address = ((ULONGLONG)CodeScanStart + i + b);
					if (*(USHORT*)((ULONGLONG)tbl_address) == (USHORT)0x8d4c)
						return ((LONGLONG)tbl_address + 7) + *(LONG*)(tbl_address + 3);
				}
			}
		}
		return 0;
	}
	
	//根据KeServiceDescriptorTable找到SSDT基址  
	PULONG GetSSDTBaseAddress()
	{
		PULONG addr = NULL;
		PKSYSTEM_SERVICE_TABLE ssdt = NULL;
	
	#ifdef _WIN64
		ssdt = (PKSYSTEM_SERVICE_TABLE)GetKeServiceDescriptorTable64();
		addr = (PULONG)(ssdt->ServiceTableBase);
	#else
		ssdt = &KeServiceDescriptorTable->ntoskrnl;    //只有在32位才能导出
		addr = (PULONG)ssdt->ServiceTableBase;    //32位中ServiceTableBase中保存的值才是地址表的基址
	#endif
	
		return addr;
	}
	
	
	//根据标号找到SSDT表中函数的地址  
	ULONGLONG GetFuncAddr(ULONG id)
	{
		LONG dwtmp = 0;
		ULONGLONG addr = 0;
		PULONG stb = NULL;
		stb = GetSSDTBaseAddress();
	
	#ifdef _WIN64
		dwtmp = stb[id];
		dwtmp = dwtmp >> 4;
		addr = (LONGLONG)dwtmp + (ULONGLONG)stb;
	#else
		addr = stb[id];
	#endif
	
		//DbgPrint("SSDT TABLE BASEADDRESS:%11x", addr);
		g_SsdtAdd = stb; //写入到全局变量
		return addr;
	}
	
	
	//查找函数
	NTSTATUS DllFileMap(UNICODE_STRING ustrDllFileName, HANDLE* phFile, HANDLE* phSection, PVOID* ppBaseAddress)
	{
		NTSTATUS status = STATUS_SUCCESS;
		HANDLE hFile = NULL;
		HANDLE hSection = NULL;
		OBJECT_ATTRIBUTES objectAttributes = { 0 };
		IO_STATUS_BLOCK iosb = { 0 };
		PVOID pBaseAddress = NULL;
		SIZE_T viewSize = 0;
	
		// 打开 DLL 文件, 并获取文件句柄
		InitializeObjectAttributes(&objectAttributes, &ustrDllFileName, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);
		status = ZwOpenFile(&hFile, GENERIC_READ, &objectAttributes, &iosb, FILE_SHARE_READ, FILE_SYNCHRONOUS_IO_NONALERT);
		if (!NT_SUCCESS(status))
		{
			KdPrint(("ZwOpenFile Error! [error code: 0x%X]", status));
			return status;
		}
	
		// 创建一个节对象, 以 PE 结构中的 SectionALignment 大小对齐映射文件
		status = ZwCreateSection(&hSection, SECTION_MAP_READ | SECTION_MAP_WRITE, NULL, 0, PAGE_READWRITE, 0x1000000, hFile);
		if (!NT_SUCCESS(status))
		{
			ZwClose(hFile);
			KdPrint(("ZwCreateSection Error! [error code: 0x%X]", status));
			return status;
		}
	
		// 映射到内存
		status = ZwMapViewOfSection(hSection, NtCurrentProcess(), &pBaseAddress, 0, 1024, 0, &viewSize, ViewShare, MEM_TOP_DOWN, PAGE_READWRITE);
		if (!NT_SUCCESS(status))
		{
			ZwClose(hSection);
			ZwClose(hFile);
			KdPrint(("ZwMapViewOfSection Error! [error code: 0x%X]", status));
			return status;
		}
	
		// 返回数据
		*phFile = hFile;
		*phSection = hSection;
		*ppBaseAddress = pBaseAddress;
	
		return status;
	}
	
	//映射文件
	ULONGLONG  GetIndexFromExportTable(PVOID pBaseAddress, PCHAR pszFunctionName)
	{
		ULONGLONG  ulFunctionIndexAdd = 0; 	// Dos Header
	
		PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)pBaseAddress;  // NT Header
	
		PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)((PUCHAR)pDosHeader + pDosHeader->e_lfanew); // Export Table
	
		PIMAGE_EXPORT_DIRECTORY pExportTable = (PIMAGE_EXPORT_DIRECTORY)((PUCHAR)pDosHeader + pNtHeaders->OptionalHeader.DataDirectory[0].VirtualAddress);  // 有名称的导出函数个数
		ULONG ulNumberOfNames = pExportTable->NumberOfNames;
	
		// 导出函数名称地址表
		PULONG lpNameArray = (PULONG)((PUCHAR)pDosHeader + pExportTable->AddressOfNames);
		PCHAR lpName = NULL;
	
		// 开始遍历导出表
		for (ULONG i = 0; i < ulNumberOfNames; i++)
		{
			lpName = (PCHAR)((PUCHAR)pDosHeader + lpNameArray[i]);
	
			// 判断是否查找的函数
			if (0 == _strnicmp(pszFunctionName, lpName, strlen(pszFunctionName)))
			{
				// 获取导出函数地址
				USHORT uHint = *(USHORT*)((PUCHAR)pDosHeader + pExportTable->AddressOfNameOrdinals + 2 * i);
				ULONG ulFuncAddr = *(PULONG)((PUCHAR)pDosHeader + pExportTable->AddressOfFunctions + 4 * uHint);
				PVOID lpFuncAddr = (PVOID)((PUCHAR)pDosHeader + ulFuncAddr);
	
				
				// 获取 SSDT 函数 Index
	#ifdef _WIN64
				ulFunctionIndexAdd = *(ULONG*)((PUCHAR)lpFuncAddr + 4);
	#else
				ulFunctionIndexAdd = *(ULONG*)((PUCHAR)lpFuncAddr + 1);
	#endif
				
				DbgPrint("Function = %s, %d, %x\n", lpName, ulFunctionIndexAdd, lpFuncAddr);
			
				break;
			}
		}
	
		return ulFunctionIndexAdd;
	}
	
	//获取函数名对应编号
	ULONGLONG GetSSDTFunctionIndexAndAdd(UNICODE_STRING ustrDllFileName, PCHAR pszFunctionName)
	{
		ULONGLONG  ulFunctionIndexAdd = 0;
		NTSTATUS status = STATUS_SUCCESS;
		HANDLE hFile = NULL;
		HANDLE hSection = NULL;
		PVOID pBaseAddress = NULL;
	
		// 内存映射文件
		status = DllFileMap(ustrDllFileName, &hFile, &hSection, &pBaseAddress);
		if (!NT_SUCCESS(status))
		{
			KdPrint(("DllFileMap Error!\n"));
			return ulFunctionIndexAdd;
		}
	
		// 根据导出表获取导出函数地址, 从而获取 SSDT 函数索引号
		ulFunctionIndexAdd = GetIndexFromExportTable(pBaseAddress, pszFunctionName);
	
		// 释放
		ZwUnmapViewOfSection(NtCurrentProcess(), pBaseAddress);
		ZwClose(hSection);
		ZwClose(hFile);
	
		return ulFunctionIndexAdd;
	}
	
	BOOLEAN MDLModAddress(PVOID pModAddress, ULONGLONG pFunctionOffset, SIZE_T isize)
	{
		PVOID pNewAddress = NULL;
		PMDL pMdl = NULL;
	
		// 创建 MDL
		pMdl = MmCreateMdl(NULL, pModAddress, isize);  //32和64位数 不一样
		if (NULL == pMdl)
		{
			DbgPrint("MmCreateMdl Error!\n");
			return FALSE;
		}
	
		// 更新 MDL 对物理内存的描述
		MmBuildMdlForNonPagedPool(pMdl);
	
		pNewAddress = MmMapLockedPages(pMdl, KernelMode);
		if (NULL == pNewAddress)
		{
			IoFreeMdl(pMdl);
			DbgPrint("MmMapLockedPages Error!\n");
			return FALSE;
		}
	
		// 写入原函数地址
		RtlCopyMemory(pNewAddress, &pFunctionOffset, isize);
	
		// 释放
		MmUnmapLockedPages(pNewAddress, pMdl);
		IoFreeMdl(pMdl);
	
		return TRUE;
	}
	
	BOOLEAN LoadSSDTHookX64(ULONGLONG ulSSDTFunctionIndex)
	{
		ULONGLONG pFunctionOffset = NULL; //函数偏移
	
		//计算函数偏移，在X64中，SSDT保存的是函数的偏移 ：真实函数地址 = ssdt(基址) + ssdt[nIndex]>>4 , 32位则不用
	
		ULONGLONG pMiTerminateProcess = g_pMiTerminateProcess << 4;
		pFunctionOffset = pMiTerminateProcess - (ULONG)g_SsdtAdd;
	
		//64位下CR0可能会失效，使用MDL方式
		MDLModAddress(&g_SsdtAdd[ulSSDTFunctionIndex], pFunctionOffset, sizeof(pFunctionOffset));
	
		return TRUE;
	}
	
	BOOLEAN LoadSSDTHookX32(ULONGLONG ulSSDTFunctionIndex)
	{
		ULONG pFunctionOffset = NULL; //函数偏移
		
		//计算函数偏移，在X64中，SSDT保存的是函数的偏移 ：真实函数地址 = ssdt(基址) + ssdt[nIndex]>>4 , 32位则不用
		pFunctionOffset = g_pMiTerminateProcess;
	
		//32位下可以通过CR0，但是64位下CR0可能会失效，使用MDL方式
		MDLModAddress(&g_SsdtAdd[ulSSDTFunctionIndex], pFunctionOffset, sizeof(pFunctionOffset));
	
		return TRUE;
	}
	
	BOOLEAN LoadSSDTHookX(ULONGLONG ulSSDTFunctionIndex)
	{
	#ifdef _WIN64
		
		MDLModAddress(&g_SsdtAdd[ulSSDTFunctionIndex], g_OldNtTerminateProcessAdd, sizeof(g_OldNtTerminateProcessAdd));
	
	#else
		
		ULONG OldNtTerminateProcessAdd = (ULONG)g_OldNtTerminateProcessAdd;
		MDLModAddress(&g_SsdtAdd[ulSSDTFunctionIndex], OldNtTerminateProcessAdd, sizeof(OldNtTerminateProcessAdd));
	
	#endif
	
		
		return TRUE;
	}
	
	//新函数
	NTSTATUS New_NtTerminateProcess(HANDLE ProcessHandle, NTSTATUS ExitStatus)
	{
		NTSTATUS stRet = STATUS_PROCESS_IS_TERMINATING;
	
		//将句柄转换为进程名
		PEPROCESS pProcessObj = NULL;
		ObReferenceObjectByHandle(ProcessHandle, NULL, *PsProcessType, KernelMode, (PVOID)&pProcessObj, NULL);
		
		//函数名在 PEPROCESS 的 0x16c 处 ：  char ImageFileName[15];  
		char* ProcessName = (char* )((ULONG)pProcessObj + 0x16c);
		DbgPrint("ProcessName = %s\n", ProcessName);
	
		PCHAR pForbidName = "notepad.exe";
	
		if (0 != _strnicmp(pForbidName, ProcessName, strlen(pForbidName)))
		{
			//调用原来函数
			PMiTerminateProcess oldNtTerminateProcess = (PMiTerminateProcess)g_OldNtTerminateProcessAdd;
			stRet = oldNtTerminateProcess(ProcessHandle, ExitStatus);
		}
	
		return stRet;
	}
	
	
	//卸载
	VOID DriverUpLoad(PDRIVER_OBJECT pdriver)
	{
		DbgPrint("卸载了\n");
		
	
		LoadSSDTHookX(g_ulSSDTFunctionIndex);
	
	}
	
	NTSTATUS DriverEntry(PDRIVER_OBJECT pdriver, PUNICODE_STRING pReg) //返回一个地址、返回驱动被注册到注册表的某个地方
	{
		DbgPrint("pdriver = %wZ, , %x\n", pReg, pdriver);
		pdriver->DriverUnload = DriverUpLoad;
	
		//获取SSDT 函数编号
		UNICODE_STRING ustrDllFileName;
		RtlInitUnicodeString(&ustrDllFileName, L"\\??\\C:\\Windows\\System32\\ntdll.dll");
		g_ulSSDTFunctionIndex = GetSSDTFunctionIndexAndAdd(ustrDllFileName, "NtTerminateProcess"); //要HOOk的函数名
	
		//根据编号获取函数地址
		g_OldNtTerminateProcessAdd = GetFuncAddr(g_ulSSDTFunctionIndex);
		DbgPrint("NtTerminateProcess = %x", g_OldNtTerminateProcessAdd);
	
		if (g_SsdtAdd == NULL)
		{
			DbgPrint("Not SSDT Address!");
			return STATUS_SUCCESS;
		}
	
		//新函数地址
		g_pMiTerminateProcess = New_NtTerminateProcess;
	
		DbgPrint("New_NtTerminateProcess = %11x\n", g_pMiTerminateProcess);
	
	#ifdef _WIN64
		LoadSSDTHookX64(g_ulSSDTFunctionIndex);
	#else
		LoadSSDTHookX32(g_ulSSDTFunctionIndex);
	#endif
	
	
		return STATUS_SUCCESS;
	}

## 薰衣草--等待爱情

