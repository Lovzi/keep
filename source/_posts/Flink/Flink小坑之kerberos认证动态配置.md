---
title: Flink小坑之kerberos认证动态配置
categories: Flink
tags:
  - Flink
date: 2021-10-12 14:09:39
---




## Flink为什么需要kerberos认证

当Hadoop生态使用了kerberos认证之后，Flink向Yarn提交任务时同样需要kerberos认证，否则无法提交作业。 



