---
title: MAC&IDEA&ITEM自定义快捷键
abbrlink: 2bd54fe2
---


## 使用

目前公司使用地址：https://git.ty.ink/， 通过飞书授权登陆后，联系方哥申请开发权限，并邀请进组。 

### 修改语言环境

在成功登陆之后， 点击 Settings -> Preferences ->  Localization 

![image-20210518171141949](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210518171141949.png)



### 禁用流水线

目前流水线还没有接入真正的外部测试组件， 但是流水线会一直卡在那里，因为暂时禁用流水线

#### 操作方法

可以在项目的 **Settings > General > Permissions** 下找到启用或禁用 GitLab CI/CD 的设置。选择“禁用”

![image-20210518205754258](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210518205754258.png)

### Fork流开发

#### 发起合并请求

在个人仓库中点击`新建合并请求`

![image-20210518173920031](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210518173920031.png)

选择源分支 -> 目的分支

![image-20210518174030489](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210518174030489.png)

这里需要编写本次合并请求的标题和描述，用来描述这次提交主要的功能。  

在这里可以选择性的添加代码负责人和reviewer， reviewer帮助来提高代码的审核，减少上线前可能出现的问题。

最后选择提交合并请求

![image-20210518174804730](https://gitee.com/Goook/pictures/raw/master/uPic/image-20210518174804730.png)

