---
title: Eclat算法实现
date: 2017-12-05 13:51:22
tags:
	- 数据挖掘
	- python
	- 算法实现
	- 频繁模式挖掘
categories: 数据挖掘
---

之前实现了{% post_link Apriori算法实现 Apriori算法 %}以及{% post_link FP-growth算法实现 FP-growth算法 %}，用的同一个数据集:

| 单号 | 购物清单 |
|:---:|:----:|
|T100 | I1,I2,I5 |
|T200 | I2,I4 |
|T300 | I2,I3 |
|T400 | I1,I2,I4 |
|T500 | I1,I3 |
|T600 | I2,I3 |
|T700 | I1,I3 |
|T800 | I1,I2,I3,I5|
|T900 | I1,I2,I3 |

`Apriori`和`FP-growth`算法都是以单号为一条记录，依次读入一条这样的事务数据并进行頻数计数。

而`Eclat`算法主体思想是先将这样的数据做一次"转换"，直接以每条事务记录中的项为关键字，在这个样例数据中，这样的一条事务数据中的项就成为了单号。例如上述的数据集就可以转化成了:

| 物品ID | 包含该物品的单号 |
|:---:|:----:|
| I1 | T100,T400,T500,T700,T800,T900 |
| I2 | T100,T200,T300,T400,T600,T800,T900 |
| I3 | T300,T500,T600,T700,T800,T900 |
| I4 | T200,T400 |
| I5 | T100,T800 |

这样在统计支持度的时候就可以通过计算每条记录中的元素个数了。例如在统计1-频繁项集的时候，对每条记录施以计算集合的大小，就可以得出，`{I1}`的支持度计数为6等，然后筛选。在进行2-频繁项集的时候对剩下的记录进行笛卡尔积，然后求交集的大小即可得到2-频繁项集的支持度。依次类推得到k-频繁项集的支持度计数。

## 代码

读入数据部分与前面两个算法一样，采用的[样例数据](http://archive.ics.uci.edu/ml/machine-learning-databases/voting-records/house-votes-84.data)也一样，在此就不展开了。

较为关键的是数据变换的部分:
``` python
def transpose_dataset(dataset, min_sup = 0):

    transed_dta = {}

    record_idx = 0

    for record in dataset:				# 原单条的事务数据

        for ele in record:

            if frozenset({ele}) in transed_dta: 	# 以原事务数据中的项为关键字查找记录

                transed_dta[frozenset({ele})].add(record_idx)

            else:

                transed_dta[frozenset({ele})] = {record_idx}

        record_idx += 1

    qualified_data = {}

    for k, v in transed_dta.items():	# 筛选支持度不符的数据

        if len(v) >= min_sup:

            qualified_data[k] = v

    return qualified_data
```

后面大体的过程与`Apriori`算法差不多，不过就简单了很多:
``` python
def eclat(dataset, min_sup=0):

    tran_data = transpose_dataset(dataset, min_sup)	# 转换数据，顺便做了1-频繁项集的筛选

    k_sets = [tran_data]

    frequent_itemsets = []

    frequent_item_list = []

    for rec, val in tran_data.items():

        frequent_item_list.append((rec, len(val)))

    frequent_itemsets.append(frequent_item_list.copy())

    k = 1											# 从2-频繁项集开始

    while len(k_sets[k - 1]) > 0:

        k_1_set = k_sets[k - 1]

        k_set = {}

        for k1, v1 in k_1_set.items():

            for k2, v2 in k_1_set.items():

                new_key = k1.union(k2)				# 由k-1频繁项集並集产生k-频繁项集

                if len(new_key) == (k + 1) and new_key not in k_set:	# 检查是否为k

                    intersec = v1.intersection(v2)	# 由原来两个k-1频繁项集的数据集合交集产生k频繁项集的数据集合

                    if len(intersec) >= min_sup:	# 这里的支持度计数就很简单了，直接球集合的大小

                        k_set[new_key] = intersec

        frequent_item_list.clear()

        for rec, val in k_set.items():

            frequent_item_list.append((rec, len(val)))

        frequent_itemsets.append(frequent_item_list.copy())	# 加入结果集

        k_sets.append(k_set)

        k += 1

    return frequent_itemsets
```
`Eclat`的算法很简单，但是其思想启发了数据挖掘的一个新思路。