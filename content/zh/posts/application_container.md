---
title: "清晰架构（Clean Atchitecture）的Go微服务: 程序容器（Application Container）"
shortTitle: "程序容器"
date: 2019-07-29
draft: false
tags: ["Golang", "Clean Architecture", "Application Container"]
categories: ["Go Microservice"]
tags: ["Golang", "Grpc", "清晰架构", "程序容器"]
categories: ["Go微服务"]

#canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-application-container-477cc3a11e77"

description: "程序容器（Application Container）负责创建具体类型并将它们注入每个函数。 它是本程序中最复杂的部分"

---
清晰架构（Clean Atchitecture）的一个理念是隔离程序的框架，使框架不会接管你的应用程序，而是由你决定何时何地使用它们。在本程序中，我特意不在开始时使用任何框架，因此我可以更好地控制程序结构。只有在整个程序结构布局完成之后，我才会考虑用某些库替换本程序的某些组件。这样，引入的框架或第三方库的影响就会被正确的依赖关系所隔离。目前，除了logger，数据库，gRPC和Protobuf（这是无法避免的）之外，我只使用了两个第三方库[ozzo-validation](https://github.com/go-ozzo/ozzo-validation)¹和[YAML](https://github.com/go-yaml/yaml/tree/v2.2.2)²，而其他所有库都是Go的标准库。

你可以使用本程序作为构建应用程序的基础。你可能会问，那么本框架岂不是要接管整个应用程序吗？是的。但事实是，无论是你自建框架还是引进第三方框架，你都需要一个基本框架作为构建应用程序的基础。该基础需要具有正确的依赖性和可靠的设计，然后你可以决定是否引入其他库。你当然可以自己建立一个框架，但你最终可能会花费大量的时间和精力来完善它。你也可以使用本程序作为起点，而不是构建自己的项目，从而为你节省大量时间和精力。

程序容器是项目中最复杂的部分，是将应用程序的不同部分粘合在一起的关键组件。本程序的其他部分是直截了当且易于理解的，但这一部分不是。好消息是，一旦你理解了这一部分，那么整个程序就都在掌控之中。

##### **容器包（“container” package）的组成部分:**
<br/>
容器包由五部分组成:

1. “容器”（“container”）包：它负责创建具体类型并将它们注入其他文件。 顶级包中只有一个文件“container.go”，它定义容器的接口。

    ![container](/images/container.jpg)

2. “servicecontainer”子包：容器接口的实现。 只有一个文件“serviceContainer.go”，这是“容器”包的关键。 以下是代码。 它的起点是“InitApp”，它从文件中读取配置数据并设置日志记录器（logger)。

    {{< highlight go "linenos=table,linenostart=1" >}}
    type ServiceContainer struct {
        FactoryMap map[string]interface{}
        AppConfig  *config.AppConfig
    }
    
    func (sc *ServiceContainer) InitApp(filename string) error {
        var err error
        config, err := loadConfig(filename)
        if err != nil {
            return errors.Wrap(err, "loadConfig")
        }
        sc.AppConfig = config
        err = loadLogger(config.Log)
        if err != nil {
            return errors.Wrap(err, "loadLogger")
        }
    
        return nil
    }
    
    // loads the logger
    func loadLogger(lc config.LogConfig) error {
        loggerType := lc.Code
        err := logFactory.GetLogFactoryBuilder(loggerType).Build(&lc)
        if err != nil {
            return errors.Wrap(err, "")
        }
        return nil
    }
    
    // loads the application configurations
    func loadConfig(filename string) (*config.AppConfig, error) {
    
        ac, err := config.ReadConfig(filename)
        if err != nil {
            return nil, errors.Wrap(err, "read container")
        }
        return ac, nil
    }
    {{< / highlight >}}

3. “configs”子包：负责从YAML文件加载程序配置，并将它们保存到“appConfig”结构中以供容器使用。

    ![config](/images/config.jpg)

4. “logger”子包：它里面只有一个文件“logger.go”，它提供了日志记录器接口和一个“Log”变量来访问日志记录器。 因为每个文件都需要依赖记录，所以它需要一个独立的包来避免循环依赖。

    ![logger](/images/logger.jpg)

5. 最后一部分是不同类型的工厂（factory）。
   
     它的内部分层与应用层分层相匹配。 对于“usecase”和“dataservice”层，有“usecasefactory”和“dataservicefactory”。 另一个工厂是“datastorefactory”，它负责创建底层数据处理链接。 因为数据提供者可以是gRPC或除数据库之外的其他类型的服务，所以它被称为“datastorefactry”而不是“databasefactory”。 日志记录组件（logger）也有自己的工厂。
   
**用例工厂（Use Case Factory）:**

对于每个用例，例如“registration”，接口在“usecase”包中定义，但具体类型在“usecase”包下的“registration”子包中定义。 此外，容器包中有一个对应的工厂负责创建具体的用例实例。 对于“注册（registration）”用例，它是“registrationFactory.go”。 用例与用例工厂之间的关系是一对一的。 用例工厂负责创建此用例的具体类型（concrete type）并调用其他工厂来创建具体类型所需的成员（member in a struct)。 最低级别的具体类型是sql.DBs和gRPC连接，它们需要被传递给持久层，这样才能访问数据库中的数据。

如果Go支持泛型，你可以创建一个通用工厂来构建不同类型的实例。 现在，我必须为每一层创建一个工厂。 另一个选择是使用反射（refection)，但它有不少问题，因此我没有采用。

**“Registration” 用例工厂（Use Case Factory）:**

每次调用工厂时，它都会构建一个新类型。以下是“注册（Registration）”用例创建具体类型的代码。 它是工厂方法模式（factory method pattern）的典型实现。 如果你想了解有关如何在Go中实现工厂方法模式的更多信息，请参阅[此处](https://stackoverflow.com/a/49714445)³.

{{< highlight go "linenos=table,linenostart=1" >}}
// Build creates concrete type for RegistrationUseCaseInterface
func (rf *RegistrationFactory) Build(c container.Container, appConfig *config.AppConfig, key string) (UseCaseInterface, error) {
	uc := appConfig.UseCase.Registration
	udi, err := buildUserData(c, &uc.UserDataConfig)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	tdi, err := buildTxData(c, &uc.TxDataConfig)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	ruc := registration.RegistrationUseCase{UserDataInterface: udi, TxDataInterface: tdi}

	return &ruc, nil
}

func buildUserData(c container.Container, dc *config.DataConfig) (dataservice.UserDataInterface, error) {
	dsi, err := dataservicefactory.GetDataServiceFb(dc.Code).Build(c, dc)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	udi := dsi.(dataservice.UserDataInterface)
	return udi, nil
}
{{< / highlight >}}

<br/>

**数据存储工厂（Data store factory）:**

“注册（Registration）”用例需要通过数据存储工厂创建的数据库链接来访问数据库。 所有代码都在“datastorefactory”子包中。 我详细解释了数据存储工厂如何工作，请看这篇文章[依赖注入（Dependency Injection）](https://jfeng45.github.io/posts/dependency_injection/)。

数据存储工厂的当前实现支持两个数据库和一个微服务，MySql和CouchDB，以及gRPC缓存服务; 每个实现都有自己的工厂文件。 如果引入了新数据库，你只需添加一个新的工厂文件，并在以下代码中的“dsFbMap”中添加一个条目。

{{< highlight go "linenos=table,linenostart=1" >}}

// To map "database code" to "database interface builder"
// Concreate builder is in corresponding factory file. For example, "sqlFactory" is in "sqlFactory".go
var dsFbMap = map[string]dsFbInterface{
	config.SQLDB:      &sqlFactory{},
	config.COUCHDB:    &couchdbFactory{},
	config.CACHE_GRPC: &cacheGrpcFactory{},
}

// DataStoreInterface serve as a marker to indicate the return type for Build method
type DataStoreInterface interface{}

// The builder interface for factory method pattern
// Every factory needs to implement Build method
type dsFbInterface interface {
	Build(container.Container, *config.DataStoreConfig) (DataStoreInterface, error)
}

//GetDataStoreFb is accessors for factoryBuilderMap
func GetDataStoreFb(key string) dsFbInterface {
	return dsFbMap[key]
}

{{< / highlight >}}

<br/>

以下是MySql数据库工厂的代码，它实现了上面的代码中定义的“dsFbInterface”。 它创建了MySql数据库链接。

容器内部有一个注册表（registry），用作数据存储工厂创建的链接（如DB或gRPC连接）的缓存，它们在整个应用程序创建一次。 无论何时需要它们，需首先从注册表中检索它，如果没有找到，则创建一个新的并将其放入注册表中。 以下是“Build”代码。

{{< highlight go "linenos=table,linenostart=1" >}}
// sqlFactory is receiver for Build method
type sqlFactory struct{}

// implement Build method for SQL database
func (sf *sqlFactory) Build(c container.Container, dsc *config.DataStoreConfig) (DataStoreInterface, error) {
	key := dsc.Code
	//if it is already in container, return
	if value, found := c.Get(key); found {
		sdb := value.(*sql.DB)
		sdt := databasehandler.SqlDBTx{DB: sdb}
		logger.Log.Debug("found db in container for key:", key)
		return &sdt, nil
	}

	db, err := sql.Open(dsc.DriverName, dsc.UrlAddress)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	// check the connection
	err = db.Ping()
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	dt := databasehandler.SqlDBTx{DB: db}
	c.Put(key, db)
	return &dt, nil

}
{{< / highlight >}}

<br/>

**Grpc Factory:**

对于“listUser”用例，它需要调用gRPC微服务（缓存服务），而创建它的工厂是“cacheFactory.go”。 目前，数据服务的所有链接都是由数据存储工厂创建的。 以下是gRPC工厂的代码。 “Build”方法与“SqlFactory”的非常相似。

{{< highlight go "linenos=table,linenostart=1" >}}

// DataStoreInterface serve as a marker to indicate the return type for Build method
type DataStoreInterface interface{}

// cacheGrpcFactory is an empty receiver for Build method
type cacheGrpcFactory struct{}

func (cgf *cacheGrpcFactory) Build(c container.Container, dsc *config.DataStoreConfig) 
     (DataStoreInterface, error) {
	key := dsc.Code
	//if it is already in container, return
	if value, found := c.Get(key); found {
		return value.(*grpc.ClientConn), nil
	}
	//not in map, need to create one
	logger.Log.Debug("doesn't find cacheGrpc key=%v need to created a new one\n", key)

	conn, err := grpc.Dial(dsc.UrlAddress, grpc.WithInsecure())
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	c.Put(key, conn)
	return conn, err
}
{{< / highlight >}}

<br/>

**Logger factory:**

Logger有自己的子包名为“loggerfactory”，其结构与“datastorefactory”子包非常相似。 “logFactory.go”定义了日志记录器工厂构建器接口（builder interface）和映射（map）。 每个单独的日志记录器都有自己的工厂文件。 以下是日志工厂的代码：

{{< highlight go "linenos=table,linenostart=1" >}}
// logger mapp to map logger code to logger builder
var logfactoryBuilderMap = map[string]logFbInterface{
	config.ZAP:    &ZapFactory{},
	config.LOGRUS: &LogrusFactory{},
}

// interface for logger factory
type logFbInterface interface {
	Build(*config.LogConfig) error
}

// accessors for factoryBuilderMap
func GetLogFactoryBuilder(key string) logFbInterface {
	return logfactoryBuilderMap[key]
}

{{< / highlight >}}
<br/>

以下是ZAP工厂的代码。 它类似于数据存储工厂。 只有一个区别。 由于记录器创建功能仅被调用一次，因此不需要注册表。

{{< highlight go "linenos=table,linenostart=1" >}}
// receiver for zap factory
type ZapFactory struct{}

// build zap logger
func (mf *ZapFactory) Build(lc *config.LogConfig) error {
	err := zap.RegisterLog(*lc)
	if err != nil {
		return errors.Wrap(err, "")
	}
	return nil
}
{{< / highlight >}}

<br/>

##### **配置文件:**
<br/>

配置文件使你可以全面了解程序的整体结构：

![config1](/images/config1.jpg)

上图显示了文件的前半部分。 第一部分是它支持的数据库配置; 第二部分是带有gRPC的微服务; 第三部分是它支持的日志记录器; 第四部分是本程序在运行时使用的日志记录器

下图显示了文件的后半部分。 它列出了应用程序的所有用例以及每个用例所需的数据服务。

![config2](/images/config2.jpg)

##### **配置文件中应保存哪些数据？**
<br/>
不同的组件具有不同的配置项，一些组件可能具有许多配置项，例如日志记录器。 我们不需要在配置文件中保存所有配置项，这可能使其太大而无法管理。 通常我们只需要保存需要在运行时更改的选项或者可以在不同环境中（dev, prod, qa)值不同的选项。

#####  **设计是如何进化的?**
<br/>
容器包里似乎有太多东西，问题是我们是否需要所有这些？如果你不需要所有功能，我们当然可以简化它。当我开始创建它时，它非常简单，我不断添加功能，最终它才越来越复杂。

最开始时，我只是想使用工厂方法模式来创建具体类型，没有日志记录，没有配置文件，没有注册表。

我从用例和数据存储工厂开始。最初，对于每个用例，都会创建一个新的数据库链接，这并不理想。因此，我添加了一个注册表来缓存所有连接，以确保它们只创建一次。

然后我发现（我从[这里](https://itnext.io/how-i-pass-around-shared-resources-databases-configuration-etc-within-golang-projects-b27af4d8e8a)获得了一些灵感⁵）将所有配置信息放在一个文件中进行集中管理是个好主意，这样我就可以在不改变代码的情况下进行更改。
我创建了一个YAML文件（appConfig [type] .yaml）和“appConfig.go”来将文件中的内容加载到应用程序配置结构（struct） - “appConfig”中并将其传递给工厂构建器（factory builder）。 “[type]”可以是“prod”，“dev”，“test”等。配置文件只加载一次。目前，它没有使用任何第三方库，但我想将来切换到[Vipe](https://github.com/spf13/viper)⁶，因为它可以支持从配置服务器中动态重新加载程序配置。要切换到Vipe，我只需要更改一个文件“appConfig.go”。

对于日志记录，整个程序我只想要一个logger实例，这样我就可以为整个程序设置相同的日志配置。我在容器内创建了一个日志记录器包。我还尝试了不同的日志库来确定哪一个是最好的，然后我创建了一个日志工厂，以便将来更容易添加新的日志记录器。有关详细信息，请阅读[日志管理](https://jfeng45.github.io/posts/go_logging_and_error_handling/)⁷。

##### **源程序:**

完整的源程序链接 [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

##### **其他文章:**

请在这里阅读本系列的其他文章 “[清晰架构（Clean Architecture）的Go微服务](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

##### **索引:**

[1] [ozzo-validation]
(https://github.com/go-ozzo/ozzo-validation)

[2] [YAML support for the Go language]
(https://github.com/go-yaml/yaml/tree/v2.2.2)

[3][Golang Factory Method]
(https://stackoverflow.com/a/49714445)

[4][Go Microservice with Clean Architecture: Dependency Injection]
(https://jfeng45.github.io/posts/dependency_injection/)

[5] [How I pass around shared resources (databases, configuration, etc) within Golang projects]
(https://itnext.io/how-i-pass-around-shared-resources-databases-configuration-etc-within-golang-projects-b27af4d8e8a)

[6][viper]
(https://github.com/spf13/viper)

[7][Go Microservice with Clean Architecture: Application Logging]
(https://jfeng45.github.io/posts/go_logging_and_error_handling/)
