---
title: 评分卡模型WOE+IV+ROC
date: 2017-12-24 11:28:27
tags:
    - Machine Learning
    - WOE
    - IV
    - ROC
---

## WOE（Weight Of Evidence：证据权重）
WOE的全称是“Weight of Evidence”，即证据权重。WOE是对原始自变量的一种编码形式。
要对一个变量进行WOE编码，需要首先把这个变量进行分组处理（也叫离散化、分箱等等，说的都是一个意思）。分组后，对于第i组，**WOE的计算公式**如下：
$$
WOE_i = \ln(\frac{py_i}{pn_i}) = \ln(\frac{\#y_i / \#y_T}{\#n_i / \#n_T})
$$

其中，$py_i$是这个组中响应客户（风险模型中，对应的是违约客户，总之，指的是模型中预测变量取值为“是”或者说1的个体）占所有样本中所有响应客户的比例，$pn_i$是这个组中未响应客户占样本中所有未响应客户的比例，$\\#y_i$是这个组中响应客户的数量，$\\#n_i$是这个组中未响应客户的数量，$\\#y_T$是样本中所有响应客户的数量，$\\#n_T$是样本中所有未响应客户的数量。

从这个公式中我们可以体会到，**WOE表示的实际上是“当前分组中响应客户占所有响应客户的比例”和“当前分组中没有响应的客户占所有没有响应的客户的比例”的差异**。

<!--more-->

对这个公式做一个简单变换，可以得到：
$$
  WOE_i = \ln(\frac{py_i}{pn_i}) = \ln(\frac{\#y_i/\#y_T}{\#n_i/\#n_T}) = \ln(\frac{\#y_i/\#n_i}{\#y_T/\#n_T})
$$

变换以后我们可以看出，WOE也可以这么理解，他表示的是当前这个组中响应的客户和未响应客户的比值，和所有样本中这个比值的差异。这个差异是用这两个比值的比值，再取对数来表示的。WOE越大，这种差异越大，这个分组里的样本响应的可能性就越大，WOE越小，差异越小，这个分组里的样本响应的可能性就越小。

## IV（Information Value：信息量）
有了前面的介绍，我们可以正式给出**IV的计算公式**。对于一个分组后的变量，第$i$组的WOE前面已经介绍过，是这样计算的：
$$
  WOE_i = \ln(\frac{py_i}{pn_i}) = \ln(\frac{\#y_i/\#y_T}{\#n_i/\#n_T}) = \ln(\frac{\#y_i/\#n_i}{\#y_T/\#n_T})
$$

同样，对于分组$i$，也会有一个对应的IV值，计算公式如下：
$$
 IV_i = (py_i - pn_i) * WOE_i = (py_i - pn_i) * \ln(\frac{py_i}{pn_i}) = (\frac{\#y_i}{\#y_T} - \frac{\#n_i}{\#n_T}) * \ln(\frac{\#y_i/\#y_T}{\#n_i/\#n_T})
$$

有了一个变量各分组的IV值，我们就可以计算整个变量的IV值，方法很简单，就是把各分组的IV相加：
$$
  IV = \sum_i^n IV_i
$$

其中，$n$为变量分组个数。

## ROC+WOE+IV意义解析
{% asset_img 1.png %}
数据来自注明的German credit dataset，取了其中一个自变量来说明问题。
- 第一列是自变量的取值，N表示对应每个取值的样本数。
- n1和n0表示违约样本和正常样本数，p1和p0分别表示违约样本与正常样本占各自总体的比例。
- cump1和cump0分别表示了p1和p0的累积和
- woe是对应自变量每个取值的WOE(ln(p1/p0))，iv是woe*(p1-p0)。

对iv求和（可以看成是对woe的加权求和），就得到**IV（Information Value），是衡量自变量对目标变量影响的指标之一（类似于gini，entropy）**，此处为0.666.

上述过程研究了一个自变量对目标变量的影响，事实上也可以看成是单个变量的评分模型，更进一步的说，可以直接将自变量的取值当做是某种信用评分的得分，此时需要假定自变量是某种有序变量，也就是仅仅根据这个有序的自变量对目标变量进行预测。

正是基于这种视角，可以将“模型效果的评价”与“自变量筛选及编码”这两个过程统一起来。筛选合适的自变量，并进行适当的编码，事实上就是挑选并构造出对目标变量有较高预测能力（predictive power）的自变量，同时也可以认为，由这些自变量分别建立的单变量评分模型，其模型效果也是比较好的。

以上面这个表格为例，其中的cump1和cump0，从某种角度看就是我们做ROC曲线时的TPR和FPR。例如，此时的评分排序为A12、A11、A14、A13，若以A14位cutoff，则此时的TPR=cumsum(p1)[3]/(sum(p1))， FPR=cumsum(p0)[3]/(sum(p0))，就是cump1[3]和cump0[3]。于是我们可以画出相应的ROC曲线：
{% asset_img 2.png %}

如上图所示，ROC曲线的量化指标AUC（曲线下方的面积），衡量了TPR和FPR之间的距离。根据上面的描述，从另一个角度看TPR和FPR，可以理解为这个自变量（也就是某种评分规则的得分）关于0/1目标变量的条件分布，例如TPR，即cump1，也就是当目标变量取1时，自变量（评分得分）的一个累计分布。当这两个条件分布距离较远时，说明这个自变量对目标变量有较好的辨识度。

既然条件分布函数能够描述这种辨识能力，那么条件密度函数行不行呢？这就引出了IV和WOE的概念。事实上，我们同样可以衡量两个条件密度函数的距离，这就是IV。这从IV的计算公式里面可以看出来，IV=sum((p1-p0)*log(p1/p0))，其中的p1和p0就是相应的密度值。IV这个定义是从相对熵演化过来的，里面仍然可以看到x*lnx的影子。

至此应该已经可以总结到：评价评分模型的效果可以从“条件分布函数距离”与“条件密度函数距离”这两个角度出发进行考虑，从而分别得到AUC和IV这两个指标。这两个指标当然也可以用来作为筛选自变量的指标，IV似乎更加常用一些。而WOE就是IV的一个主要成分。


那么，到底为什么要用WOE来对自变量做编码呢？主要有两个考虑：
1. 提升模型的预测结果，
2. 提高模型的可理解性。

首先，对已经存在的一个评分规则，
例如上述的A12,A11,A14,A13，对其做各种函数变化，可以得到不同的ROC结果。但是，如果这种函数变化是单调的，那么ROC曲线事实上是不发生变化的。因此，想要提高ROC，必须寄希望于对评分规则做非单调的变换。传说中的NP引理证明了，使得ROC达到最优的变换就是计算现有评分的一个WOE，这似乎叫做“条件似然比”变换。

用上述例子，我们根据计算出的WOE值，对评分规则（也就是第一列的value）做排序，得到新的一个评分规则。
{% asset_img 3.png %}

可以看出来，经过WOE的变化之后，模型的效果好多了。事实上，WOE也可以用违约概率来代替，两者没有本质的区别。用WOE来对自变量做编码的一大目的就是实现这种“条件似然比”变换，极大化辨识度。

同时，WOE与违约概率具有某种线性关系，从而通过这种WOE编码可以发现自变量与目标变量之间的非线性关系（例如U型或者倒U型关系）。在此基础上，我们可以预料到模型拟合出来的自变量系数应该都是正数，如果结果中出现了负数，应当考虑是否是来自自变量多重共线性的影响。

另外，WOE编码之后，自变量其实具备了某种标准化的性质，也就是说，自变量内部的各个取值之间都可以直接进行比较（WOE之间的比较），而不同自变量之间的各种取值也可以通过WOE进行直接的比较。进一步地，可以研究自变量内部WOE值的变异（波动）情况，结合模型拟合出的系数，构造出各个自变量的贡献率及相对重要性。一般地，系数越大，woe的方差越大，则自变量的贡献率越大（类似于某种方差贡献率），这也能够很直观地理解。

总结起来就是，做信用评分模型时，自变量的处理过程（包括编码与筛选）很大程度上是基于对单变量模型效果的评价。而在这个评价过程中，ROC与IV是从不同角度考察自变量对目标变量的影响力，基于这种考察，我们用WOE值对分类自变量进行编码，从而能够更直观地理解自变量对目标变量的作用效果及方向，同时提升预测效果。

这么一总结，似乎信用评分的建模过程更多地是分析的过程（而不是模型拟合的过程），也正因此，我们对模型参数的估计等等内容似乎并不做太多的学习，而把主要的精力集中于研究各个自变量与目标变量的关系，在此基础上对自变量做筛选和编码，最终再次评估模型的预测效果，并且对模型的各个自变量的效用作出相应的评价。
