---
abbrlink: ''
categories:
- - 代码审计
date: '2026-04-16T17:24:38.711095+08:00'
tags:
- codeql
title: CodeQL 入门
updated: '2026-04-16T17:24:40.294+08:00'
---
## CodeQL 是什么

这就涉及到代码审计的进化史了，最早期，安全人员会通过人工代码审计的方式来审计项目代码，查找危险函数并检查危险函数是否可控，如果可控，那就存在安全漏洞。后来随着项目越来越大，人工审计的难度越来越高，纯靠人工的方式进行代码审计很难实现所有项目的覆盖测试了，所以出现了一些辅助工具，比如 rips、cobra，通过这些工具可以先把危险函数检索出来，再通过人工来审计是否存在安全漏洞。

上面的方式依然是通过人工的方式来进行代码审查的，工作量还是很大，并且非常依赖网络安全工程师的个人水平，所以就开始出现了一些自动化代码审查工具，比如 Checkmarx、Fortify SCA，但是这些软件都是商业化的，小公司和个人根本付不起这种费用。

与此同时，GitHub 为了解决其托管库海量项目的安全问题，收购了 CodeQL 的创业公司，并且开源了 CodeQL 规则库，对于安全工程师来言，就多了一个非商业化的开源代码自动化审查工具，CodeQL 支持的语言如下：

![image](https://img2.eval.moe/image-20260416093238-0sm1vmg.png)

## 环境信息

在网上找了个靶场 [micro-service-lab](https://github.com/l4yn3/micro_service_seclab/)

参考教程：[CodeQL从入门到放弃](https://www.freebuf.com/articles/web/283795.html "CodeQL从入门到放弃")

本机环境：Windows 11

Java 环境：jdk1.8（我是用 idea 新建项目里下载的 jdk1.8 然后配置环境变量，这种方式好用，强推）

maven 环境：[apache-maven-3.9.14-bin.zip](https://maven.apache.org/download.cgi "apache-maven-3.9.14-bin.zip")

## CodeQL 的安装

这里演示的是在 Windows 11 中 CodeQL 的安装步骤

CodeQL 包含两部分：解析引擎 + SDK，解析引擎不开源，但可以在官网下载，SDK 开源。

解析器：https://github.com/github/codeql-cli-binaries/releases

SDK：https://github.com/github/codeql

新建一个专门的文件夹存放解析器和 SDK，在 C:\\Users\\aaa\\AppData\\Roaming 下新建一个 CodeQL 文件夹

解析器直接下载然后移动并解压就行，SDK 的话直接 git clone

```yaml
git clone https://github.com/github/codeql.git codeql_sdk
```

![image](https://img2.eval.moe/image-20260416114459-b79rt5x.png)

将解析器添加进环境变量

![image](https://img2.eval.moe/image-20260416114615-l4chebc.png)

这时候新开一个 cmd 或 powershell，输入 codeql 命令会有回显

![image](https://img2.eval.moe/image-20260416115609-2wb2bvk.png)

进入 vscode，添加 codeql 扩展，并配置解释器路径

![image](https://img2.eval.moe/image-20260416114755-xny3kle.png)

打开命令行，生成数据库命令

```powershell
codeql database create codeql_test  --language="java"  --command="mvn clean install --file pom.xml -DskipTests -Dmaven.test.skip=true -Drat.skip=true" --source-root=C:\Users\aaa\Documents\micro_service_seclab-main\micro_service_seclab-main

解释一下：
codeql_test 是我们要创建的数据库名
--language="java" 标识当前语言为 Java
--command="mvn clean install --file pom.xml -DskipTests -Dmaven.test.skip=true -Drat.skip=true" 编译命令，因为 Java 是编译语言，所以需要使用该命令对源码进行编译，再进行转换
-DskipTests: 编译测试代码，但不执行测试用例。
-Dmaven.test.skip=true: 更彻底，连测试代码都不编译（通常这个最管用）。
-Drat.skip=true: （可选）跳过 Apache 许可证合规性检查，防止某些开源项目因此报错。
不加上面的三个参数可能会报错，因为编译后默认会运行单元测试，但是这是一个不影响 codeql 结果的步骤，没什么必要还浪费时间，可能还会导致数据库创建失败
--source-root=... 表示这个项目的路径
```

等待数据库成功创建后，在 vscode 导入 codeql 数据库，选择 From a floder，然后选择 codeql 数据库目录，可以导入说明数据库没问题，这一步是为了测试数据库创建是否有问题

![image](https://img2.eval.moe/image-20260416141706-i9a6ynp.png)

然后打开在 CodeQL 的 SDK 文件夹里右键 -> 通过 vscode 打开，然后在 java->ql->example 下创建 demo.ql，输入 ql 语句，右键->Run Query on Selected Database，等待一会儿，就可以看到右边显示结果，这个语句是打印 hello world 的意思

![image](https://img2.eval.moe/image-20260416142521-7oglsx2.png)

## CodeQL 基本语法

codeql 的语法类似于 sql，

我们是 Java 项目，所以要 import java，

from int i 的意思是我们定义一个变量 i，变量 i 表示所有 int 类型数据

where i = 1 表示当 i 的值为 1 的时候，符合条件

select i 表示输出 i

```graphql
import java
 
from int i
where i = 1
select i
```

![image](https://img2.eval.moe/image-20260416144033-but89w2.png)

ql 的语法结构如下：

```pascal
from [datatype] var
where condition(var = something)
select var
```

## CodeQL 类库

codeql 将源码转换为 codeql 可识别的数据库的过程，其实也就是将我们的源码转换为了可识别的 AST 语法库

类库就是 AST 之间的对应关系，也就是说，在 AST 中 Method 表示类当中的方法，我们常用的方法如下：


| 名称         | 解释                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------------- |
| Method       | 方法类，Method method 表示获取当前项目中所有的方法                                                      |
| MethodAccess | 方法调用类，MethodAccess call 表示获取当前所有方法调用，在某个版本之后，MethodAccess 变成 MethodCall 了 |
| Parameter    | 参数类，Parameter 表示获取当前项目所有的参数                                                            |

> Method.getName() 获取的是当前方法的名称
>
> Method.getDeclaringType() 获取的是当前方法所属的类的名称
>
> Method.hashName(var) 显而易见，只显示包含指定名称的方法

结合刚刚所学的，我们获取当前项目所有的方法

```kotlin
import java
 
from Method method
where method.hasName("getStudent")
select method.getName(), method.getDeclaringType()
```

![image](https://img2.eval.moe/image-20260416151704-dz3l57k.png)

## CodeQL 谓词

如果 where 语句过长的话，不仅不美观，还会难以理解，所以 CodeQL 提供一种机制可以将长长的 where 语句转换成函数，这个函数就叫做谓词。

将上面的查询语句转换成谓词：

```graphql
import java
predicate isStudent(Method method){
	exists(|method.hasName("getStudent"))
}
from Method method 
where isStudent(method)
select method
```

语法解释：

* predicate 表示当前方法没有返回值
* exists 子查询，它根据内部的子查询返回 true | false，来决定返回哪些数据

![image](https://img2.eval.moe/image-20260416151926-87ufd9l.png)

## Source、Sink 和 Sanitizer

学静态分析必会的三元组：Source、Sink 和 Sanitizer

Source：可控数据点，比如 http 的传参就是 Source 点；

Sink：危险函数，比如 sql 的查询函数 query 就是一个 Sink 点，它指的是函数的最终执行点；

Sanitizer：过滤函数，又叫净化函数，如果在漏洞链条中，如果某一个方法阻断了整个传递链，那么这个方法就叫做 Sanitizer；

只有当 Source 到 Sink 是通的，我们才认可它是一个漏洞

比如此处的 username 就是一个 Source 点：

![image](https://img2.eval.moe/image-20260416160151-5x1o6wh.png)

我们可以用下面的谓词获取所有的 Source 点，这个是 SDK 的规则，里面包含了大多数的 Source 入口

```ql
import java
import semmle.code.java.dataflow.FlowSources

predicate isSource(DataFlow::Node src) { src instanceof RemoteFlowSource }

from DataFlow::Node src
where isSource(src)
select src, "是一个Source点"
```

这里解释一下部分语句：

* DataFlow::Node 表示一个流，这个流可以是变量、表达式、参数等等
* RemoteFlowSource 是一个 CodeQL 的预定义类，包含了常见的远程输入源
* semmle.code.java.dataflow.FlowSources 数据源流库，定义了什么是“外部输入”

![image](https://img2.eval.moe/image-20260416163231-otta5gl.png)

在这个靶场中，判断 sql 注入最简单的一个方法就是寻找 query 方法，在下面的谓词里，我们就定义了所有给 query 的第一个参数的表达式为 Sink 点。

```ql
predicate isSink(DataFlow::Node sink) {
exists(Method method, MethodCall call |
  method.hasName("query")
  and
  call.getMethod() = method and
  sink.asExpr() = call.getArgument(0)
)
}
```

完整代码：

```ql
import java
import semmle.code.java.dataflow.FlowSources

predicate isSink(DataFlow::Node sink) {
  exists(Method method, MethodCall call |
    method.hasName("query") and
    call.getMethod() = method and
    sink.asExpr() = call.getArgument(0)
  )
}

from DataFlow::Node sink
where isSink(sink)
select sink, "传入 query() 方法的第一个参数是敏感点"
```

## 结语

好了，现在你已经学会怎么用 codeql 对代码进行自动化审计了，你可以去审计 GitHub 的各种开源代码了！
