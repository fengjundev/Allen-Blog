---
title: Android多渠道打包实践
date: 2017-02-20 13:52:46
tags: 
- android
- 多渠道
- 打包
categories: 
- android
- 效率
---

现在的app大多会在内部增加“渠道”这一标识信息，用于进行活跃度统计或者开展有针对性的运营活动。国内渠道成百上千，渠道的概念不只局限于应用市场，还可能是一个推广方式。当渠道越来越多的时候，我们实现多渠道打包的方式也需要不断优化。

本文主要介绍目前主流的几种多渠道打包方式及我们的多渠道打包实现，供读者参考。

# Gradle 















验证：aapt dump --include-meta-data badging /Users/fengjun/Downloads/yizhangtong-opposhop_sd-release.apk
