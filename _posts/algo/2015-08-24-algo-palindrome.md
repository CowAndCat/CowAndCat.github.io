---
layout: post
title: Palindrome
category: algorithm
comments: false
---
## 回文检测相关问题和算法

**回文（Palindrome）**是指从正序和逆序读取都是一样的文本段。  
如aa, abccba等。

1.检测字符串是否为回文:

	public boolean isPalindrome(String str){
		//首位依次比较
		int begin=0, end=str.length()-1;
		while(end>begin){
			if(str.charAt(begin)!=str.charAt(end))
				return false;
			begin++;
			end--;
		}
		return true;
	}

2.求一个数字是否是回文数:  

	public boolean isPalindrome(long num){
		//求出逆序数
		long num2=0, tmp=num;
		while(tmp!=0){
			num2 = num2*10+tmp%10;
			tmp = tmp/10;
		}
		if(num2==num) return true;
		else return false;
	}

3.求解最长的回文子字符串：  
	思路： manacher算法。  
    算法充分利用了字符匹配的特殊性，避免了大量不必要的重复匹配。这个算法有一个很巧妙的地方，它把奇数的回文串和偶数的回文串统一起来考虑了。  
    算法大致过程是这样。先在每两个相邻字符中间插入一个分隔符，当然这个分隔符要在原串中没有出现过。一般可以用‘#’分隔。这样就非常巧妙的将奇数长度回文串与偶数长度回文串统一起来考虑了（见下面的一个例子，回文串长度全为奇数了），然后用一个辅助数组P记录以每个字符为中心的最长回文串的信息。P［id］记录的是以字符str［id］为中心的最长回文串，当以str［id］为第一个字符，这个最长回文串向右延伸了P［id］个字符。  

	    原串：    w aa bwsw f d  
	    新串：   # w# a # a # b# w # s # w # f # d #  
	    辅助数组P：  1 2 1 2 3 2 1 2 1 2 1 4 1 2 1 2 1 2 1  
	
 P［id］-1就是该回文子串在原串中的长度（包括‘#’）  
	 由于这个算法是线性从前往后扫的。那么当我们准备求P［i］的时候，i以前的P［j］我们是已经得到了的。我们用mx记在i之前的回文串中，延伸至最右端的位置。同时用id这个变量记下取得这个最优mx时的id值。

	    void pk()
		{
		    int i;
		    int mx = 0;
		    int id;
		    for(i=1; i<n; i++)
		    {
		        if( mx > i )
		            p[i] = MIN( p[2*id-i], mx-i );        
		        else
		            p[i] = 1;
		        for(; str[i+p[i]] == str[i-p[i]]; p[i]++)
		            ;
		        if( p[i] + i > mx )
		        {
		            mx = p[i] + i;
		            id = i;
		        }
		    }
		}

4.判断一个单链链表是否构成回文结构  
解法一：两次遍历链表，第一次把链表值存入一个list，第二次从list的逆序和从链表的正序比较值。time & space 复杂度：O(n).

解法二：两次遍历链表，第一次找出中间位置，然后翻转mid到end之间的链表，第二次的遍历循环size/2次。能达到O(1)的空间复杂度。
附上代码：

    public boolean isPalindrome(ListNode head) {
        if(head==null) return true;
    	//利用追赶法找出中间位置，并将中间位置的前一个节点的next置为null
    	ListNode slow = head, fast = head, preslow=slow;
    	while(fast!=null){    		
    		fast = fast.next;
    		if(fast==null) {
    			break;
    		}
    		preslow = slow;
    		slow = slow.next;
    		fast = fast.next;
    	}
    	preslow.next = null;
    	//slow就是中间位置

    	ListNode head2 = reverseList(slow); //翻转链表

        while(head!=null && head2!=null){
        	if(head.val != head2.val) return false;
        	head = head.next;
        	head2 = head2.next;
        }
        return true;
    }

	public ListNode reverseList(ListNode head) {
    	if(head==null || head.next==null){
    		return head;
    	}
    	ListNode p1 = new ListNode(-1);
    	ListNode p2 = head;
    	p1.next=head;

		ListNode p3 = p2.next;
    	if(p3==null) return p1.next;
    	ListNode p4 = p3.next;
    	ListNode p5 = p2;
    	while(p3!=null){
    		p2.next = p4;
    		p3.next = p5;
    		p1.next = p3;

    		p3 = p2.next;
    		if(p3==null) return p1.next;
    		p4 = p3.next;
    		p5 = p1.next;
    	}
    	return p1.next;    	
    }

后面遇到相关问题再补上。
