---
title: Java instrumentation - hello javaagent
date: 2020-09-13 11:25:51
tags: 
  - java instrumentation
---
想直接看[Code](https://github.com/tzuyichao/java-basic/tree/master/instrumentation/hello-javaagent)的話，直接點連結。

這一篇講用<code>-javaagent</code>的方式使用java instrumentation的hello world project。這是個multi module maven project。有兩個module一個是client一個是agent部分。另外，還有一個run.xml的ant script裡面有兩個task用來執行client，一個task有啟動agent一個沒有，可以感覺一下差異。

### Java Instrumentation

Instrumentation讓我們可以在既有（compiled)方法插入額外的bytecode，而達到收集使用中資料來進一步分析使用。instrumentation底層是透過<code>JVM Tool Interface</code>(<code>JVMTI</code>)來達成，JVMTI是JVM暴露出來的extension point。

在Java 5的時候讓我們可以在啟動的時候使用<code>-javaagent</code>的方式，指定agent。在Java 6的時候則更近一步提供了attach的方式，讓我們可以針對已啟動的JVM process執行agent的工作。

- 啟動JVM時啟動代理 （本篇）
- 針對已啟動JVM啟動代理 （下一篇）

### 入口

入口class我們要提供<code>premain()</code>這個方法，需要兩個參數

> ``` java
> public static void premain(String agentArgs, Instrumentation inst)
> ```

第一個參數是啟動agent時，可以由命令列給予agent的參數，第二個則是JVM會提供agent class使用的Instrumentation的入口。

### Instrumentation Interface

### Configuration

### Using ASM for manipulate bytecode

### Cleint

### Execute it!!!

