---
abbrlink: ''
categories:
- - 技巧手法
date: '2026-05-07T18:04:18.383743+08:00'
tags:
- edge
title: Edge 转储明文账密
updated: '2026-05-07T18:04:18.659+08:00'
---
## 简介

起因是刷推看到了这个

![image](https://img2.eval.moe/image-20260507141450-zupo701.png)

## 复现步骤

### Step 1：模拟环境

首先要保证 Edge 的密码本里有信息

![image](https://img2.eval.moe/image-20260507141656-w7e182j.png)

### Step 2：创建内存转储文件

![image](https://img2.eval.moe/image-20260507141922-fsj2mgt.png)

然后你就会得到一个 DMP 结尾的文件

### Step 3：筛选字符串

现在用微软官方的字符串工具筛选出所有字符串，方便查看

下载链接：[Strings v2.54](https://learn.microsoft.com/en-us/sysinternals/downloads/strings)

执行命令：

```perl
.\strings64.exe -accepteula -u .\msedge.DMP > output.txt
```

### Step 4：查找关键词

看到了明文存储的账号密码

![image](https://img2.eval.moe/image-20260507180019-eqlsr07.png)

Edge 里显示的账号密码，一模一样

![image](https://img2.eval.moe/image-20260507175950-glyyovq.png)

## 总结

这算是一个奇淫技巧吧，但是目前发现的问题是这些字符串都是乱序存储的，还没怎么掌握它们的特征，所以到 Step 4 查找关键字的时候会比较麻烦一点
