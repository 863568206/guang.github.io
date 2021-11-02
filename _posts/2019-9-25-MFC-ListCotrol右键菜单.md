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

# 获取选中行

	int iItem = 0;
	for ( iItem = m_list.GetItemCount(); iItem >= 0; iItem--)
	{
		if ( LVIS_SELECTED == m_listFTM.GetItemState(iItem, LVIS_SELECTED))     //发现选中行
		{
			break;
		}
	} 


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
	m_list.SetItem(nRow, 0, LVIF_TEXT | LVIF_IMAGE  | LVIF_STATE, strFileName, iImageIndex, NULL, NULL, NULL);

# 获取磁盘、桌面、我的文档等图标添加（Xp 无效）

	//获取计算机图标
	SHFILEINFO sfi_root;
	LPITEMIDLIST pidl = NULL;
	SHGetSpecialFolderLocation(NULL, CSIDL_DRIVES, &pidl);
	SHGetFileInfo((LPCTSTR) pidl, 0, &sfi_root, sizeof(sfi_root), SHGFI_DISPLAYNAME|SHGFI_ICON | SHGFI_PIDL);//如果只获取图标，可以去掉SHGFI_DISPLAYNAME
	
	HICON computer = sfi_root.hIcon;
	m_ImageList.Add(sfi_root.hIcon);

	//获取驱动器图标
	SHFILEINFO sfi_drive;
	wchar_t driver_name[4] = L"C:\\";
	::SHGetFileInfo(driver_name,0, &sfi_drive, sizeof(sfi_drive), SHGFI_DISPLAYNAME|SHGFI_ICON);
	m_ImageList.Add(sfi_drive.hIcon);
	
	//桌面图标
	wchar_t path_Desktop[512]={0};
	SHGetSpecialFolderPath(0,path_Desktop,CSIDL_DESKTOPDIRECTORY,0); 
	SHFILEINFO sfi_DeskTop;
	::SHGetFileInfo(path_Desktop,0, &sfi_DeskTop, sizeof(sfi_DeskTop),SHGFI_ICON);
	m_ImageList.Add(sfi_DeskTop.hIcon);
	
	//我的文档图标
	wchar_t path_Document[512] = {0};
	SHGetSpecialFolderPath(0,path_Document,CSIDL_MYDOCUMENTS,0); 
	SHFILEINFO sfi_Document;
	::SHGetFileInfo(path_Document,0, &sfi_Document, sizeof(sfi_Document),SHGFI_ICON);
	m_ImageList.Add(sfi_Document.hIcon);
	
	HICON m_hIcon;
	SetIcon(m_hIcon, TRUE);			// 设置大图标
	SetIcon(m_hIcon, FALSE);		// 设置小图标

	m_List.SetExtendedStyle(LVS_EX_FULLROWSELECT|LVS_EX_GRIDLINES);

	m_List.SetImageList(&m_ImageList, LVSIL_NORMAL);
	m_List.SetImageList(&m_ImageList, LVSIL_SMALL);


	m_List.InsertItem(0,_T("我的电脑"),0);
	m_List.InsertItem(1,_T("C盘"),1);
	m_List.InsertItem(2,_T("桌面"),2);
	m_List.InsertItem(3,_T("我的文档"),3);
	
# 虚拟列表设置

ListCotrol列表提供了虚拟列表用来提高列表数据的显示速度。[参考](https://blog.csdn.net/daiafei/article/details/6825034)

## 声明
  	
	FILE_LIST_INFO //结构体

	CListCtrl m_list;
	afx_msg void GetDispInfo(NMHDR* pNMHDR, LRESULT* pResult);   //显示事件
	afx_msg void OnNMClickListClick(NMHDR* pNMHDR, LRESULT* pResult); //Checked更新
	CArray<FILE_LIST_INFO, FILE_LIST_INFO> m_arInsertListInfo;   //保存List数据的缓冲区
	void xxxDlg::SortColumn(int iCol, bool bAsc)； //行号，排序标志，点击表头排序比较函数


## 事件

	ON_NOTIFY(LVN_GETDISPINFO, IDC_LIST_xxx, &xxxDlg::GetDispInfo)
	ON_NOTIFY(NM_CLICK, IDC_LIST_xxx, &xxxDlg::OnNMClickListClick)

## 定义

GetDispInfo：：

	void xxxDlg::GetDispInfo(NMHDR* pNMHDR, LRESULT* pResult) 
	{
		LV_DISPINFO* pDispInfo = (LV_DISPINFO*)pNMHDR;
		LV_ITEM* pItem= &(pDispInfo)->item;
		int itemid = pItem->iItem; //行号
		FILE_LIST_INFO rListInfo = m_arInsertListInfo.ElementAt(pItem->iItem);
	
		if (pItem->mask & LVIF_TEXT)  //内容
		{
			switch(pItem->iSubItem)
			{
			case 0:
				{
					lstrcpy(pItem->pszText, rListInfo.strFileName);
				}
				break;
			case 1:
				{
					lstrcpy(pItem->pszText, rListInfo.strFilePath);
				}			
				break;
			case 2:
				{
					lstrcpy(pItem->pszText, rListInfo.strFileSize);
				}
				break;
			default:
				//ASSERT(0);
				break;
			}
		}
	
		if (pItem->mask & LVIF_IMAGE)   //显示图片
		{
			// then display the appropriate column
			switch(pItem->iSubItem)
			{
			case 0:
				{
					int iImageIndex = 0;
					CString strFilePath = rListInfo.strFileClientPath;
					int iPos = strFilePath.ReverseFind('.');
					CString strIcon = strFilePath.Right(strFilePath.GetLength()-iPos-1);
					iImageIndex = OnAddIcon(strIcon);//获取文件的ico
					
					pItem->iImage = iImageIndex;
				}
				break;
			default:
				//ASSERT(0);
				break;
			}
		}
	
		//添加复选框
		pItem->mask |= LVIF_STATE;
	
		pItem->stateMask = LVIS_STATEIMAGEMASK;
	
		if(rListInfo.bPitchOnFlag)//判断结构中保存的当前行的选中状态
		{
			pItem->state |=   INDEXTOSTATEIMAGEMASK(2);   //取你自己保存的state状态,   需要用到INDEXTOSTATEIMAGEMASK 
		}
		else
		{
			pItem->state |=   INDEXTOSTATEIMAGEMASK(1);   //未选中
		}
	
		*pResult = 0;
	}

OnNMClickList（复选框点击事件）：

	void CFileTimeMachineClientDlg::OnNMClickListClick(NMHDR* pNMHDR, LRESULT* pResult)
	{
		NMLISTVIEW* pNMListView = (NM_LISTVIEW*)pNMHDR;
	
		LVHITTESTINFO hitinfo;
	
		hitinfo.pt = pNMListView->ptAction;
		int item = m_listFileTime.HitTest(&hitinfo);
	
		if(item != -1)
		{
	
			//看看鼠标是否单击在 check box上面了?
			if( (hitinfo.flags & LVHT_ONITEMSTATEICON) != 0)
			{
				m_arInsertListInfo[item].bPitchOnFlag = !m_arInsertListInfo[item].bPitchOnFlag;//改变状态
				m_listFileTime.RedrawItems(item, item); //重绘当前项 		
			}
		}
	
		*pResult = 0;
	}


写入缓冲区数据：
	
	//这段将数据写入到 m_arInsertListInfo 中
	FILE_LIST_INFO Info;	
	int iCount = m_arInsertListInfo.GetSize(); 
	m_arInsertListInfo.SetAtGrow(iCount, (*pInfo)); //写入数据

	//设置显示多少行,UpdateList, 用来更新列表数据
	::AfxGetApp()->DoWaitCursor(1);
	int iTotalSize = m_arInsertListInfo.GetSize();
	m_listFileTime.SetItemCountEx(iTotalSize, LVSICF_NOINVALIDATEALL);
	m_listFileTime.Invalidate();
	::AfxGetApp()->DoWaitCursor(0);

一般都是将数据全部写入到 m_arInsertListInfo 中后，通过后面的更新List数据代码来更新List。

添加排序：
	
	int __cdecl CmpBackUpFileName(const void *elem1, const void *elem2)
	{
		FILE_LIST_INFO p1 = (BACKUP_FILE_INSERT_LIST_INFO*)elem1;
		FILE_LIST_INFO p2 = (BACKUP_FILE_INSERT_LIST_INFO*)elem2;
	
		return (p1->strFileName.CompareNoCase(p2->strFileName));
	}
	
	int __cdecl CmpBackUpFilePath(const void *elem1, const void *elem2)
	{
		FILE_LIST_INFO p1 = (BACKUP_FILE_INSERT_LIST_INFO*)elem1;
		FILE_LIST_INFO p2 = (BACKUP_FILE_INSERT_LIST_INFO*)elem2;
	
		return (p1->strFileClientPath.CompareNoCase(p2->strFileClientPath));
	}
	
	int __cdecl CmpBackUpFileSize(const void *elem1, const void *elem2)
	{
		FILE_LIST_INFO p1 = (BACKUP_FILE_INSERT_LIST_INFO*)elem1;
		FILE_LIST_INFO p2 = (BACKUP_FILE_INSERT_LIST_INFO*)elem2;
		
		if (p1->iFileSize == p2->iFileSize)
		{
			return 0;
		}
		else if (p1->iFileSize < p2->iFileSize)
		{
			return -1;
		}
		else if (p1->iFileSize > p2->iFileSize)
		{
			return 1;
		}
	
		return 0;
		//return (p1->strFileClientPath.CompareNoCase(p2->strFileClientPath));
	}

	void xxxDlg::SortColumn(int iCol, bool bAsc) //行号，排序标志
	{
		switch(iCol)
		{
		case 0:
			{
				::AfxGetApp()->DoWaitCursor(TRUE);
				if (bAsc)
				{
					qsort( static_cast<void*>(&m_arInsertListInfo[0]), m_arInsertListInfo.GetSize(), sizeof(FILE_LIST_INFO), CmpBackUpFileName);
				}
				else
				{
					qsort( static_cast<void*>(&m_arInsertListInfo[0]), m_arInsertListInfo.GetSize(), sizeof(FILE_LIST_INFO), CmpBackUpFileName);
				}			
				::AfxGetApp()->DoWaitCursor(FALSE);
			}
			break;
		case 1:
			{
				::AfxGetApp()->DoWaitCursor(TRUE);
				if (bAsc)
				{
					qsort( static_cast<void*>(&m_arInsertListInfo[0]), m_arInsertListInfo.GetSize(), sizeof(FILE_LIST_INFO), CmpBackUpFilePath);
				}
				else
				{
					qsort( static_cast<void*>(&m_arInsertListInfo[0]), m_arInsertListInfo.GetSize(), sizeof(FILE_LIST_INFO), CmpBackUpFilePath);
				}			
				::AfxGetApp()->DoWaitCursor(FALSE);
			}
			break;
		case 2:
			{
				::AfxGetApp()->DoWaitCursor(TRUE);
				if (bAsc)
				{
					qsort( static_cast<void*>(&m_arInsertListInfo[0]), m_arInsertListInfo.GetSize(), sizeof(FILE_LIST_INFO), CmpBackUpFileSize);
				}
				else
				{
					qsort( static_cast<void*>(&m_arInsertListInfo[0]), m_arInsertListInfo.GetSize(), sizeof(FILE_LIST_INFO), CmpBackUpFileSize);
				}			
				::AfxGetApp()->DoWaitCursor(FALSE);
			}
			break;
		default:
			break;
		}
		
		UpDateListInfo();  //更新List
	}

	


	typedef stdext::hash_map<string,int> MapSysIconInfo;


	//弹出菜单设置
	void CFileTimeMachineCustomizeBackupSettingsDlg::OnNMRClickListCotrol(NMHDR *pNMHDR, LRESULT *pResult)
	{
		// TODO: 在此添加控件通知处理程序代码
		do 
		{
			LPNMITEMACTIVATE pNMItemActivate = reinterpret_cast<LPNMITEMACTIVATE>(pNMHDR);
			CMenu menu, *popup;

			menu.LoadMenu(IDR_MENU_FILE_TIME_MACHINE_CUSTOMIZE_BACKUP_SETTINGS);
			CPoint point;

			GetCursorPos(&point);
			popup = menu.GetSubMenu(0);
			CXTPCommandBars::TrackPopupMenu(popup, 0, point.x, point.y, this);

		} while (FALSE);

		*pResult = 0;
		return;
	}
	
# 将列表的文本复制到剪贴板

List本身并不支持剪贴板的复制，所以我们加一个菜单将他复制到剪贴板上：

	void xxx::OnViewlog()  //菜单事件
	{
		//获取选中行号
		POSITION pos = m_list.GetFirstSelectedItemPosition();
		LVITEM item;
		ZeroMemory(&item, sizeof(item));
		item.mask = LVIF_INDENT|LVIF_PARAM;
		item.iItem = m_list.GetNextSelectedItem(pos);
	
		//处理该菜单对应的功能
		CString strInfo;
		strInfo = m_pListTemp->GetItemText(item.iItem, 5);   //获取复制内容
	
		if (!strInfo.IsEmpty())
		{
			if (!OpenClipboard()) //打开剪贴板
			{
				CString strMsg = _T("打开剪贴板错误！");
				return;			
			}
			
			if(!EmptyClipboard()) //清空剪贴板
			{
				CString strMsg = _T("清空剪贴板失败！");
				return;	
			}		
	
			LPTSTR  lptstrCopy; 
			HGLOBAL  hglbCopy = GlobalAlloc(GMEM_MOVEABLE, (strInfo.GetLength() + 1) * sizeof(TCHAR)); 
			if (hglbCopy == NULL) 
			{ 
				CloseClipboard(); 
				return;
			} 
	
			// Lock the handle and copy the text to the buffer. 
	
			lptstrCopy = (LPTSTR)GlobalLock(hglbCopy); 
			_tcscpy(lptstrCopy, strInfo) ;
			GlobalUnlock(hglbCopy); 
	
			SetClipboardData(CF_UNICODETEXT, hglbCopy);
			CloseClipboard(); 
	
			CString strMsg = _T("复制内容到剪贴板成功！");
		}
	}
	
# CListControl中点击Checkbox事件

在**ListControl**中，要点击其中的行，或者其中的**Checkbox**时，要触发点击事件。

在xxx.h中添加：

	afx_msg void OnClickList(NMHDR* pNMHDR, LRESULT* pResult);
	CListCtrl m_list; //关联列表框的控件变量

在xxx.cpp中添加：

当前的**LVN_ITEMCHANGED**产生消息的时间为当某个项已经发生变化时

	ON_NOTIFY(LVN_ITEMCHANGED, IDC_xxx_列表框, &xxx::OnClickList)

	void  xxx::OnClickList(NMHDR* pNMHDR, LRESULT* pResult)
	{
		DWORD dwPos = GetMessagePos();
		CPoint point(LOWORD(dwPos), HIWORD(dwPos));

		m_list.ScreenToClient(&point);

		LVHITTESTINFO lvinfo;
		lvinfo.pt = point;
		lvinfo.flags = LVHT_ABOVE;

		UINT nFlag;
		int nItem = m_list.HitTest(point, &nFlag); //获取点击的行号

		//判断是否点击的Checkbox
		if (nFlag == LVHT_ONITEMSTATEICON)
		{
			if (m_lists.GetCheck(nItem) == FALSE) 
			{
				//点击未选中的时
			}
			else if (m_list.GetCheck(nItem) == TRUE) 
			{
				//点击选中的时
			}
		}
	}


## 蒲公英 -- 无法停留的爱
