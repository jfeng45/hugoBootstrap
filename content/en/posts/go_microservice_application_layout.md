---
title: "Go Microservice with Clean Architecture: Application Layout"
shortTitle: "Go Microservice Application Layout"
date: "2019-07-21"
draft: false
tags: ["Golang", "Grpc", "Clean Architecture", "Microservice", "Application Layout"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-micro-service-with-clean-architecture-application-layout-e6216dbb835a"
description: '"Standard Go Project Layout" is not a good fit for Go Microservice gRPC project. This article Using 
a real application to show what is a good what is a good Microservice application layout and the reason behind it.'

---

I created a Microservice with Go and gRPC and tried to figure out what is the best project layout, which can be used as a template for my future projects. I came from Java background and found myself keeping struggling between Java and Go idioms, which are quite different. I wrote a series of articles about decisions and trade-offs I made when working on the project. This is the first one, which is about project structure.

## Project layout resource

The best resource for Go’s standard project layout probably is “[Standard Go Project Layout](https://github.com/golang-standards/project-layout)”¹ on Github, but it is not a good fit for my project. After reading [Sylvain Wallez’s article](https://bluxte.net/musings/2018/04/10/go-good-bad-ugly/)², I finally got some context information on the reason behind it. Go is designed for API and network servers and most Go projects on Github are written as libraries, so “Standard Go Project Layout” probably is a good fit. Business Microservice project is a totally different animal, which requires a different project structure. I did adopt some of advice from “Standard Go Project Layout” that is applicable, which counts for about 30%.

Generally speaking, there are two different ways to create an application layout: based on business features or based on technical layout. It is [common agreement](http://www.javapractices.com/topic/TopicAction.do?Id=205)³ that based on business features is better, which is true for monolithic project. In Microservice architecture, the situation changed because each service has it’s own repository. So in-side each Microservice, it actually makes sense to create the project structure based on technical layout.

I also found some good advice from Ben Johnson on [application structure](https://medium.com/@benbjohnson/structuring-applications-in-go-3b04be4ff091)⁴ and [package layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)⁵. But it didn’t give a whole solution for my project, so I decided to create my own. Project layout depends on the project requirements and here are them.

## Project Requirements:

1. Support different databases ( Sql and NoSql database) on data persistence layer

2. Support data coming from other Microservices using different protocols such as gRPC or REST

3. Loose coupled and highly cohesive

4. Support easy and consistent logging and be able to change it ( for example, logging level and logging provider) without modifying logging statements in every file.

5. Support business level transaction.

Project layout is also influenced by application design. I applied [Bob Martin’s Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)⁶ with Go’s [simplicity](https://talks.golang.org/2012/splash.article)⁷ design style.

On the business logic side, there are three layers: “model”, which is domain model; “dataservice”, which is data persistence (database) layer; “usecase”, which is business logic layer.

## Top level project structure:

![service tmpl](/images/serviceTmpl.jpg)

**adapter:** this is the interface between your application and outside data service, for example another gRPC service. All the data conversion and transformation happened here, so your business logic code doesn’t need to be aware of the detail implementation (whether it gRPC or REST) of outside services.

**cmd:** the command. All different types of “main.go” are here and you can have multiple ones. This is the starting point of the application.

**config:** configuration code and data for the application, including configuration files.

**container:** the dependency injection container, which is responsible for creating concrete types and injecting them into each function.

**dataservice:** persistence layer, which is responsible for retrieving and modifying data for the domain model. It only depends on the model layer. Data service could get data from database or from other Microservices by RPC or RESTFul call.

**model**: domain module layer, which has domain structs. All other layers depend on them and they don’t depend on any other layers.

**script:** scripts related to setting up an application, for example, database scripts.

**tool:** third-party tools.

**usecase** : This is an important layer and the entry point of business logic. Each business feature is implemented by a use case. It is the top level layer, so no other layer depends on it ( except “cmd”), but it depends on other layers.

Use case and data service layer functions are all called by interface, so the implementation can be changed easily.

## **Sub folder under each top level:**

**adapter:**

![adapter](/images/adapter.jpg)

When a program needs to interact with Microservices or other outside services, you need to create interfaces to reduce dependency. For example, the cache service is a gRPC Microservice. Each outside service will have it’s own sub-package and files. For example, cache service has “cacheclient” package and “cacheClient.go” file, which defines the types and the methods to interact with outside “cache” Microservice.

In our example, not like other data service, “cacheClient.go” doesn’t define an interface for cache service. Actually it has one, but the interface is defined in “dataservice” package because cache service is also a data service. The more clear way probably is creating an interface in both packages, that will keep the package structure consist. But the two interfaces will be identical and redundant, so I removed the one in adapter.

Since we also need to expose the application itself as a gRPC service, “userclient” sub-package is created here. “generatedclient” sub-package has generated code for gRPC and Protobuf. “userGrpc.go” is created to handle format translation between domain model structs and gRPC structs.

**cmd:**

![cmd](/images/cmd.jpg)

The command of the app, and starting point for the app. You can either run the application as a local app or run it as a Microservice application, in which case, you have both client side (grpcClientMain.go) and server side (grpcServerMain.go) main files. All future main files will also go here, for example, web application server main file.

**config:**

![config](/images/config.jpg)

Save all application configuration files here. “appConfig.go” is responsible for reading from configuration files and load them into application configuration structs. You can have different configuration files (YAML files) for different environments, for example “Dev” and “Prod”.

**container:**


![container](/images/container.jpg)

The most complex package in this app, which creates the concrete structs for each interface and injects them into other layers. The sub-level structure in it resembles the application layer, which also have “use case”, “data service” and “data store” layer.

Under the top level, “container.go” defines the container interface, and the implementation file “serviceContainer.go” is in “servicecontainer” package. It is the entry point or this package, which glues other code in this folder together. “registrationFactory.go” in “usecasefactory” sub-package and other factories using factory method pattern to create structs. [Logging](https://jfeng45.github.io/en/posts/go_logging_and_error_handling/)⁸ doesn’t belongs to any application layer, so I created a separate sub-package “loggerfactory” for it. It also has a “logger” sub-package to define logging interface. I explain everything in this package in my article “[Application Container](https://jfeng45.github.io/en/posts/application_container/)”⁹.

**dataservice:**

![dataservice](/images/dataservice.jpg)

The persistence layer for domain model. There is one file-“dataService.go” on the top level, which has all the interfaces for data service, including data services provided by other Microservices ( for example, gRPC). Other packages only need to depend on the top level data service interfaces and don’t need to be aware of the implementation detail of a specific database.

For each type in the domain model, there is a corresponding data service. For example, for model “User”, there is a “userdata” sub-package, which has all implementations of user persistence service including MySql and CouchDB.

**model:**

![model](/images/model.jpg)

This layer doesn’t need to have an interface, and models are all concrete structs.

**Script:**

![script](/images/scripts.jpg)

Currently it only has database scripts for MySql.

**tool:**

![tool](/images/tools.jpg)

This package is for third-party tools. If you don’t want to depend on third-party library directly, or you need to enhance those tools, create a wrapper here. Don’t be confused with “adapter” package, which is also dealing with third-part libraries, but those are only for application level data service. “tool” package is more for lower level libraries.

**useCase:**

![use case](/images/usecase.jpg)

There is one file “useCase.go” under top level folder, which has all the use case interfaces. Currently, there are three use cases: “RegistrationUseCase” , “ListUserUseCase” and “ListCourse”.

For each use case, there is a sub-package for it. The files in that sub-package define the concrete structs that implement the use case interface. All other packages only depend on the use case interface in the top level and don’t need to be aware of the implementation detail of each use case.

### Potential problem:

This application layout ends up with a lot of small sub-packages each only has one or a few files in it. This contradicts with the Go idiom “[Consider fewer, larger packages](https://dave.cheney.net/practical-go/presentations/qcon-china.html#_consider_fewer_larger_packages)”¹⁰, which suggests creating package like this for use case:

![use case alt](/images/usecaseNew.jpg)

If you write a library for others to import into their code, it is a good rule to put code into one big package, so people don’t need multiple import statements in order to use your library. But inside your application, it is fine to have small packages, especially when you only expose interfaces to other layers.

Here is why small package is better for the use case layer. First “useCase.go” only defines interfaces in it, and other packages ( except container) only depend on interfaces, so “useCase.go” deserves a standalone package. Second, it is more clear to separate each use case by folder.

## Conclusion:

For a gRPC Microservice project, the “Standard Go Project Layout” may not be a good fit. The common agreement that creating the application layout based on business logic instead of technical layout is a good advice for monolithic project, not necessarily for Microservice project. This article uses a real application as an example to show what could be a good Microservice application layout and the reason behind it.

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Other articles:

Please read the rest of the articles in this series in “[Go Microservice with Clean Architecture](https://jfeng45.github.io/en/posts/clean_architecture_with_go/)”.

## Reference:

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
(https://jfeng45.github.io/en/posts/go_logging_and_error_handling/)

[9][Go Microservice with Clean Architecture: Application Container]
(https://jfeng45.github.io/en/posts/application_container/)

[10][Practical Go: Real world advice for writing maintainable Go programs]
(https://dave.cheney.net/practical-go/presentations/qcon-china.html)
