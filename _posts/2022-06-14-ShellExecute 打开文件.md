---
layout:     post
title:      ShellExecute 打开文件
subtitle:   c++
date:       2022-06-14
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

# ShellExecute 打开文件

[ShellExecute介绍](https://docs.microsoft.com/zh-cn/windows/win32/shell/shell-shellexecute?redirectedfrom=MSDN)

程序：

	wstring  strFilePath; //文件路径
	
	HINSTANCE  Hinstance = ShellExecute(NULL, _T("open"), strFileLocalPath.c_str(), NULL, NULL, SW_SHOW);
	if ((int)Hinstance <= 32)
	{
		if ((int)Hinstance == SE_ERR_NOASSOC)
		{
			//提示：没有关联的程序打开文件
			
		}
	}

## 蓝色妖姬 -- 相守