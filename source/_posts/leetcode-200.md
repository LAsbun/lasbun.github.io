---
title: 无聊刷刷Leetcode(200. 岛屿数量)
tags:
  - python
  - leetcode
date: 2020-04-20 23:55:31
---


Leetcode200. 岛屿数量  题解

<!-- more -->

[题目链接]()

#### 思路
- 做的时候第一时间就想到了DFS. 从左至右遍历，找到某个1，就使用深搜找到所有相关联的1。并将搜索到的位置的值设置为0(因为是同一个岛屿)

- 看了下讨论区，还可以用BFS广搜, 或者是并查集....

##### 代码
```
class Solution:
    def numIslands(self, grid: List[List[str]]) -> int:

        count = 0

				// 直接输入[]
        max_raw = len(grid)
        if max_raw == 0:
            return count
				// 输入['1','0','1','1']
        if not isinstance(grid[0], List):
            flag = False
            for i in grid:
                if i == '1':
                    if not flag:
                        count += 1
                        flag = True
                else:
                    flag = False

            return count


        max_col = len(grid[0])

        for i in range(max_raw):
            for j in range(max_col):
                if grid[i][j] == '1':
                    count += 1
                    self.dfs(grid, max_col, max_raw, j, i)

        return count

    def dfs(self, grid, max_col, max_row, col_inx, row_inx):

        grid[row_inx][col_inx] = '0'

        up_row = row_inx - 1
        down_row = row_inx + 1

        left_col = col_inx - 1
        right_col = col_inx + 1

        # 上面
        if up_row >= 0 and grid[up_row][col_inx] == '1':
            self.dfs(grid, max_col, max_row, col_inx, up_row)

        # 左
        if left_col >= 0 and grid[row_inx][left_col] == "1":
            self.dfs(grid, max_col, max_row, left_col, row_inx)

        # 右
        if right_col < max_col and grid[row_inx][right_col] == "1":
            self.dfs(grid, max_col, max_row, right_col, row_inx)

        # 下
        if down_row < max_row and grid[down_row][col_inx] == '1':
            self.dfs(grid, max_col, max_row, col_inx, down_row)

```