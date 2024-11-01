---
layout:  post
title:   "pg数据库查询优化器optimizer"
date:   2024-10-31 9:40:00
author:  'Xiangxiao'
header-img: 'img/post-bg-2015.jpg'
catalog:   false
tags:
- c
- optimizer
- 查询优化器
- postgres内核
---

# 1 查询优化器概述
查询优化器在pg中的位置是一个承上启下的地方，它接收语法分析模块传递过来的查询树，并在这个语法树的基础上进行逻辑上的等价变换和物理执行路径的筛选，并且把选择出来的最优的执行路径传递给数据库的执行器模块。

> 它接收的是查询树，输出的是查询执行计划。

数据库的查询优化分为两种，基于规则的查询优化（逻辑优化，Rule Based Optimizetion）；基于代价的查询优化（物理优化，Cost Based Optimization）。

# 2 文件介绍
在pg中查询优化器的源码位置在`/src/backend/optimizer`中，其中还有五个子目录，分别是`plan`、`prep`、`geqo`、`path`、`util`。

其中plan是总的入口，它调用prep目录实现逻辑优化，调用path和geqo目录实现物理优化，而util目录则是一些公共函数，供所有子目录使用。

