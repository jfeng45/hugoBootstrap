---
title: "Go Microservice with Clean Architecture"
shortTitle: "Go Microservice Series(*****)"

date: 2019-07-22
draft: false
tags: ["Golang", "Grpc", "Clean Architecture", "Microservice"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-a08fa916a5db"

description: "I wrote a series of articles on Go Microservice with gRPC in Clean Architecture and SOLID design.
It talked about best practice in application design, application layout and project structure, logging, error handling, 
transaction management, application container and Dependency Injection." 
---

I created a Microservice project with Go and gRPC and tried to figure out what is the best project layout, which can be used as a template for my future projects. I also tried to apply the best practice in application design and programming, such as Clean Architecture, Dependency Injection, logging, error handling and so on, to the Go Microservice application. I came from Java background and found myself keeping struggling between Java and Go idioms, which are quite different. I wrote a series of articles about decisions and trade-offs I made when working on the project.

**Who is the audience?**

Knowing basic Go is not required for reading the articles, but definitely will help you. If you don’t know Go and can’t figure what the code is doing, then you probably want to learn some basic Go from “[Go by Example](https://gobyexample.com/)”¹ (You don’t need to finish all of the topics, just the first several). The “Transaction support” one in this series is database heavy and require some database knowledge in Go and you can acquire it from “[Go database/sql tutorial](http://go-database-sql.org/index.html)”². If you are interested in Go Microservice programming and thinking about the code design and application layers, project structure, coding style, logging, transaction management and Dependency Injection, then this series is perfect for you.

**What is it about?**

The series is not about how to create a quick working prototype, but about how to create an application with a good design that easily adapts to changes, for example, you may want to split part of functions into a separate Microservice, or add transaction support after a business feature is up-running, or switch to a better logger, but don’t want to change every logging statement in your code. By applying the design in this project, business logic code is never touched when making the above changes. You seldom change any existing code (except for the container code), but only add new code, thus the QA effort was greatly reduced. You can use this application as the starting point to base your next application on. The only cost is to figure how different pieces of the application are wired together. If you have Spring (Java) background or coming from object-oriented paradigm, or know Clean Architecture or SOLID, the code should be pretty familiar to you.

You don’t need to read articles in the following order. If you are familiar with [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)³ or [SOLID](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)⁴, you can read the articles in any order without a problem. However, I do recommend you start from the first one, which will give you a overview of the whole project, then you can pick the ones you are interested in.

1. [Go Microservice with Clean Architecture: Application Layout](https://jfeng45.github.io/en/posts/go_microservice_application_layout/)

1. [Go Microservice with Clean Architecture: Application Design](https://jfeng45.github.io/en/posts/clean_architecture_application_design/)

1. [Go Microservice with Clean Architecture: Design Principle](https://jfeng45.github.io/en/posts/clean_architecture_design_principle/)

1. [Go Microservice with Clean Architecture: Coding Style](https://jfeng45.github.io/en/posts/coding_style/)

1. [Go Microservice with Clean Architecture: Transaction Support](https://jfeng45.github.io/en/posts/transaction_support/)

1. [Go Microservice with Clean Architecture: Application Logging](https://jfeng45.github.io/en/posts/go_logging_and_error_handling/)

1. [Go Microservice with Clean Architecture: Application Container](https://jfeng45.github.io/en/posts/application_container/)

1. [Go Microservice with Clean architecture: Dependency Injection](https://jfeng45.github.io/en/posts/dependency_injection/)

1. Go Microservice with Clean architecture: Robust Service

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Reference:

[1] [Go by Example]
(https://gobyexample.com/)

[2] [Go database/sql tutorial]
(http://go-database-sql.org/index.html)

[3][The Clean Code Blog]
(https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

[4] [S.O.L.I.D is for the first five object-oriented design (OOD) principles introduced by Robert C. Martin, popularly known as Uncle Bob and the acronym is introduced later by Michael Feathers]
(http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod)
