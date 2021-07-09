---
title: Maven部署项目
categories: Maven
tags:
  - Maven
abbrlink: ff0da5c1
date: 2021-07-09 10:55:17
---




# Maven私服部署和下载项目

公司目前搭建了Maven的私服， 私服的地址是http://maven.jindidata.com/nexus 

我们在其中有一个自定义的组： http://maven.jindidata.com/nexus/content/groups/data/， 可以直接使用此组进行jar的下载。 

## 下载依赖

使用私服我们有几种简单的方式。 第一种最简单的方式就是在`pom.xml`里面增加以下代码引用

```xml
<repositories>
  <repository>
    <id>jindi-nexus</id>
    <url>http://maven.jindidata.com/nexus/content/groups/data/</url>
  </repository>
</repositories>
```

但是这种方法是一次性的，也就是启动一个新项目就要重新拷贝一次代码。 

有一种一劳永逸的方法，就是修改maven的settings.xml. 在profiles增加以下代码

```xml
 <profile>
        <id>nexus</id>
        <repositories>
            <repository>
                <id>jindi-nexus</id>
                <name>Nexus</name>
                <url>http://maven.jindidata.com/nexus/content/groups/data/</url>
                <releases>
                    <enabled>true</enabled>
                </releases>
                <snapshots>
                    <enabled>true</enabled>
                </snapshots>
            </repository>
      </repositories>

      <pluginRepositories>
            <pluginRepository>
                <id>jindi-nexus</id>
                <name>Nexus</name>
                <url>http://maven.jindidata.com/nexus/content/groups/data/</url>
                <releases>
                      <enabled>true</enabled>
                </releases>
                <snapshots>
                      <enabled>true</enabled>
                </snapshots>
            </pluginRepository>
      </pluginRepositories>
    </profile>
```

并在其中的activeProfiles中增加repository的id, 已激活这个仓库，这样相当于每个maven工程的pom.xml都会有这样一个内部仓库。 

```xml
<activeProfiles>
    <activeProfile>nexus</activeProfile>
</activeProfiles>
```

## 部署项目

我们公司的私服是使用nexus搭建的， 如果想将自己开发的项目的jar上传到公司的私服，供他人使用，只需要以下几步就能简单实现。 

### 自动部署

我们需要在settings.xml里面的servers配置我们私服的管理员账号密码， 自动部署的时候会从里面拿到账号密码进行认证

在<server></server>里面粘贴下面的代码块

```xml
    <server>
        <id>jindi-nexus-snapshots</id>
        <username>admin</username>
        <password>admin123</password>
    </server>
    <server>
        <id>jindi-nexus-release</id>
        <username>admin</username>
        <password>admin123</password>
    </server>
```

同时，我们需要在自己的项目中添加以下代码。

```xml
<distributionManagement>
    <repository>
        <id>jindi-nexus-release</id>
        <name>maven-releases</name>
        <url>http://maven.jindidata.com/nexus/content/repositories/jindi_data-release/</url>
    </repository>
    <snapshotRepository>
        <id>jindi-nexus-snapshots</id>
        <name>maven-snapshots</name>
        <url>http://maven.jindidata.com/nexus/content/repositories/jindi_data-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

这里大家可能有点疑惑，为什么会有repository 和 snapshotRepository 两个配置？简单来说

> repository是正式版本的，一旦上传某个版本就无法在修改里面的内容，相当于发版。 
>
> snapshotRepository是测试版本， 即使上传了某个测试版本（如1.0-SNAPSHOT)， 我们也能重复的去部署（上传）

那怎么确定我们的jar包上传到了那个仓库？

> 如果你的pom.xml的版本包含SNAPSHOT, 则上传的是测试仓库（snapshotRepository）, 否则上传的是正式仓库。

还有一个要注意的： 为什么我们上传的URL和下载的URL不是同一个？

> 咱们下载的私服仓库地址实际上是几个仓库组成起来的一个组，相当于下载URL包含了上传的两个仓库以及其他的代理仓库。 

下面这张截图展示了我们组的配置， 我们下载某个jar时会从其中的一个仓库去获取

![image-20210527110838798](/Users/bamboo/Documents/image-20210527110838798.png)

### 手动部署

如果咱们的jar是从其他地方下载的，而不是自己的工程项目，则可以使用此命令上传本地的jar包。 

```
mvn deploy:deploy-file -Dmaven.test.skip=true -Dfile=/Users/bamboo/tethys-ylinkc-1.2.1.jar -DgroupId=com.tyc -DartifactId=tethys-ylinkc -Dversion=LF.1.2.1 -Dpackaging=jar -DrepositoryId=jindi-nexus-release  -Durl=http://maven.jindidata.com/nexus/content/repositories/jindi_data-release/
```

注意这里的属性-DrepositoryId代表的是servers里面配置的id.  而-Durl代表我们要上传的仓库。 这里需要保证我们正式版指定`正式版仓库`，测试版上传到`测试仓库` 

参考文档：

maven私服搭建： https://www.hangge.com/blog/cache/detail_2844.html

maven 中settings.xml详解：https://www.cnblogs.com/jingmoxukong/p/6050172.html

Maven的settings.xml文件结构之Servers，Mirror和Repository: https://blog.csdn.net/qq32933432/article/details/104440175

Maven 多仓库和镜像配置: https://einverne.github.io/post/2019/04/maven-multiple-repository-and-mirror.html

官方指南： https://maven.apache.org/guides/mini/guide-mirror-settings.html



