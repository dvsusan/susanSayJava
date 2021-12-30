## 前言
最近一段时间，我们团队在生产环境出现了几次线上问题，有部分比较严重，直接影响用户功能的使用，惹得领导不高兴了，让我想办法提升代码质量，这时候项目工程代码质量检测神器——SonarQube，出现在我们的视线当中。 

## 一 sonarqube是做什么的
SonarQube®是一种自动代码审查工具，用于检测代码中的错误，漏洞和代码味道。它可以与您现有的工作流程集成，以实现跨项目分支和提取请求的连续代码检查。通过插件形式，可以支持包括 java, C#, C/C++, PL/SQL, Cobol, JavaScrip, Groovy 等二十几种编程语言的代码质量管理与检测。sonarqube可以从以下7个维度检测代码质量，而作为开发人员至少需要处理前5种代码质量问题。

### 1.1 不遵循代码标准
sonarqube可以通过CheckStyle等代码规则检测工具规范代码编写。

### 1.2 存在的缺陷漏洞
sonarqube可以通过Findbugs等等代码规则检测工具检测出潜在的缺陷。

### 1.3  糟糕的复杂度分布
文件、类、方法等，如果复杂度过高将难以改变，这会使得开发人员 难以理解它们, 且如果没有自动化的单元测试，对于程序中的任何组件的改变都将可能导致需要全面的回归测试。

### 1.4  重复
显然程序中包含大量复制粘贴的代码是质量低下的，sonarqube可以展示源码中重复严重的地方。

### 1.5  注释不足或者过多
没有注释将使代码可读性变差，特别是当不可避免地出现人员变动 时，程序的可读性将大幅下降 而过多的注释又会使得开发人员将精力过多地花费在阅读注释上，亦违背初衷。

### 1.6  缺乏单元测试
sonarqube可以很方便地统计并展示单元测试覆盖率。

### 1.7  糟糕的设计
通过sonarqube可以找出循环，展示包与包、类与类之间的相互依赖关系，可以检测自定义的架构规则 通过sonarqube可以管理第三方的jar包，可以利用LCOM4检测单个任务规则的应用情况， 检测耦合。sonarqube可以很方便地统计并展示单元测试覆盖率。


总览：

![](https://pic.imgdb.cn/item/611525015132923bf847c97e.jpg)

在典型的开发过程中：

1. 开发人员在IDE中开发和合并代码（最好使用SonarLint在编辑器中接收即时反馈），然后将其代码签入ALM。

2. 组织的持续集成（CI）工具可以检出，构建和运行单元测试，而集成的SonarQube扫描仪可以分析结果。

3. 扫描程序将结果发布到SonarQube服务器，该服务器通过SonarQube界面，电子邮件，IDE内通知（通过SonarLint）以及对拉取或合并请求的修饰（使用Developer Edition及更高版本时）向开发人员提供反馈。



SonarQube实例包含三个组件：

![](https://pic.imgdb.cn/item/611525285132923bf8482b06.jpg)

1. SonarQube服务器运行以下过程：

- 提供SonarQube用户界面的Web服务器。

- 基于Elasticsearch的搜索服务器。

- 计算引擎负责处理代码分析报告并将其保存在SonarQube数据库中。

2. 该数据库存储以下内容：

- 代码扫描期间生成的代码质量和安全性的度量标准和问题。

- SonarQube实例配置。

3. 在构建或连续集成服务器上运行的一台或多台扫描仪可以分析项目。

## 二 sonarqube如何搭建
官网地址：https://www.sonarqube.org/，选择“文档”菜单

![](https://pic.imgdb.cn/item/611525765132923bf848e681.jpg)

在出现的文档页面中可以选择版本，目前最新的版本是8.5。笔者尝试过三个版本：

8.5：它是目前最新的版本，需要安装JDK11，并且只支持oracle、sqlserver和PostgreSQL数据库

7.9：它是一个长期支持的版本，非常文档，也需要安装JDK11，并且只支持oracle、sqlserver和PostgreSQL数据库 。

7.6：它是一个老版本，只需安装JDK8，支持oracle、sqlserver和PostgreSQL数据库，以及mysql数据库。

刚开始我们为了省事，安装了 7.6的版本，因为mysql数据库我们已经在用了，无需额外安装其他数据库，并且JDK8也在使用，安装成本最小。但是后来发现，如果需要安装汉化版插件，或者mybatis插件，这些插件要求的SonarQube版本必须在7.9以上，并且需要运行在JDK11以上。经过权衡之后，我们决定安装最新版的。

### 2.1 安装JDK11和postgreSQL
   JDK下载地址：https://www.oracle.com/java/technologies/javase-jdk11-downloads.html

   JDK的安装比较简单，我在这里就不过多介绍了，网上有很多教程。

   PostgreSQL它自己号称自己是世界上最先进的开源数据库，具有许多功能，旨在帮助开发人员构建应用程序，管理员来保护数据完整性和构建容错环境，并帮助您管理数据，无论数据集的大小。除了免费和开源之外，PostgreSQL也是高度可扩展的。例如，您可以定义自己的数据类型，构建自定义函数，甚至可以使用不同的编程语言编写代码，而无需重新编译数据库。

   PostgreSQL的安装与使用可以参数：https://www.jianshu.com/p/7d133efccaa4



### 2.2 从zip文件安装sonarqube

SonarQube无法在root基于Unix的系统上运行，因此，如有必要，请为SonarQube创建专用的用户帐户。

$ SONARQUBE-HOME（下面）指的是SonarQube发行版已解压缩的目录的路径。

设置对数据库的访问
编辑$ SONARQUBE-HOME / conf / sonar.properties以配置数据库设置。模板可用于每个受支持的数据库。只需取消注释并配置所需的模板，然后注释掉专用于H2的行：
```java
Example for PostgreSQL
sonar.jdbc.username=sonarqube
sonar.jdbc.password=mypassword
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
```
配置Elasticsearch存储路径
默认情况下，Elasticsearch数据存储在$ SONARQUBE-HOME / data中，但不建议将其用于生产实例。相反，您应该将此数据存储在其他位置，最好是在具有快速I / O的专用卷中。除了保持可接受的性能外，这样做还可以简化SonarQube的升级。

编辑$ SONARQUBE-HOME / conf / sonar.properties以配置以下设置：
```jaa
sonar.path.data=/var/sonarqube/data
sonar.path.temp=/var/sonarqube/temp
```
用于启动SonarQube的用户必须具有对这些目录的读写权限。

启动Web服务器
默认端口为“ 9000”，上下文路径为“ /”。这些值可以在$ SONARQUBE-HOME / conf / sonar.properties中进行更改：
```java
sonar.web.host=192.0.0.1
sonar.web.port=80
sonar.web.context=/sonarqube
```
执行以下脚本来启动服务器：

- 在Linux上：bin / linux-x86-64 / sonar.sh start
- 在macOS上：bin / macosx-universal-64 / sonar.sh start
- 在Windows上：bin / windows-x86-64 / StartSonar.bat

调整Java安装
如果服务器上安装了多个Java版本，则可能需要明确定义使用哪个Java版本。

要更改SonarQube使用的Java JVM，请编辑$ SONARQUBE-HOME / conf / wrapper.conf并更新以下行：
```java
wrapper.java.command=/path/to/my/jdk/bin/java
```
您现在可以在http：// localhost：9000上浏览SonarQube （默认的系统管理员凭据为admin/ admin）。第一次访问这个地址比较会停留在这个页面一段时间，因为SonarQube会做一些初始化工作，包含往空数据库中建表
![](https://pic.imgdb.cn/item/6115261e5132923bf84a8b7a.jpg)

初始化成功后运行的页面：

![](https://pic.imgdb.cn/item/6115262d5132923bf84aac2b.jpg)

同时会生成20多张表：

![](https://pic.imgdb.cn/item/6115263a5132923bf84acb52.jpg)

### 2.3 安装插件
根据个人需要，可以安装汉化插件，sonarqube默认是英文界面。

github地址：https://github.com/SonarQubeCommunity/sonar-l10n-zh

将项目下载编译打包后，将jar放到$SONARQUBE-HOME\extensions\plugins

目录下即可，然后执行：./sonar.sh restart命令重启sonarqube服务。

此外，还有mybatis插件

gitee地址：https://gitee.com/mirrors/sonar-mybatis

我个人用过，觉得作用不大，不过可以基于这个代码扩展自己需要的功能。

## 三 sonarqube如何使用
### 3.1 在maven项目中集成sonarqube
先在maven的settings.xml文件中增加如下配置：
```java
<pluginGroups>
    <pluginGroup>org.sonarsource.scanner.maven</pluginGroup>
</pluginGroups>
<profiles>
    <profile>
      <id>sonar</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <!-- Optional URL to server. Default value is http://localhost:9000 -->
        <sonar.host.url>
          http://localhost:9000
        </sonar.host.url>
      </properties>
    </profile>
</profiles>
```
然后在pom.xml文件中增加配置：
```java
<plugin>
  <groupId>org.sonarsource.scanner.maven</groupId>
  <artifactId>sonar-maven-plugin</artifactId>
  <version>3.3.0.603</version>
</plugin>
```
在项目目录下运行代码检测命令：  
```java
mvn clean complie -U -Dmaven.test.skip=true sonar:sonar
```

看到这几句话，就表示检测成功了

![](https://pic.imgdb.cn/item/611526815132923bf84b708c.jpg)

然后在sonar后台查看检测报告

![](https://pic.imgdb.cn/item/611526915132923bf84b9a8e.jpg)

报告里面包含：bug、漏洞、异味、安全热点、覆盖、重复率等，对有问题的代码能够快速定位。

点击某个bug可以查看具体有问题代码：

没有关闭输入流问题：

![](https://pic.imgdb.cn/item/611526b65132923bf84bf736.jpg)

空指针问题：

![](https://pic.imgdb.cn/item/611526c35132923bf84c1750.jpg)

错误的用法：

![](https://pic.imgdb.cn/item/611526d05132923bf84c35b5.jpg)

SimpleDateFormat不应该被定义成static的。

检测出的代码问题类型太多，这里就不一一列举了。总之，记住一句话：sonar很牛逼。它不光可以检测出代码问题，还对一些不好的代码写法和用法有更好的建议。

## 彩蛋
sonarqube非常强大，上面只介绍了它的基本用法。一般情况下，我们可以使用jenkins配置需要代码检测的项目，从gitlab上下载代码，执行maven编译打包代码测试命令，可直接生成报告。jenkins触发执行代码检测的时机是：1.有代码提交，或者指定比如test分支有代码提交，项目数量少可以这样做。2.定时执行，我们公司就是配置在凌晨定时执行，因为jenkins部署的项目太多了，为了不影响正常的项目部署。

此外，我们可以自定义代码检测的执行规则，根据实际的项目需求自己开发插件，比如：我们自己开发了mybatis插件，扫描mapper和xml文件名称不一致的情况。

![](https://pic.imgdb.cn/item/611526e65132923bf84c6c03.jpg)


> 总之，sonar的功能非常强大，强烈建议大家在项目中使用，真的可以减少很多隐藏的bug，提高代码质量，如果你用过就会发现它的好处。如果想了解更多sonar的用法，可以在公众号中回复：sonar，可以获取更详细的用法。