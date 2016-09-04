title: 'NLP PA: Hidden Markov Models'
date: 2013-03-24 13:27:49
categories: NLP
tags: [nlp, coursera, hmm]
---

[传送门](https://class.coursera.org/nlangp-001/class/index "NLP") 

HMM 是通过预测一连串的 $y_i$ 来得到模型 $p(x_1 \dots x_n, y_1 \dots y_n)$ 的概率最大值。本周的 Programming 主要是利用 HMM 来做针对句子的词性 tagger。

<!--more-->

{% math %}
p(x_1 \dots x_n, y_1 \dots y_{n+1}) = 
\prod_{i=1}^{n+1} q(y_i|y_{i-2},y_{i-1}) \times \prod_{i=1}^n e(x_i|y_i)
{% endmath %}

为方便求解，一般把待标注的句子变成\*,\*,sentence,STOP，即 $ y_0=y_{-1}=*, y_n=STOP $。在标注前，先对数据进行处理：把所有在训练集中出现且次数 <5 的单词替换成 `RARE`。

### Part 1
作为 baseline，一开始忽略句子内部的联系，用MLE来估计每个 $x_i$ 所属的 tag。也就是求$${\arg\max_y} e(x|y)$$

其中

$$
e(x|y) = \frac{Count(x, y)}{Count(y)}
$$

$Count(x,y)$ 表示 $x$ 和 $y$ 一起出现的概率。如果对于未出现在训练集的词，$p(x\vert y_i)=0$，而无法取 max 值。所以这时把 $x$ 当成是 RARE，对比 $p(RARE\vert y_i)$ 的值。

### Part 2
这部分实现的是Viterbi算法，经典的 $O(n|S|^3)$ 的 DP。DP(i,u,v) 表示以 i 为结尾，前两个连续的单词为 u,v 的最大值。转移方程为

{% math %}
DP(i,y_i,y_{i-1}) = max\{ DP(i-1, y_{i-1}, y_{i-2}) \times p(y_i|y_{i-2},y_{i-1}) \times e(x_i|y_i) \}
{% endmath %}


初始化$DP(0,\*,\*) = 1$。目标状态为$DP(n,STOP,u)$。

求解过程中，对于未出现在训练集中的词，同样当成 RARE 处理。而对于出现在训练集中，而 $Count(x,y)=0$ 的词，其 $e(x\vert y)$ 值为 0。另外，在 DP 过程中，$y_{-1,0,n+1}$ 的 tag 都为已知，所以也要加特判。

### Part 3
为了增加对 `RARE` 的判别性，这部分把 RARE 再进行了细化，比如全为大写的为一类，出现数字的为一类。

对于 `gene.test`，三种方法的 F1-Score 如下：


| #Part 1 | #Part 2 | #Part 3 |
|:----:|:----:|
|  0.26308   |  0.36514   | 0.39519

除了 #Part 2 跟 Goal F1-Score 差 了0.00030，其他都一致。
