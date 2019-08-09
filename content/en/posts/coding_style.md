---
title: "Go Microservice with Clean Architecture: Coding Style"
shortTitle: "Coding Style"
date: 2019-07-24
draft: false
tags: ["Golang", "Coding Style", "Clean Architecture"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-micro-service-with-clean-architecture-coding-style-a4da35a27d90"

description: "Coding style is crucial to make programming effective. Good naming makes code self documenting. It includes three parts: application layout, coding rules and style, naming convention"

---


Coding style is a relatively boring topic in programming, but it is crucial to have an appropriate coding style to be an effective programmer. There are three components in it:

1. Application layout and folder structure

1. Coding rules or style

1. Naming convention

I already talked about application layout in “[Go Microservice with Clean Architecture: Application Layout](https://jfeng45.github.io/posts/go_microservice_application_layout/)”¹ , so this article covers the rest.

## **Coding rules or style**

1. No package level variables.

    Package level variables break the function encapsulation and make functions in-deterministic. I followed this rule in the application and the only exception is in “container” package, because of dealing with application level configurations, it is hard to do it here. It is possible to remove package level variables even in “container” package, but it needs a lot of effort, which probably is overkill. However, since the package level variable is confined only in “container” package, the resulted damage is greatly reduced.

2. Use less constants

    Constants are also package level, because of immutability, they are less harmful as global variables, but they still break function encapsulation and need to be restricted.

3. Dependent on interfaces not on concrete types

When you try to minimize the impact of outside changes to your program, no matter it is a function, a package or an application, make your code depends on interfaces rather than concrete types.

## **Naming convention**

Best code is self documenting and good naming plays a critical role. Coming from Java background, Go’s naming convention is very contradicting. After giving it a try, I found it made sense and creating concise code without reducing readability in you apply it right. Go is a relatively low level language and people use it to write networking, driver and docker container code. In those environments, it is appropriate to use concise names.

However, when writing a business application, we need to create many different types or structs to deal with the similar business concepts, concise naming doesn’t apply anymore. For example, to deal with “User” related concepts, we have “User”, “UserDataService”, “RegistrationUseCase”, “RegistrationUseCaseIterface” and “UserDataInterface”, and they all related to “User”, but are all quite different. You do need a relatively long name to distinguish them. I purposely violated some of Go’s naming convention in order to achieve good readability, I will explain them one by one.

One rule I followed is “The greater the distance between a name’s declaration and its uses, the longer the name should be” by [Andrew Gerrand](https://talks.golang.org/2014/names.slide#4)².I learned it from a great article from [Dave Cheney](https://dave.cheney.net/practical-go/presentations/qcon-china.html)³ . Based on it, I created my own rule: “Use a long name for a type (struct, interface) or a function to make it clear, and a use short name for a local variable”, because a local variable has shorter distance between declaration and usage.

### Name for types:

When I encounter a name, it is important to understand which type I am dealing with, which layer it is in. For example, “UserDataInterface”, tells me that it is for domain model “User”, and it provides data service (persistence layer) and it is an interface.

**Model:
**This layer is for domain model and the easiest one to give a name. For example, “User” for domain type user.

**Dataservice:**
This layer is for data persistence service. For example, use the name “UserDataMySql” for user persistence service for MySQL database; use the name “UserDataCouchdb” for user persistence service for CouchDB database. Different implementations for user data service share the same prefix “UserData” and the interface for them is “UserDataInterface”.

“CourseDataInterface” is the interface for course data service and “CourseDataMySql” is the name of concrete type of course data service for MSql database. All data service share the same prefix “[model]Data”.

**Use Case:**
This layer is for use cases. “RegistrationUseCase” is the concrete type for registration use case and the interface is “RegistrationUseCaseInterface”. “ListCourseUseCase” is the concrete type for list course use case. All use case share the same suffix “UseCase”

**Interface:**
In Clean Architecture, all business logic is programmed on interfaces. It is important to recognize a type is an interface when you encounter it , so I add suffix “Interface” to all interfaces, for example “UseCaseInterface” . If you think it is too long, you can replace “Interface” with Acronyms like “I”, “If” or “Intf”. To me, typing is not a problem further more IDE took care of it most of the time.

**Constants:**
For constants, it is important to distinguish them from variables, so I use all caps and if it is a multi-words constant I use the Snake case instead of the camel case (I knew it against the Go naming convention). For example, “QUERY_USER_BY_NAME” instead of “queryUserByName”, which makes it a lot easier to read.

## Conclusion:

Coding style is crucial to make programming effective. Good naming makes code self documenting. Using long name for a type or a function and short name for local variables. It is fine to put “interface” in the name of an interface. Using all caps in the snake case to name constants to make them easy to identify.

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Other articles:

Please read the rest of the articles in this series in “[Go Microservice with Clean Architecture](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

## Reference:

[1][Go Microservice with Clean Architecture: Application Layout]
(https://jfeng45.github.io/posts/go_microservice_application_layout/)

[2][What’s in a name?]
(https://talks.golang.org/2014/names.slide#4)

[3][Practical Go: Real world advice for writing maintainable Go programs]
(https://dave.cheney.net/practical-go/presentations/qcon-china.html)
