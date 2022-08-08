---
layout:     post
title:      TimeSetEvent定时器
subtitle:   c++
date:       2022-08-08
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

# 添加一个定时器

timeSetEvent 是一种精确度非常高的定时器，与windows定时器不同的是，多媒体定时器不使用容易丢失的窗口消息。

**头文件：**

	#include <mmsystem.h>

**定义：**
	
	//定义时间类型
	MMRESULT timeProcOne;
	MMRESULT timeProcTwo;  
	
	//定义线程函数
	void CALLBACK TimeProcOne(UINT uTimerID, UINT uMsg, DWORD_PTR dwUser, DWORD_PTR dw1, DWORD_PTR dw2);
	void CALLBACK TimeProcTwo(UINT uTimerID, UINT uMsg, DWORD_PTR dwUser, DWORD_PTR dw1, DWORD_PTR dw2);

**实现：**

	int main()
	{
		//初始化
		TIMECAPS tc;
		timeGetDevCaps(&tc, sizeof(TIMECAPS));
		DWORD uResolution = min(max(tc.wPeriodMin, 0), tc.wPeriodMax);
		timeBeginPeriod(uResolution);
	
		//定时器one
		timeProcOne = timeSetEvent(
			5 * 1000,        //毫秒     
			uResolution,
			TimeProcOne,     //回调函数名
			(DWORD_PTR)this, //用于传递参数，可以为NULL
			TIME_PERIODIC);
	
		//定时器two
		timeProcTwo = timeSetEvent(
			60 * 1000,        //毫秒      
			uResolution,
			TimeProcTwo,      //回调函数名
			(DWORD_PTR)this,  //用于传递参数，可以为NULL
			TIME_PERIODIC);
	
	
		return 0;
	}

	void CALLBACK TimeProcOne(UINT uTimerID, UINT uMsg, DWORD_PTR dwUser, DWORD_PTR dw1, DWORD_PTR dw2)
	{
		OutputDebugStringA("-----------Time-5s------TimeProcOne--------");
	
	}
	
	void CALLBACK TimeProcTwo(UINT uTimerID, UINT uMsg, DWORD_PTR dwUser, DWORD_PTR dw1, DWORD_PTR dw2)
	{
		OutputDebugStringA("-----------Time-60s------TimeProcTwo--------");
	}



## 薰衣草 -- 等待爱情
