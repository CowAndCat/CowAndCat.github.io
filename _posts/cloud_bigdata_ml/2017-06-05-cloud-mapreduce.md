---
layout: post
title: Map/Reduce
category: cloud
comments: false
---
# 1. Map/Reduce 基本组件

用于大规模数据集的分布式并行计算的编程模型.

- Hadoop：提供可靠存储HDFS以及MapReduce编程范式以便大规模并行处理数据。
- Spark：提供基于分布式**内存**的大规模并行处理框架，从而大大提高大数据分析性能。Spark提供了SQL查询接口、流数据处理以及机器学习。Spark通过拓展内存计算可在海量数据的迭代式计算和交互式计算中提供远快于Hadoop的运算速度。
- HBase：大规模分布式NoSQL数据库，提供随机存取大量的非结构化和半结构化的海量数据。
- Hive：Hive是基于Hadoop的数据仓库工具，提供海量数据的读取、写入、管理和分析，具有易扩展的存储能力和计算能力。允许使用类似于SQL语法进行数据查询，适合数据仓库的分析任务。
- Pig：是一种过程语言，可加载数据、表达转换数据以及存储最终结果，使得日志等半结构化数据变得有意义。
- Hue：为了方便管理Hadoop集群以及执行Hive或者Pig脚本而提供的一系列网页应用。
- Sqoop：用于Hadoop与传统的数据库间的数据导入和导出。
- Kafka：开源的、高吞吐量的分布式消息队列系统，支持Hadoop并行数据加载。
- Zeppelin：Web版的notebook，用于数据分析和可视化，可无缝对接Hive、SparkSQL等。