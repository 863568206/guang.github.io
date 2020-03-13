---
layout:     post
title:      获取TrustedInstaller权限
subtitle:   c++
date:       2019-11-26
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

# 获取TrustedInstaller权限

TrustedInstaller其实是windows系统中的一个虚拟用户,当你需要对C盘的一些系统文件进行修改的时候就会提示你“需要TrustedInstaller提供的权限才能修改此文件”。（有种VIP用户的感觉）

就不介绍手动修改了手动修改在网上很多，可以去查一下。

主要使用dos命令来进行修改，在C++中执行dos命令可以用下面这个函数：

	void ExeCmd(wstring pszCmd)
	{
		wstring wstrCmd = pszCmd;
	
		HANDLE hWrite = NULL;
	
		//设置命令行参数
		STARTUPINFO si = { sizeof(STARTUPINFO) };
		GetStartupInfo(&si);
		//si.wShowWindow = SW_HIDE; //隐藏窗口
		si.hStdError = hWrite;
		si.hStdOutput = hWrite;
	
		//启动命令
		PROCESS_INFORMATION pi;
	
		if (!CreateProcess(
						NULL,
						(LPWSTR)wstrCmd.c_str(),
						NULL,
						NULL,
						TRUE,
						NULL,
						NULL,
						NULL,
						&si,
						&pi))
		{
			return ;
		}
		CloseHandle(pi.hProcess);
		CloseHandle(pi.hThread);  //不加会导致资源泄露
		CloseHandle(hWrite);//关闭管道的输入端口
	}

dos命令：

takeown命令：

	takeown /f C:\Windows\System32 /a /r /d y  

**takeown** 将会把当前的文件夹的所有者都改成当前的管理员用户。这个可以把当前文件夹中的所有文件都该掉。

	takeown /f C:\Windows\System32\devmgmt.msc /a

这个用于修改单个文件。

icacls命令：

	icacls C:\Windows\System32 /grant administrators:F /T

**icacls** 将会为当前文件夹的所有的文件添加 **administrators** 的访问权限。这个可以把当前文件夹中的所有文件都该掉。

	 icacls C:\Windows\System32\compmgmt.msc /grant administrators:F /T

这个用于修改单个文件。

备注：先调用 **takeown** 把获取TrustedInstaller权限的文件改为管理员用户，然后调用 **icacls** 将文件添加 **administrators** 的访问权限就可以修改了。

## 紫藤花 -- 依依的思念，对你执着

