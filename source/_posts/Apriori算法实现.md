---
title: Apriori算法实现
date: 2017-11-20 22:33:59
tags: 
- 数据挖掘
- python
- 算法实现
- 频繁模式挖掘
categories: 数据挖掘
---

数据挖掘作为当前比较热门的领域之一，已经有较为权威的书籍指导入门了，如陈封能著，范明译的《数据挖掘导论》以及力量黑书(机械工业出版社)出版的《数据挖掘概念与技术》。

在介绍完基本的数据挖掘概念之后，第一个动手写代码实现的算法就是`Apriori`算法了。`Apriori`算法是用于挖掘关联规则的频繁项集算法。

## 环境支持
-	Python 3.5
-	美国国会84年投票记录[UCI下载链接](http://archive.ics.uci.edu/ml/machine-learning-databases/voting-records/house-votes-84.data)

## Apriori中的剪枝步骤

在介绍伪代码前要介绍`Apriori`算法中的`剪枝`步骤。在产生`K=1,...`频繁项集的过程中，一共有$\sum_{i=1}^{K}C_n^i$个候选集，不用百万级的数据，光是`n`大于1000的时候都可以产生组合爆炸，更别说对产生的组合进行统计。所以`Apriori`算法在统计候选集之前先要把产生的`K`候选集作一个剪枝，删除不频繁的`K`候选集再开始统计。

具体的做法是检验该`K`项集的`K-1`子项集是否为频繁项集，如果该`K`项集的所有子项集都是频繁项集，那么该`K`项集才有可能是频繁项集。

假设有`{A,B,C,D}`这个全集，在`K=3`有`{A,B,C}`和`{A,B,D}`等组合(其它组合情况省略)。设频繁阈值为1，而`K-1`时存在`{\{A,B},{A,C\}}`项集而没有`{A，D}`项集，那么`{A,B,D}`肯定不是频繁项集，因而在下一轮统计前就可以删除了。

## Apriori算法伪代码

``` code
Input:
	D:	事务数据库
	min_sup: 最小支持度阈值
Ouput:
	L,D中的频繁项集
Procedure:
	L1=find_frequent_1_itemsets(D);
	for(k=2; Lk-1!=EmptySet; k++)
	{
		Ck=apriori_gen(Lk-1);
		for each transaction t in D 		// 	扫描D,进行计数
		{		
			Ct=subset(Ck,t);				//	得到t的子集，候选集
			for each candidate c in Ct 
			{
				c.count++;
			}
		}
		Lk={c(Ck|c.count>=min_sup)}
	}
	return L union_k Lk;

Procedure apriori_gen(Lk-1: frequent(k-1)itemset)
	for each itemset l1 in Lk-1
	{
		for each itemset l2 in Lk-1
		{
			if(l1[1]=l2[1])and...and(l1[k-2]=l2[k-2])and(l1[k-1]<l2[k-1]) then
			{
				c=l1 × l2 (Cartesian product)
				if has_infrequent_subset(c, Lk-1) then
					delete c;						// 删除非频繁的候选
				else
					add c to Ck;
			}
		}
	}
	return Ck;

Procedure has_infrequent_subset(c: candidate k itemset; Lk-1: frequent(k-1)itemset)
	for each (k-1) subset s of c
	{
		if s not in Lk-1 then
			return true;
	}
	return false;
```

简单介绍了基本概念以及算法之后，一般读者都会迷迷糊糊的，这很正常，毕竟这篇博客是用来讲实现了，伪代码也因为Markdown的代码块里面不允许加载特殊html语法写得难看。建议真的想搞懂的话，就去看上述的书，然后手解模拟一下过程，一般懂了之后，实现的问题就不大了。

如上述环境支持，在这里实现的时候用的是美国国会84年的投票记录，实际上是什么数据集并没有关系，无非就是数据预处理的过程不一样了而已。针对这个数据集，笔者实现了如下代码用于加载与处理一行记录:
``` python
def translate_record(record):
    items = []

    if record[0] == 'republican':

        items.append('rep')

    elif record[0] == 'democrat':

        items.append('demo')

    if record[1] == 'y':

        items.append('hci')  # handicapped-infants

    elif record[1] == 'n':

        items.append('hci-n')

    if record[2] == 'y':

        items.append('wpcs')  # water-project-cost-sharing

    elif record[2] == 'n':

        items.append('wpcs-n')

    if record[3] == 'y':
        items.append('aotbr')  # adoption-of-the-budget-resolution

    elif record[3] == 'n':
        items.append('aotbr-n')

    if record[4] == 'y':

        items.append('pff')  # physician-fee-freeze

    elif record[4] == 'n':

        items.append('pff-n')

    if record[5] == 'y':

        items.append('esa')  # el-salvador-aid

    elif record[5] == 'n':

        items.append('esa-n')

    if record[6] == 'y':
        items.append('rgis')  # religious-groups-in-schools

    elif record[6] == 'n':
        items.append('rgis-n')

    if record[7] == 'y':

        items.append('astb')  # anti-satellite-test-ban

    elif record[7] == 'n':

        items.append('astb-n')

    if record[8] == 'y':

        items.append('atnc')  # aid-to-nicaraguan-contras

    elif record[8] == 'n':

        items.append('atnc-n')

    if record[9] == 'y':

        items.append('mxm')  # mx-missile

    elif record[9] == 'n':

        items.append('mxm-n')

    if record[10] == 'y':

        items.append('imm')  # immigration

    elif record[10] == 'n':

        items.append('imm-n')

    if record[11] == 'y':

        items.append('scc')  # synfuels-corporation-cutback

    elif record[11] == 'n':

        items.append('scc-n')

    if record[12] == 'y':

        items.append('es')  # education-spending

    elif record[12] == 'n':

        items.append('es-n')

    if record[13] == 'y':

        items.append('srts')  # superfund-right-to-sue

    elif record[13] == 'n':

        items.append('srts-n')

    if record[14] == 'y':

        items.append('cri')  # crime

    elif record[14] == 'n':

        items.append('cri-n')

    if record[15] == 'y':
        items.append('dfe')  # duty-free-exports

    elif record[15] == 'n':
        items.append('dfe-n')

    if record[16] == 'y':
        items.append('eaasa')  # export-administration-act-south-africa

    elif record[16] == 'n':
        items.append('eaasa-n')

    return items

def load_data(file_path):
    src_data = open(file_path)

    votes_records = []

    for line in src_data:
        votes_records.append(translate_record(line.strip('\n').split(',')))

    src_data.close()

    return votes_records
```
其中，`load_data`用于读入数据集并返回一个处理好的数据列表。

然后就是伪代码第一行，就是产生1项集，并统计1频繁项集:
``` python
def find_frequent_1_itemsets(vote_records, min_sup=0):
    itemsets = []

    for record in vote_records:

        for element in record:

            if [element] not in itemsets:
                itemsets.append([element])

    itemsets = map(frozenset, itemsets)						# 使用frozenset使得集合可以作为dict中的key

    itemsets_with_count = {}

    for candidate in itemsets:								# 产生候选项

        for record in vote_records:

            if candidate.issubset(record):					# 如果该候选是记录中的子集，增加计数值

                if candidate in itemsets_with_count:

                    itemsets_with_count[candidate] += 1

                else:

                    itemsets_with_count[candidate] = 1

    qualified_itemsets = {}

    for key, val in itemsets_with_count.items():			# 筛选候选项

        if val >= min_sup:

            qualified_itemsets[key] = val

    return qualified_itemsets
```
注意，为了能够用`dict`来统计频繁项集的个数，其中的`key`的类型是一个集合，而一般的集合是不能来当作`key`的，幸好python提供了`frozenset`使得集合作为`dict`的`key`。不过这样的实现也有很大的性能缺陷，频繁地在普通的集合以及`frozenset`中转化，深度复制，性能损失可想而知。

当时实现的时候一个优化的想法就是用一个4Byte的数据结构来表示一个集合，其中的`0`和`1`分别代表`yes`和`no`，然而数据还存在第三个状态，用扩展位来解决这个问题，可以通过拼凑的方法使得这样的数据结构可以兼容任意个选项。这样的方法通用性不好，加上这里是第一次实现，所以就还是采取`frozenset`这个平凡的办法来处理了。

然后在`K>=2`开始，需要用到的`has_infrequent_set`,用于检测该候选项是否存在非频繁子项集:
``` python
def has_infrequent_set(un_set, frozen_set):

    unfrozen_un = set(un_set)

    for ele in unfrozen_un:

        sub_k_1_set = unfrozen_un.copy()		# 复制该候选项，并一次迭代中删除一个元素，产生子集

        sub_k_1_set.remove(ele)

        sub_k_1_set = frozenset(sub_k_1_set)	# frozen化使之能够成为索引

        if sub_k_1_set not in frozen_set:		# 如果子集不在频繁项集中，则为非频繁项集

            return True

    return False
```
这里就体现出了普通的集合和`frozenset`的频繁转化了。

接下来是产生`K(K>1)`项集的`apriori_gen`:
``` python
def apriori_gen(lk):

    frozen_set = lk.keys()									# 取出K-1项集

    if len(frozen_set) <= 0:

        return []

    for ele in frozen_set: set_size = len(ele) + 1; break	# 设定K的值

    gen_set = []

    for ele1 in frozen_set:

        for ele2 in frozen_set:

            if not (ele1.issubset(ele2) and ele1.issuperset(ele2)):	# 如果不是同一个集合

                un_set = ele1.union(ele2)							# 取并集

                if len(un_set) == set_size and un_set not in gen_set and not has_infrequent_set(un_set, frozen_set):
                	# 测试并集是否为K项集并且还未被之前的过程产生，而且没有非频繁K-1子集
                    gen_set.append(un_set)

    return gen_set
```
准备好`Apriori`用到的主要过程之后，就是`Apriori`算法的主体了:
``` python
def apriori(record_set, min_sup):

    l1 = find_frequent_1_itemsets(record_set, min_sup)	# 产生1项集

    L = [l1]

    k = 1

    while len(L[k - 1]) > 0:							# 只要K频繁项集还有，算法就不结束

        ck = apriori_gen(L[k - 1])						# 从K-1项集中产生K项

        count_set = {}

        verified_count_set = {}

        for candidate in ck:							# 统计产生的K项集的支持度

            count_set[candidate] = 0

            for record in record_set:

                if candidate.issubset(record):

                    count_set[candidate] += 1

        for key, val in count_set.items():				# 统计符合要求的K频繁项

            if val >= min_sup:

                verified_count_set[key] = val

        L.append(verified_count_set.copy())				# 加入到结果中，准备下一轮迭代

        count_set.clear()

        verified_count_set.clear()

        k += 1

    return L
```
经典的`Apriori`算法的主体代码实现就到这里就结束了。

算法实现并不难，难得是弄懂这个过程，所以还是看书，自己手动模拟过程更快一些。