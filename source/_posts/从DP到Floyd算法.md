---
title: 从DP到Floyd算法
date: 2024-04-26 15:28:59
tags:
- 算法
- 图论
- 最短路
- Floyd
- DP
categories:
- 想成为算法高手
---

> **Floyd** 算法用于求图中任意两点之间的最短路，它的本质其实就是图的一种动态规划算法。下面以一道题为例，演示从动态规划到记忆化搜索，最后推出 Floyd 算法。

# 题目

[LeetCode 1334](https://leetcode.cn/problems/find-the-city-with-the-smallest-number-of-neighbors-at-a-threshold-distance/description/)

题目的核心就是要求出图中任意两点之间的距离。



# 寻找子问题

![DP分析过程](./从DP到Floyd算法/DP分析过程.png)

# 递归怎么写：状态定义与状态转移方程

定义 *dfs(k, i, j)* 表示从 *i* 到 *j* 的最短路长度，并且这条最短路的中间节点编号都 *≤k*。注意中间节点不包含 *i* 和 *j*。则有如下两种情况：

- 不选 *k*，那么中间节点的编号都 *≤k−1*，即 *dfs(k, i, j) = dfs(k−1, i, j)*。
- 选 *k*，问题分解成从 *i* 到 *k* 的最短路，以及从 *k* 到 *j* 的最短路。由于这两条最短路的中间节点都不包含 *k*，所以中间节点的编号都 *≤k−1*，故得到 *dfs(k, i, j) = dfs(k−1, i, k) + dfs(k−1, k, j)*。


这两种情况取最小值，就得到了最终的状态转移方程：
$$
dfs(k, i, j) = Math.min(dfs(k - 1, i, j), dfs(k, i, k) + dfs(k, k, j))
$$
递归边界：*dfs(−1, i, j) = w\[i][j]*。*k = −1* 表示 *i* 和 *j* 之间没有任何中间节点，此时最短路长度只能是连接 *i* 和 *j* 的边的边权，即 *w\[i][j]*。如果没有连接 *i* 和 *j* 的边，则 *w\[i][j] = ∞*。

递归代码如下：

```java
public class Solution {
    public int findTheCity(int n, int[][] edges, int distanceThreshold) {
        int[][] w = new int[n][n];
        for (int[] row : w) {
            Arrays.fill(row, Integer.MAX_VALUE / 2); // 防止加法溢出
        }
        for (int[] e : edges) {
            int x = e[0], y = e[1], wt = e[2];
            w[x][y] = w[y][x] = wt;
        }

        int ans = 0;
        int minCnt = n;
        for (int i = 0; i < n; i++) {
            int cnt = 0;
            for (int j = 0; j < n; j++) {
                if (j != i && dfs(n - 1, i, j, w) <= distanceThreshold) {
                    cnt++;
                }
            }
            if (cnt <= minCnt) { // 相等时取最大的 i
                minCnt = cnt;
                ans = i;
            }
        }
        return ans;
    }

    private int dfs(int k, int i, int j, int[][] w) {
        if (k < 0) { // 递归边界
            return w[i][j];
        }
        return Math.min(dfs(k - 1, i, j, w), dfs(k - 1, i, k, w) + dfs(k - 1, k, j, w));
    }
}
```

**复杂度**：

- 时间复杂度：O(n^2^3^n^)
- 空间复杂度：O(n)



# 记忆化搜索

递归过程中含有大量的重复调用（入参相同），由于递归过程中没有“副作用”，同样的入参无论计算多少次，算出来的结果都是一样的，因此可以用**记忆化搜索**来优化：如果一个状态（递归入参）是第一次遇到，那么可以在返回前，把状态及其结果记录起来，下次再计算到这个状态时，直接返回保存的之前计算的结果。

记忆化搜索的代码如下：

```java
public class Solution {
    public int findTheCity(int n, int[][] edges, int distanceThreshold) {
        int[][] w = new int[n][n];
        for (int[] row : w) {
            Arrays.fill(row, Integer.MAX_VALUE / 2); // 防止加法溢出
        }
        for (int[] e : edges) {
            int x = e[0], y = e[1], wt = e[2];
            w[x][y] = w[y][x] = wt;
        }
        int[][][] memo = new int[n][n][n];

        int ans = 0;
        int minCnt = n;
        for (int i = 0; i < n; i++) {
            int cnt = 0;
            for (int j = 0; j < n; j++) {
                if (j != i && dfs(n - 1, i, j, memo, w) <= distanceThreshold) {
                    cnt++;
                }
            }
            if (cnt <= minCnt) { // 相等时取最大的 i
                minCnt = cnt;
                ans = i;
            }
        }
        return ans;
    }

    private int dfs(int k, int i, int j, int[][][] memo, int[][] w) {
        if (k < 0) { // 递归边界
            return w[i][j];
        }
        if (memo[k][i][j] != 0) { // 之前计算过
            return memo[k][i][j];
        }
        return memo[k][i][j] = Math.min(dfs(k - 1, i, j, memo, w),
                dfs(k - 1, i, k, memo, w) + dfs(k - 1, k, j, memo, w));
    }
}
```

> [!WARNING]
>
> *memo* 数组的初始值一定不能等于要记忆化的值。

**复杂度**：

- 时间复杂度：O(n^3^)
- 空间复杂度：O(n^3^)



# Floyd

将递归 1:1 翻译成递推，其实最后得到的就是 Floyd 算法。

- *dfs* 改成 *f*数组；
- 递归改成循环（每个参数都对应一层循环）；
- 递归边界改成 *f* 数组的初始值。

Floyd 代码如下：

```java
class Solution {
    public int findTheCity(int n, int[][] edges, int distanceThreshold) {
        int m = edges.length;
        int INF = Integer.MAX_VALUE / 2;  // 防止加法溢出
        int[][] g = new int[n][n];
        for (int[] row : g) {
            Arrays.fill(row, INF);
        }
        for (int[] edge : edges) {
            g[edge[0]][edge[1]] = edge[2];
            g[edge[1]][edge[0]] = edge[2];
        }

        int[][][] f = new int[n + 1][n][n];
        f[0] = g;  // 边界
        // Floyd核心部分
        for (int k = 0; k < n; k++) {
            for (int i = 0; i < n; i++) {
                for (int j = 0; j < n; j++) {
                    f[k + 1][i][j] = Math.min(f[k][i][j], f[k][i][k] + f[k][j][k]);
                }
            }
        }

        // 结果统计
        int ans = 0;
        int minCnt = n;
        for (int i = 0; i < n; i++) {
            int cnt = 0;
            for (int j = 0; j < n; j++) {
                if (i != j && f[n][i][j] <= distanceThreshold) cnt++;
            }
            if (cnt <= minCnt) {
                minCnt = cnt;
                ans = i;
            }
        }
        return ans;
    }
}
```

**复杂度**：

- 时间复杂度：O(n^3^)
- 空间复杂度：O(n^3^)

还可以采用**滚动数组**对空间复杂度进行进一步优化。
