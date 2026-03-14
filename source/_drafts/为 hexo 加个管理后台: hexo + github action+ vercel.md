---
abbrlink: ''
categories: []
date: '2026-03-14T20:33:37.257006+08:00'
tags: []
title: '为 hexo 加个管理后台: hexo + github action+ vercel'
updated: '2026-03-14T20:33:37.877+08:00'
---
写博客的时候发现总是要自己 hexo c g d 三件套，总是觉得有点麻烦，刚好发现 Qexo 终于更新了，很久之前就用过这个框架当 hexo 的后台，但是那时候优化还不够好，不算很好用，刚好现在有这个需求，尝试一下新版，却发现回不去了......

## 简介

先说一下我当前写博客的步骤：

1. 写一篇文章
2. 找到博客目录
3. 将文章复制到博客的 \_post 目录
4. hexo g -f -d

这一套流程看着就觉得有点繁琐，首先我肯定不会随时有自己的电脑，其次是每次都要打开博客目录

部署了 Qexo 和 github action 之后，我写博客的步骤会变成这样

1. 写一篇文章
2. 打开 Qexo，新建文章，将文章内容复制到 Qexo
3. 保存，发布

有了 Qexo 之后就不用每次都要打开博客目录了，在公司摸鱼的时候也可以写；部署了 github action 后，每次我们在 Qexo 保存好文章，点击发布之后，github action 就会自动触发部署——类似于自动在我们博客目录执行 hexo g -f -d

怎么样，是不是觉得没那么繁琐了

> 项目地址：https://github.com/Qexo/Qexo
>
> 项目文档：https://oplog.cn/qexo/start.html

在此非常感谢作者 [am-abudu](https://github.com/am-abudu) 开源这么好用的项目

## 开始部署

我们的部署的步骤大概是：

1. 新建 rep，将博客根目录 push 到这个 rep
2. 按照 [Qexo 教程](https://oplog.cn/qexo/start/build.html)部署项目
3. 编写 github action yml 文件
4. 大功告成

第一步我就不做了，我这从第二步开始

## 部署 Qexo

我是用 Vercel 部署 (MongoDB) 的方式部署的，MongoDB 数据库我已建好，大概就是这么一串字符串

```undefined
mongodb+srv://username:password@cluster0.***.mongodb.net/?appName=***
```

1. 点击 Qexo 教程的一键部署

![image](https://img2.eval.moe/image-20260314190044-oc2kg5x.png)

2. 给项目起个名

![image](https://img2.eval.moe/image-20260314190333-4vgsx7r.png)

3. 点击 Create 一键部署，显示部署失败是正常的，我们要修改一下环境变量，在项目的 Settings->Environment Variables 这里，添加所需环境变量

![image](https://img2.eval.moe/image-20260314190727-u44j81s.png)

4. 返回项目界面，点击 Redeploy

![image](https://img2.eval.moe/image-20260314190900-969rwja.png)

5. Redeploy 完成后进入管理面板，Qexo 的部署就不再多说了，可以跟着文档完成部署

## 部署 github action

1. 在博客源代码仓库点击 GitHub action

![image](https://img2.eval.moe/image-20260314191716-ffz86hf.png)

2. 新建一个 workflow

![image](https://img2.eval.moe/image-20260314191809-8j8wh1u.png)

3. 把我的这个 workflow 复制进去

```undefined
name: 自动部署

on:
  push:
    branches:
      - main


jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
        deploy_key: ${{ secrets.DEPLOYKEY }}
    steps:
    - name: 检查分支
      uses: actions/checkout@v2
      with:
        ref: main

    - name: Setup SSH
      uses: webfactory/ssh-agent@v0.9.0
      with:
        ssh-private-key: ${{ secrets.DEPLOYKEY }}

    - name: 安装 Node
      uses: actions/setup-node@v1
      with:
        node-version: "24.14"

    - name: 安装 Hexo
      run: |
        export TZ='Asia/Shanghai'
        npm install hexo-cli -g

    - name: 缓存 Hexo
      uses: actions/cache@v4
      id: cache
      with:
        path: node_modules
        key: ${{runner.OS}}-${{hashFiles('**/package-lock.json')}}

    - name: 安装依赖
      if: steps.cache.outputs.cache-hit != 'true'
      run: |
        npm install --save

    - name: Configure Git
      run: |
        git config --global user.email "${{secrets.USERNAME}}"
        git config --global user.name "${{secrets.EMAIL}}"
        git config --global init.defaultBranch ${{secrets.BRANCH}}
 
    - name: 部署
      run: |
        hexo g -f -d 

```

4. 重点来了，在自己电脑生成 ssh 密钥对

![image](https://img2.eval.moe/image-20260314194930-50pxb71.png)

```undefined
-t rsa	密钥类型	使用 RSA 算法（最通用的类型）
-m pem	输出格式	以 PEM 格式保存密钥（兼容性最好）
-b 4096	密钥长度	4096 位（安全性高，比默认的 2048 更强）
-C "..."	注释	通常是邮箱，用来标识这个密钥的用途
```

5. 将公钥上传到你博客静态资源的那个 Repository，公钥就是上一步密钥对生成完了之后 \*.pub 后缀的文件的内容，另一个默认叫 id\_rsa 的文件内容我们也复制下来，备用

![image](https://img2.eval.moe/image-20260314195215-zt0n4b6.png)

6. 回到博客源代码仓库，填写这四个 secrets

![image](https://img2.eval.moe/image-20260314201028-zyw7nxx.png)

* BRANCH：静态资源那个仓库的分支，我这里是 main，填 main
* DEPLOYKEY：填写第四步生成的密钥对里的私钥，默认是叫 id\_rsa 的
* EMAIL：你的邮箱，可以填你当前登录的 GitHub 账号绑定的邮箱
* USERNAME：你的用户名，可以填你当前登录的 GitHub 账号的用户名

7. 可以尝试去 Qexo 后台新建一篇文章，然后保存并发布，发布之后 github action 这里会有自动部署的记录

![image](https://img2.eval.moe/image-20260314201440-yonvd5u.png)

8. 自动部署完成后我们可以看静态资源的那个仓库，如果有更新的话，就代表安装完成了

![image](https://img2.eval.moe/image-20260314202340-437pbin.png)

## 拓展：解决给 Qexo 加域名，访问 400 问题

给 Qexo 加完域名之后会报 400 错误，是因为这个域名不在安全域名内，加个环境变量 ALLOWED\_HOSTS ，然后 Redeploy 一下就可以解决啦

![image](https://img2.eval.moe/image-20260314202848-zgxdh04.png)
