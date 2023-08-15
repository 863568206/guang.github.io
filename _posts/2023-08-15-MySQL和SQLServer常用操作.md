---
layout:     post
title:      MySQL和SQLServer常用操作
subtitle:   SQL
date:       2023-08-15
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - SQL
---

## 建表

SQLServer：

	IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[CLIENT_TABLE_]') AND type in (N'U'))
	BEGIN
	CREATE TABLE [dbo].[CLIENT_TABLE_](
		[index] [int] IDENTITY(1,1) NOT NULL,
		[FileName] [nvarchar] (1024) NULL,
		[ProcessName] [nvarchar] (1024) NULL,
		[ProofTestValue] [nvarchar] (1024) NULL,
		[Signature] [nvarchar] (1024) NULL,
		[MD5] [nvarchar] (1024) NULL,
		[PrivilegeIndex] int NULL,
		[Remark] [nvarchar] (1024) NULL,
	 CONSTRAINT [PK_CLIENT_TABLE_] PRIMARY KEY CLUSTERED 
	(
		[index] ASC
	)WITH (PAD_INDEX  = OFF, IGNORE_DUP_KEY = OFF) ON [PRIMARY]
	) ON [PRIMARY]
	END

Mysql：

	CREATE TABLE IF NOT EXISTS `CLIENT_TABLE_` (
	  `index` int(11) unique key  AUTO_INCREMENT,
	  `PrivilegeIndex` int(11) DEFAULT NULL,
	  `SettingName` varchar(150) DEFAULT NULL,
	  `IPOrDomainList` varchar(2048) DEFAULT NULL,
	  `CtrlPort` varchar(1024) DEFAULT NULL,
	  `UploadDecryptType` varchar(1024) DEFAULT NULL,
	  `DownLoadEncryptType` varchar(1024) DEFAULT NULL,
	  `ProcessName` varchar(1024) DEFAULT NULL,
	  `AllowAccessOtherNet`  boolean
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;


## 插入字段

SQLServer：

	if   not   exists(select   1   from   syscolumns   where   id=object_id('CLIENT_TABLE_')   and   name='NewValue')   
	      alter   table   CLIENT_TABLE_   add   NewValue   nvarchar (max)  
	
	if   not   exists(select   1   from   syscolumns   where   id=object_id('CLIENT_TABLE_')   and   name='NewBit')   
	      alter   table   CLIENT_TABLE_   add   NewBit  bit not null default (0)  

Mysql：

	IF NOT EXISTS (SELECT * FROM information_schema.COLUMNS WHERE table_schema=CurrentDatabase AND TABLE_NAME = 'CLIENT_TABLE_' AND COLUMN_NAME = 'NewValue') THEN  
	  alter   table   CLIENT_TABLE_   add   NewValue   int default 5;
	END IF;
	
	IF NOT EXISTS (SELECT * FROM information_schema.COLUMNS WHERE table_schema=CurrentDatabase AND TABLE_NAME = 'CLIENT_TABLE_' AND COLUMN_NAME = 'NewVarchar') THEN  
	  alter   table   CLIENT_TABLE_   add   NewVarchar   varchar(128) ;
	END IF;
	
	IF NOT EXISTS (SELECT * FROM information_schema.COLUMNS WHERE table_schema=CurrentDatabase AND TABLE_NAME = 'CLIENT_TABLE_' AND COLUMN_NAME = 'Newbit') THEN  
	  alter   table   CLIENT_TABLE_   add   Newbit   bit default false ;
	END IF;
  
