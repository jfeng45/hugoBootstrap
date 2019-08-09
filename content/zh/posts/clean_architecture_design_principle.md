---
title: "清晰架构（Clean Architecture）的Go微服务: 设计原则"
shortTitle: "设计原则"
date: "2019-07-22"
draft: false
tags: ["Golang", "清晰架构"]
categories: ["Go微服务"]

canonicalUrl: "https://medium.com/@jfeng45/go-micro-service-with-clean-architecture-design-principle-118d2eeef1e6"
description: "我使用Spring的基于接口的编程和依赖注入（Dependency Injection）来实现Bob Martin的清晰架构（Clean Architecture），并遵循了Go的简单编程风格。"

---

##### **设计灵感:**
<br/>

我最近写了一个Go微服务应用程序，这个程序的设计来自三个灵感：

* [清晰架构"Clean Architecture"](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)¹ and [SOLID (面向对象设计)](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)² 设计 [Go SOLID](https://dave.cheney.net/2016/08/20/solid-go-design)³

* [Spring的容器技术（Spring’s application context）](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans)⁴

* [Go的简洁设计](https://talks.golang.org/2012/splash.article)⁵ 特别是 [Go的面向对象的设计](https://spf13.com/post/is-go-object-oriented)⁶

我使用Spring的基于接口的编程和依赖注入（Dependency Injection）来实现Bob Martin的清晰架构（Clean Architecture），并遵循了Go的简单编程风格。当它们之间存在冲突时，进行了取舍。我只采用了Clean Architecture的设计原则（主要是SOLID），因此实现的细节可能与其他SOLID实现不同。

我来自Java背景，对前两个设计思想非常熟悉。在学习了Go之后，我逐渐认同了Go的简单风格。粗略来说，有两种不同的编程风格，一种是面向对象的， 它强调设计;另一种是非面向对象的，它信奉用最简单的代码来实现用户需要的功能，无需预先设计。 Go更接近第二阵营，尽管它有一些面向对象的功能。 Go的编程思路为我提供了一个重新评估面向对象编程的新视角，并影响了我的编码风格。结果是我只在必要时才进行面向对象的设计，而我更倾向于使用更简单的解决方案而不是完美的方案。

##### **设计原则:**
<br/>

1. [基于接口编程（Programming on interface）](https://en.wikipedia.org/wiki/Interface-based_programming)⁷

     本程序有三个主要业务层，用例（usecase），数据服务（dataservice）和域模型（model），其中只有域模型没有接口，因为没有必要。 当您访问外部服务时，您可以通过接口进行访问。

    {{< highlight go "linenos=table,linenostart=1" >}}
    // sqlUserDataServiceFactory is a empty receiver for Build method
    type sqlUserDataServiceFactory struct{}
    
    func (sudsf *sqlUserDataServiceFactory) Build(c container.Container, dataConfig *config.DataConfig)
        (dataservice.UserDataInterface, error) {
    
        dsc := dataConfig.DataStoreConfig
        dsi, err := datastorefactory.GetDataStoreFb(dsc.Code).Build(c, &dsc)
        if err != nil {
            return nil, errors.Wrap(err, "")
        }
        ds := dsi.(gdbc.SqlGdbc)
        uds := sqldb.UserDataSql{DB: ds}
        logger.Log.Debug("uds:", uds.DB)
        return &uds, nil
    
    }
    {{< / highlight >}}
    
    <br/>
    基于接口的编程的关键是将接口作为参数传递给函数，并返回接口而不是具体类型。 例如，在上面的代码中，返回值-“dataservice.UserDataInterface”，它是一个接口，而不是struct。 调用函数不需要知道返回的具体结构，因为接口封装了它需要的所有信息。 这使您可以非常灵活地将返回的结构替换为另一个结构，而不会影响调用函数。

2. 用工厂方法模式（factory method pattern）通过依赖注入（Dependency Injection）创建具体类型.

     程序容器负责创建具体类型并将其注入函数。 我将在 “[依赖注入（Dependency Injection）](https://jfeng45.github.io/posts/dependency_injection/)”⁸中进行详细解释.

3. 建立正确的依赖关系

      它意味着以下内容:<br/>
      + 程序中的各层或组件都有自己的单独的包。 接口在顶级包中定义，具体类型隐藏在子包中。
      + 不同层之间仅依赖于接口而不依赖于具体类型
      + 从顶层向下的依赖层次是：“用例”，“数据服务”和“模型”。
      
     <br/>
  
    衡量依赖关系质量的一种方法是看导入（import）语句的多少，导入语句越少，依赖关系越好。
  
4. [开闭原则（Open-close principle）](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)⁹

     这是我最喜欢的设计原则。 它要求你在需要添加新功能时，不要修改现有代码，而是添加新代码。 实现它的方法是使用上面讲到的＃1和＃2。 这个原则有许多很好的现实世界的例子，例如，[数据访问对象（DAO）](https://en.wikipedia.org/wiki/Data_access_object)¹⁰。 好处是你不会无意中搞乱现有代码，因为只添加新代码，这将大大减少测试工作量。
   
##### **是否过度设计了?**
<br/>

与Java中的类似解决方案相比，由于Go的语言本身的简单设计，本程序中的代码量要少很多，也非常简洁。 但是对于来自其他编程语言（特别是动态语言如PHP，Ruby）的人来说，这个程序的设计可能有些重。 我也问了自己同样的问题。 为了得到答案，需要比较成本和收益以得出最终结论。

通常来说有两种类型的需求变更，业务逻辑变更和技术方案变更。 在编写业务代码时，您不希望关注数据是来自MongoDB还是MySQL还是微服务。 在进行技术修改时，最大的噩梦是意外破坏业务逻辑。 一个好的设计将这两种类型的编码在程序中分开，让你一次只关注一个。

一般来说，技术方案变更不会像业务逻辑变化那样频繁发生，但随着微服务的普及，新技术将被更快地采用，这将加速技术变更。

##### **设计带来的好处:**
<br/>

以下是几个示例，向您展示当需求变更时需要对程序进行的改动。 如果您看不太懂本节，可能需要先阅读“[程序设计](https://jfeng45.github.io/posts/clean_architecture_application_design/)¹¹，它将为您提供程序结构的描述。

##### **从MySQL改成MongoDB:**
<br/>

首先，假设我们需要将域模型“User”的持久层从MySQL更改为MongoDB。以下是步骤：

1. 在“appConfig [type] .yaml”文件中添加MongoDB的新配置信息

1. 将“appConfig [type] .yaml”文件中“useCaseConfig”部分下的“userConfig”值更改为指向MongoDB而不是MySql

1. 在“appConfig.go”中为MongoDB创建一个新的结构类型

1. 在“configValidator.go”中为MongoDB添加一个新常量并创建校验规则。

1. 在“datastorefactory”包中创建一个新的MongoDB工厂（MongoDB factory），并在“datstoreFactory.go”的“dbFactoryBuilderMap”中为MongoDB添加一个新条目。

1. 在“userdata”下创建一个新文件夹“mongodb”，并添加MongoDB实现的代码。

通过当前的设计，大大减少了需求变化带来的影响。整个代码修改没有涉及业务逻辑代码。更改仅涵盖数据服务层和应用程序容器，“用例”或“模型”层没有任何更改。对于数据服务层（步骤6），我们只为MongoDB添加新代码，并且没有更改任何现有的MySql代码。

通过步骤1到5，我们对容器（依赖注入）进行了更改以将MongoDB注入到应用程序中，这部分更改了现有代码，但只触及了类型创建部分，其他一切代码都完好无损。

##### **改变用户注册用例（registration use case）调用另一个RESTFul服务:**
<br/>

其次，假设随着功能增多，应用程序变得越来越大，您决定将部分功能拆分为另一个微服务，例如支付服务。现在，您的代码需要调用另一个微服务，它是用RESTFul协议中实现的。以下是步骤：

1. 在“appConfig [type] .yaml”文件中为RESTFul配置添加新条目

1. 将“useCaseConfig”部分下的“userConfig”值更改为指向RESTFul配置

1. 在“appConfig.go”中为RESTFul用户配置创建新的结构类型

1. 在“configValidator.go”中为RESTFul添加一个新常量并创建校验规则。

1. 在“datastorefactory”子包中创建一个新的RESTFul工厂

1. 将新的RESTFul数据接口添加到“RegistrationUseCase”结构中，并修改“registrationFactory.go”为其创建具体类型。

1. 在“adaptor”下创建一个新文件夹，并为RESTFul支付服务创建代码。

通过步骤1到6，我们对容器（依赖注入）进行了更改，以将RESTFul注入到程序中，此部分将触及现有代码。但是通过把更改限制在只对容器，它大大降低了修改的影响，并保护业务逻辑不会被意外更改。第7步是RESTFul服务的真正实现。
##### **设计的成本:**
<br/>

接下来，让我们评估设计的成本。

1. 为用例（usecase）层创建接口

1. 为数据服务层（dataservice）创建接口

1. 创建调用其他微服务的接口

1. 创建程序容器以执行依赖注入

步骤1到3几乎没有额外的工作，对于第3步，您可能无法绕过。

第4步有一定的工作量，并且比较复杂性。这是基于接口编程的结果。每个函数都通过接口调用另一个函数，但是你需要一个地方来创建具体的类型，那就是应用程序容器，其中所有的复杂性都在其中。大多数复杂性来自于我们希望简化创建新类型带来的工作，因此容器必须足够灵活以适应新类型的加入。

如果您的程序不会引入很多新类型，或者您宁愿将来花费更多时间但想现在节省一些时间，那么您可以通过以下步骤使其更加简单。首先，如果您不需要灵活地切换到另一个日志记录器，请删除“logger”包。其次，删除“config”包。这样您不需从YAML文件中读取配置，但是您也失去了通过配置文件更改应用程序行为的灵活性。第三，您甚至可以删除工厂方法模式。但是，您还将失去上述所有优势，并且可能会在进行技术更改时冒险破坏业务逻辑的风险。

**配置管理:**

某些修改的复杂性来自需要从文件中读取配置。 它是为了将来可以从配置服务器（configuration server）（管理应用程序配置的程序）读取配置做准备。 在微服务环境（特别是Docker或Kubernetes环境）中，服务器URL是动态生成和销毁的，无法在静态文件中进行管理。 我认为动态加载应用程序配置的功能是必须的而不是可有可无的。 使用当前的设计，我可以轻松地将“appConfig.go”更改为使用[Viper](https://github.com/spf13/viper)¹²，它支持配置管理。

##### **结论:**

当前的设计为程序增加了一些复杂性，但在动态部署（docker或Kubernetes）环境中可能无法避免其中的一些。 总的来说，你可以从这些额外的工作中获得很大的好处，所以我不认为这个设计是过度的。

##### **源程序:**

完整的源程序链接 [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

##### **其他文章:**

请在这里阅读本系列的其他文章 “[清晰架构（Clean Architecture）的Go微服务](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

##### **索引:**
[1][The Clean Code Blog]
(https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

[2][S.O.L.I.D is for the first five object-oriented design (OOD) principles introduced by Robert C. Martin, popularly known as Uncle Bob and the acronym is introduced later by Michael Feathers]
(http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)

[3][SOLID Go Design]
(https://dave.cheney.net/2016/08/20/solid-go-design)

[4][IoC Container ( Dependency Injection)]
(https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans)

[5][Go at Google: Language Design in the Service of Software Engineering]
(https://talks.golang.org/2012/splash.article)

[6][Is Go An Object Oriented Language?]
(https://spf13.com/post/is-go-object-oriented/)

[7][Interface-based programming]
(https://en.wikipedia.org/wiki/Interface-based_programming)

[8] [Go Microservice with Clean architecture: Dependency Injection]
(https://jfeng45.github.io/posts/dependency_injection/)

[9][Open–closed principle]
(https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)

[10][Data access object]
(https://en.wikipedia.org/wiki/Data_access_object)

[11][Go Microservice with Clean Architecture: Application Design]
(https://jfeng45.github.io/posts/clean_architecture_application_design/)

[12][viper]
(https://github.com/spf13/viper)
