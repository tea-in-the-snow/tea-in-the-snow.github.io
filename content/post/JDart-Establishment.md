---
title: "JDart 构建记录"
description: 在 Ubuntu24 系统上构建与配置 JDart 框架的记录。
date: 2025-10-21T13:36:35+08:00
hidden: false
comments: true
draft: false
categroies:
  - Static Analysis
tags:
  - JDart
---

## 环境

- Ubuntu 24.04 (wsl version on Windows11-pro)
- Java 8.0.462-amzn

## 前置组件配置

### JPF-core 8

需要使用采用 java8 的 jpf-core

```bash
git clone git@github.com:javapathfinder/jpf-core.git
git checkout java-8
```

构建 jpf-core 

```bash
./gradlew buildJars
```

### jConstraints

JDart 使用 jConstraints 库与求解器进行交互。jConstraints 通过插件支持多种约束求解器。

这里使用 z3 求解器，因此需要安装配置 z3 和 jConstraints-z3，构建 z3 需要 Python, C++ 编译器和 make，这里使用的编译器是 clang。

```bash
git clone git@github.com:psycopaths/jconstraints.git
cd jconstrainsts
mvn install
```

### z3

jConstraints-z3 插件需要 z3 4.4.1

```bash
git clone https://github.com/Z3Prover/z3.git
cd z3
git checkout z3-4.4.1
python scripts/mk_make.py --java
cd build
make all
```

可能会出现构建失败的报错：

```terminal
../src/util/debug.cpp:79:14: error: no viable conversion from 'basic_istream<char, char_traits<char>>' to 'bool'
   79 |         bool ok = (std::cin >> result);
      |              ^    ~~~~~~~~~~~~~~~~~~~~
/usr/bin/../lib/gcc/x86_64-linux-gnu/13/../../../../include/c++/13/bits/basic_ios.h:117:16: note: explicit conversion function is not a candidate
  117 |       explicit operator bool() const
      |                ^
1 error generated.
make: *** [Makefile:126: util/debug.o] Error 1
```

解决方法，将 `src/util/debug.cpp:79` 修改为：

```cpp
bool ok = (std::cin>>result)?true:false;
```

构建成功后，将 `libz3.so` 和  `libz3java.so` java 能够加载的系统库目录

```bash
sudo cp libz3.so /usr/lib/
sudo cp libz3java.so /usr/lib/
sudo ldconfig
```

将 `com.microsoft.z3.jar` 安装到本地 maven 仓库中

```bash
mvn install:install-file -Dfile=com.microsoft.z3.jar -DgroupId=com.microsoft -DartifactId=z3 -Dversion=4.4.1 -Dpackaging=jar
```

### jConstraints-z3

```bash
git clone git@github.com:psycopaths/jconstraints-z3.git
cd jconstraints-z3
mvn install
```

将 `com.microsoft.z3.jar`, `jConstraints-z3-[version].jar` 放到文件夹 `~/.jconstraints/extensions` 中

```bash
mkdir -p ~/.jconstraints/extensions
cp z3/build/com.microsoft.z3.jar ~/.jconstraints/extensions/
cp jconstraints-z3/target/jconstraints-z3-0.9.1-SNAPSHOT.jar ~/.jconstraints/extensions/
```

## JDart 配置

```bash
git clone git@github.com:psycopaths/jdart.git
cd jdart
ant
```

在 `site.properties` ($HOME/.jpf) 文件中写入配置

```bash
midir ~/.jpf
vim ~/.jpf/siteproperties
```

`site.properties` 内容如下：

```properities
jpf-core = /path/to/jpf-core
jpf-jdart = /path/to/jdart
extensions=${jpf-core},${jpf-jdart}
```

可以试运行 JDart 中的 example 来查看是否构建成功：

```bash
$ /path/to/jpf-core/bin/jpf /path/to/jdart/src/examples/features/simple/test_foo.jpf
```

## 参考

- https://github.com/psycopaths/jdart
- https://github.com/psycopaths/jConstraints
- https://github.com/psycopaths/jConstraints-z3
- https://github.com/Z3Prover/z3
