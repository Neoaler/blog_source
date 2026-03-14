---
title: 图床方案：Cloudflare + WebP Cloud + Picgo
tags: 图床
categories:
  - - 博客配置
  - - 图床
date: 2026-02-14 23:19:32
---
## 前言

这段时间一直冥思苦想，图床方案到底用哪个比较好，我心中的选择只有这几样

|               | 优点                   | 缺点                       |
| ------------- | -------------------- | ------------------------ |
| 阿里云 oss       | 国内访问速度快，体验非常不错       | 公网访问太贵                   |
| 七牛云           | 国内访问速度快，有 10G 免费空间额度 | 需要备案域名                   |
| cloudflare R2 | 10G 免费空间，还有免费流量额度    | 国内访问速度非常慢，得通过中转或其他方式提高速度 |

## Cloudflare R2

R2 是 Cloudflare 推出的免费对象存储服务，需要免费注册一个  [Cloudflare 账号](https://sspai.com/link?target=https%3A%2F%2Fwww.cloudflare.com%2Fzh-cn%2F)才能使用，注册登录后，点击左侧边栏的 R2 访问服务，但需要注意的是开通 R2 服务需要绑定信用卡（国内外主流信用卡皆可），但并不会扣费，主要是为了验证用户身份使用。

1. 创建一个 R2 存储桶，因为后续需要对接 WebP Cloud，所以选的区域是美西
   ![file-20260214130530709](https://img2.eval.moe/file-20260214130530709.jpg)
2. 启用公共开发 URL
   ![file-20260214232348936](https://img2.eval.moe/file-20260214232348936.jpg)
3. 创建 R2 API
   ![file-20260214131616086](https://img2.eval.moe/file-20260214131616086.jpg)
   ![file-20260214131627650](https://img2.eval.moe/file-20260214131627650.jpg)

## Picgo

1. 下载 s3 插件
   ![file-20260214132340791](https://img2.eval.moe/file-20260214132340791.jpg)
2. 配置 amz s3,有几个要比较注意的点：

- **应用密钥 ID**，填写 R2 API 中的 Access Key ID（访问密钥 ID）
- **应用密钥**，填写 R2 API 中的 Secret Access Key（机密访问密钥）
- **桶名**，填写 R2 中创建的 Bucket 名称，如我上文的  img
- **文件路径**，上传到 R2 中的文件路径，我选择使用  `{fileName}.{extName}`  来保留原文件的文件名和扩展名。
- **自定义节点**，填写 R2 API 中的「为 S3 客户端使用管辖权地特定的终结点」，即  `xxx.r2.cloudflarestorage.com`  格式的 S3 Endpoint
- **自定义输出 URL 模板**，填写上文生成的  `xxx.r2.dev/{fileName}`  格式的域名或自定义域名，如我配置的  `img.eval.moe/{fileName}`
  ![file-20260214132957479](https://img2.eval.moe/file-20260214132957479.jpg)

## WebP Cloud

完成上述步骤之后，我们的图床搭建基本完成，但俗话说 cloudflare 之下，众生平等，所以在 cloudflare 的基础上，我打算再加上 WebP Cloud。
WebP Cloud 简单来说就是一个类 CDN 的图片代理服务，可以在几乎不改变画质的情况下，压缩图片体积，加快整站加载速度，除此之外，还提供了缓存，图片水印，自定义 header 等功能，普通用户的免费额度对于我这种小博客来说非常够用。
登陆后是这样的
![file-20260214130051520](https://img2.eval.moe/file-20260214130051520.jpg)

1. 创建代理，关于代理模式有什么区别，具体可以看这篇文章：[WebP Cloud 加入模式选择——Rapid（极速模式） 和 Consistency（一致模式）](https://blog.webp.se/rapid-consistency-mode-zh/)，总的来说，如果你想把 WebP Cloud 当作一个 CDN 用，就用后者；如果是想当作图片转换工具使用，就用前者，这里我选择快速模式，它比较满足我的需求。
   ![file-20260214162243299](https://img2.eval.moe/file-20260214162243299.jpg) 2.最后就是用 picgo 上传测试了，我上传了一张 5.8m 的图片，通过 WebP Cloud 压缩后只有 288kb，这个压缩还是可以的
   ![file-20260214193833231](https://img2.eval.moe/file-20260214193833231.jpg)
   ![file-20260214193909154](https://img2.eval.moe/file-20260214193909154.jpg)

## 写在最后

虽然这套组合性价比拉满，几乎不花什么钱，且一些都部署在云上，没什么很大的安全问题，但是至于稳定性还得有待观察，但是应该不是很怕的，图片都存储在 R2 上，就算 WebP Cloud 出了问题，也影响不了图片本身。
