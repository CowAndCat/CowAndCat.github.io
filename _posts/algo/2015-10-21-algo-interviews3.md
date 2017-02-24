---
layout: post
title: 算法面试题集之好代码
category: algorithm
comments: false
---
代码之美

### 1、Remove Duplicates from Sorted Array
【问题描述】  
Given a sorted array, remove the duplicates in place such that each element appear only once and return the new length.

Do not allocate extra space for another array, you must do this in place with constant memory.

For example,
Given input array nums = [1,1,2],

Your function should return length = 2, with the first two elements of nums being 1 and 2 respectively. It doesn't matter what you leave beyond the new length.

【解答】
这道题主要考验人对问题考虑的是否全面，本人提交了10次，才把边界问题全面处理妥当。

然后在讨论里面看到一个回答，这段代码相比我的，要好n倍，也很巧妙：

	public class Solution {
		public int removeDuplicates(int[] nums) {
			if(nums.length==0)return 0;
			int j=1,index=1;
		    for(int i=0;i<nums.length-1;i++){
		        if(nums[i+1]!=nums[i]){
		           nums[j++]=nums[i+1];
		           index++;
		        }
		    }
		    return index;
		}
	} //去掉index变量，最后返回j也是可以的。

### 2、计算n!末尾所包含0的个数
例如，5！=120，其末尾所含有的“0”的个数为1；10！= 3628800，其末尾所含有的“0”的个数为2；20！= 2432902008176640000，其末尾所含有的“0”的个数为4。

其计算公式为：
令f(x)表示正整数x末尾所含有的“0”的个数，则有：

      当0 < n < 5时，f(n!) = 0;

      当n >= 5时，f(n!) = k + f(k!), 其中 k = n / 5（取整）。

从而可以递归求解。

### 3、计算从1到n，计算1出现的个数

例如：
N= 2，写下 1，2。这样只出现了 1 个“1”。
N= 12，我们会写下 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12。这样，1的个数是 5。

总结方法：

假设N表示为a[n]a[n-1]...a[1]，其中`a[i](1<=i<=n)`表示N的各位数上的数字。

c[i]表示从整数1到整数a[i]...a[1]中包含数字1的个数。

x[i]表示从整数1到10^i - 1中包含数字1的个数，例如，x[1]表示从1到9的个数，结果为1；x[2]表示从1到99的个数，结果为20；

当a[i]=1时，c[i] = a[i-1]*...*a[1] +1+ c[i-1]+x[i-1];

当a[i]>1时，c[i] = a[i]x[i-1]+c[i-1]+10^(i-1);

没怎么懂----
[计算0到N中包含数字1的个数](http://blog.sina.com.cn/s/blog_6933011901013cdb.html)


### 4、非基于比较的排序算法（三种）
计数排序、基数排序、桶排序是非比较排序算法，平均时间复杂度都是O(N).

这些排序元素，因为其关键值本身就含有了定位特征，因而不需要比较就可以确定其前后位置。
#### 4.1 计数排序
计数排序要求待排序的元素的关键值是位于0-k之间的正整数。因而是个非常特殊的情况。

当输入的元素是 n 个 0 到 k 之间的整数时，它的运行时间是 Θ(n + k)。

计数排序是用来排序0到100之间的数字的最好的算法，但是它不适合按字母顺序排序人名。

算法的步骤如下：

1.找出待排序的数组中最大和最小的元素

2.统计数组中每个值为i的元素出现的次数，存入数组C的第i项

3.对所有的计数累加（从C中的第一个元素开始，每一项和前一项相加）

4.反向填充目标数组：将每个元素i放在新数组的第C(i)项，每放一个元素就将C(i)减去1

当k不是很大时，这是一个很有效的线性排序算法。更重要的是，它是一种稳定排序算法，即排序后的相同值的元素原有的相对位置不会发生改变(表现在Order上)，这是计数排序很重要的一个性质，就是根据这个性质，我们才能把它应用到基数排序。


#### 4.2 基数排序
基数排序(Radix sort)是一种非比较型整数排序算法，其原理是将整数按位数切割成不同的数字，然后按每个位数分别比较。

基数排序的思想就是将待排数据中的每组关键字依次进行桶分配.

基数排序的方式可以采用LSD（Least significant digital）或MSD（Most significant digital），LSD的排序方式由键值的最右边开始，而MSD则相反，由键值的最左边开始。

LSD的基数排序适用于位数小的数列，如果位数多的话，使用MSD的效率会比较好，MSD的方式恰与LSD相反，是由高位数为基底开始进行分配，其他的演算方式则都是相同。

基数排序的时间复杂度是 O(k·n)，其中n是排序元素个数，k是数字位数。

- 时间效率：设待排序列为n个记录，d个关键码，关键码的取值范围为radix，则进行链式基数排序的时间复杂度为O(d(n+radix))，其中，一趟分配时间复杂度为O(n)，一趟收集时间复杂度为O(n)，共进行d趟分配和收集。

- 空间效率：需要2*radix个指向队列的辅助空间，以及用于静态链表的n个指针.

- 缺点：1不呈现时空局部性，因为在按位对每个数进行排序的过程中，一个数的位置可能发生巨大的变化，所以不能充分利用现代机器缓存提供的优势。2同时计数排序作为中间稳定排序的话，不具有原地排序的特点，当内存容量比较宝贵的时候，还是有待商榷

很明显，基数排序的性能比桶排序要略差。每一次关键字的桶分配都需要O(N)时间复杂度，而且分配之后得到新的关键字序列又需要O(N)的时间复杂度。

假如待排数据可以分为d个关键字，则基数排序的时间复杂度将是O(d*2N) ，当然d要远远小于N，因此基本上还是线性级别的。基数排序的空间复杂度为O(N+M)，其中M为桶的数量。一般来说N>>M，因此额外空间需要大概N个左右。

 但是，对比桶排序，基数排序每次需要的桶的数量并不多。而且基数排序几乎不需要任何“比较”操作，而桶排序在桶相对较少的情况下，桶内多个数据必须进行基于比较操作的排序。因此，在实际应用中，基数排序的应用范围更加广泛。

#### 4.3 桶排序
假设有一组长度为N的待排关键字序列K[1....n]。首先将这个序列划分成M个的子区间(桶) 。

然后基于某种映射函数 ，将待排序列的关键字k映射到第i个桶中(即桶数组B的下标 i) ，那么该关键字k就作为B[i]中的元素(每个桶B[i]都是一组大小为N/M的序列)。接着对每个桶B[i]中的所有元素进行比较排序(可以使用快排)。然后依次枚举输出B[0]....B[M]中的全部内容即是一个有序序列。

如待排序列K= {49、 38 、 35、 97 、 76、 73 、 27、 49 }。这些数据全部在1—100之间。因此我们定制10个桶，然后确定映射函数f(k)=k/10。则第一个关键字49将定位到第4个桶中(49/10=4)。依次将所有关键字全部堆入桶中，并在每个非空的桶中进行快速排序。

桶排序代价分析  

- 桶排序利用函数的映射关系，减少了**几乎**所有的比较工作。实际上，桶排序的f(k)值的计算，**其作用就相当于快排中划分**，已经把大量数据分割成了基本有序的数据块(桶)。然后只需要对桶中的少量数据做先进的比较排序即可。

- 桶排序的平均时间复杂度为线性的O(N+C)，其中C=N\*(logN-logM)。如果相对于同样的N，**桶数量M**越大，其效率越高，最好的时间复杂度达到O(N)。 当然桶排序的空间复杂度 为O(N+M)，如果输入数据非常庞大，而桶的数量也非常多，则空间代价无疑是昂贵的。此外，桶排序是稳定的。

### 5、Rotate Array in O(1) space.
Rotate an array of n elements to the right by k steps.

For example, with n = 7 and k = 3, the array [1,2,3,4,5,6,7] is rotated to [5,6,7,1,2,3,4].

思路：

The basic idea is that, for example, nums = [1,2,3,4,5,6,7] and k = 3, first we reverse [1,2,3,4], it becomes[4,3,2,1]; then we reverse[5,6,7], it becomes[7,6,5], finally we reverse the array as a whole, it becomes[4,3,2,1,7,6,5] ---> [5,6,7,1,2,3,4].

Reverse is done by using two pointers, one point at the head and the other point at the tail, after switch these two, these two pointers move one position towards the middle.

好代码：

	public void rotate(int[] nums, int k) {

	    if(nums == null || nums.length < 2){
	        return;
	    }

	    k = k % nums.length;
	    reverse(nums, 0, nums.length - k - 1);
	    reverse(nums, nums.length - k, nums.length - 1);
	    reverse(nums, 0, nums.length - 1);
	}

	private void reverse(int[] nums, int i, int j){
	    int tmp = 0;       
	    while(i < j){
	        tmp = nums[i];
	        nums[i] = nums[j];
	        nums[j] = tmp;
	        i++;
	        j--;
	    }
	}

### 6、Compare Version Numbers
Here is an example of version numbers ordering:

	0.1 < 1.1 < 1.2 < 13.37

7行代码：

	public int compareVersion(String version1, String version2) {
		String[]v1=version1.split("\\."),v2=version2.split("\\.");
		int i;
		for( i =0;i<v1.length&&i<v2.length;i++)
		if(Integer.parseInt(v1[i])!=Integer.parseInt(v2[i]))return Integer.parseInt(v1[i])>Integer.parseInt(v2[i])?1:-1;
		for(;i<v1.length;i++)if(Integer.parseInt(v1[i])!=0)return 1;
		for(;i<v2.length;i++)if(Integer.parseInt(v2[i])!=0)return -1;
		return 0;
	}

### 7、堆排序复杂度分析

辅助空间(空间复杂度）：O(1)

时间复杂度：O(nlog2(n))

在构建堆的过程中，因为我们是完全二叉树从最下层最右边的非终端结点开始构建，将它与其孩子进行比较和若有必要的互换，对于每个非终端结点来说，其实最多进行两次比较和互换操作，因此整个构建堆的时间复杂度为O(n)。

调整堆的时间复杂度为O(logn)。

不过由于记录的比较与交换是跳跃式进行，因此堆排序也是一种**不稳定**的排序方法。

- 建堆，建堆是不断调整堆的过程，从len/2处开始调整，一直到第一个节点，此处len是堆中元素的个数。建堆的过程是线性的过程，从len/2到0处一直调用调整堆的过程，相当于o(h1)+o(h2)…+o(hlen/2) 其中h表示节点的深度，len/2表示节点的个数，这是一个求和的过程，结果是线性的O(n)。

- 调整堆：调整堆在构建堆的过程中会用到，而且在堆排序过程中也会用到。利用的思想是比较节点i和它的孩子节点left(i),right(i)，选出三者最大(或者最小)者，如果最大（小）值不是节点i而是它的一个孩子节点，那边交互节点i和该节点，然后再调用调整堆过程，这是一个递归的过程。调整堆的过程时间复杂度与堆的深度有关系，是log（n）的操作，因为是沿着深度方向进行调整的。

- 堆排序：堆排序是利用上面的两个过程来进行的。首先是根据元素构建堆。然后将堆的根节点取出(一般是与最好一个节点进行交换)，将前面len-1个节点继续进行堆调整的过程，然后再将根节点取出，这样一直到所有节点都取出。堆排序过程的时间复杂度是O(nlgn)。因为建堆的时间复杂度是O(n)（调用一次）；调整堆的时间复杂度是lgn，调用了n-1次，所以堆排序的时间复杂度是O(nlgn)

### 8、最长自增子序列
>http://www.cnblogs.com/lonelycatcher/archive/2011/07/28/2119123.html


【问题】设L=<a1,a2,…,an>是n个不同的实数的序列，L的递增子序列是这样一个子序列Lin=<aK1,ak2,…,akm>，其中k1<k2<…<km且aK1<ak2<…<akm。求最大的m值。

【求解】

1 . 转换成最长公共子序列问题（LCS）：设序列X=<b1,b2,…,bn>是对序列L=<a1,a2,…,an>按递增排好序的序列。那么显然X与L的最长公共子序列即为L的最长递增子序列。这样就把求最长递增子序列的问题转化为求最长公共子序列问题LCS了。

  最长公共子序列问题用动态规划的算法可解。设Li=< a1,a2,…,ai>,Xj=< b1,b2,…,bj>，它们分别为L和X的子序列。令C[i,j]为Li与Xj的最长公共子序列的长度。则有如下的递推方程：

  这可以用时间复杂度为O(n2)的算法求解，由于这个算法上课时讲过，所以具体代码在此略去。求最长递增子序列的算法时间复杂度由排序所用的O(nlogn)的时间加上求LCS的O(n^2)的时间，算法的最坏时间复杂度为O(nlogn)＋O(n^2).

-----

2.动态规划方法(初步）

设f(i)表示L中以ai为末元素的最长递增子序列的长度。则有如下的递推方程：

这个递推方程的意思是，在求以ai为末元素的最长递增子序列时，找到所有序号在L前面且小于ai的元素aj，即j<i且aj<ai。

如果这样的元素存在，那么对所有aj,都有一个以aj为末元素的最长递增子序列的长度f(j)，把其中最大的f(j)选出来，那么f(i)就等于最大的f(j)加上1，即以ai为末元素的最长递增子序列，等于以使f(j)最大的那个aj为末元素的递增子序列最末再加上ai；如果这样的元素不存在，那么ai自身构成一个长度为1的以ai为末元素的递增子序列.

	public void lis(float[] L) {
		int n = L.length;
		int[] f = new int[n];// 用于存放f(i)值；
		f[0] = 1;// 以第a1为末元素的最长递增子序列长度为1；
		for (int i = 1; i < n; i++)// 循环n-1次
		{
			f[i] = 1;// f[i]的最小值为1；
			for (int j = 0; j < i; j++)// 循环i 次
			{
				if (L[j] < L[i] && f[j] > f[i] - 1)
					f[i] = f[j] + 1;// 更新f[i]的值。
			}
		}
		System.out.println(f[n - 1]);
	}

所以T(n)=O(n2)。这个算法的最坏时间复杂度与第一种算法的阶是相同的。但这个算法没有排序的时间，所以时间复杂度要优于前一种方法。

------

3.改进

在第二种算法中，在计算每一个f(i)时，都要找出最大的f(j)(j<i)来，由于f(j)没有顺序，只能顺序查找满足aj<ai最大的f(j)，如果能将让f(j)有序，就可以使用二分查找，这样算法的时间复杂度就可能降到O(nlogn)。于是想到用一个数组B来存储“子序列的”最大递增子序列的最末元素，即有  `B[f(j)] = aj  `

在计算f(i)时，在数组B中用二分查找法找到满足j<i且B[f(j)]=aj<ai的最大的j,并将B[f[j]+1]置为ai。下面先写出代码，再证明算法的证明性。

	lis1(float[] L)
	{
	    int n = L.length;
	    float[] B = new float[n+1];//数组B；
	    B[0]=-10000;//把B[0]设为最小，假设任何输入都大于-10000；
	    B[1]=L[0];//初始时，最大递增子序列长度为1的最末元素为a1
	    int Len = 1;//Len为当前最大递增子序列长度，初始化为1；
	    int p,r,m;//p,r,m分别为二分查找的上界，下界和中点；
	    for(int i = 1;i<n;i++)
	    {
	        p=0;r=Len;
	        while(p<=r)//二分查找最末元素小于ai+1的长度最大的最大递增子序列；
	        {
	           m = (p+r)/2;
	           if(B[m]<L[i]) p = m+1;
	           else r = m-1;
	        }
	        B[p] = L[i];//将长度为p的最大递增子序列的当前最末元素置为ai+1;
	        if(p>Len) Len++;//更新当前最大递增子序列长度；


	    }
	    System.out.println(Len);
	}
