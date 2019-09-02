# Elasticsearch5.6源码编译与运行

一、软件环境

- Intellij Idea:2018.2.8

- Elasticsearch源码版本:5.6

- JDK:1.8.0_221

- Gradle:4.3

二、编译运行

1、编译执行gradlew build出现如下错误

```
you must run gradle idea from the root of elasticsearch before importing into intellij
```

解决办法

运行命令：gradlew idea

>注意不是jdk不是8u40以上的版本，会报下面的错误
JDK Oracle Corporation 1.8.0_20 has compiler bug JDK-8052388, update your JDK to at least 8u40
---

2、运行

导入依赖完成后，运行编译命令：gradlew build，过程比较慢。

