- 这个得多练习。我的经验是自己实现常见数据结构(练控制细节的能力，不抄代码，自己根据原理写，少调试)，然后是分门别类刷题(练熟练度)。
- 经典结构，例如实现双向链表，操作包括插入，查询，删除。然后封装一下实现栈，队列。还有优先级队列(堆的实现)，各种排序算法实现(重点有快排，归并，堆排，顺便熟悉递归)。二叉树树的遍历方式(前中后，考虑递归非递归，熟悉栈的模拟)。全排列(递归非递归，掌握回朔)。哈希表实现。
- 刷题除了一些专门的算法技巧例如DP，很多都是和基础数据结构的熟练度有很大关系。可以分类别(可以参考网上分类)，做题要争取一次bugfree，宁跳过不要看答案(除了学算法课程这种，做题不要这样)，一个月内密集刷个一两百题，就应该有不错的熟练度。

## leetcode 数据结构经典题
   - [https://youtu.be/3toa_cJu49g](https://youtu.be/3toa_cJu49g)
   - [https://youtu.be/8ULrqL0P0kA](https://youtu.be/8ULrqL0P0kA)


1. Recursion / Backtracking
    - [39. Combination Sum](https://leetcode.cn/problems/combination-sum/)
    - [40. Combination Sum II](https://leetcode.cn/problems/combination-sum-ii/)
    - [78. Subsets ](https://leetcode.cn/problems/subsets/)
    - [90. Subsets II ](https://leetcode.cn/problems/subsets-ii)
    - [46. Permutations ](https://leetcode.cn/problems/permutations)
    - [47. Permutations II](https://leetcode.cn/problems/permutations-ii) 

2. Graph Traversal - DFS, BFS, Topological Sorting
    - [133. Clone Graph](https://leetcode.cn/problems/clone-graph) (BFS / DFS)
    - [127. Word Ladder](https://leetcode.cn/problems/word-ladder) (BFS)
    - `490. The Maze `
    - [210. Course Schedule II](https://leetcode.cn/problems/course-schedule-ii) (Topological Sorting)
    - `269. Alien Dictionary` (Topological Sorting)

3. Binary Tree / Binary Search Tree (BST)
    - [94. Binary Tree Inorder Traversal ](https://leetcode.cn/problems/binary-tree-inorder-traversal)
    - [236. Lowest Common Ancestor of a Binary Tree](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree)
    - [297. Serialize and Deserialize Binary Tree ](https://leetcode.cn/problems/serialize-and-deserialize-binary-tree)
    - [98. Validate Binary Search Tree ](https://leetcode.cn/problems/validate-binary-search-tree)
    - [102. Binary Tree Level Order Traversal ](https://leetcode.cn/problems/binary-tree-level-order-traversal)
 
4. Binary Search
    - [34. Find First and Last Position of Element in Sorted Array](https://leetcode.cn/problems/find-first-and-last-position-of-element-in-sorted-array)
    - [162. Find Peak Element](https://leetcode.cn/problems/find-peak-element) 
    - [69. Sqrt(x)](https://leetcode.cn/problems/sqrtx)
  
5. Data Structure
    - [242. Valid Anagram ](https://leetcode.cn/problems/valid-anagram) (Hash Table)
    - [133. Clone Graph](https://leetcode.cn/problems/clone-graph)  (Hash Table)
    - [127. Word Ladder](https://leetcode.cn/problems/word-ladder) (Hash Table)
    - [155. Min Stack](https://leetcode.cn/problems/min-stack)  (Stack)
    - [225. Implement Stack using Queues](https://leetcode.cn/problems/implement-stack-using-queues) (Stack / Queue)
    - [215. Kth Largest Element in an Array](https://leetcode.cn/problems/kth-largest-element-in-an-array) (PriorityQueue)
    - [23. Merge k Sorted Lists](https://leetcode.cn/problems/merge-k-sorted-lists) (PriorityQueue)

6. Linked List Manipulation
    - [237. Delete Node in a Linked List](https://leetcode.cn/problems/delete-node-in-a-linked-list)
    - [92. Reverse Linked List II ](https://leetcode.cn/problems/reverse-linked-list-ii)
    - [876. Middle of the Linked List](https://leetcode.cn/problems/middle-of-the-linked-list) 
    - [143. Reorder List](https://leetcode.cn/problems/reorder-list)

7. Pointer Manipulation
    - [239. Sliding Window Maximum](https://leetcode.cn/problems/sliding-window-maximum) 
    - [3. Longest Substring Without](https://leetcode.cn/problems/longest-substring-without-repeating-characters) 
    - [76. Minimum Window Substring](https://leetcode.cn/problems/minimum-window-substring) 

8. Sorting
    - Time -- O(N log N)
    - Merge Sort -- Space O(N)
    - Quick Sort
    - [148. Sort List](https://leetcode.cn/problems/sort-list)

9. Convert Real Life Problem to Code
   - [146. LRU Cache](https://leetcode.cn/problems/lru-cache)
   - `1066. Compus Bike`
   - `490. The Maze`

10. Time Space Complexity
   - 一般面试的时候 你说完算法 就要说 这个算法的 time / space complexity是什么
   - 每次你做完一道题 给自己养成一个习惯 就是想一下他的时间空间复杂度是多少

## 二叉树专题

[参考《代码随想录》](https://www.programmercarl.com/%E4%BA%8C%E5%8F%89%E6%A0%91%E7%90%86%E8%AE%BA%E5%9F%BA%E7%A1%80.html)

### 二叉树遍历

> 深度优先遍历中（递归｜迭代）：
> 中在什么位置即为 什么排序；左永远在右前;
- 前序：中左右
- 中序：左中右
- 后续：左右中
> 广度优先遍历：层次遍历（迭代）
#### 递归为什么还能退回父节点？
https://www.zhihu.com/question/58529163


### 递归三要素
1. 确定递归函数参数和返回值；
2. 确定终止条件；
3. 确认单层递归的逻辑；

