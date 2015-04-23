---
layout: post
title: golang与ide使用过程中遇到的问题……
categories: [golang]
tags: [golang,记录,intellij]
---

* ide版本：14.1 golang插件版本：0.9.15.3 问题：import "gopkg.in/tomb.v1" 包后，无法使用tomb关键字 解决： 给import加别名：tomb "gopkg.in/tomb.v1"

好像已经快要出1.0版本了，静等