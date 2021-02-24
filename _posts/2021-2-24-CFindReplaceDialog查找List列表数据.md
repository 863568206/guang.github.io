---
layout:     post
title:      CFindReplaceDialog查找List列表
subtitle:   c++
date:       2021-2-24
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - MFC
---

# CFindReplaceDialog查找List列表

通过调用[CFindReplaceDialog](https://docs.microsoft.com/zh-tw/previous-versions/657yfec6(v=vs.110))来查找List列表里的数据，对每一行的每一列都进行查找。

## 实现

定义：
	
	virtual BOOL PreTranslateMessage(MSG* pMsg);
	afx_msg LONG OnFindReplace(WPARAM wParam, LPARAM lParam);
	void OnListFind();
	BOOL m_bIsFindDialogShow;

设置一个消息：

	static UINT WM_FINDREPLACE = ::RegisterWindowMessage(FINDMSGSTRING);

添加响应：

	ON_REGISTERED_MESSAGE(WM_FINDREPLACE, OnFindReplace)

函数实现：

	 BOOL CxxxDlg::PreTranslateMessage(MSG* pMsg)
	 {
		 if(GetFocus() == GetDlgItem(IDC_LIST))   
		 { 
			 if ((pMsg->message == WM_KEYDOWN))
			 {
				 BOOL bCtrl=::GetKeyState(VK_CONTROL) & 0x8000;
				 if (bCtrl)
				 {
					 switch( pMsg->wParam )
					 {
					 case 'F':
						 {
							OnListFind(); 
						 }
	
						 break;
					 default:
						 break;
					 }
				 }
			 }
		 }
	
		 return CXTResizeDialog::PreTranslateMessage(pMsg);
	 }


	void CxxxDlg::OnListFind()
	 {
		 if (m_bIsFindDialogShow)
		 {
			 return;
		 }
		 CFindReplaceDialog* pDlg = new CFindReplaceDialog(); 
		 pDlg->Create( true, NULL, NULL, FR_DOWN, this ); //创建查找对话框
		 //pDlg->Create( true, NULL, NULL, FR_DOWN, m_FileTypeList.); //创建查找对话框
		 //pDlg->CenterWindow();
		 pDlg->ShowWindow( SW_SHOW ); //显示对话框
		 m_bIsFindDialogShow = TRUE;	
	 }

	LONG CAddEncryptFileSettingDlg::OnFindReplace(WPARAM wParam,LPARAM lParam) 
	 {
		 // 可以用下面这些成员函数取得用户输入
		 CFindReplaceDialog* pDlg = CFindReplaceDialog::GetNotifier(lParam); 
		 CString strFindStrig = pDlg->GetFindString(); //查找串
	
		 BOOL    bDown  = pDlg->SearchDown();//True向上，False向下
	
		 BOOL    bCase  = pDlg->MatchCase();//区分大小写
	
		 BOOL    bMarch = pDlg->MatchWholeWord();//全字匹配
	
		 BOOL    bTerminating  = pDlg->IsTerminating();//关闭对话框
	
		 CString m_FindString = strFindStrig;
	
		 if (bTerminating)
		 {
			 m_bIsFindDialogShow = FALSE;
			 return 0;
		 }
			
		//每列都查找
	
		 if( pDlg->FindNext() ) //查找下一个
		 {
			 BOOL bFind = FALSE;
			 POSITION pos = m_List.GetFirstSelectedItemPosition();
			 int iSel= m_List.GetNextSelectedItem(pos);		
	
			 int iItem = m_header.GetItemCount();
	
			 //先取消已选中的
			 for(int i=0; i<m_List.GetItemCount();i++)
			 {
				 m_List.SetItemState(i,0,LVIS_SELECTED);
			 }
	
			 if (bDown)
			 {
				 if (iSel < 0)
				 {
					 iSel = -1;
				 }
				 for(int i=iSel+1; i<m_List.GetItemCount();i++)
				 {
					 for (int j = 0; j< iItem;j++)
					 {
						 CString strFileName = m_List.GetItemText(i, j);
						 if (bMarch)
						 {
							 if (bCase)
							 {
								 if (strFileName.Compare(m_FindString) == 0)
								 {
									 m_List.SetItemState(i,LVIS_SELECTED,LVIS_SELECTED);
									 m_List.EnsureVisible(i,FALSE);
									 m_List.SetFocus();
									 //break;
									 bFind = TRUE;
									 return 0;
								 }
							 }
							 else
							 {
								 if (strFileName.CompareNoCase(m_FindString) == 0)
								 {
									 m_List.SetItemState(i,LVIS_SELECTED,LVIS_SELECTED);
									 m_List.EnsureVisible(i,FALSE);
									 m_List.SetFocus();
									 //break;
									 bFind = TRUE;
									 return 0;
								 }
							 }						
						 }
						 else 
						 {
							 if (!bCase)
							 {
								 strFileName.MakeUpper();
								 m_FindString.MakeUpper();
							 }
	
							 if (strFileName.Find(m_FindString) != -1)
							 {
								 m_List.SetItemState(i,LVIS_SELECTED,LVIS_SELECTED);
								 m_List.EnsureVisible(i,FALSE);
								 m_List.SetFocus();
								 //break;
								 bFind = TRUE;
								 return 0;
							 }						
						 }
					 }
				 }
			 }
			 else
			 {
				 if (iSel < 0)
				 {
					 iSel = m_List.GetItemCount() +1;
				 }
				 for(int i=iSel-1; i >=0;i--)
				 {				
					 for (int j = 0;j<iItem;j++)
					 {
						 CString strFileName = m_List.GetItemText(i, j);
						 if (bMarch)
						 {
							 if (bCase)
							 {
								 if (strFileName.Compare(m_FindString) == 0)
								 {
									 m_List.SetItemState(i,LVIS_SELECTED,LVIS_SELECTED);
									 m_List.EnsureVisible(i,FALSE);
									 m_List.SetFocus();
									 //break;
									 bFind = TRUE;
									 return 0;
								 }
							 }
							 else
							 {
								 if (strFileName.CompareNoCase(m_FindString) == 0)
								 {
									 m_List.SetItemState(i,LVIS_SELECTED,LVIS_SELECTED);
									 m_List.EnsureVisible(i,FALSE);
									 m_List.SetFocus();
									 //break;
									 bFind = TRUE;
									 return 0;
								 }
							 }						
						 }
						 else 
						 {
							 if (!bCase)
							 {
								 strFileName.MakeUpper();
								 m_FindString.MakeUpper();
							 }
	
							 if (strFileName.Find(m_FindString) != -1)
							 {
								 m_List.SetItemState(i,LVIS_SELECTED,LVIS_SELECTED);
								 m_List.EnsureVisible(i,FALSE);
								 m_List.SetFocus();
								 //break;
								 bFind = TRUE;
								 return 0;
							 }						
						 }
					 }
				 }
			 }
	
			 if (!bFind)
			 {
				 m_FileTypeList.SetItemState(iSel,LVIS_SELECTED,LVIS_SELECTED);
				 m_FileTypeList.EnsureVisible(iSel,FALSE);			 
				 //提示错误框

			 }
		 }
	
		 return 0;
	 }

## 薰衣草 -- 等待爱情