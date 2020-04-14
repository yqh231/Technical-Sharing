# 广度搜索算法

广度搜索算法是面试中最常考的算法之一，很多面试题都可以用广度搜索算法来解决。我们这里不谈枯燥的理论，只谈广度搜索算法在实际面试题中的运用。

## 模板代码

```python
from queue import deque

q = queue()

q.append(ele)

while len(q) > 0:
    ele = q.popleft()

    doSomethine(ele)

    for el in findAround(ele):
        q.append(ele)
```

这段伪码是最核心的广度搜索代码，所有的广度搜索都是这段代码推演而来。广度搜索依赖于队列，需要对自己常用语言的队列实现熟练掌握。我们先将第一个元素推入队列，然后将其拿出，遍历这个元素周围的元素，将其推入队列。反复进行这个过程，直到队列为空。

## 实际运用

### 1. 二叉树的广度遍历

```
从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。

例如:
给定二叉树: [3,9,20,null,null,15,7],

    3
   / \
  9  20
    /  \
   15   7

返回：
[3,9,20,15,7]
```

分析
> 这是广度遍历最基本的运用没什么好说的，直接套模板代码

实现

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func levelOrder(root *TreeNode) []int {
    results := make([]int, 0)

    if root == nil {
        return results
    }
    queue := make([]*TreeNode, 0)

    queue = append(queue, root)

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1: ]
        if node.Left != nil {
            queue = append(queue, node.Left)
        }

        if node.Right != nil {
            queue = append(queue, node.Right)
        }
        results = append(results, node.Val)
    }
    return results
}
```

练习地址

[从上到下打印出二叉树](https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-lcof/)


### 2. 多源广度搜索

有的题目，不会这么直白地告诉我们，应该使用bfs。需要我们自己去提炼出关键信息。

```
腐烂的橘子

在给定的网格中，每个单元格可以有以下三个值之一：

值 0 代表空单元格；
值 1 代表新鲜橘子；
值 2 代表腐烂的橘子。
每分钟，任何与腐烂的橘子（在 4 个正方向上）相邻的新鲜橘子都会腐烂。

返回直到单元格中没有新鲜橘子为止所必须经过的最小分钟数。如果不可能，返回 -1。

输入：[[2,1,1],[1,1,0],[0,1,1]]
输出：4

输入：[[2,1,1],[0,1,1],[1,0,1]]
输出：-1
解释：左下角的橘子（第 2 行， 第 0 列）永远不会腐烂，因为腐烂只会发生在 4 个正向上。

输入：[[0,2]]
输出：0
解释：因为 0 分钟时已经没有新鲜橘子了，所以答案就是 0 。
```
![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/02/16/oranges.png)

分析

> 烂橘子感染好橘子，然后由中心向外围扩散，这个思想就是典型的bfs。唯一和上面不同的，初始情况可能有多个烂橘子，我们只需要在初始化得时候，把所有烂橘子放入队列即可。

实现

```python
from queue import Queue

class Solution:
    def orangesRotting(self, grid: List[List[int]]) -> int:
        q = Queue()

        count = 0

        for i in range(len(grid)):
            for j in range(len(grid[0])):
                if grid[i][j] == 2:
                    q.put((i, j))

        rotote = [(1, 0), (-1, 0), (0, 1), (0, -1)]
        while True:
            new = Queue()
            flag = False

            while not q.empty():
                i, j = q.get()
                for r in rotote:
                    if self.legal_index(i+r[0], j+r[1], grid):
                        if grid[i+r[0]][j+r[1]] == 1:
                            flag = True
                            grid[i+r[0]][j+r[1]] = 2
                            new.put((i+r[0], j+r[1]))
            if flag:
                count += 1
            q = new
            if new.empty():
                break 
        
        for i in range(len(grid)):
            for j in range(len(grid[0])):
                if grid[i][j] == 1:
                    return -1

        return count

    
    def legal_index(self, i, j, grid):
        if i < 0 or i >= len(grid):
            return False

        if j < 0 or j >= len(grid[0]):
            return False

        return True
```
练习

[烂橘子](https://leetcode-cn.com/problems/rotting-oranges/)


### 3. 权重广度搜索

```
通知所有员工所需时间


公司里有 n 名员工，每个员工的 ID 都是独一无二的，编号从 0 到 n - 1。公司的总负责人通过 headID 进行标识。

在 manager 数组中，每个员工都有一个直属负责人，其中 manager[i] 是第 i 名员工的直属负责人。对于总负责人，manager[headID] = -1。题目保证从属关系可以用树结构显示。

公司总负责人想要向公司所有员工通告一条紧急消息。他将会首先通知他的直属下属们，然后由这些下属通知他们的下属，直到所有的员工都得知这条紧急消息。

第 i 名员工需要 informTime[i] 分钟来通知它的所有直属下属（也就是说在 informTime[i] 分钟后，他的所有直属下属都可以开始传播这一消息）。

返回通知所有员工这一紧急消息所需要的分钟数。

输入：n = 7, headID = 6, manager = [1,2,3,4,5,6,-1], informTime = [0,6,5,4,3,2,1]
输出：21
解释：总负责人 id = 6。他将在 1 分钟内通知 id = 5 的员工。
id = 5 的员工将在 2 分钟内通知 id = 4 的员工。
id = 4 的员工将在 3 分钟内通知 id = 3 的员工。
id = 3 的员工将在 4 分钟内通知 id = 2 的员工。
id = 2 的员工将在 5 分钟内通知 id = 1 的员工。
id = 1 的员工将在 6 分钟内通知 id = 0 的员工。
所需时间 = 1 + 2 + 3 + 4 + 5 + 6 = 21 。
```
[通知所有员工所需时间](https://leetcode-cn.com/problems/time-needed-to-inform-all-employees/)

分析

> 这个题看似很唬人，实际仔细分析后，整个员工结构图就是一个树，所求就是从跟到叶子节点的路径。不同的节点之间，路径的权重不同，找出整个权重最大值即可。

代码
```python
from queue import Queue
from collections import defaultdict
class Solution:
    def numOfMinutes(self, n: int, headID: int, manager: List[int], informTime: List[int]) -> int:

        q = Queue()
        tmp = defaultdict(list)

        for i in range(len(manager)):
            if manager[i] == -1:
                continue
            tmp[manager[i]].append(i)

        q.put((headID, 0))

        result = 0

        while not q.empty():
            this_id, val = q.get()

            for id_ in tmp[this_id]:
                q.put((id_, val+informTime[this_id]))
                result = max(result, val+informTime[this_id])

        return result
```

练习

[通知员工所需时间](https://leetcode-cn.com/problems/time-needed-to-inform-all-employees/)

## 小结

不同的题，问题描述完全不一样。但是有一个共同点，从中心向外扩散，或者从跟节点向叶节点扩散。这种一圈圈扩散的题目，都可以考虑用bfs来解决。