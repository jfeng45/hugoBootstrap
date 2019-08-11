---
title: "清晰架构（Clean Architecture）的Go微服务: 编码风格"
shortTitle: "编码风格"
date: 2019-07-24
draft: false

tags: ["Golang", "清晰架构",  "编码风格"]
categories: ["Go微服务"]

# canonicalUrl: "https://medium.com/@jfeng45/go-micro-service-with-clean-architecture-coding-style-a4da35a27d90"

description: "编码风格对于编程高效至关重要。 良好的命名使代码实现自我解释。 它包括三个部分：程序结构，编码规则和风格，命名约定"

---


编码风格在编程中是一个相对乏味的主题，但是合适的编码风格对一个有效的程序员是至关重要的。 它有三个组成部分：

1. 程序结构 （ application layout）

1. 编码规则或风格

1. 命名约定

我已经在[清晰架构（Clean Architecture）的Go微服务: 程序结构](https://jfeng45.github.io/posts/go_microservice_application_layout/)¹中讨论了程序结构，因此本文将介绍后两点。


##### **编码规则或风格**
<br/>

1. 没有包级别(package level)变量。

       包级别变量打破了函数封装并使函数有了不确定。我在本程序中遵循了这个规则，唯一的例外是在“容器”包中，因为它负责程序级配置，在这里很难做到。其实即使在“容器”包中也可以去除掉包级别变量，但它需要付出很多努力，有些得不偿失。但是，由于包级别变量仅限于“容器”包中，因此导致的破坏大大减少。

2.  尽量少使用常量

       常量也是包级别，因为具有不变性，它们作为全局变量的危害较小，但它们仍然会破坏函数封装并需要进行限制。

3. 依赖于接口而不是具体类型

    当你尝试最小化外部更改对程序的影响时，无论外来者是函数，包还是应用程序，都要使代码依赖于接口而不是具体类型。

##### **命名规则**

最佳的代码是自我解释的，良好的命名起着至关重要的作用。Go的命名约定与Java非常矛盾。尝试之后，我发现它是有效的，在创建简洁的代码同时并没有降低可读性。 Go是一种相对与更底层接近的语言，人们用它来编写网络，驱动程序和docker容器代码。在这些环境中，使用简洁的名称是合适的。

但是，在编写业务应用程序时，我们需要创建许多不同的类型或结构来处理相似的业务概念，简洁的命名便不再适用。例如，为了处理与“用户”相关概念，我们有“User”，“UserDataService”，“RegistrationUseCase”，“RegistrationUseCaseIterface”和“UserDataInterface”，它们都与“User”有关，但都完全不同。你确实需要一个相对较长的名字来区分它们。为了获得良好的可读性，我有意违反了Go的一些命名约定，我将逐一解释它们。

我遵循的一条规则是“变量声明（name declaration）与其使用之间的距离越大，名称应该越长”[Andrew Gerrand](https://talks.golang.org/2014/names.slide#4)². 我从[Dave Cheney](https://dave.cheney.net/practical-go/presentations/qcon-china.html)³的一篇文章中学到了这一点。基于它，我创建了自己的命名规则：“为类型（结构，接口）或函数命名使用长名称使其清晰易读，为局部变量命名使用短名称”，因为局部变量声明和使用之间的距离较短。

##### **给类型（types）命名:**

当我看到一个名字时，重要的是要了解它是哪种类型，它处于哪个层。例如，“UserDataInterface”，告诉我它是域模型“User”，它提供数据服务（持久性），并且它是一个接口。

**Model:**

域模型层，是最容易提供名称的层。 例如，域模型用户的“User”。

**Dataservice:**

数据持久性服务层。 例如“UserDataMySql”作为用户持久性服务MySQL数据库的命名; “UserDataCouchdb”作为CouchDB数据库的用户持久性服务的命名。 用户数据服务的不同实现共享相同的前缀“UserData”，并且它们的接口是“UserDataInterface”。

“CourseDataInterface”是课程数据服务的接口，“CourseDataMySql”是具体课程数据服务MSql数据库实现的名称。 所有数据服务共享相同的前缀“[model]Data”。

**Use Case:**

用例层。 “RegistrationUseCase”是注册用例的具体类型，接口是“RegistrationUseCaseInterface”。 “ListCourseUseCase”是课程列表用例的具体类型。 所有用例共享相同的后缀“UseCase”

**Interface:**

在清晰架构（Clean Architecture）中，所有业务逻辑都是基于接口调用的。 在遇到类型时，重要的是要识别类型是接口还是具体实现，因此我在所有接口上添加后缀“Interface”，例如“UseCaseInterface”。 如果你认为它太长，你可以用“I”，“If”或“Intf”等缩略语替换“Interface”。 对我来说，打字不是一个问题，IDE在大多数时间里都已经把问题解决了。

**Constants:**

对于常量，重要的是将它们与变量区分开来，所以我使用全部大写，如果是多字段常量（multi-words constant），我使用蛇形命名法（Snake Case）而不是驼峰命名法（Camel Case）（我知道它与Go命名约定相左）。 例如，“QUERY_USER_BY_NAME”而不是“queryUserByName”，这使它易读性更好。

##### **结论:**

编码风格对于使编程效率至关重要。 良好的命名使代码自我解释。 对类型或函数使用长名称，对局部变量使用短名称。 将“interface”放在接口名称中是有帮助的。 用蛇形命名法（Snake Case）并且所有字母大写来命名常量，以便于识别。

##### **源程序:**

完整的源程序链接 [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

##### **其他文章:**

请在这里阅读本系列的其他文章 “[清晰架构（Clean Architecture）的Go微服务](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

##### **索引:**

[1][Go Microservice with Clean Architecture: Application Layout]
(https://jfeng45.github.io/posts/go_microservice_application_layout/)

[2][What’s in a name?]
(https://talks.golang.org/2014/names.slide#4)

[3][Practical Go: Real world advice for writing maintainable Go programs]
(https://dave.cheney.net/practical-go/presentations/qcon-china.html)
