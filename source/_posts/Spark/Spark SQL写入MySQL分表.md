---
title: Spark SQL写入MySQL分表
categories: Spark
tags:
  - Spark
abbrlink: cb3eb8b2
date: 2021-07-09 10:55:17
---






在业务逻辑使用Spark离线处理完数据之后， 有时候需要写入MySQL提供给后端使用。 但是Spark提供的接口目前不支持分表，这里需要自定义一个写入MySQL的接口

