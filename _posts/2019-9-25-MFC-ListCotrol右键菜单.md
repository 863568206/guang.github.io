---
layout:     post
title:      MFC-ListCotrol
subtitle:   MFC
date:       2019-07-26
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - MFC
---

# MFC List初始化

	LONG lStyle;
	lStyle = GetWindowLong(m_list.m_hWnd, GWL_STYLE); //获取当前窗口style
	lStyle &= ~LVS_TYPEMASK; //清除显示方式位
	lStyle |= LVS_REPORT; //设置style
	lStyle |= LVS_SINGLESEL;//单选模式
	SetWindowLong(m_list.m_hWnd, GWL_STYLE, lStyle);//设置style

	DWORD dwStyle = m_list.GetExtendedStyle();
	dwStyle |= LVS_EX_FULLROWSELECT;//选中某行使整行高亮（只适用与report风格的listctrl）//557   335
	dwStyle |= LVS_EX_GRIDLINES;//网格线（只适用与report风格的listctrl）
	dwStyle |= LVS_EX_CHECKBOXES;//item前生成checkbox控件
	dwStyle |= LVS_EX_SUBITEMIMAGES; //添加图片
	dwStyle |= LVS_OWNERDATA;
	m_list.SetExtendedStyle(dwStyle); //设置扩展风格


# MFC列表框右键菜单

## 列表框右键弹出菜单设置

经常看到表里右键弹出菜单，然后点击弹出窗口或删除当前数据等操作。
首先我们要在资源视图中添加一个Menu菜单并设置选项，然后可以添加在 **ListCotrol** 中添加菜单的 **NM_RCLICK** 消息的响应函数，下面是手动添加。

弹出菜单消息相应函数：

   xxx.h

	afx_msg void OnNMRClickListCotrol(NMHDR *pNMHDR, LRESULT *pResult);

   xxx.cpp
	
	//在cpp前面添加消息
	ON_NOTIFY(NM_RCLICK, IDR_List_xxx_列表ID, &xxx::OnNMRClickListCotrol)

	//弹出菜单设置
	void xxx::OnNMRClickListCotrol(NMHDR *pNMHDR, LRESULT *pResult)
	{
		// TODO: 在此添加控件通知处理程序代码
		do 
		{
			LPNMITEMACTIVATE pNMItemActivate = reinterpret_cast<LPNMITEMACTIVATE>(pNMHDR);
			//LPNMITEMACTIVATE pNMItemActivate = LPNMITEMACTIVATE(pNMHDR);
			//判断列表是否为空
			if (m_list.GetItemCount() <= 0)
			{
				break;
			}
			//判断是否有列表选中
			if (m_list.GetSelectedCount() > 0)
			{
				CMenu menu, *popup;
				//添加的主菜单
				if (menu.LoadMenu(IDR_MENU_xxx_菜单ID) == NULL)
				{
					break;
				}

				//获取要显示菜单的句柄（0表示第一个）
				popup = menu.GetSubMenu(0);

				//位置信息
				CPoint point;
				ClientToScreen(&point);
				GetCursorPos(&point);
				popup->TrackPopupMenu(TPM_LEFTALIGN | TPM_RIGHTBUTTON, point.x, point.y, this);
				*pResult = 0;
			}
	
		} while (FALSE);

		return;
	}

## 随便位置弹出

头文件和消息和上面一样。

   cpp
   
	//弹出菜单设置
	void xxx::OnNMRClickListCotrol(NMHDR *pNMHDR, LRESULT *pResult)
	{
		// TODO: 在此添加控件通知处理程序代码
		LPNMITEMACTIVATE pNMItemActivate = reinterpret_cast<LPNMITEMACTIVATE>(pNMHDR);

		CMenu menu, *popup;
		//获取句柄
		menu.LoadMenu(IDR_MENU_xxx_菜单ID)
			popup = menu.GetSubMenu(0);
		//位置信息
		CPoint point;
		ClientToScreen(&point);
		GetCursorPos(&point);
		popup->TrackPopupMenu(TPM_LEFTALIGN | TPM_RIGHTBUTTON, point.x, point.y, this);
		*pResult = 0;
	}

## 点击菜单项实现功能

点击弹出的菜单可实现删除、添加或打开别的窗口等功能(手动添加)。

   xxx.h

	afx_msg void OnViewlog();
	afx_msg void OnUpdateViewlog(CCmdUI *pCmdUI);

   xxx.cpp

 
	//在cpp前面添加消息
	ON_COMMAND(ID_菜单项的ID, &xxx::OnViewlog)
	ON_UPDATE_COMMAND_UI(ID_菜单项的ID, &xxx::OnUpdateViewlog)
		
	void xxx::OnViewlog()
	{
		//获取选中行号
		POSITION pos = m_list.GetFirstSelectedItemPosition();
		LVITEM item;
		ZeroMemory(&item, sizeof(item));
		item.mask = LVIF_INDENT|LVIF_PARAM;
		item.iItem = m_list.GetNextSelectedItem(pos);
		
		//处理该菜单对应的功能
		xxx dlg;
		dlg.DoModal();
	
	}
	void xxx::OnUpdateViewlog(CCmdUI *pCmdUI)
	{
		//处理菜单对应的用户界面显示状态
	}

## 双击事件

双击List列表的某一个字段时可以产生一个事件进行一些变更操作。

xxx.h

	afx_msg void OnNMDblclkListNotAdjust(NMHDR *pNMHDR, LRESULT *pResult);
	
xxx.cpp

	ON_NOTIFY(NM_DBLCLK, IDC_LIST_xxx, &xxx::OnNMDblclkListNotAdjust)
	
	//双击事件
	void xxx::OnNMDblclkListNotAdjust(NMHDR *pNMHDR, LRESULT *pResult)
	{
		LPNMITEMACTIVATE pNMItemActivate = reinterpret_cast<LPNMITEMACTIVATE>(pNMHDR);
		*pResult = 0;
		
		int iSubItem = pNMItemActivate->iSubItem； //双击的列号
		int iItem = pNMItemActivate->iItem；  //双击的行号
		
	}

# List根据路径获取对用的Icon插入到List中

## 获取Icon函数
	
	MapSysIconInfo m_mapSysIconInfo;
	CImageList m_ilIcons;
	
	int CFileTimeMachineClientDlg::OnAddIcon(CString strFileType)  //接受文件后缀
	{
		if (strFileType.GetLength() >= 15)
		{
			strFileType = _T("eff-soft");
		}
		int iImageIndex = 0;
		CString strText = _T("foo.");
		int iPos = strFileType.Find(_T(";"));
		if (iPos>0)
		{
			strFileType = strFileType.Left(iPos);
		}
		strText += strFileType;

		MapSysIconInfo::iterator iter = m_mapSysIconInfo.find(string(CW2A(strText)));
		if (iter != m_mapSysIconInfo.end())
		{
			//已经存在了,忽略
			iImageIndex = iter->second;						
		}
		else
		{
			SHFILEINFO shfi;
			DWORD_PTR hr=0;
			memset(&shfi,0,sizeof(shfi));
			hr = SHGetFileInfo(strText,
				FILE_ATTRIBUTE_NORMAL,
				&shfi,
				sizeof(shfi),
				SHGFI_ICON|SHGFI_USEFILEATTRIBUTES);

			if (hr)
			{
				VERIFY(shfi.hIcon);	
				iImageIndex = m_ilIcons.Add(shfi.hIcon);
				if (iImageIndex >= 0)
				{
					m_mapSysIconInfo.insert(std::make_pair(string(CW2A(strText)), iImageIndex));
				}	
			}

		}

		return iImageIndex;

	}

## 构造函数

	m_ilIcons.Create(16, 16, ILC_MASK|ILC_COLOR32, 3,3);
	
## 析构函数
	
	for (int i = m_ilIcons.GetImageCount()-1; i>=0;i--)
	{//删除显示的ico
		m_ilIcons.Remove(i);
	}

## 调用
	
	
	m_listFileTime.SetImageList(&m_ilIcons, 位置);
	
	CString strFilePath； //文件路径
	int iPos = strFilePath.ReverseFind('.');
	CString strIcon = strFilePath.Right(strFilePath.GetLength()-iPos-1);
	int iImageIndex = OnAddIcon(strFilePath);//获取文件的ico
	m_list.SetItem(nRow, 0, LVIF_IMAGE, strFileName, iImageIndex, NULL, NULL, NULL);


## 蒲公英 -- 无法停留的爱
