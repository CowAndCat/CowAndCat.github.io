---
layout: post
title: LeetCode
category: algorithm
comments: false
---
## LeetCode 543---- Diameter of Binary Tree

Given a binary tree, you need to compute the length of the diameter of the tree. The diameter of a binary tree is the length of the longest path between any two nodes in a tree. This path may or may not pass through the root.

Example:
Given a binary tree 

	      1
	     / \
	    2   3
	   / \     
	  4   5    

Return 3, which is the length of the path [4,2,1,3] or [5,2,1,3].

Note:The length of path between two nodes is represented by the number of edges between them.

#### 算法分析:
对根节点递归计算左右子树的Diameter，通过和类内的变量 diameter 进行比较，保存较大值。在每一次递归结束后，返回这棵子树的深度，根节点获取了左右子树的深度后，将二者相加就是该根节点下的Diameter。

>Java算法实现：

	/**
	 * Definition for a binary tree node.
	 * public class TreeNode {
	 *     int val;
	 *     TreeNode left;
	 *     TreeNode right;
	 *     TreeNode(int x) { val = x; }
	 * }
	 */
	public class Solution {
	    int diameter = 0;

	    public int diameterOfBinaryTree(TreeNode root) {
	        diameter = 0;
	        getDepth(root);
	        return diameter;
	    }

	    public int getDepth(TreeNode root) {
	        if (root == null) {
	            return -1;
	        }
	        int leftDepth = getDepth(root.left);
	        int rightDepth = getDepth(root.right);
	        int temp = leftDepth + rightDepth + 2;
	        if (temp > diameter) {
	            diameter = temp;
	        }
	        return Math.max(leftDepth, rightDepth) + 1;
	    }
	}

掌握：

1. 递归求树的深度
2. 算法思想


