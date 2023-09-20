---
layout:     post
title:      lamda表达式与find_if
subtitle:   c++
date:       2023-09-20
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - c++
---

# lamda表达式

lamda语法

	[捕获列表](参数){ 逻辑语句 }

如：
	
	int a = 6;
	[a](int x, int y){ return x + y + a; }



## lamda表达式与find\_if

find\_if函数 带条件的查找元素，可以通过find\_if函数来实现查找满足特定条件的元素，将lamda表达式作为查找条件。

例：用find_if去查找vector中保存符合条件结构体的值

	typedef struct _LAMBDA_TEST
	{
		int a;
		int b;
	
	} LAMBDA_TEST, * PLAMBDA_TEST;

	void LambdaAndFindIfTest()
	{
		int iTest = 6;
		std::vector<LAMBDA_TEST> vecLamebda = { {7, 8}, {9, 20}, {5, 9} };
	
		std::vector<LAMBDA_TEST>::iterator iter = std::find_if(vecLamebda.begin(), vecLamebda.end(), [iTest](LAMBDA_TEST lambdaTest)
		{
			return lambdaTest.a < iTest && lambdaTest.b > iTest;  //返回是否符合条件
		}
		);
	
		if (iter != vecLamebda.end())
		{
			//找到了
			std::cout << iter->a << iter->b << std::endl;
	
		}
	}

不然的话，可能要循环vector一个个比较。