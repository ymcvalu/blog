---
title: dp上手
date: 2020-03-24 21:26:55
tags:
    - leetcode 
    - dp
---
一直没有写过leetcode刷题相关的内容，确实是以前刷题比较少。最近一段时间有开始在慢慢刷题，毕竟作为一名合格的程序员，基本的算法能力还是要掌握的。

今天就来看一下动态规划。

动态规划听上去是挺高大上的，但是实际很容易理解的，首先来看一个简单的例子：
```go
func skipStage(n int) int {
	if n == 0 {
		return 1
	}
	dp1 := 1
	dp2 := 1
	for i := 2; i <= n; i++ {
		dp2, dp1 = dp2+dp1, dp2
	}
	return dp2
}
```
上面是[跳台阶问题](https://leetcode-cn.com/problems/qing-wa-tiao-tai-jie-wen-ti-lcof/)的一个dp求解。如果你细心的话，就会发现实际上跳台阶就是求斐波那契数列。

**动态规划是一个自底向上的过程**。能够使用DP求解的问题，需要具有这几个性质：
- **最优子结构**：大规模问题集的最优解可以由小规模问题集的最优解推导出来
- **无后效性**：未来与过去无关，当前阶段所作的结果不会对未来阶段产生影响。

因为具有最优子结构，我们可以先求解小规模问题集的解，然后再组合成大规模问题集的解，比如上面的跳台阶问题，假设函数f是问题规模n到结果的映射，当n=0的时候，可以看成只有一种方案，因此f(0)=1，当n=1的时候，明显也只有一种方案，因此f(1)=1，而当n=2的时候，也可从0跳2步到2，也可以从1跳1步到2，因此f(2)=f(1)+f(0)，这样递推下去，我们就得到了**状态转移方程**：
```
f(n) = f(n-1) + f(n-2)
```
这个状态转移方程是DP问题求解的核心，只要我们能够找出状态转移方程，程序就可以写出来了。

现在看来，动态规划很简单吧。

接下来，我们来看一下几个dp题目。

### 硬币找零问题
问题描述：
给定不同面额的硬币coins和一个总金额account，求解可以凑成总金额的所需的最少硬币个数，为了一定能够凑足零钱，规定conins中存在最小面额1，可以认为每种coin的数量都是无限的。

我们可以使用dp[i]表示，面额为i的时候，所需要的最少硬币个数，那么状态转换方程就很明显了：
```
dp[i] = min(dp[i-x]+1), foreach x in conins
```
那么，代码就很容易写出来了：
```go
func coin(n int, coins []int) (int, map[int]int) {
	if n == 0 || len(coins) == 0 {
		return -1, nil
	}

	// 零钱有序排列
	sort.Slice(coins, func(i, j int) bool {
		return coins[i] < coins[j]
	})

	// 最小值要为1
	if coins[0] != 1 {
		return -1, nil
	}

	dp := make([]int, n+1)

	// dp求最值
	for i := 1; i <= n; i++ {
		for j := 0; j < len(coins) && coins[j] <= i; j++ {
			if dp[i] == 0 {
				dp[i] = dp[i-coins[j]] + 1
			} else {
				dp[i] = min(dp[i], dp[i-coins[j]]+1)
			}
		}
	}

	solve := map[int]int{}
	c := dp[n]
	m := n
	// 回溯，求找零方案
	for i := n - 1; i >= 0; i-- {
		if dp[i] == c-1 {
			solve[m-i] += 1
			c = dp[i]
			m = i
		}
	}

	return dp[n], solve
}
```
在上面的代码中，我们除了计算出需要的最少硬币数量，还通过对dp数组进行回溯，求解出了一个可能的最优解。

### 编辑距离问题
问题描述：
对于两个单词word1和word2，计算出将word1转换为word2所需使用的最少操作数。操作主要有3种：
- 插入一个单词
- 删除一个单词
- 替换一个单词

这个题也挺容易的，我们可以使用dp[i][j]表示将word1[:i+1]转变为word2[:j+1]的最少操作数，那么状态转换方程就很明显了：
```
if word1[i] != word2[j]
    dp[i][j] = min(min(dp[i-1][j-1]+1, dp[i-1][j]+1), dp[i][j-1]+1)
else 
    dp[i][j] = dp[i-1][j-1]
```
其中：
- dp[i-1][j-1]+1 可以表示替换一个字符
- dp[i-1][j]+1 可以表示从word1中插入一个字符
- dp[i][j-1]+1 可以表示从word1中删除一个字符

对此，我们的代码就很容易写出来了：
```go
func minDistance(s1, s2 string) int {
	ln1 := len(s1)
    ln2 := len(s2)
    
	if ln1 == 0 {
		return ln2
	} else if ln2 == 0 {
		return ln1
	}

	m := make([][]int, ln1+1)
	m[0] = make([]int, ln2+1)
	for i := 0; i < ln1+1; i++ {
		m[i] = make([]int, ln2+1)
	}
	for i := 1; i < ln1+1; i++ {
		m[i][0] = i
	}
	for i := 1; i < ln2+1; i++ {
		m[0][i] = i
	}

	for i := 1; i < ln1+1; i++ {
		for j := 1; j < ln2+1; j++ {
			if s1[i-1] == s2[j-1] {
				m[i][j] = m[i-1][j-1]
			} else {
				m[i][j] = min(min(m[i-1][j], m[i][j-1]), m[i-1][j-1]) + 1
			}
		}
	}

	return m[ln1][ln2]
}

func min(i, j int) int {
	if i < j {
		return i
	}
	return j
}
```

### 按摩师问题
问题描述：
一个有名的按摩师会收到源源不断的预约请求，每个预约都可以选择接或不接。在每次预约服务之间要有休息时间，因此她不能接受相邻的预约。给定一个预约请求序列，替按摩师找到最优的预约集合（总预约时间最长），返回总的分钟数。

这个题目和另外一个打家劫舍其实是同一个问题，我们可以使用dp[i]表示到第i个预约，总的最长的预约时长，那么状态转移方程很明显了：
```
dp[i] = max(dp[i-1], dp[i-2]+num[i])
```
有了状态转移方程，代码就很简单了：
```go
func massage(nums []int) int {
    if len(nums)==0{
        return 0
    }
    dp:=make([]int,len(nums)+1)
    dp[0]=0
    dp[1]=nums[0]
    for i:=1;i<len(nums);i++{
        dp[i+1]=max(dp[i],dp[i-1]+nums[i])
    }
    return dp[len(nums)]
}

func max(i,j int)int{
    if i>j{
        return i 
    }
    return j 
}
```

### 可被三整除的最大和
问题描述：
给你一个整数数组 nums，请你找出并返回能被三整除的元素最大和。

这个问题的状态转移方程就没有那么容易看出来了。我们先分析问题，对于元素的和，mod 3之后，结果为0, 1, 2，如果等于0，说明刚好被3整除。我们可以使用一个大小为3的数组来记录当前，mod 3分别为0, 1, 2的最大的sum，我们直接来看代码实现，因为状态转移方程不像前面的那么好直接写出来：

```go
func maxSumDivThree(nums []int) int {
	if len(nums) == 0 {
		return 0
	}

 	dp := [3]int{}
	dp[nums[0]%3] = nums[0]
 
    // 状态转移
	for i := 1; i < len(nums); i++ {
		_dp := dp
		for j := 0; j < 3; j++ {
			sum := dp[j] + nums[i]
			mod := sum % 3
			_dp[mod] = max(_dp[mod], sum)
		}

		dp = _dp
 
	}

	return dp[0] // mod 3==0
}

func max(i, j int) int {
	if i > j {
		return i
	}
	return j
}
```
### 最长公共子序列
求两个字符串的最长公共子序列。
这个问题就简单的多了，直接dp[i][j]表示str1[:i+1]和str2[:j+1]的最长公共子序列，代码很简单：
```go
func LCS(s1, s2 string) string {
	ln1, ln2 := len(s1), len(s2)
	max := ln1
	if max < ln2 {
		max = ln2
	}

	ln1++
	ln2++

	// 构造二维数组，空间换时间
	m := make([][]int, ln1)
	for i := 0; i < ln1; i++ {
		m[i] = make([]int, ln2)
	}

	for i := 1; i < ln1; i++ {
		for j := 1; j < ln2; j++ {
			if s1[i-1] == s2[j-1] {
				m[i][j] = m[i-1][j-1] + 1
			} else {
				v := m[i-1][j-1]
				if v < m[i][j-1] {
					v = m[i][j-1]
				}
				if v < m[i-1][j] {
					v = m[i-1][j]
				}
				m[i][j] = v
			}
		}
	}

	// 回溯
	i, j := ln1-1, ln2-1
	len := m[i][j]
	ret := make([]byte, len)

	for len > 0 {
		if n := m[i][j]; n > m[i-1][j-1] {
			len--
			ret[len] = s1[i-1]
			i--
			j--
		} else if n > m[i-1][j] {
			len--
			ret[len] = s1[i-1]
			i--
		} else if n > m[i][j-1] {
			len--
			ret[len] = s1[i-1]
			j--
		} else {
			i--
			j--
		}
	}

	return string(ret)
}
```

### 割绳子
一段长度为k的绳子，将其分割成多段（至少两段），然后将每段长乘积，求最大值。

经过前面的学习，相信这题肯定难不倒你，话不多说，我们直接看代码实现：
```go
func cuttingRope(l int) int {
	if l <= 1 {
		return 0
	}

	dp := make([]int, l+1)
	dp[1] = 1

	for i := 2; i <= l; i++ {
		m := 0
		for j := 1; j < i; j++ {
			m = max(m, j*dp[i-j])
        }
        
		if i < l {
			m = max(m, i)
		}
		dp[i] = m
	}

	return dp[l]
}

func max(i,j int)int{
    if i>j{
        return i 
    }
    return j
}
```
### 0-1背包问题
最后，以一个01背包问题来收尾，01背包问题算是dp的入门题目了。假如现在有容量为capacity的背包，weights数组为物品的重量，values为物品的价值，现在要求每个物品最多只能装一次，求背包能够装的最大价值。

可以使用`dp[i][j]`表示只考虑前i个物品的时候，容量j可以装的最大价值，那么可以得到状态转移方程：
```
dp[i][j]=max(dp[i-1][j], dp[i-1][j-weight[i]]+values[i]) 
- dp[i-1][j]：表示不装入当前物品
- dp[i-1][j-weight[i]]+values[i]表示装入当前物品
```
具体的代码实现就是：
```go
func one_zero_bag(capacity int, weights []int, values []int) int {
    wn := len(weights)
    dp := make([][]int, wn+1)
    for i := 0; i < wn+1; i++ {
        dp[i] = make([]int, capacity+1)
    }

    for i := 1; i <= wn; i++ {
        for j := weights[i-1]; j <= capacity; j++ {
            dp[i][j] = max(dp[i-1][j], dp[i-1][j-weights[i-1]]+values[i-1])
        }
    }

    return dp[wn][capacity]
}

func max(i, j int) int {
    if i > j {
        return i
    }
    return j
}
```


可以看到，DP问题的求解，就是找它的状态转移方程，有时候灵感来了，一眼就看出答案了！！！

当然，本文只讲了简单的dp基础，但是对于入门，却是够用了。dp题是很多公司的面试笔试题爱出的，很有必要掌握。