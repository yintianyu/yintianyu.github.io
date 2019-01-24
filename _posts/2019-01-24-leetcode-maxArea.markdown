---
layout: post
title: "【Leetcode题目练习】 Container With Most Water"
date: 2019-01-24 16:20:04 +0800
author: "尹天宇"
tags:
    - java
    - leetcode
---

## 问题描述
[Container With Most Water](https://leetcode.com/problems/container-with-most-water/)

Given n non-negative integers a<sub>1</sub>, a<sub>2</sub>, ..., a<sub>n</sub> , where each represents a point at coordinate (i, a<sub>i</sub>). n vertical lines are drawn such that the two endpoints of line i is at (i, a<sub>i</sub>) and (i, 0). Find two lines, which together with x-axis forms a container, such that the container contains the most water.

**Note**: You may not slant the container and n is at least 2.

![](/img/question_11.jpg)

The above vertical lines are represented by array [1,8,6,2,5,4,8,3,7]. In this case, the max area of water (blue section) the container can contain is 49.

**Example**:

```
Input: [1,8,6,2,5,4,8,3,7]
Output: 49
```


## 问题分析
这是一个求最值问题，这种问题的关键就是把所有的case覆盖完全。
一开始我只能想到这种穷举所有可能的办法：
```java
class Solution {
    public int maxArea(int[] height) {
        int length = height.length;
        int max = 0;
        for(int i = 0;i < length;i++){
            for(int j = i + 1;j < length;j++){
                int a = height[i];
                int b = height[j];
                int v = Math.min(a, b) * (j - i);
                if(v > max){
                    max = v;
                }
            }
        }
        return max;
    }
}
```

但是很容易看出这种暴力方法时间复杂度为O(n<sup>2</sup>)，放在leetcode里执行时间为234ms，效率非常低。

于是我们有了一个突破的方向就是在确保覆盖完全的情况下，减少不必要的case的检查。

现在的想法是，用一头一尾两个指针去检查容器体积，然后每次把较短的那个指针向中间移动，直到两个指针重合位置，找出最大值。
这样做的理由在于，当一对数值比较完之后，如果较大的指针向中间移动，结果肯定不会比前一个好（因为容器的高度被较小的数值钳位，而容器的宽度也变小了），所以这是不必要的。因此将较小的指针向中间移动即可。

代码如下：
```java
 class Solution {
    public int maxArea(int[] height) {
        int length = height.length;
        int max = 0;
        int i = 0;
        int j = length - 1;
        while(i != j){
            int h = Math.min(height[i], height[j]);
            max = Math.max(max, h * (j - i));
            while(height[i] == h && i < j) i++;
            while(height[j] == h && i < j) j--;
        }
        return max;
    }
}
```

这种方法在leetcode中只有4ms。

