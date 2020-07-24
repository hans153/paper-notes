# Password Cracking Using Probabilistic Context-Free Grammars

WEIR M, AGGARWAL S, MEDEIROS B de, 等. Password Cracking Using Probabilistic Context-Free Grammars[C]//2009 30th IEEE Symposium on Security and Privacy. 2009: 391–405.

## 问题提出

由于口令猜解主要是对哈希值进行恢复，一种优化方法是增加算力，例如采用专用硬件或分布式集群，另一种方法则是减少猜解次数，与之对应的猜解技术为暴力猜解与字典猜解。

暴力猜解指针对全部口令空间进行逐个猜解，通常利用预计算，以空间换时间的方式提高效率。

字典猜解指选取特定的一组字符串进行逐个猜解，通常选择常用的英文单词来作为基础字典。然而用户经常对单词进行变化以满足口令强度策略，致使攻击者也需要根据预先设置好的规则对基础词库进行变化。选取合适的规则将至关重要，如何选取合适的方式生成字典？

## 数学工具

### 上下文无关文法（context-free grammer）

上下文无关文法是现代编译器中最常使用的语法分析方法，可以构造快速的分析算法来检测给定字符串是否合法。形式化定义如下：
$$
G=(V,\Sigma,S,R)
$$
其中$V$表示非终止符的变量的有限集；$\Sigma$表示终止符的有限集合，与$V$没有交集，两者构成句子的实际内容；$S$表示开始符，必须是$V$的元素，$R$表示生成式规则的有限集，即所有$\alpha \to \beta$ 的集合，其中$\alpha$表示一个变量，$\beta$表示一组变量，可以包括终止符。由此可以定义出所有由开始符派生出的所有句子的集合。

### 概率上下文无关文法（Probabilistic context-free grammar）

概率上下文无关文法是基于统计方法对上下文无关文法进行拓展的一种语法分析方法，在自然语言处理与RNA结构分析中均有应用。形式化定义如下：
$$
G=(V,\Sigma,S,R,P)
$$
其中，$V$、$\Sigma$、$S$、$R$的定义与上下文无关文法一致，$P$是$R$中的生成式规则的概率的集合。

## 基于概率的猜解

作者提出了一个假设：并非所有的猜测都具有相同的成功猜解可能性（已经通过大量的真实用户口令的分布研究证实）。因此基于概率的猜解就是找出依次找出最可能成功的口令进行猜解。

### 数据预处理

作者选取了一些公开泄露的真实用户口令集，将其划分为训练集与测试集，并保证其中没有重叠的口令。

令$L$表示仅含字母的子字符串，$D$表示仅含数字的子字符串，$S$表示所有非数字与字母的子字符串，利用子字符串表示训练集中所有口令的结构，例如“@password123”可表示为$S_1L_8D_3$或简单表示为$SLD$。作者在这里做出了经验性的假设，认为口令结构的简单表示不太可能与上下文无关，因此在统计概率时选取复杂表示，即$L_nD_nS_n$的形式。统计所有口令的结构出现概率。

接着作者统计了训练集中所有$D$子字符串与$S$子字符串的出现概率。之所以未统计仅含字母的子字符串，作者认为用户可能使用的$L$子串会比训练集中出现的大得多，因此在此不进行统计，在构造字典时使用基础字典来生成。

### 基于PCFG的字典生成

作者阐述了如何构造语法。对于生成式规则集合$R$，规定$\alpha$，即生成式的左侧，为开始符号$S$与单独的$D_n$、$S_n$子字符串，其中开始符号$S$仅能推出结构。文法推出的句子中包含$L_n$子串，作者称之为“pre-terminal structure”，需要配合具体的字典使用。一个示例如下：

![image-20200724012924738](Paper%20-%20Password%20Cracking%20Using%20Probabilistic%20Context-Free%20Grammers.assets/image-20200724012924738.png)

则考虑一个如下的推导：
$$
S \to L_3D_1S_1 \to L_34S_1 \to L_34!
$$
其概率为0.0975。

给定这样一个“pre-terminal structure”，再给定例如一个字典{cat,hat,stuff,monkey}，则会生成以下两个猜测口令{cat4!, hat4!}，即最终的“terminal structure”。

在获得所有的“pre-terminal structure”后如何生成“terminal structure”并计算排序，作者提出了几种种方案：

1. **pre-terminal probability order**：选取最高概率的“pre-terminal structure”，并填充所有满足长度的$L$子串，接着选取次高概率依次处理，在这种情况下，每类“pre-terminal structure”生成的猜测口令的概率是相同的。
2. **terminal probability order**：选取字典中不同长度的单词的比例作为$L$子串的概率，在填充时将其与“pre-terminal structure”的概率相乘，将依据概率从高到低排序。
3. **pre-terminal sampled order**：根据所有的“pre-terminal structure”的概率进行采样，然后填充合适的$L$子串，生成字典。此时生成的字典不一定严格按照概率顺序。

### 生成方法的优化

作者考虑该方法不事先生成所有的字典再排序，而是随着猜测过程动态生成，即优先生成出当前最有可能的口令。

作者构造了一个基于队列的数据结构来计算所有组合。队列的初始值为训练集的结构与高可能性的子串形成的组合，以上文实例文法为例，队列初始情况为：

| 结构           | pre-terminal | 概率  |
| -------------- | ------------ | ----- |
| $D_1L_3S_2D_1$ | $4L_3\$\$4$  | 0.188 |
| $L_3D_1S_1$    | $L_34!$      | 0.097 |

接着，队列顶端的元素将出队，并以次高概率的子串进行替换并计算概率，并生成子队列插入主队列，为：

| 结构           | pre-terminal | 概率  |
| -------------- | ------------ | ----- |
| $L_3D_1S_1$    | $L_34!$      | 0.097 |
| $D_1L_3S_2D_1$ | $4L_3**4$    | 0.081 |
| $D_1L_3S_2D_1$ | $5L_3\$\$4$  | 0.063 |
| $D_1L_3S_2D_1$ | $4L_3\$\$5$  | 0.063 |

以此类推，最终逐个生成最优可能的组合，完整的生成过程如下图：

![image-20200724015919069](Paper%20-%20Password%20Cracking%20Using%20Probabilistic%20Context-Free%20Grammers.assets/image-20200724015919069.png)

## 实验

### 口令数据集

作者使用以下三个公开的口令数据集：

- Myspace数据集：主要通过钓鱼攻击获取，数量67042条；
- SilentWhisper数据集：主要通过SQL注入攻击获取，数量7480条；
- Finnish数据集：主要通过SQL注入攻击获取，数量15699条。

### 实验设置

作者采用John the ripper哈希猜解工具作为对照组，并直接使用了其字典作为基础字典。该字典主要为常用英语单词的语料。

作者将自己算法的允许猜测次数设置为与john the ripper工具提供的字典大小一致，以保证公平性。

在划分数据集上，作者保证了训练集与测试集没有重复口令。

### 实验结果

在各个数据集上的猜解结果：

![image-20200724023239088](Paper%20-%20Password%20Cracking%20Using%20Probabilistic%20Context-Free%20Grammers.assets/image-20200724023239088.png)

![image-20200724023357887](Paper%20-%20Password%20Cracking%20Using%20Probabilistic%20Context-Free%20Grammers.assets/image-20200724023357887.png)

![image-20200724023433468](Paper%20-%20Password%20Cracking%20Using%20Probabilistic%20Context-Free%20Grammers.assets/image-20200724023433468.png)

![image-20200724023449925](Paper%20-%20Password%20Cracking%20Using%20Probabilistic%20Context-Free%20Grammers.assets/image-20200724023449925.png)

其效果在采用上文提到的**terminal probability order**方式生成字典时，效果均强于john the ripper。

## 总结与展望

1. 本文是首次提出利用概率方式构造字典的口令猜解算法，其基本思路与后期提出的基于各类机器学习方法的口令猜解算法一致，均为生成出最有可能的口令来依次猜解口令。
2. 尽管文章发表较早，但其将口令划分为不同结构的子串的思路值得借鉴。
3. 作者采用上下文无关文法的方式构建口令时，其最基础的假设为用户生成口令时是上下文无关的，但在实际生活中上下文相关的口令也并不少见，例如采用类似email形式的口令：hans@buaa、或常见的姓名+生日的方式等。