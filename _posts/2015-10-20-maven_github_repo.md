---
type: post
layout: post
category:	Maven
title: Github搭建Maven的远程仓库
tagline: by Snail
tags: [Java, Maven]
---

利用Github搭建Maven的远程仓库

<!--more-->

##Depoly到本地目录

在项目中配置本地的部署地址，并且depoly到本地

```
<distributionManagement>
  <repository>
    <id>alienxt-mvn-repo</id>
    <url>file:/usr/local/maven/my_repo/</url>
  </repository>
</distributionManagement>
```

##把本地目录提交到Github上

可使用git命令将本地的仓库提交到github上，也可以使用如SourceTree，Git客户端提交

```
cd /usr/local/maven/my_repo/
git init
git add repository/*
git commit -m 'deploy maven repo'
git remote add origin git@github.com:alienxt/repository.git
git push origin master
```

##引用远程仓库的依赖

只需在maven项目中的pom.xml中添加github上自己的仓库地址，即可下载到

```
<repositories>
  <repository>
    <id>alienxt-maven-repo</id>
    <url>https://raw.githubusercontent.com/alienxt/repository/master/</url>
  </repository>
</repositories>
```
