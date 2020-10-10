---
layout:     post
title:      CreateDirectory创建路径
subtitle:   c++/MFC
date:       2020-10-10
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - windows
---

# CreateDirectory创建路径

[CreateDirectory](https://docs.microsoft.com/en-us/dotnet/api/system.io.directory.createdirectory?view=netcore-3.1)可以用来创建路径，但是这个函数不是递归的。它可以在一个路径中创建唯一的最终目录。也就是说，如果父目录或中间目录不存在，该函数将失败并显示错误消息**ERROR_PATH_NOT_FOUND**。

依次创建文件路径：

	void CreateClientFileBackUpPath(CString strFileBackUpPath)
	{	
		if (strFileBackUpPath.IsEmpty())
		{
			return;
		}
	
		vector<CString> vecFilePath;
		for (int i = 0; i != strFileBackUpPath.GetLength(); i++)
		{
			CString strSuperiorFileClinetPath;
			if (strFileBackUpPath[i] == '\\')
			{
				strSuperiorFileClinetPath = strFileBackUpPath.Mid(0, i);
				vecFilePath.push_back(strSuperiorFileClinetPath);
			}
		}
	
		if (vecFilePath.empty())
		{
			return;
		}
	
		vector<CString>::iterator iterVec = vecFilePath.begin();
		for (; iterVec != vecFilePath.end(); iterVec++)
		{
			if (PathFileExistsW(*iterVec))
			{
				continue;
			}
	
			CreateDirectoryW((*iterVec).GetString(), NULL);
		}
	}


## 蝴蝶花 -- 相信就是幸福 