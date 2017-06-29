---
title: 数据湖：提供大数据解决方案的架构
tags:
  - big data
  - data lake
  - 数据湖
id: 1308
categories:
  - 大数据
date: 2016-10-15 12:50:02
---

自从Dixon James在2010年提出数据湖(data lake)这个概念以来(原文见[ "Pentaho, Hadoop, and Data Lakes"](https://jamesdixon.wordpress.com/2010/10/14/pentaho-hadoop-and-data-lakes/))，数据湖逐渐成为各大厂商提供大数据解决方案的架构。

那么什么是数据湖呢？
数据湖具有灵活的定义。 其核心是一个数据存储和处理存储库，其中可以放置组织中的所有数据，以便每个内部和外部系统，合作伙伴和协作者的数据流入其中，并输出分析结果。

下面的列表详细说明了数据湖是什么：
&bull;数据湖是一个巨大的资源库，各种数据保存其原始格式，直到组织中的任何人需要分析。
&bull;数据湖不是Hadoop。它使用各种不同的工具。Hadoop只是实现的功能的一个子集。
&bull;数据湖不是传统字面上理解的数据库。典型的数据湖实施使用各种NoSQL和内存数据库，可以与它的关系并存。
&bull;数据湖不能孤立地实施。它必须与数据仓库一起实施因为它补充了数据仓库的各种功能。
&bull;它存储大量的非结构化和结构化数据。它还存储来自机器传感器的快速移动的数据流数据和日志。
&bull;它提供一种Store-All方法来处理大量数据。
&bull;它针对具有高延迟批处理模式的数据处理进行了优化，其不适用于事务处理。
&bull;它有助于创建灵活的数据模型，并可以修改，而不需要数据库重新设计。
&bull;它可以快速地进行数据丰富化，有助于实现数据的增强，增加，分类和标准化。
&bull;所有存储在数据湖中的数据可用于获得全包视图， 这有助于生成多维模型。
&bull;这是一个数据科学家最喜欢的狩猎场。 他可以最细粒度地访问存储在其原始光荣中的数据，以便他可以执行任何特别查询，并且可以随时迭代地构建高级模型。经典的数据仓库的方法不支持这种在数据采集和洞察生成之间缩减处理时间的能力。

优势：
&bull;尽可能地缩放
&bull;提供不同的数据源的插件
&bull;获取高速数据
&bull;处理非结构化数据
&bull;以原生格式存储
&bull;不要担心模式
&bull;使用您最喜欢的SQL
&bull;提供高级算法
&bull;节省人力资源

挑战：
数据湖是一个复杂的解决方案，因为构建它涉及到许多层，每个层利用大量的大数据工具和技术来完成其功能。 这需要在部署，管理和维护方面的大量工作。 另一个重要方面是数据治理; 由于数据湖旨在将所有组织的数据集中在一起，因此它应该建立起足够的治理，使其不会变成一组无关的数据孤岛。

推荐阅读：
[Data Lake Development with Big Data](https://www.amazon.com/gp/product/1785888080)
[Data Lake Architecture: Designing the Data Lake and Avoiding the Garbage Dump](https://www.amazon.com/gp/product/1634621174)
[Azure Data Lake](https://azure.microsoft.com/en-us/solutions/data-lake/)
[Data Lake on AWS](https://aws.amazon.com/cn/big-data/data-lake-on-aws/)