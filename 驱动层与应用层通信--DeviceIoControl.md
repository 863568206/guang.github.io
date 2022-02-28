---
layout:     post
title:      驱动层与应用层通信--DeviceIoControl
subtitle:   c++
date:       2022-2-28
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - Driver
---

# 驱动层与应用层通信--DeviceIoControl

使用DeviceIoControl，进行驱动层与应用层的通信。

## 驱动层程序实现

	#include "DriveToolMain.h"
	#include <Ntstrsafe.h>
	
	//名称
	#define DEVICE_NAME  L"\\Driver\\DriverToolModule"
	#define SYMBOL_NAME_LINK  L"\\??\\DeviceToolLink"

	//定义消息
	#define CODE_GET_ALL_DRIVE_INFO CTL_CODE(FILE_DEVICE_UNKNOWN, 0x840, METHOD_BUFFERED, FILE_ANY_ACCESS)
	
	//消息结构体
	typedef struct _NET_MESSAGE_INFO
	{
		LONGLONG ulShareUserAdd;   //共享到R3的地址
		DWORD dwDataType;
		DWORD dwStart;

	} NET_MESSAGE_INFO, * PNET_MESSAGE_INFO;	


	//卸载
	VOID DriverUpLoad(PDRIVER_OBJECT pDriver)
	{
		UNICODE_STRING strSymbolLink;
		RtlInitUnicodeString(&strSymbolLink, SYMBOL_NAME_LINK);
	
		IoDeleteSymbolicLink(&strSymbolLink);  //卸载符号链接
	
		if (pDriver->DeviceObject != NULL)
		{
			IoDeleteDevice(pDriver->DeviceObject); //卸载对象
		}
	
		DbgPrint("卸载了\n");
	}
	
	// 不设置这个函数，则Ring3调用CreateFile会返回1
	// IRP_MJ_CREATE 处理函数
	NTSTATUS IrpCreateProc(PDEVICE_OBJECT pDevObj, PIRP pIrp)
	{
		DbgPrint("应用层连接设备.\n");
		// 返回状态如果不设置，Ring3返回值是失败
		pIrp->IoStatus.Status = STATUS_SUCCESS;
		pIrp->IoStatus.Information = 0;
		IoCompleteRequest(pIrp, IO_NO_INCREMENT);
		return STATUS_SUCCESS;
	}
	
	// IRP_MJ_CLOSE 处理函数
	NTSTATUS IrpCloseProc(PDEVICE_OBJECT pDevObj, PIRP pIrp)
	{
		DbgPrint("应用层断开连接设备.\n");
		// 返回状态如果不设置，Ring3返回值是失败
		pIrp->IoStatus.Status = STATUS_SUCCESS;
		pIrp->IoStatus.Information = 0;
		IoCompleteRequest(pIrp, IO_NO_INCREMENT);
		return STATUS_SUCCESS;
	}
	
	//发送设备设备， 设备对象， 输入输出包Irp
	NTSTATUS DisPatchCallBack(PDEVICE_OBJECT pDeviceObject, IRP* Irp)
	{
		//解析自定义消息码
		PIO_STACK_LOCATION pSl = IoGetCurrentIrpStackLocation(Irp);  //获取数据
		ULONG iCode = pSl->Parameters.DeviceIoControl.IoControlCode;  //获取控制码
	
		PVOID systemBuf = Irp->AssociatedIrp.SystemBuffer;      //文件缓冲区地址
		ULONG inLen = pSl->Parameters.DeviceIoControl.InputBufferLength; //输入长度
		ULONG outLen = pSl->Parameters.DeviceIoControl.OutputBufferLength; //输出长度
	
		switch (iCode)
		{
		case CODE_GET_ALL_DRIVE_INFO:
		{
			DbgPrint("收到消息:CODE_GET_ALL_DRIVE_INFO \n");
			NetGetAllDriveInfo(Irp);
		}
		break;
		}
	
		Irp->IoStatus.Status = STATUS_SUCCESS;
		IofCompleteRequest(Irp, IO_NO_INCREMENT);
	
		return STATUS_SUCCESS;
	}
	
	//创建一个设备对象
	NTSTATUS CreatDevice(PDRIVER_OBJECT pDriver)
	{
		UNICODE_STRING  strDeviceName;
		UNICODE_STRING  strSymbolNameLink;
		PDEVICE_OBJECT pDeviceObj;
		NTSTATUS status = 0;
	
		do
		{
			//创建一个设备对象
			RtlInitUnicodeString(&strDeviceName, DEVICE_NAME);  //初始化字符串
			status = IoCreateDevice(pDriver, 0, &strDeviceName, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, TRUE, &pDeviceObj);
			if (!NT_SUCCESS(status)) //判断是否创建成功
			{
				KdPrint(("Creat Devic Lose\n"));
				break;
			}
	
			//创建一个符号链接
			RtlInitUnicodeString(&strSymbolNameLink, SYMBOL_NAME_LINK);  //初始化字符串
			status = IoCreateSymbolicLink(&strSymbolNameLink, &strDeviceName);
			if (!NT_SUCCESS(status)) //判断是否创建成功
			{
				IoDeleteDevice(pDeviceObj);
				KdPrint(("Creat SymbolLink Lose\n"));
				break;
			}
	
			//指定IO
			pDeviceObj->Flags |= DO_BUFFERED_IO;
			pDriver->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DisPatchCallBack; //用于接受消息
			pDriver->MajorFunction[IRP_MJ_CREATE] = IrpCreateProc;
			pDriver->MajorFunction[IRP_MJ_CLOSE] = IrpCloseProc;
	
		} while (FALSE);
	
		return status;
	}
	
	
	//驱动入口函数
	NTSTATUS DriverEntry(PDRIVER_OBJECT pDriver, PUNICODE_STRING pReg) //返回一个地址、返回驱动被注册到注册表的某个地方
	{
		DbgPrint("pdriver = %wZ, , %x\n", pReg, pDriver);
		g_pDriverObject = pDriver;
		pDriver->DriverUnload = DriverUpLoad;
		NTSTATUS ntStatus;
	
		//创建一个设备对象
		CreatDevice(pDriver);
	
		return STATUS_SUCCESS;
	}
	
	
	void NetGetAllDriveInfo(IRP* Irp)
	{
		PVOID systemBuf = Irp->AssociatedIrp.SystemBuffer;      //文件缓冲区地址
		PNET_MESSAGE_INFO pNetMeassageInfo = (PNET_MESSAGE_INFO)systemBuf; //发送数据

		//使用PNET_MESSAGE_INFO的数据
	
		
		//准备发送
		memset(pNetMeassageInfo, 0, sizeof(NET_MESSAGE_INFO));
		pNetMeassageInfo->ulShareUserAdd = (LONGLONG)g_pShareUserAdd_AllUploadDrive;
		DbgPrint("NET_MESSAGE_INFO = %x \n", pNetMeassageInfo->ulShareUserAdd);
		
	
		Irp->IoStatus.Information = sizeof(NET_MESSAGE_INFO);
	}


## 应用层程序实现

	#define SYMBOL_NAME_LINK  L"\\??\\DeviceToolLink" 
	#define CODE_GET_ALL_DRIVE_INFO CTL_CODE(FILE_DEVICE_UNKNOWN, 0x840, METHOD_BUFFERED, FILE_ANY_ACCESS)
	
	int main()
	{
		HANDLE hDeviceHanlde = NULL;
		BOOL bRet = FALSE;
		
		//打开一个驱动
		hDeviceHanlde = CreateFileW(SYMBOL_NAME_LINK, GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	
		if (hDeviceHanlde != NULL)
		{
			m_DriverHanlde = hDeviceHanlde;
			bRet = TRUE;
		}
	
		NET_MESSAGE_INFO NetMessageInfo;
		memset(&NetMessageInfo, 0, sizeof(NET_MESSAGE_INFO));
	
		DWORD len = 0;
		
		if(DeviceIoControl(m_DriverHanlde, dwIoControlCode, &NetMessageInfo, sizeof(NET_MESSAGE_INFO), &NetMessageInfo, sizeof(NET_MESSAGE_INFO), &len, NULL))
		{
			//接受返回消息
		}
		
		return 0;
	}

## 黄蔷薇 – 永恒的微笑