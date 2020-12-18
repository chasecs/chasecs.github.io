---
title: 一个错误洗牌算法的分析
date: 2020-01-03 00:28:45
categories: ["Algorithm"]
math: true
---


将一个长度为 n 的有序数组，重新打乱，随机排序，它算法实现被称作洗牌算法，得名于它的典型应用，扑克牌洗牌。
<!--more-->


这个问题有一个 O(n) 复杂度的算法，[Fisher–Yates shuffle]([https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle](https://en.wikipedia.org/wiki/Fisher–Yates_shuffle))，也称 knuth shuffle，简洁优雅，用 go  实现如下：

```go
func shuffle(arr []int) {
  for i:=len(arr)-1, i >=0, i-- {
    r := rand.Int(i+1)
    arr[i], arr[r] = arr[r], arr[i]
  }
}
```

这个算法的关键点在于：

- r 的范围限于未经过洗牌的元素，已经洗过的不再进入随机选择范围

也就是说，当洗第 N 个元素时，r范围是 0～N；当洗第 N-1 个时，r的范围时 0～N-1

### 一个错误的 Navie shuffle 

第一次看到 knuth shuffle 的实现思路时，我的第一反应：把 `len(arr)` 作为 `r`  的选择范围：

```go
// bug >>>  r := rand.Int(len(arr))
func naiveShuffle(arr []int) {
  for i:=len(arr)-1, i >=0, i-- {
    r := rand.Int(len(arr))
    arr[i], arr[r] = arr[r], arr[i]
  }
}
```

这个实现虽然能输出一些看似随机的结果，但却是一个**错误的 navie shuffle**。

以数组 `[1,2,3]`  为例，分别用 knuth shffle 和 naive shuffle 重复 10 万次，你会发现使用 knuth shuffle 的情况下，每一种排列结果的概率基本相同，而使用 naive shuffle ，有一些排列结果出现的概率会明显超过其它的。

从[The Danger of Naïveté](https://blog.codinghorror.com/the-danger-of-naivete/)摘一张图可以看出明显的区别：

![naive shuffle on a 3 card deck, 600k iterations](https://blog.codinghorror.com/content/images/uploads/2007/12/6a0120a85dcdae970b0120a86da1f3970b-pi.png)

以足够多的次数执行后，各种结果出现次数的统计情况，[The Danger of Naïveté](https://blog.codinghorror.com/the-danger-of-naivete/)  和 [Shuffling](http://datagenetics.com/blog/november42014/index.html) 这两篇文章做了详细介绍。



### 为什么 Knuth Shuffle 是公平的



一个公平的洗牌算法里，对于有 k 个元素的数组，每一个元素出现在某一个位置的概率都应该是 $1/k$。knuth shuffle 算法得到概率正是这个结果。

以数组 `[1,2,3,4,5]` 为例，按照 knuth shuffle 算法：

第一次交换时， 5 仍排在第五位的概率为 ：

$$1/5$$

第二次交换时，4 仍排在第四位的概率为，第一次交换随机数r没选到 4 且第二次交换的随机数r 正好是4:

$$4/5 * 1/4 = 1/5$$

......

以此类推，每个元素经过洗牌后，仍在原来位置的概率为 1 / k，在其它位置的概率用类似的方法分析，最终也是 1/k



### Naive Shuffle 的概率是怎样

而 naive shuffle ，每个元素经过洗牌后，出现在原来位置的概率是多少？

[stackexchange](https://stats.stackexchange.com/a/3087) 上有一个回答，虽然并不是完全准确的分析。对理解这个问题很有启发，

回答里**假设，每次交换的都是两个不同的元素**。现在有 $k$ 个元素，$ k \geq 2$ ， 一个元素经过 $n+1$ 次交换后仍在原来位置，有两种可能情况：

第一种是，经过 n 次交换后，仍在最初的位置 ，且把这个概率记为 $Pn$ 。这样种情况下，当第 n+1次交换后，元素还在最初的位置的概率为 ${k-1 \choose 2}/{k \choose 2} $ 

第二种是经过 n 次交换后，元素不再当前的位置，这个概率是 $ 1- P_n$ 。第 n+1 次交换时，元素又被换回来最初的位置，这个概率为  $1/{k \choose 2}$。

因此：

$$
P_{n+1} = P_n \displaystyle \frac{k-2}{k} + (1-P_n)\displaystyle \frac{2}{k(k-1)}
$$

这个等式经过转换可以得出：

$$
P_n= \displaystyle \frac{1}{k} + (\displaystyle \frac{k-3}{k-1})^n\displaystyle \frac{k-1}{k}
$$


因为$(\displaystyle \frac{k-3}{k-1})^n\displaystyle \frac{k-1}{k}$ 不等于 0， 所以 $P_n \neq \displaystyle \frac{1}{k} $ ，即这个算法不是不是真正的随机。



假如每次交换都是两个不同的元素，那 naive shuffle的概率问题就完美地解决了。但是，这个假设不能覆盖所有的情况：因为根据算法，**每次元素交换时，有可能是元素和自身交换**，因此不完全是组合问题。

### 尝试更完整地分析这个问题

根据上面，我个人分析，第一种情况最后一次交换的概率应该是，其它元素互相交换的概率$\frac{(k-1)^2}{k^2}$ ，加上元素自己和自己交换的概率 $1/k^2$；第二种情况的则是$\frac{2}{k^2}$ 

$$
P_{n+1} = P_n \displaystyle \frac{(k-1)^2+1}{k^2} + (1-P_n)\displaystyle \frac{2}{k^2}
$$

经过转换后可以得到：

$$
P_n= \displaystyle \frac{1}{k-1} + (\displaystyle \frac{（k-1）^2-1}{k^2})^n\displaystyle \frac{k-2}{k-1}
$$
这个结果也是不等于 $1/k$ ，可以设 k=3， n=1 ，得到 $P_1 = 2/3 \ne 1/3$，故 naive shuffle 在随机概率上不够公平。

当然，这个解法是否可靠，以后有时间还要继续审查。

[stackexchange](https://stats.stackexchange.com/a/3087) 提到另一个思路：

*This problem can be analyzed fully as a Markov chain on the Cayley graph of the permutation group.*

有机会也要了解一下。
