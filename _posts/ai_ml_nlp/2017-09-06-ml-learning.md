---
layout: post
title:  Machine Learning Foundation
category: ML
comments: false
---

# Part 1. 一些概念

From [https://medium.com/@ageitgey/machine-learning-is-fun-80ea3ec3c471](https://medium.com/@ageitgey/machine-learning-is-fun-80ea3ec3c471)

通过一个房地产商定房价的例子来引出ML的应用。

- cost function（损失函数）：一个模型计算出来的结果与真实结果之间的差异。How wrong your answer are.
- batch gradient descent (批量梯度下降)：一种找到函数的最好权值的方法。不断尝试使得损失函数求的值最小。梯度下降很依赖初始值。与梯度下降的区别（batch的区别）：每次迭代都要遍历整个训练集合。
- linear regression 可以用来处理linear data, nerural networks 和 SVMs能处理non-linear data.
- overfitting (过拟合): 在训练数据里works well,但是不适用于新数据。 可以用Regularization 和 cross-validation(交叉验证) 来处理。

- least square fit : 最小二乘拟合。
- converge: 收敛

# Part 2. Neural network

[https://medium.com/@ageitgey/machine-learning-is-fun-part-2-a26a10b68df3](https://medium.com/@ageitgey/machine-learning-is-fun-part-2-a26a10b68df3)

神经网络由很多的simple node（neuron）相互chaining together而成。

- feature scaling (特征缩放)
- acitivation function (激励函数)
- 为神经网络加上Memory,使其不再是 stateless algorithm. 每次计算，都会有一个state: Save the model's current state and use that as part of our next calcuation.
- Recurrent Neural Network （RNN，卷积神经网络）


# Part 3. Linear regression

无人驾驶是一个线性回归的问题。

Notation:
    - m : the number of training example 
    - x : input vars/features
    - y : output vars/targets
    - (x,y) ： training example
    - h : 表示假设函数
    - n : the number of features.
    - theta : parameters 

Gradient descent:
    Ai := Ai - J(A)'

J(A)'是对J(A)求偏导。

不断地求下降速度最快的点，直到最低点。

# Part 4. gradient descent
梯度下降是一种找到函数的最好极值的方法。（其实就是求导，然后求最小值，有点像下山）

batch gradient descent (批量梯度下降)与梯度下降的区别：每次迭代都要遍历整个训练集合。

批量梯度下降不适合训练集很大的场景，因为要计算很多次。所以出现了incremetal gradient descent(增量梯度下降)：每次迭代的时候，都更新参数theta，因为只要遍历样本个数的次数，所以会很快。但是这个算法并不会得到一个最小值，而是在最小值附近徘徊。


# REF
> (机器学习简介 (Introduction to Machine Learning)[https://developers.google.com/machine-learning/crash-course/ml-intro?hl=zh-cn]

