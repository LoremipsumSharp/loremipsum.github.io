---
title: Maven库配置阿里云加速
---

由于国内的网络原因,通过maven拉取相关package的时候总是特别慢，极大的影响到工作效率。所幸的是阿里云提供了maven库的镜像服务，体验了一下，速度飞快。配置方法如下：

## Maven仓库配置阿里云加速
- 确定settings.xml配置文件位置
执行如下命令：

```
mvn -X
```

输入如下：

```
.....省略
[DEBUG] Reading global settings from /usr/share/maven/conf/settings.xml
[DEBUG] Reading user settings from /home/abc/.m2/settings.xml
[DEBUG] Reading global toolchains from /usr/share/maven/conf/toolchains.xml
[DEBUG] Reading user toolchains from /home/loremipsum/.m2/toolchains.xml
[DEBUG] Using local repository at /home/abc/.m2/repository

.....省略
```

- 修改settings.xml,修改mirrors xml结点下的相关配置

```
  <mirrors>
    <!-- mirror
     | Specifies a repository mirror site to use instead of a given repository. The repository that
     | this mirror serves has an ID that matches the mirrorOf element of this mirror. IDs are used
     | for inheritance and direct lookup purposes, and must be unique across the set of mirrors.
     |
    <mirror>
      <id>mirrorId</id>
      <mirrorOf>repositoryId</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://my.repository.com/repo/path</url>
    </mirror>
     -->
    <mirror>
      <id>aliyun-public</id>
      <name>aliyun public</name>
      <mirrorOf>public</mirrorOf>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>

    <mirror>
      <id>aliyun-central</id>
      <name>aliyun central</name>
      <mirrorOf>central</mirrorOf>
      <url>https://maven.aliyun.com/repository/central</url>
    </mirror>

    <mirror>
      <id>aliyun-jcenter</id>
      <name>aliyun jcenter</name>
      <mirrorOf>jcenter</mirrorOf>
      <url>https://maven.aliyun.com/repository/jcenter</url>
    </mirror>

    <mirror>
      <id>aliyun-spring</id>
      <name>aliyun spring</name>
      <mirrorOf>spring</mirrorOf>
      <url>https://maven.aliyun.com/repository/spring</url>
    </mirror>

    <mirror>
      <id>aliyun-spring-milestones</id>
      <name>aliyun spring milestones</name>
      <mirrorOf>spring-milestones</mirrorOf>
      <url>https://maven.aliyun.com/repository/spring</url>
    </mirror>

    <mirror>
      <id>aliyun-spring-plugin</id>
      <name>aliyun spring plugin</name>
      <mirrorOf>spring-plugin</mirrorOf>
      <url>https://maven.aliyun.com/repository/spring-plugin</url>
    </mirror>

    <mirror>
      <id>aliyun-gradle-plugin</id>
      <name>aliyun gradle plugin</name>
      <mirrorOf>gradle-plugin</mirrorOf>
      <url>https://maven.aliyun.com/repository/gradle-plugin</url>
    </mirror>

    <mirror>
      <id>aliyun-google</id>
      <name>aliyun google</name>
      <mirrorOf>google</mirrorOf>
      <url>https://maven.aliyun.com/repository/google</url>
    </mirror>

    <mirror>
      <id>aliyun-grails-core</id>
      <name>aliyun grails core</name>
      <mirrorOf>grails-core</mirrorOf>
      <url>https://maven.aliyun.com/repository/grails-core</url>
    </mirror>
</mirrors>

```

## 防坑：
大部分的包阿里云的maven镜像库都有，一部分比较新的库上面是没有的，如springboot，阿里云上面的版本只是到了2.2.0，实际2.2.4都已经出来了，同步不是很及时。目前我都一些没有的库做了降级处理，后续再看看怎么处理这个问题
