---
layout: post
title: 基于协同过滤的推荐系统实战
category: ML
comments: false
---

闲来无事，复习一下推荐算法。

## 一、基于协同过滤的推荐系统原理

- User-Based Collaborative Filtering（基于用户的协同过滤）：通过计算用户近似度找出近似用户，然后基于这些用户对目标商品的评分来预测目标用户的评分。
- Item-Based Collaborative Filtering（基于商品的协同过滤）：计算商品的相似度找出近似商品，然后基于用户对相似商品的评分来预算目标商品的评分。

该算法的好处是:

- 能起到意想不到的推荐效果, 经常能推荐出来一些惊喜结果
- 进行有效的长尾item
- 只依赖用户行为, 不需要对内容进行深入了解, 使用范围广

该算法的坏处是:

- 一开始需要大量的`<user,item>`行为数据, 即需要大量冷启动数据
- 很难给出合理的推荐解释

## 1.1 相似性度量的选择

- 当您的数据受用户偏好/用户的不同评分尺度影响时，请使用皮尔逊相似度
- 如果数据稀疏，则使用余弦（许多额定值未定义）
- 如果您的数据不稀疏并且属性值的大小很重要，请使用欧几里得（Euclidean）。
- 建议使用调整后的余弦（Adjusted Cosine Similarity）进行基于商品的方法来调整用户偏好。

## 1.2 评估推荐系统

评估推荐系统有很多评估指标。然而，最常用的是RMSE（均方根误差）。函数evaluateRS使用sklearn的mean_squared_error函数计算预测评级与实际评级之间的RMSE，并显示所选方法的RMSE值。 （为了简化说明，使用了小数据集，因此尚未将其分为训练集和测试集;本文中也未考虑交叉验证）

## 1.3 Item-CF和User-CF选择
可以根据user和item数量分布以及变化频率来选择：

- 如果user数量远远大于item数量, 采用Item-CF效果会更好, 因为同一个item对应的打分会比较多, 而且计算量会相对较少
- 如果item数量远远大于user数量, 则采用User-CF效果会更好, 原因同上
- 在实际生产环境中, 有可能因为用户无登陆, 而cookie信息又极不稳定, 导致只能使用Item-CF
- 如果用户行为变化频率很慢(比如小说), 用User-CF结果会比较稳定
- 如果用户行为变化频率很快(比如音乐, 电影等), 用Item-CF结果会比较稳定
- 对于item更新时效性较高的产品, 比如新闻, 就无法直接采用Item-based的CF, 因为CF是需要批量计算的, 在计算结果出来之前新的item是无法被推荐出来的, 导致数据时效性偏低; 这是用User-CF
- user-based出的更有可能有惊喜, 因为看的是人与人的相似性, 推出来的结果可能更有惊喜

## 二、代码

中文版：[https://cloud.tencent.com/developer/article/1095876](https://cloud.tencent.com/developer/article/1095876)

英文版：[https://github.com/haluomao/JupyterNotebooks-Medium/blob/master/CF%20Recommendation%20System-Examples.ipynb](https://github.com/haluomao/JupyterNotebooks-Medium/blob/master/CF%20Recommendation%20System-Examples.ipynb)

核心代码示例：
    
    #!/usr/bin/env python
    # -*- coding: utf-8 -*-

    import numpy as np
    import pandas as pd
    import matplotlib.pyplot as plt
    import sklearn.metrics as metrics
    import numpy as np
    from sklearn.neighbors import NearestNeighbors
    from scipy.spatial.distance import correlation, cosine
    import ipywidgets as widgets
    from IPython.display import display, clear_output
    from sklearn.metrics import pairwise_distances
    from sklearn.metrics import mean_squared_error
    from math import sqrt
    import sys, os
    from contextlib import contextmanager


    #M is user-item ratings matrix where ratings are integers from 1-10
    M = np.asarray([[3,7,4,9,9,7],
                    [7,0,5,3,8,8],
                   [7,5,5,0,8,4],
                   [5,6,8,5,9,8],
                   [5,8,8,8,10,9],
                   [7,7,0,4,7,8]])
    M=pd.DataFrame(M)

    #declaring k,metric as global which can be changed by the user later
    global k,metric
    k=4
    metric='cosine' #can be changed to 'correlation' for Pearson correlation similaries


    # This function finds k similar users given the user_id and ratings matrix M
    # Note that the similarities are same as obtained via using pairwise_distances
    def findksimilarusers(user_id, ratings, metric=metric, k=k):
        """
        :param user_id: 目标用户
        :param ratings: 用户的评分
        :param metric: 指标
        :param k: 返回几个相似的用户
        :return: similarities 返回相似值（第一个是自己，顺序递减），indices对应用户下标
        """
        similarities = []
        indices = []
        # 在sklearn中，NearestNeighbors方法可用于基于各种相似性度量搜索k个最近邻
        model_knn = NearestNeighbors(metric=metric, algorithm='brute')
        model_knn.fit(ratings)

        distances, indices = model_knn.kneighbors(ratings.iloc[user_id - 1, :].values.reshape(1, -1), n_neighbors=k)
        similarities = 1 - distances.flatten()
        print '{0} most similar users for User {1}:\n'.format(k - 1, user_id)
        for i in range(0, len(indices.flatten())):
            if indices.flatten()[i] + 1 == user_id:
                continue
            else:
                print '{0}: User {1}, with similarity of {2}'.format(i,
                    indices.flatten()[i] + 1, similarities.flatten()[i])

        return similarities, indices


    # This function predicts rating for specified user-item combination based on user-based approach
    def predict_userbased(user_id, item_id, ratings, metric=metric, k=k):
        prediction = 0
        similarities, indices = findksimilarusers(user_id, ratings, metric, k)  # similar users based on cosine similarity
        mean_rating = ratings.loc[user_id - 1, :].mean()  # to adjust for zero based indexing
        sum_wt = np.sum(similarities) - 1
        product = 1
        wtd_sum = 0

        for i in range(0, len(indices.flatten())):
            if indices.flatten()[i] + 1 == user_id:
                continue
            else:
                ratings_diff = ratings.iloc[indices.flatten()[i], item_id - 1] - np.mean(
                    ratings.iloc[indices.flatten()[i], :])
                product = ratings_diff * (similarities[i])
                wtd_sum = wtd_sum + product

        prediction = int(round(mean_rating + (wtd_sum / sum_wt)))
        print '\nPredicted rating for user {0} -> item {1}: (prediction value){2}'.format(user_id, item_id, prediction)

        return prediction

    #This function finds k similar items given the item_id and ratings matrix M
    def findksimilaritems(item_id, ratings, metric=metric, k=k):
        similarities=[]
        indices=[]
        ratings=ratings.T
        model_knn = NearestNeighbors(metric = metric, algorithm = 'brute')
        model_knn.fit(ratings)

        distances, indices = model_knn.kneighbors(ratings.iloc[item_id-1, :].values.reshape(1, -1), n_neighbors = k+1)
        similarities = 1-distances.flatten()
        print '{0} most similar items for item {1}:\n'.format(k,item_id)
        for i in range(0, len(indices.flatten())):
            if indices.flatten()[i]+1 == item_id:
                continue
            else:
                print '{0}: Item {1} :, with similarity of {2}'.format(i,indices.flatten()[i]+1, similarities.flatten()[i])

        return similarities,indices


    # This function predicts the rating for specified user-item combination based on item-based approach
    def predict_itembased(user_id, item_id, ratings, metric=metric, k=k):
        prediction = wtd_sum = 0
        similarities, indices = findksimilaritems(item_id, ratings)  # similar users based on correlation coefficients
        sum_wt = np.sum(similarities) - 1
        product = 1

        for i in range(0, len(indices.flatten())):
            if indices.flatten()[i] + 1 == item_id:
                continue
            else:
                product = ratings.iloc[user_id - 1, indices.flatten()[i]] * (similarities[i])
                wtd_sum = wtd_sum + product
        prediction = int(round(wtd_sum / sum_wt))
        print '\nPredicted rating for user {0} -> item {1}: {2}'.format(user_id, item_id, prediction)

        return prediction


    if __name__ == '__main__':
        similarities, indices = findksimilarusers(1, M, metric='cosine')
        # 'correlation' for Pearson correlation similarities
        similarities, indices = findksimilarusers(1, M, metric='correlation')
        predict_userbased(3, 4, M)

        similarities, indices = findksimilaritems(3, M)
        prediction = predict_itembased(1, 3, M)

输出结果：

    3 most similar users for User 1:
    1: User 5, with similarity of 0.973889935402
    2: User 4, with similarity of 0.934621684178
    3: User 6, with similarity of 0.88460045723

    3 most similar users for User 1:
    1: User 5, with similarity of 0.761904761905
    2: User 6, with similarity of 0.277350098113
    3: User 4, with similarity of 0.208179450927

    3 most similar users for User 3:
    1: User 4, with similarity of 0.90951268934
    2: User 2, with similarity of 0.874744414849
    3: User 5, with similarity of 0.86545387815

    Predicted rating for user 3 -> item 4: (prediction value)3

    4 most similar items for item 3:
    1: Item 5 :, with similarity of 0.918336125535
    2: Item 6 :, with similarity of 0.874759773038
    3: Item 1 :, with similarity of 0.810364746222
    4: Item 4 :, with similarity of 0.796917800302

    4 most similar items for item 3:
    1: Item 5 :, with similarity of 0.918336125535
    2: Item 6 :, with similarity of 0.874759773038
    3: Item 1 :, with similarity of 0.810364746222
    4: Item 4 :, with similarity of 0.796917800302

    Predicted rating for user 1 -> item 3: 7