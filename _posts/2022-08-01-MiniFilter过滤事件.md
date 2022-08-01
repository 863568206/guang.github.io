---
layout:     post
title:      MiniFilter过滤事件
subtitle:   c++
date:       2022-08-01
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - Driver
---

# MiniFilter过滤

MiniFilter可以用来过滤一些事件，[MiniFilter的创建](http://yimitumi.com/2022/02/28/%E9%A9%B1%E5%8A%A8%E5%B1%82%E4%B8%8E%E5%BA%94%E7%94%A8%E5%B1%82%E9%80%9A%E4%BF%A1-MiniFilter/).

通过过滤这些事件我们可以实现比如限制任何文件的读取、删除、覆盖重命名、执行等操作。

## 注册回调函数

在 FltRegisterFilter 函数中，第二个参数接受一个为过滤器的 FLT\_REGISTRATION 结构体。

	NTSTATUS
	FLTAPI
	FltRegisterFilter (
	    _In_ PDRIVER_OBJECT Driver,
	    _In_ CONST FLT_REGISTRATION *Registration,
	    _Outptr_ PFLT_FILTER *RetFilter
	    );

FLT\_REGISTRATION（这里只写定义好的，具体可以去插）：

	static CONST FLT_REGISTRATION FilterRegistration = {
		sizeof(FLT_REGISTRATION),         //  Size
		FLT_REGISTRATION_VERSION,           //  Version
		0,                                  //  Flags
		NULL,                               //  Context
		CallBacks,                          //  Operation callbacks
		PtUnload,                           //  MiniFilterUnload
		NULL,                               //  InstanceSetup
		PtInstanceQueryTeardown,            //  InstanceQueryTeardown
		NULL,                               //  InstanceTeardownStart
		NULL,                               //  InstanceTeardownComplete
		NULL,                               //  GenerateFileName
		NULL,                               //  GenerateDestinationFileName
		NULL                                //  NormalizeNameComponent
	};

其中第5个参数CallBacks是操作回调函数集注册，在其中可以处理所有的请求：

	//注册回调函数 {消息类型, 0, 预操作回调函数, 后操作回调函数}
	static FLT_OPERATION_REGISTRATION CallBacks[] =
	{
		{ IRP_MJ_CREATE, 0, NPPreCreate, NULL },
		{ IRP_MJ_OPERATION_END } //用于结束
	};

FLT\_OPERATION\_REGISTRATION 是CallBacks中的结构体：

	参数1 ： 事件类型
	参数2 ： 标志位
	参数3 ： 预操作回调函数（在请求完成前处理，适合拦截请求本身的情况）
	参数4 ： 后操作回调函数（事件等待请求完成后，适合拦截请求之后返回的结果的情况）

## 回调函数的声明与定义

声明：
	
	//生成和打开文件的预操作回调函数
	FLT_PREOP_CALLBACK_STATUS NPPreCreate(
	__inout PFLT_CALLBACK_DATA pCallBackData,
	 __in PCFLT_RELATED_OBJECTS pFltObjects,
	 __deref_out_opt PVOID* pCompletionContext);


定义：

	//生成和打开文件的预操作回调函数
	FLT_PREOP_CALLBACK_STATUS NPPreCreate(__inout PFLT_CALLBACK_DATA pCallBackData, __in PCFLT_RELATED_OBJECTS pFltObjects, __deref_out_opt PVOID* pCompletionContext)
	{
		DbgPrint("NPPreCreate -- Function \n");
	
		UNREFERENCED_PARAMETER(pCallBackData);
		UNREFERENCED_PARAMETER(pFltObjects);
		UNREFERENCED_PARAMETER(pCompletionContext);
		
		//处理程序，比如限制某个文件的读取、删除、覆盖、重命名、执行等
		


	
		return FLT_PREOP_SUCCESS_WITH_CALLBACK;
	}


## 蓝色妖姬 -- 相守