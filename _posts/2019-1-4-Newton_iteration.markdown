---

layout:     post
title:      "Newton iteration"
subtitle:   " \"牛顿迭代实现Sqrt\""
date:       2019-01-04 10:39:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 算法学习

---

	实现 int sqrt(int x) 函数。

	计算并返回 x 的平方根，其中 x 是非负整数。

	由于返回类型是整数，结果只保留整数的部分，小数部分将被舍去。

	示例 1:

		输入: 4

		输出: 2


	示例 2:

		输入: 8

		输出: 2

	说明: 8 的平方根是 2.82842..., 

	由于返回类型是整数，小数部分将被舍去。

	 
算法实现

	class Solution:

		def mySqrt(self, x):

			if x<=1:
			
				return x
				
			r = x
			
			while r>x / r:
			
				r = (r+x/r) / 2
				
			return int(r)
		
总结：使用牛顿迭代法实现非负整数开方。

---


