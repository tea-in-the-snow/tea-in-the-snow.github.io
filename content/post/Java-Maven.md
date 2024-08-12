---
title: "Maven Project"
date: 2024-07-21 11:37:42+0800
comments: true
categories:
    - Programming Languages
tags:
    - Java
    - Maven
---

## Java项目

一个Java项目的构建，首先需要确定要引入哪些依赖，每个依赖可能还会有自己的依赖，所以需要对依赖进行管理。其次，需要确定项目的目录结构。接着，需要配置环境，例如JDK的版本，编译打包的流程等。

这些工作非常复杂琐碎，所以需要一个标准化的Java项目管理和构建工具。Maven就是这样的一个工具，它提供了一套标准化的项目结构，提供了一套标准化的构建流程，提供了一套依赖管理机制。

## Maven项目结构

一个使用Maven管理的普通的Java项目，它的目录结构默认如下：

```java
  my-app/
  ├── pom.xml
  ├── src/
  │   ├── main/
  │   │   ├── java/
  │   │   │   └── com/
  │   │   │       └── mycompany/
  │   │   │           └── MyApp.java
  │   │   └── resources/
  │   │       └── application.properties
  │   └── test/
  │       ├── java/
  │       │   └── com/
  │       │       └── mycompany/
  │       │           └── MyAppTest.java
  │       └── resources/
  │           └── test.properties
  └── target/
```

- 目录`src/main/java`包含了项目的源代码，目录`src/test/java`包含了测试代码。
- `pom.xml`(POM: project object model)是项目配置的核心。

## POM

`pom.xml`文件大致结构如下：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>com.mycompany.app</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0-SNAPSHOT</version>
 
  <name>my-app</name>
  <!-- FIXME change it to the project's website -->
  <url>http://www.example.com</url>
 
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.7</maven.compiler.source>
    <maven.compiler.target>1.7</maven.compiler.target>
  </properties>
 
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
 
  <build>
    <pluginManagement><!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->
       ... lots of helpful plugins
    </pluginManagement>
  </build>
</project>
```

- `groupId`类似于Java的包名，通常是公司或组织名称。
- `artifactId`类似于Java的类名，通常是项目名称。
- `name`和`url`代表了项目的名称和项目网站的url。
- `properties`用于定义全局变量，在POM中任何位置都可以访问。
- `dependencies`用于声明项目所需的依赖。依赖的依赖关系和依赖之间的依赖关系会由Maven解决。
- `build`用于声明项目的目录结构，管理插件等。

## 更多Maven相关内容

[Apache Maven Getting Started Guide](https://maven.apache.org/guides/getting-started/index.html)
