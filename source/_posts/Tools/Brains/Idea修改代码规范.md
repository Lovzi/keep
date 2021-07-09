---
title: Idea修改代码规范
categories: Brains
tags:
  - Brains
abbrlink: 2a505619
date: 2021-07-09 10:56:15
---


## Scala规范

### 修改类缩进方式

之前是这样的

```scala
case class LitigationRelatedDetail(
                                      id: Long,
                                      uuid: String,
                                      plaintiffs: String,
                                      defendants: String,
                                      caseType: String,
                                      caseIdentity: String,
                                      caseReason: String,
                                      caseCodes: String,
                                      court: String,
                                      trialProcedure: String,
                                      trialTime: String,
                                      trainYear: String,
                                      caseUUID: String,
                                      gids: String,
                                      isDelete: Int,
                                      updateTime: String,
                                      createTime: String,
                                      source: String
)

```

修改成这样

```scala

case class LitigationRelatedDetail(
  id: Long,
  uuid: String,
  plaintiffs: String,
  defendants: String,
  caseType: String,
  caseIdentity: String,
  caseReason: String,
  caseCodes: String,
  court: String,
  trialProcedure: String,
  trialTime: String,
  trainYear: String,
  caseUUID: String,
  gids: String,
  isDelete: Int,
  updateTime: String,
  createTime: String,
  source: String
)
```

解决方案： Preferences ->  Editer -> Code Style -> Scala -> Wrapping and Braces -> Align when multiline 