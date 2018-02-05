---
layout: post
title: (翻译)An introduction to machine learning with scikit-learn
category: 系列
tags: 深度学习
keywords: 深度学习
description:
---

[原文](http://scikit-learn.org/stable/tutorial/basic/tutorial.html)

# An introduction to machine learning with scikit-learn

## Machine learning: the problem setting

一般情况下，一个学习问题可以被看作是根据 n 组样本数据，然后试图预测未知数据特性的问题。如果每组样品数据是多于一个单独数字，那么就可以看成一个多维记录（aka multivariate data），也就意味着拥有多个属性或特征。

我们可以将学习问题分为一个几个大类：

* [监督学习](https://en.wikipedia.org/wiki/Supervised_learning)，这种情况是指数据有我们希望的可以预测的额外属性（[Click here](http://scikit-learn.org/stable/supervised_learning.html#supervised-learning) to go to the scikit-learn supervised learning page）.这种问题可能是：
    * 分类（[classification](https://en.wikipedia.org/wiki/Classification_in_machine_learning)）：样本数据属于两个或更多的分类，而我们希望可以从已经标记分类（labeled）的数据上学习如果分类未标记分类的数据。分类问题的一个例子是手写数字的识别，手写数字的识别就是给每个输入向量对应有限数量的离散类别中的一个。另一种理解分类问题的方法是一种离散形式的监督学习（另一种是连续的），把 n 组样本数据分类或标记给正确的有限数量类别。
    * 回归（[regression](https://en.wikipedia.org/wiki/Regression_analysis)）：如果期望的到的输出结果包含一种或多种连续变量，那么这种类型的任务可以称为回归类型问题。回归问题的一个例子是，可以根据鲑鱼的年龄和重量预测长度。
* [非监督学习](https://en.wikipedia.org/wiki/Unsupervised_learning)，这种情况下训练数据包含一组输入向量 x ,这些输入并没有对应一些特定值。非监督学习的目标也许是在数据中发现一些类似的情况，这种情况被称为聚类([clustering](https://en.wikipedia.org/wiki/Cluster_analysis))；也许是为了判断在输入空间内的数据分布，也被称为密度估计([density estimation](https://en.wikipedia.org/wiki/Density_estimation))；也许是为了数据可视化的目的，将数据从高维投影到二维或三维([Click here](http://scikit-learn.org/stable/unsupervised_learning.html#unsupervised-learning) to go to the Scikit-Learn unsupervised learning page)

### Training set and testing set
Machine learning is about learning some properties of a data set and applying them to new data. This is why a common practice in machine learning to evaluate an algorithm is to split the data at hand into two sets, one that we call the training set on which we learn data properties and one that we call the testing set on which we test these properties.



> <small>lastest-update (2017-06-07)</small>