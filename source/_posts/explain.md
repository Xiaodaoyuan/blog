---
title: Mysql explain 详解
date: 2020-01-03 18:02:33
categories: Mysql
tags: mysql
---

> explain可以提供Mysql执行语句的信息，根据这些信息，我们可以对执行语句进行优化，比如调整索引和连接顺序。explain可以用于select，insert，update，delete，replace语句。

### explain输出项
**1. id列**
