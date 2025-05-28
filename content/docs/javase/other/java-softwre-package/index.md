---
title: JDK|JRE|JVM
draft: false
weight: 30
---

# JDK|JRE|JVM

## JVM：

Java 虚拟机，Java 程序能够跨平台运行的核心

- 所有的 Java 程序都会被编译为.class 的类文件，同代码在任何平台上编译字节码都相同
- .class 文件在虚拟机上运行，由虚拟机将字节码解释给本地系统执行

## JRE：

Java 运行时环境，即 Java 程序必须在 JRE 上运行

- 包含 JVM 和 Java 核心类库
- JVM 不能直接执行 class，还需要 Java 核心类库来解释 class
- 安装 jre 后有 bin 和 lib 两个文件夹，可简单理解为分别是 JVM 和 Lib

## JDK：

Java 开发工具包，包括 JRE、Java 工具、编译器和调试器组成  
自带工具：

1. java：Java 运行工具，运行.class 或 jar 包
2. javac: Java 编译工具，将 Java 源代码编译为字节码
3. javap: Java 反编译工具，将 Java 字节码反汇编为源代码
4. jmap：Java 内存映射工具，打印执行 Java 进程、核心文件或远程调试服务器的配置信息
5. jps: Java 进程状态工具，显示目标系统上的 HotSpot JVM 的 Java 进程信息
6. jinfo: Java 配置信息工具，用于打印指定 Java 进程、核心文件或远程调试服务器的配置信息
7. jstack: Java 堆栈跟踪工具，用于打印 Java 进程、核心为念 u 哦远程调试服务器的 Java 现成的堆栈跟踪信息
8. jvisualvm: Java 可视化 JVM 检测、故障分析工具。图形化界面提供指定虚拟机的 Java 应用程序的详细信息
9. jconsole：图形化界面的检测工具，监测并显示 Java 平台上的应用程序的性能和资源占用等信息
10. javadoc: Java 文档工具，根据源代码中的注释信息生成 HTML 格式的 API 帮助文档

## 三者的关系：

- JDK 包含 JRE、JRE 包含 JVM
- JVM 不能单独搞定 class 的执行，解释 class 需要使用 JRE 中的 Java 核心类库 lib
- 我们利用 JDK 开发 Java 源程序，通过 JDK 提供的 javac 编译程序将源程序编译成 Java 字节码，在 JVM 使用 JRE 的 lib 解释这些字节码，映射到 CPU 指令集或 OS 的系统调用
