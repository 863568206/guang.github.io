---
layout:     post
title:      MFC-ListCotrol右键菜单与双击事件
subtitle:   MFC
date:       2019-07-26
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - MFC
---

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



## 蒲公英 -- 无法停留的爱
