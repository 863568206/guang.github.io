---
layout:     post
title:      Dll注入--CreateRemoteThread
subtitle:   c++
date:       2020-06-18-Dll注入
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

## 安装程序

CreateRemoteThread法：就是目标进程中申请一块内存储存目标DLL路径，然后调用CreateRemoteThread创建一个线程函数是LoadLibrary，参数是存放目标DLL路径的内存指针。


	#include "stdafx.h"
	#include "windows.h"
	#include <iostream>
	#include "tlhelp32.h"
	#include <string>
	
	using namespace std;
	
	DWORD GetProcessPID(wstring strFileName)
	{
		HANDLE hProcessSnap = NULL;
		DWORD dwordRet = 0;
		do 
		{
			hProcessSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
			if (hProcessSnap == INVALID_HANDLE_VALUE)
			{
				break;
			}
	
			PROCESSENTRY32 pe32 = { 0 };
			pe32.dwSize = sizeof(PROCESSENTRY32);
			if (Process32First(hProcessSnap, &pe32))
			{
				do 
				{
					if (_wcsicmp(pe32.szExeFile, strFileName.c_str()) == 0)
					{
						dwordRet = pe32.th32ProcessID;
						break;
					}
				} while (Process32Next(hProcessSnap, &pe32));
			}
	
		} while (FALSE);
	
		if (hProcessSnap != NULL)
		{
			CloseHandle(hProcessSnap);
		}
		return dwordRet;
	}
	
	int _tmain(int argc, _TCHAR* argv[])
	{
		wstring wstrLibFile = L"C:\\Users\\Administrator\\Desktop\\HookDllNew.dll"; //绝对路径或相对路径
		//获取 notepad 的 PID
		DWORD dwProcessId = 0;
		dwProcessId = GetProcessPID(L"Taskmgr.exe");
		
		printf("%d", dwProcessId);
	
		//获取进程句柄
		HANDLE hProcess = OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_CREATE_THREAD | PROCESS_VM_OPERATION | PROCESS_VM_WRITE, FALSE, dwProcessId);
		if (hProcess == INVALID_HANDLE_VALUE)
		{
			printf("%s", "获取进程句柄失败");
		}
		
		//获取Dll的路径大小，并开辟内存，写入
		DWORD dwSize = (lstrlenW(wstrLibFile.c_str()) + 1) * sizeof(wchar_t);
		LPVOID pszLibFileRemote = (PWSTR)VirtualAllocEx(hProcess, NULL, dwSize, MEM_COMMIT, PAGE_READWRITE);
		if (pszLibFileRemote == NULL)
		{
			printf("%s", "分配内存失败");
		}
	
		DWORD n = WriteProcessMemory(hProcess, pszLibFileRemote, (PVOID)wstrLibFile.c_str(), dwSize, NULL);
		if (n == 0)
		{
			printf("%s", "写入内存失败");
		}
	
		//获取LoadLibrary函数的地址
		PTHREAD_START_ROUTINE pfnThreadRtn = (PTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle(TEXT("Kernel32")), "LoadLibraryW");
	
		HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, pfnThreadRtn, pszLibFileRemote, 0, NULL);
		if (hThread == INVALID_HANDLE_VALUE)
		{
			printf("%s", "创建线程失败");
		}
	
		return 0;
	}

## Dll程序

借鉴：[DLL程序入口DllMain详解](https://www.jianshu.com/p/b184b46a2d67)

直接建立Dll项目会包含一个 DllMain 的主函数，在其中添加我们的代码。

其中 ul_reason_for_call 指明了Dll被调用的原因。

**DLL_PROCESS_ATTACH**：

当DLL被进程 <<第一次>> 调用时，导致DllMain函数被调用，同时ul_reason_for_call的值为DLL_PROCESS_ATTACH，如果同一个进程后来再次调用此DLL时，操作系统只会增加DLL的使用次数，不会再用DLL_PROCESS_ATTACH调用DLL的DllMain函数。

**DLL_PROCESS_DETACH**：

当DLL被从进程的地址空间解除映射时，系统调用了它的DllMain，传递的ul_reason_for_call值是DLL_PROCESS_DETACH。
★如果进程的终结是因为调用了TerminateProcess，系统就不会用DLL_PROCESS_DETACH来调用DLL的DllMain函数。这就意味着DLL在进程结束前没有机会执行任何清理工作。

**DLL_THREAD_ATTACH**：

当进程创建一线程时，系统查看当前映射到进程地址空间中的所有DLL文件映像，并用值DLL_THREAD_ATTACH调用DLL的DllMain函数。 新创建的线程负责执行这次的DLL的DllMain函数，只有当所有的DLL都处理完这一通知后，系统才允许线程开始执行它的线程函数。

**DLL_THREAD_DETACH**：

如果线程调用了ExitThread来结束线程（线程函数返回时，系统也会自动调用ExitThread），系统查看当前映射到进程空间中的所有DLL文件映像，并用DLL_THREAD_DETACH来调用DllMain函数，通知所有的DLL去执行线程级的清理工作。

**★注意：如果线程的结束是因为系统中的一个线程调用了TerminateThread，系统就不会用值DLL_THREAD_DETACH来调用所有DLL的DllMain函数。**



	BOOL APIENTRY DllMain( HMODULE hModule,           //hModule参数：指向DLL本身的实例句柄；
	                       DWORD  ul_reason_for_call, //ul_reason_for_call参数：指明了DLL被调用的原因值：
	                       LPVOID lpReserved		  
						 )
	{
		switch (ul_reason_for_call)
		{
		case DLL_PROCESS_ATTACH:
			 MessageBoxW(NULL, L"注入成功，打开", L"Dll测试", 0);
			 break;
			
		case DLL_PROCESS_DETACH:
			 MessageBoxW(NULL, L"注入成功，关闭", L"Dll测试", 0);
	
			break;
		}
		return TRUE;
	}

## 黄蔷薇 -- 永恒的微笑