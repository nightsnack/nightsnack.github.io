---
title: EM 算法的基本步骤
date: 2019-10-17 15:21:06
tags:
	- 机器学习
categories: 
    - 知识
mathjax: true
---

## **EM算法的步骤**
1. 选择合适的初始参数$\theta_0$，开始迭代（EM算法是初值敏感的）
2. E步：记$\theta_0$为第i次参数的估计值，计算第i+1次的Q函数的值$Q(\theta,\theta_i) = E_Z[log(Y,Z|\theta)|Y,\theta_i]=\Sigma_Z logP(Y,Z|\theta)P(Z|Y,\theta_i)$


3. M步：求使$Q(\theta,\theta_i)$极大化的参数$\theta$，确定新的参数估计值$\theta_{i+1} = argmax_{\theta}Q(\theta,\theta_i)$
4. 重复（2），（3）直到收敛，收敛条件一般为$||\theta_{i+1}-\theta_i||\le \in_1$或者$||Q(\theta_{i+1},\theta_i)-Q(\theta_i,\theta_i)||\le \in_2$

<!-- more -->