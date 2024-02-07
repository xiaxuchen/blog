---
title: "多线程环境下SQLite解决方案"
summary: 简要介绍SQLite的工作方式，并对SQLite在并发场景下的SQLiteBusy等坑点进行解决
date: 2024-2-8
series: ["SQLite"]
weight: 5
aliases: ["/papermod-installation"]
tags: ["数据库", "SQLite", "Java"]
author: ["夏旭晨"]
# cover:
#   image: images/papermod-cover.png
#   hiddenInList: true
---

# 背景

公司项目是 toB 类型的项目，用户在本机安装进行使用的工具软件。因此外挂 MySQL 对初级用户来讲步骤比较繁琐，因此采用内嵌的 SQLite 数据库进行支持，简化安装流程。但在使用过程中经常会出现 SQLite BUSY 异常，因此对 SQLite 进行了
进一步的探究。

# 前言

SQLite 作为一个比较完善的**基于文件**的**关系型数据库**，不需要额外安装数据库，简单、易用，同时能满足正常读写性能需求，因此在安卓端作为结构化数据存储的标准方案。 而对于小型站点或者内部使用的网站，也能够采用 SQLite 支持正常业务，无需外挂如 MySQL 这种独立的数据库软件。但 SQLite 与 MySQL 在实现上存在着比较大的差异，因此在后端多线程环境下存在许多坑点，**会导致程序异常**，从而对于不那么健壮的程序失去数据的一致性。

# SQLite原理
## 并行度
SQLite作为一个文件数据库，其锁的级别是文件级别的，即一个数据库只有一把锁，因此在进行**并发写**的时候是**串行**的，读读共享，读写互斥。而在开启了WAL(Write Ahead Log)选项后，能够实现**快照读**，写的时候不阻塞读操作，提高了SQLite的读写性能。
## 事务
# 坑点
## 并发开启事务
### SQLite BUSY
### SQLite BUSY SNAPSHOT

# 参考
