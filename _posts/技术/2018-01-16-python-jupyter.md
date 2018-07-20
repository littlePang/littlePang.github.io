---
layout: post
title: python jupyter 数据分析使用
category: 技术
tags: python
keywords:
description:
---

# 文档
jupyter官网：[http://jupyter.org/](http://jupyter.org/)
推荐用 anaconda 来使用jupyter，相关的依赖包啥的都自动搞好了，不用自己搞环境，[传送门](https://www.anaconda.com/download/#macos)

# 例子
以下例子使用 python2.7的语法编写

### 需要的相关模块引入

      import pandas as pd
      import numpy as np
      import matplotlib.pyplot as plt

### 图像中文展示
前提是需要安装 SimHei 字体
查看 matplotlib 所在目录：print(matplotlib.matplotlib_fname())

      plt.rcParams['font.sans-serif'] = ['SimHei']  # 用来正常显示中文标签
      plt.rcParams['axes.unicode_minus'] = False  # 用来正常显示负号
      plt.axis('equal') # 设置圆的长宽相等，则图形展示为圆，而非椭圆

### 数据加载

      // 数据载入，列之间使用 \t 分割
      all_user_df = pd.read_csv("/Users/jaky.wang/tmp/data.csv", sep='\t')

      // 查看前几行数据
      all_user_df.head()

      // 过滤出 age=10的数据
      filter_data = all_user_df[all_user_df.age<20]

      // 多个条件过滤，过滤出age大于5小于20的数据
      filter_data = all_user_df[(all_user_df.age<20) & (all_user_df.age>5)]

      // 提取指定行的指定列，第一个参数为从第几行到第几行，第二个参数为指定列名的集合
      extract_data = all_user_df.loc[:,['user_name','age']]

      // 两个数据集merge成一个数据集，类似于sql的join
      merge_data = pd.merge(all_user_df, all_user_df, how="inner", on='userid')

      // 使用 行名 作为 merge条件，并添加字段后缀
      pd.merge(all_user_df, all_user_df, how='inner',
              left_index=True, right_index=True, suffixes=('_one', '_two'))

      // 使用数据集中的一行，算出一个新列，axis=1表示变量 x 是以行的维度处理，默认是列
      all_user_df['new_column'] = all_user_df.apply(lambda x:x.age*2, axis=1)

      // 以age进行分组,统计出现的次数
      name_count = all_user_df.groupby(all_user_df.age).count().user_name

      // 饼图展示数据，
      all_user_df.plot(kind='pie', subplots=True,autopct='%.2f',figsize=(15.0, 8.0))
      plt.show()

      // 排序，逆序
      sort_df = all_user_df.sort_values('userid', ascending=False)

      // 保存到文件
      all_user_df.to_csv("/Users/jaky.wang/data.csv")


## 多个array在同一图上做曲线图

common_all_consumed 和 all_consumed 都是普通的 list

      #创建figure窗口
      plt.figure(num=3, figsize=(15, 8))

      plt.plot(x_axix, np.array(common_all_consumed), color='red')
      plt.plot(x_axix, np.array(all_consumed), color='black')

      #设置坐标轴范围
      plt.xlim((0, 23))
      plt.ylim((0, 1500000000))
      #设置坐标轴名称
      plt.xlabel('time')
      plt.ylabel('consumed')
      #设置坐标轴刻度
      my_x_ticks = np.arange(0, 24, 1)
      my_y_ticks = np.arange(0, 1500000000, 50000000)
      plt.xticks(my_x_ticks)
      plt.yticks(my_y_ticks)

      #显示出所有设置
      plt.show()
