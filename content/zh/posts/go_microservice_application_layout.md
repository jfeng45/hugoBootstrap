---
title: "清晰架构（Clean Architecture）的Go微服务: 程序结构"
shortTitle: "Go微服务程序结构"
date: "2019-07-21"
draft: false
tags: ["Golang", "Grpc", "清晰架构", "微服务", "程序结构"]
categories: ["Go微服务"]

# canonicalUrl: "https://medium.com/@jfeng45/go-micro-service-with-clean-architecture-application-layout-e6216dbb835a"
description: '“标准Go项目结构”不适合Go微服务gRPC项目。 本文使用一个真正的应用程序来展示什么是好的微服务应用程序结构及其背后的原因。'

---

我使用Go和gRPC创建了一个微服务，并试图找出最佳的程序结构，它可以用作我未来程序的模板。 我有Java背景，并发现自己在Java和Go之间挣扎，它们之间的编程理念完全不同。我写了一系列关于在项目工作中做出的设计决策和取舍的文章。 这是其中的第一篇， 是关于程序结构的。

##### **程序结构的资源**
<br/>

Go的标准程序结构的最佳资源可能是Github上的[标准Go程序结构](https://github.com/golang-standards/project-layout)¹，但它不适合我的项目。在阅读了[Sylvain Wallez的文章](https://bluxte.net/musings/2018/04/10/go-good-bad-ugly/)²之后，我终于得到了一些关于其背后原因的信息。 Go起初是专为API和网络服务器而设计，Github上的大多数Go项目都是以库的形式编写的，因此“标准Go程序结构”可能非常适合。 商业微服务项目是一种完全不同的动物，需要不同的程序结构。但还是我采用了“标准Go程序结构”中适用的一些建议，这些建议约占30％。

一般来说，创建应用程序结构有两种不同的方法：基于业务功能或基于技术结构。[大家的共识](http://www.javapractices.com/topic/TopicAction.do?Id=205)³是基于业务功能的更好，对于单体项目（monolithic project）来说也确实如此。在微服务架构中，情况发生了变化，因为每个服务都有自己的存储库。因此，在每个微服务中，基于技术结构创建项目结构实际上是可行的。

我还在[应用程序结构](https://medium.com/@benbjohnson/structuring-applications-in-go-3b04be4ff091)⁴和[包结构](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)⁵上找到了Ben Johnson的一些好建议。但它没有为我的项目提供完整的解决方案，所以我决定创建自己的程序结构。程序结构取决于项目要求，以下是需求。

##### **项目需求**:
<br/>

1.在数据持久层上支持不同的数据库（Sql和NoSql数据库）

2.使用不同的协议（如gRPC或REST）支持来自其他微服务的数据

3.松散耦合和高度内聚

4.支持简单一致的日志记录，并能够更改它（例如，日志记录级别和日志记录库），而无需修改程序中的日志记录语句。

5.支持业务级别的事物交易。

程序结构也受到程序设计的影响。 我采用了 [Bob Martin的清晰架构（Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)⁶ 和 Go的 [简洁](https://talks.golang.org/2012/splash.article)⁷ 设计风格.

在业务逻辑方面，有三层：“模型（model）”，即域模型; “数据服务（dataservice）”，它是数据持久性（数据库）层; “用例（usecase）”，这是业务逻辑层。

##### **顶层程序结构:**
<br/>

![service tmpl](/images/serviceTmpl.jpg)

**adapter:** 这是应用程序和外部数据服务之间的接口，例如另一个gRPC服务。 所有数据转换都发生在这里，这样您的业务逻辑代码不需要了解外部服务的具体实现（无论是gRPC还是REST）。

**cmd:** 命令。 所有不同类型的“main.go”都在这里，你可以有多个。 这是应用程序的起点。

**config:** 设置程序和配置数据，包括配置文件。

**container:** 应用程序依赖注入容器，负责创建具体类型并将它们注入每个函数。

**dataservice:** 持久层，负责检索和修改域模型的数据。 它只依赖（depend）于模型（model）层。 数据服务可以通过RPC或RESTFul调用从数据库或其他微服务获取数据。

**model**: 域模型层，具有域结构（model struct）。 所有其他层依赖于此层，而此层不依赖于任何其他层。

**script:** 与设置应用程序相关的脚本，例如，数据库脚本。

**tool:** 第三方工具。

**usecase** : 这是一个重要的层并且是业务逻辑的切入点。 每个业务功能都由用例实现。 它是程序顶层，因此没有其他层依赖于它（“cmd”除外），但它依赖于其他层。
              
用例和数据服务层功能全部由接口调用，因此可以轻松更改实现。

##### **顶级包下的子文件包:**
<br/>

**adapter:**

![adapter](/images/adapter.jpg)

当程序需要与微服务或其他外部服务进行交互时，您需要创建接口以减少依赖性。例如，本程序中的“缓存服务”是一个gRPC微服务。每个外部服务都有自己的子包和文件。例如，缓存服务具有“cacheclient”包和“cacheClient.go”文件，该文件定义了与外部“缓存”微服务交互的类型和方法。

在我们的示例中，与其他数据服务不同，“cacheClient.go”文件没有定义缓存服务的接口。实际上它有一个，但是interface是在“dataservice”包中定义的，因为“缓存服务”也是一个数据服务。更明确的方法可能是在两个包中各自创建一个接口，这将保持包结构的统一。但是这两个接口将是相同且冗余的，所以我删除了适配器中的接口。

由于我们还需要将应用程序本身发布为gRPC服务，因此需要在此处创建“userclient”子包。 “generatedclient”子包是为gRPC和Protobuf生成的代码。“userGrpc.go”用来处理域模型结构和gRPC结构之间的格式转换。

**cmd:**

![cmd](/images/cmd.jpg)

应用程序的命令，是整个程序的起点。 您可以将应用程序作为本地应用程序运行，也可以将其作为微服务应用程序运行，在这种情况下，您同时拥有客户端（grpcClientMain.go）和服务器端（grpcServerMain.go）主文件。 所有未来的主文件也将在此处，例如，Web应用程序服务器主文件。

**config:**

![config](/images/config.jpg)

在此保存所有应用配置文件。 “appConfig.go”负责从配置文件中读取并数据将它们加载到应用程序配置结构中。 您可以为不同的环境提供不同的配置文件（YAML文件），例如“Dev”和“Prod”。

**container:**


![container](/images/container.jpg)

本程序中最复杂的包，它为每个接口创建具体结构并将它们注入其他层。此包中的子包结构类似于应用逻辑分层，它还具有“用例（usecase）”，“数据服务（dataservice）”和“数据存储（datastore）”层。

在本包顶层，“container.go”定义容器接口，它的实现文件“serviceContainer.go”在“servicecontainer”子包中。它是此包的入口点，它将此包中的其他代码粘合在一起。 “usecasefactory”子包中的“registrationFactory.go”和其他工厂（factory）使用工厂方法模式（factory method pattern）创建具体结构（struct）。 [日志](https://jfeng45.github.io/posts/go_logging_and_error_handling/)⁸不属于任何应用层，因此我为它创建了一个单独的子包“loggerfactory”。它还有一个“logger”子包来定义日志记录接口。我在文章[程序容器](https://jfeng45.github.io/posts/application_container/)<sup>9</sup>中解释了这个包中的所有内容。

**dataservice:**

![dataservice](/images/dataservice.jpg)

域模型的持久层。 顶层只有一个文件 - “dataService.go”，它具有数据服务的所有接口，包括其他微服务提供的数据服务（例如gRPC）。 其他包只需要依赖于顶级数据服务接口，而不需要了解特定数据库的实现细节。

对于域模型中的每种类型，都有相应的数据服务。 例如，对于模型“User”，有一个“userdata”子包，它包含用户持久性服务的所有实现，包括sqldb（MySql）和CouchDB。

**model:**

![model](/images/model.jpg)

该层不需要接口，模型都是具体结构。

**Script:**

![script](/images/scripts.jpg)

目前它只有MySql的数据库脚本。

**tool:**

![tool](/images/tools.jpg)

此程序包适用于第三方工具。 如果您不想直接依赖第三方库，或者需要增强这些第三方库，请在此处进行封装。 不要与“adapter”包混淆，后者也处理第三方库，但只适用于应用程序级数据服务。 “tool”包更适用于较低级别的库。

**useCase:**

![use case](/images/usecase.jpg)

顶级包下只有一个文件“useCase.go”，它包含所有用例接口。 目前，有三个用例：“RegistrationUseCase”，“ListUserUseCase”和“ListCourse”。

每个用例都有一个子包。 该子包中的文件定义了实现用例接口的具体结构。 所有其他包仅依赖于顶级的用例接口，不需要了解每个用例的实现细节。

##### **可能的问题:**
<br/>

这个程序结构最终会产生很多小的子包，每个子包只有一个或几个文件。 这与Go习惯用法“[考虑更少，更大的包](https://dave.cheney.net/practical-go/presentations/qcon-china.html#_consider_fewer_larger_packages)¹⁰相矛盾. 以下才是习惯用法应创建的包：

![use case alt](/images/usecaseNew.jpg)

如果您为其他人编写一个外部库，那么将代码放入一个大包中是一个很好的规则，因为人们不需要多个import语句来使用您的库。 但是在你自己的应用程序中，拥有小包是可以的，特别是当你只将接口暴露给其他层时。

本程序为什要用小包呢？ 首先“useCase.go”只定义接口，而其他包（容器除外）仅依赖于接口，因此“useCase.go”需要一个独立的包。 其次，用文件夹分隔每个用例使程序更清晰易读。

##### **结论:**

对于gRPC微服务项目，“标准Go程序结构”可能不太适合。 基于业务逻辑而不是技术结构创建应用程序结构对单体项目是一个很好的建议，不一定适合微服务项目。 本文使用一个真实的应用程序作为示例来展示什么是一个很好的微服务应用程序结构及其背后的原因。


##### **源程序:**

完整的源程序链接 [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

##### **其他文章:**

请在这里阅读本系列的其他文章 “[清晰架构（Clean Architecture）的Go微服务](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

##### **索引:**

[1][golang-standards/project-layout]
(https://github.com/golang-standards/project-layout)

[2][Go: the Good, the Bad and the Ugly]
(https://bluxte.net/musings/2018/04/10/go-good-bad-ugly/)

[3][Package by feature, not layer]
(http://www.javapractices.com/topic/TopicAction.do?Id=205)

[4][Structuring Applications in Go]
(https://medium.com/@benbjohnson/structuring-applications-in-go-3b04be4ff091)

[5][Standard Package Layout]
(https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)

[6][The Clean Code Blog]
(https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

[7][Go at Google: Language Design in the Service of Software Engineering]
(https://talks.golang.org/2012/splash.article)

[8][Go Microservice with Clean Architecture: Application Logging]
(https://jfeng45.github.io/posts/go_logging_and_error_handling/)

[9][Go Microservice with Clean Architecture: Application Container]
(https://jfeng45.github.io/posts/application_container/)

[10][Practical Go: Real world advice for writing maintainable Go programs]
(https://dave.cheney.net/practical-go/presentations/qcon-china.html)
