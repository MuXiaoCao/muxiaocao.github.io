---
title: JAVA编程——二维递增数组的查找
date:  2016-09-01
categories:  Java编程
keywords: [JAVA,面试题, 查找]
tags:  [JAVA,面试题, 查找]
---

## 题目：
 > 一个n*m的二维数组，每一行从左到右依次递增,每一列从上到下依次递增。问：给你一个数字，如何能快速的输出他在数组中的位置。

-------------------
```
	/**
	 * 一个n*m的二维数组，每一行从左到右依次递增,每一列从上到下依次递增。
	 * 问：给你一个数字，如何能快速的输出他在数组中的位置。
	 */
	public static int[] getPos(int[][] a,int tag) {
		int len = a.length;
		int col = a[0].length;
		/*
		 * 排除极端可能
		 */
		if (a[0][0]>tag || a[len-1][col-1]<tag) {
			return new int[]{-1,-1};
		}
		for (int i = 0,j = col - 1; i < len && j >= 0;) {
			if (a[i][j] == tag) {
				return new int[]{i,j};
			}else if (a[i][j] < tag) {
				i++;
			}else {
				j--;
			}
		}
		return new int[]{-1,-1};
	}
	
	public static void main(String[] args) {
		int[][] a = new int[][]{{1,2,3},{2,3,4},{5,8,9},{7,10,11}};
		int[] result = getPos(a, 6);
		System.out.println(result[0] +","+ result[1]);
		result = getPos(a, 7);
		System.out.println(result[0] +","+ result[1]);
	}
```
时间复杂度：n+m

> **注意：**转载请说明，来自**[转自itboy-木小草](http://muxiaocao.cn)**，**尊重原创，尊重技术**。