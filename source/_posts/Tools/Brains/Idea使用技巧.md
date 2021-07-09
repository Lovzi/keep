---
title: Idea使用技巧
categories: Brains
tags:
  - Brains
abbrlink: b7429b01
date: 2021-07-09 10:56:14
---


## 修改GIt版本控制导致的文件颜色

Version Conntrol -> FIle Status

## 巧妙使用TODO来快速查找代码中需要注意的问题

todo是各种工具都实现的一种注释方案，可以对注释进行分类，并对不同注释高亮显示
最常用的是以下默认的todo

- `todo`， 正则： `\btodo:\b.*` 说明在标识处有功能代码待编写，待实现的功能在说明中会简略说明
- `fixme`, 正则： `\bfixme:\b.*` 如果代码中有该标识，说明标识处代码需要修正，甚至代码是错误的，不能工作，需要修复，如何修正会在说明中简略说明。
- `hack`： 正则： `\bhack:\b.*` 英语翻译为砍。如果代码中有该标识，说明标识处代码我们需要根据自己的需求去调整程序代码。
  自定义一些todo，方面查找
- `noting` 正则： `\bnoting:\b.*` 当前看到某种有价值代码可以借鉴，想记录到笔记中。
- `noted` 正则： `\bnoted:\b.*` 已记录到笔记中
- `plato` 正则： `\bplato:\b.*` 当前阅读代码遇到的问题。
- `cim` 正则： `\bcommit:\b.*` 当前代码开源，且有一个想法去修改这段代码。

[定义方式](https://blog.csdn.net/cgl125167016/article/details/79028073)



