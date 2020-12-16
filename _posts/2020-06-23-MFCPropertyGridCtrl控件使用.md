---
layout:     post
title:      MFCPropertyGridCtrl控件使用
subtitle:   MFC
date:       2020-06-23
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - MFC
---

## MFCPropertyGridCtrl控件使用

Vs里有大量的属性控件的使用，但是vs2008在工具中并没有这个控件可以拖出来使用，所以我们只能动态创建了。

借鉴：[VS2008下使用 CMFCPropertyGridCtrl](http://blog.csdn.net/sunnyloves/article/details/5655575)

头文件：#include <afxpropertygridctrl.h>

xxx.h

	CMFCPropertyGridCtrl  m_MFCPGCtrl; 
	CMFCPropertyGridProperty* pGroup1; //分组
	CMFCPropertyGridProperty* pGroup2; //分组 每有一个组便添加一个

xxx.cpp
	
在 **OnInitDialog()** 中添加初始化：

	void CMfcGDlg::OnGridCtrl() 
	{
		
		CRect rc;
		GetClientRect(rc);
		
		//下面的注释调为占满设置的主界面
		/* （设置上下左右的位置）
		rc.bottom -= 50; 
		rc.left += 50;
		rc.right -= 50;
		*/

		m_MFCPGCtrl.Create(WS_CHILD | WS_VISIBLE | WS_TABSTOP | WS_BORDER,rc,this, IDD_画到界面的ID);
		m_MFCPGCtrl.EnableHeaderCtrl(true,_T("参数"),_T("值")); //是否启用表头，配置数据
		m_MFCPGCtrl.EnableDescriptionArea();  //是否启用描述功能
		m_MFCPGCtrl.SetVSDotNetLook();
		m_MFCPGCtrl.MarkModifiedProperties(); //是否着重显示修改
		m_MFCPGCtrl.SetAlphabeticMode(false);
	
		m_MFCPGCtrl.SetShowDragContext();
		pGroup1 = new CMFCPropertyGridProperty(_T("参数组1")); //设置组名
		pGroup2 = new CMFCPropertyGridProperty(_T("参数组2")); //设置组名
		
		pGroup1->AddSubItem(new CMFCPropertyGridProperty(_T("参数1"),_T("2.5"),_T("这是参数1的说明"))); //添加组下数据
		pGroup1->AddSubItem(new CMFCPropertyGridProperty(_T("参数2"),_T("3.5"),_T("这是参数2的说明")));
		pGroup2->AddSubItem(new CMFCPropertyGridProperty(_T("参数3"),_T("4.5"),_T("这是参数3的说明")));
		pGroup2->AddSubItem(new CMFCPropertyGridProperty(_T("参数4"),_T("5.5"),_T("这是参数4的说明")));
		
	
		m_MFCPGCtrl.AddProperty(pGroup1); 
		m_MFCPGCtrl.AddProperty(pGroup2);
	
	
		m_MFCPGCtrl.ExpandAll();
	
		return ;
	}

## 添加选择文件按钮(系统选择文件框)

	pGroup3->AddSubItem(new CMFCPropertyGridFileProperty(_T("选择文件"), TRUE, _T("D://defaule.csv"), _T("csv"), NULL, _T("csv Files(*.csv)|*.csv|All Files(*.*)|*.*||"), _T("选择csv文件")));//选择文件按钮

## 选项里添加Combox(下拉菜单) 

	 CMFCPropertyGridProperty* pGroupFrequencyTime = new CMFCPropertyGridProperty(  
		_T("是否启用"),  //属性名 
		_T("禁用"),     //初始
		_T("")); 

	pGroupFrequencyTime->AddOption(_T("启用"));   //插入项
	pGroupFrequencyTime->AddOption(_T("禁用"));     
	pGroupFrequencyTime->AllowEdit(FALSE);  //不允许对选项进行编辑  
	pGroup2->AddSubItem(pGroupFrequencyTime);

## 禁用一条属性

	pGroup2->GetSubItem(1)->Enable(FALSE);

## COleVariant转Cstring

	CString cs = L"haowan";
	_bstr_t bs=cs;
	COleVariant vt(bs);

## Cstring转COleVariant

	COleVariant vt;
	CString cs;
	cs = vt.bstrVal;

## 野百合花语 -- 永远幸福