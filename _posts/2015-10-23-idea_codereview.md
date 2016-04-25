---
type: post
layout: post
category:	Maven
title: Intellij idea配置Codereview
tagline: by Snail
tags: [Java, Maven]
---

Intellij idea上配置代码审查及格式化

<!--more-->

##为什么需要规范代码及代码审查？

1. 提高代码质量2. 及早发现潜在缺陷bug，降低事故成本3. 促进团队内部共享意识，提高团队整体水平4. 评审过程对于审查人员来说，也是思路的一种重构过程，帮助更多人理解系统


##IDEA上安装CheckStyle

1. IDEA 中，选择File ->Setting ->Plugins2. 打开Browse Repositories，搜索：CheckStyle-IDEA3. 选择Install，然后重启IDEA
4. 重新打开IDEA，选择File ->Setting -> Other Settings，可以看到CheckStyle


##IDEA配置代码审查

##CheckStyle配置及使用

1. 您可以自定义checkstyle的检查配置，也可以使用标准的规范，以下为谷歌的配置，可作为参考，也可以直接下载下来用
[*谷歌CheckStyle规范*](https://github.com/checkstyle/checkstyle/blob/master/src/main/resources/google_checks.xml)

2. 添加配置有什么用呢？我们可以配置用来检查我们的源代码是否命令规范，是否格式规范，是否有javadoc，注释是否规范等

3. 假设我们不配置的话，其实checkstyle也是有默认的规范配置文件的

4. 配置完了以后我们该怎么用呢，如图在项目中，切换底部的tab到checkstyle中，点击run，就可以看到错误信息

![](/img/2015-10-23-idea_codereview/idea_codereview_1.png)


##在Maven中添加CheckStyle插件

在开发过程中，我们肯定不能时不时手动去检查源文件，这时我们可以通过maven的插件，来实现在我们编译代码，打包的时候自动检查代码，当检查不通过时，编译失败，只需要在<build>模块中添加如下配置：

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>2.16</version>
    <configuration>
        <configLocation>/path/to/your/config.xml</configLocation>
    </configuration>
    <executions>
        <execution>
            <id>checkstyle</id>
            <phase>validate</phase>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <failOnViolation>true</failOnViolation>
            </configuration>
        </execution>
    </executions>
</plugin>
```


同时在这里推荐另外一个maven插件FindBugs，它能自动检查一些程序员大意疏忽的bug。比如：NullPoint空指针检查、没有合理关闭资源、字符串相同判断错（==，而不是equals）等许多检查，一样也是在pom.xml中的<build>中添加插件即可使用


```
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>findbugs-maven-plugin</artifactId>
    <version>3.0.1</version>
    <executions>
        <execution>
            <id>findbugs</id>
            <phase>compile</phase>
            <goals>
                <goal>findbugs</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

##File Header配置

1. IDEA 中，选择File ->Setting -> Editor -> Editor2. 选择File and Code Templates -> Includes -> 添加3. 添加文件名称为File Header，内容如下，需要修改${user}为自己的名字

```/*** <p>文件描述：</p>* <p>版权所有： 版权所有(C)2013-2099</p>* <p>公司： xxx</p>* <p>内容摘要： </p>* <p>其他说明： </p>** @version 1.0* @author <a> href="mailto:${user}@xxx.com">${user}</a>* @since ${date} ${time}*/
```
4.然后你尝试着新建个.java文件，就可以看到文件头的变化














