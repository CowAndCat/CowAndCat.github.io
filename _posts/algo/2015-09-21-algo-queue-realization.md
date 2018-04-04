---
layout: post
title: 队列的数组实现和链式实现
category: algorithm
comments: false
---

### 数组实现
1.将队列的一端固定在数组的索引0处，（注意由于队首要求固定，所以出队操作就要挨个移动元素）。维护一个整型变量count标识下一个open单元（也表示队列的当前大小）

代码如下：

	public class ArrayQueue implements QueueADT {

        private Object[] contents;//将队首固定在数组的0位置
        private int rear;//指向下一个入队的位置，且表示队列的长度
        private static int SIZE = 10;

        public static void main(String[] args) {

            ArrayQueue queue = new ArrayQueue();

            System.out.println("依次将0到24入队，然后连续出队10次");
            for(int i = 0;i < 25;i++)
                queue.enqueue(i);
            for(int i = 0;i < 10;i++)
                queue.dequeue();

            System.out.println("队列的大小为： " + queue.size());
            System.out.println("队列为空吗？： " + queue.isEmpty());
            System.out.println("队列的第一个元素为： " + queue.first());
        }

        public ArrayQueue()
        {
            contents = new Object[SIZE];
            rear = 0;
        }

        public void expand()
        {
            Object[] larger = new Object[size()*2];
            for(int index = 0;index < rear;index++)
                larger[index] = contents[index];

            contents = larger;
        }

        public int size() {
            return rear;
        }

        public boolean isEmpty() {    
            return (size() == 0);
        }

        public void enqueue(Object element) {
            if(rear == contents.length)
                expand();
            contents[rear] = element;
            rear++;
        }

        public Object dequeue(){
            if(isEmpty())
            {
                System.out.println("队列为空");
                System.exit(1);
            }

            Object result = contents[0];

            for(int index = 0;index < rear;index++)
                    contents[index] = contents[index+1];
            rear--;

            return result;        
        }

        public Object first() {
            return contents[0];
        }

	}

2.使用循环数组来实现队列  
上面提到过，将队首固定在一端，在出队的时候会依次将剩下的元素往前移动一步，对于容量较大的队列，是一个很大的开销。使用循环队列可以消除元素移动带来的效率问题，用一个front和rear整型变量作为指针来滑动标记队首和队尾元素，当遇到数组末尾时，首尾相连到数组头.。再用一个count来记录队列的大小。

代码如下：

	public class CircularArrayQueue {

	    private Object[] contents;
	    private int front,rear;//front为队头下标，rear为队尾的下一个元素的下标
	    private int count;//标记队列元素个数
	    private static int SIZE = 10;

	    public static void main(String[] args) {

	        CircularArrayQueue circlequeue = new CircularArrayQueue();

	        System.out.println("将0到7依次入队，然后连续4次出队");

	        for(int i = 0;i < 8;i++)
	            circlequeue.enqueue(i);
	        for(int i = 0;i < 4;i++)
	            circlequeue.dequeue();
	        System.out.println("队首元素为： " + circlequeue.first());
	        System.out.println("队首元素下标为： " + circlequeue.front);
	        System.out.println("队尾元素下标为： " + circlequeue.rear);
	        System.out.println("队列大小为： " + circlequeue.count + "\n");

	        System.out.println("再向队列中加入从8到11");
	        for(int i = 8;i < 12;i++)
	            circlequeue.enqueue(i);
	        System.out.println("队首元素为： " + circlequeue.first());
	        System.out.println("队首元素下标为： " + circlequeue.front);
	        System.out.println("队尾元素下标为： " + circlequeue.rear);
	        System.out.println("队列大小为： " + circlequeue.count);
	    }

	    public CircularArrayQueue()
	    {
	        contents = new Object[SIZE];
	        front = -1;
	        rear = 0;
	        count = 0;
	    }

	    public int size(){
	        return count;
	    }

	    public boolean isEmpty(){
	        return (size() == 0);
	    }

	    public void enqueue(Object element){

	        if(size() == 0)//默认如果数组中没有元素，则从0开始进队，这样简化问题的考虑
	        {
	            contents[0] = element;
	            front = 0;
	            rear = 1;
	            //rear++   错误，经过一些进队出队操作后rear可能在别的位置了
	            count++;
	        }
	        else
	        {
	            contents[rear] = element;
	            rear = (rear + 1) % contents.length;
	            //rear = (rear + 1) % size(); 错误
	            count++;
	        }
	    }

	    public Object dequeue(){
	        if(isEmpty())
	        {
	            System.out.println("队列为空！！！");
	            System.exit(1);
	        }

	        Object result = contents[front];

	        contents[front] = null;//可要可不要，下次覆盖也可
	        front = (front + 1) % contents.length;
	        //front = (front + 1) % size(); 错误
	        count--;

	        return result;    
	    }

	    public Object first(){
	        return contents[front];
	    }

	}
	
## 参考
> [队列的数组实现和链式实现](http://www.cnblogs.com/kkgreen/archive/2011/04/30/2033427.html)
