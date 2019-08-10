---
title: "应用清晰架构（Clean Architecture）的Go微服务"
shortTitle: "Go微服务系列"

date: 2019-07-22
draft: true
tags: ["Golang", "Grpc", "清晰架构", “微服务”]
categories: ["Go微服务"]

# canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-a08fa916a5db"

description: “我写了一系列关于用清晰架构（Clean Architecture）和SOLID设计Go Microservice和gRPC 的文章。它讨论了应用程序设计，应用程序布局和项目结构，日志记录，错误处理，事务管理，应用程序容器（Application Container）和依赖注入（Dependency Injection）。“ 
---

我用Go和gRPC创建了一个微服务项目，并试图找出是最好的程序结构，它可以作为我其他项目的模板。我还将程序设计和编程的最佳实践应用于Go Microservice程序，例如清晰架构（Clean Architecture），依赖注入（Dependency Injection），日志记录，错误处理等。我有Java背景，并发现自己在Java和Go之间挣扎，它们之间的编程理念完全不同。我写了一系列关于在项目工作中做出的设计决策和取舍的文章。


阅读这些文章不需要了解Go，但如果你有Go基础绝对会有帮助。如果您不会Go并且无法确定文章中的代码在做什么，那么您需要从这里[Go by Example](https://gobyexample.com/)¹(您不必完成里面的所有主题，只需要前面几个）学习一些基本的Go。本系列中的“事务支持”涉及到数据库，需要Go中的一些数据库知识，您可以从[Go database / sql tutorial](http://go-database-sql.org/index.html)获取它²。如果您对Go Microservice编程感兴趣并思考和关心代码设计，程序结构，编码风格，日志记录，事务管理和依赖注入，那么这个系列非常适合您。

**本系列的侧重点？**

本系列不是关于如何快速创建程序原型，而是关于如何进行良好的程序设计使之能适应将来的变化。例如，您可能希望将本微服务的部分功能拆分为单独的微服务，或添加事务支持，或切换到更好的日志记录器，但不想更改代码中的每个日志记录语句。运用此项目中的程序设计，在进行上述更改时不会触及业务逻辑代码。您甚至很少更改任何现有代码（容器代码除外），多数时候只添加新代码，因此QA工作量大大减少。您可以使用此程序作为下一个微服务应用的基础框架，省去了从头开始的麻烦。您唯一需要做的就是熟悉本框架的内部结构。如果你有Spring（Java）背景或来自面向对象的经历，或者知道清晰架构（Clean Architecture）或SOLID (面向对象设计)，那么这些代码应该对你很熟悉。

您无需按以下顺序阅读文章。 如果您熟悉[清晰架构（Clean Architecture）](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)³或[SOLID (面向对象设计)](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)⁴，你可以按任意顺序阅读文章而不会有任何问题。 但我还是建议您至少先读完第一篇，它将为您提供整个项目的概述，然后您可以选择后面的任何一篇的文章。

1. [清晰架构（Clean Architecture）的Go微服务: 程序结构](https://jfeng45.github.io/posts/go_microservice_application_layout/)

1. [清晰架构（Clean Architecture）的Go微服务: 程序设计](https://jfeng45.github.io/posts/clean_architecture_application_design/)

1. [清晰架构（Clean Architecture）的Go微服务: 设计原则](https://jfeng45.github.io/posts/clean_architecture_design_principle/)

1. [清晰架构（Clean Architecture）的Go微服务: 编码风格](https://jfeng45.github.io/posts/coding_style/)

1. [清晰架构（Clean Architecture）的Go微服务: 事物管理](https://jfeng45.github.io/posts/transaction_support/)

1. [清晰架构（Clean Architecture）的Go微服务: 日志管理](https://jfeng45.github.io/posts/go_logging_and_error_handling/)

1. [清晰架构（Clean Architecture）的Go微服务: 程序容器（Application Container）](https://jfeng45.github.io/posts/application_container/)

1. [清晰架构（Clean Architecture）的Go微服务: 依赖注入（Dependency Injection）](https://jfeng45.github.io/posts/dependency_injection/)

1. 清晰架构（Clean Architecture）的Go微服务: 服务的健壮性

##### **源程序:**

完整的源程序链接 [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

##### **索引:**

[1] [Go by Example]
(https://gobyexample.com/)

[2] [Go database/sql tutorial]
(http://go-database-sql.org/index.html)

[3][The Clean Code Blog]
(https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

[4] [S.O.L.I.D is for the first five object-oriented design (OOD) principles introduced by Robert C. Martin, popularly known as Uncle Bob and the acronym is introduced later by Michael Feathers]
(http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)
