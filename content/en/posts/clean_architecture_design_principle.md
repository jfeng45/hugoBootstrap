---
title: "Go Microservice with Clean Architecture: Design Principle"
shortTitle: "Design Principle"
date: "2019-07-22"
draft: false
tags: ["Golang","Clean Architecture"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-micro-service-with-clean-architecture-design-principle-118d2eeef1e6"
description: "I used Spring’s interface-based programming and Dependency Injection to implement Bob Martin’s Clean Architecture with Go’s simplicity idiom."

---

## Inspiration of design:

I wrote a Go Microservice application recently and the design of this application coming from three inspirations:

* [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)¹ and [SOLID](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)² design [principle](https://dave.cheney.net/2016/08/20/solid-go-design)³

* [Spring’s application context](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans)⁴

* [Go’s concise design](https://talks.golang.org/2012/splash.article)⁵ especially [Go’s implementation of object-oriented design](https://spf13.com/post/is-go-object-oriented)⁶

I used Spring’s interface-based programming and Dependency Injection to implement Bob Martin’s Clean Architecture with Go’s simplicity idiom. Trade-offs have been made when there are conflicts among them. I only applied Clean Architecture's high-level design principles (mainly SOLID), so the detail of the implementation may be different from other implementations.

Coming from Java background, I am quite familiar with the first two design ideas. After learning Go, I gradually identified Go’s concise design. Generally speaking, there are two different programming style, one is object-oriented; the other is non object-oriented, which believes the idea of creating simplest code that works without up-front design. Go is closer to the second camp, even though it has some object-oriented features. Programming in Go gives me a new perspective to reevaluate object-oriented programming and influenced my coding style. The result is that I am doing object-oriented design only when it is necessary and I prefer a solution with smaller effort than a perfect solution taking a lot of effort.

## Design principle:

1. [Programming on interface](https://en.wikipedia.org/wiki/Interface-based_programming)⁷

    There are three major layers in this application , use case, data service and domain model, among them only domain model has no interface because there is no need. When you access outside services, you do that through interface.


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

    The key of interface-based programming is to pass in interfaces into a function as parameters and return interfaces instead of concrete types. For example, in the above code, look at the return-“dataservice.UserDataInterface”, which is an interface, not a struct. The calling function doesn’t need to know the returned concrete struct because the interface encapsulates all the information it needed. This gives you a great flexibility to replace the returned struct with another one without impact the calling function.

2. Create concrete type by dependency injection through the factory method pattern.

    The application container is responsible for creating concrete types and inject them into functions. It will be explained in detail in another “[Dependency Injection](https://jfeng45.github.io/posts/dependency_injection/)”⁸.

3. Set up correct dependency

    It means the following
    
    * Each layer or component in the application has it’s own package. The interface is defined on the top level package and the concrete type is hidden in sub-packages.
    
    * Dependency between different layers is only on interfaces not on concrete types
    
    * The hierarchy of dependency from top level down is: “use case”, “data service” and “model”.
    
    One way to measure quality of dependency is by the size of the import, the shorter the import the better the dependency relationship.

4. [Open-close principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)⁹

    This is my favorite design principle among all of them. It basically says whenever a new feature is added, instead of modifying existing code, trying to add new code. The way to do it is by apply #1 and #2 in above. There are many great real world examples of this principle, such as, [DAO](https://en.wikipedia.org/wiki/Data_access_object)¹⁰. The benefit is that you can’t mess up existing code because only new code is added, which will greatly reduce the QA effort.

## Is the design over-engineered?

Compared with similar solution in Java, the code in this application is quite concise thanks to Go’s simple design of the language itself. But for people coming from other programming paradigms, the design of this application may be overkill. I also asked myself the same question. In order to get the answer, cost and benefit need to be compared to arrive the conclusion.

There are two types of change, business logic change and technical change. When writing business code, you don’t want to focuse on whether the data is from MongoDB or MySQL or a Microservice. When making technical changes, the nightmare is to accidentally break business features. A good design separates those two types of concern and make you only focus on one perspective at a time.

Generally speaking, technical changes are not happening as often as business changes, but with the spread of Microservice, new technology will be adopted faster, which will speed up the technical changes.

## Benefits of the design:

The following are couple examples to show you what steps are needed to make some changes. If you get a little confused with this section, you may want to read “[application design](https://jfeng45.github.io/posts/clean_architecture_application_design/)”¹¹ first, which will give you an overview of the application structure.

### Change from MySQL to MongoDB:

First, lets say we need to change the persistence layer of the domain model “User” from MySQL to MongoDB. The following are the steps:

1. Add new configuration information for MongoDB in the “appConfig[type].yaml” file

1. Change the value of “userConfig” under “useCaseConfig” section in “appConfig[type].yaml” file to point to MongoDB instead of MySql

1. Create a new struct type for MongoDB in “appConfig.go”

1. Add a new constant for MongoDB in “configValidator.go” and create validation rules.

1. Create a new MongoDB factory in “datastorefactory” package and add a new entry for MongoDB in “dbFactoryBuilderMap” in “datstoreFactory.go”.

1. Create a new folder “mongodb” under “userdata” and add the code for MongoDB implementation.

By applying the current design, the impact of the change is greatly reduced. There is no touch on the business logic code. The changes only cover the data service layer and the app container layer, and there is no change for the “use case” or model layer. For the data service layer (step 6), we only add new code for MongoDB and didn’t change any of the existing MySql code.

Through steps 1 to 5, we make changes to the container (Dependency Injection) to inject MongoDB into the application, and this part changes the existing code, but only touches the type creation part, everything else is intact.

### Change registration use case to call REST service:

Second, lets say with more features added, the application becomes bigger and bigger and you decide to split part of the functionality into another Microservice, for example payment service. Now, you code need to call another Microservice, which is implemented in RESTFul protocol. The following are the steps:

1. Add a new entry for RESTFul config in the “appConfig[type].yaml” file

1. Change the value of “userConfig” under “useCaseConfig” section to point to RESTFul config

1. Create a new struct type for RESTFul User config in “appConfig.go”

1. Add a new constant for RESTFul in “configValidator.go” and create validation rules.

1. Create a new RESTFul factory in “datastorefactory” sub-package

1. Add a new RESTFul data interface into “RegistrationUseCase” struct and modify “registrationFactory.go” to create a concrete type for it.

1. Create a new folder under “adapter” and create the code for RESTFul implementation of payment service.

Through steps 1 to 6, we make changes to the container (Dependency Injection) to inject RESTFul into the application, and this part touches the existing code. But by restricting the changes to the container, it greatly reduced the impact of the modification and protected the business logic from being accidentally changed. Step 7 is the real implementation of the RESTFul service.

## Cost of Design:

Next, let’s evaluate the cost of the design.

1. Create interface for use case layer

1. Create interface for data service layer

1. Create interface for calling other Microservices

1. Create application container to do Dependency Injection

Steps 1 to 3 has little extra work and for step 3, you probably need to do it anyway.

Step 4 is pretty involved and has some complexity. It is the result of interface-based programming. Each function is calling another one by the interface, but you do need a place to create the concrete type, and that is the application container, where all the complexity is in. Most complexity is coming from the fact that we want to make it easy to create new types in the future, so the container has to be flexible enough to accommodate new types.

If you know that your application won’t have many new types induced, or you’d rather spend a lot more time in the future and save some time now, then you can make it a lot simpler by the following steps. First, remove logger package if you don’t need the flexibility of switching to another logger. Second, remove the “config” package. That way you don’t read the configuration from a YAML file, but you also lost the flexibility to change the application behavior by configuration file. Third, you can even remove the factory method pattern. However, you will also lose all the benefit mentioned above and may risk breaking business features when working on technical changes.

**Configuration Management:**

Some complexity coming from the need to read the configuration from a file. In the future, the configuration could be read from a configuration server ( a central place to manage application configuration). In a Microservice environment ( especially a Docker or Kubernetes one), the server URL are dynamically generated and destroyed, which can’t be managed in a static file. I consider the feature of loading application configuration dynamically as a must have rather than nice to have. With the current design, I can easily change the “appConfig.go” to use [Viper](https://github.com/spf13/viper)¹², which supports configuration management.

## **Conclusion:**

The current design added some complexity to the application, but some of it may not be avoided in the dynamic deploying (docker or Kubernetes) environment. Overall, you get a great benefit over some extra work, so I don’t think the design is over-engineered.

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Other articles:

Please read the rest of the articles in this series in “[Go Microservice with Clean Architecture](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

## Reference:

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
