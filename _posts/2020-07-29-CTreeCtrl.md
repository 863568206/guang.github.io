---
layout:     post
title:      CTreeCtrl
subtitle:   MFC
date:       2020-07-29
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - MFC
---

## CTreeCtrl常用函数

[CTreeCtrl常用函数](https://blog.csdn.net/zhaiwenjuan/article/details/6443322?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

HTREEITEM hItem=GetRootItem(); //获取根结点,可能会有多个根结点

ItemHasChildren(hParent) //判断结点是否有孩子结点

hItem=GetChildItem(hParent); //获取第一个子结点

hItem=GetNextSiblingItem(hItem); //获取下一个兄弟结点结点

BOOL CTreeCtrl::ItemHasChildren(HTREEITEM hItem); //查询hItem是否有子项

HTREEITEM CTreeCtrl::GetChildItem(HTREEITEM hItem); //取得hItem 第一个子项的句柄

HTREEITEM CTreeCtrl::GetPrevSiblingItem(HTREEITEM hItem); //查询排在hItem前兄弟项

HTREEITEM CTreeCtrl::GetNextSiblingItem(HTREEITEM hItem); //查询排在hItem后兄弟项

## GetNextItem参数

TVGN_CARET 获取当前被选择的项。
 
TVGN_CHILD 获取第一个子项。hItem参数必须是NULL。
 
TVGN_DROPHILITE 获取是一次拖放操作的目标的项。
 
TVGN_FIRSTVISIBLE 获取第一个可见的项。
 
TVGN_TEXT 获取下一个兄弟项。 

TVGN_NEXTVISIBLE 获取跟随在指定项之后的下一个可视项。 

TVGN_PARENT 获取指定项的父项。 

TVGN_PREVIOUS 获取前一个兄弟项。
 
TVGN_PREVIOUSVISIBLE 获取在指定项之前的第一个可视项。 

TVGN_ROOT 获取根项的第一个子项，指定项是该根项的一个部分。 

## CTreeCtrl生成文件树状图
	
CTreeCtrl生成文件树状图。	

	void FilePath(CString strFindPath, HTREEITEM hItem/* = NULL*/, BOOL bInsertData/* = TRUE*/)
	{
		BOOL bFileExist = FALSE;
		if (hItem == NULL) //一开始获取根节点
		{
			HTREEITEM hRoot = m_treePath.GetRootItem();
			if (hRoot == NULL)
			{
				hRoot = m_treePath.InsertItem(L"此电脑");
			}
			hItem = hRoot;
		}
	
		CString strPath;
		int iLocation = strFindPath.Find('\\');
		if (iLocation != -1)
		{
			strPath = strFindPath.Mid(0, iLocation);
			strFindPath = strFindPath.Mid(iLocation + 1);
			bFileExist = TRUE;
		}
		else
		{	
			strPath = strFindPath;
			bFileExist = FALSE;
		}
	
		//循环遍历当前节点下的叶子节点是否和插入的重复GetNextSiblingItem
		HTREEITEM hBrotherItem = m_treePath.GetNextSiblingItem(hItem); //获取下一个节点的兄弟节点
		HTREEITEM hNextItem = m_treePath.GetChildItem(hItem); //获取子节点
	
		while (bInsertData)
		{
			if (hNextItem == hBrotherItem)
			{
				bInsertData = FALSE;
				break;
			}
	
			if (hNextItem != NULL)
			{
				CString strItemText = m_treePath.GetItemText(hNextItem);
				if (strItemText == strPath)
				{
					break;
				}
			}
	
			hNextItem = m_treePath.GetNextItem(hNextItem, TVGN_NEXTVISIBLE); //这个只会获取同级别的下一个节点
		}
	
		if (bInsertData)
		{
			hItem = hNextItem;
		}
		else
		{
			hItem = m_treePath.InsertItem(strPath, hItem);
		}
		
		if (bFileExist)
		{
			FilePath(strFindPath, hItem, bInsertData);
		}
		return;
	}

## CTreeCtrl点击消息
	
消息：

	ON_NOTIFY(NM_CLICK, IDC_TREE_xxxx, &xxxxxxx::OnNMClickTree1)

消息函数：

	void xxxx::OnNMClickTree1(NMHDR* pNMHDR, LRESULT* pResult)
	{
		CTreeCtrl *ctreectrl = (CTreeCtrl *)GetDlgItem(IDC_TREE1);
		
		CPoint pt = GetCurrentMessage()->pt;
		ctreectrl->ScreenToClient(&pt); 
		
		UINT uFlags = 0;  
		HTREEITEM hItem = ctreectrl->HitTest(pt, &uFlags);
	
		if ((hItem != NULL) && (TVHT_ONITEM & uFlags))//如果点击的位置是在节点位置上
		{  
			HTREEITEM hRoot = m_treePath.GetRootItem();  //获取根节点
	
		}
	
		*pResult = 0;
	}

## 获取我的电脑、我的文档、桌面等图标添加
	
构造函数：

	//获取计算机图标
	SHFILEINFO sfi_root;
	LPITEMIDLIST pidl = NULL;
	SHGetSpecialFolderLocation(NULL, CSIDL_DRIVES, &pidl);
	SHGetFileInfo((LPCTSTR) pidl, 0, &sfi_root, sizeof(sfi_root), SHGFI_DISPLAYNAME|SHGFI_ICON | SHGFI_PIDL);//如果只获取图标，可以去掉				SHGFI_DISPLAYNAME
	HICON hIcon  = sfi_root.hIcon;
	ASSERT(hIcon);
	m_ilTreeIcons.Add(sfi_root.hIcon); //0

	SHFILEINFO sfi_drive;
	//获取驱动器图标
	wchar_t driver_name[4] = L"C:\\";
	::SHGetFileInfo(driver_name,0, &sfi_drive, sizeof(sfi_drive), SHGFI_DISPLAYNAME|SHGFI_ICON);
	m_ilTreeIcons.Add(sfi_drive.hIcon); //1
		
	//路径
	hIcon = AfxGetApp()->LoadIcon(IDI_ICON_OPEN_FOLDER);//2
	m_ilTreeIcons.Add (hIcon);
		
	//桌面
	wchar_t path_Desktop[512]={0};
	SHGetSpecialFolderPath(0,path_Desktop,CSIDL_DESKTOPDIRECTORY,0); 
	SHFILEINFO sfi_DeskTop;
	::SHGetFileInfo(path_Desktop,0, &sfi_DeskTop, sizeof(sfi_DeskTop),SHGFI_ICON);
	m_ilTreeIcons.Add(sfi_DeskTop.hIcon);  //3
		
	//我的文档
	wchar_t path_Document[512] = {0};
	SHGetSpecialFolderPath(0,path_Document,CSIDL_MYDOCUMENTS,0); 
	SHFILEINFO sfi_Document;
	::SHGetFileInfo(path_Document,0, &sfi_Document, sizeof(sfi_Document),SHGFI_ICON);
	m_ilTreeIcons.Add(sfi_Document.hIcon); //4
	
初始化函数：

	m_TreePath.ModifyStyle(0, WS_VISIBLE|TVS_HASBUTTONS|TVS_HASLINES|TVS_LINESATROOT|TVS_SHOWSELALWAYS|TVS_TRACKSELECT);
	m_TreePath.SetImageList(&m_ilTreeIcons, TVSIL_NORMAL);
	
	//插入
	hRoot = m_TreePath.InsertItem(L"此电脑");
	//m_TreePath.InsertItem(L"桌面", hRoot);
	//m_TreePath.InsertItem(L"我的文档",hRoot);
	m_TreePath.InsertItem(L"桌面", 3, 3, hRoot, TVI_LAST);
	m_TreePath.InsertItem(L"我的文档", 4, 4, hRoot, TVI_LAST);

## 圣诞蔷薇 -- 追忆的爱情
