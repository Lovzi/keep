---
‘title: Flink之TimerService
categories: Flink
tags:
  - Flink
abbrlink: 477d83ab
date: 2021-09-29 16:41:04
---



## Flink的时间概念

在Flink中，时间概念主要分为三种类型，即事件事件、处理事件和接入事件。 



### Watermark Strategies

为了使用eventTime， flink1.13重构了。

