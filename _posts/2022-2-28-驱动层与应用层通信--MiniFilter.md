---
layout:     post
title:      驱动层与应用层通信--MiniFilter
subtitle:   c++
date:       2022-2-28
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - Driver
---

# 驱动层与应用层通信--MiniFilter
	
驱动层与应用层通信可以使用 DeviceIoControl 函数发送消息，但是使用 DeviceIoControl 比较麻烦，并且想让驱动层直接发消息到应用层更加麻烦，所以这里使用了MiniFilter。

在这里我依旧使用 DeviceIoControl 发送消息到驱动层（[参考：DeviceIoControl发送消息](http://yimitumi.com/2022/02/28/%E9%A9%B1%E5%8A%A8%E5%B1%82%E4%B8%8E%E5%BA%94%E7%94%A8%E5%B1%82%E9%80%9A%E4%BF%A1-DeviceIoControl/)），当驱动层收到 DeviceIoControl 的消息后，就可以通过 MiniFilter 循环发送数据到应用层了。

	
## 在R0和R3之间用于通信的数据

这里的数据因为是驱动程序和应用层程序都使用的，所以加个了全局的 .h 文件。
		
		//MiniFilter 名
		#define PortName  L"\\CommentDriveTootPort"
		
		//驱动发消息到客户端使用
		static const int NET_MINIFILTER_MESSAGE_DRIVE_INFO = 1;
		
		typedef struct _NET_MESSAGE_INFO
		{
			LONGLONG ulShareUserAdd;   //共享到R3的地址
			DWORD dwDataType;          //保存一个消息类型，比如上面的
			DWORD dwStart;             //保存一个开始结束的标志位
		
		} NET_MESSAGE_INFO, * PNET_MESSAGE_INFO;
	
	这里 ulShareUserAdd 保存的是一个R0创建的共享地址的指针（参考：[共享内存](http://yimitumi.com/2022/02/15/%E5%88%9B%E5%BB%BA%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98/)）。 
	
## 驱动层代码：创建一个MiniFilter

在使用 MiniFilter 时要将 FltLib.lib 添加到，项目的属性->链接器->输入->附加依赖项目中。


**MiniFilter 的头文件（MiniFilterSeting.h）：**
	
		#pragma once
		#pragma once
		#define _HEADER_HEAD_FILE
		#define _HEADER_HEAD_FILE
		#pragma once
		
		#include <fltKernel.h>
		#include <Ntstrsafe.h>
		//#include "PublicDefinition.h"
		
		#ifndef MAX_PATH
		#define MAX_PATH	260
		#endif
		
		
		PFLT_FILTER gFilterHandle;
		PFLT_PORT g_ServerPort;
		PFLT_PORT g_ClientPort;
		
		//创建一个微过滤器
		NTSTATUS CreatMiniFilter(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg);
		
		//注册MiniFilter
		NTSTATUS RegisterMiniFilter(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg);
		
		//设置注册表键值
		NTSTATUS SetRegeditValueKey(PUNICODE_STRING pRegPath, PUNICODE_STRING pValueName, ULONG Type, wchar_t ValueData[MAX_PATH]);
		
		////建立链接回调函数
		NTSTATUS ConnectNotifyCallback(IN PFLT_PORT ClientPort, IN PVOID ServerPortCookie, IN PVOID ConnectionContext, IN ULONG SizeOfContext, OUT PVOID* ConnectionPortCookie);
		
		//断开链接回调函数
		VOID DisconnectNotifyCallback(_In_opt_ PVOID ConnectionCookie);
		
		//发消息给R3
		NTSTATUS SendMiniFilterMessageR3(PNET_MESSAGE_INFO pNetMessageInfo);
		
		
		NTSTATUS PtUnload(__in FLT_FILTER_UNLOAD_FLAGS Flags);
		NTSTATUS PtInstanceQueryTeardown(__in PCFLT_RELATED_OBJECTS FltObjects, __in FLT_INSTANCE_QUERY_TEARDOWN_FLAGS Flags);
		
		static CONST FLT_REGISTRATION FilterRegistration = {
			sizeof(FLT_REGISTRATION),         //  Size
			FLT_REGISTRATION_VERSION,           //  Version
			0,                                  //  Flags
			NULL,                               //  Context
			NULL,                               //  Operation callbacks
			PtUnload,                           //  MiniFilterUnload
			NULL,                               //  InstanceSetup
			PtInstanceQueryTeardown,            //  InstanceQueryTeardown
			NULL,                               //  InstanceTeardownStart
			NULL,                               //  InstanceTeardownComplete
			NULL,                               //  GenerateFileName
			NULL,                               //  GenerateDestinationFileName
			NULL                                //  NormalizeNameComponent
		};
	
**MiniFilter 的实现（MiniFilterSeting.c）：**
	
	#include "MiniFilterSeting.h"
	
	//建立链接回调函数
	NTSTATUS ConnectNotifyCallback(IN PFLT_PORT ClientPort, IN PVOID ServerPortCookie, IN PVOID ConnectionContext, IN ULONG SizeOfContext, OUT PVOID* ConnectionPortCookie)
	{
		PAGED_CODE();
		UNREFERENCED_PARAMETER(ServerPortCookie);
		UNREFERENCED_PARAMETER(ConnectionContext);
		UNREFERENCED_PARAMETER(SizeOfContext);
		UNREFERENCED_PARAMETER(ConnectionPortCookie);
	
		g_ClientPort = ClientPort;
		return STATUS_SUCCESS;
	}
	
	//断开链接回调函数
	VOID DisconnectNotifyCallback(_In_opt_ PVOID ConnectionCookie)
	{
		PAGED_CODE();
		UNREFERENCED_PARAMETER(ConnectionCookie);
		FltCloseClientPort(gFilterHandle, &g_ClientPort);
	}
	
	
	NTSTATUS PtInstanceQueryTeardown(__in PCFLT_RELATED_OBJECTS FltObjects, __in FLT_INSTANCE_QUERY_TEARDOWN_FLAGS Flags)
	{
		return STATUS_SUCCESS;
	}
	
	
	NTSTATUS PtUnload(__in FLT_FILTER_UNLOAD_FLAGS Flags)
	{
		FltCloseCommunicationPort(g_ServerPort);
		FltUnregisterFilter(gFilterHandle);
		return STATUS_SUCCESS;
	}
	
	//创建一个微过滤器
	NTSTATUS CreatMiniFilter(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg)
	{
		NTSTATUS ntStatus = 0;
		PSECURITY_DESCRIPTOR sd;
		OBJECT_ATTRIBUTES oa;
		UNICODE_STRING uniString;
	
		do
		{
			if (pDriver == NULL)
			{
				ntStatus = STATUS_UNSUCCESSFUL;
				break;
			}
	
			if (pReg == NULL || pReg->Length <= 0)
			{
				ntStatus = STATUS_UNSUCCESSFUL;
				break;
			}
	
			//注册MiniFilter
			ntStatus = RegisterMiniFilter(pDriver, pReg);
			if (!NT_SUCCESS(ntStatus))
			{
	
				DbgPrint("注册MiniFilter失败！ \n");
				break;
			}
	
			//MiniFilter的回调函数
			ntStatus = FltRegisterFilter(pDriver, &FilterRegistration, &gFilterHandle);
			if (!NT_SUCCESS(ntStatus))
			{
				DbgPrint("注册MiniFilter的回调函数失败！ \n");
				break;
			}
	
			ntStatus = FltBuildDefaultSecurityDescriptor(&sd, FLT_PORT_ALL_ACCESS);
			if (!NT_SUCCESS(ntStatus))
			{
				DbgPrint("FltBuildDefaultSecurityDescriptor失败！ \n");
				break;
			}
	
			RtlInitUnicodeString(&uniString, L"\\CommentDriveTootPort");
			InitializeObjectAttributes(&oa, &uniString, OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE, NULL, sd);
	
			//创建一个通信服务器端口,接收来自用户模式应用程序的连接请求。
			ntStatus = FltCreateCommunicationPort(gFilterHandle, &g_ServerPort, &oa, NULL, ConnectNotifyCallback, DisconnectNotifyCallback, NULL, 1);
			FltFreeSecurityDescriptor(sd);
			if (!NT_SUCCESS(ntStatus))
			{
				DbgPrint("FltCreateCommunicationPort失败！ \n");
				break;
			}
	
			ntStatus = FltStartFiltering(gFilterHandle);
			if (!NT_SUCCESS(ntStatus))
			{
				DbgPrint("FltStartFiltering失败！ \n");
				break;
			}
	
		} while (FALSE);
	
	
		if (!NT_SUCCESS(ntStatus))
		{
			if (NULL != g_ServerPort) {
				FltCloseCommunicationPort(g_ServerPort);
			}
	
			if (NULL != gFilterHandle) {
				FltUnregisterFilter(gFilterHandle);
			}
		}
	
		return ntStatus;
	}
	
	//注册MiniFilter
	NTSTATUS RegisterMiniFilter(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg)
	{
		UNICODE_STRING UnicodeDriverServerName;
		UNICODE_STRING UnicodeValue;
		UNICODE_STRING UnicodeSzText;
		UNICODE_STRING UnicodeSzServerNameInstances;
		ULONG ulValue;
		HANDLE hRegister;
		ULONG ulResult;
		NTSTATUS ntStatus;
		static wchar_t szInstances[MAX_PATH] = { 0 };
		static wchar_t szServerNameInstances[MAX_PATH] = { 0 };
	
		//初始化objectAttributes    
		OBJECT_ATTRIBUTES objectAttributes;
		wchar_t* pFind = NULL;
	
		InitializeObjectAttributes(&objectAttributes, pReg, OBJ_CASE_INSENSITIVE, NULL, NULL);
	
		//创建或打开注册表项目    
		ntStatus = ZwCreateKey(&hRegister, KEY_ALL_ACCESS, &objectAttributes, 0, NULL, (ULONG)REG_OPTION_NON_VOLATILE, &ulResult);
		if (hRegister == NULL || ntStatus != STATUS_SUCCESS)
		{
			return STATUS_UNSUCCESSFUL;
		}
		ZwClose(hRegister);
	
		//DependOnService  
		RtlInitUnicodeString(&UnicodeValue, L"DependOnService");
		SetRegeditValueKey(pReg, &UnicodeValue, REG_SZ, L"FltMgr");
	
		//Instances  
		RtlStringCbPrintfExW(szServerNameInstances, sizeof(szServerNameInstances), NULL, NULL, STRSAFE_FILL_BEHIND_NULL, L"%wZ\\Instances", pReg);
		RtlInitUnicodeString(&UnicodeSzServerNameInstances, szServerNameInstances);
		InitializeObjectAttributes(&objectAttributes, &UnicodeSzServerNameInstances, OBJ_CASE_INSENSITIVE, NULL, NULL);
		ntStatus = ZwCreateKey(&hRegister, KEY_ALL_ACCESS, &objectAttributes, 0, NULL, REG_OPTION_NON_VOLATILE, &ulResult);
		if (hRegister == NULL || ntStatus != STATUS_SUCCESS) return STATUS_UNSUCCESSFUL;
		ZwClose(hRegister);
	
		//获取服务名  
		pFind = wcsrchr(pReg->Buffer, '\\');
		if (pFind)
		{
			RtlInitUnicodeString(&UnicodeDriverServerName, pFind + sizeof(char));
		}
		else
		{
			return STATUS_UNSUCCESSFUL;
		}
	
		//Default Instance  
		RtlInitUnicodeString(&UnicodeValue, L"DefaultInstance");
		RtlStringCbPrintfExW(szInstances, sizeof(szInstances), NULL, NULL, STRSAFE_FILL_BEHIND_NULL, L"%wZ Instance", &UnicodeDriverServerName);
		SetRegeditValueKey(&UnicodeSzServerNameInstances, &UnicodeValue, REG_SZ, szInstances);
	
	
		//ProtectFile Instance  
		RtlStringCbPrintfExW(szInstances, sizeof(szInstances), NULL, NULL, STRSAFE_FILL_BEHIND_NULL, L"%wZ\\%wZ Instance", &UnicodeSzServerNameInstances, &UnicodeDriverServerName);
		RtlInitUnicodeString(&UnicodeSzText, szInstances);
		InitializeObjectAttributes(&objectAttributes, &UnicodeSzText, OBJ_CASE_INSENSITIVE, NULL, NULL);
		ntStatus = ZwCreateKey(&hRegister, KEY_ALL_ACCESS, &objectAttributes, 0, NULL, REG_OPTION_NON_VOLATILE, &ulResult);
		if (hRegister == NULL || ntStatus != STATUS_SUCCESS) return STATUS_UNSUCCESSFUL;
		ZwClose(hRegister);
	
		//Altitude  
		RtlInitUnicodeString(&UnicodeValue, L"Altitude");
		SetRegeditValueKey(&UnicodeSzText, &UnicodeValue, REG_SZ, L"399999");
	
		//Flags  
		RtlInitUnicodeString(&UnicodeValue, L"Flags");
		ulValue = 0;
		SetRegeditValueKey(&UnicodeSzText, &UnicodeValue, REG_DWORD, (wchar_t*)&ulValue);
	
		return ntStatus;
	}
	
	NTSTATUS SetRegeditValueKey(PUNICODE_STRING pRegPath, PUNICODE_STRING pValueName, ULONG Type, wchar_t ValueData[MAX_PATH])
	{
		//定义变量  
		size_t pcch = 0;
		OBJECT_ATTRIBUTES objectAttribues;
		HANDLE hRegister = NULL;
		NTSTATUS ntstatus = 0;
		USHORT cbszSize = 0;
	
		//参数效验  
		if (pRegPath == NULL || pValueName == 0 || ValueData == NULL)
		{
			return FALSE;
		}
	
		switch (Type)
		{
		case REG_SZ:
		{
			//获取长度  
			RtlStringCchLengthW(ValueData, MAX_PATH, &pcch);
			if (pcch <= 0)return FALSE;
			cbszSize = (USHORT)(pcch * sizeof(wchar_t)) + sizeof(wchar_t);
		}
		break;
		case REG_DWORD:
		{
			cbszSize = sizeof(ULONG);
		}
		break;
		default:
			return STATUS_UNSUCCESSFUL;
		}
	
		//设置变量  
		InitializeObjectAttributes(&objectAttribues, pRegPath, OBJ_CASE_INSENSITIVE, NULL, NULL);
	
		//打开注册表  
		ntstatus = ZwOpenKey(&hRegister, KEY_ALL_ACCESS, &objectAttribues);
		if (!NT_SUCCESS(ntstatus) || hRegister == NULL)
		{
			return FALSE;
		}
	
		//设置REG_SZ子健  
		ntstatus = ZwSetValueKey(hRegister, pValueName, 0, Type, ValueData, cbszSize);
		ZwClose(hRegister);
		return ntstatus;
	}
	
	
	NTSTATUS SendMiniFilterMessageR3(PNET_MESSAGE_INFO pNetMessageInfo)
	{
		NTSTATUS ntStatus = 0;
		
		ntStatus = FltSendMessage(gFilterHandle, &g_ClientPort, pNetMessageInfo, sizeof(NET_MESSAGE_INFO), NULL, NULL, NULL);
		if (NT_SUCCESS(ntStatus))
		{
			DbgPrint("发送成功！ \n");
		}
		else
		{
			DbgPrint("发送失败！ \n");
		}
	
		return ntStatus;
	}


**MiniFilter 的使用（DriveMain.c）：**


	//驱动入口函数
	NTSTATUS DriverEntry(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg) //返回一个地址、返回驱动被注册到注册表的某个地方
	{
		DbgPrint("pdriver = %wZ, , %x\n", pReg, pDriver);
		g_pDriverObject = pDriver;
		pDriver->DriverUnload = DriverUpLoad;
		NTSTATUS ntStatus;
	
		//创建一个微过滤器
		ntStatus = CreatMiniFilter(pDriver, pReg);
		if (!NT_SUCCESS(ntStatus))
		{
			DbgPrint("CreatMiniFilter失败！ \n");
		}
	

		//发送消息，用上面定义的结构体
		PNET_MESSAGE_INFO pNetMeassageInfo = (PNET_MESSAGE_INFO)systemBuf; //发送数据
		memset(pNetMeassageInfo, 0, sizeof(NET_MESSAGE_INFO));

		//上面定义的消息类型
		pNetMeassageInfo->dwDataType = NET_MINIFILTER_MESSAGE_DRIVE_INFO;
		SendMiniFilterMessageR3(pNetMeassageInfo);

		return STATUS_SUCCESS;
	}


## 应用层代码

**R3的代码是写了个类，进行封装了一下，头文件（CNetDriveMessage.h）：**

	#pragma once
	#include <windows.h>
	#include <string>
	#include <Fltuser.h>
	#include "PublicDefinition.h"
	
	using namespace std;
	
	#define SCANNER_DEFAULT_REQUEST_COUNT       5
	#define SCANNER_DEFAULT_THREAD_COUNT        2
	#define SCANNER_MAX_THREAD_COUNT            64
	
	typedef struct _SCANNER_THREAD_CONTEXT 
	{
		HANDLE Port;
		HANDLE Completion;
	
	} SCANNER_THREAD_CONTEXT, * PSCANNER_THREAD_CONTEXT;
	
	
	//接受消息结构体
	typedef struct _SCANNER_MESSAGE
	{
	
		//
		//  Required structure header.
		//
	
		FILTER_MESSAGE_HEADER MessageHeader;  //固定的
	
	
		//
		//  Private scanner-specific fields begin here.特定于扫描仪的私有字段从这里开始。
		//
		//下面才是我们可以自己设定的

		NET_MESSAGE_INFO Notification;  //前面定义的专门用来传送消息的结构体
	
		//
		//  Overlapped structure: this is not really part of the message
		//  However we embed it instead of using a separately allocated overlap structure
		//
	
		OVERLAPPED Ovlp;   //固定的
	
	} SCANNER_MESSAGE, * PSCANNER_MESSAGE;
	
	
	//发送消息结构体
	typedef struct _SCANNER_REPLY_MESSAGE {
	
		//
		//  Required structure header.  
		//
	
		FILTER_REPLY_HEADER ReplyHeader;  //固定的
	
		//
		//  Private scanner-specific fields begin here.特定于扫描仪的私有字段从这里开始。
		//
		//下面才是我们可以自己设定的

		NET_MESSAGE_INFO Reply; 
	
	} SCANNER_REPLY_MESSAGE, * PSCANNER_REPLY_MESSAGE;
	
	
	class CNetDriveMessage
	{
	
	public:
	
		CNetDriveMessage();	// 标准构造函数
		~CNetDriveMessage();
	
	
	public:
	
		HANDLE m_DriverHanlde;
		HANDLE m_hPort;
		SCANNER_THREAD_CONTEXT context;
	
	public:
		
		
		BOOL OpenDevice(LPCWSTR lpFileName);
		
		//使用 DeviceIoControl 发送一个消息到，驱动
		BOOL SendMessageDriver(DWORD dwIoControlCode);
	
		//创建一个接受驱动层消息的线程，接受一个函数地址
		BOOL ConnectMessage(LPTHREAD_START_ROUTINE lpStartAddress); 
		
		BOOL ReplyMessageToDevic(HANDLE m_hPort, PSCANNER_MESSAGE pScannerMessage, PNET_MESSAGE_INFO pNetMessageInfo);
	};

用于发送和接受消息的两个结构体，是固定格式的，但名字可以改变，我们从R0发送到R3的消息是经过了这个固定格式进行封装的。

**CNetDriveMessage类的实现（CNetDriveMessage.cpp）：**

	#include "pch.h"
	#include "CNetDriveMessage.h"
	
	CNetDriveMessage::CNetDriveMessage()
	{
		m_DriverHanlde = NULL;
		m_hPort = NULL;
	
		memset(&context, 0, sizeof(SCANNER_THREAD_CONTEXT));
	}
	
	CNetDriveMessage::~CNetDriveMessage()
	{
		if (m_DriverHanlde != NULL)
		{
			CloseHandle(m_DriverHanlde);
		}
	}
	
	BOOL CNetDriveMessage::OpenDevice(LPCWSTR lpFileName)
	{
	
		HANDLE hDeviceHanlde = NULL;
		BOOL bRet = FALSE;
	
		hDeviceHanlde = CreateFileW(lpFileName, GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	
		if (hDeviceHanlde != NULL)
		{
			m_DriverHanlde = hDeviceHanlde;
			bRet = TRUE;
		}
	
		return bRet;
	}
	
	//创建一个线程来接受消息
	BOOL CNetDriveMessage::ConnectMessage(LPTHREAD_START_ROUTINE lpStartAddress)
	{
		DWORD requestCount = SCANNER_DEFAULT_REQUEST_COUNT;
		DWORD threadCount = SCANNER_DEFAULT_THREAD_COUNT;
		HANDLE threads[SCANNER_MAX_THREAD_COUNT] = { INVALID_HANDLE_VALUE };
		HANDLE port, completion;
		HRESULT hr;
		DWORD threadId;
		int i = 0;
	
		hr = FilterConnectCommunicationPort(PortName,
			0,
			NULL,
			0,
			NULL,
			&m_hPort);
	
		if (IS_ERROR(hr)) {
	
			return FALSE;
		}
		
		context.Port = m_hPort;
	
	
		for (int i = 0; i < threadCount; i++)
		{
			threads[i] = CreateThread(
				NULL,
				0,
				lpStartAddress,
				this,
				0,
				&threadId);
	
			if (threads[i] == NULL) {
	
				//
				//  Couldn't create thread.
				//
	
				hr = GetLastError();
				goto main_cleanup;
			}
		}
	
		hr = S_OK;
	
		return TRUE;
	
	main_cleanup:
	
		//CloseHandle( completion );
	
		return FALSE;
	}
	
	BOOL CNetDriveMessage::ReplyMessageToDevic(HANDLE m_hPort, PSCANNER_MESSAGE pScannerMessage, PNET_MESSAGE_INFO pNetMessageInfo)
	{
		SCANNER_REPLY_MESSAGE replyMessage;
		memset(&replyMessage, 0, sizeof(SCANNER_REPLY_MESSAGE));
	
		replyMessage.ReplyHeader.Status = 0;
		replyMessage.ReplyHeader.MessageId = pScannerMessage->MessageHeader.MessageId;
	
		memcpy(&replyMessage.Reply, pNetMessageInfo, sizeof(NET_MESSAGE_INFO));
		
		HRESULT hr = FilterReplyMessage(m_hPort, (PFILTER_REPLY_HEADER)&replyMessage, sizeof(replyMessage));
		if (SUCCEEDED(hr))
		{
			printf("Replied message\n");
			return TRUE;
		}
		else
		{
			printf("Scanner: Error replying message. Error = 0x%X\n", hr);
			return FALSE;
		}
	}
	
	BOOL CNetDriveMessage::SendMessageDriver(DWORD dwIoControlCode)
	{
		NET_MESSAGE_INFO NetMessageInfo;
		memset(&NetMessageInfo, 0, sizeof(NET_MESSAGE_INFO));
	
		DWORD len = 0;
		DeviceIoControl(m_DriverHanlde, dwIoControlCode, &NetMessageInfo, sizeof(NET_MESSAGE_INFO), &NetMessageInfo, sizeof(NET_MESSAGE_INFO), &len, NULL);
	
		return TRUE;
	}


**CNetDriveMessage类的使用**


	int Main()
	{
		CNetDriveMessage NetDriveMessage;
	
		NetDriveMessage.ConnectMessage(EncryptWorkerThread);
		
		return 0;
	}

其中，ConnectMessage 接受一个线程函数地址，这个线程负责用来接受消息很重要：

	DWORD WINAPI EncryptWorkerThread(__in LPVOID lpParam)
	{
		CNetDriveMessage* netDriveMessage = (CNetDriveMessage*)lpParam;
		BOOL result;
		DWORD outSize;
		LPOVERLAPPED pOvlp;
		PVOID pShareUserAdd = NULL;
		ULONG_PTR key;
		HRESULT hr = NULL;
	
		PSCANNER_MESSAGE message = (PSCANNER_MESSAGE)malloc(sizeof(SCANNER_MESSAGE));
		if (message == NULL) {
	
			//hr = ERROR_NOT_ENOUGH_MEMORY;
			return 0;
		}
	
		memset(message, 0, sizeof(SCANNER_MESSAGE));
	
	
		OVERLAPPED OverLapped = { 0 };
		OverLapped.hEvent = CreateEvent(NULL, FALSE, FALSE, NULL);
	
		while (TRUE)
		{
			//获取消息
			hr = FilterGetMessage(netDriveMessage->m_hPort,
				&message->MessageHeader,
				//FIELD_OFFSET( DRIVER_MESSAGE, Ovlp ),
				sizeof(SCANNER_MESSAGE),
				&OverLapped);
	
			WaitForSingleObject(OverLapped.hEvent, INFINITE);
			ResetEvent(OverLapped.hEvent);
			
			//从接受消息结构体中获取我们自己定义的结构体
			NET_MESSAGE_INFO netMessageInfo = message->Notification;

			switch (netMessageInfo.dwDataType)   //根据发送的消息类型接受消息
			{
			case NET_MINIFILTER_MESSAGE_DRIVE_INFO:
	
				MessageBox(NULL, TEXT("这是对话框"), TEXT("你好"), MB_ICONINFORMATION | MB_YESNO);
				
				//这里就可以使用共享内存的指针来获取数据了

				
			default:
				break;
			}
		}
	
		delete message;
		return 0;
	}