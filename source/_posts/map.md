---
title: MAP最大后验估计
date: 2019-10-05 23:21:37
tags:
 - 机器学习
categories: 
    - 知识
---

## 为什么要引入最大后验？

最大似然估计仅仅是求得似然函数的极值点，考虑到实验结果数量较少，有的时候仅靠最大似然估计不足以求得较为准确参数估计值。
拿这枚硬币抛了10次，得到的数据（$x_0$）是：反正正正正反正正正反。我们想求的正面概率θθ是模型参数，而抛硬币模型我们可以假设是 二项分布。

<!-- more -->



那么，出现实验结果$x_0$（即反正正正正反正正正反）的似然函数是多少呢？

$$\begin{split}
f(x_0,\theta) &= (1−\theta)\times\theta\times\theta×\theta×\theta×(1−\theta)\times\theta×\theta×\theta×(1−\theta) \\
&=\theta^7(1−\theta)^3\\
&=f(θ)\\
\end{split}$$



注意，这是个只关于$\theta$的函数。而最大似然估计，顾名思义，就是要最大化这个函数。我们可以画出$f(θ)$的图像：

![likeli](https://tva1.sinaimg.cn/large/006tNbRwly1galj8v8jn8j31bk0qomyt.jpg)

可以看出，在$θ=0.7$时，似然函数取得最大值。

这样，我们已经完成了对$θ$的最大似然估计。即，抛10次硬币，发现7次硬币正面向上，最大似然估计认为正面向上的概率是0.7。（ummm..这非常直观合理，对吧？）

且慢，一些人可能会说，硬币一般都是均匀的啊！ 就算你做实验发现结果是“反正正正正反正正正反”，我也不信$θ=0.7$。

这里就包含了贝叶斯学派的思想了——要考虑先验概率。 为此，引入了最大后验概率估计。



## 最大后验估计

最大似然估计是求参数$θ$, 使似然函数$P(x0|θ)$最大。最大后验概率估计则是想求$θ$使$P(x0|θ)P(θ)$最大。求得的$θ$不单单让似然函数大，$θ$自己出现的先验概率也得大。 （这有点像正则化里加惩罚项的思想，不过正则化里是利用加法，而MAP里是利用乘法）

MAP其实是在最大化$P(θ|x_0)=\frac{P(x_0|θ)P(θ)}{P(x_0)}$，不过因为$x_0$是确定的（即投出的“反正正正正反正正正反”），$P(x_0)$是一个已知值，所以去掉了分母$P(x0)$（假设“投10次硬币”是一次实验，实验做了1000次，“反正正正正反正正正反”出现了n次，则$P(x_0)=n/1000P(x_0)=n/1000$。总之，这是一个可以由数据集得到的值）。最大化$P(θ|x_0)$的意义也很明确，$x_0$已经出现了，要求$θ$取什么值使$P(θ|x_0)$最大。顺带一提，$P(θ|x_0)$即后验概率，这就是“最大后验概率估计”名字的由来。


对于投硬币的例子来看，我们认为（”先验地知道“）$θ$取0.5的概率很大，取其他值的概率小一些。我们用一个高斯分布来具体描述我们掌握的这个先验知识，例如假设$P(θ)$为均值0.5，方差0.1的高斯函数，如下图：

![ptheta](https://tva1.sinaimg.cn/large/006tNbRwly1galkg7t34cj30zk0qomyd.jpg)



则$P(x0|θ)P(θ)$的函数图像为：

![map1](https://tva1.sinaimg.cn/large/006tNbRwly1galkgdntyfj30zk0qodgz.jpg)

注意，此时函数取最大值时，$θ$取值已向左偏移，不再是0.7。实际上，在$θ=0.558$时函数取得了最大值。即，用最大后验概率估计，得到$θ=0.558$
最后，那要怎样才能说服一个贝叶斯派相信$θ=0.7$呢？你得多做点实验。。

如果做了1000次实验，其中700次都是正面向上，这时似然函数为:

![likeli2](https://tva1.sinaimg.cn/large/006tNbRwly1galkglnlzaj30zk0qomy1.jpg)

如果仍然假设$P(θ)$为均值0.5，方差0.1的高斯函数，$P(x0|θ)P(θ)$的函数图像为：

![map2](https://tva1.sinaimg.cn/large/006tNbRwly1galkgtqxixj30zk0qoaat.jpg)

在$θ=0.696$处，$P(x0|θ)P(θ)$取得最大值。

这样，就算一个考虑了先验概率的贝叶斯派，也不得不承认得把$θ$估计在0.7附近了。

PS. 要是遇上了顽固的贝叶斯派，认为$P(θ=0.5)=1$ ，那就没得玩了。。 无论怎么做实验，使用MAP估计出来都是$θ=0.5$。这也说明，一个合理的先验概率假设是很重要的。（通常，先验概率能从数据中直接分析得到）

## 最大似然估计和最大后验概率估计的区别
相信读完上文，MLE和MAP的区别应该是很清楚的了。MAP就是多个作为因子的先验概率P(θ)P(θ)。或者，也可以反过来，认为MLE是把先验概率$P(θ)$认为等于1，即认为$θ$是均匀分布。



## MAP 分类

说实话我也不知道这是个啥东西，5644的一次作业题里有这样一个问题：

![image-20200105123707614](https://tva1.sinaimg.cn/large/006tNbRwly1galkxp2e62j31go0u048z.jpg)

大体意思就是服从高斯分布的三个类，给了均值和方差，你可以生成出来三组数据，混合在一起的。然后希望你用MAP的方式把它分开，跟踪分对和分错的数量，给出confusion matrix。

操作方式就是：

对于所有的点来说，他们都有可能分属于3个类别，算出来对于每一个类别（有多大可能性属于它），然后选出来最大的那个类别，就是这个点最可能属于的类。



计算的公式我觉得这里叫做MAP其实不太合适，这只是个后验概率而已，公式是后验=条件概率x先验。当然，还要取对数。先验已知了。$ln(P(x|L_k))$ 这个条件概率（就是已知这个点属于第K类的时候，它的概率密度）由高斯分布pdf的公式可以求出来。

![image-20200105125105925](https://tva1.sinaimg.cn/large/006tNbRwly1gallc873upj30so0mg104.jpg)

Numbers of each sample:
    
sample 1: 1517 
    
sample 2: 3441 
    
sample 3: 5042 


  Classification Rule (MAP) Description:

  From the data generated, we could see this is a typical GMM model, with 3 class prior and each class's $\Sigma$ and $\mu$，Let $P(L_k) = P(L=k),k ={1,2,3}$



  $$\begin{gather*}
    g_k(x) = ln(P(x|L_k)) + ln(P(L_k)) \\
    ln(P(x|L_k))  = -\frac{1}{2}ln(2\pi)-\frac{1}{2}ln|\Sigma_{k}|- \frac{1}{2}(x-\mu)\Sigma^{-1}(x-\mu)'   \\     
\end{gather*}$$  



  For each point, by using pdf we can calculate all the points for 3 class's $arg_{C_k}maxg_k(x)$ 
  as a score. Then the maximum of 3 scores is the decision of coresponding class decision label. 
$$
confusion\_matrix = 
              \left[
        \begin{matrix}
    1226 & 120& 22\\
    
    218 &   3102  &     101\\
    
    73   &  219   &  4919\end{matrix} 
    \right]
$$

total misclassified samples: 753 

probability of error 0.075300 

![image-20200105125128320](https://tva1.sinaimg.cn/large/006tNbRwly1gallcmnakrj30kq0ku459.jpg)

脚本链接：
https://github.com/nightsnack/ce564_ml/blob/master/Midterm/Question1.m