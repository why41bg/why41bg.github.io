---
title: DP子序列问题
date: 2024-06-12 00:02:56
mathjax: true
tags:
- DP
- 算法
- 子序列
categories:
- 想成为算法高手
---

>  学习以子序列为背景的动态规划问题时，最经典的一个例子是求**最长上升子序列**，即 LIS 问题。

# LIS问题

[LeetCode 300](https://leetcode.cn/problems/longest-increasing-subsequence/description/)

从选哪个的角度来思考，我们可以遍历数组元素，考虑如果选择当前遍历到的元素，那么状态该如何转移？因此，我们可以定义数组`dp[i]` 表示选第 `i` 个元素所能构成的最长递增子序列长度。`dp[i]` 可以从 `i` 之前的任何一个第 `k` 个元素转移过来，即 `dp[k]`，前提是第 `k` 个元素严格小于第 `i` 个元素，`dp[i]` 取这里面全部结果的最大值即可。如果 `dp[i]` 不能从之前任何一个 `k` 转移过来，那么第 `i` 个元素就作为子序列的第一个元素，即 `dp[i] = 1`。

因此，状态转移方程定义为：`dp[i] = dp[k], k < i && nums[k] < nums[i]`。我们可以将 dp 数组全部初始化为 1，然后再进行状态转移。

``` java
class Solution {
    public int lengthOfLIS(int[] nums) {
        // 定义一个ans作为答案，在转移过程中记录
        int n = nums.length, ans = 1;
        int[] dp = new int[n]; 
        Arrays.fill(dp, 1);
        for (int i = 1; i < n; i++) {
            for (int j = i - 1; j >= 0; j--) {
                if (nums[i] > nums[j]) {
                    dp[i] = Math.max(dp[i], dp[j] + 1);
                }
            }
            ans = Math.max(ans, dp[i]);
        }
        return ans;
    }
}
```

由于最近学了 Go，以后都会贴一个 Go 的写法。

``` go
func lengthOfLIS(nums []int) int {
    n, ans := len(nums), 1;
    dp := make([]int, n)
    dp[0] = 1
    for i := 1; i < n; i++ {
        dp[i] = 1
        for j := 0; j < i; j++ {
            if nums[i] > nums[j] && dp[j] + 1 > dp[i]{
                dp[i] = dp[j] + 1
            }
        }
        if dp[i] > ans {
            ans = dp[i]
        }
    }

    return ans
}
```



# 132场双周赛T3

[LeetCode 3176](https://leetcode.cn/problems/find-the-maximum-length-of-a-good-subsequence-i/description/)

这道题就是一道与子序列有关的动态规划问题，这道题的思路和 LIS 大致类似，也可以从选哪个的角度出发。但是这里题目对子序列有额外的要求，因此我们可以定义 `dp[i][j]` 表示选择第 `i` 个元素，子序列中不相等元素对的数量至多为 `j`。同样的，`dp[i][j]` 也可以由 `i` 之前的任意一个 `p` 转移过来。状态转移方程具体如下：
$$
dp[i][j] = dp[p][j], nums[i] == nums[p]
$$

$$
dp[i][j] = dp[p][j - 1]，nums[i] != nums[p]
$$

如果 `dp[i][j]` 不能由之前任何一个 `p` 转移过来，则 `dp[i][j] = 1`。这里还有个小细节，因为状态转移方程中出现了 `j - 1`，而 `j` 是有可能为 0 的，那么就会出现数组越界，因此可以将 `j` 统一加一偏移一下，`j = 0` 就作为边界。`dp[i][0] = 0`。

Java 写法如下：

``` java
class Solution {
    public int maximumLength(int[] nums, int k) {
        int n = nums.length, ans = 1;
        int[][] dp = new int[n][k + 2];  // [0, k]   [1, k+1]
        for (int i = 0; i <= k; i++) {
            dp[0][i + 1] = 1;
        }

        for (int i = 1; i < n; i++) {
            for (int j = 0; j <= k; j++) {
                dp[i][j + 1] = 1;
                for (int p = 0; p < i; p++) {
                    if (nums[i] == nums[p]) {
                        dp[i][j + 1] = Math.max(dp[i][j + 1], dp[p][j + 1] + 1);
                    } else {
                        dp[i][j + 1] = Math.max(dp[i][j + 1], dp[p][j] + 1);
                    }
                }
                ans = Math.max(ans, dp[i][j + 1]);
            }     
        }
        return ans;
    }
}
```

Go 写法如下：

```go
func maximumLength(nums []int, k int) int {
    n, ans := len(nums), 1
    dp := make([][]int, n)
    for i := 0; i < n; i++ {
        dp[i] = make([]int, k + 2)
    }
    for i := 0; i <= k; i++ {
        dp[0][i + 1] = 1
    }

    for i := 0; i < n; i++ {
        for j := 0; j <= k; j++ {
            dp[i][j + 1] = 1
            for p := 0; p < i; p++ {
                if nums[i] == nums[p] {
                    dp[i][j + 1] = max(dp[i][j + 1], dp[p][j + 1] + 1)
                } else {
                    dp[i][j + 1] = max(dp[i][j + 1], dp[p][j] + 1)
                }
            }
            ans = max(ans, dp[i][j + 1])
        }
    }
    return ans
}
```

