---
title: 算法进阶第四五课.md
date: 2019-04-01 12:09:50
categories:
  - leetcode
---

### 题目：大楼的轮廓线

给定一个N行3列二维数组， 每一行表示有一座大楼， 一共有N座大楼。
所有大楼的底部都坐落在X轴上， 每一行的三个值(a,b,c)代表每座大楼的从(a,0)点开始， 到(b,0)点结束， 高度为c。
输入的数据可以保证a<b,且a， b， c均为正数。 大楼之间可以有重合。
请输出整体的轮廓线。

例子： 给定一个二维数组
[[1, 3, 3],
[2, 4, 4],
[5, 6, 1]]

输出为轮廓线
[[1, 2, 3],
[2, 4, 4],
[5, 6, 1]] 

<br/>

{% asset_img picture13.jpg %}

<br/>

思路：

把给定的二维数组设计成如下的方式：

(1,3,上)、(3,3,下)、(2,4,上)、(4,4,下)、(5,1,上)、(6,1,下)；

然后按照位置排序

(1,3,上)、(2,4,上)、(3,3,下)、(4,4,下)、(5,1,上)、(6,1,下)；

一个轮廓的产生，一定是轮廓发生变化了，而且是最大高度发生变化了，[1, 2, 3] 这个轮廓就是因为1的位置上高度发生变化了，从0变为了1；[2, 4, 4] 这个轮廓是因为2位置上高度从3变成了4，[5, 6, 1] 是因为5位置上高度从0变成了1；

<br/>

{% asset_img picture14.jpg %}

treemap 中key是高度，value是出现的次数，(1,4,上)进来的时候，发现1位置上4出现了一次，此时最大高度发生了变回，从0变成了4，此时产生轮廓，(2,3,上) 进来的时候，3比4小，最大高度没有发生变化，(4,3,下)进来的时候，3从map中删除，此时最大高度还是4，不产生轮廓，(5,5,上) 表示5位置上5高度出现了一次，此时产生最大高度，所以产生轮廓线，  (6,4,下)  map中除去4，最大高度不发生变化，还是5，不产生轮廓线。。。之后的过程同理。

所以在1，5，8，10 处 产生轮廓线。 

<br/>

代码

```java
public class Code_01_Building_Outline {

    public static class Node {
        public boolean isUp; //上还是下
        public int posi;  //哪个位置
        public int h;   //哪个高度

        public Node(boolean isAddHeight, int position, int height) {
            isUp = isAddHeight;
            posi = position;
            h = height;
        }
    }

    //比较器
    public static class NodeComparator implements Comparator<Node> {
        @Override
        public int compare(Node o1, Node o2) {
            if (o1.posi != o2.posi) {
                return o1.posi - o2.posi;
            }
            //相同的高度，上的位置排在前面，下的位置排在后面，
            if (o1.isUp != o2.isUp) {
                return o1.isUp ? -1 : 1;
            }
            return 0;
        }
    }

    /**
     * @param buildings n行3列的矩阵，n就是大楼的个数
     */
    public static List<List<Integer>> buildingOutline(int[][] buildings) {
        Node[] nodes = new Node[buildings.length * 2];//生成的节点个数是大楼个数的两倍；
        for (int i = 0; i < buildings.length; i++) {
            //0号大楼的位置放在0，1位置，1号大楼放在2，3位置，等
            nodes[i * 2] = new Node(true, buildings[i][0], buildings[i][2]);// true 表示上
            nodes[i * 2 + 1] = new Node(false, buildings[i][1], buildings[i][2]);
        }
        Arrays.sort(nodes, new NodeComparator()); // 按照位置排序
        TreeMap<Integer, Integer> heightTimesMap = new TreeMap<>(); //某一个高度出现的次数
        TreeMap<Integer, Integer> xMaxHeightMap = new TreeMap<>();  //当前位置最大高度是多少
        for (int i = 0; i < nodes.length; i++) {
            if (nodes[i].isUp) {//加高度的操作
                if (!heightTimesMap.containsKey(nodes[i].h)) {//如果不含有此高度，则次数为1，否则次数加1
                    heightTimesMap.put(nodes[i].h, 1);
                } else {
                    heightTimesMap.put(nodes[i].h, heightTimesMap.get(nodes[i].h) + 1);
                }
            } else {//减高度的操作
                if (heightTimesMap.get(nodes[i].h) == 1) { //如果减完之后次数为0，则直接删除这条记录
                    heightTimesMap.remove(nodes[i].h);
                } else {
                    heightTimesMap.put(nodes[i].h, heightTimesMap.get(nodes[i].h) - 1);
                }
            }
            //跟踪记录每个位置最大高度的变化
            if (heightTimesMap.isEmpty()) {
                xMaxHeightMap.put(nodes[i].posi, 0);
            } else {
                xMaxHeightMap.put(nodes[i].posi, heightTimesMap.lastKey());
            }
        }
        //生成最后的轮廓线，只用 xMaxHeightMap 就可以建立出来
        List<List<Integer>> res = new ArrayList<>();
        int start = 0;
        int height = 0;
        for (Map.Entry<Integer, Integer> entry : xMaxHeightMap.entrySet()) {
            int curPosition = entry.getKey();    //当前位置
            int curMaxHeight = entry.getValue();  //当前最大高度
            if (height != curMaxHeight) {  //之前的高度不等于现在的最大高度，说明要开始产生轮廓线了；
                if (height != 0) {
                    List<Integer> newRecord = new ArrayList<>();
                    newRecord.add(start);
                    newRecord.add(curPosition);
                    newRecord.add(height);
                    res.add(newRecord);
                }
                start = curPosition;
                height = curMaxHeight;
            }
        }
        return res;
    }
}
```

<br/>

### 题目：未排序数组中累加和为给定值的最长子数组

给定一个无序数组arr，其中元素可正，可负，可0，给定一个整数k。

求arr所有的子数组中累加和为k的最长子数组长度。

<br/>

思路：

求出数组中必须以每个位置结尾的累加和为aim的最长子数组；答案一定在其中；

如何求以i位置为结尾的累加和为aim的最长子数组？用一个变量sum表示0到i的所有数的累加和，假设0到1000位置sum=2000，aim=800，此时求以1000位置为结尾的情况下，最长子数组的长度，此时只要找到以0位置开始，哪个位置最早累加到sum-aim：1200，假设17位置最早累加到1200，那18到1000位置，累加和必是800；

<br/>

实现:

数组7，3，2，1，1，7，-6，-1，7；   aim=7

准备一个map，key是一个累加和，value是这个累加和第一次出现的位置；

map中最开始存在一个值<0,-1>表示0这个累加和第一次出现在-1位置，0位置7进来，sum=7，<7,0>放进map，然后7-7=0，0在map中存在，说明找到以0位置结尾的最长子数组，从0到0，长度为1，1位置3进来，sum=10，10-7=3，map中不存在，说明以1位置结尾找不到符合条件的最长子数组，2位置2进来，sum=12，12-7=5，map中不存在，说明以2位置结尾找不到符合条件的最长子数组，3位置1进来，sum=13，13-7=6，map中不存在，说明以3位置结尾找不到符合条件的最长子数组，4位置1进来，sum=14，14-7=7，之前存在<7,0>，所以1位置到4位置存在符合条件的最长子数组，长度为4，后面的过程继续即可；

{% asset_img picture15.jpg %}

<br/>

注意：如果不加<0,-1>这个记录，那么所以从0开头的可能性就都错过了，因为我们查出一个位置，从这个位置的下一个位置开始到当前位置作为最长子数组的；

<br/>

 代码

```java
public class Code_05_LongestSumSubArrayLength {

    public int maxLength(int[] arr, int aim) {
        if (arr == null || arr.length == 0) {
            return 0;
        }

        Map<Integer, Integer> map = new HashMap<Integer, Integer>();
        map.put(0, -1); // important
        int len = 0;
        int sum = 0;
        for (int i = 0; i < arr.length; i++) {
            sum += arr[i];
            if (map.containsKey(sum - aim)) {
                len = Math.max(i - map.get(sum - aim), len); // i - map.get(sum - aim) 就表示以i位置为结尾的最长数组的长度
            }
            if (!map.containsKey(sum)) {  //加入累加和
                map.put(sum, i); 
            }
        }
        return len;
    }
}
```

<br/>

### 扩展：最长子数组

1. 给定一个无序数组arr，其中元素可正，可负，可0。求arr所有的子数组中正数与负数个数相等的最长子数组长度。

思路：

将数组所有的正数都变为1，负数都变为-1，0不变，然后求累加和为0的最长子数组长度。

<br/>

2. 给定一个无序数组arr，其中元素只是0、1、2。求arr所有的子数组中1和2个数相等的最长子数组长度

思路：

0不变，1不变，2变成-1，求累加和为0的最长子数组；  

<br/>

<br/>

### 题目：数组的异或和

定义数组的异或和的概念：数组中所有的数异或起来， 得到的结果叫做数组的异或和，
比如数组{3,2,1}的异或和是， 3^2^1 = 0
给定一个数组arr， 你可以任意把arr分成很多不相容的子数组， 你的目的是：
分出来的子数组中， 异或和为0的子数组最多。
请返回： 分出来的子数组中， 异或和为0的子数组最多是多少？ 

<br/>

性质

异或性质满足交换律和结合律

0和任何数异或都是内个数

任何数和自己异或，结果一定都是0

<br/>

思路

0到i位置最多可以切出多少个异或和为0的子数组

0到i+1位置最多可以切出多少个异或和为0的子数组

....

0到N-1位置最多可以切出多少个异或和为0的子数组

<br/>

0到i位置最多可以切出多少个异或和为0的子数组，有两种可能性，

首先假设0到i位置客观上有一种最优的切分方式，然后0到i位置已经按照这种方式切分好了，那么此时有两种情况

1. i位置所在的最后一个子数组的异或和不是0，此时0到i位置上可以最多可以切多少个符合条件的子数组和0到i-1位置上可以切分出的子数组个数一致；即dp[i] = dp[i-1]，dp问题
2. i位置所在的最后一个子数组的异或和是0，假设最后一个字数组是k位置开头的，那么k中间肯定不会存在一个位置j，使得j到i这个子数组异或和为0，因为我们一开始就假设的切法是最完美的，如果存在j，那么j位置就应该再切一刀，所以k位置一定是左边离i最近的使得子数组异或和为0的位置；所以这个问题变成了：如果0到i异或和为sum，我们就是在找从0到i-1中间异或还是sum的最晚的位置，这个最晚的位置的下一个位置就是k位置。（之前的例子是找一个累加和最早出现的位置，是相同的行为）；

<br/>

{% asset_img picture16.jpg %}

<br/>

代码：

```java
/**
 * 情况1：从 0 到 i 存在最优划分，i 所在的子数组不是异或和为 0 的部分，即不在最优划分里面，
 *  那么 0 到 i 部分能划分多少个异或和为0的子数组 和 0 到 i- 1 能划分多少个异或和为 0 的子数组的结果是一样的，
 *  dp[i] = dp[i - 1]
 *
 *  情况2：从 0 到 i 存在最优划分，i 所在的子数组是异或和为 0 的部分，即是最优划分的中异或和为0的部分，
 *  设 k 是 i 左边离它最近的异或和为0的位置，则问题变成，假设从 0 到 i 异或和为 sum，
 *  那么只需要找到从 0 到 i - 1 异或和为 sum 的位置的下一个位置就是 k 位置。
 *  dp[i] = dp[k - 1] + 1
 */
public class Code_06_Most_EOR {

    public static int mostEOR(int[] arrays){
        if(arrays == null || arrays.length == 0) return 0;
        Map<Integer, Integer> map = new HashMap<>();
        map.put(0, -1);
        int xor = 0;
        int res = 0;
        int[] dp = new int[arrays.length];
        for(int i = 0; i < arrays.length; i++){
            xor ^= arrays[i];
            if(map.containsKey(xor)){
                int pre = map.get(xor);
                dp[i] = pre == -1 ? 1 : (dp[pre] + 1);
            }
            if(i > 0){
                dp[i] = Math.max(dp[i], dp[i - 1]);
            }
            res = Math.max(dp[i], res);
            map.put(xor, i);
        }
        return res;
    }
    // for test
    public static int comparator(int[] arr) {
        if (arr == null || arr.length == 0) {
            return 0;
        }
        int[] eors = new int[arr.length];
        int eor = 0;
        for (int i = 0; i < arr.length; i++) {
            eor ^= arr[i];
            eors[i] = eor;
        }
        int[] mosts = new int[arr.length];
        mosts[0] = arr[0] == 0 ? 1 : 0;
        for (int i = 1; i < arr.length; i++) {
            mosts[i] = eors[i] == 0 ? 1 : 0;
            for (int j = 0; j < i; j++) {
                if ((eors[i] ^ eors[j]) == 0) {
                    mosts[i] = Math.max(mosts[i], mosts[j] + 1);
                }
            }
            mosts[i] = Math.max(mosts[i], mosts[i - 1]);
        }
        return mosts[mosts.length - 1];
    }

    // for test
    public static int[] generateRandomArray(int maxSize, int maxValue) {
        int[] arr = new int[(int) ((maxSize + 1) * Math.random())];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = (int) ((maxValue + 1) * Math.random());
        }
        return arr;
    }

    // for test
    public static void printArray(int[] arr) {
        if (arr == null) {
            return;
        }
        for (int i = 0; i < arr.length; i++) {
            System.out.print(arr[i] + " ");
        }
        System.out.println();
    }

    // for test
    public static void main(String[] args) {
        int testTime = 500000;
        int maxSize = 300;
        int maxValue = 100;
        boolean succeed = true;
        for (int i = 0; i < testTime; i++) {
            int[] arr = generateRandomArray(maxSize, maxValue);
            int res = mostEOR(arr);
            int comp = comparator(arr);
            if (res != comp) {
                succeed = false;
                printArray(arr);
                System.out.println(res);
                System.out.println(comp);
                break;
            }
        }
        System.out.println(succeed ? "Nice!" : "Fucking fucked!");
    }
}
```

<br/>

<br/>

### 题目：给定一棵二叉树的头节点head，请返回最大搜索二叉子树的大小

<br/>

套路（求整棵树的什么东西都可以用这个套路）

假设以每一个节点为头的这棵树，它的最大搜索二叉子树是什么，最终整棵树的最大搜索二叉子树，答案一定在其中；因为我们已经枚举了每一个节点为头的子树的最大搜索二叉子树；

第一步先列出可能性：以x为头节点的子树，有哪些可能性：

1.可能来自它的左子树，

2.可能来自右子树

3.左边是搜索二叉树，右边是搜索二叉树，左边的最大值小于x，右边的最小值大于x，以x为头的整棵树都是二叉搜索树

{% asset_img picture17.jpg %}

<br/>

整个递归流程就是左树收集一些信息，右树收集一些信息，然后返回给父树，父树利用这些信息做整合，继续往上返；

所以就要看需要收集那些信息：

1.左树最大二叉搜索子树的大小是多少，

2.右树最大二叉搜索子树的大小是多少，

3.左树最大二叉搜索子树的头部是什么，如果头部等于x的左子树，说明左树整个都是二叉搜索子树， 否则，则不是，

4.右树最大二叉搜索子树的头部是什么，来判断右树是否整体是二叉搜索树

5.左树上的最大值，

6.右树上的最小值

<br/>

如果你设计了一个过程，一个头节点，左树收信息，右树收信息，这些信息足以让我们判断当前的x头节点是属于哪种情况了，最终得出整体的最大二叉搜索子树

<br/>

化简（需要的信息）

1.不管是左树还是右树，都返回最大二叉搜索子树的大小

2.不管是左树还是右树，都返回二叉子树的头部

3.不管是左树还是右树，返回这个二叉搜索子树的最大值和最小值是什么

<br/>

代码

```java
public class Code_04_BiggestSubBSTInTree {

	public static class Node {
		public int value;
		public Node left;
		public Node right;

		public Node(int data) {
			this.value = data;
		}
	}

	public static Node biggestSubBST(Node head) {
		int[] record = new int[3]; // 0->size, 1->min, 2->max
		return posOrder(head, record);
	}
	
	public static class ReturnType{
		public int size;   //最大二叉搜索子树的大小
		public Node head;   //二叉子树的头部
		public int min;		//二叉搜索子树的最小值
		public int max;		//二叉搜索子树的最大值
		
		public ReturnType(int a, Node b,int c,int d) {
			this.size =a;
			this.head = b;
			this.min = c;
			this.max = d;
		}
	}
	
	public static ReturnType process(Node head) {
		if(head == null) {
			return new ReturnType(0,null,Integer.MAX_VALUE, Integer.MIN_VALUE);
		}

		//收集左树上的四个信息，右树上的四个信息，把他们当成黑盒
		Node left = head.left;
		ReturnType leftSubTressInfo = process(left);
		Node right = head.right;
		ReturnType rightSubTressInfo = process(right);
		
		int includeItSelf = 0;  //可能性3
		if(leftSubTressInfo.head == left 
				&&rightSubTressInfo.head == right
				&& head.value > leftSubTressInfo.max
				&& head.value < rightSubTressInfo.min
				) {  //说明整棵树都是二叉搜索树
			includeItSelf = leftSubTressInfo.size + 1 + rightSubTressInfo.size;
		}
		int p1 = leftSubTressInfo.size;    //可能性1
		int p2 = rightSubTressInfo.size;		//可能性2
		int maxSize = Math.max(Math.max(p1, p2), includeItSelf);
		
		Node maxHead = p1 > p2 ? leftSubTressInfo.head : rightSubTressInfo.head;
		if(maxSize == includeItSelf) {
			maxHead = head;
		}
		
		return new ReturnType(maxSize,
				maxHead, 
				Math.min(Math.min(leftSubTressInfo.min,rightSubTressInfo.min),head.value),
				Math.max(Math.max(leftSubTressInfo.max,rightSubTressInfo.max),head.value));	
	}

	public static Node posOrder(Node head, int[] record) {
		if (head == null) {
			record[0] = 0;
			record[1] = Integer.MAX_VALUE;
			record[2] = Integer.MIN_VALUE;
			return null;
		}
		int value = head.value;
		Node left = head.left;
		Node right = head.right;
		Node lBST = posOrder(left, record);
		int lSize = record[0];
		int lMin = record[1];
		int lMax = record[2];
		Node rBST = posOrder(right, record);
		int rSize = record[0];
		int rMin = record[1];
		int rMax = record[2];
		record[1] = Math.min(rMin, Math.min(lMin, value)); // lmin, value, rmin -> min 
		record[2] =  Math.max(lMax, Math.max(rMax, value)); // lmax, value, rmax -> max
		if (left == lBST && right == rBST && lMax < value && value < rMin) {
			record[0] = lSize + rSize + 1;
			return head;
		}
		record[0] = Math.max(lSize, rSize);
		return lSize > rSize ? lBST : rBST;
	}

	// for test -- print tree
	public static void printTree(Node head) {
		System.out.println("Binary Tree:");
		printInOrder(head, 0, "H", 17);
		System.out.println();
	}

	public static void printInOrder(Node head, int height, String to, int len) {
		if (head == null) {
			return;
		}
		printInOrder(head.right, height + 1, "v", len);
		String val = to + head.value + to;
		int lenM = val.length();
		int lenL = (len - lenM) / 2;
		int lenR = len - lenM - lenL;
		val = getSpace(lenL) + val + getSpace(lenR);
		System.out.println(getSpace(height * len) + val);
		printInOrder(head.left, height + 1, "^", len);
	}

	public static String getSpace(int num) {
		String space = " ";
		StringBuffer buf = new StringBuffer("");
		for (int i = 0; i < num; i++) {
			buf.append(space);
		}
		return buf.toString();
	}

	public static void main(String[] args) {

		Node head = new Node(6);
		head.left = new Node(1);
		head.left.left = new Node(0);
		head.left.right = new Node(3);
		head.right = new Node(12);
		head.right.left = new Node(10);
		head.right.left.left = new Node(4);
		head.right.left.left.left = new Node(2);
		head.right.left.left.right = new Node(5);
		head.right.left.right = new Node(14);
		head.right.left.right.left = new Node(11);
		head.right.left.right.right = new Node(15);
		head.right.right = new Node(13);
		head.right.right.left = new Node(20);
		head.right.right.right = new Node(16);

		printTree(head);
		Node bst = biggestSubBST(head);
		printTree(bst);
	}
}
```

<br/>

<br/>

### 题目：求一棵二叉树的最远距离

二叉树中，一个节点可以往上走和往下走，那么从节点A总能走到节点 B。 

节点A走到节点B的距离为：A走到B最短路径上的节点个数。

 求一棵二叉树上的最远距离；

<br/>

举例

下图的最远距离是6-10，总共经过8个节点；

{% asset_img picture18.jpg %}

<br/>

大套路

求每一个节点的最远距离是多少，答案一定在其中

<br/>

列出可能性

1.可能只来自左子树的最大距离，而和x头结点没有关系，而且是不经过x的，

2.可能只来自右子树的最大距离

3.如果在经过x节点的时候产生的最大距离，左子树到深度最低的地方，右子树到深度最低的地方，中间经过x节点

<br/>

收集的信息

1.子树的最大距离是多少

2.子树的深度是多少

<br/>

代码

```java
public class Code_03_MaxDistanceInTree {

	public static class Node {
		public int value;
		public Node left;
		public Node right;

		public Node(int data) {
			this.value = data;
		}
	}

	public static int maxDistance(Node head) {
		int[] record = new int[1];
		return posOrder(head, record);
	}
	
	public static class ReturnType{
		public int maxDistance;  //最大距离
		public int h;	    // 深度
		
		public ReturnType(int m, int h) {
			this.maxDistance = m;;
			this.h = h;
		}
	}
	
	public static ReturnType process(Node head) {
		if(head == null) {
			return new ReturnType(0,0);
		}
		//黑盒
		ReturnType leftReturnType = process(head.left);
		ReturnType rightReturnType = process(head.right);

		//可能性3：左边的深度+1+右边的深度
		int includeHeadDistance = leftReturnType.h + 1 + rightReturnType.h;
		int p1 = leftReturnType.maxDistance;  //可能性1
		int p2 = rightReturnType.maxDistance;  //可能性2

		//构建返回给上层的结构
		int resultDistance = Math.max(Math.max(p1, p2), includeHeadDistance);
		int hitself  = Math.max(leftReturnType.h, leftReturnType.h) + 1;  //深度
		return new ReturnType(resultDistance, hitself);
	}

	public static int posOrder(Node head, int[] record) {
		if (head == null) {
			record[0] = 0;
			return 0;
		}
		int lMax = posOrder(head.left, record);
		int maxfromLeft = record[0];
		int rMax = posOrder(head.right, record);
		int maxFromRight = record[0];
		int curNodeMax = maxfromLeft + maxFromRight + 1;
		record[0] = Math.max(maxfromLeft, maxFromRight) + 1;
		return Math.max(Math.max(lMax, rMax), curNodeMax);
	}

	public static void main(String[] args) {
		Node head1 = new Node(1);
		head1.left = new Node(2);
		head1.right = new Node(3);
		head1.left.left = new Node(4);
		head1.left.right = new Node(5);
		head1.right.left = new Node(6);
		head1.right.right = new Node(7);
		head1.left.left.left = new Node(8);
		head1.right.left.right = new Node(9);
		System.out.println(maxDistance(head1));

		Node head2 = new Node(1);
		head2.left = new Node(2);
		head2.right = new Node(3);
		head2.right.left = new Node(4);
		head2.right.right = new Node(5);
		head2.right.left.left = new Node(6);
		head2.right.right.right = new Node(7);
		head2.right.left.left.left = new Node(8);
		head2.right.right.right.right = new Node(9);
		System.out.println(maxDistance(head2));
	}
}
```

<br/>

<br/>

### 题目：返回最大的活跃值

一个公司的上下节关系是一棵多叉树，这个公司要举办晚会，你作为组织者已经摸清了大家的心理：一个员工的直 接上级如果到场，这个员工肯定不会来。每个员工都有一个活跃度的值，决定谁来你会给这个员工发邀请函，怎么 让舞会的气氛最活跃？返回最大的活跃值。

举例：

给定一个矩阵来表述这种关系

matrix = {

1,6 

1,5

1,4 

}

这个矩阵的含义是：

matrix[0] = {1 , 6}，表示0这个员工的直接上级为1,0这个员工自己的活跃度为6

matrix[1] = {1 , 5}，表示1这个员工的直接上级为1（他自己是这个公司的最大boss）,1这个员工自己的活跃度 为5

matrix[2] = {1 , 4}，表示2这个员工的直接上级为1,2这个员工自己的活跃度为4

为了让晚会活跃度最大，应该让1不来，0和2来。最后返回活跃度为10

<br/>

举例

{% asset_img picture19.jpg %}

此时应该让a不来（a来了就错过了d这个300的活跃值），让c不来（c来了，h和i就不来了，他们俩的活跃值加起来是10，比c一个人的大），等等，最终返回最大的活跃值；

<br/>

大套路

分析以每一个节点作为头结点的多叉树的最大活跃度，

<br/>

列出可能性

1.x来，此时活跃度就是x自己的活跃度加上x的子节点都不来的活跃度

{% asset_img picture20.jpg %}

<br/>

2.x不来，此时的活跃度就是x的子节点的来和不来中选最大的加和，

{% asset_img picture21.jpg %}

<br/>

需要收集的信息

1.一棵树在头结点不来的时候的活跃度

2.一棵树在头结点来的时候的活跃度

<br/>

代码：

```JAVA
public class Code_04_MaxHappy {

	public static class Node{
		public int huo;
		public List<Node> nexts;
		public Node(int huo){
			this.huo = huo;
			this.nexts = new ArrayList<>();
		}
	}

	public static int getMaxHuo(Node head){
		ReturnData data = process(head);
		return Math.max(data.bu_lai_huo,data.lai_huo);
	}

	public static class ReturnData{
		public int lai_huo;
		public int bu_lai_huo;

		public ReturnData(int lai_huo, int bu_lai_huo){
			this.lai_huo = lai_huo;
			this.bu_lai_huo = bu_lai_huo;
		}
	}

	public static ReturnData process(Node head){
		int lai_huo = head.huo;
		int bu_lai_huo = 0;

		for (int i = 0; i < head.nexts.size(); i++){
			Node next = head.nexts.get(i);
			ReturnData nextData = process(next);

			lai_huo = nextData.bu_lai_huo;
			bu_lai_huo = Math.max(nextData.bu_lai_huo,nextData.lai_huo);
		}

		return new ReturnData(lai_huo,bu_lai_huo);
	}


	public static int maxHappy(int[][] matrix) {
		int[][] dp = new int[matrix.length][2];
		boolean[] visited = new boolean[matrix.length];
		int root = 0;
		for (int i = 0; i < matrix.length; i++) {
			if (i == matrix[i][0]) {
				root = i;
			}
		}
		process(matrix, dp, visited, root);
		return Math.max(dp[root][0], dp[root][1]);
	}

	public static void process(int[][] matrix, int[][] dp, boolean[] visited, int root) {
		visited[root] = true;
		dp[root][1] = matrix[root][1];
		for (int i = 0; i < matrix.length; i++) {
			if (matrix[i][0] == root && !visited[i]) {
				process(matrix, dp, visited, i);
				dp[root][1] += dp[i][0];
				dp[root][0] += Math.max(dp[i][1], dp[i][0]);
			}
		}
	}

	public static void main(String[] args) {
		int[][] matrix = { { 1, 8 }, { 1, 9 }, { 1, 10 } };
		System.out.println(maxHappy(matrix));
	}
}
```

<br/>

<br/>

### 题目：LRU

设计可以变更的缓存结构（LRU） 

【题目】 

设计一种缓存结构，该结构在构造时确定大小，假设大小为K，并有两个功能：

set(key,value)：将记录(key,value)插入该结构。 

get(key)：返回key对应的value值。 

【要求】

 1．set和get方法的时间复杂度为O(1)。

 2．某个key的set或get操作一旦发生，认为这个key的记录成了最经常使用的。 

3．当缓存的大小超过K时，移除最不经常使用的记录，即set或get最久远的。

【举例】 假设缓存结构的实例是cache，大小为3，并依次发生如下行为： 

1．cache.set("A",1)。最经常使用的记录为("A",1)。 

2．cache.set("B",2)。最经常使用的记录为("B",2)，("A",1)变为最不经常的。

3．cache.set("C",3)。最经常使用的记录为("C",2)，("A",1)还是最不经常的。

4．cache.get("A")。最经常使用的记录为("A",1)，("B",2)变为最不经常的。 

5．cache.set("D",4)。大小超过了3，所以移除此时最不经常使用的记录("B",2)， 加入记录 ("D",4)，并且为最经常使用的记录，然后("C",2)变为最不经常使用的记录

<br/>

思路

哈希表+双向链表

<br/>

map：<key，<key,value>>，里面的<key,value> 是双向链表的内存地址，可以直接在双向链表中找到这个节点，

双向链表：需要自己定制，可以做到从尾部加，从头部出。尾部是优先级高的。

<br/>

put时，先加入map，然后加到双向链表的末尾；

get时，现在map中找到对应的节点，然后在双向链表中找到对应的节点（直接可以找到），把这个节点从双向链表中分离出来，然后挂到最后；也就是使用这个链表维持优先级；

修改时，先从map中取出来，直接修改Node的值，因为和其在双向链表中是一个值，所以直接修改即可；然后从双向链表中的位置移到末尾；

<br/>

替换时，直接找到双向链表的头指针指向的位置，移除，然后从map中根据key找到，然后从map中移除；

{% asset_img picture22.jpg %}

<br/>

 代码

```java
public class Code_02_LRU {

	public static class Node<V> {
		public V value;
		public Node<V> last;
		public Node<V> next;

		public Node(V value) {
			this.value = value;
		}
	}

	public static class NodeDoubleLinkedList<V> {
		private Node<V> head;  //头指针
		private Node<V> tail;   //尾指针

		public NodeDoubleLinkedList() {
			this.head = null;
			this.tail = null;
		}

		//新增节点
		public void addNode(Node<V> newNode) {
			if (newNode == null) {
				return;
			}
			if (this.head == null) { //第一个节点
				this.head = newNode;
				this.tail = newNode;
			} else {
				this.tail.next = newNode; //
				newNode.last = this.tail;
				this.tail = newNode;
			}
		}

		//移动节点到尾部，尾部的优先级高
		public void moveNodeToTail(Node<V> node) {
			if (this.tail == node) { //尾节点不用动
				return;
			}
			if (this.head == node) {  //头节点
				this.head = node.next;
				this.head.last = null;
			} else {
				node.last.next = node.next;
				node.next.last = node.last;
			}
			node.last = this.tail;
			node.next = null;
			this.tail.next = node;
			this.tail = node;
		}

		//移除头部
		public Node<V> removeHead() {
			if (this.head == null) {
				return null;
			}
			Node<V> res = this.head;
			if (this.head == this.tail) { //只有一个节点
				this.head = null;
				this.tail = null;
			} else {
				this.head = res.next;
				res.next = null;
				this.head.last = null;
			}
			return res;
		}
	}

	public static class MyCache<K, V> {
		private HashMap<K, Node<V>> keyNodeMap;
		private HashMap<Node<V>, K> nodeKeyMap;
		private NodeDoubleLinkedList<V> nodeList;
		private int capacity;

		public MyCache(int capacity) {
			if (capacity < 1) {
				throw new RuntimeException("should be more than 0.");
			}
			this.keyNodeMap = new HashMap<K, Node<V>>();
			this.nodeKeyMap = new HashMap<Node<V>, K>();
			this.nodeList = new NodeDoubleLinkedList<V>();
			this.capacity = capacity;
		}

		public V get(K key) {
			if (this.keyNodeMap.containsKey(key)) {
				Node<V> res = this.keyNodeMap.get(key);
				this.nodeList.moveNodeToTail(res);
				return res.value;
			}
			return null;
		}

		public void set(K key, V value) {
			if (this.keyNodeMap.containsKey(key)) {
				Node<V> node = this.keyNodeMap.get(key);
				node.value = value;
				this.nodeList.moveNodeToTail(node);
			} else {
				Node<V> newNode = new Node<V>(value);
				this.keyNodeMap.put(key, newNode);
				this.nodeKeyMap.put(newNode, key);
				this.nodeList.addNode(newNode);
				if (this.keyNodeMap.size() == this.capacity + 1) {
					this.removeMostUnusedCache();
				}
			}
		}

		private void removeMostUnusedCache() {
			Node<V> removeNode = this.nodeList.removeHead();
			K removeKey = this.nodeKeyMap.get(removeNode);
			this.nodeKeyMap.remove(removeNode);
			this.keyNodeMap.remove(removeKey);
		}

	}

	public static void main(String[] args) {
		MyCache<String, Integer> testCache = new MyCache<String, Integer>(3);
		testCache.set("A", 1);
		testCache.set("B", 2);
		testCache.set("C", 3);
		System.out.println(testCache.get("B"));
		System.out.println(testCache.get("A"));
		testCache.set("D", 4);
		System.out.println(testCache.get("D"));
		System.out.println(testCache.get("C"));

	}
}
```

<br/>

<br/>

### 题目：LFU

根据频度来决定，添加新元素的时候，查看之前每个元素的访问频率，到达容量的时候，把访问频率最小的元素删除；

当需要删除的时候，如果有多个元素的访问频率一致，就删除内个距离现在访问时间最远的内个；

get和set都是O(1)的操作

<br/>

数据结构

一个双向链表，取名次数链表，每个节点的值代表一个节点出现的次数，其中每个节点的下面再挂一个双向链表，代表具体的节点。一个具体的节点进来之后，根据它被访问的次数来决定挂在哪个次数链表节点上。

<br/>

具体步骤

put / get 的时候，先检查put或get的key在上述结构中是否存在：

如果没有，put就新增一个，新增到次数链表为1的节点上，get的时候没有，就返回null。

如果key在结构中存在，首先找到这个key是属于次数链表中哪个头的，然后把这个key从这个头链表中分离出来，然后看这个次数链表中次数+1的头结点是否存在：

如果不存在，创建出一个次数+1的头，如果存在但是不是次数+1的头，也需要创建，然后放在次数链表中这两个头中间；最后这个key挂在这个节点的尾部上。

如果存在，直接挂在这个头的尾部上就可以；

次数如果到size了，需要删除元素了，直接删除次数链表最左边的头下面挂的双向链表的头节点即可；因为每次put或get的时候，都会把对应的节点取出来，然后挂到次数链表的一个新的头节点（肯定是往右移了）挂着的双向链表的末尾上的。

<br/>

看图

{% asset_img picture23.jpg %}

<br/>

{% asset_img picture24.jpg %}

<br/>

代码

```java
public class Code_03_LFU {

    public static class Node { //次数链表下面挂的小链表
        public Integer key;
        public Integer value;
        public Integer times;  //次数
        public Node up;   //双向链表
        public Node down;

        public Node(int key, int value, int times) {
            this.key = key;
            this.value = value;
            this.times = times;
        }
    }

    public static class LFUCache {

        public static class NodeList { //一个次数链表的头+下面挂的子双向链表，就是竖着的一串；
            public Node head;
            public Node tail;
            public NodeList last;   //下一个竖着的链表，
            public NodeList next;

            public NodeList(Node node) { //必修给一个Node，才能创建NodeList，
                head = node;
                tail = node;
            }

            //新增的时候加到头部，说明头部是最新的；和讲的有些不一致，讲的是新增到尾部，一个意思；
            public void addNodeFromHead(Node newHead) {
                newHead.down = head;
                head.up = newHead;
                head = newHead;
            }

            public boolean isEmpty() {
                return head == null;
            }

            //删除一个节点，get或put一个节点的时候，要从一个NodeList删除，然后加到新的NodeList中
            public void deleteNode(Node node) {
                if (head == tail) {
                    head = null;
                    tail = null;
                } else {
                    if (node == head) {
                        head = node.down;
                        head.up = null;
                    } else if (node == tail) {
                        tail = node.up;
                        tail.down = null;
                    } else {
                        node.up.down = node.down;
                        node.down.up = node.up;
                    }
                }
                node.up = null;
                node.down = null;
            }
        }

        private int capacity;  //容量。
        private int size;   // 当前大小
        private HashMap<Integer, Node> records; // key(Integer) -> Node
        private HashMap<Node, NodeList> heads;  //get的时候要查询一个Node属于哪个NodeList，就是找头，
        private NodeList headList;  //整个结构中第一个 NodeList 是谁， 整个结构由多个NodeList组成

        public LFUCache(int capacity) {
            this.capacity = capacity;
            this.size = 0;
            this.records = new HashMap<>();
            this.heads = new HashMap<>();
            headList = null;
        }

        // 核心逻辑
        public void set(int key, int value) {
            if (records.containsKey(key)) { //说明之前加过
                Node node = records.get(key);
                node.value = value;
                node.times++;
                NodeList curNodeList = heads.get(node); //属于哪个NodeList，
                move(node, curNodeList); //从原来的NodeList中删除，放到下一个 NodeList 上；
            } else {  //新的节点
                if (size == capacity) {  //需要删除了
                    //找到该删的节点，都删除，并消除它的影响。
                    Node node = headList.tail;  // headList 的尾节点是需要删除的，因为新增的时候是新增到头部的；
                    headList.deleteNode(node);
                    modifyHeadList(headList);//如果这个headList为空了，需要把headList换成新的；
                    records.remove(node.key);
                    heads.remove(node);
                    size--;
                }
                Node node = new Node(key, value, 1);//新节点
                if (headList == null) {     //整个大结构都没有生成
                    headList = new NodeList(node);
                } else {
                    if (headList.head.times.equals(node.times)) {  //看有没有词频为1的头节点
                        headList.addNodeFromHead(node);
                    } else {                                    //没有的话需要新建出来一个词频为1的头节点headList；
                        NodeList newList = new NodeList(node);
                        newList.next = headList;
                        headList.last = newList;
                        headList = newList;
                    }
                }
                records.put(key, node);
                heads.put(node, headList);
                size++;
            }
        }

        /**
         * 把 @param node 从老的 @param oldNodeList 移动到新的nodelist上
         */
        private void move(Node node, NodeList oldNodeList) {
            oldNodeList.deleteNode(node);

            //node 即将进入一个新的nodelist中，那么这个新的nodelist 的上一个链表是谁呢？如果oldNodeList删完之后没有元素了，那么
            //上一个链表就是oldNodeList的上一个nodelist，否则就是oldNodeList本身，
            NodeList preList = modifyHeadList(oldNodeList) ? oldNodeList.last
                    : oldNodeList;
            //新的nodelist的下一个nodelist
            NodeList nextList = oldNodeList.next;
            if (nextList == null) {     //说明oldNodeList就是整个结构的尾部，这个node的词频加完之后是最高的
                NodeList newList = new NodeList(node);

                //下面两个if其实就是前后重连的意思，
                if (preList != null) {
                    preList.next = newList;
                }
                newList.last = preList;
                if (headList == null) {
                    headList = newList;
                }
                heads.put(node, newList);
            } else {        //说明oldNodeList不是整个结构的尾部，此时就需要nodelist之间的前后重连了
                if (nextList.head.times.equals(node.times)) {  //下一个nodelist正好是次数+1的
                    nextList.addNodeFromHead(node); //直接挂在nextList的头部
                    heads.put(node, nextList);
                } else {
                    NodeList newList = new NodeList(node);  //下一个nodelist是更大的，需要在中间加一个，需要重连；
                    if (preList != null) {
                        preList.next = newList;
                    }
                    newList.last = preList;
                    newList.next = nextList;
                    nextList.last = newList;
                    if (headList == nextList) {
                        headList = newList;
                    }
                    heads.put(node, newList);
                }
            }
        }

        // delete完之后，是否需要把整个链表都删掉；
        private boolean modifyHeadList(NodeList nodeList) {
            if (nodeList.isEmpty()) {  // 如果当前nodeList为空了，就删掉这个 NodeList
                if (headList == nodeList) { // 是否是整个结构的头部，需要换头
                    headList = nodeList.next;
                    if (headList != null) {
                        headList.last = null;
                    }
                } else {
                    //删除当前nodelist，然后重连 nodelist
                    nodeList.last.next = nodeList.next;
                    if (nodeList.next != null) {
                        nodeList.next.last = nodeList.last;
                    }
                }
                return true;
            }
            return false;
        }

        public Integer get(int key) {
            if (!records.containsKey(key)) {
                return null;  // 不存在
            }
            Node node = records.get(key);
            node.times++;
            NodeList curNodeList = heads.get(node);
            move(node, curNodeList); //从原来的NodeList中删除，放到下一个 NodeList 上；
            return node.value;
        }
    }
}
```

<br/>

<br/>

### 题目：计算公式的值

给定一个字符串str，str表示一个公式，公式里可能有整数、加减乘除符号和 左右括号，返回公式的计算结果。 【举例】 str="48*((70-65)-43)+8*1"，返回-1816。 str="3+1*4"，返回7。 str="3+(1*4)"，返回7。 

【说明】 

1．可以认为给定的字符串一定是正确的公式，即不需要对str做公式有效性检查。

 2．如果是负数，就需要用括号括起来，比如"4*(-3)"。但如果负数作为公式的 开头或括号部分的开头，则可以没有括号，比如"-3*4"和"(-3*4)"都是合法的。

 3．不用考虑计算过程中会发生溢出的情况

<br/>

过程

1. 没有()

准备一个栈，所有的字符一次扔进栈中，数字进栈的时候，先看栈顶是不是乘除，如果是，取出栈顶的数字和符号，和当前数字进行计算，然后再放进栈中；最后栈中只剩下加减了

{% asset_img picture25.jpg %}

<br/>

2. 有()

准备一个函数，参数是一个string和一个index，函数的作用是计算这个string从index位置开始的计算，一直计算到遇到右括号或者string末尾，这个函数结束

例如：针对下面的string，f(5)只计算3+1

{% asset_img picture26.jpg %}

<br/>

那么整个的计算逻辑就是直接调用f(0)，遇到左括号，就直接调用相应位置的子过程，例如下图，就是调用f(5)，f(5)计算过程中，又遇到左括号，所以计算f(10)，f(10)返回给f(5)，f(5)返回个f(0)，在调用子过程结束后，子过程会返回两个值，一个是计算结果，另一个是父过程要从哪个位置开始继续计算。然后父过程继续计算，最终返回结果；

{% asset_img picture27.jpg %}

<br/>

代码

```java
public class Code_07_ExpressionCompute {

    public static int getValue(String str) {
        return value(str.toCharArray(), 0)[0];
    }

    /**
     * 递归函数
     * @param str
     * @param i index 从哪个位置开始算；
     * @return 两个值，第一个值代表子过程的返回结果，第二个是父过程要从哪个位置继续计算
     */
    public static int[] value(char[] str, int i) {
        LinkedList<String> que = new LinkedList<String>();
        int pre = 0; //收集数字用的，因为多位的数字可能是31+2+7，31本身是一个数字，但是是两个字符，计算的时候需要还原成31
        int[] bra = null;
        while (i < str.length && str[i] != ')') {
            if (str[i] >= '0' && str[i] <= '9') {  //如果遇到的一直是数字，就一直累加上，计算出原本的数字31；而不是3
                pre = pre * 10 + str[i++] - '0';
            } else if (str[i] != '(') { //遇到了运算符号 +-*/
                addNum(que, pre);
                que.addLast(String.valueOf(str[i++]));  //收集到栈中；
                pre = 0;  //等待处理下一个数字
            } else {  //遇到了  (
                bra = value(str, i + 1);  //递归
                pre = bra[0];
                i = bra[1] + 1; //从 bra[1] + 1 位置开始继续计算
            }
        }
        addNum(que, pre);
        return new int[]{getNum(que), i};
    }

    public static void addNum(LinkedList<String> que, int num) {
        if (!que.isEmpty()) {
            int cur = 0;
            String top = que.pollLast();
            if (top.equals("+") || top.equals("-")) {
                que.addLast(top);
            } else {
                cur = Integer.valueOf(que.pollLast());
                num = top.equals("*") ? (cur * num) : (cur / num);
            }
        }
        que.addLast(String.valueOf(num));
    }

    public static int getNum(LinkedList<String> que) {
        int res = 0;
        boolean add = true;
        String cur = null;
        int num = 0;
        while (!que.isEmpty()) {
            cur = que.pollFirst();
            if (cur.equals("+")) {
                add = true;
            } else if (cur.equals("-")) {
                add = false;
            } else {
                num = Integer.valueOf(cur);
                res += add ? num : (-num);
            }
        }
        return res;
    }

    public static void main(String[] args) {
        String exp = "48*((70-65)-43)+8*1";
        System.out.println(getValue(exp));

        exp = "4*(6+78)+53-9/2+45*8";
        System.out.println(getValue(exp));

        exp = "10-5*3";
        System.out.println(getValue(exp));

        exp = "-3*4";
        System.out.println(getValue(exp));

        exp = "3+1*4";
        System.out.println(getValue(exp));
    }
}

```

<br/>

<br/>

### 题目：求子数组的最大异或和

给定一个数组，求子数组的最大异或和。 

一个数组的异或和为，数组中所有的数异或起来的结果。

<br/>

暴力解法  O(n^3)

```java
public static int getMaxE1(int[] arr){
    int max = Integer.MIN_VALUE;
    for(int i = 0; i < arr.length; i++){  //每个位置以i结尾
        for(int start = 0; start <= i; start++){  //开头和结尾定义好了；
            int res = 0;
            for (int k = start; k <= i; k++){
                res ^= arr[k];
            }
            max = Math.max(max,res);
        }
    }
    return max;
}

```

<br/>

异或的一个规律

E1^E2 = E3

则 E1^E3=E2，E2^E3=E1

所以start到i的异或和 = （0到i的异或和） ^ （0到start-1的异或和）

解法  O(n^2)

```java
/**
	 * E1^E2 = E3
	 * 则 E1^E3=E2，E2^E3=E1
	 */
public static int getMaxE2(int[] arr){
    int max = Integer.MIN_VALUE;
    int[] dp = new int[arr.length];
    int eor = 0;
    //枚举所有以i结尾的子数组
    for(int i = 0; i < arr.length; i++){  //每个位置以i结尾
        eor ^= arr[i];  //0..i的异或值
        max = Math.max(max,eor);
        //0到i的结果有了，再处理1到i，2到i，。。。。
        for(int start = 1; start <= i; start++){  //开头和结尾定义好了；
            int curEor = eor ^ dp[start - 1];
            max = Math.max(max,curEor);
        }
        dp[i] = eor;
    }
    return max;
}

```

<br/>

解法  O(n)

上面两种做法求0到i的异或结果的时候，都是从0到i计算一遍，1到i计算一遍，2到i计算一遍，等等，循环计算的过程中，收集最大结果，每次的计算结果都是独立的，这是导致效率低下的原因， 也就是要优化这个过程。

我们希望有一个黑盒，里面存放着0到0的异或结果，0到1的异或结果，0到2的异或结果…..0到i-1的异或结果，0到i的异或结果我们是每次都在计算的，是EOR，这个黑盒可以帮我们选择出哪个值和EOR异或之后的结果最大，例如0到3的异或结果和EOR异或出来的结果最大， 那4到i就是得到最大异或和的子数组（也是上述的规律）。

{% asset_img picture28.jpg %}

<br/>

这个黑盒就是前缀树，假设整数是4位二进制组成的（实际32位），方便画图，最高位是符号位，假设0到3的EOR和是0000，因为我们找的是最终的EOR最大的，所以我们肯定希望最高位是0，所以我们会首先选择分支为0的路，剩下的每一步都希望异或的结果是1，所以就选择和对应位上的数字不一致的数字，所以第二三四步走的时候，都优先选择为1的路，如果没有，就选走0的路，这样最终选择出来的数字就是和0到i的数组异或和最大的；

如果符号位是1，即是负数，那么除了最高位的选择有特殊性之外，其他的计算过程和正数是一致的，也是优先选择异或之后结果为1的，因为负数求的是补码，高位为1，代表的值也是大的。

{% asset_img picture29.jpg %}

<br/>

代码

```java
public class Code_05_Max_EOR {

	public static int getMaxE1(int[] arr){
		int max = Integer.MIN_VALUE;
		//前两个循环枚举所有以i结尾的子数组
		for(int i = 0; i < arr.length; i++){  //每个位置以i结尾
			for(int start = 0; start <= i; start++){  //开头和结尾定义好了:0到i，1到i，2到i，等等
				int res = 0;
				//0到i的异或结果怎么求，从start到i再遍历一遍；
				for (int k = start; k <= i; k++){
					res ^= arr[k];
				}
				max = Math.max(max,res);
			}
		}
		return max;
	}

	/**
	 * E1^E2 = E3
	 * 则 E1^E3=E2，E2^E3=E1
	 */
	public static int getMaxE2(int[] arr){
		int max = Integer.MIN_VALUE;
		int[] dp = new int[arr.length];
		int eor = 0;
		//枚举所有以i结尾的子数组
		for(int i = 0; i < arr.length; i++){  //每个位置以i结尾
			eor ^= arr[i];  //0..i的异或值
			max = Math.max(max,eor);
			//0到i的结果有了，再处理1到i，2到i，。。。。
			for(int start = 1; start <= i; start++){  //开头和结尾定义好了；
				int curEor = eor ^ dp[start - 1];
				max = Math.max(max,curEor);
			}
			dp[i] = eor;
		}
		return max;
	}


	public static class Node {
		public Node[] nexts = new Node[2]; //0的路还是1的路
	}

	/**
	 * 加一个数还是选择一个数都是常数级别的；
	 */
	public static class NumTrie {
		public Node head = new Node();

		//把num对应的二进制提出来，加到前缀树，
		public void add(int num) {
			Node cur = head;
			// for循环就是依次往下建立节点，
			for (int move = 31; move >= 0; move--) {
				int path = ((num >> move) & 1);  //每次取出最高位，
				cur.nexts[path] = cur.nexts[path] == null ? new Node() : cur.nexts[path];
				cur = cur.nexts[path];
			}
		}

		/**
		 * @param num  从0到i的异或结果
		 * @return  最优结果，
		 */
		public int maxXor(int num) {
			Node cur = head;
			int res = 0;
			for (int move = 31; move >= 0; move--) {
				int path = (num >> move) & 1;  //从高位到低位依次提取出0和1
				//对于符号位，希望选的路和二进制最高位是一样的，这样异或出来的最高位就是0，代表正数
				//对于剩下的其他位，希望选的路和当前的二进制位数不一样，这样异或出来的当前位的二进制数就是1，大，
				int best = move == 31 ? path : (path ^ 1);	//期望选择的路
				best = cur.nexts[best] != null ? best : (best ^ 1); //实际选择的路
				res |= (path ^ best) << move;  //设置每一位的答案
				cur = cur.nexts[best]; //继续往下走
			}
			return res;
		}

	}

	public static int maxXorSubarray(int[] arr) {
		if (arr == null || arr.length == 0) {
			return 0;
		}
		int max = Integer.MIN_VALUE;
		int eor = 0;
		NumTrie numTrie = new NumTrie(); //存放着之前所有从0到i-1的计算结果，
		numTrie.add(0);
		for (int i = 0; i < arr.length; i++) {
			eor ^= arr[i];  	//eor永远是从0到i的异或和；
			max = Math.max(max, numTrie.maxXor(eor));  //黑盒直接返回最大的
			numTrie.add(eor);
		}
		return max;
	}

	// for test
	public static int comparator(int[] arr) {
		if (arr == null || arr.length == 0) {
			return 0;
		}
		int max = Integer.MIN_VALUE;
		for (int i = 0; i < arr.length; i++) {
			int eor = 0;
			for (int j = i; j < arr.length; j++) {
				eor ^= arr[j];
				max = Math.max(max, eor);
			}
		}
		return max;
	}

	// for test
	public static int[] generateRandomArray(int maxSize, int maxValue) {
		int[] arr = new int[(int) ((maxSize + 1) * Math.random())];
		for (int i = 0; i < arr.length; i++) {
			arr[i] = (int) ((maxValue + 1) * Math.random()) - (int) (maxValue * Math.random());
		}
		return arr;
	}

	// for test
	public static void printArray(int[] arr) {
		if (arr == null) {
			return;
		}
		for (int i = 0; i < arr.length; i++) {
			System.out.print(arr[i] + " ");
		}
		System.out.println();
	}

	// for test
	public static void main(String[] args) {
		int testTime = 500000;
		int maxSize = 30;
		int maxValue = 50;
		boolean succeed = true;
		for (int i = 0; i < testTime; i++) {
			int[] arr = generateRandomArray(maxSize, maxValue);
			int res = maxXorSubarray(arr);
			int comp = comparator(arr);
			if (res != comp) {
				succeed = false;
				printArray(arr);
				System.out.println(res);
				System.out.println(comp);
				break;
			}
		}
		System.out.println(succeed ? "Nice!" : "Fucking fucked!");
	}
}

```

