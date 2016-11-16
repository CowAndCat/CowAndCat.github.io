---
layout: post
title: Maximum Sub-Sequence
category: algorithm
comments: false
---
## 找出一组数中的有着最大和的子序列，只要求出最大和的值
算法如下：

	int maxSubSeq(int[] array){
		int res=0;
		int currentSum=0;
		for(int i=0; i<array.length; i++){
			currentSum += array[i];
			if(currentSum>res){
				res=currentSum;
			}else if(currentSum<0){
				currentSum=0;
			}
		}
		return res;
	}
