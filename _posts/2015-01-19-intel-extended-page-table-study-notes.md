---
layout: post
title = "Intel EPT心得笔记"
date = "2015-01-19T01:11:13-05:00"
categories: blog
tags: [总结,FreeBSD,X86,Virtualization]
description: 结合bhyve的源代码分析了Intel EPT的实现。
---

### 缘起

最近对x86的虚拟化技术比较感兴趣，想对此进行一些更深入的认识。个人认为FreeBSD的bhyve是个很精简的实现，内核相关的代码主要集中在src/sys/amd64/vmm目录下，所有的代码加起来也就两三万行。于是结合Intel 的x86 software developer's guide volume 3C,对涉及到的技术文档匆匆翻了一遍。只有EPT这部分第一遍翻阅的时候没有理解，第二遍深入阅读的时候也有些云里雾里。

后来google了一些资料和文章，发觉对这部分解释最详细的还是Intel的一篇官方文档。个人觉得其他分析文章似乎对其中一些关键点没有明确说明，即使看了文章也很难弄明白EPT是如何实现的。为此我在这里结合Intel的这篇官方文档中提供的两个图，通过和amd64架构下非EPT时MMU的工作原理进行对比，对EPT涉及到的原理和为什么这样设计尝试进行分析。


### amd64下非EPT时MMU的工作原理

### amd64下EPT时MMU的工作原理

### 为什么需要EPTP

### 和EPT同理的directIO技术
