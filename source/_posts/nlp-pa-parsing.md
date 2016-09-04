title: 'NLP PA: Parsing'
date: 2013-04-05 13:57:15
categories: NLP
tags: [nlp, coursera, prasing]
---
本周的 PA 是对给定的句子求出相应的句法树（Parsing tree）。实际上，Parsing 经常也做为一些实际应用的预处理，比如机器翻译（Machine Translation, MT），因为不同语言的句子结构（如主谓宾）有很大的区别，通过句法分析得到句子的结构及单词词性，再根据不同语言本身的句法特点进行变换。

<!--more-->

在 NLP 中，词类标注（Part-of-speech Tagging, POS）也跟 Parsing 有关系。POS 可以作为 Parsing 的预处理，因为在我们通过 tagging 确定了每个词语的词性时，后续的 Parsing 规则就可以得到简化。不过由于 tagging 也会出现错误，所以加了效果不一定更好。

[PCFG](http://www.cs.columbia.edu/~mcollins/courses/nlp2011/notes/pcfgs.pdf "PCFG")是一种基于统计、规则的求解 Parsing 的方法，当然该方法已经很早了，只能作为 baseline，不过现在很多的方法和模型还是以此为基础。设有元组

{% math %}
G = \left( N, \Sigma, S, R, q\right)
{% endmath %}

其中，$N$ 为非叶结点（non-terminals），$\Sigma$ 为 Parsing tree 的叶结点（vocabulary），$R$ 为规则（rule），$q$ 为规则的参数（parameter）。在本题中，$R$ 是满足 Chomsky Normal Form（[CNF][1]）的，即只有以下两种形式：

[1]: http://en.wikipedia.org/wiki/Chomsky_normal_form "CNF"

{% math %}
\begin{eqnarray*}
&& X \to Y_1Y_2, 其中X \in N, Y_1 \in N, Y_2 \in N \\
&& X \to Y, 其中X \in N, Y \in \Sigma
\end{eqnarray*}
{% endmath %}

### Part 1 (20 points)

该步骤跟上次的 HMM tagger 预处理一样，把训练集中的词汇分成两种：出现频率 <5 的为 `RARE`，否则为 `non-RARE`，该步骤也可以理解成是一个平滑操作。

<a href="https://gist.github.com/ctLiu3/5317639#file-gistfile1-py" title="A2P2" target="_blank">[code]</a>

### Part 2 (40 points)

得到 Parsing tree 的过程用的是[CKY算法][2]，其实就是个 bottom-up 的 3 维DP。DP(i,j,X) 表示待分析句子的第 i 个单词到第 j 个单词，规则的 left-hand side 为 X 的概率最大值。目标状态为 DP(1,n,"SBARQ")，"SBARQ" 为题目指定的值，当然自己去遍历所有的 symbol 也是可以的。状态转移方程为：

[2]: http://en.wikipedia.org/wiki/CYK_algorithm "CKY Algorithm"

{% math %}
DP(i,j,X) = max\{P(X \to Y \ Z) \times DP(i,s,Y) \times DP(s+1,j,Z) \}
{% endmath %}

几个要注意的坑：

1 初始化。初始的部分是 Parsing tree 的叶结点。在 Part 1 中我们已经将训练集分成两类：RARE 和 non-RARE。该初始化必须遵守以下步骤：

* 如果 P(X->Xi) 在 `parse_train.counts.out` 出现过，即说明 Xi 不是 `RARE`，那么直接赋值 P(X->Xi)；
* 否则如果 Xi 没有出现在 `parse_train.counts.out` 并且 P(X->`RARE`) 存在，那么赋值 P(X->`RARE`)；
* 否则，赋值为 0。

因为概率相乘会导致过小的值，可以用 log 转化，当然本题不转也可以。

2 结果输出。按题目要求，求解得到的 Parsing tree 是用 `json` 来解析的，比如以下是一个输出结果：

<pre>
["SBARQ", ["WHNP+PRON", "What"], ["SBARQ", ["SQ", ["VERB", "are"], ["NP+NOUN", "geckos"]], [".", "?"]]]
</pre>

如果是用 python 直接 `print` 的话，字符串是以单引号形式出现的，直接用 json 解析会出错。在 VIM 下可以用全局替换命令进行替换：`%s/'/"/g`。

<a href="https://gist.github.com/ctLiu3/5317639#file-gistfile2-py" title="A2P3" target="_blank">[code]</a>

### Part 3 (1 point)

该部分只是在 Part 2 的基础上，在 Parsing tree 的每个结点上加入了 `parent` 信息。因为对于原始的 PCFG 来说：Independence assumption is too strong! 由于该信息数据已经给出，所以求解过程跟 Part 2 差不多。只不过这时的 non-terminal 数据量会增加，复杂度会高些。如果你 Part 2 的代码已经是比较优化的，跑 Part 3 的数据也不会慢多少。

### Weakness of PCFG

作为一个教科书上的算法，PCGF 劣势 >=2：

1 没有充分利用句中单词带来的信息量。算法过程中只有在求解 $P(X \to X_i)$ 用到了单词的信息，而该信息没有作用到全局上。

2 没有充分利用句子的结构信息。同样的 rule，同样的 probability value，可能导致不同的 Parsing tree。
