+++
title = "演化多目标优化：基于支配的经典算法NSGA-II"
date = 2021-02-05

[taxonomies]
categories = ["Notes", "Legacy"]
tags = ["Algorithm", "Evolutionary Computation"]
+++

最近在学习并尝试实现演化多目标优化的相关算法，希望我能通过这篇博客把算法了解得更加透彻orz

## 基础篇

### 帕累托支配

在多目标优化领域，一般来说，决策变量和目标都是由多个元素组成的，而不同目标之间往往难以比较。如果将决策/目标视作一个向量$\mathbf{x}$，我们用**帕累托支配（Pareto dominance）**来刻画向量间的关系。对于两个$n$维向量$\mathbf{x}^{(1)} = (x^{(1)}_1, \dots, x^{(1)}_n)^T, \mathbf{x}^{(2)} = (x^{(2)}_1, \dots, x^{(2)}_n)^T$，我们称$\mathbf{x}^{(1)}$支配$\mathbf{x}^{(2)}$（记作$\mathbf{x}^{(1)} \prec \mathbf{x}^{(2)}$）当且仅当$\forall 1 \leq i \leq n,\ x^{(1)}_i \leq x^{(2)}_i$，且$\exists 1 \leq i \leq n,\ x^{(1)}_i < x^{(2)}_i$。

<!-- more -->

在最大化问题的上下文中，上述不等号需反向。算法最终得出的互不支配的解集，称为**帕累托集**（Pareto set, PS），对应的目标集合称为**帕累托前沿**（Pareto front, PF）。PF的形状和收敛性往往被用来作为评估算法的重要指标。

###  非支配排序

由于支配关系是一种偏序关系，我们只能退而求其次去将整个集合分割成若干子集$F_1, F_2, \dots, F_m$，其中同一子集的元素相互不支配，且$\forall 1 \leq i < j \leq m, \forall \mathbf{x} \in F_i, \forall \mathbf{y} \in F_j,\ \mathbf{x} \prec \mathbf{y}$。该排序过程称为**非支配排序**（Non-dominated sort）。

在介绍NSGA-II的论文中，作者提出了一个较为简单的实现方式：如果将整个集合和支配关系视作一个有向图，则每次只需将入度为0的点归入一个子集，然后从图中删掉这些点和对应的边。重复此过程直到整张图为空。这样我们自然地得到了若干非支配的集合。

此外，还有一种更高效的基于树的排序方法T_ENS，实现暂时留空。

### 竞争选择

在演化计算中，为了加速结果收敛的速度，多数算法会采取一些选择策略来决定下一代的个体。其中一个经典的模型就是**竞争选择**（Tournament selection）模型。假设要从某集合中选择$N$个个体（允许重复），则每次从集合中随机选择$K$个个体，然后从这$K$个个体中选择**适应值**（Fitness value）最优（一般取最小）的一个。如此重复$N$次。这样可以保证适应值较优的个体有更大的选择概率。PlatEMO中的代码实现如下：

``` matlab
% 该函数接受K、N以及个体的适应值（可能有多个），并返回选择的下标
function index = TournamentSelection(K, N, varargin)
    % 将适应值整理成一列cell并比较得到rank
    varargin  = cellfun(@(S)reshape(S, [], 1), varargin, 'UniformOutput', false);
    [~, rank] = sortrows([varargin{:}]);
    [~, rank] = sort(rank);
    % 生成一个K*N的随机下标矩阵
    Parents   = randi(length(varargin{1}), K, N);
    % 比较每一列的rank值，得到最小值下标并返回
    [~, best] = min(rank(Parents), [], 1);
    index     = Parents(best + (0:N-1)*K);
end
```

代码中连续两次sort的原理参考[这篇博客](@/posts/permutation-rank.md)。最后一行利用了一个MATLAB的语法糖，有兴趣的读者可以自行实验。

### 遗传算法的交叉和变异

感觉这一部分适合用一篇单独的文章来记录……具体内容请移步[这里](@/posts/ga-cross-mutate.md)。

## NSGA-II的主体框架

NSGA-II（Non-dominated Sorting Genetic Algorithm II），顾名思义，是基于非支配排序进行选择的算法。对每一代个体，会先利用竞争选择来选出亲代个体进行交叉和变异，然后将当前代和新个体放在一起进行非支配排序并选出下一代个体。在这个选择过程中，有可能会出现“某一个非支配子集的个体选不完”的情况，此时就需要利用**拥挤距离**（Crowding distance）来做进一步选择。

在某个非支配子集中，一个个体的拥挤距离被定义为：每个维度上将这个个体“夹住”的两个个体的“归一化差值”。“归一化差值”指的是这两个个体在该维度上分量差的绝对值除以这个维度上所有个体的最大值和最小值的差。如果某个体在某一维度上已经是最大/最小值，那该个体的拥挤距离为无穷大。基于拥挤距离的选择将选出拥挤距离最大的若干个体来维持整个解集的多样性，和其它个体距离过近的个体将会被剔除。

基于支配的演化算法在实现上非常直观，但其在高维问题上表现不甚理想。这是因为在高维决策空间中，得到的解大多是互相不支配的，这样整个演化的过程无异于随机选择，在收敛上表现较差。