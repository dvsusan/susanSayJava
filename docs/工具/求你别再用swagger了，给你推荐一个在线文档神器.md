## 前言
最近公司打算做一个openapi开放平台，让我找一款好用的在线文档生成工具，具体要求如下：

1. 必须是开源的
2. 能够实时生成在线文档
3. 支持全文搜索
4. 支持在线调试功能
5. 界面优美

说实话，这个需求看起来简单，但是实际上一点的都不简单。

我花了几天时间到处百度，谷歌，技术博客 和 论坛查资料，先后调研了如下文档生成工具：

## gitbook
github地址：https://github.com/GitbookIO/gitbook

开源协议：Apache-2.0 License

Star: 22.9k

开发语言：javascript

用户：50万+

推荐指数：★★★

示例地址：https://www.servicemesher.com/envoy/intro/arch_overview/dynamic_configuration.html

![](https://pic.imgdb.cn/item/610e9d6b5132923bf8fa0ae9.jpg)
gitBook是一款文档编辑工具。它的功能类似金山WPS中的word或者微软office中的word的文档编辑工具。它可以用来写文档、建表格、插图片、生成pdf。当然，以上的功能WPS、office可能做得更好，但是，gitBook还有更最强大的功能：它可以用文档建立一个网站，让更多人了解你写的书。

另外，最最核心的是，他支持Git，也就意味着，它是一个分布式的文档编辑工具。你可以随时随地来编写你的文档，也可以多人共同编写文档，哪怕多人编写同一页文档，它也能记录每个人的内容，然后告诉你他们之间的区别，也能记录你的每一次改动，你可以查看每一次的书写记录和变化，哪怕你将文档都删除了，它也能找回来！这就是它继承git后的厉害之处！

优点：使用起来非常简单，支持全文搜索，可以跟git完美集成，对代码无任何嵌入性，支持markdown格式的文档编写。

缺点：需要单独维护一个文档项目，如果接口修改了，需要手动去修改这个文档项目，不然可能会出现接口和文档不一致的情况。并且，不支持在线调试功能。

个人建议：如果对外的接口比较少，或者编写之后不会经常变动可以用这个。

## smartdoc
gitee地址：https://gitee.com/smart-doc-team/smart-doc

开源协议：Apache-2.0 License

Star: 758

开发语言：html、javascript

用户：小米、科大讯飞、1加

推荐指数：★★★★

示例地址：https://gitee.com/smart-doc-team/smart-doc/wikis/文档效果图?sort_id=1652819

![](https://pic.imgdb.cn/item/610e9da65132923bf8fa63f6.jpg)

smart-doc是一个java restful api文档生成工具，smart-doc颠覆了传统类似swagger这种大量采用注解侵入来生成文档的实现方法。smart-doc完全基于接口源码分析来生成接口文档，完全做到零注解侵入，只需要按照java标准注释的写就能得到一个标准的markdown接口文档。

优点：基于接口源码分析生成接口文档，零注解侵入，支持html、pdf、markdown格式的文件导出。

缺点：需要引入额外的jar包，不支持在线调试

个人建议：如果实时生成文档，但是又不想打一些额外的注解，比如：使用swagger时需要打上@Api、@ApiModel等注解，就可以使用这个。

## redoc
github地址：https://github.com/Redocly/redoc

开源协议：MIT License

Star: 10.7K

开发语言：typescript、javascript

用户：docker、redocly

推荐指数：★★★☆

示例地址：https://docs.docker.com/engine/api/v1.40/

![](https://pic.imgdb.cn/item/610e9dd35132923bf8faa4a3.jpg)

redoc自己号称是一个最好的在线文档工具。它支持swagger接口数据，提供了多种生成文档的方式，非常容易部署。使用redoc-cli能够将您的文档捆绑到零依赖的 HTML文件中，响应式三面板设计，具有菜单/滚动同步。

优点：非常方便生成文档，三面板设计

缺点：不支持中文搜索，分为：普通版本 和 付费版本，普通版本不支持在线调试。另外UI交互个人感觉不适合国内大多数程序员的操作习惯。

个人建议：如果想快速搭建一个基于swagger的文档，并且不要求在线调试功能，可以使用这个。

## knife4j
gitee地址：https://gitee.com/xiaoym/knife4j

开源协议：Apache-2.0 License

Star: 3k

开发语言：java、javascript

用户：未知

推荐指数：★★★★

示例地址：http://swagger-bootstrap-ui.xiaominfo.com/doc.html

![](https://pic.imgdb.cn/item/610e9df45132923bf8fada62.jpg)

knife4j是为Java MVC框架集成Swagger生成Api文档的增强解决方案,前身是swagger-bootstrap-ui,取名kni4j是希望她能像一把匕首一样小巧,轻量,并且功能强悍。

优点：基于swagger生成实时在线文档，支持在线调试，全局参数、国际化、访问权限控制等，功能非常强大。

缺点：界面有一点点丑，需要依赖额外的jar包

个人建议：如果公司对ui要求不太高，可以使用这个文档生成工具，比较功能还是比较强大的。

## yapi
github地址：https://github.com/YMFE/yapi

开源协议：Apache-2.0 License

Star: 17.8k

开发语言：javascript

用户：腾讯、阿里、百度、京东等大厂

推荐指数：★★★★★

示例地址：http://swagger-bootstrap-ui.xiaominfo.com/doc.html

![](https://pic.imgdb.cn/item/610e9e225132923bf8fb2729.jpg)

yapi是去哪儿前端团队自主研发并开源的，主要支持以下功能：

- 可视化接口管理
- 数据mock
- 自动化接口测试
- 数据导入（包括swagger、har、postman、json、命令行）
- 权限管理
- 支持本地化部署
- 支持插件
- 支持二次开发

优点：功能非常强大，支持权限管理、在线调试、接口自动化测试、插件开发等，BAT等大厂等在使用，说明功能很好。

缺点：在线调试功能需要安装插件，用户体检稍微有点不好，主要是为了解决跨域问题，可能有安全性问题。不过要解决这个问题，可以自己实现一个插件，应该不难。

个人建议：如果不考虑插件安全的安全性问题，这个在线文档工具还是非常好用的，可以说是一个神器，笔者在这里强烈推荐一下。

## apidoc
github地址：https://github.com/apidoc/apidoc

开源协议：MIT License

Star: 8.7k

开发语言：javascript

用户：未知

推荐指数：★★★★☆

示例地址：https://apidocjs.com/example/#api-User

![](https://pic.imgdb.cn/item/610e9e505132923bf8fb6c0f.jpg)

apidoc 是一个简单的 RESTful API 文档生成工具，它从代码注释中提取特定格式的内容生成文档。支持诸如 Go、Java、C++、Rust 等大部分开发语言，具体可使用 apidoc lang 命令行查看所有的支持列表。

apidoc 拥有以下特点：

- 跨平台，linux、windows、macOS 等都支持；
- 支持语言广泛，即使是不支持，也很方便扩展；
- 支持多个不同语言的多个项目生成一份文档；
- 输出模板可自定义；
- 根据文档生成 mock 数据；

优点：基于代码注释生成在线文档，对代码的嵌入性比较小，支持多种语言，跨平台，也可自定义模板。支持搜索和在线调试功能。

缺点：需要在注释中增加指定注解，如果代码参数或类型有修改，需要同步修改注解相关内容，有一定的维护工作量。

个人建议：这种在线文档生成工具提供了另外一种思路，swagger是在代码中加注解，而apidoc是在注解中加数据，代码嵌入性更小，推荐使用。



## showdoc
github地址：https://github.com/star7th/showdoc

开源协议：Apache Licence

Star: 8.1k

开发语言：javascript、php

用户：超过10000+互联网团队正在使用

推荐指数：★★★★☆

示例地址：https://www.showdoc.com.cn/demo?page_id=9

![](https://pic.imgdb.cn/item/610e9e7a5132923bf8fbae56.jpg)

ShowDoc就是一个非常适合IT团队的在线文档分享工具，它可以加快团队之间沟通的效率。

它都有些什么功能：

1. 响应式网页设计，可将项目文档分享到电脑或移动设备查看。同时也可以将项目导出成word文件，以便离线浏览。
2. 权限管理，ShowDoc上的项目有公开项目和私密项目两种。公开项目可供任何登录与非登录的用户访问，而私密项目则需要输入密码验证访问。密码由项目创建者设置。
3. ShowDoc采用markdown编辑器，点击编辑器上方的按钮可方便地插入API接口模板和数据字典模板。
4. ShowDoc为页面提供历史版本功能，你可以方便地把页面恢复到之前的版本。
5. 支持文件导入，文件可以是postman的json文件、swagger的json文件、showdoc的markdown压缩包，系统会自动识别文件类型。

优点：支持项目权限管理，多种格式文件导入，全文搜索等功能，使用起来还是非常方便的。并且既支持部署自己的服务器，也支持在线托管两种方式。

缺点：不支持在线调试功能

个人建议：如果不要求在线调试功能，这个在线文档工具值得使用。