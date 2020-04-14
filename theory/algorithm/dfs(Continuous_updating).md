# 深度搜索算法

深度搜索算法绝对是大小面试中最常用的算法。我做题的经验，至少50%的题目，可以通过深度搜索的思想去解决（不一定是最优解）。深度搜索也有很多变种，我也在持续学习中，这篇文章会分享部分常见的套路。

和bfs不一样，dfs没有特别固定的代码模板，但是可以分为几个大类，找出其中的共性。


## 简单树类的深度搜索

这类题目属于大公司的热身题目，中小公司的常考题目，难度不高。但如果长时间没练习算法，也容易卡壳。

### 模板代码

```go
// 中序遍历
func dfs(root *TreeNode) {
    if root == nil {
        return
    }
    dfs(root.Left)
    dosomthing()
    dfs(root.Right)
}
```

简单树类的dfs基本都是前，中，后序遍历的变形。要对这段代码深刻理解，可以自己画图看看具体遍历流程。

### 实际运用
```
给定两个二叉树，编写一个函数来检验它们是否相同。

如果两个树在结构上相同，并且节点具有相同的值，则认为它们是相同的。

输入:       1         1
          / \       / \
         2   3     2   3

        [1,2,3],   [1,2,3]

输出: true

输入:      1          1
          /           \
         2             2

        [1,2],     [1,null,2]

输出: false

输入:       1         1
          / \       / \
         2   1     1   2

        [1,2,1],   [1,1,2]

输出: false
```
分析

> 要使二叉树相同，那么我们判断每个节点相同就可以了。这里可以利用前序遍历，先检测节点是否相同，如果相同继续往下递归搜索。

代码
```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func isSameTree(p *TreeNode, q *TreeNode) bool {
    if (p == nil && q!= nil) || (p != nil && q == nil) {
        return false
    }

    if p == nil && q == nil {
        return true
    }
    if p.Val != q.Val {
        return false
    }
    // 将dosomething() 替换成需要做的事，这里是比较节点是否相同
    return isSameTree(p.Left, q.Left) && isSameTree(p.Right, q.Right)

}
```
算法思路很简单，有一点需要注意，终止条件一定要好好考虑，很多错误都是因为终止条件编写正确。

练习

[相同的树](https://leetcode-cn.com/problems/same-tree/)


```
二叉树最近的公共祖先

给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。

百度百科中最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”

例如，给定如下二叉树:  root = [3,5,1,6,2,0,8,null,null,7,4]

输入: root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 1
输出: 3
解释: 节点 5 和节点 1 的最近公共祖先是节点 3。
```
![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/15/binarytree.png)


分析
> 如果先找到p，q对应的节点，再从下往上推导得出公共节点，理论上是可行。但实际操作基本不可能，这和树的指针方向相反。这种思考方式是dfs中的自底向上。需要逆向思考，顺着根节点往下推导，自顶向下就能很方便解决问题。如果p, q分别在root两侧，那么root节点就是p，q的根节点。如果p或者q，等于root，当然也是root节点。这三种情况都不是，就去左子树或者右子树寻找。

实现
```go
/**
 * Definition for TreeNode.
 * type TreeNode struct {
 *     Val int
 *     Left *ListNode
 *     Right *ListNode
 * }
 */
 func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
     if root == nil || root == p || root == q {
         return root
     }
    
    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)
     /* 如果左右节点都存在，说明p,q分布在root两侧，那么就是root。
        如果某一个存在，那么存在的就是公共节点，说明p，q分布在同一侧。
     */
    if left != nil && right != nil {
         return root
     }
     if left == nil {
         return right
     }
     if right == nil {
         return left
     }
     return nil
}
```
lowestCommonAncestor实际上有两个作用。1. 返回最近公共节点。 2. 返回节点自身。这和固化思维里面，递归函数只能做一件事有所不同。

[二叉树最近的公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)


## 二维矩阵中的搜索

这类题目也是常考题型，通常会给一个二维矩阵，让根据各种条件去遍历。

### 模板代码
```go
def dfs(board [][]int, i, j int) {
    // 首先判断是否有越界
    if i < 0 || i >= len(board) {
        return
    }

    if j < 0 || j >= len(board[0]) {
        return
    }

    if board[i][j] == '-' {
        return
    }

    tmp := board[i][j]
    board[i][j] = '-' //重要技巧，占位处理，表明这个位置已经访问过了。如果不做占位会陷入死循环
    dfs(board, i+1, j)
    ...
    dfs(board, i, j+1) // 遍历其他方向

    board[i][j] = tmp //恢复原状
}
```

```
矩阵中的路径

请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。路径可以从矩阵中的任意一格开始，每一步可以在矩阵中向左、右、上、下移动一格。如果一条路径经过了矩阵的某一格，那么该路径不能再次进入该格子。例如，在下面的3×4的矩阵中包含一条字符串“bfce”的路径（路径中的字母用加粗标出）。

[["a","b","c","e"],
["s","f","c","s"],
["a","d","e","e"]]

但矩阵中不包含字符串“abfb”的路径，因为字符串的第一个字符b占据了矩阵中的第一行第二个格子之后，路径不能再次进入这个格子

输入：board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word = "ABCCED"
输出：true

输入：board = [["a","b"],["c","d"]], word = "abcd"
输出：false
```

分析

> 从矩阵中找一条路径，典型的dfs搜索，可以直接套用模板

```go
func exist(board [][]byte, word string) bool {
    for i := 0; i < len(board); i++ {
        for j := 0; j < len(board[0]); j++ {
            if board[i][j] == word[0] {
                if dfs(board, word, i, j, 0) {
                    return true
                }
            }
        }
    }
    return false
}


func dfs(board [][]byte, word string, i, j, depth int) bool {
    if len(word) <= depth {
        return true
    }

    if i < 0 || i >= len(board) {
        return false
    }

    if j < 0 || j >= len(board[0]) {
        return false
    }
    if board[i][j] == '-' || board[i][j] != word[depth]{
        return false
    }
    tmp := board[i][j]
    board[i][j] = '-'
    flag := dfs(board, word, i+1, j, depth+1) || dfs(board, word, i-1,j, depth+1) || dfs(board, word, i, j+1, depth+1) || dfs(board, word, i, j-1, depth+1)
    board[i][j] = tmp
    return flag
}
```

练习

[矩阵中的路径](https://leetcode-cn.com/problems/ju-zhen-zhong-de-lu-jing-lcof/)

## 回溯算法（不剪枝）

这些问题不会直接给一棵树或者一个矩阵，但是我们可以用相似的思维，将所有情况按一颗树的模型展开。

### 模板代码
```python
def dfs(nums: List[int], depth: int):
    if depth == len(nums):
        dosomething()
        return

    for i in nums:
        dosomething()
        dfs(nums, depth+1)
```
枚举所有的可能情况，对相应的情况做处理即可。

```
全排列

给定一个 没有重复 数字的序列，返回其所有可能的全排列。

输入: [1,2,3]
输出:
[
  [1,2,3],
  [1,3,2],
  [2,1,3],
  [2,3,1],
  [3,1,2],
  [3,2,1]
]
```

分析

> 全排列就是枚举数字所有的组合情况，我们可以直接套用模板代码。我们先考虑第一个数，第一个数可以是任意数字，第二个数是除掉第一个数，剩余的数字，依次类推。

```go
func permute(nums []int) [][]int {
    results := make([][]int, 0)
    dfs(nums, &results, 0)
    return results
}

func dfs(nums []int, result *[][]int, depth int) {
    if depth == len(nums) {
        buffer := make([]int, len(nums))
        copy(buffer, nums) // go语言的slice是指向同一块底层数组的指针，需要拷贝
        *result = append(*result, buffer)
        return
    }
    /* 
    假设有[1,2,3]
    那么第一个数
    [1], 剩余[2,3]
    [2], 剩余[1,3]
    [3], 剩余[1,2]

    第二个数
    [1, 2], 剩余[3]
    [1, 3], 剩余[2]

    ...

    第三个数
    [1, 2, 3] 剩余[]
    [1, 3, 2] 剩余[]
    ...
    
    for i := 0; i < length; i++ {
        buffer := make([]int, len(nums))
        copy(buffer, nums) 每次都拷贝一份，然后从nums从去除这个数
        dfs(append(buffer[:i], buffer[i+1:]...), append(tmp, nums[i]), results)
    }
    */
    for i := depth; i < len(nums); i ++ {
        nums[depth], nums[i] = nums[i], nums[depth]
        dfs(nums, result, depth+1)
        nums[depth], nums[i] = nums[i], nums[depth]
    }
    /*
    小技巧：交换顺序，可以避免重复的拷贝。思想是[0: depth]为已经使用的元素，[depth:]为未使用的元素。用所有未使用的元素，依次去填充nums[depth]。这轮深度搜索完之后，将其交换回来。

    */

}
```

练习
[全排列](https://leetcode-cn.com/problems/permutations/submissions/)





