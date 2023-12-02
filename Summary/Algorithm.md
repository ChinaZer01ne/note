#flashcards 
# 链表
TODO
# 栈
## 单调栈
### 单调递增还是单调递减
* 如果求**当前元素后面第一个比他大的元素**就是**单调递增**
	* 当前元素比栈顶元素小则入栈否则出栈
	>如果找到比栈顶大的元素，就会处理，只有小的时候会入栈，所以栈内是单调递增的
* 如果求**当前元素后面第一个比他小的元素**就是**单调递减**
	* 当前元素比栈顶元素大则入栈否则出栈
	>如果找到比栈顶小的元素，就会处理，只有大的时候会入栈，所以栈内是单调递减的
	
### 单调栈解决什么问题
* 求比当前元素大（或小）的下一个元素
* 单调栈可以同时找到比当前元素大（或小）的左右两边的元素
### [每日温度](https://leetcode.cn/problems/daily-temperatures/)
### [接雨水](https://leetcode.cn/problems/trapping-rain-water/)
#### **整体思路**
找到当前元素左边和右边比自己大的元素，能形成凹槽。
#### 思路一：前缀最大值 + 后缀最大值 + 纵向求解
#### 思路二：使用双指针优化掉前缀最大值和后缀最大值数组
#### 思路二：单调栈 + 横向求解
### [柱状图中最大的矩形](https://leetcode.cn/problems/largest-rectangle-in-histogram/)
#### **整体思路**
找到当前元素左边和右边比自己小的元素，能形成连续的矩形。
# 队列
## 单调队列

# 树
## 遍历方式
树的递归遍历，每个节点都会到达三次。第一次是进入当前节点，第二次是从左子树返回，第三次是从右子树返回。在不同的位置增加代码会有不同的表现。
例如：以下是计数的代码，最终`countMap`中记录的节点数量就是真实数量，而不是节点的三倍，因为我们只在第一次进入当前节点的时候做了操作，第二次和第三次返回到当前节点的时候并没有操作。
```java
Map<Integer, Integer> countMap = new HashMap<>();
public void travel(TreeNode root) {  
	if (root == null) {  
		return;  
	}  
	countMap.put(root.val, countMap.getOrDefault(root.val, 0) + 1);  
	travel(root.left);  
	travel(root.right);  
}
```

### [前序遍历](https://leetcode.cn/problems/binary-tree-preorder-traversal/)
```java
public List<Integer> preorderTraversal(TreeNode root) {  
    if (root == null) {  
        return new ArrayList<>();  
    }  
    List<Integer> list = new ArrayList<>();  
    // 根 左 右  
    Deque<TreeNode> stack = new ArrayDeque<>();  
    stack.push(root);  
    // 先压右，再压左  
    while (!stack.isEmpty()) {  
        TreeNode cur = stack.pop();  
        if (cur.right != null) {  
            stack.push(cur.right);  
        }  
        if (cur.left != null) {  
            stack.push(cur.left);  
        }  
        list.add(cur.val);  
    }  
    return list;  
}

```
### [中序遍历](https://leetcode.cn/problems/binary-tree-inorder-traversal/)
* 中序遍历的顺序是：左 根 右  
* 因为需要先访问左子树，那么需要将左边界全部入栈  
* 入栈后的栈顶元素就是我们需要访问的最左元素  
* 访问元素后，我们需要访问其右子树, 如果右子树不为空，需要将右子树左边全部入栈，否则继续访问下一个栈顶元素
```java
public List<Integer> inorderTraversal(TreeNode root) {  
    if (root == null) {  
        return new ArrayList<>();  
    }  
    Deque<TreeNode> stack = new ArrayDeque<>();  
    List<Integer> res = new ArrayList<>();  
  
    while (root != null || !stack.isEmpty()) {  
        if (root != null) {  
            // 将左子树左边全部入栈  
            stack.push(root);  
            root = root.left;  
        } else {  
            TreeNode cur = stack.pop();  
            // 访问元素  
            res.add(cur.val);  
            if (cur.right != null) {  
                root = cur.right;  
            }  
        }  
    }  
    return res;  
}
```
### [后序遍历](https://leetcode.cn/problems/binary-tree-postorder-traversal/)
#### 思路一：
* 前序遍历（根左右），用栈先压右子树，在压左子树；
* 如果我们先压左子树，在压右子树，那么打印的就是根右左的顺序
* 最后整体翻转数组，就是左右根，后序遍历。  
```java
public List<Integer> postorderTraversal(TreeNode root) {  
    if (root == null) {  
        return new ArrayList<>();  
    }  
    List<Integer> res = new ArrayList<>();  
    Deque<TreeNode> stack = new ArrayDeque<>();  
    stack.push(root);  
  
    // 根 右 左  
    while (!stack.isEmpty()) {  
        TreeNode cur = stack.pop();  
        // 访问根  
        res.add(cur.val);  
        if (cur.left != null) {  
            stack.push(cur.left);  
        }  
        if (cur.right != null) {  
            stack.push(cur.right);  
        }  
    }  
    // 左 右 根  
    Collections.reverse(res);  
    return res;  
}

```
#### 思路二：
* 先将树的左边界压到栈中，因为最左的节点是要访问的节点。
* 当节点出栈（访问）的时候，要先判断出栈节点的右子树是不是空或者已经访问过，如果没有访问过，则压入右子树，否则则访问根节点。  
* 出栈（访问）节点的时候，要记录访问节点，避免重复访问。  
```java
public List<Integer> postorderTraversal(TreeNode root) {  
    if (root == null) {  
        return new ArrayList<>();  
    }  
    List<Integer> res = new ArrayList<>();  
    Deque<TreeNode> stack = new ArrayDeque<>();  
    TreeNode lastestAccessNode = null;  
    while (root != null || !stack.isEmpty()) {  
        if (root != null) {  
            stack.push(root);  
            root = root.left;  
        } else {  
            // 说明root == null，树左边的边都压到了栈中  
            // 访问右子树  
            TreeNode cur = stack.peek();  
            if (cur.right == null || cur.right == lastestAccessNode) {  
                // 右子树为空或者右子树已经访问过了，则访问根节点  
                stack.pop();  
                res.add(cur.val);  
                lastestAccessNode = cur;  
            } else {  
                // 右子树不为空，则需要访问右子树，将右子树的左边压到栈中  
                root = cur.right;  
            }  
        }  
    }  
    return res;  
}
```
### [层序遍历](https://leetcode.cn/problems/binary-tree-level-order-traversal/)
```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        if (root == null) {
            return new ArrayList<>();
        }
        List<List<Integer>> result = new ArrayList<>();
        Deque<TreeNode> queue = new ArrayDeque<>();
        queue.add(root);

        while(!queue.isEmpty()) {
            int levelSize = queue.size();
            List<Integer> levelList = new ArrayList<>();
            while(levelSize-- > 0) {
                TreeNode node = queue.poll();
                levelList.add(node.val);
                TreeNode left = node.left;
                if(left != null) {
                    queue.add(left);
                }
                TreeNode right = node.right;
                if(right != null) {
                    queue.add(right);
                }
            }
            result.add(levelList);
        }
        
        return result;
    }
}
```
## [翻转二叉树](https://leetcode.cn/problems/invert-binary-tree/)
## [对称二叉树](https://leetcode.cn/problems/symmetric-tree/)
## [完全二叉树节点的数量](https://leetcode.cn/problems/count-complete-tree-nodes/)
## [二叉树的所有路径](https://leetcode.cn/problems/binary-tree-paths/)
## [左叶子之和](https://leetcode.cn/problems/sum-of-left-leaves/)
## [从中序与后续遍历序列构造二叉树](https://leetcode.cn/problems/construct-binary-tree-from-inorder-and-postorder-traversal/)
##  [二叉搜索树中的众数](https://leetcode.cn/problems/find-mode-in-binary-search-tree/)
## [删除二叉搜索树中的节点](https://leetcode.cn/problems/delete-node-in-a-bst/)
## [修剪二叉搜索树](https://leetcode.cn/problems/trim-a-binary-search-tree/)
## [把二叉搜索树转换为累加树](https://leetcode.cn/problems/w6cpku/)
## [二叉树的最近公共祖先](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree)
## [二叉搜索树的最近公共祖先](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-search-tree)

# 回溯
## 子集相关

## 组合相关

## 排列相关

## 二维相关

# 动态规划
## 经典动态规划
### 解题思路
#### 间接
从递归到动态规划
#### 直接
* 确定dp数组含义
* 找状态转移方程
* 初始化dp数组
* dp数组遍历顺序
* 打印dp数组
### 题目
#### [不同路径 II](https://leetcode.cn/problems/unique-paths-ii)
#### [整数拆分](https://leetcode.cn/problems/integer-break)
#### [不同的二叉搜索树](https://leetcode.cn/problems/unique-binary-search-trees)

## 背包问题
### 01背包问题
题目描述：每个物品使用一次，装满背包获取的最大价值。

dp含义：`0 - i`位置装容量为`j`的背包获取到的最大价值
* 二维dp：`dp[i][j] = max(dp[i - 1][j], dp[i - 1][j - weight[i]] + value[i])`
* 压缩数组：`dp[j] = max(dp[j], dp[j - weight[i]] + value[i])`
	* 先遍历物品再遍历背包，背包从后往前遍历，保证物品只用一次

#### 题目
##### [分割等和子集](https://leetcode.cn/problems/partition-equal-subset-sum)
* 装容量为`j`的背包是否能装满（最大容量刚好等于`sum / 2`）: `dp[j] = max(dp[j], dp[j - nums[i]] + nums[i])`
##### [最后一块石头的重量 II](https://leetcode.cn/problems/last-stone-weight-ii)
* 装容量为`j`的背包的最大装多少（`dp[sum / 2]`）：`dp[j] = max(dp[j], dp[j - nums[i]] + nums[i])`
##### [目标和](https://leetcode.cn/problems/target-sum)
* 装满容量`j`的背包有多少种方法（`dp[(target + sum) / 2]`）：`dp[j] += dp[j - nums[i]]`
##### [一和零](https://leetcode.cn/problems/ones-and-zeroes)
* 二维背包问题：`dp[i][j] = max(dp[i][j], dp[i - x][j - y])`

### 数组压缩技巧

注意遍历顺序
* 先遍历物品还是先遍历背包
* 从前往后遍历还是从后往前遍历（覆盖问题）

### 完全背包
相比01背包而言，完全背包同一物品可以使用多次。

题目描述：每个物品使用多次，装满背包获取的最大价值。
dp含义：`0 - i`位置装容量为`j`的背包获取到的最大价值
* 二维dp：`dp[i][j] = max(dp[i - 1][j], dp[i - 1][j - weight[i]] + value[i])`
* 压缩数组：`dp[j] = max(dp[j], dp[j - weight[i]] + value[i])`
	* 先遍历物品再遍历背包，背包从前往后遍历，保证物品可以重复使用
#### 问题
##### [零钱兑换 II](https://leetcode.cn/problems/coin-change-ii)
* 装满容量为`j`的背包有`dp[i]`种方法
* 求组合
* **遍历顺序影响排列还是组合**
	* 先遍历物品再遍历背包：组合
	* 先遍历背包再遍历物品：排列
##### [组合总和 Ⅳ](https://leetcode.cn/problems/combination-sum-iv)
* 装满容量为`j`的背包有`dp[i]`种方法
* 求排列
##### 一步爬1~m层楼梯，求多少方法？
* 求排列
#### [零钱兑换](https://leetcode.cn/problems/coin-change)
* 装满容量为`j`的背包最少用`dp[j]`个的物品
* 遍历顺序无要求
#### [单词拆分](https://leetcode.cn/problems/word-break)
* 回溯可解
* 动态规划可解
	* 完全背包：数组字符是否能组成目标串
#### [打家劫舍](https://leetcode.cn/problems/house-robber)
#### [打家劫舍 II](https://leetcode.cn/problems/house-robber-ii)
#### [打家劫舍 III](https://leetcode.cn/problems/house-robber-iii)
#### [买卖股票的最佳时机](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock)
#### [买卖股票的最佳时机 II](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-ii)
#### [买卖股票的最佳时机 III](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iii)
#### [买卖股票的最佳时机 IV](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iv)
#### [买卖股票的最佳时机含冷冻期](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-cooldown)
#### [买卖股票的最佳时机含手续费](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-transaction-fee)
#### [最长递增子序列](https://leetcode.cn/problems/longest-increasing-subsequence)
#### [最长连续递增序列](https://leetcode.cn/problems/longest-continuous-increasing-subsequence)
#### [最长重复子数组](https://leetcode.cn/problems/maximum-length-of-repeated-subarray)
#### [最长公共子序列](https://leetcode.cn/problems/longest-common-subsequence)
#### [不相交的线](https://leetcode.cn/problems/uncrossed-lines)
#### [最大子数组和](https://leetcode.cn/problems/maximum-subarray)
#### [判断子序列](https://leetcode.cn/problems/is-subsequence/)
#### [不同的子序列](https://leetcode.cn/problems/distinct-subsequences/)
#### [两个字符串的删除操作](https://leetcode.cn/problems/delete-operation-for-two-strings/)
#### [编辑距离](https://leetcode.cn/problems/edit-distance/)
#### [回文子串](https://leetcode.cn/problems/palindromic-substrings/)
#### [最长回文子序列](https://leetcode.cn/problems/longest-palindromic-subsequence/)
# 贪心

## 问题

### [分发饼干](https://leetcode.cn/problems/assign-cookies/)
### [摆动序列](https://leetcode.cn/problems/wiggle-subsequence/)
### [最大子数组和](https://leetcode.cn/problems/maximum-subarray/)
### [买卖股票的最佳时机 II](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-ii)
### [跳跃游戏](https://leetcode.cn/problems/jump-game/)
### [跳跃游戏 II](https://leetcode.cn/problems/jump-game-ii/)
### [K 次取反后最大化的数组和](https://leetcode.cn/problems/maximize-sum-of-array-after-k-negations/)
### [加油站](https://leetcode.cn/problems/gas-station/)
### [分发糖果](https://leetcode.cn/problems/candy/)
### [柠檬水找零](https://leetcode.cn/problems/lemonade-change/)
### [根据身高重建队列](https://leetcode.cn/problems/queue-reconstruction-by-height/)
### [用最少数量的箭引爆气球](https://leetcode.cn/problems/minimum-number-of-arrows-to-burst-balloons/)
### [无重叠区间](https://leetcode.cn/problems/non-overlapping-intervals/)
### [划分字母区间](https://leetcode.cn/problems/partition-labels/)
### [合并区间](https://leetcode.cn/problems/merge-intervals/)
### [单调递增的数字](https://leetcode.cn/problems/monotone-increasing-digits/)
### [监控二叉树](https://leetcode.cn/problems/binary-tree-cameras/)